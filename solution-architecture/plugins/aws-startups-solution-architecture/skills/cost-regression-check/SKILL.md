---
name: cost-regression-check
description: "This skill should be used when a team that pays its AWS bill from a fixed cost ceiling, with no FinOps or platform reviewer, wants the cost of every infrastructure-as-code change (Terraform, AWS CDK, or AWS CloudFormation) reported on its pull request before apply, including when it asks to add Infracost or another cost estimator to CI. It sets up, once, a CI job and a repository-owned script that report two numbers per change: the fixed monthly floor added at zero traffic, and the change in cost per unit of the team's own usage, such as per 1,000 requests. Both are compared at list price against absolute thresholds derived from the ceiling and kept in the repository, so the author and the bill owner decide on the pull request. Not for spend that already happened (aws-core:aws-billing-and-cost-management), one-off price lookups, advice on infrastructure code as it stands, general release risk, or choosing services that avoid a floor."
license: Apache-2.0
metadata:
  audience: startup
---

# Cost Regression Check

Report the cost a change adds on the pull request that adds it, before apply. A team paying from a fixed cost ceiling usually has no FinOps reviewer, and often no platform engineer between a merge and production, so the pull request is the last point where a person sees the cost and can still decide. The bill shows the same change a month later, after the money is spent.

Set this up once, in a working session: add a CI job and a cost script to the repository, which owns and reviews both like any other code. On each pull request the job runs the cost script, which reads the resource changes, prices them, computes both numbers, and comments. On the default path the script is the estimator; if the team uses a third-party estimator for the floor, the script reads its output. No model runs per pull request, so the check costs nothing to attend to until a threshold is crossed.

## Decisions, and why the startup answer differs

| Decision                   | Large-company default                                   | This skill                                                   | Reason                                                                                                                                                                                                                                                                  |
| -------------------------- | ------------------------------------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Report the floor apart     | One blended monthly delta at an assumed volume          | Two numbers: floor added, and change in cost per unit        | A new NAT gateway, load balancer, or database minimum is noise in a large bill and can be a material share of a small ceiling. A blended number hides which of the two moved.                                                                                           |
| Use the team's own unit    | Percentage change per service against last month        | Cost per 1,000 requests, per tenant, or per job              | No FinOps team turns a per-service change into what a customer costs to serve. The bill owner compares cost per unit with what a unit earns, and that number can double while every price stays flat, for example after a Lambda memory increase at unchanged duration. |
| Absolute thresholds        | A percentage of a stable baseline, set by a FinOps team | Two amounts derived from the ceiling, kept in the repository | A small or credit-funded bill has no stable base for a percentage. A large percentage on a near-zero bill is noise; a new fixed floor that takes a visible share of the ceiling is not.                                                                                 |
| List price                 | Negotiated or commitment-adjusted rates                 | Public on-demand price, labelled as such on every comment    | Credits hide the bill that arrives when they run out, and the post-credit bill is the one the ceiling has to survive.                                                                                                                                                   |
| Decide on the pull request | Route an exceedance to a FinOps queue                   | The author and the bill owner decide in the pull request     | There is no queue behind them. A decision recorded in the pull request is the only review the change gets.                                                                                                                                                              |

## Scope

Apply this skill when all of these hold:

- Infrastructure is defined in Terraform, AWS CDK, or AWS CloudFormation, and changes reach production through pull requests, or the team is willing to make them do so.
- One person owns the AWS bill and can state a monthly ceiling.
- The team can name a usage unit it already counts or can count from logs or metrics.

Do not apply it to spend that already happened (use `Skill("aws-core:aws-billing-and-cost-management")`), to resources the application creates at runtime outside the plan, to optimization advice on infrastructure code as it stands, or to choosing services without a floor. This skill detects that a change added a floor; it does not decide which services have one.

## Workflow

1. Read **`references/thresholds.md`**. Get the ceiling from the bill owner, derive the floor threshold and the per-unit threshold, and write the thresholds file with an owner.
2. Read **`references/floor-and-unit.md`**. Choose the unit, classify each price dimension the stack uses as floor or usage, and write the usage assumptions file with a source for every value.
3. Read **`references/estimators.md`**. Choose how the job gets resource changes for the IaC format in use and where prices come from, then write the cost script and its tests.
4. Read **`references/ci-placement.md`**. Add the job after plan, synth, or change set and before apply, in advisory mode, with the comment format defined there.
5. Run advisory for a set period. After merged changes have run for a billing period, compare estimates with the bill through `Skill("aws-core:aws-billing-and-cost-management")`, then decide with the bill owner whether an exceedance blocks a merge.

## Required deliverables

- A thresholds file in the repository with the ceiling, the floor threshold, the stack floor cap, the per-unit threshold, the unit, the owner, the date each value was set, the `accepted` list of approved exceedances, and `on_estimator_failure`.
- A usage assumptions file mapping each usage-priced dimension to a quantity per unit, each with a source and a date.
- A cost script with tests and committed fixture plans, owned by the repository.
- A CI job that runs on every pull request touching infrastructure code and updates one comment in place.
- A short record of the advisory period: estimated against billed floor for each merged change that crossed or nearly crossed a threshold.

## Rules

- Take every number from a tool: the cost script, an estimator whose output it reads, or AWS Price List data. In the setup session, follow the deterministic-calculation rule in `Skill("aws-core:aws-billing-and-cost-management")` for any arithmetic, and never state a price from memory.
- Label every reported number as list price, on demand, before credits, Free Tier, Savings Plans, and private pricing.
- Report a resource the job cannot price by name, as unpriced. Never count it as zero.
- Never use the CloudFormation `EstimateTemplateCost` API. It returns a link to the Simple Monthly Calculator, which AWS no longer supports.
- Start advisory. Make the check blocking only after the advisory record shows the estimates track the bill.

## Upstream skills

Invoke these for the mechanics they own:

- **`Skill("aws-core:aws-billing-and-cost-management")`**: Price List service codes and attributes, the deterministic-calculation rule, Cost Explorer to compare estimates with the bill, and Budgets and Cost Anomaly Detection for spend after merge.
- **`Skill("aws-core:aws-cdk")`**: `cdk synth` and `cdk diff`. It prices nothing.
- **`Skill("aws-core:aws-cloudformation")`**: change sets and pre-deployment validation. It prices nothing.
- **`Skill("aws-core:aws-deployment")`**: CodePipeline and CodeBuild, when the job runs there. It does not cover GitHub Actions or GitLab CI.
- **`Skill("aws-core:aws-iam")`**: the CI role and its read-only pricing permission.
- **`Skill("aws-startups-solution-architecture:agentcore-patterns")`**: only if a team later adds a model to the job and its output may block a merge.

## Reference files

- **`references/floor-and-unit.md`**: Read before writing the usage assumptions file. Defines the floor and the per-unit change, classifies price dimensions rather than resources, shows where floors hide in a plan, and explains how a usage assumption goes wrong.
- **`references/thresholds.md`**: Read before setting any threshold. Derives both thresholds and the stack floor cap from the ceiling, explains why a percentage fails on a small bill, defines the thresholds file, and records who decides on an exceedance and how.
- **`references/estimators.md`**: Read before choosing an estimator or writing the cost script. Covers what Terraform, CDK, and CloudFormation each expose about a change, the Price List and third-party options, what every estimator misses, and how to test the script.
- **`references/ci-placement.md`**: Read before adding the job. Covers placement for plan-then-apply and apply-on-push flows, credentials and fork pull requests, the comment format, and the move from advisory to blocking.

## Anti-patterns

- One blended monthly number at an assumed volume, which hides whether the floor or the unit moved.
- A percentage threshold against last month's bill on a bill that is small, new, or paid by credits.
- Prices looked up or summed by a model in the pull request instead of by the job.
- An unpriced resource silently counted as zero.
- Reporting after credits, Free Tier, or discounts, so the comment describes a bill that ends when the credits do.
- A usage assumption with no source, or one left unchanged by a change that alters it, such as Lambda duration after a memory change.
- Thresholds that live in CI settings instead of a reviewed file the bill owner owns.
- Blocking merges from the first day, before anyone knows whether the estimates match the bill.
- Many changes each under the floor threshold, with no cap on the total floor of the stack.
