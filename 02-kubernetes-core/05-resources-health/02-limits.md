# Limits

> Phase 2 — Kubernetes Core (highest priority) › Resources & health

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Kubernetes resource limits define the maximum CPU and memory resources a container is allowed to consume.

They are configured under `resources.limits` in a container specification.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

In this example:

- **CPU request:** 250 millicpu, or 0.25 CPU cores.
- **CPU limit:** 500 millicpu, or 0.5 CPU cores.
- **Memory request:** 128 MiB.
- **Memory limit:** 256 MiB.

Requests help Kubernetes schedule Pods and account for resources. Limits constrain resource consumption. A container can use more than its request when resources are available, but it cannot exceed its effective resource limits indefinitely.

### CPU limits

CPU is a *compressible resource*. When a container attempts to use more CPU than its configured limit, the Linux kernel's CPU control mechanisms throttle its CPU execution.

For example:

```yaml
resources:
  limits:
    cpu: "1"
```

The container is limited to approximately one CPU core's worth of CPU time over time. It can run on different CPU cores, but its total CPU consumption is constrained by the limit.

CPU throttling does not normally terminate the container. Instead, the application may become slower because it cannot execute as much CPU work as it needs.

**Example:** A data-processing application has a CPU limit of `500m`, but its workload needs 1.5 CPU cores to complete quickly. The container may experience CPU throttling, increasing processing time.

### Memory limits

Memory is generally a *non-compressible resource*. Kubernetes cannot throttle memory consumption in the same way it throttles CPU.

When a container exceeds its effective memory limit, the operating system's memory control mechanisms can trigger an out-of-memory (OOM) kill. The container may terminate and restart, depending on the Pod's restart policy and workload controller.

For example:

```yaml
resources:
  limits:
    memory: "512Mi"
```

The container is limited to 512 MiB of memory under the applicable container memory accounting rules.

**Example:** A Spark-related application inside a container needs 700 MiB of memory but has a 512 MiB limit. It may be terminated with an OOM condition rather than being allowed to consume the additional memory.

The exact behaviour depends on the runtime, kernel, and workload. Node-level memory pressure can also cause Pods to be evicted independently of a container exceeding its own memory limit.

### Requests vs limits

| Property | Request | Limit |
|---|---|---|
| Purpose | Guides scheduling and resource accounting | Constrains resource consumption |
| CPU | Influences CPU weighting under contention | Excess usage is throttled |
| Memory | Helps determine whether a Pod fits on a node | Exceeding the effective limit can trigger OOM termination |
| Must be equal? | No | No |
| Can usage exceed it? | Yes, if permitted by limits and resources available | CPU usage is throttled; memory overuse can trigger termination |

A common configuration is to set requests lower than limits so the application can use additional resources when available.

However, a limit that is too low can damage performance or cause repeated restarts. A limit that is too high may allow one workload to consume excessive node resources.

### What happens when limits are omitted?

Kubernetes does not automatically require every container to declare resource limits.

If a limit is omitted, the container may have no explicit limit for that resource, unless a namespace policy, admission configuration, or other defaulting mechanism supplies one.

A `LimitRange` can specify default requests and limits or enforce minimum and maximum values within a namespace.

This means you should inspect the **effective Pod configuration**, not assume that the submitted YAML is the final configuration.

### How limits affect Quality of Service

Kubernetes assigns Pods a Quality of Service (QoS) class based on their resource requests and limits.

- **Guaranteed:** Every relevant container has CPU and memory requests and limits set, and each container's request equals its limit for both resources.
- **Burstable:** The Pod has at least one CPU or memory request or limit but does not qualify as Guaranteed.
- **BestEffort:** No container in the Pod specifies CPU or memory requests or limits.

QoS classification influences eviction behaviour under node pressure. It does not mean that a Guaranteed Pod is immune to all failures or that its applications can never be terminated.

## Key points

- Limits are defined under `resources.limits` for each container.
- CPU limits constrain CPU time and can cause throttling when a workload tries to exceed its configured allowance.
- Memory limits constrain memory consumption. Exceeding the effective limit can cause an OOM kill.
- CPU and memory have different behaviours: CPU can be throttled, but memory cannot be throttled in the same way.
- Requests and limits are independent settings, although Kubernetes admission and defaulting rules can affect the effective values.
- A container can use more CPU or memory than its request when resources are available and its limits permit it.
- A container's CPU request does not reserve an exclusive CPU core by default. Exclusive CPU allocation requires additional configuration, such as the appropriate CPU Manager policy.
- A memory limit is not a reservation. The memory request is the value primarily used for scheduling decisions.
- Setting CPU limits too low can cause unnecessary throttling and poor application performance.
- Setting memory limits too low can cause repeated OOM kills, application restarts, and failed jobs.
- Setting limits too high can increase the risk of node-level resource contention if workloads consume their permitted maximums simultaneously.
- A Pod can be evicted because of node memory pressure even when its container has not exceeded its own memory limit.
- If a limit is specified but the corresponding request is omitted, Kubernetes may default the request to the limit, unless another admission mechanism or policy determines the effective value.
- Namespace-level `LimitRange` policies can supply default limits and requests or enforce allowed resource ranges.
- A resource quota can restrict the total requested or limited resources across a namespace.
- Resource limits are configured per container. A Pod with multiple containers can have different limits for each container, with aggregate resource behaviour depending on the Pod configuration and Kubernetes features in use.
- Use observed workload metrics and representative load tests to determine suitable limits rather than guessing.

## Commands / manifests

### 1. Create a Pod with CPU and memory limits

Save this as `resource-limits.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limit-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
```

Apply the manifest:

```bash
kubectl apply -f resource-limits.yaml
```

Inspect the Pod:

```bash
kubectl get pod limit-demo
kubectl describe pod limit-demo
```

The container requests 100 millicpu and 64 MiB of memory. Its limits are 500 millicpu and 128 MiB of memory.

### 2. Inspect the configured limits

Display the resource settings:

```bash
kubectl get pod limit-demo \
  -o jsonpath='{.spec.containers[0].resources}'
```

Or inspect the full Pod specification:

```bash
kubectl get pod limit-demo -o yaml
```

Check the actual container status:

```bash
kubectl get pod limit-demo \
  -o jsonpath='{.status.containerStatuses[*].state}'
```

The Pod specification shows configured limits. It does not, by itself, show current CPU or memory usage.

### 3. Monitor CPU and memory usage

If Metrics Server is installed and functioning:

```bash
kubectl top pod limit-demo --containers
kubectl top nodes
```

Compare the observed usage against the configured limits.

For example, if the container is consistently approaching its memory limit, investigate whether the application has a memory leak, uses large working sets, or needs a higher limit.

`kubectl top` reports resource usage but does not directly show CPU throttling. For detailed throttling analysis, use container or node monitoring metrics that expose CPU throttling data.

### 4. Demonstrate CPU throttling

The following Pod runs a CPU-intensive process. It requests 100 millicpu but has a CPU limit of 200 millicpu.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-limit-demo
spec:
  containers:
    - name: stress
      image: busybox:1.36
      command: ["sh", "-c", "while true; do :; done"]
      resources:
        requests:
          cpu: "100m"
          memory: "16Mi"
        limits:
          cpu: "200m"
          memory: "64Mi"
```

Apply and observe it:

```bash
kubectl apply -f cpu-limit-demo.yaml
kubectl get pod cpu-limit-demo
kubectl top pod cpu-limit-demo --containers
```

The process attempts to consume CPU continuously, but the CPU limit constrains its usage over time.

The measured usage may vary because of sampling intervals, node contention, and the container runtime. CPU throttling metrics provide stronger evidence of throttling than a single `kubectl top` reading.

Delete the test Pod when finished:

```bash
kubectl delete pod cpu-limit-demo
```

### 5. Demonstrate a memory limit failure

This example intentionally allocates more memory than the configured limit. Run it only in a disposable development cluster.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-limit-demo
spec:
  restartPolicy: Never
  containers:
    - name: memory-test
      image: python:3.12-slim
      command:
        - python
        - -c
        - |
          chunks = []
          while True:
              chunks.append(bytearray(10 * 1024 * 1024))
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "500m"
```

Apply and inspect:

```bash
kubectl apply -f memory-limit-demo.yaml
kubectl get pod memory-limit-demo
kubectl describe pod memory-limit-demo
```

The process attempts to allocate memory continuously. If it exceeds the effective container memory limit, it may be OOM-killed.

Inspect termination details:

```bash
kubectl get pod memory-limit-demo \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

The exact termination state and reason depend on the runtime and timing. A failed allocation or process-level error may occur before an OOM kill is reported.

Clean up:

```bash
kubectl delete pod memory-limit-demo
```

### 6. Configure limits in a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: "200m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Apply and inspect:

```bash
kubectl apply -f deployment-limits.yaml
kubectl rollout status deployment/web-app
kubectl get pods -l app=web-app
```

With three replicas, the containers collectively request 600 millicpu and 384 MiB of memory. Their configured limits total 1.5 CPU cores and 768 MiB of memory, excluding other Pod overheads and workloads.

The total limit is not a reservation. It represents the combined maximums configured for the three containers.

### 7. Update a Deployment's limits

Use `kubectl set resources` to update resource settings:

```bash
kubectl set resources deployment/web-app \
  --containers=nginx \
  --requests=cpu=300m,memory=192Mi \
  --limits=cpu=750m,memory=384Mi
```

Check the rollout:

```bash
kubectl rollout status deployment/web-app
kubectl describe deployment web-app
```

Changing the Pod template triggers a Deployment rollout. New Pods are created with the updated settings, subject to scheduling and admission constraints.

For declarative workflows, update the source YAML and apply it so the repository remains the source of truth.

### 8. Inspect namespace resource policies

List `LimitRange` objects:

```bash
kubectl get limitrange
kubectl describe limitrange <limitrange-name>
```

Inspect resource quotas:

```bash
kubectl get resourcequota
kubectl describe resourcequota <quota-name>
```

A `LimitRange` can provide defaults or enforce per-container or per-Pod resource constraints. A `ResourceQuota` can limit aggregate resource consumption or requested and limited capacity across a namespace.

### 9. Clean up practice resources

```bash
kubectl delete pod limit-demo
kubectl delete deployment web-app
```

Delete any additional demonstration Pods that you created.

## Common failures & fixes

| Failure | Likely cause | Fix |
|---|---|---|
| Container is repeatedly `OOMKilled` | Memory usage exceeds its effective memory limit | Inspect memory usage, termination state, and application behaviour; increase the limit if justified |
| Application becomes slow under load | CPU throttling caused by a restrictive CPU limit | Check CPU throttling metrics and workload demand; review the CPU limit |
| Pod remains Pending | Requests cannot fit on any eligible node | Review requests, node allocatable capacity, scheduling constraints, and cluster capacity |
| Pod creation is rejected | A `LimitRange` or `ResourceQuota` policy is violated | Inspect namespace policies and adjust the resource configuration or obtain an approved policy change |
| Effective request differs from expected value | Request defaulting or admission mutation occurred | Inspect the live Pod YAML and `LimitRange` settings |
| `kubectl top` does not work | Metrics Server is missing or unhealthy | Check Metrics Server deployment and availability |
| CPU usage appears below the limit but the application is still slow | Contention, throttling bursts, I/O bottlenecks, or other application issues | Inspect throttling metrics, node pressure, and application-level performance data |
| Pod is evicted even though it has not exceeded its memory limit | Node-level memory pressure or another eviction condition | Inspect Pod events, node conditions, and node memory pressure |
| Memory usage grows until the container restarts | Memory leak, large working set, or insufficient limit | Profile the application and distinguish expected peak usage from a leak |
| Limits are missing from the live Pod | No explicit limit was set and no policy supplied one | Configure limits in the workload specification or define appropriate namespace defaults |

Useful debugging commands:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml
kubectl logs <pod-name> --previous
kubectl top pods --containers
kubectl describe node <node-name>
kubectl get limitrange
kubectl get resourcequota
kubectl get events --sort-by=.metadata.creationTimestamp
```

## References

- [Kubernetes: Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes: Assign CPU Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)
- [Kubernetes: Assign Memory Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
- [Kubernetes: Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Kubernetes: Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Kubernetes: Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
