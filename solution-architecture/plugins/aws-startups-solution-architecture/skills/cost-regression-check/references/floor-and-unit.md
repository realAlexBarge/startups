# Floor and Unit

Report two numbers for every change, because they answer different questions on different timelines.

- Floor added: what the change adds to the monthly bill at zero traffic. These are charges that accrue whether or not anything uses the resource. The floor is what the team pays today, before the product has users.
- Change in cost per unit: how much more or less one unit of the team's own usage costs to serve after the change. This is the number that decides whether the bill grows in line with usage or faster.

A change can move one and not the other. A new NAT gateway raises the floor even if no traffic reaches it. A chattier function or a per-tenant index raises the cost per unit and leaves the floor where it was. A blended monthly number at an assumed volume hides which one moved.

## Classify price dimensions, not resources

Many resources carry both kinds of charge, so classify each price dimension the stack uses. A NAT gateway is charged for each hour it is available and for each gigabyte it processes ([NAT gateway pricing](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html)): the hourly charge is floor, the per-gigabyte charge is usage.

| Dimension kind                        | Examples                                                                                                                                                                                 | Counts toward |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| Hourly while provisioned              | NAT gateway hour; Application Load Balancer hour; interface VPC endpoint hour in each Availability Zone; public IPv4 address hour; EC2 instance hour; provisioned database instance hour | Floor         |
| Provisioned capacity per month        | EBS storage per GiB-month; gp3 IOPS and throughput provisioned above the included baseline                                                                                               | Floor         |
| Minimum capacity of a scaling service | Aurora serverless minimum ACUs                                                                                                                                                           | Floor         |
| Per request, per byte, per duration   | Lambda requests and GB-seconds; NAT gateway and interface endpoint data processed; load balancer capacity units; data transfer                                                           | Usage         |

Convert hourly dimensions to a month with the same convention the job uses everywhere. AWS Pricing Calculator uses 730 hours a month ([Pricing Calculator](https://docs.aws.amazon.com/pricing-calculator/latest/userguide/migrate-SMC.html)), and using the same number keeps the comment comparable to an estimate a founder already has.

Sources for the rows above, read before relying on a row:

- Application Load Balancer: charged for each hour or partial hour it runs, plus load balancer capacity units used ([Elastic Load Balancing pricing](https://aws.amazon.com/elasticloadbalancing/pricing/)).
- Interface VPC endpoints: charged for each hour the endpoint is provisioned in each Availability Zone, plus data processed ([AWS PrivateLink pricing](https://aws.amazon.com/privatelink/pricing/)). An endpoint in three Availability Zones is three hourly charges.
- Public IPv4 addresses: charged per address per hour, whether attached to a resource or not, since 2024-02-01 ([announcement](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/)). A subnet setting that assigns public addresses adds floor per instance.
- gp3 volumes: a baseline of 3,000 IOPS and 125 MiB/s is included with the price of storage, and IOPS or throughput provisioned above it costs extra ([EBS General Purpose volumes](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html)). Storage performance set by a module default can cost more than the storage itself.
- Aurora serverless (formerly Aurora Serverless v2): the configured minimum capacity is the floor. A minimum of 0 ACUs turns on automatic pause on engine versions that support it, and instance capacity is not charged while paused ([auto-pause](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html)). The same page lists situations where an instance does not pause, so treat a 0 ACU minimum as a zero instance floor only after checking those against the workload.

## Where floors hide in a plan

Read the resource changes in the plan, not the module call in the diff. The line a reviewer sees is rarely the line that costs money.

- Module defaults. A network module can create one NAT gateway per Availability Zone, interface endpoints for several services, and public addresses on every instance. A storage module can provision IOPS and throughput above the baseline.
- Multipliers. `count`, `for_each`, one resource per Availability Zone, per environment, or per tenant. A one-line change to that list can multiply every floor the module creates.
- Changes to existing resources. A larger instance class, a higher minimum capacity, or more provisioned IOPS raises the floor with no new resource in the plan. The estimator must price the before and after of updates, not only creates.
- Environments. A change merged once can apply to several stacks. Price each stack the change reaches, or state which one the comment covers.

## Choose the unit

Pick one primary unit the team already counts or can count: requests, active tenants, jobs, processed documents, generated images. Prefer the unit the product is priced on, because that is the one the bill owner can compare with revenue.

Write a usage assumptions file in the repository. For each usage-priced dimension in the stack, it states the quantity consumed per unit, where that quantity came from, and when it was measured. For example: the API function uses 1,024 MB for a measured average duration per request, taken from function metrics on a given date; the NAT gateway processes a measured number of megabytes per 1,000 requests, taken from flow logs or gateway metrics.

- Every value carries a source and a date. A value without a source is a guess, and the comment says so next to the number it produced.
- Keep the file small. Cover the dimensions that dominate cost per unit, and list the rest as not modelled rather than inventing values.
- Store raw measurements, not derived costs. The script multiplies by current prices on every run, so a price change shows up without editing the file.

## How a usage assumption goes wrong

Lambda charges per request and for duration in GB-seconds, which is memory multiplied by duration ([Lambda pricing](https://aws.amazon.com/lambda/pricing/)). Raising memory from 512 MB to 1,024 MB at unchanged duration doubles the duration charge per request and leaves the request charge alone. Duration often falls when memory rises, because CPU scales with memory ([Lambda performance](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-2/)), so the real change can be close to zero.

The script cannot know the new duration until the change runs. Make it detect when a change touches an input of an assumption (memory for duration, instance size for throughput) and print the assumption as unchanged and unverified in the comment. A confident number from a stale assumption is worse than a flagged one.

## What neither number covers

State these in the comment footer rather than leaving a reader to assume they are included:

- Resources the application creates at runtime, such as per-tenant indexes or queues. The plan cannot see them, so model them as usage per tenant in the assumptions file or list them as not covered.
- Data transfer, unless the assumptions file has a measured volume per unit for it.
- Credits, Free Tier, Savings Plans, Reserved Instances, and private pricing. The comment reports list price on purpose; see `thresholds.md`.
