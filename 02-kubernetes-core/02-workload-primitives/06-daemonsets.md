# DaemonSets

> Phase 2 — Kubernetes Core (highest priority) › Workload primitives

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **DaemonSet** is a Kubernetes workload resource that ensures a copy of a particular Pod runs on each eligible node in a cluster.

Unlike a Deployment, where you specify the desired number of replicas, a DaemonSet manages Pod placement based on the nodes that qualify to run its workload.

For example, a cluster has five worker nodes. You need a log collection agent running on every worker so that logs from all five nodes can be collected. A DaemonSet is designed for this purpose.

Common use cases include:

- **Log collection:** Run Fluent Bit or another logging agent on each node.
- **Monitoring:** Deploy node-level monitoring agents that collect CPU, memory, disk, and network metrics.
- **Networking:** Run node-level networking components where the networking implementation uses a DaemonSet.
- **Security:** Deploy host-level security monitoring agents.
- **Storage:** Run node-level storage agents when required by the storage platform.
- **Node maintenance:** Deploy agents that inspect node health or collect system information.

### How a DaemonSet works

1. You create a DaemonSet containing a Pod template.
2. The DaemonSet controller identifies nodes that are eligible to run the Pod.
3. Kubernetes schedules the required Pods onto those nodes, subject to scheduling rules and resource availability.
4. When a new eligible node joins the cluster, Kubernetes arranges for the DaemonSet Pod to run on it.
5. When an eligible node becomes ineligible, the corresponding Pod may be removed.
6. If a DaemonSet Pod fails or is deleted, Kubernetes attempts to restore the desired coverage.

A DaemonSet does not necessarily run exactly one Pod on every node in the cluster. It targets **every eligible node**. Node selectors, affinity rules, taints, tolerations, and other scheduling constraints can affect which nodes qualify.

### Example: DaemonSet across worker nodes

Suppose a cluster has three worker nodes:

```text
Kubernetes Cluster
|
|-- Worker Node 1
|   |-- Log Agent Pod
|
|-- Worker Node 2
|   |-- Log Agent Pod
|
|-- Worker Node 3
|   |-- Log Agent Pod
```

If another eligible worker node joins the cluster, the DaemonSet controller arranges for a Log Agent Pod to run there as well.

The number of Pods is therefore driven by eligible node coverage, not by a fixed replica count.

### DaemonSet vs Deployment vs StatefulSet

| Feature | DaemonSet | Deployment | StatefulSet |
|---|---|---|---|
| Primary purpose | Run node-level workloads | Run and update replicated applications | Manage applications that need stable identities or storage associations |
| Pod placement | Typically one Pod per eligible node | Based on replica count and scheduling | Based on ordered or stable Pod identities |
| Replica configuration | No ordinary `replicas` field | Uses `spec.replicas` | Uses `spec.replicas` |
| New node joins | Adds a Pod if the node qualifies | Does not automatically increase replicas just because a node joins | Does not automatically increase replicas just because a node joins |
| Common examples | Log agents, node monitoring | APIs, web applications | Databases and clustered stateful systems |
| Scaling model | Changes with eligible node coverage | Set desired replica count | Set desired replica count |

### DaemonSet vs a Deployment with one replica

A Deployment with `replicas: 1` runs one Pod across the cluster, not one Pod on every node.

For example:

- A Deployment with `replicas: 1` usually maintains one Pod.
- A Deployment with `replicas: 5` aims to maintain five Pods, subject to scheduling and resource constraints.
- A DaemonSet aims to maintain one Pod on each eligible node, so the number of Pods changes as eligible node coverage changes.

If you need one monitoring agent per node, a DaemonSet is generally the appropriate resource.

## Key points

- **One Pod per eligible node:** This is the defining behavior of a DaemonSet.
- **Node coverage drives Pod count:** The number of Pods can increase or decrease as the set of eligible nodes changes.
- **No ordinary replica count:** Unlike Deployments, DaemonSets do not use `spec.replicas` to control their workload size.
- **New nodes are handled automatically:** The DaemonSet controller attempts to create the required Pod on each newly eligible node.
- **Failed Pods are recreated:** Kubernetes works to restore the desired DaemonSet coverage.
- **Scheduling rules determine eligibility:** Node selectors, affinity, taints, tolerations, and resource availability influence where Pods can run.
- **DaemonSets are useful for node-level infrastructure:** Logging, monitoring, networking, and security agents are common examples.
- **DaemonSet Pods can use host resources:** Some node agents need host filesystem access, host networking, or elevated permissions. These settings should be granted only when necessary.
- **DaemonSet does not mean one Pod per container or application:** It means a Pod is managed for each eligible node.
- **Control-plane nodes may be excluded:** Some clusters apply taints to control-plane nodes, preventing ordinary DaemonSet Pods from running there unless the relevant tolerations are configured.
- **DaemonSet Pods still need sufficient resources:** If a node lacks CPU or memory capacity, its Pod may remain unscheduled.
- **Updates use a rollout strategy:** DaemonSets support rolling updates, allowing new Pod versions to replace old ones across nodes.
- **A DaemonSet is not a batch workload:** Use a Job for finite work, a CronJob for scheduled work, and a DaemonSet for ongoing node-level coverage.

## Commands / manifests

### 1. Create a basic DaemonSet

Save this manifest as `daemonset.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: agent
          image: busybox:1.36
          command:
            - sh
            - -c
            - 'while true; do echo "Node agent running"; sleep 30; done'
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              memory: 32Mi
```

Apply it:

```bash
kubectl apply -f daemonset.yaml
```

What this manifest does:

- `kind: DaemonSet`: Creates a DaemonSet resource.
- `selector.matchLabels`: Identifies the Pods managed by the DaemonSet.
- `template.metadata.labels`: Supplies matching labels for the Pods.
- `containers`: Defines the container that runs on each eligible node.
- `command`: Keeps the example process running so the Pod remains active.
- `resources.requests`: Specifies the CPU and memory requested for scheduling.
- `resources.limits.memory`: Limits the container's memory usage.

The example demonstrates DaemonSet behavior. A production node agent would use an appropriate logging, monitoring, networking, or security image rather than a BusyBox loop.

### 2. Inspect the DaemonSet

```bash
kubectl get daemonsets
kubectl describe daemonset node-agent
kubectl get daemonset node-agent -o wide
```

Check these fields:

- `DESIRED`: Number of nodes that should run a DaemonSet Pod.
- `CURRENT`: Number of nodes that currently have a DaemonSet Pod created.
- `READY`: Number of DaemonSet Pods that are ready.
- `AVAILABLE`: Number of Pods available according to the DaemonSet's availability requirements.
- `UP-TO-DATE`: Number of nodes running the updated Pod template.

For example, if `DESIRED` is `3` and `READY` is `2`, the DaemonSet currently has incomplete ready coverage. Inspect the Pods and events to identify the cause.

### 3. Check which nodes run the Pods

```bash
kubectl get pods -l app=node-agent -o wide
```

The `-o wide` option includes node placement information.

Example output:

```text
NAME                READY   STATUS    NODE
node-agent-abc12    1/1     Running   worker-1
node-agent-def34    1/1     Running   worker-2
node-agent-ghi56    1/1     Running   worker-3
```

Pod names will differ in your cluster. The important detail is the `NODE` column, which helps verify node coverage.

List nodes separately:

```bash
kubectl get nodes
kubectl get nodes -o wide
```

Compare the nodes with the DaemonSet's desired coverage. Do not assume every node in the cluster is eligible.

### 4. Restrict the DaemonSet to selected nodes

Suppose you want the agent to run only on nodes labeled `workload-type=worker`.

First, label a node:

```bash
kubectl label node <node-name> workload-type=worker
```

Add the following under `spec.template.spec` in the DaemonSet manifest:

```yaml
nodeSelector:
  workload-type: worker
```

Apply the updated manifest:

```bash
kubectl apply -f daemonset.yaml
```

Verify placement:

```bash
kubectl get nodes --show-labels
kubectl get pods -l app=node-agent -o wide
kubectl describe daemonset node-agent
```

The DaemonSet will target nodes matching the selector. Pods on nodes that no longer qualify may be removed, while qualifying nodes without a Pod will receive one.

### 5. Inspect a DaemonSet Pod's logs

List its Pods:

```bash
kubectl get pods -l app=node-agent -o wide
```

Read logs from a specific Pod:

```bash
kubectl logs <pod-name>
```

Follow logs continuously:

```bash
kubectl logs -f <pod-name>
```

For node agents, inspect logs from multiple nodes when investigating node-specific issues. A problem affecting only one node may not appear in the logs from another.

### 6. Update the DaemonSet image

Change the container image:

```bash
kubectl set image daemonset/node-agent \
  agent=busybox:1.36
```

This example uses the same image as before. In a real deployment, replace it with the intended new image.

Watch the rollout:

```bash
kubectl rollout status daemonset/node-agent
kubectl rollout history daemonset/node-agent
```

A DaemonSet rolling update replaces Pods across eligible nodes according to its update strategy and availability settings.

### 7. Roll back a DaemonSet update

If the DaemonSet has a recorded previous revision, attempt a rollback:

```bash
kubectl rollout undo daemonset/node-agent
```

Check the result:

```bash
kubectl rollout status daemonset/node-agent
kubectl describe daemonset node-agent
```

Rollback availability depends on the recorded rollout history. Verify the running image and configuration after the rollback.

### 8. Delete the DaemonSet

```bash
kubectl delete daemonset node-agent
```

Deleting the DaemonSet normally deletes the Pods it manages. Use caution when deleting node-level infrastructure agents because this can interrupt log collection, monitoring, or other node services.

### 9. Useful debugging commands

```bash
kubectl get daemonsets -A
kubectl describe daemonset <daemonset-name>
kubectl get pods -o wide
kubectl get pods -l <label-key>=<label-value> -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get nodes --show-labels
kubectl get events --sort-by=.metadata.creationTimestamp
```

Use `-A` to inspect DaemonSets across all namespaces. Replace the placeholders with the actual resource names and labels.

## Common failures & fixes

| Problem | Likely cause | Fix |
|---|---|---|
| DaemonSet shows fewer Pods than expected | Some nodes are ineligible or Pods cannot be scheduled | Inspect `kubectl describe daemonset <name>`, node labels, taints, tolerations, and events |
| `DESIRED` is lower than the total node count | Node selectors, affinity, or scheduling rules exclude some nodes | Check the Pod template's scheduling constraints and verify node labels |
| Pod remains `Pending` | Insufficient resources, taints, affinity constraints, or unavailable nodes | Run `kubectl describe pod <pod-name>` and inspect scheduling events |
| Pod is in `ImagePullBackOff` | Image name is invalid or registry access fails | Verify the image reference and registry credentials |
| Pod enters `CrashLoopBackOff` | The container exits repeatedly or encounters runtime errors | Inspect logs, exit codes, commands, and configuration |
| DaemonSet Pods do not run on control-plane nodes | Control-plane taints prevent scheduling | Add appropriate tolerations only if running the workload on those nodes is intended |
| DaemonSet creates too many resource-consuming Pods | Every eligible node runs a copy of the workload | Review resource requests and limits, reduce eligibility if appropriate, and optimize the agent |
| Updated Pods are not becoming ready | The new image or configuration is broken, or readiness checks fail | Inspect rollout status, Pod events, and container logs; correct the template |
| Rolling update is stuck | New Pods cannot become available or update constraints prevent progress | Inspect the DaemonSet strategy, availability settings, node capacity, and Pod readiness |
| A Pod is missing from one node | The node is ineligible, unschedulable, or unable to host the Pod | Compare node labels and taints, then inspect scheduling events |
| Monitoring or logs are missing for one node | The agent is absent, unhealthy, or misconfigured on that node | Identify the node using `kubectl get pods -o wide`, then inspect that Pod and the node's status |
| DaemonSet selector does not match Pod labels | The selector and Pod template labels are inconsistent | Ensure `spec.selector.matchLabels` matches the labels under `spec.template.metadata.labels` |
| Host files are inaccessible | Required host paths or permissions are not configured | Configure only the necessary `hostPath` mounts and permissions, following security best practices |

### Practical debugging sequence

When a DaemonSet does not cover the expected nodes:

1. Run `kubectl get daemonset <name>` and compare `DESIRED`, `READY`, and `AVAILABLE`.
2. Run `kubectl get pods -o wide` to identify the nodes running the DaemonSet Pods.
3. Inspect the DaemonSet with `kubectl describe daemonset <name>`.
4. Inspect a missing or unhealthy Pod with `kubectl describe pod <pod-name>`.
5. Review node labels, taints, tolerations, affinity rules, and resource availability.
6. Read the Pod logs and fix application-level errors.
7. If a new version was deployed, inspect the rollout and verify that the updated Pods become available.

Remember: the DaemonSet controller aims to provide a Pod on each eligible node. A low `READY` count does not automatically mean the controller is broken. It often means some Pods cannot start or become ready.

## References

- [Kubernetes documentation: DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Kubernetes documentation: Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes documentation: Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes documentation: Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes documentation: Updating a DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/)
