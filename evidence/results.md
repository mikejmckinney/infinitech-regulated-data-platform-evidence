# Measured results

Each demo cycle stands the platform up in a commercial AWS demo account,
runs it, and tears it down. The numbers below come from GitHub Actions runs
and readouts in Infinitech's private product repository; run IDs are given
so Infinitech can show you the run itself on request.

All results are at demo volume unless a row says otherwise. Recovery times
are from drills, not from production incidents.

## Cycle 15

2026-11-03 to 11-04. Step Functions with Fargate tasks, Firehose ingest,
Redshift Serverless at 4 RPU in us-east-1, standbys in us-west-2. This is the
cycle the videos were recorded on.

| What | Result | Source |
| --- | --- | --- |
| Deploy | The prepare run planned 134 core resources and passed the policy gate; the execute run applied the saved plan after checking its checksums | Runs 41730443068 (prepare) and 41730613489 (execute) |
| Ingest | Two loads of 50,000 generated events each, 500 of them malformed on purpose, through Firehose | Load run 41732801220 for the first |
| Pipeline | 49,500 events in Redshift after the first load, one row each; all seven data checks passed after each load. The run took about four minutes | Load run 41732801220 |
| Scheduled runs | 19 runs, 18 succeeded and one still running when the readout was taken | [`runs/pipeline-readout.txt`](runs/pipeline-readout.txt) |
| Forced failure | Curate stopped three times on the 23:30 run; Step Functions retried twice and failed the run; a redrive from the failed step succeeded. 6 min 42 s from start to finish | Drill run 41739293671; the drilled execution in [`runs/pipeline-readout.txt`](runs/pipeline-readout.txt) |
| Alarm | `irdp-demo-sfn-run-failed` went into alarm 25 seconds after the run failed; `irdp-demo-sfn-no-successful-run` stayed OK | CloudWatch alarm history, 23:34:14 UTC |
| Read-only role | A presenter signed in to Query editor v2 through IAM Identity Center. `SELECT` returned the first batch's 49,500 events; `DELETE FROM analytics.events` failed with "permission denied for relation events" | Recorded in the walkthrough |
| Snapshot copy | Arrived in us-west-2 18.6 minutes after the snapshot was taken | Snapshot run 41741392524 |
| Warm standby | Recovered in 2.1 minutes with an RPO of 0: 2 of 2 curated files loaded, 99,000 events against the primary's 99,000 | [`runs/failover-drill.txt`](runs/failover-drill.txt) |
| Snapshot restore | Recovered in 8.2 minutes with an RPO of 20.8 minutes; the restore itself took 5.8 minutes and brought back 99,000 events | [`runs/failover-drill.txt`](runs/failover-drill.txt) |
| Teardown | 294 resources destroyed across three stacks (27, 133, and 134), each destroy plan read before it ran. The zero-resource report found nothing billable in either region; three Redshift recovery points remained, which AWS deletes within a day | [`runs/zero-resource-report.txt`](runs/zero-resource-report.txt), run 41758199785 |

The snapshot restore's RPO is the age of the snapshot the drill took
itself. In production it would be the age of the newest recovery point
(Redshift Serverless takes one about every 30 minutes) plus the time the
copy takes to reach the second region.

## Failover drills across cycles

| Cycle | Events in the warehouse | Warm standby: RTO, RPO | Snapshot restore: RTO, RPO | Snapshot copy took |
| --- | ---: | --- | --- | --- |
| 10 | 4,950 | 2.7 min, 0 | 8.1 min, 7.3 min | 6.3 min |
| 11 | 2,475,000 | 1.8 min, 0 | 7.3 min, 25.1 min | 24.1 min |
| 12 (8 RPU) | 990,000 | 2.0 min, 0 | 7.3 min, 7.4 min | 3.6 min |
| 15 | 99,000 | 2.1 min, 0 | 8.2 min, 20.8 min | 18.6 min |

The copy's time doesn't follow the data's size. In each snapshot drill
the RPO is mostly the time spent waiting for the copy.

Two drills ended red before these, and each was fixed: in cycle 10 the
first drill restored nothing because the primary had taken only one
recovery point, so the drill now copies a snapshot it takes itself. In
cycle 11 AWS refused a recovery point that wasn't yet available, so the
drill now moves to the next newest one. That drill's snapshot copy took
25.6 min, the longest of any drill. It isn't in the table, because AWS
refused that drill's restore.

## Queries

Cycle 10 ran a TPC-H-derived benchmark at scale factor 100: the 22 queries,
one at a time, on Redshift Serverless at 4 RPU and on Snowflake X-Small.
These results are TPC-H-derived and not comparable with published TPC-H
results.

| Round | Redshift at 4 RPU | Snowflake X-Small | Answers matching |
| --- | --- | --- | --- |
| Run 50, round 1 | 143.0 s, $0.10 | 317.2 s, $0.42 | 22 of 22 |
| Run 50, round 2 | 138.2 s, $0.10 | 294.3 s, $0.39 | |

An earlier run, 48, matched 19 of 22: Redshift averaged a `DECIMAL` at the
input's scale. Run 50 cast those columns to `DOUBLE PRECISION` on both
warehouses.

## Loads at the typical customer's volume

Cycle 8 loaded a 10 GB bulk file and a day of the typical customer's events
on both warehouses. The day's loads cost $3.60 on Redshift at 4 RPU against
$11.27 on Snowflake X-Small; Snowflake loaded the 10 GB bulk in 67.0 s
against Redshift's 297.8 s. Redshift became the reference warehouse on
2026-10-31.
