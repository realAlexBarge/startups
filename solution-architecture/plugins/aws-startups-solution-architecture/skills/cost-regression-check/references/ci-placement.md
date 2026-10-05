# CI Placement

Run the job on the pull request, after the plan, synth, or change set exists and before anything is applied. It reads what the IaC tool produced, runs the cost script, and updates one comment. It does not apply anything and needs no write access to the account.

## Placement by delivery flow

| Flow                                                         | Where the job goes                                                                                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Plan on the pull request, apply on merge                     | In the pull request pipeline, after the plan step, reading the saved plan                                                                  |
| CDK with `cdk deploy` on merge                               | In the pull request pipeline, after `cdk synth` on the branch and on the base branch                                                       |
| CloudFormation change sets reviewed, then executed on merge  | In the pull request pipeline, after the change set is created and described                                                                |
| Apply on push to the production branch, with no pull request | Nowhere yet. Require pull requests into that branch first; without one there is no point where a person decides                            |
| Apply driven by an IaC automation service                    | Where that service produces the plan, if it exposes the plan as an artifact; otherwise run a plan in the pull request pipeline for the job |

The fourth row is common in small teams and is the case this check exists for: a change goes from a laptop to production with no second pair of eyes. Adding a required pull request is the first deliverable there, and the cheapest one.

For CodePipeline and CodeBuild mechanics, use `Skill("aws-core:aws-deployment")`. That skill does not cover GitHub Actions or GitLab CI, so write those jobs from the CI provider's own documentation.

## Credentials

Keep the cost job's credentials separate from the deploy credentials.

- Producing the plan usually needs read access to state and the account. Reuse whatever the existing plan step uses; the cost job adds nothing to it.
- Pricing through the Price List Query API needs only `pricing:GetProducts`. Give the job its own role with that permission and nothing else. For role and federation mechanics, use `Skill("aws-core:aws-iam")`.
- Pricing from downloaded price list files needs no AWS credentials (see `estimators.md`).
- On GitHub, a `pull_request` run from a forked repository gets no secrets other than `GITHUB_TOKEN`, and that token is read-only ([events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)). The job can post or update no comment there, and a plan that needs credentials to read state, such as most Terraform plans, cannot run. If the repository takes pull requests from forks:
  - Make the fork run write its result to the job summary (`GITHUB_STEP_SUMMARY`) and to an artifact. Where it can still compute the numbers without secrets (CDK with a committed `cdk.context.json`, priced from downloaded price files), that result is the numbers. Where it cannot, the result says the check did not run and why, and the job exits as `on_estimator_failure` says (non-zero under `fail`, the value set at setup), so a fork pull request never passes the check silently.
  - To get the comment on fork pull requests too, add a separate workflow in the base repository triggered by `workflow_run`, which runs with write access, downloads the artifact, and posts or updates the comment. It treats the artifact as data: it reads the result and the pull request number, checks that number against the triggering run, and never runs anything from the artifact.
  - Never use `pull_request_target` to check out and run code from the fork. It runs with the base repository's secrets and a write token, so the fork's code would run with both ([securely using pull_request_target](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target)).

## The comment

Post one comment per pull request and update it on every push, so the thread shows the current numbers and not a history of stale ones. Use a fixed layout so readers learn where to look:

```text
Cost check: list price, on demand, before credits, Free Tier, Savings Plans and private pricing

Floor added by this change:    <amount>/month   threshold <amount>/month   <within | EXCEEDS>
Stack floor after this change: <amount>/month   cap <amount>/month         <within | EXCEEDS>
Cost per <unit>:               <before> -> <after> (<change>)   threshold <amount>   <within | EXCEEDS>

Largest contributors: <resource, dimension, amount> (up to five)
Unpriced: <resource and reason> | none
Assumptions flagged as unverified: <assumption and the change that touched it> | none
Not covered: values known only after apply, resources outside the plan, Spot, time-limited Free Tier offers, usage not in the assumptions file
Inputs: thresholds <file at commit>, assumptions <file at commit>, prices <source and publication date>
```

Every amount in the comment comes from the script output. When a line shows EXCEEDS, add one sentence naming the three outcomes from `thresholds.md` (change it, accept it in the thresholds file, or move the threshold in a separate pull request), so the author does not have to look them up.

## Advisory first, then blocking

Start advisory: the job comments and never fails the build on an exceedance. A run that produced no number follows `on_estimator_failure`, which is `fail` from setup, so a check that stopped working shows up during the advisory period instead of looking like a run with nothing to report. Blocking a merge on numbers nobody has checked against a bill teaches the team to work around the check.

During the advisory period, for each merged change that crossed or nearly crossed a threshold, compare the estimated floor with what the bill shows once the change has run for a full billing period. Use `Skill("aws-core:aws-billing-and-cost-management")` for the Cost Explorer query, and do the comparison in a script.

Move to blocking when the bill owner agrees the estimates track the bill. Then:

- Make the job a required status check that fails on an exceedance with no matching entry in `accepted` in the thresholds file.
- Leave the override where it is: an `accepted` entry, which needs the bill owner's review because they own the file.
- Review `on_estimator_failure` with the bill owner: keep `fail`, or set `pass` if an estimator or price source outage must not block merges once the check is required. Either way, the comment says the check did not run. A silent pass is the one outcome to rule out.

## Keep the job cheap to live with

- Run it only when infrastructure paths change.
- Fail on the job's own errors loudly, with the step that failed, rather than printing a zero.
- Pin the IaC tool, the estimator, and the price source in the job, and change them in their own pull requests, so a change in the numbers is never a side effect of a tool upgrade.
