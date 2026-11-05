# Fail over to the Azure warm standby

> Copied from `runbook/01-failover-azure-gov.md` in the IRDP product. [What was changed](README.md#what-was-changed).

Use this when AWS `us-east-1` is down and isn't coming back soon, or for a
DR drill. It makes the copied lake readable in Azure. It doesn't move the
pipeline, Airflow, or Snowflake there, because the reference doesn't build
those on Azure (`docs/azure-gov-dr.md`).

**Status (2026-10-25): can't be completed yet.** Three things are missing:

1. The `irdp-dr-demo` subscription, which is blocked on an Azure
   subscription limit.
2. A Terraform root that applies the DR overlay
   (`modules/azure/azure-gov-overlay`).
   `environments/azure` vends the subscription only.
3. The scheduled lake copy job
   (design).

The commands below are written against the overlay's outputs and the Azure
CLI's documented flags. None has run against a real standby. Run the first
drill as soon as those three exist, then record the results in the
changelog and in the RTO/RPO table below.

## What fails over

| Piece | In Azure at failover | Notes |
| --- | --- | --- |
| Lake `landing` and `curated` | Yes, up to the last completed copy | ADLS Gen2 filesystems of the same names, encrypted with the Key Vault CMK |
| Lake `quarantine` and `logs` | No | Left out of the copy on purpose |
| MSK topics, events not yet in `landing` | No | Lost; producers replay from their source |
| EKS, Airflow, the pipeline | No | Not built on Azure. Nothing processes new data until AWS returns |
| Snowflake | No | Unavailable while its AWS region is down. The production answer is a failover group in another region or cloud (`docs/azure-gov-dr.md`) |
| The failover decision, telling readers | Manual | A person declares the outage |

## RTO and RPO assumptions

Nothing here is measured yet. Expect data loss, and expect hours, not
minutes.

| Scope | RPO | RTO |
| --- | --- | --- |
| Lake reads | Copy interval plus copy duration: one hour planned, set when the copy job is built. `check` below measures the real value | Time to grant readers access: minutes once the standby exists, plus Azure's role-assignment propagation |
| Lake consistency | Copies are per object, not point-in-time; the copy job copies only completed partitions | n/a |
| Streaming | Everything MSK held that wasn't flushed to `landing` | Producers replay after AWS returns |
| Processing (pipeline, Airflow) | n/a | Not available on Azure: unbounded until AWS returns |
| Snowflake, reference | Not covered | Not covered |

## Who does what

| Step | Who | Where |
| --- | --- | --- |
| Declare the outage | Incident commander | Anywhere |
| `check` | Anyone with Storage Blob Data Reader on the lake account | A host the lake account accepts (below) |
| `grant-readers`, `revoke-readers` | A person with Owner or User Access Administrator on the lake account | Workstation, with Entra MFA. Azure Government deployments may require PIV/CAC here (secrets and identity §5) |

The lake account and Key Vault deny public traffic unless the overlay's
`management_ip_rules` lists your address. Run the Azure steps from a host
in the standby VNet, or ask for your egress address to be added for the
duration.

## Before an outage

- The DR root is applied, and you have its `lake_storage_account_id`
  output. The account name is the last part of that ID.
- The copy job has run green for at least a day.
- An Entra security group exists for the people who read the standby, and
  you have its object ID.
- Azure CLI 2.x, signed in to the right cloud: `az cloud set --name
  AzureCloud` for the reference, `AzureUSGovernment` for a government
  deployment.

## Procedure

Print the whole procedure without running anything:

```bash
bash scripts/failover-azure-gov.sh --dry-run
```

1. **Declare.** Record who declared the outage, when (T0, UTC), and what
   failed. Write it in the incident's ticket or postmortem
   (template).
2. **Stop the drills**, if the cluster still answers. A drill mid-outage
   only adds noise:

   ```bash
   platform/chaos/suspend-all.sh
   ```

3. **Measure what the standby has.** For today's ingest date, the newest
   object in each copied filesystem. Add `--aws-account` while AWS still
   answers, and compare the two timestamps; the gap is the RPO you're
   accepting:

   ```bash
   scripts/failover-azure-gov.sh check --account <lake account>
   scripts/failover-azure-gov.sh check --account <lake account> --aws-account <account id> --aws-region us-east-1
   ```

   `PathNotFound` means nothing for that date reached the standby. Run it
   again with `--date` set to the day before.
4. **Open the standby to readers:**

   ```bash
   scripts/failover-azure-gov.sh grant-readers --account <lake account> --group <group object id>
   ```

5. **Tell the readers where the data is.** The DFS endpoint, and the layout
   the pipeline writes, `events/v1/ingest_date=YYYY-MM-DD/partition=P/`:

   ```bash
   az storage account show --name <lake account> --query primaryEndpoints.dfs -o tsv
   ```

6. **Hold.** Producers, the pipeline, and Snowflake stay down until AWS
   returns. Don't write to the standby; nothing copies it back.

## Verify

A member of the group, signed in as themselves, lists the curated data:

```bash
az storage fs file list --file-system curated --account-name <lake account> --auth-mode login --path events/v1 --recursive false -o table
```

Role assignments can take several minutes to apply. A `403
AuthorizationPermissionMismatch` in the first minutes usually means it
hasn't yet.

## Rollback and failback

The reference only reads from the standby, and S3 stays the source of
truth, so failback is short:

1. Once AWS answers, check the lake and the pipeline are healthy (the
   [failure modes](failure-modes.md) list the checks), then let producers
   replay.
2. Take the readers' access away:

   ```bash
   scripts/failover-azure-gov.sh revoke-readers --account <lake account> --group <group object id>
   ```

3. `platform/chaos/suspend-all.sh --resume` once the drill window has
   passed.

**The rollback boundary:** anything written to the standby during the outage
stays there. No job copies Azure back to S3, and the reference grants
readers no write access for that reason. A deployment that lets people
write in Azure during an outage needs a failback copy designed first;
`docs/azure-gov-dr.md` lists failback as
manual and not designed.

## A drill

Run the same steps with a test group and no declared outage, then
`revoke-readers`. Record T0, the RPO `check` measured, and how long each
step took, in the changelog, and replace the assumptions in
the RTO/RPO table above and in
`docs/azure-gov-dr.md`
with what you measured.
