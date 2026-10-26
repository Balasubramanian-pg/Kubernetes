# CronJobs

> Phase 2 — Kubernetes Core (highest priority) › Workload primitives

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **CronJob** is a Kubernetes workload resource that creates Jobs on a recurring schedule. It is used when a task needs to run at a specified time or interval rather than continuously.

For example, a data engineering team might use a CronJob to:
- Run a daily ETL process at 2:00 AM.
- Validate data quality every 30 minutes.
- Generate weekly analytics reports.
- Clean up expired records every night.
- Refresh reference data at a scheduled time.
- Execute periodic database maintenance scripts.

A CronJob does not directly run containers. It creates **Jobs**, and each Job creates Pods that execute the workload.

### How a CronJob works

1. You define a CronJob with a schedule, a Job template, and execution policies.
2. The CronJob controller checks the schedule.
3. When a scheduled time arrives, Kubernetes creates a Job.
4. The Job creates one or more Pods.
5. The Pods execute the task and terminate.
6. Kubernetes records the Job's completion or failure.
7. The CronJob controller creates future Jobs according to the schedule.

The resource hierarchy is:

```text
CronJob
   |
   | Scheduled execution
   v
  Job
   |
   | Creates and manages
   v
  Pod
   |
   | Runs
   v
Container / Batch task
```

For a data engineering example, the CronJob could launch a Python container that extracts source data, transforms records, and writes results to a destination. Kubernetes handles scheduling and workload execution, while the application handles the actual data processing.

### Cron schedule syntax

A standard Kubernetes CronJob schedule uses five fields:

```text
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sunday-Saturday)
│ │ │ │ │
* * * * *
```

Examples:

| Schedule | Meaning |
|---|---|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | At the start of every hour |
| `0 2 * * *` | Every day at 2:00 AM |
| `30 6 * * *` | Every day at 6:30 AM |
| `0 9 * * 1` | Every Monday at 9:00 AM |
| `0 0 1 * *` | At midnight on the first day of every month |

Cron expressions are evaluated in the configured time zone. If no time zone is specified, the schedule is interpreted using the Kubernetes controller manager's local time zone, which may not match the application team's intended time zone.

For predictable scheduling, use `spec.timeZone`, such as `Asia/Kolkata`, where supported.

### CronJob vs Job vs Deployment

| Feature | CronJob | Job | Deployment |
|---|---|---|---|
| Purpose | Schedule recurring tasks | Execute finite tasks | Maintain continuously running applications |
| Starts work | According to a schedule | When created or triggered | Maintains desired replicas |
| Creates | Jobs | Pods | ReplicaSets and Pods |
| Typical use | Daily ETL, hourly cleanup | One-time migration | API server, web application |
| Completion | Each scheduled Job can complete | Tracks task completion | Pods are expected to remain available |
| Scheduling control | Cron expression and scheduling policies | Job execution settings | Replica and rollout settings |

A CronJob is effectively a scheduler for Jobs. It does not replace a full workflow orchestrator when a pipeline needs complex dependencies, conditional execution, backfills, or sophisticated retry coordination.

### Important CronJob configuration fields

- `spec.schedule`: Required cron expression defining when Jobs should be created.
- `spec.timeZone`: Optional time zone for schedule interpretation, for example `Asia/Kolkata`.
- `spec.jobTemplate`: Template used to create each Job.
- `spec.concurrencyPolicy`: Controls whether scheduled Jobs may overlap.
  - `Allow`: Permits concurrent Jobs. This is the default.
  - `Forbid`: Skips a scheduled execution if a previous Job is still running.
  - `Replace`: Replaces a currently running Job with the newly scheduled one.
- `spec.suspend`: When `true`, prevents future scheduled Job starts. It does not stop Jobs that are already running.
- `spec.startingDeadlineSeconds`: Limits how late a missed schedule can start and determines when a missed execution is considered too late.
- `spec.successfulJobsHistoryLimit`: Number of successful Jobs retained. Defaults to `3`.
- `spec.failedJobsHistoryLimit`: Number of failed Jobs retained. Defaults to `1`.
- `spec.jobTemplate.spec.backoffLimit`: Retry policy inherited by each created Job.
- `spec.jobTemplate.spec.activeDeadlineSeconds`: Maximum active duration for an individual Job.

### Concurrency policies

Imagine a daily ETL process scheduled at 2:00 AM that normally takes 20 minutes but sometimes takes 90 minutes.

- `Allow`: The next scheduled Job may start while the previous one is still running. This can cause overlapping writes or duplicate processing if the application is not designed for concurrency.
- `Forbid`: A new scheduled execution is skipped while the previous Job is active. This avoids overlap, but the skipped execution is not automatically guaranteed to run later.
- `Replace`: Kubernetes attempts to replace the active Job with the newly scheduled one. The previous execution may be terminated before completing its work.

Choose the policy based on the workload's behavior. For ETL pipelines that write to shared tables, `Forbid` can be a useful starting point, but application-level locking and idempotency may still be necessary.

CronJobs are approximate schedulers, not exact-once execution guarantees. A scheduled Job can occasionally be missed or created more than once, so the application should tolerate retries and repeated execution.

## Key points

- **A CronJob schedules Jobs, not containers directly.** The Job controller manages Pods for each scheduled execution.
- **The schedule uses a cron expression.** Understand minute, hour, day of month, month, and day of week.
- **Each execution creates a separate Job.** Each Job has its own status, logs, retries, and completion state.
- **CronJobs are appropriate for recurring batch workloads.** Examples include ETL, data validation, cleanup, and scheduled reporting.
- **Use `timeZone` explicitly.** This prevents ambiguity when the controller's local time differs from the time zone expected by the application.
- **Concurrency policy matters.** `Allow`, `Forbid`, and `Replace` determine how scheduled executions behave when a previous Job is still running.
- **`Forbid` skips overlapping executions.** It does not queue every skipped execution for automatic replay.
- **`startingDeadlineSeconds` controls late starts.** It is not the execution timeout for the container.
- **Use `activeDeadlineSeconds` for Job execution limits.** Configure this under the Job template when a batch task must not run indefinitely.
- **Suspending a CronJob affects future scheduling.** Existing Jobs continue unless you separately stop or delete them.
- **Job history limits are not the same as log retention.** Removing old Jobs and Pods does not provide durable log storage. Use centralized logging when logs must be retained.
- **CronJob execution is not guaranteed to happen exactly once.** Make recurring tasks idempotent and protect shared resources against duplicate or overlapping runs.
- **CronJobs are not ideal for complex data orchestration.** Dependencies, backfills, conditional branching, and pipeline-wide observability may require a dedicated orchestrator.

## Commands / manifests

### 1. Create a simple CronJob

Save this manifest as `cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
spec:
  schedule: "*/5 * * * *"
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 120
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 60
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: task
              image: busybox:1.36
              command:
                - sh
                - -c
                - 'echo "Task started at $(date)"; sleep 10; echo "Task completed"'
```

Apply the manifest:

```bash
kubectl apply -f cronjob.yaml
```

What this configuration does:

- `schedule: "*/5 * * * *"`: Schedules execution every five minutes.
- `timeZone: "Asia/Kolkata"`: Interprets the schedule in Indian Standard Time.
- `concurrencyPolicy: Forbid`: Prevents overlapping scheduled Jobs.
- `startingDeadlineSeconds: 120`: Limits how late a missed scheduled execution may start.
- `successfulJobsHistoryLimit: 3`: Retains up to three successful Jobs in the CronJob's history.
- `failedJobsHistoryLimit: 2`: Retains up to two failed Jobs in the CronJob's history.
- `backoffLimit: 2`: Allows up to two counted Pod failures before an individual Job is marked failed.
- `activeDeadlineSeconds: 60`: Limits the active duration of an individual Job to 60 seconds.
- `restartPolicy: Never`: Allows failed execution attempts to be handled through replacement Pods rather than container restarts inside the same Pod.

`timeZone` is supported by modern Kubernetes releases. If the cluster rejects this field, check the cluster version and its API support rather than silently assuming the schedule uses IST.

### 2. Inspect the CronJob

```bash
kubectl get cronjobs
kubectl describe cronjob hello-cronjob
kubectl get cronjob hello-cronjob -o yaml
```

These commands help you inspect:
- The configured schedule.
- The last scheduled execution time.
- Whether the CronJob is suspended.
- Active Jobs.
- Recent events and scheduling errors.
- The Job template and concurrency policy.

### 3. Watch Jobs created by the CronJob

```bash
kubectl get jobs -w
```

List Jobs and Pods:

```bash
kubectl get jobs
kubectl get pods
kubectl get pods -o wide
```

A CronJob creates Jobs only when a scheduled execution is due. After applying the manifest, you may need to wait until the next five-minute boundary before the first Job appears.

### 4. Inspect logs from a scheduled execution

List the Jobs:

```bash
kubectl get jobs
```

List Pods associated with a particular Job:

```bash
kubectl get pods -l job-name=<job-name>
```

Read the logs:

```bash
kubectl logs job/<job-name>
```

Replace `<job-name>` with the actual Job name created by the CronJob.

If a CronJob has created multiple Jobs, select the correct Job before reading logs. For durable operational monitoring, send logs to a centralized logging system because historical Jobs and their Pods may be deleted.

### 5. Trigger a Job manually

You do not need to wait for the next scheduled time to test a CronJob. You can create a one-off Job from its template:

```bash
kubectl create job manual-test \
  --from=cronjob/hello-cronjob
```

Inspect the execution:

```bash
kubectl get job manual-test
kubectl describe job manual-test
kubectl logs job/manual-test
```

This is useful when testing a script or debugging a failed scheduled task. The manually created Job is independent of the CronJob's future schedule and does not change the CronJob's schedule.

### 6. Suspend and resume scheduled execution

Suspend future scheduled Jobs:

```bash
kubectl patch cronjob hello-cronjob \
  -p '{"spec":{"suspend":true}}'
```

Resume scheduling:

```bash
kubectl patch cronjob hello-cronjob \
  -p '{"spec":{"suspend":false}}'
```

Suspending the CronJob does not terminate Jobs already running. When resuming, Kubernetes may consider eligible missed schedules, subject to scheduling deadlines and other controls. Do not assume that every missed run will be replayed.

### 7. Change the schedule

Change the schedule to run daily at 2:00 AM IST:

```bash
kubectl patch cronjob hello-cronjob \
  -p '{"spec":{"schedule":"0 2 * * *","timeZone":"Asia/Kolkata"}}'
```

Verify the change:

```bash
kubectl get cronjob hello-cronjob -o yaml
kubectl describe cronjob hello-cronjob
```

The new schedule affects future scheduling decisions. It does not retroactively change the start time of Jobs that have already been created.

### 8. Delete a CronJob

```bash
kubectl delete cronjob hello-cronjob
```

Deleting a CronJob normally deletes the Jobs it manages and their dependent Pods through cascading deletion. This can interrupt running work. If a running Job must finish, consider suspending the CronJob first and handling existing Jobs deliberately.

### 9. Useful debugging commands

```bash
kubectl get cronjobs
kubectl describe cronjob <cronjob-name>
kubectl get jobs --sort-by=.metadata.creationTimestamp
kubectl get pods -l job-name=<job-name>
kubectl describe job <job-name>
kubectl logs job/<job-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

Use the CronJob name to inspect scheduling configuration and the generated Job name to investigate an individual execution.

## Common failures & fixes

| Problem | Likely cause | Fix |
|---|---|---|
| No Job is created | The scheduled time has not arrived, the schedule is invalid, or the CronJob is suspended | Run `kubectl describe cronjob <cronjob-name>` and verify the schedule, time zone, and `suspend` field |
| Job runs at the wrong time | The time zone is unspecified or misunderstood | Set `spec.timeZone`, for example `Asia/Kolkata`, and verify cluster support |
| Job is skipped while another runs | `concurrencyPolicy: Forbid` prevents overlap | Reduce execution time, change the policy if safe, or design the workload to handle concurrency |
| Multiple Jobs run simultaneously | `concurrencyPolicy: Allow` permits overlapping executions | Use `Forbid` or application-level locking when overlapping writes are unsafe |
| A previous Job is terminated | `concurrencyPolicy: Replace` replaces an active Job | Use `Forbid` if the previous task must finish before the next starts |
| A scheduled run is missed | The controller was unavailable, the deadline expired, or too many schedules were missed | Check events, controller health, `startingDeadlineSeconds`, and the scheduling configuration |
| Job is created but Pod fails | Application error, invalid command, missing dependency, or bad image | Inspect `kubectl describe job`, Pod events, and container logs |
| `ImagePullBackOff` | Image reference, registry access, or credentials are incorrect | Validate the image name and registry credentials |
| Job keeps retrying | The task exits unsuccessfully and retries remain available | Check the exit code and logs; configure an appropriate `backoffLimit` |
| Job runs longer than expected | The process hangs or depends on a slow external system | Add application timeouts and configure `activeDeadlineSeconds` |
| Duplicate data appears | A task was retried or scheduled more than once | Use idempotent writes, deduplication, transactions, or execution identifiers |
| Old Jobs disappear | Job history cleanup or TTL cleanup removes completed resources | Configure suitable history limits and use centralized log storage |
| CronJob stays suspended | `spec.suspend` is `true` | Set it to `false` after confirming it is safe to resume |
| Too many missed schedules are reported | The controller has accumulated missed executions or the schedule/deadline configuration is unsuitable | Inspect the deadline and scheduling history; avoid relying on CronJobs for unlimited backfill |
| The schedule is rejected | The cron expression or time zone is invalid or unsupported | Validate the five cron fields and check Kubernetes version support |

### Practical debugging sequence

When a scheduled workload does not run:

1. Check the CronJob's schedule, time zone, and suspension status.
2. Inspect its events and last schedule time.
3. Determine whether a Job was created for the expected execution.
4. If a Job exists, inspect its Pods and logs.
5. Check retry limits, deadlines, and concurrency policy.
6. Verify that the application can safely retry and that its output is correct.

For production data pipelines, track more than Kubernetes status. Record the intended processing window, execution identifier, source and destination counts, and final validation results. A Kubernetes Job marked `Complete` only means its completion criteria were satisfied, not necessarily that the business data is correct.

## References

- [Kubernetes documentation: CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Kubernetes documentation: Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes documentation: TTL-after-finished controller](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
- [Kubernetes documentation: kubectl commands](https://kubernetes.io/docs/reference/kubectl/)
