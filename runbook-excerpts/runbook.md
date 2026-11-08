# IRDP runbook

> Copied from `RUNBOOK.md` in the IRDP product. [What was changed](README.md#what-was-changed).

How to run the Infinitech Regulated Data Platform reference after handoff.
Each procedure lists who may run each step, the exact commands, what proves
a step worked, and how to back out. Infinitech doesn't operate IRDP for
you: the reference comes with no on-call or 24x7 support.

## Procedures

| Procedure | Use it when | Script | Tested live |
| --- | --- | --- | --- |
| [Recover in the second AWS region](recover-in-second-region.md) | The primary region is down: restore the copied backup, the default, or use the warm standby | `scripts/recover-second-region.py` | Drilled with `make demo-failover` in cycles 10, 11, 12, 15 and 16, both paths recovered. In cycle 16 the script ran every step live (`both`, run) against a primary declared lost, not down; the steps haven't been run by hand |
| Fail over to the Azure standby | A deployment with cross-cloud recovery, built on request: AWS `us-east-1` is down, or a DR drill | `scripts/failover-azure-gov.sh` | No. Blocked until the Azure DR subscription, its Terraform root, and the lake copy job exist. Only the lake fails over |
| [Rotate keys](rotate-keys.md) | The rotation schedule, a suspected leak, someone leaving, a handoff | `scripts/rotate-keys.sh` | No; not yet run in a demo cycle |
| Scale Airflow | The queue keeps growing, or workers are Pending | `scripts/scale-airflow.sh` | No; not yet run in an Airflow cycle |
| Review a chaos drill | Weekly, and after any drill | | No; drill records exist from cycles 3 and 4, but none has been reviewed with this procedure |
| First GovCloud run | Before a customer's first GovCloud deployment | | No; nothing has run in GovCloud |

Also: the [failure-mode catalog](failure-modes.md) (symptom, first
command, fix), the changelog, and the
[handoff checklist](handoff-checklist.md).

## Every script prints its procedure first

Each runbook script prints every command before running it, and stops at
the first one that fails. `--dry-run` prints all of them and runs none; it
needs no credentials:

```bash
bash scripts/failover-azure-gov.sh --dry-run
bash scripts/rotate-keys.sh --dry-run
bash scripts/scale-airflow.sh --dry-run
scripts/recover-second-region.py both --dry-run
```

Steps that only a person can take (Snowsight, an Entra sign-in, checking a
fingerprint, declaring an outage) print "By hand" and wait for you to type `done`.

## Who runs what

| Who | Runs | Signs in with |
| --- | --- | --- |
| CI, through the gated workflows | Every Terraform apply, the secrets copy, demo loads, drills, and the demo report | GitHub OIDC; no stored cloud keys (`docs/applying.md`) |
| You, from a terminal with `gh` | Dispatching those workflows; each saved plan waits for you to type `apply` | `gh`, signed in as the repository owner |
| You, at a workstation | Key generation, Snowsight as `ACCOUNTADMIN`, Entra app secrets, Azure role assignments, PIV/CAC steps where policy requires them | Your own SSO with MFA |
| An operator in the cluster | Reading and editing the `airflow` and `monitoring` namespaces, the UIs | Identity Center `IrdpDemoOperator`, through the Session Manager tunnel |

The `IrdpDemoOperator` permission set can open the tunnel and edit those
two namespaces, and nothing else: no KMS, no S3, no node group changes.
Steps that need more say so.

## Tools

AWS CLI v2 with the Session Manager plugin, `kubectl` 1.34 to 1.36, `jq`,
`gh`, OpenSSL, and Python 3. Add the Azure CLI for failover and the Entra
secret. To reach the cluster:

```bash
eval "export $(scripts/eks-tunnel.sh start)"     # sets KUBECONFIG
scripts/eks-tunnel.sh stop
```

Run `stop` when you're done: `start` leaves a watchdog running that
reopens a stalled tunnel, and after your AWS session expires it keeps
retrying about once a minute. `ORCHESTRATOR=airflow make demo-ui` stops
both when you press Ctrl-C.

(`docs/eks.md`,
`docs/demo.md`).

## Keeping it true

`bash scripts/runbook-test.sh` checks that every link, repository path,
`make` target, workflow input, and runbook-script call in these pages still
exists or runs. pre-commit runs it, so CI's lint job fails on drift. When a
procedure changes, or a drill or incident shows one is wrong, change it in
the same pull request and add a line to the
changelog.
