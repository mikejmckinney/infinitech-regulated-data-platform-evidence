# Failure modes

> 3 of the 19 entries in `runbook/failure-modes.md`, the runbook's failure-mode catalog, copied from the IRDP product. [What was changed](README.md#what-was-changed).

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
reviews will correct them (changelog).

## Pipeline and data

### A pipeline run failed

- **Signal:** `AirflowDagRunFailed`.
- **First command:** `scripts/pipeline-run-logs.sh --logs-bucket <bucket>`
  prints the latest run's task states and each task's last log lines from
  the lake's `logs` bucket (`--run <run id>` for another run); the Airflow UI
  shows the same logs. The `demo-ops` load action prints them when its run
  or checks fail.
- **Fix:** depends on the task. Rerunning is safe: `MERGE` inserts no event
  twice (`docs/demo-pipeline.md`).
  `land` failing with "Unknown topic or partition" means the topic is
  missing. The platform apply creates it before Airflow starts; if that Job
  failed, `kubectl -n airflow logs job/irdp-create-topic` says why, and the
  next platform apply runs it again.

## Demo cycle

### A Helm release never finished installing

- **Signal:** the next platform plan wants to create a `helm_release`
  again after an apply failed on it, with "Error checking release
  existence" (the runner lost the cluster mid-install) or "Helm release
  error". A new install then refuses the name ("cannot re-use a name that
  is still in use"); Helm's `replace` reuses only uninstalled or failed
  releases, and not Jobs.
- **Cause:** revision 1 is still `pending-install` or `failed`, and the
  cut-off install's Jobs are still there. A new install adopts the other
  objects but can't change a Job's pod template ("field is immutable").
  Seen on 2026-10-26: first the CI tunnel stopped passing traffic about 20
  minutes in, then the re-install hit the leftover migration Job.
- **First command**, with the tunnel open: `scripts/recover-helm-release.sh airflow --dry-run`,
  then run it without `--dry-run`. Or, with no tunnel of your own:
  `gh workflow run demo-ops.yml -f action=recover -f release=airflow -f confirmation=recover/demo-ops/demo`.
- **Fix:** the script stops, changing nothing, if the release ever
  deployed; that needs `helm rollback`. Otherwise it deletes the
  unfinished revisions and the release's Jobs. Fix why the pods didn't
  start first ("Pods won't start"), then prepare and execute the platform
  stack again.

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
