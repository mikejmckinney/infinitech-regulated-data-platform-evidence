# Failure modes

> 3 of the 27 entries in `runbook/failure-modes.md`, the runbook's failure-mode catalog, copied from the IRDP product. [What was changed](README.md#what-was-changed).

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

The Step Functions entries (2026-11-05) cover the default pipeline's
alarms (`dashboards.tf`):
two on runs, and a third on Firehose delivery, which demo cycle 16 fired
with a deny on the delivery role.
The failed-run entry's redrive is what demo cycle 15's forced-failure drill
did. The three after them, also 2026-11-05, were added because the code
guards against them or a cycle showed them: a task that can't start, the
daily RPU-hour limit, and runs over the 300 s target. None of the three has
been worked through live with these commands.

Each Step Functions alarm's description links its entry by the anchor
under its heading, named after the alarm (`#run-failed`,
`#no-successful-run`, `#firehose-not-delivering`), so a renamed heading
keeps the link;
`modules/orchestration/sfn-fargate-pipeline/tests/test_alarms.py` fails if
one goes missing.

## Step Functions pipeline

### A Step Functions run failed

<a id="run-failed"></a>

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

- **Fix:** fix what the log names, then rerun the failed run. How depends
  on whether the fix changed the stack:
  - **Nothing was re-applied** (a grant, an endpoint, a limit, a source
    file): redrive the run from the step that failed:
    `aws stepfunctions redrive-execution --execution-arn <execution arn>`.
    A redrive reruns the failed step and the ones after it, with the run's
    original input, its scheduled time; the steps that succeeded don't run
    again, and each rerun step gets its retries back. AWS allows it within
    14 days of the run's end
    ([redrive](https://docs.aws.amazon.com/step-functions/latest/dg/redrive-executions.html)).
  - **The sfn stack was re-applied** (a new image, task definition or
    state machine): don't redrive. A redrive runs the failed run's own
    state machine definition, and that definition names the exact task
    definition revision, so it reruns the old image and fails the same
    way. Start a new run with the failed run's input instead, and check
    that it succeeds:

    ```bash
    input=$(aws stepfunctions describe-execution --execution-arn <execution arn> --query input --output text)
    aws stepfunctions start-execution --state-machine-arn "$machine" --input "$input"
    ```

  Rerunning is safe either way: `MERGE` inserts no event twice
  (`docs/demo-pipeline.md`).
  In demo cycle 15, `DRILL=redrive make demo-chaos` stopped Curate on all
  three attempts of one run: the run failed, `irdp-demo-sfn-run-failed`
  went into alarm, and the run succeeded on its redrive
  (`docs/demo-results.md`).

### No successful run in 30 minutes

<a id="no-successful-run"></a>

- **Signal:** `irdp-demo-sfn-no-successful-run`: no run succeeded in three
  10-minute periods in a row, so the last success was 20 to 40 minutes
  ago. A period without a success counts as 0, so it also fires when no
  run starts at all. Periods from before the stack was applied count as
  successes: after `make demo-up` it should stay OK while the first run is
  due, and fire if no run has succeeded 20 to 40 minutes after the apply.
  Cycle 17 checks both timings; until then an ALARM minutes after
  `make demo-up` that clears after the first run is the known issue 236,
  not an outage.
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

### A step's task stopped before its container ran

- **Signal:** `irdp-demo-sfn-run-failed`, and the `logs` action prints
  "no log stream" for the failed attempts: the container never started, so
  it wrote nothing. The run's cause, the stopped ECS task as JSON, says why
  in `StoppedReason`.
- **First command:** the reason of the latest failed run, with `$machine`
  found as in [A Step Functions run failed](#a-step-functions-run-failed):

  ```bash
  run=$(aws stepfunctions list-executions --state-machine-arn "$machine" --status-filter FAILED \
    --max-items 1 --query 'executions[0].executionArn' --output text)
  aws stepfunctions describe-execution --execution-arn "$run" --query cause --output text | jq -r .StoppedReason
  ```

- **Fix**, by the reason
  ([CannotPullContainer errors](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_cannot_pull_image.html),
  [ResourceInitializationError](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/resource-initialization-error.html)):
  - `CannotPullContainerError` with "failed to resolve ref" or "not
    found": the image tag the task definition names isn't in ECR. Compare
    the two:

    ```bash
    taskdef=$(aws stepfunctions describe-state-machine-for-execution --execution-arn "$run" \
      --query definition --output text | jq -r '[.States[].Parameters.TaskDefinition // empty][0]')
    aws ecs describe-task-definition --task-definition "$taskdef" --query 'taskDefinition.containerDefinitions[0].image'
    aws ecr describe-images --repository-name irdp/pipeline-task --query 'imageDetails[].imageTags[]'
    ```

    The first command reads the task definition revision the failed run
    used, from the definition it ran; the family's latest revision can be a
    newer one.

    `irdp/pipeline-task` keeps its 10 newest images
    (`ecr.tf`). ECR counts them by push
    time, not by whether a task definition still names one, and expires
    the rest within 24 hours
    ([lifecycle policies](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)).
    A stack applied at an older commit loses its image after ten more
    pushes. Apply the sfn stack at a commit whose image is there;
    `apply-demo.yml` checks that before it applies, and
    `build-pipeline-task-image` with `push=true` pushes one. In the standby
    region the image is ECR's replica of the push, so a new push shows up
    there a little later.
  - "There is a connection issue between the task and Amazon ECR", or a
    `ResourceInitializationError` about registry auth or the log group: the
    task can't reach ECR or CloudWatch Logs. Its subnets have no route to
    the internet; it reaches them through the core's `ecr.api`, `ecr.dkr`
    and `logs` interface endpoints, and the S3 gateway endpoint, since ECR
    serves image layers from S3
    (`endpoints.tf`).
    Check they still exist and that the workload security group allows
    HTTPS to them.

  Then rerun the failed runs
  ([A Step Functions run failed](#a-step-functions-run-failed)): start new
  runs after applying the sfn stack at a newer image, since a redrive would
  pull the expired tag again; redrive when the fix left the sfn stack as it
  was, such as an endpoint put back or the same image tag pushed again.
