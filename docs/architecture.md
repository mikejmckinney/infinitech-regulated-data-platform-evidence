# Architecture

IRDP's reference build on AWS, as demo cycle 15 ran it. Each part below
says what it does and what the walkthrough shows of it.

![IRDP architecture: message sources through Amazon Data Firehose and file sources straight to the S3 lake, the Step Functions pipeline, Redshift Serverless, read-only access and CloudWatch in us-east-1, with the alternatives dashed; the VPC workload subnets that hold the pipeline and Redshift, with each network path labelled on its arrow; dotted lines from the components CloudWatch and Prometheus watch; snapshot copies (the default), S3 replication and a warm standby in us-west-2; and the controls on every change](architecture.svg)

Cards with dashed borders are alternatives a deployment can switch on; the
demo cycle and the walkthrough run the default path.

## Ingest

Sources send records to Amazon Data Firehose. Firehose buffers them for up
to 60 seconds and writes them to the landing bucket, encrypted with a
customer-managed KMS key. There are no servers to run. Sources that already
produce files write to the landing bucket directly; sources that publish to
Kafka are read by an extra pipeline step instead.

## Lake

Three zones, each its own versioned bucket encrypted with a customer-managed
key:

- **Landing** keeps every record as it arrived.
- **Curated** holds the records that passed validation, and the manifest
  each warehouse load used.
- **Quarantine** keeps the records that failed validation, for review.

## Pipeline

A Step Functions state machine starts every 10 minutes and runs each step
as a short-lived Fargate task:

- **Curate** validates new landing files and splits them into curated and
  quarantined records.
- **Load** copies the curated files into Redshift from a manifest, then
  merges them into the tables, so a file loaded twice adds no rows. One load
  runs at a time.
- **Verify** runs data checks against the tables, such as one row per event
  and no extra rows from a replayed file.

Step Functions retries a failed task twice. A run that still fails can be
redriven from the failed step without repeating the steps that succeeded.

## Warehouse

Redshift Serverless at a 4 RPU base, in private subnets, with public access
off and data encrypted with the platform's own KMS key. Compute is billed
while it runs queries and loads. People read the data through a read-only
role, signing in to Query editor v2 through IAM Identity Center; the
database refuses that role's writes.

## Recovery in a second region

Two standbys in us-west-2, measured separately in each drill:

- **Snapshot copies.** Redshift snapshots are copied from us-east-1 to
  us-west-2 and restored there into a new namespace. This is the default.
- **Warm standby.** A second Redshift workgroup in us-west-2, loaded by its
  own pipeline from the curated files that S3 replication copies there. It
  is offered for buyers who need the tables back within minutes.

[`../evidence/results.md`](../evidence/results.md#failover-drills-across-cycles)
has the measured times.

## Monitoring

Two CloudWatch dashboards, one for health and one for operations, show how
runs ended, how fresh the ingested data is, ingest volume, and warehouse
capacity. Two alarms watch the pipeline: one for a failed run, one for no
successful run in a set window. People open the dashboards through IAM
Identity Center, and each role sees only the dashboards it names.

## Governed changes

Every change is Terraform, applied in two steps. A prepare run plans the
change, runs the policy gate against the plan, and saves the plan with
checksums. A person reviews the saved plan. The execute run checks the
checksums and applies that plan, so what was reviewed is what is built.

The policy gate runs on every pull request and before every apply. It
rejects a plan that:

- makes a resource public (a public workgroup, bucket, or security group);
- leaves a data resource without a customer-managed key;
- lacks a valid data classification tag;
- uses a region outside the approved profile (commercial demo regions, or
  GovCloud).

## Teardown

Each demo cycle ends by destroying every stack, each destroy plan read
before it runs, and a zero-resource report that lists anything billable
left in either region.
