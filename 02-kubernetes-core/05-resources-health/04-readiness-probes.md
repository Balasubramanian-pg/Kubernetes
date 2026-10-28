# Readiness probes

> Phase 2 — Kubernetes Core (highest priority) › Resources & health

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **readiness probe** tells Kubernetes whether a container is ready to serve application traffic.

A container can be running without being ready. For example, an API server may have started, but it might still be loading configuration, warming a cache, establishing required connections, or initializing application state.

A readiness probe helps prevent Kubernetes Services from routing normal traffic to Pods that are not ready to handle requests.

### How a readiness probe works

1. Kubernetes starts a container.
2. The kubelet performs the configured readiness check.
3. If the check succeeds, the Pod can become ready, assuming its other readiness conditions are satisfied.
4. If the check fails, the Pod is marked not ready.
5. Kubernetes updates the relevant Service EndpointSlices so the Pod is normally excluded from regular Service traffic.
6. When the check succeeds again, the Pod can return to the set of ready endpoints.

Unlike a liveness probe, **a failed readiness probe does not restart the container**. It changes whether the Pod is considered ready to serve traffic.

Readiness checks continue throughout the container's lifetime. This allows Kubernetes to detect when an application temporarily becomes unable to serve requests and restore traffic routing when it recovers.

### Example scenario

Imagine a Deployment running three API replicas:

- Pod A is ready.
- Pod B is ready.
- Pod C is still initializing.

A Service normally routes traffic to Pods A and B because Pod C has not passed its readiness check.

When Pod C finishes initialization and passes its readiness probe, it can be added to the set of ready Service endpoints.

If Pod B later becomes unready, Kubernetes can stop sending normal Service traffic to it while leaving its container running.

### Types of readiness probes

Kubernetes supports the same primary probe mechanisms for readiness as for liveness.

**1. HTTP GET probe**

Kubernetes sends an HTTP request to an application endpoint. HTTP status codes from 200 through 399 count as success.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
```

Use this when the application exposes a readiness endpoint that reports whether it can handle requests.

**2. TCP socket probe**

Kubernetes attempts to establish a TCP connection to a configured port.

```yaml
readinessProbe:
  tcpSocket:
    port: 8080
```

This verifies that a connection can be established, but it does not necessarily prove that the application can process requests correctly.

**3. Exec probe**

Kubernetes executes a command inside the container. An exit code of `0` means the probe succeeded.

```yaml
readinessProbe:
  exec:
    command:
      - cat
      - /tmp/ready
```

This is useful when the application can expose its readiness state through a local command or file.

**4. gRPC probe**

Kubernetes can check an application's gRPC health endpoint when it implements the supported gRPC health-check protocol.

```yaml
readinessProbe:
  grpc:
    port: 50051
```

### Readiness vs liveness vs startup probes

| Probe | Purpose | What happens when it fails? |
|---|---|---|
| Readiness | Determines whether the Pod should receive normal Service traffic | Pod is marked not ready and normally excluded from ready Service endpoints |
| Liveness | Determines whether the container should be restarted | Repeated failures trigger a container restart |
| Startup | Determines whether the application has completed startup | Repeated failures trigger a container restart; liveness and readiness checks are held back until startup succeeds when configured |

These probes solve different problems.

For example, if an application is alive but waiting for a required initialization step, readiness can keep it out of normal traffic without repeatedly restarting it.

If the application is permanently stuck and restarting it may recover the situation, liveness can detect that condition.

For a slow-starting application, a startup probe prevents liveness and readiness checks from running before startup succeeds.

### Readiness and Kubernetes Services

A Kubernetes Service provides a stable way to access a group of Pods.

For a typical selector-based Service, Kubernetes uses EndpointSlices to track the available endpoints. Readiness influences which Pod endpoints are considered ready for normal traffic.

For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
```

If a matching Pod fails its readiness probe, it normally stops being a ready endpoint for this Service.

There are important nuances:

- Readiness affects endpoint eligibility, not whether the container process remains running.
- A Pod can be running but not ready.
- Direct Pod access and custom traffic-routing systems may behave differently from ordinary Service routing.
- A Service with `publishNotReadyAddresses: true` changes normal endpoint publication behaviour.
- Readiness does not guarantee that every existing connection is immediately terminated or that every client stops using an endpoint instantly.

### When to use readiness probes

Use readiness probes when an application has meaningful conditions that determine whether it can serve requests, such as:

- Completing application initialization.
- Loading required configuration.
- Warming a cache.
- Establishing essential internal state.
- Reaching a minimum level of capacity before accepting traffic.
- Detecting temporary overload or a condition where the application should stop receiving new requests.

A readiness endpoint should reflect the application's actual ability to serve requests. Avoid making it depend indiscriminately on every external service, because a shared dependency outage can make all replicas unready and remove the entire application from Service routing.

## Key points

- Readiness probes determine whether a container is ready to serve traffic.
- A failed readiness probe does not restart the container.
- A Pod can be in the `Running` phase while its `Ready` condition is false.
- For ordinary selector-based Services, readiness normally determines whether a Pod is included as a ready endpoint.
- Readiness probes continue running after the application becomes ready, allowing Kubernetes to detect later failures.
- When readiness succeeds again, the Pod can become ready and return to normal Service endpoint selection.
- HTTP probes succeed for status codes from 200 through 399.
- TCP probes test connection establishment, not full application correctness.
- Exec probes succeed when their configured command exits with status code `0`.
- gRPC probes require an application endpoint that implements the supported gRPC health-check protocol.
- `initialDelaySeconds` controls the delay before probing begins, unless startup-probe behaviour changes when readiness checks begin.
- `periodSeconds` controls the normal interval between checks.
- `timeoutSeconds` specifies how long an individual check may take.
- `failureThreshold` controls how many consecutive failures cause the probe to be considered failed.
- `successThreshold` controls how many consecutive successes are needed to recover readiness after a failure. It must be `1` for startup and liveness probes, but readiness probes can use a higher value.
- A readiness probe should test whether the application can serve requests, not merely whether its process exists.
- A liveness probe should focus on failures that restarting the container can reasonably fix.
- A startup probe is useful for applications that require substantial initialization time.
- If every replica becomes unready because of a shared dependency failure, the Service may have no ready endpoints. Design health checks carefully to avoid unnecessary cascading failures.
- Readiness does not replace application monitoring, metrics, logs, or distributed tracing.
- A successful readiness check is not a guarantee that every individual request will succeed. It is a signal used for traffic-routing decisions.

## Commands / manifests

### 1. Create a Pod with a readiness probe

The following example uses NGINX and checks its default HTTP page on port `80`.

Save as `readiness-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
  labels:
    app: readiness-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 5
        timeoutSeconds: 2
        failureThreshold: 3
        successThreshold: 1
```

Apply the manifest:

```bash
kubectl apply -f readiness-pod.yaml
```

Inspect the Pod:

```bash
kubectl get pod readiness-demo
kubectl describe pod readiness-demo
```

The `READY` column in `kubectl get pods` indicates how many containers are ready out of the total number of containers in the Pod.

### 2. Inspect the Pod's readiness condition

```bash
kubectl get pod readiness-demo \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

Possible output:

```text
True
```

`True` means the Pod's Ready condition is satisfied. `False` means the Pod is not ready.

You can also inspect the container's readiness state:

```bash
kubectl get pod readiness-demo \
  -o jsonpath='{.status.containerStatuses[*].ready}'
```

### 3. Create a Service for the application

Save as `readiness-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: readiness-service
spec:
  selector:
    app: readiness-demo
  ports:
    - port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f readiness-service.yaml
```

Inspect the Service and its EndpointSlices:

```bash
kubectl get service readiness-service
kubectl get endpointslice \
  -l kubernetes.io/service-name=readiness-service
```

When the Pod is ready, it should normally appear as a ready endpoint for this Service.

### 4. Demonstrate a readiness failure

The following example starts an NGINX container but configures its readiness probe to check a path that does not exist.

Save as `readiness-failure.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-failure-demo
  labels:
    app: readiness-failure-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /does-not-exist
          port: 80
        periodSeconds: 5
        timeoutSeconds: 2
        failureThreshold: 2
```

Apply and inspect:

```bash
kubectl apply -f readiness-failure.yaml
kubectl get pod readiness-failure-demo
kubectl describe pod readiness-failure-demo
```

The HTTP request should return `404`, causing the readiness probe to fail. The container should continue running, but the Pod should remain unready.

This demonstrates the key difference between readiness and liveness: **readiness failure removes the Pod from normal ready endpoint selection without restarting the container**.

### 5. Compare readiness and restart counts

Inspect the Pod's status:

```bash
kubectl get pod readiness-failure-demo
```

Check its restart count:

```bash
kubectl get pod readiness-failure-demo \
  -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

The restart count should normally remain unchanged because the failed probe is a readiness probe, not a liveness probe.

### 6. Use a readiness probe in a Deployment

For a real application, define the probe in the Deployment's Pod template.

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
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
```

Replace the example image and health endpoint with those supported by your application.

Apply and inspect:

```bash
kubectl apply -f api-deployment.yaml
kubectl rollout status deployment/api-app
kubectl get pods -l app=api-app
```

A Deployment rollout can be affected by readiness. If new Pods never become ready, the Deployment may be unable to complete its rollout successfully.

### 7. Troubleshoot readiness failures

Inspect Pod details and events:

```bash
kubectl describe pod <pod-name>
```

Check the application's logs:

```bash
kubectl logs <pod-name>
```

Inspect the configured probe:

```bash
kubectl get pod <pod-name> -o yaml
```

Check Service endpoint readiness:

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=<service-name> -o yaml
```

Look for endpoint conditions such as `ready` to understand whether the Pod is available for normal Service traffic.

### 8. Clean up practice resources

```bash
kubectl delete pod readiness-demo
kubectl delete pod readiness-failure-demo
kubectl delete service readiness-service
kubectl delete deployment api-app
```

Only delete resources created for this exercise.

## Common failures & fixes

| Failure | Likely cause | Fix |
|---|---|---|
| Pod is `Running` but not `Ready` | Readiness probe is failing or another readiness condition is unmet | Inspect `kubectl describe pod` and verify the configured probe |
| Readiness probe returns HTTP `404` | The configured endpoint does not exist | Use the application's actual readiness path |
| Readiness probe returns connection refused | The application is not listening on the configured port or has not started | Verify the port, application binding, and startup behaviour |
| Pod is ready but requests still fail | The readiness endpoint is too shallow or does not reflect actual request-serving capability | Improve the endpoint and investigate application logs and metrics |
| Service has no ready endpoints | All matching Pods are unready, selectors do not match, or endpoints are otherwise unavailable | Check Pod readiness, Service selectors, and EndpointSlices |
| Deployment rollout never completes | New Pods cannot pass readiness checks | Inspect new Pod events, application startup, probe paths, and timeout settings |
| Pod repeatedly becomes unready under load | The application cannot serve requests within the configured thresholds | Investigate resource pressure, application latency, and whether the readiness check is too strict |
| All replicas become unready when a database is unavailable | The readiness endpoint depends on a shared external service | Decide whether the application truly cannot serve any requests without that dependency; avoid unnecessarily removing every replica |
| Readiness changes are not reflected as expected in traffic routing | Endpoint updates and client-side routing behaviour may take time to propagate | Inspect EndpointSlices and allow for propagation; check proxies, caches, and existing connections |
| Readiness check succeeds before initialization is complete | The endpoint does not represent true application readiness | Update the application's readiness logic or use a startup probe for slow initialization |

Useful debugging commands:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get service <service-name>
kubectl get endpointslice \
  -l kubernetes.io/service-name=<service-name>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl rollout status deployment/<deployment-name>
```

## References

- [Kubernetes: Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes: Configure Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes: Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes: EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
