# kubectl get events, kubectl explain

> Phase 2 — Kubernetes Core (highest priority) › kubectl fluency

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

`kubectl get events` and `kubectl explain` are two essential commands for troubleshooting Kubernetes workloads and understanding Kubernetes resource definitions.

**`kubectl get events`**

Kubernetes events are records of notable occurrences in the cluster. They help explain why a Pod failed to start, why a container restarted, why a Deployment could not create replicas, or why a volume could not be mounted.

For example, a Pod may remain in `Pending` because the cluster has insufficient CPU or memory. Its events may contain a `FailedScheduling` message explaining the problem.

Events provide operational clues about what happened, but they are not a complete audit trail or a permanent log. They are typically retained for a limited period, depending on cluster configuration.

**`kubectl explain`**

`kubectl explain` displays documentation for Kubernetes API resource types and their fields directly from the API schema.

For example, if you need to understand `spec.replicas` in a Deployment manifest, you can run:

```bash
kubectl explain deployment.spec.replicas
```

This tells you what the field means without requiring you to search through external documentation.

It is particularly useful when writing YAML manifests because it helps you understand which fields exist, their data types, their purpose, and how nested fields are structured.

**How they work together**

1. Use `kubectl get` to identify a resource that is failing.
2. Use `kubectl describe` and `kubectl get events` to investigate the failure.
3. Use `kubectl explain` to understand the configuration fields involved.
4. Correct the manifest or workload configuration.
5. Apply the fix and verify the result.

For example, if a Pod cannot schedule because of resource constraints, events can explain the scheduling failure. `kubectl explain pod.spec.containers.resources` can help you understand the configuration for CPU and memory requests and limits.

## Key points

- **Events describe occurrences:** They commonly report scheduling decisions, image-pull failures, volume problems, container startup failures, probe failures, and scaling activity.
- **Events are different from logs:** Events describe Kubernetes-related occurrences; container logs contain output produced by an application or process.
- **Events are different from audit logs:** Events are not a complete record of every API request or cluster action.
- **Events have a limited retention period:** Old events may disappear. Collect important operational events externally if longer retention is required.
- **Namespace scope matters:** `kubectl get events` normally displays events in the current namespace. Use `-n` for a specific namespace or `-A` for all namespaces.
- **Event ordering:** Use `--sort-by=.metadata.creationTimestamp` to order events by their recorded creation timestamps. This is useful, although timestamps and event aggregation can make the sequence imperfect.
- **Repeated events may be aggregated:** Kubernetes can aggregate repeated occurrences into a single event object with a count. Do not assume every occurrence appears as a separate row.
- **Event types:** `Normal` events typically describe expected operations, while `Warning` events commonly indicate a problem that needs investigation. A `Normal` event does not guarantee that the application is healthy.
- **Useful event reasons:** `FailedScheduling`, `FailedMount`, `FailedAttachVolume`, `BackOff`, `Unhealthy`, and `Failed` are examples of reasons worth investigating.
- **`kubectl describe` complements events:** It shows resource details, status, and related events. It is often the fastest first step when investigating a particular Pod or Deployment.
- **`kubectl explain` reads API documentation:** It uses the Kubernetes API schema available to the selected cluster, helping you inspect supported fields for the cluster's API resources.
- **Use dot notation for nested fields:** For example, `kubectl explain deployment.spec.template.spec.containers` explores container definitions inside a Deployment's Pod template.
- **Use `--recursive`:** This displays nested fields and helps you explore a resource's structure.
- **Use `--api-version` when needed:** This lets you request documentation for a particular API version, provided that version is supported by the command and schema source.
- **Required fields and defaults:** `kubectl explain` can show field types, descriptions, and required-field information when available in the schema. It does not necessarily show every admission default or cluster-specific validation rule.
- **API versions matter:** A field supported in one API version may be unavailable or behave differently in another. Check the version used in your manifest.
- **CRDs have their own schemas:** Custom Resource Definitions may provide documentation through `kubectl explain` when their schemas are available from the cluster.
- **A schema is not a full operational guide:** `kubectl explain` helps with structure and field meaning, but it does not explain every runtime behavior, controller interaction, or application-specific requirement.
- **Practical troubleshooting order:** Check resource status, inspect its events, review logs when relevant, and then consult the API schema for the configuration fields involved.

## Commands / manifests

### 1. List events in the current namespace

```bash
kubectl get events
```

This displays events associated with resources in the current namespace.

Typical columns include:

- `LAST SEEN`: How recently the event was observed, depending on kubectl and cluster behavior.
- `TYPE`: Usually `Normal` or `Warning`.
- `REASON`: A short classification of the event.
- `OBJECT`: The resource associated with the event.
- `MESSAGE`: A description of what happened.

Example output:

```text
LAST SEEN   TYPE      REASON              OBJECT          MESSAGE
2m          Warning   FailedScheduling    pod/batch-job   Insufficient cpu
1m          Normal    Scheduled           pod/web-abc     Successfully assigned default/web-abc
```

These rows are illustrative. Actual messages and displayed columns vary by Kubernetes version and cluster.

### 2. List events across all namespaces

```bash
kubectl get events -A
```

This is useful when you do not know which namespace contains the failing resource.

You can also specify a namespace:

```bash
kubectl get events -n development
```

Remember that namespace-scoped events do not show every event in the cluster.

### 3. Sort events by creation timestamp

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

To inspect events across namespaces in timestamp order:

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

This helps identify recent failures and their surrounding activity. Event aggregation and timestamp behavior mean that the output should not be treated as a perfect chronological audit trail.

### 4. Display only Warning events

```bash
kubectl get events --field-selector type=Warning
```

This narrows the output to Warning events when supported by the cluster's event API.

To check for warnings in a particular namespace:

```bash
kubectl get events -n development --field-selector type=Warning
```

Warning events are a useful starting point, but some problems may be represented by `Normal` events or may not generate events at all.

### 5. Filter events for a specific resource

```bash
kubectl get events --field-selector involvedObject.name=my-pod
```

This filters events by the name of the involved object. Field-selector support depends on the resource and fields exposed by the API.

For a Pod in a specific namespace:

```bash
kubectl get events -n development --field-selector involvedObject.name=my-pod
```

If you are unsure of the Pod's exact name, first run:

```bash
kubectl get pods -n development
```

### 6. Describe a Pod and inspect its events

```bash
kubectl describe pod my-pod
```

The output includes details such as:

- Pod phase and conditions
- Node assignment
- Container states and restart counts
- Image and container configuration
- Volume mounts
- Recent events associated with the Pod

For a Pod in another namespace:

```bash
kubectl describe pod my-pod -n development
```

This is often more useful than viewing the full event list when troubleshooting one workload.

### 7. Explain a basic Kubernetes resource

```bash
kubectl explain pod
kubectl explain deployment
kubectl explain service
```

These commands describe the API fields for a Pod, Deployment, and Service.

Use them when you need to understand a resource's structure before writing a manifest.

### 8. Explain a nested field

```bash
kubectl explain deployment.spec.replicas
```

This explores the `replicas` field under the Deployment's `spec`.

Other useful examples:

```bash
kubectl explain deployment.spec.selector
kubectl explain deployment.spec.template
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources
kubectl explain service.spec.ports
```

These commands help you understand how selectors, Pod templates, container settings, and Service ports are defined.

### 9. Explore the full resource structure

```bash
kubectl explain deployment --recursive
```

This displays nested fields within the Deployment schema.

For a Pod:

```bash
kubectl explain pod --recursive
```

Recursive output can be extensive. Prefer exploring a specific field when you know which part of the manifest you need to understand.

### 10. Understand a field before configuring it

Suppose you want to configure CPU and memory requests for a container.

First, inspect the schema:

```bash
kubectl explain pod.spec.containers.resources
```

Then inspect its nested fields:

```bash
kubectl explain pod.spec.containers.resources.requests
kubectl explain pod.spec.containers.resources.limits
```

Example manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: nginx:1.28
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

In this example:

- `requests` describes the CPU and memory resources used for scheduling and resource allocation decisions.
- `limits` sets resource ceilings enforced by Kubernetes and the container runtime, with behavior depending on the resource type.
- `250m` means one quarter of a CPU core.
- `128Mi` means 128 mebibytes of memory.

Save the file as `resource-demo.yaml`, then validate and apply it in a suitable development environment:

```bash
kubectl apply --dry-run=server -f resource-demo.yaml
kubectl apply -f resource-demo.yaml
kubectl describe pod resource-demo
```

The Pod may remain unscheduled if the cluster cannot satisfy its requests. A valid manifest does not guarantee that the workload can run successfully.

### 11. Explain a resource from a specific API version

```bash
kubectl explain deployment --api-version=apps/v1
```

This requests the schema for the specified API version.

Use the API version that matches the resource's `apiVersion` when investigating version-specific fields.

### 12. Investigate a Pod stuck in Pending

Start with:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for messages such as:

- `FailedScheduling`
- `Insufficient cpu`
- `Insufficient memory`
- Taints or affinity constraints preventing scheduling

Then inspect the relevant configuration:

```bash
kubectl explain pod.spec.containers.resources
kubectl explain pod.spec.affinity
```

The second command is relevant when node affinity or Pod affinity rules may affect scheduling.

Use the event message to guide the investigation instead of changing resource requests or scheduling rules blindly.

### 13. Investigate an image-pull failure

Run:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get events --field-selector type=Warning
```

Look for `ErrImagePull` or `ImagePullBackOff`.

Possible causes include an incorrect image name or tag, registry access problems, missing image-pull credentials, or network connectivity issues.

Inspect the image configuration:

```bash
kubectl explain pod.spec.containers.image
```

Correct the image or registry configuration, then apply the updated manifest if necessary.

### 14. Investigate a container that keeps restarting

Run:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

`--previous` retrieves logs from the previous container instance when available. It is particularly useful when the current container has restarted after a crash.

Inspect the events for `BackOff` or `Unhealthy`, then check the container command, environment variables, configuration, and probes.

For probe configuration:

```bash
kubectl explain pod.spec.containers.livenessProbe
kubectl explain pod.spec.containers.readinessProbe
kubectl explain pod.spec.containers.startupProbe
```

### 15. Hands-on troubleshooting exercise

Create a Pod manifest named `broken-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
spec:
  containers:
    - name: nginx
      image: nginx:invalid-tag
      ports:
        - containerPort: 80
```

Apply the manifest:

```bash
kubectl apply -f broken-pod.yaml
```

Inspect the Pod:

```bash
kubectl get pods
kubectl describe pod broken-pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

The invalid image tag should cause an image-pull problem, assuming the tag does not exist in the configured registry.

Use `kubectl describe` and the events to identify the cause rather than guessing.

Correct the image to a valid tag, then apply the manifest again:

```bash
kubectl apply -f broken-pod.yaml
kubectl get pods
```

Wait for the Pod to become ready. When finished, remove the test resource:

```bash
kubectl delete -f broken-pod.yaml
```

Run this exercise only in a development namespace or other environment where you are authorized to create and delete resources.

## Common failures & fixes

- **No events are displayed**
  - Cause: No events exist in the selected namespace, relevant events have expired, or the current context or namespace is incorrect.
  - Fix: Verify `kubectl config current-context`, inspect `kubectl get events -A`, and check the resource directly with `kubectl describe`.

- **Events do not explain the problem**
  - Cause: Not every failure generates a useful event, and events may be aggregated or expire.
  - Fix: Combine event inspection with `kubectl describe`, container logs, resource status, and application-level monitoring.

- **Events appear out of order**
  - Cause: Event aggregation, timestamps, and updates can make the displayed sequence differ from the exact order of underlying operations.
  - Fix: Sort by `.metadata.creationTimestamp` and correlate events with Pod conditions, container states, and logs. Do not use events as a complete audit record.

- **A Warning event appears repeatedly**
  - Cause: A problem may be recurring, such as repeated scheduling failures, probe failures, or image-pull attempts.
  - Fix: Inspect the event's reason and message, then investigate the underlying resource. Repeated events do not necessarily represent separate event objects.

- **`Forbidden` while reading events**
  - Cause: Your Kubernetes identity lacks permission to list events in the selected namespace.
  - Fix: Check `kubectl auth can-i list events -n <namespace>` and request the required least-privilege permission.

- **`kubectl explain` reports that a resource or field cannot be found**
  - Cause: The resource kind, field path, or API version may be incorrect or unsupported by the selected cluster.
  - Fix: Check the spelling, run `kubectl api-resources`, and inspect the parent field before exploring nested fields.

- **A Custom Resource Definition cannot be explained**
  - Cause: The CRD is not installed, its API is not available, or its schema does not expose the expected information.
  - Fix: Verify the CRD with `kubectl get crds`, confirm its API version, and inspect the CRD schema.

- **`kubectl explain` output differs from an online tutorial**
  - Cause: The cluster's Kubernetes version or API schema differs from the tutorial's version.
  - Fix: Check `kubectl version` and use documentation that matches the cluster's API version.

- **A manifest passes schema checks but the workload fails**
  - Cause: Schema validation checks structure and supported fields, not every runtime condition, external dependency, or application requirement.
  - Fix: Inspect events, resource conditions, logs, image availability, permissions, and cluster capacity.

- **A Pod remains Pending with `FailedScheduling`**
  - Cause: The scheduler cannot find a suitable node because of resource requests, node selectors, affinity, taints, or other constraints.
  - Fix: Read the event message, inspect node capacity with `kubectl describe nodes`, and review the relevant scheduling configuration.

- **An image-pull error appears**
  - Cause: The image tag is invalid, the registry is inaccessible, or authentication is missing.
  - Fix: Verify the image reference, registry access, and image-pull secrets. Use the Pod events to distinguish the likely cause.

- **A Pod has restarted, but the current logs are empty**
  - Cause: The previous container instance may have produced the useful error before it exited.
  - Fix: Run `kubectl logs <pod-name> --previous` and inspect the Pod's restart count and events.

- **A field appears in `kubectl explain` but is rejected when applied**
  - Cause: The field may be used in the wrong location, the manifest may target a different API version, or admission and validation rules may reject the configuration.
  - Fix: Inspect the complete field path, check `apiVersion`, and use `kubectl apply --dry-run=server -f <file.yaml>` to get server-side validation feedback.

## References

- [kubectl get events](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [kubectl explain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [Debugging Applications](https://kubernetes.io/docs/tasks/debug/)
- [Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Kubernetes Events API](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/)
- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
