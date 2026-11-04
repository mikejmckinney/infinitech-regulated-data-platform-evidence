# Runbook and handoff

The customer receives a runbook for the procedures their deployment uses, a
catalog of failure modes, and a handoff checklist. This page shows how they
are organized; the procedures themselves come with a deployment.

## Runbook

- **Procedures**, each with a script that prints its steps before it acts
  and a `--dry-run` mode: failing over to a second site, rotating keys,
  scaling the orchestrator, reviewing a drill, and a first run in AWS
  GovCloud.
- **Failure modes**, each as a signal, the first command to run, and the
  fix. For example, from the pipeline section:

  > **A pipeline run failed.** *Signal:* the run-failed alarm. *First
  > command:* print the latest run's task states and each task's last log
  > lines. *Fix:* depends on the task. Rerunning is safe, because the load
  > merges by event, so no event is inserted twice.

- **A changelog** of what each demo cycle found and how the runbook changed.

## Handoff checklist

The handoff runs in eight steps, each closed by evidence the customer keeps:

1. Agree on what is handed over, and its state.
2. Transfer ownership of the accounts' resources and the repository.
3. Rotate everything Infinitech could read.
4. Remove Infinitech's access.
5. Run each procedure in a sandbox.
6. Hand over the evidence.
7. Record the open items.
8. Sign off.
