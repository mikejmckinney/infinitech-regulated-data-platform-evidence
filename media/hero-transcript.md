# IRDP hero transcript

## Opening

This is a regulated data platform on AWS, recorded live.

## Policy gate

A change that would make the warehouse public fails the policy gate before it
can merge.

## Pipeline

Records stream in through Firehose, land in an encrypted lake, and load into
Redshift on a Step Functions pipeline.

## Read-only access

A read-only user can query them. When they try to delete, Redshift refuses.

## Failover

We failed it over to a second region. The warm standby held every event two
point one minutes later, with no data lost.

## Teardown

Then we tore it all down, and the report found nothing billable left from the
demo.

## Closing

Infinitech's Regulated Data Platform. Watch the full walkthrough.
