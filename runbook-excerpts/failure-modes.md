# Failure modes

> 3 of the 21 entries in `runbook/failure-modes.md`, the runbook's failure-mode catalog, copied from the IRDP product. [What was changed](README.md#what-was-changed).

What IRDP failures look like, the first command to run, and what fixes
them. Each entry links the procedure that goes further. The Airflow entries
are the situations in
Airflow's runbook hooks, which
the alerts' `runbook` annotations and the chaos drills point at.

The cluster commands need `KUBECONFIG` from `scripts/eks-tunnel.sh start`
(`docs/demo.md`).

**Status (2026-10-26):** compiled from the design and offline tests. Four
entries, "Pods won't start", "A Helm release never finished installing",
"A pipeline run failed", and "The Chaos Mesh controller keeps restarting",
come from a live demo cycle; the rest haven't been seen live yet, and drill
reviews will correct them.

The Step Functions entries (2026-11-05) cover the default pipeline's two
alarms (`dashboards.tf`).
The failed-run entry's redrive is what demo cycle 15's forced-failure drill
did.

## Step Functions pipeline

### A Step Functions run failed

- **Signal:** `irdp-demo-sfn-run-failed`: a run of `irdp-demo-sfn-pipeline`
  failed in the last 10 minutes. Each step is tried three times, a minute
  apart, before a run fails
  (`state-machine.tf`),
  so the failure repeated.
- **First command:**
  `gh workflow run demo-sfn-ops.yml -f action=logs -f confirmation=logs/demo-sfn-ops/demo`
  prints the last 40 log lines of each failed attempt in the latest three
  failed runs (`scripts/sfn_runs.py` `logs`). A
  failed run's own cause says only that the container exited; its log
  says why. To find the run itself:

  ```bash
  machine=$(aws stepfunctions list-state-machines --query "stateMachines[?name=='irdp-demo-sfn-pipeline'].stateMachineArn" --output text)
  aws stepfunctions list-executions --state-machine-arn "$machine" --status-filter FAILED --max-items 3
  ```

- **Fix:** fix what the log names, then redrive the run from the step that
  failed:
  `aws stepfunctions redrive-execution --execution-arn <execution arn>`.
  A redrive reruns the failed step and the ones after it, with the run's
  original input, its scheduled time; the steps that succeeded don't run
  again, and each rerun step gets its retries back. AWS allows it within
  14 days of the run's end
  ([redrive](https://docs.aws.amazon.com/step-functions/latest/dg/redrive-executions.html)).
  Rerunning is safe: `MERGE` inserts no event twice
  (`docs/demo-pipeline.md`).
  In demo cycle 15, `DRILL=redrive make demo-chaos` stopped Curate on all
  three attempts of one run: the run failed, `irdp-demo-sfn-run-failed`
  went into alarm, and the run succeeded on its redrive
  (`docs/demo-results.md`).

### No successful run in 30 minutes

- **Signal:** `irdp-demo-sfn-no-successful-run`: no run succeeded in three
  10-minute periods. Missing data counts as breaching, so it also fires
  when no run starts at all. Right after `make demo-up` it's in alarm until
  the first run succeeds; that's expected.
- **First command:** the schedule's state, then the latest runs, with
  `$machine` found as in the entry above:

  ```bash
  aws scheduler get-schedule --name irdp-demo-sfn-pipeline --query State
  aws stepfunctions list-executions --state-machine-arn "$machine" --max-items 5
  ```

- **Fix**, by what they show:
  - Runs failing: [A Step Functions run failed](#a-step-functions-run-failed).
  - The schedule is `DISABLED`: the failover drill turns it off while it
    measures and back on before it exits, even after an error
    (`scripts/failover_drill.py`), so a
    drill whose runner was cut off can leave it off. Apply the sfn stack
    (`apply-demo.yml`, `stack=sfn`) to put back the committed schedule.
  - The schedule is `ENABLED` and no run starts: Scheduler doesn't retry a
    start that fails (`maximum_retry_attempts = 0`). Check that the
    `irdp-demo-sfn-scheduler` role still allows `states:StartExecution`
    on the state machine.
  - A run stays `RUNNING`: each step's attempt times out after 480 s
    (`step_timeout_seconds`) and is retried, so it ends as a failed run;
    then see above.

## Keys

### A KMS key is disabled or pending deletion

- **Signal:** EKS reports the cluster degraded; S3, MSK, or Secrets
  Manager calls fail with KMS invalid-state errors.
- **First command:**
  `aws kms describe-key --key-id alias/irdp-confidential --query 'KeyMetadata.[KeyId,KeyState,DeletionDate]'`.
- **Fix:** `aws kms cancel-key-deletion --key-id <key id>`, then
  `aws kms enable-key --key-id <key id>`; a cancelled deletion leaves the
  key disabled. Re-enable an EKS cluster's key within 30 days, or the
  cluster may not recover; a deleted key can't be recovered at all
  ([EKS envelope encryption](https://docs.aws.amazon.com/eks/latest/userguide/envelope-encryption.html)).
