# Handoff checklist

> Copied from `docs/handoff-checklist.md` in the IRDP product. [What was changed](README.md#what-was-changed).

The steps that move IRDP from Infinitech to the customer, in order, and
the sign-off that closes them. Fill it in at the handoff meeting and keep
the completed copy with the customer's system records. Each step names its
evidence; a step without evidence isn't done.

**Out of scope for the handoff:** Infinitech doesn't run IRDP afterwards
and offers no 24x7 operations in the reference build. ATO sponsorship,
assessor (3PAO) engagement, and continuous-monitoring services aren't part
of IRDP; standing it up doesn't produce an authorization
(`README.md`).

## 1. Agree on what is handed over

State on 2026-11-02, and the state at handoff:

| Piece | State on 2026-11-02 | At handoff | Evidence |
| --- | --- | --- | --- |
| Regulated AWS overlay (`environments/aws`) | Applied to the reference account | | The latest `apply-aws` run |
| Demo cycle stacks (`environments/demo-core`, `demo-sfn`, `demo-platform`, `demo-dr`) | Run live in 14 cycles, each torn down to a clean zero-resource report (`demo-results.md`) | | The latest cycle's readout (`make demo-show`) and `artifacts/zero-resource-report.txt` |
| Snowflake (`environments/snowflake`) | Applied and standing; a cycle that loads Snowflake adds its PrivateLink endpoint and service user, and the teardown removes them | | `make verify-snowflake-between-cycles` output |
| Azure DR overlay | Validates offline only; the DR subscription doesn't exist | | `plan-azure` runs |
| Runbook | Written, checked by `scripts/runbook-test.sh`; no procedure run live | | Changelog |
| Known gaps | [Section 7](#7-record-the-open-items) | | |

- [ ] Both sides agree the table is accurate.

## 2. Transfer ownership

- [ ] **Repository:** transferred to, or forked into, the customer's
  GitHub organization; customer administrators added; branch protection
  applied (`scripts/setup-branch-protection.sh`); the GitHub environments
  re-created with deployment from `main` only, if the repository was forked.
- [ ] **CI trust follows the repository.** The GitHub OIDC subjects the
  AWS roles trust name the repository's owner and IDs, for example
  `repo:<owner>@<owner id>/<repository>@<repository id>:environment:demo-snowflake`.
  After a transfer or fork, update every trust policy to the new subjects
  (`docs/oidc-trust-setup.md`), or CI can't sign in.
  Evidence: a green `plan-aws` run in the new repository.
- [ ] **Reference values replaced.** The settings files, and the examples
  in the runbook, name the reference: account `<account id>`, `us-east-1`,
  bucket names such as `irdp-logs-<account id>-us-east-1`, and the
  `irdp-demo` cluster, which `scale-airflow.sh` uses unless told otherwise
  (`scripts/scale-airflow.sh status --cluster <cluster name>`). Replace them
  with the customer's values, or pass the customer's on each command.
- [ ] **AWS:** the organization's management account and the IRDP accounts
  (IMCLZ) under the customer's control, root user credentials and MFA
  devices handed over in person.
- [ ] **Snowflake:** `ACCOUNTADMIN` held by customer staff with MFA.
- [ ] **Entra ID:** the Airflow app registration's owners are customer
  staff.
- [ ] **Azure:** the DR subscription's Owner is customer staff, once it
  exists.

## 3. Rotate everything Infinitech could read

Infinitech had these values or could have set them. Rotate each after
ownership moves, with the [key rotation runbook](rotate-keys.md):

| Credential | Procedure | Done (date, by) |
| --- | --- | --- |
| `SNOWFLAKE_PRIVATE_KEY` (`IRDP_TERRAFORM`) | [Snowflake: `IRDP_TERRAFORM`](rotate-keys.md#snowflake-irdp_terraform) | |
| `SNOWFLAKE_ORCHESTRATOR_PRIVATE_KEY` (`IRDP_AIRFLOW`) | [Snowflake: `IRDP_AIRFLOW`](rotate-keys.md#snowflake-irdp_airflow) | |
| `AIRFLOW_ENTRA_CLIENT_SECRET` | [Entra client secret](rotate-keys.md#entra-client-secret) | |
| `DEMO_ACCESS_ENTRA_CLIENT_SECRET` (Verified Access) | [Verified Access client secret](rotate-keys.md#verified-access-client-secret) | |
| Airflow's own secrets, in a long-lived deployment | [Airflow's own secrets](rotate-keys.md#airflows-own-secrets) | |
| `PROJECT_D_EVIDENCE_TOKEN`, if it exists | [Project D token](rotate-keys.md#project-d-token) | |
| Every password manager entry Infinitech could open | The customer's own process | |

CMK material doesn't need rotating for a handoff: no person ever holds it.

## 4. Remove Infinitech's access

- [ ] GitHub: no Infinitech collaborators, team members, or app
  installations. Check with
  `gh api repos/<owner>/<repo>/collaborators --jq '.[].login'`.
- [ ] AWS: no Infinitech users or permission set assignments in IAM
  Identity Center, and no Infinitech IAM users.
- [ ] Snowflake: no Infinitech users (`SHOW USERS;`).
- [ ] Entra ID and Azure: no Infinitech guests, app owners, or role
  assignments (`az role assignment list --all`).

## 5. Run each procedure in a sandbox

A customer operator runs each procedure from the runbook alone, with
Infinitech watching but not typing. Record the run:

| Procedure | Run by | Date | Evidence | Result |
| --- | --- | --- | --- | --- |
| [Fail over to the Azure standby](failover-to-azure.md) (blocked until the DR standby exists) | | | | |
| [Rotate keys](rotate-keys.md) | | | | |
| Scale Airflow | | | | |
| Review a chaos drill | | | | |

- [ ] Every step the operator had to ask about is now a fix to the runbook,
  in a pull request, with a changelog entry.

## 6. Hand over the evidence

- [ ] OSCAL component definitions: `make validate-oscal`, then the files in
  `compliance/oscal/emitted/` (`docs/oscal.md`).
- [ ] The latest demo report and zero-resource report from `artifacts/`
  (`docs/demo.md`).
- [ ] Chaos drill records (`runbook/04-review-chaos-drill.md`).
- [ ] Apply history: the workflow runs and their saved-plan artifacts.
  GitHub deletes artifacts after the repository's retention period, so
  download any the customer must keep.

## 7. Record the open items

Known on 2026-11-02; add anything found since:

- [ ] Azure DR: the subscription, a DR root, the lake copy job, and compute
  on Azure aren't built (`docs/azure-gov-dr.md`).
- [ ] Snowflake replication (failover groups) is documented, not built.
- [ ] HSM-backed keys (`custom_key_store_id`) are untested.
- [ ] Nothing has run in GovCloud, Azure Government, or SnowGov
  (first GovCloud run).
- [ ] Airflow alternative only: the demo maps `IrdpDemoOperator` to
  `AmazonEKSEditPolicy` in the Airflow namespaces, which can read their
  Secrets (the Snowflake key, the database password). Leave the customer's
  operators out of `namespace_editors` (`modules/compute/eks-private-cluster`)
  and give them a Kubernetes role limited to port-forwarding, which the
  module doesn't build.
- [ ] Airflow alternative with the Session Manager tunnel only: the node
  role's `AmazonSSMManagedInstanceCore` can read SSM parameters
  account-wide. Ordinary pods can't reach the node's credentials (IMDSv2,
  hop limit 1), and the Airflow namespace enforces Pod Security `baseline`,
  which refuses `hostNetwork`. Pods with `hostNetwork: true` can reach them:
  node-exporter in the monitoring namespace, anything in the Chaos Mesh
  namespace (both `privileged`), and the EKS add-ons in `kube-system`
  ([AWS](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)).
  Keep secrets out of plain SSM parameters in that account, or turn the
  tunnel off (`node_ssm_tunnel`) where operators reach the API another way.
- [ ] No runbook procedure yet for failing over to the second AWS region;
  the drill is `make demo-failover` (`demo.md`).
- [ ] Chaos Mesh drills run only on the Airflow alternative; the default
  Step Functions path has the forced-failure drills (`make demo-chaos`)
  (issue 121).
- [ ] Upstream IMCLZ gaps:
  #178
  (GovCloud accounts and regions) and
  #186
  (Azure Government workspace).

## 8. Sign off

Sign when sections 1 to 7 are complete, or list what's deferred and who
owns it.

| Role | Name | Date | Signature |
| --- | --- | --- | --- |
| Infinitech delivery lead | | | |
| Customer platform owner | | | |
| Customer security officer (ISSO) | | | |
| Customer operations lead | | | |

Deferred items, with owner and due date:

| Item | Owner | Due |
| --- | --- | --- |
| | | |
