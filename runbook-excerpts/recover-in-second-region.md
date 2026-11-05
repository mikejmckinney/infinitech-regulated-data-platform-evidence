# Recover in the second AWS region

> Copied from `runbook/06-recover-in-second-region.md` in the IRDP product. [What was changed](README.md#what-was-changed).

Use this when the primary region (`us-east-1` in the reference) is down and
isn't coming back soon. It brings the warehouse back in the standby region
(`us-west-2`). The default path restores the newest backup of the primary
that Redshift Serverless copied there into a namespace kept ready for it.
The warm standby, an option, is a second workgroup that the replicated lake
keeps loaded, so it needs no restore. The demo's standby stack builds both
(`environments/demo-dr`); a deployment
has the default alone or both
(`docs/stack-choices.md`).
Recovery in Azure is a separate procedure, built on request.

**Status (2026-11-05):** drilled in demo cycles 10, 11, 12 and 15 with
`make demo-failover`, which makes the AWS calls below against a primary
whose schedule it stopped, not a region that's down. The snapshot restore
recovered in 7.3 to 8.2 min and the warm standby in 1.8 to 2.7 min, with
every event back on both
(`docs/demo-results.md`).
Nobody has run these steps by hand, and no script runs them yet: the
commands are the ones `scripts/failover_drill.py`
issues.

## What comes back

| Piece | In the standby region | Notes |
| --- | --- | --- |
| Warehouse, snapshot restore (default) | The whole database, as of the newest backup copied before the outage | Restored into `irdp-demo-restore`. Redshift Serverless takes a recovery point about every 30 minutes or 5 GB of changes, and the copy's trip took 3.6 to 25.6 min in the drills; AWS states no upper bound for it ([backup copy](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-backup-copy.html)) |
| Warehouse, warm standby (option) | The pipeline's tables, up to the last curated file S3 replicated | `irdp-demo-dr`, loaded every 10 minutes by the load-only pipeline `irdp-demo-dr-pipeline`. Tables people built by hand in the primary aren't in it |
| Lake: curated files and delete requests | Yes, as replicated | S3 replication with Replication Time Control, 99.9% of new objects within 15 minutes (`replication.tf`). Landing and quarantine aren't replicated |
| Ingest (Firehose), Curate, the primary pipeline | No | They run in the primary region only. New events wait until it returns |
| The decision, telling readers | Manual | A person declares the outage |

## Who does what

| Step | Who | Where |
| --- | --- | --- |
| Declare the outage | Incident commander | Anywhere |
| Everything else | An AWS administrator session in the account | A terminal with the AWS CLI, signed in for the standby region |

The steps call Redshift Serverless, the Redshift Data API, Step Functions
and S3 in the standby region. The `IrdpDemoOperator` permission set can do
none of them. The gated workflows can't run them either: `apply-demo.yml`
and `demo-sfn-ops.yml` read Terraform state from a bucket in the primary
region (`demo.s3.tfbackend`).

## Before an outage

Write these down somewhere you can read without the primary region, for the
same reason:

```bash
terraform -chdir=environments/demo-dr init -input=false -backend-config=demo.s3.tfbackend
terraform -chdir=environments/demo-dr output -json copy             # primary_namespace_id
terraform -chdir=environments/demo-dr output -json restore_target   # workgroup and database names
terraform -chdir=environments/demo-dr output -json standby          # the warm standby's
terraform -chdir=environments/demo-dr output -json pipeline         # the standby pipeline's state machine
terraform -chdir=environments/demo-dr output -json replication      # the primary and replica curated buckets
```

- The primary copies its backups to the standby region. `apply-demo.yml`
  turns that on after the standby stack applies
  (`dr-snapshot-copy.sh`). Check:

  ```bash
  aws redshift-serverless list-snapshot-copy-configurations --region us-east-1 --namespace-name irdp-demo
  ```

  One entry with `destinationRegion` `us-west-2` means it's on.
- Your sign-in works with the primary region down. Find the region your
  IAM Identity Center instance is in; if it's the primary, keep a way in
  that doesn't need it.
- Your readers' roles can reach the standby region's workgroups. The
  reference doesn't set that up.

## Procedure

1. **Declare.** Record who declared the outage, when (T0, UTC), and what
   failed, in the incident's ticket or postmortem.
2. **Find the newest backup** of the primary in the standby region, a
   recovery point or a snapshot. Match on the primary namespace's ID, not
   its name: a name repeats every cycle, and the standby region lists an
   earlier namespace's backups for a day.

   ```bash
   aws redshift-serverless list-recovery-points --region us-west-2 \
     --query "recoveryPoints[?ends_with(namespaceArn, '<primary namespace id>')].[recoveryPointCreateTime, recoveryPointId]" --output table
   aws redshift-serverless list-snapshots --region us-west-2 \
     --query "snapshots[?status=='AVAILABLE' && ends_with(namespaceArn, '<primary namespace id>')].[snapshotCreateTime, snapshotArn]" --output table
   ```

   Take the newest of either list. Its age at T0 is the data you're
   accepting to lose.
3. **Restore it** into the namespace kept for it. The restore replaces
   everything in that namespace:

   ```bash
   aws redshift-serverless restore-from-recovery-point --region us-west-2 --recovery-point-id <recovery point id> --namespace-name irdp-demo-restore --workgroup-name irdp-demo-restore
   aws redshift-serverless restore-from-snapshot --region us-west-2 --snapshot-arn <snapshot arn> --namespace-name irdp-demo-restore --workgroup-name irdp-demo-restore
   ```

   "Recovery point … is not in available state" means AWS listed a copy
   it can't restore from yet. Use the next newest; cycle 11's drill met
   one 5 minutes old.
4. **Wait for the restore.** The namespace leaves `AVAILABLE` and comes
   back to it; the restore itself took 4.9 to 5.8 min in the drills:

   ```bash
   aws redshift-serverless get-namespace --region us-west-2 --namespace-name irdp-demo-restore --query namespace.status
   ```

   AWS says the restore's first phase completes in a few minutes and the
   tuning after it can run for hours or longer
   ([restore](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-snapshot-restore.html)).
5. **Verify it** (below).
6. **The warm standby, if the deployment has one.** Its own schedule keeps
   loading what S3 replicated. Start one more run now, so it loads every
   file that arrived, and wait for it to succeed:

   ```bash
   machine=$(aws stepfunctions list-state-machines --region us-west-2 \
     --query "stateMachines[?name=='irdp-demo-dr-pipeline'].stateMachineArn" --output text)
   aws stepfunctions start-execution --region us-west-2 --state-machine-arn "$machine" \
     --input '{"scheduled_time": "<the time now, UTC, such as 2026-11-05T12:00:00Z>"}'
   aws stepfunctions describe-execution --region us-west-2 --execution-arn <execution arn> --query status
   ```

7. **Tell readers where the data is.** The workgroup's endpoint, database
   `irdp`: `irdp-demo-restore` for the restored copy, `irdp-demo-dr` for
   the warm standby.

   ```bash
   aws redshift-serverless get-workgroup --region us-west-2 --workgroup-name irdp-demo-restore --query workgroup.endpoint.address --output text
   ```

8. **Hold.** Nothing ingests or curates new events until the primary
   returns. Don't load anything into the standby region's warehouses by
   hand: nothing copies them back.

## Verify

Count the events in the restored namespace as its admin user. Read the
secret from the namespace itself, because AWS doesn't document whose admin
credentials a restore keeps:

```bash
aws redshift-serverless get-namespace --region us-west-2 --namespace-name irdp-demo-restore --query namespace.adminPasswordSecretArn --output text
aws redshift-data execute-statement --region us-west-2 --workgroup-name irdp-demo-restore --database irdp --secret-arn <secret arn> --sql "SELECT COUNT(*) FROM analytics.events"
aws redshift-data get-statement-result --region us-west-2 --id <statement id>
```

For the warm standby, run the same count on `irdp-demo-dr` with
`--secret-arn` from its `standby` output. Then read the files it loaded,
`SELECT source_file FROM landing.loaded_files`, and keep their keys:

```bash
aws redshift-data get-statement-result --region us-west-2 --id <statement id> --output json \
  | jq -r '.Records[][0].stringValue' | sort > loaded.txt
```

**The load backlog:** curated files that reached the replica bucket and
aren't loaded yet. Step 6's run should leave none:

```bash
aws s3api list-objects-v2 --region us-west-2 --bucket <replica bucket> --prefix events/v1/ --output json \
  | jq -r '.Contents[].Key' | sort > replicated.txt
comm -23 replicated.txt loaded.txt
```

This can't show what the warm standby lost. A file the primary wrote that
never replicated is in neither list, so while the primary region's bucket
can't be listed, the warm standby's loss is unknown. Replication Time
Control replicates 99.9% of new objects within 15 minutes, so the files
most likely missing are the last ones written before the outage.

**The loss, once the primary's curated bucket can be listed again:** the
comparison the drill makes. Every curated file the primary wrote up to
T0, less the ones the standby loaded, is what it lost; the oldest one's
write time, subtracted from T0, is the RPO. Give T0 in the form the CLI
prints, so the two compare as text:

```bash
aws s3api list-objects-v2 --region us-east-1 --bucket <primary curated bucket> --prefix events/v1/ --output json \
  | jq -r --arg t0 "<T0, such as 2026-11-05T12:00:00+00:00>" '.Contents[] | select(.LastModified <= $t0) | .Key' \
  | sort > primary.txt
comm -23 primary.txt loaded.txt
```

Nothing listed means nothing was lost. The reference keeps no S3 Inventory
report of the curated bucket, so this waits for the primary region.

## Failback

S3 and the primary namespace stay the source of truth, so failback is
short:

1. Once the primary region answers, check that its runs succeed again:
   `irdp-demo-sfn-no-successful-run` back to OK
   ([failure modes](failure-modes.md#no-successful-run-in-30-minutes)).
2. With the warm standby, work out what it lost
   ([Verify](#verify), the loss once the primary's bucket can be listed)
   and record it with T0.
3. Move readers back to `irdp-demo`.
4. `irdp-demo-restore` keeps the restored data, billed as storage, until
   the next restore replaces it or the standby stack is destroyed.

**The failback boundary:** anything written in the standby region during
the outage stays there. No job copies it back to the primary, and a
deployment that lets people write there needs that designed first.

## A drill

```bash
DR=true make demo-up    # ... sfn, then dr
make demo-load
make demo-failover      # snapshots the primary, waits for its copy, then fails over
DR=true make demo-down
```

The drill stops the primary's schedule instead of losing the region, and
takes a snapshot of its own first so a copy newer than the last recovery
point exists. It then makes the calls above on both paths, writes
`artifacts/failover-drill-<run>/`, and deletes its snapshots
(`docs/demo.md`). Add its
results to `docs/demo-results.md` and a line to
the changelog.
