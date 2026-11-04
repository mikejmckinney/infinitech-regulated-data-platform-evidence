# Design decisions

Why the reference build uses what it uses, and when Infinitech would choose
something else for a customer. Each default was settled by measurements in
the demo cycles, documented service limits, or both. Prices are US list
prices read in late October 2026 for a typical customer's volume; a quote
for your workload comes from your own profile.

## Ingest: Firehose for messages, plain S3 for files

| If your sources | We use | Why |
| --- | --- | --- |
| Send files: extracts, X12, FHIR Bulk NDJSON, CSV | Files written straight to S3 | Nothing to run, under $1 a month at every volume we priced |
| Send messages: HL7 v2 through an interface engine, application events, API posts | Amazon Data Firehose | It batches events into files and retries when S3 is unavailable. About $34 a month for the typical customer, against about $477 for Amazon MSK. Every demo batch since cycle 5 has gone through it |
| Can only push SFTP | AWS Transfer Family in front of S3 | About $220 to $231 a month, almost all of it the endpoint |
| Already publish to Kafka | A pipeline step that reads your Kafka cluster | Adds no infrastructure to yours; MSK when Infinitech hosts the topics |

## Orchestration: Step Functions with Fargate tasks

For AWS-only buyers whose steps are short and who don't need the Airflow
interface. Demo cycle 5 met all four of the decision's rules: 24 of 24 runs
succeeded, the data checks matched, a stopped task retried and a failed run
redrove, and the average task billed 60.8 seconds. About $60 a month for the
typical customer.

Airflow on EKS stays supported for buyers who need Airflow in GovCloud,
where Amazon MWAA isn't offered. It ran the pipeline in demo cycles 3, 4, 6,
and 14.

## Warehouse: Redshift Serverless

The reference warehouse since 2026-10-31, for buyers whose disaster recovery
stays in AWS. It needs no separate vendor account and carries no government
surcharge.

- **Measured:** a day of the typical customer's loads cost $3.60 on Redshift
  at 4 RPU against $11.27 on Snowflake X-Small. On a TPC-H-derived benchmark
  at 100 GB, Redshift answered the 22 queries in 138.2 to 143.0 seconds,
  Snowflake in 294.3 to 317.2 seconds. Snowflake loaded a 10 GB bulk file
  faster: 67.0 seconds against 297.8.
- **Priced:** about $3,500 a month for the typical customer, against $9,000
  for Snowflake, and $12,700 for Snowflake in government regions.

Snowflake Business Critical is the choice when disaster recovery must cross
to Azure Government, because it is the one option that runs as the same
product on both and replicates between them. Its cross-cloud path has no RPO
SLA.

## Recovery: snapshot copies by default, warm standby on request

Copied snapshots carry the whole database, including what people build in
the warehouse, and cost the least. Drills restored them in 7.3 to 8.2
minutes. Their recovery point in production is the newest recovery point
(Redshift Serverless takes one about every 30 minutes) plus the copy's trip
to the second region, which took 3.6 to 25.6 minutes in the drills. AWS
states no upper bound for that trip.

The warm standby recovered in 1.8 to 2.7 minutes with nothing lost in every
drill. It holds the pipeline's tables only, so tables people build by hand
need their own copy.

## Monitoring: CloudWatch dashboards and alarms

Step Functions, Firehose, and Redshift publish their metrics to CloudWatch
at no charge, and two dashboards and two alarms fit the free tier in
commercial regions. CloudWatch is in both GovCloud regions, without
dashboard sharing, so people sign in through IAM Identity Center and each
role sees the dashboards it names. Nothing to run or patch.

## Changes: a policy gate and a two-step apply

Plans are checked by Open Policy Agent rules before anything changes, on
pull requests and again before each apply. The apply runs only the plan a
person reviewed, matched by checksum. The rules a customer gets start from
the four in the reference build and grow with the customer's own controls.
