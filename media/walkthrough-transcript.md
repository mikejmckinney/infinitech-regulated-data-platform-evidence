# IRDP walkthrough transcript

## Opening

Infinitech's Regulated Data Platform is a reference implementation for teams
that move sensitive records through AWS. Records arrive through Firehose, land
in an encrypted lake, and load into Redshift Serverless, with backups copied to
a second region. Everything in this walkthrough was recorded live, on one demo
cycle.

## Governed deployment

The platform is Terraform, applied in two steps. A prepare run plans the core
stack, here one hundred thirty-four resources, and runs the policy gate against
that plan. It saves the plan with checksums. A person reads the saved plan
before anything changes. The execute run then checks those checksums and
applies that exact plan, so what was reviewed is what gets built.

## Policy gate

To show the gate stopping a bad change, we opened a pull request that makes the
Redshift workgroup publicly accessible. Terraform plans the change. The policy
gate rejects it: its no-public-resources rule names the workgroup and fails the
check, so the change cannot merge. The same gate requires
customer-managed keys, data classification tags, and approved regions, and it
runs on pull requests and before every apply.

## Ingest

Records arrive through Amazon Data Firehose. This load sent fifty thousand
generated test events, five hundred of them malformed on purpose. Firehose
buffers them for up to sixty seconds, then writes them to the landing bucket.
There are no servers to run, and the stream's error log stayed empty.

## Lake zones

The lake has three zones, each its own bucket. Landing keeps what arrived,
exactly as it arrived. Curated holds the validated records, beside the manifest
each Redshift load used. Quarantine keeps the records that failed validation for
review: here, the five hundred bad events from each batch. Every bucket is
versioned and encrypted with a customer-managed key.

## Pipeline run

Every ten minutes, a Step Functions state machine looks for new data. Each step
is a short-lived Fargate task. Curate validates the records and splits them.
Load copies them into Redshift and merges them, so a replayed file adds no
rows. Verify checks the tables: one row per event, and no extra rows from a
replay. This run took about four minutes, and all seven checks passed on
forty-nine thousand five hundred events.

## Locked-down warehouse

The warehouse is Redshift Serverless at four RPUs, and its compute is billed
only while it runs queries.
It sits in private subnets with public access turned off. The policy gate
checks those settings on every plan.

## Read-only access

People who need to see the data get a read-only role. This presenter signs in
to Query editor through single sign-on, with no database password. The role can
read the analytics tables: the three event types add up to all forty-nine
thousand five hundred events. When the same user tries to delete them, Redshift
refuses with permission denied.

## Failure and redrive

The drill stopped the Curate task three times in a row. Step Functions retried it twice, then marked the run failed.
The run was then redriven from the failed step, without starting over, and it
finished green. Start to finish took six minutes and forty-two seconds, and the
execution's event log keeps every attempt.

## Operations dashboards

Two CloudWatch dashboards cover health and operations: how runs ended, how
fresh the ingested data is, ingest volume, and warehouse capacity. The forced
failure shows as a spike. The run-failed alarm fired less than a minute later,
and the alarm for no successful run stayed quiet, because the redrive
succeeded.

## Recovery copies

For disaster recovery, snapshots of the warehouse are copied from us-east-1 to
us-west-2. This copy arrived about nineteen minutes after the snapshot was taken. A warm standby workgroup waits
in us-west-2 as well, kept loaded from the curated files that replicate there.

## Failover drill

Then we failed over. The drill stopped the primary's pipeline and timed two
paths. The warm standby held all ninety-nine thousand events two point one
minutes after the failure, with no data lost. Restoring the copied snapshot
took eight point two minutes. Its twenty point eight minute data-loss window is
the age of the snapshot the drill restored.

## Teardown

Every demo cycle ends by tearing it all down: two hundred ninety-four resources
across three stacks, each destroy plan read before it ran. A zero-resource
report then checks both regions. It found nothing billable left from the demo.

## Closing

Infinitech stands the platform up in your AWS accounts and hands it over, with
the repository, the runbook, and the evidence from your own drills. Ready for
GovCloud. To plan a deployment, contact Infinitech.
