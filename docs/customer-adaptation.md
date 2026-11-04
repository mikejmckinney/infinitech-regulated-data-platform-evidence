# What a deployment in your accounts involves

The reference build ran in Infinitech's commercial demo account. A customer
deployment runs in the customer's own AWS accounts, and Infinitech fits it to
them before handing it over.

## What Infinitech adapts

| Area | What changes for you |
| --- | --- |
| Accounts and regions | Your AWS Organization and accounts, in commercial regions or AWS GovCloud (US), with the second region you choose for recovery |
| Identity | Your IAM Identity Center or identity provider, your groups, and the roles that read data, open dashboards, and approve changes |
| Network | Your VPC layout, private connectivity, and the endpoints your sources and readers use |
| Sources | Firehose for message sources, plain S3 for file sources, SFTP or Kafka where you need them |
| Keys and classification | One customer-managed KMS key per data classification you use, and the tags your controls require |
| Policy gate | The four reference rules plus your own controls, checked on every pull request and before every apply |
| Recovery | Snapshot copies, a warm standby, or both, sized to your recovery time and recovery point objectives, with a drill to measure them |
| Monitoring | Dashboards and alarms for your runs, freshness, volume, and capacity, routed to your on-call |

## What you receive

- The platform running in your accounts, from a repository you own.
- The runbook for the procedures your deployment uses.
- The evidence from drills run in your own accounts: failover, a forced
  pipeline failure and its redrive, and the policy gate rejecting a
  non-compliant change.
- A handoff, step by step, that transfers ownership, rotates every
  credential Infinitech could read, and removes Infinitech's access.

## What it doesn't include

Infinitech doesn't operate the platform after the handoff, and the
reference build has no 24x7 operations. Standing it up doesn't produce an
authorization to operate: ATO sponsorship, a third-party assessor, and
continuous monitoring services are separate.
