# Rotate keys

> Copied from `runbook/02-rotate-keys.md` in the IRDP product. [What was changed](README.md#what-was-changed).

How to rotate each key and secret IRDP uses, who may run each step, what
proves the new one works, and how to back out.
`scripts/rotate-keys.sh` runs the steps in
order and prints each command first; `--dry-run` prints them all and runs
none.

**Status (2026-10-25):** none of these has run live. The commands follow
the AWS, Snowflake, Azure CLI, and GitHub CLI documentation cited below.

## When

| Credential | Cadence | Also rotate after |
| --- | --- | --- |
| Classification CMKs (`alias/irdp-confidential`, `alias/irdp-restricted`) | Automatic every 365 days; on demand when needed | A suspected key compromise; an auditor's request |
| `SNOWFLAKE_ORCHESTRATOR_PRIVATE_KEY` (`IRDP_AIRFLOW`) | 60–90 days | A suspected leak; a maintainer leaving; a change in the service user's scope |
| `SNOWFLAKE_PRIVATE_KEY` (`IRDP_TERRAFORM`) | 60–90 days | The same |
| `AIRFLOW_ENTRA_CLIENT_SECRET` | 90 days | A suspected leak; the app's scope growing; a maintainer leaving |
| `DEMO_ACCESS_ENTRA_CLIENT_SECRET` (Verified Access) | 90 days | The same |
| Airflow's own secrets (database password, Fernet key, API and JWT secrets, Redis password) | Every demo cycle makes new ones | A suspected leak, in a long-lived deployment |
| `PROJECT_D_EVIDENCE_TOKEN` | 30 days, once it exists | Any unexpected use |
| OIDC trust conditions | Quarterly review | A new environment; wider privileges |

Cadences are from
secrets and identity §7.
**After a handoff, rotate everything in this table that Infinitech could
read** ([handoff checklist](handoff-checklist.md)).

## Where each step runs

| Step | Runs | Why |
| --- | --- | --- |
| Make a key pair | **Workstation only** | The private key and passphrase never leave your machine and the password manager |
| Snowsight statements as `ACCOUNTADMIN` | **Workstation only**, a person with MFA | `IRDP_TERRAFORM` is created by the bootstrap SQL, not Terraform; no CI identity holds `ACCOUNTADMIN` |
| Entra client secret | **Workstation only**, an owner of the app registration or an Application Administrator | Needs a person's Entra sign-in; the Azure bootstrap credential has no Graph permissions |
| `gh secret set`, `gh variable set` | Workstation, `gh` signed in as the repository owner | Values go from a file or a prompt straight to GitHub |
| Snowflake applies, the secrets copy, demo loads | CI, dispatched from your terminal by the script | Only the gated workflows change cloud or Snowflake state; you approve each saved plan by typing `apply` |
| CMK rotation | An AWS administrator session in the account | No workflow does it, and the `IrdpDemoOperator` permission set has no KMS access |

GovCloud and Azure Government deployments may require PIV/CAC for the
workstation steps.
Nothing here automates smart-card interaction.

## Classification CMKs

Each key rotates its material automatically every `rotation_period_in_days`
(365 by default). To rotate now:

```bash
export AWS_REGION=us-east-1     # and an administrator session in the account
scripts/rotate-keys.sh cmk
scripts/rotate-keys.sh cmk --class restricted
```

For each alias the script finds the key, checks its material is
AWS-generated, starts an on-demand rotation, and shows the rotation status
and the newest completed rotation.

- **Proof:** `aws kms list-key-rotations` lists the new rotation with
  `RotationType` `ON_DEMAND`. It can take a few minutes to appear.
- **Effect on consumers:** none. KMS keeps every earlier key material and
  uses it to decrypt older ciphertext; EKS, MSK, S3, Secrets Manager, and
  CloudWatch Logs keep the same key ARN.
- **Limits:** 25 on-demand rotations per key, symmetric keys only, and not
  for keys in a custom key store. An on-demand rotation doesn't move the
  automatic schedule
  ([AWS KMS on-demand rotation](https://docs.aws.amazon.com/kms/latest/developerguide/rotating-keys-on-demand.html)).
- **Rollback:** none needed, and none possible: the old material is
  retained, so nothing that used it breaks.
- **In a demo cycle** the keys are created and destroyed with the cycle, so
  rotation matters only for a long-lived deployment.

### HSM-backed keys

The reference doesn't use them. A deployment that sets
`custom_key_store_id` gets keys that KMS can't rotate automatically or on
demand; the only rotation is a new key. Repointing an alias isn't enough:
IRDP's modules take key ARNs, not aliases, so each consumer has to be
re-pointed by a Terraform change. AWS documents no way to change the key of
an existing EKS cluster (`AssociateEncryptionConfig` adds a key to a
cluster that has none), so plan on rebuilding the cluster for its key
(uncertain; confirm with AWS before relying on it). **Never delete or
disable the old key while a cluster uses it:** a deleted key leaves the
cluster unrecoverable
([EKS envelope encryption](https://docs.aws.amazon.com/eks/latest/userguide/envelope-encryption.html)).

## Snowflake: `IRDP_AIRFLOW`

Each Snowflake service user has two public-key slots, `RSA_PUBLIC_KEY` and
`RSA_PUBLIC_KEY_2`, and accepts either key, so a rotation needs no gap
([Snowflake key-pair authentication](https://docs.snowflake.com/en/user-guide/key-pair-auth)).
Terraform sets both for `IRDP_AIRFLOW` from the GitHub Variables
`SNOWFLAKE_ORCHESTRATOR_PUBLIC_KEY` and `SNOWFLAKE_ORCHESTRATOR_PUBLIC_KEY_2`.

Make the new pair first, on your workstation:

```bash
scripts/rotate-keys.sh snowflake-airflow --stage new-key
```

It writes `irdp_airflow_key.p8`, `.pub`, and `.pub.line` into a directory
only you can read, and prints the fingerprint. The directory is
`--key-dir`, default `~/irdp-key-rotation`, and later stages read the pair
from the same place. Give it a directory used for nothing else: `new-key`
sets its permissions to owner-only, and the script refuses your home
directory. It stops if a pair is already there, so a second run can't
overwrite a key that's halfway through a rotation. Store the `.p8` and its passphrase in the
password manager, apart from each other, and remove the directory when the
rotation is finished.

### Between demo cycles (the usual case)

`IRDP_AIRFLOW` exists only while a cycle runs, so between cycles there is
nothing to keep working. Replace the key:

```bash
scripts/rotate-keys.sh snowflake-airflow --stage replace
```

The next `make demo-up` creates `IRDP_AIRFLOW` with the new public key, and
its `demo-secrets` step refuses a private key that matches neither slot.

- **Proof:** the next cycle's `make demo-load` passes its Snowflake checks.
- **Rollback:** run `replace` again with the old pair from the password
  manager (`--key-dir` pointing at a directory holding it).
- **Don't run `replace` while a cycle is up.** The workers keep the old key
  until `demo-secrets` runs, and the next Snowflake apply moves Snowflake to
  the new one; between those two the pipeline can't sign in. Use the
  stages below instead.

### While a cycle runs (no gap)

You need the cycle's Snowflake endpoint ID; `make demo-up` prints it at
step 3 ("Snowflake admits endpoint vpce-...").

```bash
scripts/rotate-keys.sh snowflake-airflow --stage add    --vpce-id <vpce id>
scripts/rotate-keys.sh snowflake-airflow --stage switch
scripts/rotate-keys.sh snowflake-airflow --stage finish --vpce-id <vpce id>
```

| Stage | Does | Proof | Back out |
| --- | --- | --- | --- |
| `add` | Sets `SNOWFLAKE_ORCHESTRATOR_PUBLIC_KEY_2`, then applies the Snowflake root (you review the plan and type `apply`) | `DESC USER IRDP_AIRFLOW` in Snowsight: `RSA_PUBLIC_KEY_2_FP` is the new fingerprint | `gh variable delete SNOWFLAKE_ORCHESTRATOR_PUBLIC_KEY_2 --env demo-snowflake`, then apply again |
| `switch` | Stores the new private key and passphrase in GitHub, copies them into Secrets Manager with `demo-secrets`, then runs a new batch through the pipeline | `make demo-load` passes. Each task reads the key when it starts, so nothing restarts | Store the old `.p8` and passphrase again (`gh secret set`) and dispatch `demo-secrets`; the old key is still in slot 1 |
| `finish` | Moves the new key into slot 1, deletes the slot 2 Variable, applies, and runs another batch | `DESC USER IRDP_AIRFLOW`: `RSA_PUBLIC_KEY_FP` is the new fingerprint, `RSA_PUBLIC_KEY_2_FP` is null | Forward only: the old public key is gone from Snowflake, so rotate again |

During `finish`, Terraform rewrites both slots in one apply. Snowflake
doesn't document whether there's a moment when neither slot holds the new
key; if there is, a task that signs in then fails and Airflow retries it.

## Snowflake: `IRDP_TERRAFORM`

The bootstrap SQL created this user, so its slots are set in Snowsight, by
a person, as `ACCOUNTADMIN`. CI holds the private key as
`SNOWFLAKE_PRIVATE_KEY`.

```bash
scripts/rotate-keys.sh snowflake-terraform --stage new-key
scripts/rotate-keys.sh snowflake-terraform --stage add
scripts/rotate-keys.sh snowflake-terraform --stage switch
scripts/rotate-keys.sh snowflake-terraform --stage finish
```

| Stage | Does | Proof | Back out |
| --- | --- | --- | --- |
| `add` | Prints the `ALTER USER IRDP_TERRAFORM SET RSA_PUBLIC_KEY_2 = '...'` statement for you to run in Snowsight; use the slot `DESC USER` shows empty | `DESC USER IRDP_TERRAFORM` shows the new fingerprint in that slot | `ALTER USER IRDP_TERRAFORM UNSET RSA_PUBLIC_KEY_2;` |
| `switch` | Stores the new private key and passphrase in GitHub, then dispatches a Snowflake plan and waits for it | The plan run succeeds: planning reads every Snowflake object as `IRDP_TERRAFORM`. Don't execute that plan; pass `--vpce-id` during a cycle so it reads clean | Store the old key again with `gh secret set` |
| `finish` | You unset the old slot in Snowsight; the script runs the plan again | The plan run succeeds | Forward only |

## Entra client secret

```bash
scripts/rotate-keys.sh entra          # between cycles
scripts/rotate-keys.sh entra --live   # a cycle is up: also copy it in and restart the API server
```

The script signs you in to the tenant, lists the app's secrets and notes
their IDs, adds a new secret with `az ad app credential reset --append`,
and pipes it into `gh secret set AIRFLOW_ENTRA_CLIENT_SECRET`. The value
never reaches your screen or a command line. With `--live` it also runs
`demo-secrets` and restarts `airflow-api-server`, which reads the secret
when it starts. Last, it deletes the secrets that existed before
([`az ad app credential`](https://learn.microsoft.com/en-us/cli/azure/ad/app/credential)).

**That's every older secret on the app registration, not only Airflow's.**
The reference's app is used by Airflow alone. If anything else signs in
with this app's secrets, move it to the new secret before you type `done`,
or it stops signing in.

- **Proof:** with `--live`, you sign in to Airflow through the tunnel
  (`docs/demo.md`). Between cycles,
  the next cycle's sign-in.
- **Rollback:** none. Entra shows a secret's value only once and GitHub
  secrets can't be read back, so the old value is gone; if the new one is
  wrong, rotate again. The script deletes the old secrets only after you
  type `done` at its sign-in check.
- `--years 1` sets the expiry. Rotation every 90 days keeps well inside it.
  If your tenant limits secret lifetimes, change it to fit.

## Verified Access client secret

```bash
scripts/rotate-keys.sh demo-access          # between cycles
scripts/rotate-keys.sh demo-access --live   # a cycle is up: also put it on the trust provider
```

The same steps as the [Entra client secret](#entra-client-secret), for the
"IRDP demo access" app (`demo_access.client_id` in
`environments/demo-core/demo.tfvars`),
piped into `gh secret set DEMO_ACCESS_ENTRA_CLIENT_SECRET`. With `--live` it
runs `demo-secrets`, which puts the new secret on the cycle's Verified
Access trust provider (`verified-access.md`).
Nothing restarts: Verified Access uses the secret at each sign-in. The
script stops if the core stack sets no `demo_access`.

The same warning applies: it deletes every older secret on the "IRDP demo
access" app, which Verified Access alone uses.

- **Proof:** with `--live`, sign in at
  `https://airflow.demo.infinitechplatforms.com` in a private window. Between
  cycles, the next cycle's sign-in.
- **Rollback:** none, as for Airflow's secret; rotate again.

## Airflow's own secrets

Terraform generates the database password, the Redis password and broker
URL, the Fernet key, and the API and JWT secrets, and writes them
write-only, so they never reach state
(`docs/airflow.md`). A demo cycle creates new ones
every time. In a long-lived deployment:

1. In `environments/demo-platform/main.tf`,
   add `secrets_revision = 2` (or one more than the current value) to
   `module "airflow"`, in a pull request.
2. Apply the platform stack (`apply-demo.yml`, `stack=platform`).
3. Restart everything in the namespace so it reads the new values; Redis
   is a StatefulSet, the rest are Deployments:

   ```bash
   kubectl -n airflow rollout restart statefulset
   kubectl -n airflow rollout restart deployment
   ```

The Fernet key encrypts connections and variables stored in the metadata
database, and a new key can't read values the old one encrypted. IRDP keeps
none there: its one connection, `aws_default`, is an environment variable.
If you add some, rotate the Fernet key Airflow's way instead: set
`fernet_key` to `new,old`, run `airflow rotate-fernet-key`, then set it to
`new` ([Airflow Fernet docs](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/fernet.html)).
The Grafana admin password works the same way through the observability
module's `secrets_revision`.

## Project D token

Not in use: `PROJECT_D_EVIDENCE_ENDPOINT` is unset, so `make post-oscal`
posts nothing (`docs/oscal.md`). No workflow reads the
token yet. Once Project D exists and a workflow posts to it, issue a new
token in Project D, store it with
`gh secret set PROJECT_D_EVIDENCE_TOKEN`, run `make post-oscal` once to
prove it, then revoke the old token in Project D. OIDC to Project D is the
preferred path and would remove the token.

## OIDC trust review

Quarterly, and whenever an environment is added, read each role's trust
policy and check that every `sub` it accepts is one you expect:

```bash
aws iam get-role --role-name irdp-demo-airflow-secrets-writer --query Role.AssumeRolePolicyDocument
aws iam list-roles --query "Roles[?starts_with(RoleName, 'github-ci')].RoleName"
```

The IMCLZ roles (`github-ci-*`) are reviewed in IMCLZ; the subjects they
accept are listed in `docs/oidc-trust-setup.md`.
