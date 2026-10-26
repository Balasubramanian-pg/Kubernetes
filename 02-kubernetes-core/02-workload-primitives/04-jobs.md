# Jobs

> Phase 2 — Kubernetes Core (highest priority) › Workload primitives

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **Job** is a Kubernetes workload resource that runs one or more Pods to completion. Unlike a Deployment, which keeps applications running, a Job manages tasks that are expected to finish successfully.

Jobs are useful for batch processing, one-time scripts, data migration, database initialization, report generation, file processing, and other finite workloads.

For example, a data engineering pipeline might use a Job to:
- Process a batch of files from object storage.
- Run a one-time database migration.
- Execute a data quality validation script.
- Rebuild a reference table.
- Generate a daily report from a fixed dataset.

Once the task succeeds, the Job records the successful completion. Kubernetes does not continuously restart the workload just to keep a completed Pod running.

### How a Job works

1. You create a Job manifest specifying the container image, command, and completion requirements.
2. The Job controller creates Pods to perform the task.
3. The container executes its workload and exits.
4. If the Pod exits successfully, Kubernetes records a successful completion.
5. If the Pod fails, the Job controller may create another Pod and retry, subject to its retry and failure policies.
6. The Job remains as a Kubernetes API object after completion until it is deleted or cleaned up by a configured mechanism.

The Job controller aims to achieve the requested number of successful completions. It does not guarantee that a program executes exactly once. A task can run more than once after failures, so batch processing should ideally be **idempotent**, meaning repeated execution produces the same intended result without duplicating side effects.

### Job vs Deployment

| Feature | Job | Deployment |
|---|---|---|
| Purpose | Run finite tasks to completion | Run continuously available applications |
| Typical workloads | Batch processing, migrations, scripts | APIs, web applications, workers |
| Successful exit | Counts toward Job completion | Pod is replaced to maintain the desired replica count |
| Failure handling | Retries according to Job policies | Replaces failed Pods to maintain replicas |
| Completion tracking | Tracks successful completions | Tracks desired and available replicas |
| Example | Process 10,000 records and exit | Serve HTTP requests continuously |

A Job is not a replacement for a Deployment. Choose based on whether the workload should finish or remain running.

### Important Job configuration fields

- `spec.template`: Pod template describing the container or containers that execute the task.
- `spec.completions`: Number of successful Pod completions required. Defaults to `1` when unspecified.
- `spec.parallelism`: Maximum number of Pods the Job should run concurrently. Defaults to `1`.
- `spec.backoffLimit`: Number of counted Pod failures before the Job is considered failed. Defaults to `6`.
- `spec.activeDeadlineSeconds`: Maximum duration for the Job to remain active before Kubernetes terminates it and marks it failed.
- `spec.ttlSecondsAfterFinished`: Optional cleanup delay before a finished Job can be automatically deleted.
- `spec.template.spec.restartPolicy`: Must be `Never` or `OnFailure` for a Job. It cannot be `Always`.

### Completions and parallelism

Suppose a Job must process 10 independent batches.

- `completions: 10` means 10 successful completions are required.
- `parallelism: 3` means Kubernetes can run up to 3 Pods concurrently.
- Kubernetes creates more Pods as needed until the completion target is reached or the Job fails.

The exact execution order is not guaranteed. If the task requires each batch to process a specific partition, the application must know which partition it is responsible for, or the workload must be designed to coordinate that assignment.

## Key points

- **A Job runs a finite workload.** Its goal is successful completion, not continuous availability.
- **The Job controller manages Pods.** It creates Pods based on the Job's completion and parallelism requirements.
- **Successful Pods count toward completion.** A Job can remain in the `Complete` state even though its Pods have terminated.
- **Failed work may be retried.** `backoffLimit` controls the number of counted failures before the Job is marked failed.
- **Container retries and Pod retries are different.** With `restartPolicy: OnFailure`, a failed container may be restarted inside the same Pod. With `restartPolicy: Never`, Kubernetes can create a replacement Pod after a failure.
- **Jobs are not exactly-once execution mechanisms.** A retry can repeat work, so design database writes and external side effects to be idempotent where possible.
- **Parallelism increases throughput, not guaranteed speed.** More Pods can also increase CPU, memory, database, and API pressure.
- **Jobs do not automatically disappear when finished.** Completed Jobs and their Pods can remain available for inspection until they are deleted or cleaned up.
- **Use `ttlSecondsAfterFinished` for automatic cleanup.** This is useful for short-lived batch workloads, but retain completed Jobs if you need longer operational visibility.
- **Use `activeDeadlineSeconds` to limit execution time.** This is particularly useful for jobs that must not run indefinitely.
- **A Job is not a scheduler for recurring work.** Use a CronJob when the workload must run on a time-based schedule.
- **A Job does not automatically provide data-pipeline orchestration.** Dependencies, ordering, data quality gates, and cross-job recovery may require an orchestrator or additional application logic.
- **A Job can complete successfully even if the application did not produce the expected business result.** Kubernetes primarily observes process exit status and Job completion conditions. Validate the actual output separately.

## Commands / manifests

### 1. Create a simple Job

Save this manifest as `job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox:1.36
          command: ["sh", "-c", "echo 'Job started'; sleep 5; echo 'Job completed'"]
```

Apply the manifest:

```bash
kubectl apply -f job.yaml
```

What this does:
- `apiVersion: batch/v1`: Uses the Kubernetes API version for Jobs.
- `kind: Job`: Creates a Job resource.
- `metadata.name`: Names the Job `hello-job`.
- `containers`: Defines the container that executes the task.
- `command`: Runs a shell command, prints messages, waits five seconds, and exits successfully.
- `restartPolicy: Never`: Allows Kubernetes to create another Pod after a failure rather than restarting the container inside the same Pod.

### 2. Inspect Job status

```bash
kubectl get jobs
kubectl describe job hello-job
kubectl get pods
```

Look for:
- `Complete`: The required successful completion count has been reached.
- `Failed`: The Job has exhausted its retry policy or failed another terminal condition.
- `Active`: Pods are still executing or the Job is still working toward completion.

A Pod may briefly appear as `Running`, then become `Completed`. The Job can be complete even though no container remains running.

### 3. Read the Job's logs

First identify the Pod:

```bash
kubectl get pods -l job-name=hello-job
```

Then inspect its logs:

```bash
kubectl logs job/hello-job
```

You can also inspect an individual Pod:

```bash
kubectl logs <pod-name>
```

Replace `<pod-name>` with the actual Pod name. Job-level log retrieval is convenient when the Job has a single execution Pod. For Jobs with multiple Pods, inspect individual Pods when you need to identify which execution produced particular output.

### 4. Configure retries and a timeout

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: retry-job
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 120
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox:1.36
          command: ["sh", "-c", "echo 'Attempting task'; exit 1"]
```

This example deliberately fails.

- `backoffLimit: 3`: Allows up to three counted Pod failures before the Job is marked failed. The total number of attempts is not necessarily three because the limit counts failures, and behavior depends on the restart policy and failure conditions.
- `activeDeadlineSeconds: 120`: Limits the Job's active duration to 120 seconds.
- `exit 1`: Returns a nonzero exit status, indicating failure.

Apply and inspect it:

```bash
kubectl apply -f retry-job.yaml
kubectl describe job retry-job
kubectl get pods -l job-name=retry-job
```

Kubernetes uses exponential backoff between retries. The active deadline can stop further retries even if the retry limit has not been reached.

### 5. Run multiple successful completions

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 10
  parallelism: 3
  backoffLimit: 4
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox:1.36
          command: ["sh", "-c", "echo 'Processing batch'; sleep 3; echo 'Batch complete'"]
```

Apply and monitor:

```bash
kubectl apply -f batch-job.yaml
kubectl get jobs -w
kubectl get pods -l job-name=batch-job
```

The Job aims for 10 successful completions and allows up to 3 Pods to run concurrently. Because the example executes the same command in every Pod, it demonstrates parallel execution but does not assign unique data partitions to each Pod.

For real batch processing, implement explicit work partitioning, such as reading distinct queue messages or claiming non-overlapping database records.

### 6. Automatically clean up completed Jobs

Add the following field under `spec`:

```yaml
ttlSecondsAfterFinished: 300
```

For example:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cleanup-job
spec:
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox:1.36
          command: ["sh", "-c", "echo 'Finished'"]
```

The Job becomes eligible for deletion about 300 seconds after it reaches a finished state. The cleanup controller performs deletion asynchronously, so cleanup is not guaranteed to occur at the exact second. Deleting the Job also deletes its dependent Pods under the normal cascading deletion behavior.

### 7. Delete a Job

```bash
kubectl delete job hello-job
```

Deleting a Job normally deletes the Pods it created as well. If you need to preserve Pods for debugging, Kubernetes supports orphaning dependents, but use that deliberately because the remaining Pods are no longer managed by the deleted Job.

### 8. Useful inspection commands

```bash
kubectl get jobs -o wide
kubectl describe job <job-name>
kubectl get pods --selector=job-name=<job-name>
kubectl logs job/<job-name>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl delete job <job-name>
```

Replace `<job-name>` with the actual Job name.

## Common failures & fixes

| Problem | Likely cause | Fix |
|---|---|---|
| Job stays `Active` | The process has not exited, or Pods cannot start | Inspect Pods with `kubectl get pods -l job-name=<job-name>` and review logs and events |
| Job becomes `Failed` | The process repeatedly exits with an error | Run `kubectl describe job <job-name>` and `kubectl logs job/<job-name>`; fix the underlying application error |
| `ImagePullBackOff` | Invalid image name, unavailable registry, authentication issue, or missing image pull secret | Verify the image reference, registry access, and `imagePullSecrets` configuration |
| `CrashLoopBackOff` | A container repeatedly fails and is restarted, often with `restartPolicy: OnFailure` | Check container logs, command arguments, dependencies, and exit codes |
| `Pending` Pods | Insufficient resources, scheduling constraints, or unavailable nodes | Inspect `kubectl describe pod <pod-name>` and review events, requests, taints, tolerations, and node capacity |
| Job never completes | The application does not exit or the completion target is too high | Make the task terminate after processing its work; verify `completions` and application logic |
| Job fails after retries | `backoffLimit` is exhausted | Correct the underlying error before rerunning the Job |
| Job exceeds its time limit | `activeDeadlineSeconds` is too low or processing is too slow | Optimize the task, review dependencies, or adjust the deadline to a realistic value |
| Duplicate records or side effects | A retry repeated work that had already partially succeeded | Make processing idempotent; use unique keys, transactions, checkpoints, or deduplication |
| Some batch items are missing | Parallel Pods did not receive distinct work assignments, or processing failed partway through | Track each item or partition explicitly and validate that all expected work was completed |
| Logs are unavailable | The Pod was deleted, logs have rotated, or the wrong Pod was inspected | Inspect the Job and Pod status promptly; configure centralized logging for durable retention |
| Completed Jobs accumulate | No cleanup policy is configured | Set `ttlSecondsAfterFinished` where appropriate, or use an explicit retention policy |

### Practical debugging sequence

When a Job fails, investigate in this order:

```bash
kubectl get job <job-name>
kubectl describe job <job-name>
kubectl get pods -l job-name=<job-name>
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

1. Confirm whether the Job is active, complete, or failed.
2. Review Job events for retry exhaustion, deadline expiration, or scheduling issues.
3. Inspect the associated Pods to identify container state and exit codes.
4. Read application logs to locate the actual failure.
5. Fix the cause and create a new Job or deliberately rerun the workload.

Changing a Job's Pod template in place is generally restricted because the Pod template is normally immutable. For a changed command or image, create a new Job or delete and recreate the existing one, taking care not to interrupt required processing.

## References

- [Kubernetes documentation: Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes documentation: Job patterns](https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-patterns)
- [Kubernetes documentation: Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes documentation: kubectl commands](https://kubernetes.io/docs/reference/kubectl/)
- [Kubernetes documentation: TTL-after-finished controller](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
