# Liveness probes

> Phase 2 — Kubernetes Core (highest priority) › Resources & health

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **liveness probe** is a health check that Kubernetes uses to determine whether a container is still functioning correctly. If the probe repeatedly fails beyond its configured failure threshold, Kubernetes treats the container as unhealthy and restarts it.

The purpose of a liveness probe is to recover from situations where an application is running but has become unable to make progress.

For example, an application might:
- Become stuck in an infinite loop.
- Deadlock while waiting for a resource.
- Stop responding to requests because of an internal failure.
- Remain running even though its internal processing has stalled.

Without a liveness probe, Kubernetes may consider the container healthy simply because its process is still running.

With a liveness probe, Kubernetes can detect certain failure conditions and restart the affected container.

### How a liveness probe works

1. Kubernetes starts the container.
2. The kubelet executes the configured health check at the specified intervals.
3. The probe succeeds or fails based on its configured test.
4. If the probe fails repeatedly and reaches `failureThreshold`, Kubernetes marks the container as unhealthy.
5. Kubernetes restarts the container according to the Pod's restart policy.

A failed liveness probe does not normally delete the Pod or replace it with a new Pod. It causes the affected container to be restarted. Controllers such as Deployments manage replica availability separately.

### Types of liveness probes

Kubernetes supports several probe mechanisms.

**1. HTTP GET probe**

Kubernetes sends an HTTP request to a specified path and port. A successful response with an HTTP status code from 200 through 399 counts as success.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

Use this when the application exposes an endpoint that reports whether it is functioning properly.

**2. TCP socket probe**

Kubernetes attempts to establish a TCP connection to the configured port. If the connection succeeds, the probe passes.

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
```

This is useful when an application does not expose an HTTP health endpoint. However, a successful TCP connection does not prove that the application can process requests correctly.

**3. Exec probe**

Kubernetes executes a command inside the container. The probe succeeds when the command exits with status code `0`.

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
```

This is useful when health can be determined through a local command or file check. The command must genuinely reflect application health, rather than merely checking that an unrelated file exists.

**4. gRPC probe**

Kubernetes can use a gRPC health-check endpoint when the application implements the required gRPC health-check protocol.

```yaml
livenessProbe:
  grpc:
    port: 50051
```

The application must support the gRPC health-check protocol. Kubernetes gRPC probes do not support arbitrary hostnames or custom authentication parameters in the same way a general gRPC client might.

### Liveness vs readiness vs startup probes

These three probes serve different purposes.

| Probe | Main question | What happens when it fails? |
|---|---|---|
| Liveness | Is the container still functioning, or should it be restarted? | Repeated failure triggers a container restart |
| Readiness | Should this Pod receive regular Service traffic? | The Pod is marked not ready and normally removed from matching Service endpoints |
| Startup | Has the application completed its startup process? | Repeated failure triggers a container restart; liveness and readiness checks are held back until startup succeeds when configured |

Example scenarios:

- **Liveness:** A web server has deadlocked and no longer responds to its health endpoint.
- **Readiness:** A web server is alive but temporarily cannot serve requests because it is warming a cache.
- **Startup:** A Java application needs 90 seconds to initialize before it can respond reliably to health checks.

A common production configuration uses all three probes when the application needs them.

### Why startup probes matter

An application that takes a long time to start can fail a liveness probe before it has become operational.

For example, suppose a liveness probe begins after 5 seconds and fails three times. A slow-starting application might be restarted repeatedly before initialization completes.

A startup probe gives the application additional time to initialize. While the startup probe has not succeeded, Kubernetes does not run the configured liveness and readiness probes for that container.

### Understanding probe timing

Common liveness probe fields include:

- `initialDelaySeconds`: How long to wait after the container starts before beginning probe checks.
- `periodSeconds`: How frequently to perform checks.
- `timeoutSeconds`: How long to wait for an individual probe attempt.
- `failureThreshold`: How many consecutive failures are needed before the probe is considered failed.
- `successThreshold`: How many consecutive successes are required to recover from a failed probe. For liveness probes, this must be `1`.

For example:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

The first check starts after the initial delay, and subsequent checks occur according to the configured period. Three consecutive failed checks trigger a restart. The exact restart time depends on probe scheduling, timeout behaviour, and the time at which failures occur.

**Important:** A liveness endpoint should normally check the application's ability to make progress, not every external dependency. If a shared database becomes unavailable and every application instance fails its liveness check, Kubernetes may restart all instances without fixing the database problem.

## Key points

- Liveness probes detect certain application failures that cannot be identified merely by checking whether a process exists.
- A failed liveness probe causes the kubelet to restart the container after the configured failure threshold is reached.
- Kubernetes does not automatically restart a container after a single failed probe unless the threshold is configured accordingly.
- Liveness probes do not directly determine whether a Pod receives Service traffic. Readiness probes handle that purpose.
- HTTP probes pass for status codes in the 200 through 399 range.
- TCP probes check whether a connection can be established, not whether the application can successfully process a request.
- Exec probes run a command inside the container and use its exit code to determine success or failure.
- gRPC probes require the application to implement the supported health-check protocol.
- `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, and `failureThreshold` control probe timing and failure detection.
- `successThreshold` must be `1` for liveness probes.
- A startup probe is useful for slow-starting applications and prevents liveness checks from prematurely restarting them before startup succeeds.
- Liveness probes should detect failures that a restart can reasonably fix.
- Do not make liveness checks unnecessarily dependent on remote databases, APIs, or other shared services. Temporary dependency failures can otherwise cause cascading restarts.
- The health endpoint must be implemented by the application. Kubernetes does not automatically create an HTTP `/health` endpoint.
- A probe can fail because of an incorrect path, port, timeout, application binding address, or an overly aggressive threshold.
- A liveness probe is not a complete monitoring system. Use metrics, logs, traces, and alerts to investigate the underlying cause of failures.

## Commands / manifests

### 1. Create a sample application with a liveness probe

The following example uses NGINX and checks its HTTP response on port `80`.

Save as `liveness-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

Apply the manifest:

```bash
kubectl apply -f liveness-pod.yaml
```

Check the Pod status:

```bash
kubectl get pod liveness-demo
kubectl describe pod liveness-demo
```

NGINX serves its default page at `/`, so the HTTP probe should normally succeed after startup.

### 2. Inspect probe events

```bash
kubectl describe pod liveness-demo
```

Look at the `Events` section for messages about successful operation, failed liveness checks, or container restarts.

Check restart counts:

```bash
kubectl get pod liveness-demo \
  -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

A restart count greater than zero indicates that a container has restarted, but does not by itself prove that a liveness probe caused it. Inspect the events and container termination details.

### 3. Check the container's last termination state

```bash
kubectl get pod liveness-demo \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

To inspect more detail:

```bash
kubectl describe pod liveness-demo
```

If the container restarted, investigate the termination reason and the associated events. Possible causes include liveness probe failures, application crashes, and out-of-memory termination.

### 4. Configure an exec-based liveness probe

This example creates a file and checks that it exists:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: exec-liveness-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          touch /tmp/healthy
          sleep 30
          rm -f /tmp/healthy
          sleep 600
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 2
```

Apply it:

```bash
kubectl apply -f exec-liveness.yaml
```

The application initially creates `/tmp/healthy`. After 30 seconds, it removes the file. The probe then fails repeatedly, and Kubernetes should restart the container.

Inspect the restart count:

```bash
kubectl get pod exec-liveness-demo
kubectl describe pod exec-liveness-demo
```

This is a learning demonstration. In a real application, a health check should validate a meaningful application condition rather than rely on a file that is unrelated to the application's actual health.

### 5. Configure liveness, readiness, and startup probes together

Example for an application that exposes health endpoints on port `8080`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-app
  template:
    metadata:
      labels:
        app: api-app
    spec:
      containers:
        - name: api
          image: your-registry/your-api:latest
          ports:
            - containerPort: 8080
          startupProbe:
            httpGet:
              path: /health/startup
              port: 8080
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
```

Replace the example image and endpoints with those supported by your application.

The startup probe allows approximately 150 seconds of failed checks before it reaches its configured failure threshold, assuming the probe continues at its configured period. Liveness and readiness checks are held back until startup succeeds.

The readiness endpoint should determine whether the application is ready to serve traffic. The liveness endpoint should determine whether the application can continue making progress.

### 6. Troubleshoot failed liveness checks

Inspect Pod details and events:

```bash
kubectl describe pod <pod-name>
```

Read the current container logs:

```bash
kubectl logs <pod-name>
```

Read logs from the previous container instance after a restart:

```bash
kubectl logs <pod-name> --previous
```

Inspect the configured probe:

```bash
kubectl get pod <pod-name> -o yaml
```

Check the current restart count:

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

### 7. Clean up practice resources

```bash
kubectl delete pod liveness-demo
kubectl delete pod exec-liveness-demo
kubectl delete deployment api-app
```

Only delete resources that you created for this exercise.

## Common failures & fixes

| Failure | Likely cause | Fix |
|---|---|---|
| `Liveness probe failed: HTTP probe failed with statuscode: 404` | The configured endpoint does not exist | Configure the correct health-check path |
| `Liveness probe failed: connection refused` | The application is not listening on the configured port or has not started | Verify the container port, application binding, and startup time |
| Container restarts repeatedly during startup | The liveness probe starts too early or is too aggressive | Add a startup probe or adjust the timing thresholds |
| Container restarts during temporary high load | The health endpoint cannot respond within the configured timeout | Investigate CPU or memory pressure, application responsiveness, and probe timeouts |
| Probe succeeds even though the application is broken | The endpoint checks only whether the HTTP server responds | Implement a health endpoint that accurately reflects the application's ability to make progress |
| Every replica restarts when a database is unavailable | Liveness checks depend on a shared external dependency | Keep liveness checks focused on recoverable local failures; use readiness to control traffic when appropriate |
| Probe works locally but fails in Kubernetes | Incorrect port, path, bind address, or network assumptions | Inspect the container configuration and verify that the application listens on the expected interface and port |
| `kubectl logs --previous` has no output | The container has not restarted, or previous logs are unavailable | Inspect current logs, Pod events, and container status |
| Restart count increases but no liveness failure is visible | Another failure, such as an application crash or OOM kill, caused the restart | Inspect termination reason, exit code, events, and previous logs |

Useful troubleshooting commands:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get pod <pod-name> -o yaml
```

## References

- [Kubernetes: Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes: Configure Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes: Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
