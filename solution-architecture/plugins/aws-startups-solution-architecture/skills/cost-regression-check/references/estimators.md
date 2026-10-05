# Estimators

The job needs two inputs for every pull request: the resource changes the infrastructure code will make, and a price for each priced dimension of those resources. Take the first from the IaC tool and the second from AWS Price List data or an IaC cost estimator. The cost script combines them with the usage assumptions file.

## What each IaC format exposes about a change

| Format         | Produce                                                          | Read                                                                                                                                                       | Credentials                                                                                                                                                                                         |
| -------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Terraform      | `terraform plan -out=<file>`, then `terraform show -json <file>` | `resource_changes`: one entry per resource instance, with `change.actions` (for example `create`, `update`, `delete` then `create`), `before`, and `after` | Whatever the plan already needs to read state                                                                                                                                                       |
| AWS CDK        | `cdk synth`                                                      | CloudFormation templates in `cdk.out`; synthesize the base branch and the pull request branch and compare, or create a change set from the template        | None if `cdk.context.json` is committed; context lookups otherwise need read access to the account ([CDK pipelines](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.pipelines-readme.html)) |
| CloudFormation | A change set against the deployed stack, then describe it        | Each `ResourceChange`: `Action` (`Add`, `Modify`, `Remove`, `Import`, `Dynamic`, `SyncWithActual`) and `Replacement` (`True`, `False`, `Conditional`)      | Read access to the target stack, and change set creation                                                                                                                                            |

Sources: [Terraform JSON output format](https://developer.hashicorp.com/terraform/internals/json-format), [CDK synthesis](https://docs.aws.amazon.com/cdk/v2/guide/stages.html), [viewing a change set](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets-view.html), [ResourceChange](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_ResourceChange.html).

Notes per format:

- Terraform marks values known only after apply in `after_unknown`. A count or size that depends on one of those cannot be priced from the plan; report it as unpriced with the reason.
- Credit a floor reduction only for a `resource_changes` entry whose `change.actions` include `delete`. A `removed` block with `destroy = false` takes a resource out of state and leaves it running ([removed block](https://developer.hashicorp.com/terraform/language/block/removed)), so it is still billed.
- `cdk diff` shows what will change and whether a resource is replaced. It prices nothing. Price the synthesized templates. For CDK mechanics, use `Skill("aws-core:aws-cdk")`.
- A change set is created in the target account and region. For change set mechanics and pre-deployment validation, use `Skill("aws-core:aws-cloudformation")`.
- Handle every change set `Action`, not only the first three. `Dynamic` means CloudFormation cannot determine the action yet; report the resource as unpriced with that reason. `Import` brings a resource that already exists under the stack, so it counts toward the stack floor but adds nothing to the floor added. `SyncWithActual` changes only CloudFormation metadata and costs nothing. A `Remove` with `PolicyAction` `Retain` leaves the resource running outside the stack, and `ReplaceAndRetain` leaves the old resource running next to its replacement, so count a retained resource as still billed. Otherwise a retained resource offsets a real increase in the same change set and the comment reports a smaller net floor added.

The stack floor cap needs the floor of the whole stack, not only of the changed resources, and a change set or `cdk diff` lists only what changes. Price the whole stack from Terraform `planned_values`, or from every `resource_changes` entry including those with the `no-op` action. For CDK, price the full synthesized template. For CloudFormation, price the template of the change set, which `GetTemplate` returns when called with the change set name ([GetTemplate](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_GetTemplate.html)).

## Where prices come from

Pick one source for the floor and keep it for the advisory period, so comments stay comparable.

### AWS Price List data in the cost script

The script maps each resource type the stack uses to a Price List service code and filter attributes, then reads the price for each dimension. For service codes and attribute gotchas, use `Skill("aws-core:aws-billing-and-cost-management")`.

- The Price List Query API (`GetProducts`) needs an IAM identity with the `pricing:GetProducts` permission ([Price List actions](https://docs.aws.amazon.com/service-authorization/latest/reference/list_pricing.html)). Its endpoints are in us-east-1, eu-central-1, and ap-south-1; the endpoint region is unrelated to the region being priced ([AWS Price List](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/price-changes.html)).
- Price list files can also be downloaded from plain HTTPS URLs per service and region ([getting price list files manually](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/using-the-aws-price-list-bulk-api-fetching-price-list-files-manually.html)). This suits a job that should not hold AWS credentials for pricing. On pull requests from forks it removes only the need for a pricing credential; the plan and the comment have their own limits there (see `ci-placement.md`). Some files are large: NAT gateway prices are in the `AmazonEC2` file, which for us-east-1 was about 480 MB on 2026-10-05. Rather than download it on every run, have a script extract the products the mapping uses into a pinned file in the repository, refresh it in its own pull request, and print its publication date in the comment.
- Price list data includes perpetual Free Tier offers at USD 0, next to the paid rate for the same usage ([AWS Price List](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/price-changes.html)). In the Lambda file for us-east-1, for example, the requests Free Tier and the GB-seconds Free Tier are separate zero-priced products (SKUs) with a usage type that starts with `Global-`, apart from the paid SKUs, so filter at the product level. Many paid dimensions are also tiered, with a rate per usage range given by `beginRange` and `endRange`. Drop the Free Tier products, and take the paid rate for the tier the account's monthly usage falls in, or the first paid tier when that usage is unknown. A script that takes the first product matching its filter can price a Lambda memory change at zero.
- The mapping is code the team owns. Cover the resource types the stack actually uses, and report every unmapped type as unpriced. Extending the mapping is a normal pull request.

This path needs no third-party account and keeps every number reproducible from the repository and a dated price file.

### An IaC cost estimator

A third-party estimator can produce the floor without a hand-written mapping. Infracost is one example. Its free CI/CD tier lists cost estimates for Terraform, CloudFormation, and CDK ([pricing](https://www.infracost.io/pricing/)), and it reports baseline costs apart from usage costs, with usage values read from an `infracost-usage.yml` file ([usage costs](https://www.infracost.io/docs/features/usage_based_resources/)). It needs an Infracost account. Thresholds and merge blocking are its Cost Guardrails, which the pricing page lists in a paid plan.

If the team uses it, take only the baseline figure for the floor and keep the per-unit change, the thresholds, and the decision in the repository as described in this skill. Read its data-handling documentation before sending infrastructure code to a third party, and pin the version the job runs, since its commands have changed over time.

### AWS Pricing Calculator API

The Pricing Calculator API models workload estimates from usage lines and can apply rates before discounts, after discounts, or after discounts and commitments ([CreateWorkloadEstimate](https://docs.aws.amazon.com/boto3/latest/reference/services/bcm-pricing-calculator/client/create_workload_estimate.html)). It needs account credentials. Use it, if at all, to check a per-unit assumption at a modelled volume during setup, not on every pull request.

### Never EstimateTemplateCost

The CloudFormation `EstimateTemplateCost` API returns "an AWS Simple Monthly Calculator URL" ([API reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_EstimateTemplateCost.html)), and the Simple Monthly Calculator is no longer supported ([migration note](https://docs.aws.amazon.com/pricing-calculator/latest/userguide/migrate-SMC.html)). A model asked to price a template tends to reach for it. Do not.

## What every estimator misses

Print these as not covered in the comment footer, rather than letting a reader assume they are in the number:

- Values known only after apply, and counts that depend on them.
- Resources created outside the plan: by the application at runtime, by an operator in the console, or by another stack.
- Spot prices and time-limited Free Tier offers, which the Price List does not include.
- Usage-priced dimensions with no entry in the usage assumptions file, including most data transfer.
- Credits, Savings Plans, Reserved Instances, and private pricing, by design.

## Test the cost script

The script decides what the comment says, so test it like any other code that gates a merge:

- Commit small fixture plans (or synthesized templates, or change set descriptions) with the expected floor and per-unit output, using a pinned price file.
- Include a fixture with a resource type the mapping does not know, and assert that it is reported as unpriced and not as zero.
- Include an update that raises a minimum or provisioned capacity on an existing resource, and assert that the floor rises.
- Include a change that touches an input of a usage assumption, such as Lambda memory, and assert that the assumption is flagged as unverified.
- Include the same Lambda memory change against a pinned price file that contains the Free Tier products, and assert that requests and GB-seconds are priced at a nonzero paid rate.
- In the setup session, run the script against the current main branch and record the stack floor and cost per unit as the starting point, from the script output only.
