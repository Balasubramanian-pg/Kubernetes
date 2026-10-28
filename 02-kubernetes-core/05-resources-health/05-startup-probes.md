# Startup probes

> Phase 2 — Kubernetes Core (highest priority) › Resources & health

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **startup probe** tells Kubernetes whether a container's application has completed its startup process.

It is designed for applications that need more time to initialize before they can respond reliably to health checks. Examples include Java applications that load large frameworks, APIs that warm caches, and data-processing services that initialize connections or load configuration.

Without a startup probe, a liveness probe might fail repeatedly while an application is still starting, causing Kubernetes to restart it before initialization completes.

A startup probe prevents this by giving the application a dedicated startup window.

### How a startup probe works

1. Kubernetes starts the container.
2. The startup probe checks whether the application has completed startup.
3. While the startup probe has not succeeded, Kubernetes does not execute the configured liveness and readiness probes for that container.
4. When the startup probe succeeds, Kubernetes stops running the startup probe and allows the liveness and readiness probes to take over.
5. If the startup probe fails repeatedly and reaches its `failureThreshold`, Kubernetes treats the container as having failed startup and restarts it according to the Pod's restart policy.

**Key idea:** A startup probe protects slow-starting applications from premature liveness failures. It does not permanently replace liveness or readiness checks.

### Example scenario

Suppose a Java API takes 90 seconds to initialize.

Without a startup probe, a liveness probe configured with a short initial delay might start checking the application too early. Repeated failures could trigger restarts, preventing the application from ever finishing initialization.

With a startup probe, Kubernetes can allow a longer startup window.

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  periodSeconds: 5
  failureThreshold: 30
```

This configuration allows approximately 150 seconds of failed probe checks before the startup probe reaches its failure threshold, assuming checks continue at the configured period.

Once the startup probe succeeds, the liveness and readiness probes can begin operating.

The application must implement the `/health/startup` endpoint. Kubernetes does not create this endpoint automatically.

### Startup vs liveness vs readiness

| Probe | Main purpose | Failure behaviour |
|---|---|---|
| Startup | Determine whether initialization has completed | Repeated failures trigger a container restart |
| Liveness | Determine whether the running application should be restarted | Repeated failures trigger a container restart |
| Readiness | Determine whether the application should receive normal Service traffic | Pod is marked not ready and normally excluded from ready Service endpoints |

Consider an application with these behaviours:

- During initialization, it loads configuration and establishes internal state.
- After initialization, it may occasionally become unresponsive.
- During operation, it may temporarily be unable to serve requests.

The probes address each situation separately:

- **Startup:** Has initialization finished?
- **Liveness:** Is the application still functioning, or should it be restarted?
- **Readiness:** Can the application serve requests right now?

### Startup probe timing

The most important settings are:

- `initialDelaySeconds`: Delay before the first probe, unless startup-probe sequencing determines when checks begin.
- `periodSeconds`: Interval between probe checks.
- `timeoutSeconds`: Maximum time allowed for an individual probe attempt.
- `failureThreshold`: Number of consecutive failed checks allowed before the probe is considered failed.
- `successThreshold`: Number of consecutive successful checks needed to recover from failure. For startup probes, this must be `1`.

For example:

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 30
```

The startup probe begins checking without an initial delay. It allows 30 consecutive failed checks, with checks scheduled at five-second intervals. The exact time before a restart depends on probe scheduling and the duration of individual checks.

### Choosing a startup probe threshold

The startup window should accommodate realistic startup times under normal and expected load.

For a rough estimate:

\[
\text{Startup window} \approx \text{periodSeconds} \times \text{failureThreshold}
\]

For example:

\[
5 \text{ seconds} \times 30 = 150 \text{ seconds}
\]

This is a planning approximation, not an exact guarantee. Probe timeouts, scheduling, and the timing of the first check affect the actual duration.

Set the window long enough for normal initialization, but not so long that a genuinely broken application remains stuck indefinitely.

### Probe types

Startup probes support the same main mechanisms as liveness and readiness probes.

**HTTP GET**

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
```

**TCP socket**

```yaml
startupProbe:
  tcpSocket:
    port: 8080
```

**Exec command**

```yaml
startupProbe:
  exec:
    command:
      - cat
      - /tmp/startup-complete
```

**gRPC**

```yaml
startupProbe:
  grpc:
    port: 50051
```

Choose the mechanism that accurately represents application startup completion. A TCP connection alone may succeed before the application is fully initialized.

## Key points

- Startup probes are intended for slow-starting applications.
- They prevent liveness and readiness probes from running until the startup probe succeeds.
- A failed startup probe does not immediately restart the container. Consecutive failures must reach the configured `failureThreshold`.
- When the startup probe reaches its failure threshold, Kubernetes treats the container as having failed startup and restarts it according to the Pod's restart policy.
- After success, the startup probe stops running for that container instance.
- Startup probes can use HTTP, TCP, exec, or gRPC checks.
- The application must implement the health endpoint or check used by the probe.
- `periodSeconds`, `failureThreshold`, and `timeoutSeconds` are key settings for controlling startup behaviour.
- `successThreshold` must be `1` for startup probes.
- A startup probe is especially useful when application startup time varies significantly between deployments or environments.
- Do not set the startup window based only on the fastest observed startup. Account for realistic initialization time under expected load.
- A startup probe should test whether the application has completed initialization, rather than simply checking whether the container process exists.
- A startup probe does not replace liveness or readiness probes. It protects the initialization phase, while the other probes handle health and traffic eligibility afterward.
- A startup probe should not be used to conceal an application that never finishes initializing. Investigate repeated startup failures rather than continually increasing the threshold.
- If a startup probe is configured, liveness and readiness probes for that container are held back until startup succeeds.
- Probe configuration is defined per container. In a multi-container Pod, each container can have its own probe settings.
- Startup probes do not guarantee that the application will become ready immediately after startup. Readiness may remain false while the application completes additional work required to serve traffic.

## Commands / manifests

### 1. Create a Pod with a startup probe

The following example uses NGINX and checks its HTTP endpoint.

Save as `startup-probe.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        timeoutSeconds: 2
        failureThreshold: 12
      livenessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        timeoutSeconds: 2
        failureThreshold: 3
```

Apply the manifest:

```bash
kubectl apply -f startup-probe.yaml
```

Inspect the Pod:

```bash
kubectl get pod startup-demo
kubectl describe pod startup-demo
```

This example allows approximately 60 seconds of failed startup checks before the startup probe reaches its failure threshold. NGINX should normally start much faster, so the startup probe will usually succeed before using the full window.

### 2. Inspect startup probe events

```bash
kubectl describe pod startup-demo
```

Check the `Events` section for startup probe failures and related messages.

Inspect the Pod's container status:

```bash
kubectl get pod startup-demo \
  -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

A restart count greater than zero does not prove that the startup probe caused the restart. Inspect the events and termination details to determine the cause.

### 3. Simulate a slow-starting application

This example delays creating a health-check file for 30 seconds. The startup probe checks for the file, allowing the container time to initialize.

Save as `slow-start-demo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Initializing application..."
          sleep 30
          touch /tmp/startup-complete
          echo "Initialization complete."
          sleep 3600
      startupProbe:
        exec:
          command:
            - cat
            - /tmp/startup-complete
        periodSeconds: 5
        timeoutSeconds: 2
        failureThreshold: 12
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/startup-complete
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

Apply the manifest:

```bash
kubectl apply -f slow-start-demo.yaml
```

Watch the Pod:

```bash
kubectl get pod slow-start-demo -w
```

In another terminal, inspect the logs:

```bash
kubectl logs slow-start-demo
```

The startup probe initially fails because the file does not exist. After the application creates the file, the startup probe succeeds and the liveness probe can begin.

This is a learning demonstration. A real application should expose a meaningful startup condition rather than relying on a file unrelated to its initialization.

### 4. Inspect the effective probe configuration

```bash
kubectl get pod startup-demo -o yaml
```

This displays the probe settings recorded in the Pod specification.

To inspect the container's last termination reason:

```bash
kubectl get pod startup-demo \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

The field may be empty if the container has not terminated previously.

### 5. Configure startup probes in a Deployment

For a slow-starting API, define the startup probe in the Deployment's Pod template.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
spec:
  replicas: 3
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
            timeoutSeconds: 2
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

Replace the example image and health endpoints with those supported by your application.

Apply and inspect:

```bash
kubectl apply -f api-deployment.yaml
kubectl rollout status deployment/api-app
kubectl get pods -l app=api-app
```

The startup probe protects each container during initialization. The readiness probe determines when each Pod can receive normal Service traffic, while the liveness probe checks for recoverable failures after startup.

### 6. Troubleshoot startup failures

Inspect the Pod and events:

```bash
kubectl describe pod <pod-name>
```

Read current container logs:

```bash
kubectl logs <pod-name>
```

Read previous container logs after a restart:

```bash
kubectl logs <pod-name> --previous
```

Inspect the probe configuration:

```bash
kubectl get pod <pod-name> -o yaml
```

Check restart counts:

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

### 7. Clean up practice resources

```bash
kubectl delete pod startup-demo
kubectl delete pod slow-start-demo
kubectl delete deployment api-app
```

Only delete resources created for this exercise.

## Common failures & fixes

| Failure | Likely cause | Fix |
|---|---|---|
| Container restarts before initialization completes | Startup window is too short or the probe is too aggressive | Measure realistic startup time and adjust `periodSeconds` and `failureThreshold` |
| Startup probe repeatedly returns HTTP `404` | The configured endpoint does not exist or is not available during startup | Use the correct startup endpoint and verify its behaviour during initialization |
| Startup probe returns connection refused | The application has not opened the configured port yet, or the port is incorrect | Verify the application startup sequence, listening address, and port |
| Container keeps restarting despite a long startup window | Application startup is failing, hanging, or taking longer than expected | Inspect logs, termination details, and initialization dependencies |
| Startup succeeds but liveness subsequently fails | The application becomes unhealthy after startup, or the liveness check is too strict | Investigate application health and adjust the liveness check based on evidence |
| Startup succeeds but Pod remains unready | The readiness probe fails or another readiness condition is unmet | Inspect the readiness endpoint, Pod conditions, and events |
| Probe times out intermittently during initialization | The application is overloaded or the probe timeout is too short | Investigate resource pressure and tune the timeout if justified |
| Container is restarted due to OOM before startup completes | The application exceeds its effective memory limit | Inspect memory usage, OOM events, and container limits |
| Startup probe succeeds before initialization is actually complete | The endpoint checks only whether a server or port is available | Implement a startup check that represents completion of the required initialization |
| Application starts slowly only in production | Production workload, configuration, dependency latency, or resource constraints increase startup time | Compare startup logs and resource usage across environments, then size the startup window appropriately |

Useful troubleshooting commands:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl get pod <pod-name> -o yaml
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl rollout status deployment/<deployment-name>
```

## References

- [Kubernetes: Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes: Configure Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes: Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
