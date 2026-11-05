# Runbook and handoff

A team that takes over IRDP gets a runbook with six procedures, a catalog
of failure modes, and a changelog of what each demo cycle found, plus the
checklist that hands the platform from Infinitech to them. The pages here
are copied from that runbook.

| File | What's in it |
| --- | --- |
| [`runbook.md`](runbook.md) | The runbook's front page: the procedures, when to use each, whether each has run live, who may run which step, and the tools an operator needs |
| [`recover-in-second-region.md`](recover-in-second-region.md) | The whole procedure for bringing the warehouse back in the second AWS region when the primary is down: restoring the newest copied backup (the default) or switching to the warm standby (an option), how to verify each, and failback. The failover drills of demo cycles 10, 11, 12 and 15 ran its calls |
| [`rotate-keys.md`](rotate-keys.md) | The whole key rotation procedure: the KMS keys, the Snowflake service users' key pairs (with no sign-in gap while the pipeline runs), the Entra client secrets, Airflow's generated secrets, and the quarterly review of who CI trusts. None of it has run live yet |
| [`failure-modes.md`](failure-modes.md) | Three entries from the failure-mode catalog, each with the signal, the first command to run, and the fix: a failed Step Functions run and its redrive, no successful run in 30 minutes, and a pipeline task that can't start because its image or its network path to ECR is gone |
| [`handoff-checklist.md`](handoff-checklist.md) | The whole handoff checklist: eight steps, the evidence that closes each, every credential to rotate, the open items on the day, and the sign-off |

## How the procedures run

Each procedure lists who may run each step, the exact commands, what
proves a step worked, and how to back out. Recovery in the second AWS
region, failover to Azure, key rotation, and Airflow scaling each have a
script that prints every command before running it and stops at the first
one that fails. `--dry-run`
prints them all, runs none, and needs no credentials. Steps only a person
can take, such as a Snowsight statement or an Entra sign-in, wait for the
operator to type `done`. A check in the product's CI fails when a link,
file path, `make` target, workflow input, or script call in the runbook no
longer matches the code.

## The recovery drills

The walkthrough's failover is one of those drills: the warm standby
recovered in 2.1 minutes with nothing lost, and the snapshot restore in
8.2 ([run record](../evidence/runs/failover-drill.txt),
[all drills](../evidence/results.md#failover-drills-across-cycles)). A
drill stops the primary's schedule rather than losing the region, and a
script runs it; the procedure's steps haven't been run by hand, and the
script that runs them hasn't run live.

## What comes with a deployment

The rest of the runbook: scaling Airflow, reviewing a chaos drill, the
first run in AWS GovCloud, the other failure modes, and the changelog.
Recovery to Azure, instead of a second AWS region, is built on request and
has its own procedure. The scripts the procedures call, and the Terraform
and workflows they run against, come with the product, set up for your
accounts. Email
[hello@infinitechplatforms.com](mailto:hello@infinitechplatforms.com) to
plan one.

## What was changed

The pages are generated from the runbook for each release of this
repository, with three changes:

- A link to a file that isn't here is plain text. A path such as
  `docs/oscal.md` names a file in the product.
- The demo account's ID reads `<account id>` and the product
  repository's name `<repository>`. In the GitHub OIDC subject example in
  the handoff checklist, the owner and repository IDs read `<owner id>`
  and `<repository id>`.
- `failure-modes.md` keeps the catalog's introduction and three entries,
  under their own section headings.
