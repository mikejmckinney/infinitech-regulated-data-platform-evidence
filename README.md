# Infinitech Regulated Data Platform: reference delivery and evidence

This repository holds the evidence behind the Infinitech Regulated Data
Platform (IRDP) videos: the run records, policy results, and design decisions
from a live demo cycle on AWS. It lets you check the walkthrough's numbers
against the records they came from.

It doesn't deploy anything. The Terraform modules, policy source, pipeline
code, and delivery automation stay in Infinitech's private product and come
with a customer engagement.

## The 15-minute evidence path

1. Watch the [hero](https://media.infinitechplatforms.com/irdp/v1/irdp-hero.mp4)
   and the [walkthrough](https://media.infinitechplatforms.com/irdp/v2/irdp-walkthrough.mp4).
   [`media/`](media/) has their transcripts and captions.
2. Read the [architecture](docs/architecture.md) and the
   [design decisions](docs/design-decisions.md).
3. Match the walkthrough's numbers to the records in the table below.
4. Compare the [passing](examples/policy-results/passing.json) and
   [rejected](examples/policy-results/intentional-rejection.json) policy
   results for a private and a public Redshift workgroup.
5. Read [customer adaptation](docs/customer-adaptation.md) for what a
   deployment in your own accounts involves.

## What the walkthrough shows

<table>
  <tr>
    <td width="50%" valign="top">
      <img src="https://media.infinitechplatforms.com/irdp/v1/gallery/irdp-policy-gate.webp" alt="A Terraform plan for a public Redshift workgroup, rejected by the policy gate" width="100%">
      <h3>The policy gate</h3>
      <p>A pull request makes the warehouse public. Terraform plans it; the gate's no-public-resources rule fails it.</p>
    </td>
    <td width="50%" valign="top">
      <img src="https://media.infinitechplatforms.com/irdp/v1/gallery/irdp-pipeline.webp" alt="A Step Functions execution graph: Curate, Load, Verify" width="100%">
      <h3>The pipeline</h3>
      <p>Every ten minutes, Step Functions runs Curate, Load, and Verify, each as a short-lived Fargate task.</p>
    </td>
  </tr>
  <tr>
    <td width="50%" valign="top">
      <img src="https://media.infinitechplatforms.com/irdp/v1/gallery/irdp-read-only.webp" alt="Query editor v2 refusing a DELETE with permission denied" width="100%">
      <h3>Read-only access</h3>
      <p>A read-only role can query the events. When it tries to delete them, Redshift refuses.</p>
    </td>
    <td width="50%" valign="top">
      <img src="https://media.infinitechplatforms.com/irdp/v1/gallery/irdp-redrive.webp" alt="A Step Functions execution redriven from its failed step" width="100%">
      <h3>Failure and redrive</h3>
      <p>The drill stops Curate three times. The run fails, then succeeds when redriven from the failed step.</p>
    </td>
  </tr>
  <tr>
    <td width="50%" valign="top">
      <img src="https://media.infinitechplatforms.com/irdp/v1/gallery/irdp-failover.webp" alt="The failover drill's run summary with both recovery times" width="100%">
      <h3>Failover drill</h3>
      <p>The warm standby recovered in 2.1 minutes with nothing lost; the snapshot restore took 8.2.</p>
    </td>
    <td width="50%" valign="top">
      <img src="https://media.infinitechplatforms.com/irdp/v1/gallery/irdp-teardown.webp" alt="The zero-resource report showing nothing billable left" width="100%">
      <h3>Teardown</h3>
      <p>Every stack destroyed, then a report that checks both regions for anything still billing.</p>
    </td>
  </tr>
</table>

## The walkthrough's numbers and where they come from

All but the last row are from demo cycle 15, 2026-11-03 to 11-04, in a
commercial AWS demo account. [`evidence/results.md`](evidence/results.md)
has the earlier cycles. The walkthrough's "Deletion and erasure" and
"Alerting" chapters were recorded on cycle 16, 2026-11-06 to 11-07; this
package doesn't hold that cycle's records yet.

| Claim in the walkthrough | Record |
| --- | --- |
| 50,000 events sent, 500 malformed; 49,500 loaded, one row each; all seven checks passed | [`evidence/results.md`](evidence/results.md#cycle-15) |
| A forced failure: three stopped attempts, then a redrive to green in 6 min 42 s | [`evidence/results.md`](evidence/results.md#cycle-15), and the drilled run in [`evidence/runs/pipeline-readout.txt`](evidence/runs/pipeline-readout.txt) |
| Warm standby: all 99,000 events in us-west-2 2.1 minutes after the failure, nothing lost | [`evidence/runs/failover-drill.txt`](evidence/runs/failover-drill.txt) |
| Snapshot restore: 8.2 minutes, a 20.8-minute RPO from the drill snapshot's age | [`evidence/runs/failover-drill.txt`](evidence/runs/failover-drill.txt) |
| 294 resources destroyed; nothing billable left from the demo in either region | [`evidence/results.md`](evidence/results.md#cycle-15) for the destroys, [`evidence/runs/zero-resource-report.txt`](evidence/runs/zero-resource-report.txt) for the report |
| A public workgroup fails the policy gate | [`examples/policy-results/intentional-rejection.json`](examples/policy-results/intentional-rejection.json), produced by the same rules the gate runs |

## What's here

| Path | Contents |
| --- | --- |
| `docs/` | Architecture, design decisions, and what a customer deployment adds |
| `evidence/results.md` | Measured results across demo cycles: loads, failover drills, queries |
| `evidence/runs/` | Cycle 15's run records as the workflows wrote them, account ID redacted |
| `examples/policy-results/` | Policy gate output for a passing and a rejected plan |
| `compliance/oscal/` | OSCAL component definitions, mapped to NIST SP 800-53, for the components the walkthrough shows |
| `media/` | Video transcripts and WebVTT captions |
| `runbook-excerpts/` | Pages from the runbook: its front page, the procedures for recovery in the second AWS region and key rotation, three failure modes, and the handoff checklist |
| `evidence/validation.json`, `evidence/publication-boundary.json` | The build's own checks of this package |

## What isn't here

- Terraform modules, environment roots, and saved plans or state
- The Rego policy source and its tests (only the gate's results are here)
- Pipeline code, container images, and SQL
- Workflows, scripts, or credentials that reach AWS
- The cost model and pricing research

The OSCAL documents name files in the private product as evidence. Those
paths don't resolve here.

## Provenance and limits

`PROVENANCE.json` records the private product commit this package was built
from. `SHA256SUMS` covers every file except itself, and this repository's
only workflow checks it.

These are results from demo cycles at demo volume, in a commercial AWS
region. They aren't a compliance attestation, an SLA, or a measurement of
your workload. The account ID is redacted from the run records; it appears
in the videos, where AWS consoles show it.

## Talk to us

Every deployment is fitted to the customer's accounts, identity, network,
regions, and controls. Email
[hello@infinitechplatforms.com](mailto:hello@infinitechplatforms.com) to plan
one.

Use of this repository is governed by the [Infinitech LLC Public Evaluation
Terms](LICENSE).
