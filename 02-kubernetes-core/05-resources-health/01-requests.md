# Requests

> Phase 2 — Kubernetes Core (highest priority) › Resources & health

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Kubernetes resource requests define the amount of CPU and memory a container asks Kubernetes to account for when scheduling a Pod onto a node.

A **resource request is not the same as a resource limit**. Requests influence scheduling and resource allocation decisions, while limits constrain how much of a resource a container can consume.

Kubernetes uses two primary resource settings for containers:

- **CPU request:** The amount of CPU used for scheduling decisions and CPU resource weighting under contention.
- **Memory request:** The amount of memory used for scheduling decisions and memory-based resource accounting.

### Requests vs limits

| Property | Request | Limit |
|---|---|---|
| Main purpose | Helps the scheduler choose a suitable node | Sets a maximum allowed resource usage |
| CPU behaviour | Influences CPU allocation and relative weighting under contention | CPU usage above the limit is throttled |
| Memory behaviour | Used for scheduling and resource accounting | Exceeding the effective memory limit can cause the container to be terminated with an OOM condition |
| Must be equal to the other value? | No | No |
| Can it be omitted? | Yes, subject to admission defaults and policies | Yes, subject to admission defaults and policies |

For example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

This container requests 0.5 CPU cores and 256 MiB of memory. It can use up to 1 CPU core and 512 MiB of memory under these limits.

**Important:** Requests do not guarantee that a container will always receive its requested CPU execution time or that its memory will never be reclaimed. They guide scheduling and resource management. CPU is a compressible resource, meaning workloads can be throttled under contention. Memory is generally non-compressible, meaning memory pressure can lead to eviction or out-of-memory termination.

### CPU units

Kubernetes measures CPU in cores. One CPU unit represents one logical CPU core, or the equivalent CPU capacity exposed by the node.

| CPU value | Meaning |
|---|---|
| `1` | One CPU core |
| `500m` | Half a CPU core |
| `250m` | One-quarter of a CPU core |
| `100m` | One-tenth of a CPU core |
| `2000m` | Two CPU cores |

The suffix `m` means *millicpu*, or one-thousandth of a CPU unit.

For example, a request of `500m` does not mean the container permanently owns half a physical CPU. It represents half a CPU core's worth of requested capacity for scheduling and resource accounting.

### Memory units

Memory is measured in bytes, usually expressed using binary or decimal suffixes.

| Memory value | Meaning |
|---|---|
| `128Mi` | 128 mebibytes |
| `256Mi` | 256 mebibytes |
| `1Gi` | 1 gibibyte |
| `512M` | 512 megabytes |
| `1G` | 1 gigabyte |

`Mi` and `Gi` are binary units, while `M` and `G` are decimal units. For most Kubernetes application configurations, `Mi` and `Gi` are common choices.

### How scheduling uses requests

Suppose a Kubernetes node has 4 CPU cores and 8 GiB of allocatable memory.

Its existing Pods have already requested:

- 2.5 CPU cores
- 5 GiB of memory

The remaining allocatable capacity is approximately:

- 1.5 CPU cores
- 3 GiB of memory

A new Pod requests 1 CPU core and 2 GiB of memory. If its other scheduling requirements are satisfied, the node has sufficient remaining capacity for that Pod's requests.

If another Pod requests 2 CPU cores and 4 GiB of memory, the scheduler cannot fit it on that node based on the remaining requested capacity, even if actual usage is currently low.

This is why **scheduling is based primarily on requests, not just current CPU and memory usage**.

### Why requests matter in production

- Prevents the scheduler from placing more requested workload capacity on a node than it can accommodate.
- Helps Kubernetes make more predictable placement decisions.
- Influences CPU allocation under contention.
- Affects Pod Quality of Service (QoS) classification when combined with limits and other resource settings.
- Helps cluster autoscalers identify Pods that cannot be scheduled due to insufficient node capacity.
- Improves capacity planning and reduces the risk of resource contention when requests reflect realistic workload needs.

Requests should be based on observed workload behaviour, profiling, and expected load rather than arbitrary values.

## Key points

- Requests are specified under `resources.requests` in a container specification.
- CPU requests influence scheduling and relative CPU weighting under contention.
- Memory requests influence scheduling and memory resource accounting.
- Requests are not hard maximums. A container may use more than its request if it is allowed by its limits and the node has resources available.
- Limits define maximum resource usage. CPU limits can cause throttling, while exceeding a memory limit can lead to OOM termination.
- Kubernetes schedules Pods based on their resource requests and node allocatable capacity, along with other scheduling constraints.
- A node's allocatable capacity is the capacity available to Pods after accounting for resources reserved for the operating system, Kubernetes components, and other configured reservations.
- The scheduler considers the sum of relevant Pod resource requests, not merely the current observed utilization.
- Resource requests are configured per container. A Pod with multiple containers has aggregate resource requirements based on its containers, with additional considerations for init containers and Pod-level resource features.
- Requests can be set without limits, and limits can be set without explicitly specifying requests. Admission policies and defaulting behaviour can affect the final configuration.
- If a limit is specified but a request is omitted, Kubernetes may default the request to the limit for that resource, subject to admission configuration and policy.
- If neither request nor limit is specified and no admission default applies, the container may be scheduled without an explicit request, making capacity planning less predictable.
- A CPU request of `500m` means half a CPU core of requested capacity. A memory request of `256Mi` means 256 mebibytes of requested memory.
- Requests influence QoS classification:
  - `Guaranteed`: Containers meet the required matching CPU and memory request/limit conditions.
  - `Burstable`: The Pod has at least one resource request or limit but does not qualify as Guaranteed.
  - `BestEffort`: No container in the Pod specifies CPU or memory requests or limits.
- QoS classification affects eviction priority under node pressure, but it does not replace correct capacity planning or resource sizing.
- Setting requests too high can leave allocatable capacity unused and prevent Pods from scheduling. Setting them too low can cause overpacking, CPU contention, or memory pressure.
- Resource requests do not guarantee node-level isolation or prevent all forms of resource contention.

## Commands / manifests

### 1. Create a Pod with CPU and memory requests

Save the following as `resource-requests.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Apply the manifest:

```bash
kubectl apply -f resource-requests.yaml
```

Check the Pod:

```bash
kubectl get pod resource-demo
kubectl describe pod resource-demo
```

The container requests 250 millicpu and 128 MiB of memory. Its limits are 500 millicpu and 256 MiB of memory.

### 2. Inspect the resource configuration

Display the configured requests and limits:

```bash
kubectl get pod resource-demo \
  -o jsonpath='{.spec.containers[0].resources}'
```

Alternatively, inspect the full resource configuration:

```bash
kubectl get pod resource-demo -o yaml
```

This shows the configuration recorded in the Pod specification. It does not show current CPU or memory consumption.

### 3. Check node allocatable capacity

List nodes:

```bash
kubectl get nodes
```

Inspect a node's capacity and allocatable resources:

```bash
kubectl describe node <node-name>
```

Look for the following sections:

- `Capacity`: Total node resources reported to Kubernetes.
- `Allocatable`: Resources available for Pods.
- `Allocated resources`: Summary of resources requested and limited by Pods on that node.

The `Allocated resources` section helps explain why a Pod may remain Pending even when actual utilization appears low.

### 4. Compare requests with current usage

If the Kubernetes Metrics Server is installed and working:

```bash
kubectl top nodes
kubectl top pods
kubectl top pod resource-demo --containers
```

These commands report observed CPU and memory usage.

Compare observed usage with configured requests:

```bash
kubectl describe pod resource-demo
```

Remember that `kubectl top` reports usage, while the Pod specification records requests and limits. They answer different questions.

### 5. Observe an unschedulable Pod

Create a Pod with an intentionally large request:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oversized-request
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "100"
          memory: "100Gi"
```

Apply it:

```bash
kubectl apply -f oversized-request.yaml
```

Inspect its status and events:

```bash
kubectl get pod oversized-request
kubectl describe pod oversized-request
```

If no eligible node can satisfy the requests, the Pod remains Pending and events commonly report that the nodes have insufficient CPU or memory.

This is a demonstration of scheduling constraints, not a recommended production configuration.

Clean up:

```bash
kubectl delete pod oversized-request
```

### 6. Configure requests in a Deployment

For a replicated application, define resources in the Deployment's Pod template:

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
kubectl apply -f deployment-resources.yaml
kubectl get deployment web-app
kubectl get pods -l app=web-app
kubectl describe deployment web-app
```

For three replicas, the application requests approximately 600 millicpu and 384 MiB of memory in total, excluding other workloads and any additional resource requirements.

### 7. Update a Deployment's resource requests

You can update a container's resource configuration using `kubectl set resources`:

```bash
kubectl set resources deployment/web-app \
  --containers=nginx \
  --requests=cpu=300m,memory=192Mi \
  --limits=cpu=600m,memory=384Mi
```

Check rollout progress:

```bash
kubectl rollout status deployment/web-app
kubectl get pods -l app=web-app
```

Changing a Deployment's Pod template triggers a rollout, replacing Pods with new ones using the updated resource configuration.

For a Git-managed workflow, update the YAML manifest and apply it so that the declared configuration remains consistent with the deployment source.

## Common failures & fixes

| Failure | Likely cause | Fix |
|---|---|---|
| Pod remains Pending with `Insufficient cpu` | No eligible node has enough unallocated CPU request capacity | Reduce requests where justified, scale the workload differently, or add node capacity |
| Pod remains Pending with `Insufficient memory` | The memory request exceeds the remaining allocatable capacity on eligible nodes | Review requests and node capacity; scale out if needed |
| Actual CPU usage is low but the Pod cannot schedule | The scheduler evaluates requested resources rather than just current usage | Inspect node allocated requests using `kubectl describe node <node-name>` |
| Container is repeatedly OOMKilled | Memory usage exceeds its effective limit, or the node experiences memory pressure | Inspect Pod status and events, measure memory usage, and adjust requests or limits based on evidence |
| Application is slow under CPU contention | CPU limit throttling or insufficient CPU capacity may be affecting the workload | Inspect CPU usage and throttling metrics; review requests and limits |
| Resource values are rejected | Invalid quantity syntax or unsupported values | Use valid quantities such as `250m`, `1`, `128Mi`, or `1Gi` |
| Deployment rollout fails after changing resources | New Pods cannot schedule with the updated requests, or admission policies reject the specification | Inspect rollout status, Pod events, namespace policies, and node capacity |
| Requests appear different from the manifest | Limit-to-request defaulting, a LimitRange, or admission mutation changed the effective Pod configuration | Inspect the live Pod YAML and namespace `LimitRange` resources |
| `kubectl top` returns an error | Metrics Server is unavailable or has not collected metrics | Verify Metrics Server health and allow time for metrics collection |
| A workload monopolizes CPU or memory | Requests are missing or too low relative to real workload demand | Measure the workload under representative load and set realistic requests and limits |

Useful troubleshooting commands:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml
kubectl describe node <node-name>
kubectl get limitrange
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top nodes
kubectl top pods
```

## References

- [Kubernetes: Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes: Assign CPU Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)
- [Kubernetes: Assign Memory Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
- [Kubernetes: Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Kubernetes: Node Status and Allocatable Resources](https://kubernetes.io/docs/concepts/architecture/nodes/)
