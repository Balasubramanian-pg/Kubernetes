# Where workloads actually run

> Phase 2 — Kubernetes Core (highest priority) › Cluster model

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

In Kubernetes, application workloads run inside **Pods**, which are assigned to cluster nodes. A Pod is the smallest deployable unit in Kubernetes and contains one or more containers that share certain resources, including a network namespace and, when configured, shared storage volumes.

The hierarchy is:

- **Cluster:** The complete Kubernetes environment.
- **Node:** A physical machine or virtual machine that provides resources for workloads.
- **Pod:** The deployable unit Kubernetes schedules onto a node.
- **Container:** The process-isolated application environment running inside a Pod.
- **Process:** The actual application code executing inside a container.

For example, a data engineering application might run a Python processing service inside a container. Kubernetes schedules the Pod containing that container onto a suitable worker node. The node's kubelet works with the container runtime to start the container.

```text
Kubernetes Cluster
|
|-- Control Plane
|   |-- API Server
|   |-- Scheduler
|   |-- Controllers
|   `-- etcd
|
|-- Worker Node 1
|   |-- kubelet
|   |-- Container Runtime
|   `-- Pod A
|       |-- Python Application Container
|       `-- Sidecar Container (optional)
|
`-- Worker Node 2
    |-- kubelet
    |-- Container Runtime
    `-- Pod B
        `-- Python Application Container
```

The control plane coordinates workload placement and lifecycle management. The worker node supplies the compute resources where the containers execute.

### How workload placement happens

Suppose you deploy a Kubernetes Deployment with three replicas of a data processing API.

1. You submit a manifest requesting three replicas.
2. The API server validates the request and stores the relevant cluster state.
3. The Deployment and ReplicaSet controllers work to create the required Pod objects.
4. The scheduler selects eligible nodes for Pods that have not yet been assigned to a node.
5. The kubelet on each selected node observes the assigned Pods and works with the container runtime to start their containers.
6. Kubernetes monitors Pod and container status and continues reconciling the desired state.

The three Pods may run on three different nodes, or multiple Pods may run on the same node. Kubernetes does not automatically guarantee one replica per node. Placement depends on resource availability, scheduling constraints, topology rules, and other cluster configuration.

If a node fails, Kubernetes may schedule replacement Pods on other eligible nodes, provided the workload controller, cluster state, available resources, and scheduling constraints permit recovery.

## Key points

- **Pods are the units Kubernetes schedules**
  - Kubernetes schedules Pods onto nodes, not individual containers independently.
  - A Pod can contain one or more containers.
  - Containers within the same Pod are scheduled together and share the Pod's network namespace.
  - Containers in a Pod can communicate over `localhost`.
  - Containers in the same Pod can share volumes when those volumes are mounted into each container.
  - Separate Pods generally have separate network identities, even when they run on the same node.

- **Containers execute on nodes**
  - A node provides CPU, memory, networking, and storage resources for its workloads.
  - The kubelet manages Pods assigned to the node.
  - The container runtime starts and manages the containers.
  - Containers are not directly executed by the Kubernetes API server or scheduler.
  - A Pod can be assigned to a node before its containers successfully start.

- **The scheduler selects a node**
  - The scheduler evaluates resource requests, node eligibility, affinity rules, taints and tolerations, topology constraints, and other scheduling requirements.
  - It selects a node for an unscheduled Pod and records that assignment through the Kubernetes API.
  - The scheduler does not normally start the container itself.
  - If no eligible node is available, the Pod can remain in `Pending` with scheduling events explaining the problem.

- **A Deployment does not run application code itself**
  - A Deployment declares the desired state of a replicated application.
  - Its controller manages ReplicaSets, which maintain the requested number of Pods.
  - The Pods are scheduled onto nodes and their containers execute there.
  - Deleting or updating a Deployment can change the lifecycle of its managed Pods.

- **Workloads may be spread across nodes**
  - Multiple Pods can run on one node if resources and scheduling rules allow it.
  - Replicas can be spread across nodes to reduce the impact of an individual node failure.
  - Pod anti-affinity and topology spread constraints can influence distribution.
  - A multi-replica Deployment does not, by itself, guarantee that each replica runs on a different node.

- **The control plane is different from the workload execution environment**
  - The control plane makes scheduling and reconciliation decisions.
  - Worker nodes typically execute user application workloads.
  - Some clusters allow workloads on control-plane nodes, while others prevent this through taints or provider-specific configuration.
  - In managed Kubernetes, the control plane may run on infrastructure that is not exposed as ordinary worker nodes to the user.

- **A running Pod is not necessarily a healthy application**
  - Pod phase `Running` means the Pod has been bound to a node and its containers have been created, with at least one container running or starting or restarting.
  - It does not guarantee that every container is ready to serve requests.
  - Readiness probes, container states, application logs, and actual service responses provide additional information.
  - A Pod can be running but not ready, or it can be repeatedly restarting.

- **Pod placement and application access are separate concerns**
  - A Pod can run successfully without being accessible from outside the cluster.
  - A Kubernetes Service can provide a stable endpoint for a set of Pods.
  - Ingress or Gateway resources, together with a suitable controller or implementation, can provide HTTP or HTTPS routing.
  - `containerPort` in a Pod manifest documents a container port but does not, by itself, publish that port on the node or expose the application externally.

- **Resource requests affect scheduling**
  - CPU and memory requests help the scheduler determine whether a node can accommodate a Pod.
  - CPU and memory limits constrain resource consumption according to Kubernetes' resource-management behavior.
  - A Pod can remain pending if its requests cannot be satisfied by any eligible node.
  - The scheduler considers configured requests, not simply the instantaneous CPU and memory usage reported by monitoring tools.

- **Workload placement can be inspected**
  - `kubectl get pods -o wide` displays the node assigned to each Pod.
  - `kubectl describe pod` provides scheduling events and container status.
  - `kubectl describe node` displays node capacity, allocatable resources, conditions, and assigned Pods.
  - These commands help distinguish scheduling problems from container startup and application problems.

## Commands / manifests

**1. Identify the nodes in the cluster**

```bash
kubectl get nodes -o wide
```

This lists the cluster nodes and additional information, including their IP addresses and Kubernetes versions.

Inspect one node:

```bash
kubectl describe node NODE_NAME
```

Replace `NODE_NAME` with a name from the previous command.

Look at the node's conditions, capacity, allocatable resources, and assigned Pods.

**2. Deploy an application with three replicas**

Create a file named `workload-placement.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: placement-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: placement-demo
  template:
    metadata:
      labels:
        app: placement-demo
    spec:
      containers:
        - name: web
          image: nginx:stable
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
```

Apply the manifest:

```bash
kubectl apply -f workload-placement.yaml
```

Check the Deployment:

```bash
kubectl get deployment placement-demo
```

Check the ReplicaSet:

```bash
kubectl get replicasets
```

Check the Pods:

```bash
kubectl get pods -l app=placement-demo -o wide
```

Example output:

```text
NAME                              READY   STATUS    NODE
placement-demo-7b8c9d6f4a-ab123   1/1     Running   worker-1
placement-demo-7b8c9d6f4a-cd456   1/1     Running   worker-2
placement-demo-7b8c9d6f4a-ef789   1/1     Running   worker-1
```

The names and node assignments above are illustrative. Your cluster may use different names, and all three Pods may run on one node.

**3. Determine exactly where each Pod runs**

```bash
kubectl get pods \
  -l app=placement-demo \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,PHASE:.status.phase'
```

This extracts three fields:

- `POD`: The Pod name.
- `NODE`: The node assigned to the Pod.
- `PHASE`: The Pod's reported lifecycle phase.

To retrieve the assigned node for a particular Pod:

```bash
kubectl get pod POD_NAME \
  -o jsonpath='{.spec.nodeName}{"\n"}'
```

If the node field is empty, the Pod has not yet been assigned to a node.

**4. Inspect the Pod's scheduling and execution details**

```bash
kubectl describe pod POD_NAME
```

Review:

- `Node`: The node assigned to the Pod.
- `Status`: The Pod's current phase.
- `Conditions`: Scheduling, initialization, and readiness information.
- `Containers`: Container state, image, restart count, and readiness.
- `Events`: Scheduling decisions and errors.

For example, an event such as `FailedScheduling` can indicate insufficient CPU or memory, an untolerated taint, or another scheduling constraint.

**5. Compare Pod placement with node capacity**

Display node resources:

```bash
kubectl describe node NODE_NAME
```

Look for the `Capacity`, `Allocatable`, and `Allocated resources` sections.

If Metrics Server or another supported metrics provider is available, view current node usage:

```bash
kubectl top nodes
```

View Pod usage:

```bash
kubectl top pods -A
```

Remember that resource requests and current usage are different. Scheduling decisions rely on configured requests and eligibility constraints rather than simply placing a Pod on the node with the lowest current CPU reading.

**6. Control placement with a node selector**

First, label a node:

```bash
kubectl label node NODE_NAME workload-tier=general
```

Then add the following to the Pod template's `spec`, alongside `containers`:

```yaml
spec:
  nodeSelector:
    workload-tier: general
  containers:
    - name: web
      image: nginx:stable
```

This is an illustrative fragment of the Pod template specification. Merge it into the existing Deployment manifest rather than replacing the entire manifest with this partial example.

The `nodeSelector` requires the Pod to be scheduled onto a node carrying the label `workload-tier=general`.

Verify the node label:

```bash
kubectl get nodes --show-labels
```

Verify Pod placement:

```bash
kubectl get pods -l app=placement-demo -o wide
```

If no eligible node has the required label, the Pod will remain unscheduled.

**7. Spread replicas across nodes**

For workloads that benefit from node-level distribution, use topology spread constraints:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread-demo
  template:
    metadata:
      labels:
        app: spread-demo
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: spread-demo
      containers:
        - name: web
          image: nginx:stable
```

Apply and inspect:

```bash
kubectl apply -f spread-demo.yaml
kubectl get pods -l app=spread-demo -o wide
```

This constraint aims to keep the number of matching Pods balanced across eligible hostname domains, subject to the scheduler's rules. Because `whenUnsatisfiable` is set to `DoNotSchedule`, the scheduler can leave a Pod pending if it cannot satisfy the constraint. With too few eligible nodes or unsuitable topology labels, all replicas may not become scheduled.

Topology spread constraints influence placement; they do not create extra nodes or guarantee that the application is available during every failure.

**8. Observe the difference between a Pod and a container**

List the containers in a Pod:

```bash
kubectl get pod POD_NAME \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

View container statuses:

```bash
kubectl get pod POD_NAME \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{" state="}{.state}{" restarts="}{.restartCount}{"\n"}{end}'
```

For a Pod with multiple containers, Kubernetes schedules the Pod as one unit. Each container has its own process and container state, but containers within the same Pod share the Pod's network namespace and can communicate over `localhost`.

**9. Clean up the practice workloads**

```bash
kubectl delete -f workload-placement.yaml
kubectl delete -f spread-demo.yaml
```

Remove the node label if it is no longer required:

```bash
kubectl label node NODE_NAME workload-tier-
```

The trailing hyphen removes the specified label. Only remove a label if you added it for this exercise and no other workload depends on it.

## Common failures & fixes

- **A Pod remains in `Pending`**
  - **Cause:** No eligible node can satisfy the Pod's resource requests or scheduling constraints.
  - **Fix:** Run `kubectl describe pod POD_NAME` and inspect scheduling events, resource requests, node selectors, affinity rules, taints, and tolerations.
  - **Verify:** Confirm that a suitable node becomes available and the Pod receives a node assignment.

- **The Pod is assigned to a node but its container does not start**
  - **Cause:** The image cannot be pulled, the entrypoint fails, a required volume cannot be mounted, or the container runtime encounters an error.
  - **Fix:** Inspect the Pod's events and container states, then check logs with `kubectl logs POD_NAME`.
  - **Verify:** Confirm that the container starts and reaches the expected state.

- **All replicas run on one node**
  - **Cause:** Kubernetes is allowed to place multiple replicas on one node when no placement rules prevent it.
  - **Fix:** Configure topology spread constraints or pod anti-affinity if node-level distribution is a requirement.
  - **Verify:** Check the assigned nodes using `kubectl get pods -o wide` and confirm that enough eligible nodes exist.

- **A node selector prevents scheduling**
  - **Cause:** No eligible node has the required label.
  - **Fix:** Check `kubectl get nodes --show-labels` and correct the selector or apply the intended label.
  - **Verify:** Confirm that the Pod can be assigned to a node satisfying the selector.

- **The Pod is running but not ready**
  - **Cause:** A readiness probe is failing, the application has not finished initialization, or a required dependency is unavailable.
  - **Fix:** Check `kubectl describe pod POD_NAME` and `kubectl logs POD_NAME`.
  - **Verify:** Confirm that the readiness condition becomes true and the application responds as expected.

- **A Pod disappears after a node failure**
  - **Cause:** Pods are tied to a particular node while they exist, and node failure can make them unavailable. A managing controller may create replacements elsewhere.
  - **Fix:** Inspect the Deployment, ReplicaSet, node conditions, and Pod events. Confirm that other eligible nodes have enough resources.
  - **Verify:** Check that the desired number of replicas becomes available again. Recovery may take time and is not guaranteed if the cluster lacks capacity or the workload has restrictive scheduling rules.

- **A node has free CPU but cannot accept a Pod**
  - **Cause:** The Pod's CPU or memory requests cannot be satisfied alongside existing requests, or other scheduling rules make the node ineligible.
  - **Fix:** Review allocatable resources, allocated requests, taints, affinity rules, and topology constraints.
  - **Verify:** Confirm that the scheduler's reported reason is resolved. Low instantaneous CPU usage alone does not mean that a node can accept another Pod.

- **The Pod IP is reachable from one place but not another**
  - **Cause:** Pod networking, network policies, routing, or the testing location may affect connectivity.
  - **Fix:** Check the cluster's network plugin, Pod IP, relevant NetworkPolicies, and whether the source is inside or outside the cluster.
  - **Verify:** Test connectivity from the appropriate network context. Use a Service or another supported exposure mechanism when a stable application endpoint is required.

- **A local single-node cluster does not demonstrate multi-node placement**
  - **Cause:** Only one eligible node is available.
  - **Fix:** Create a multi-node local cluster with a tool such as kind or Minikube, or use an appropriately configured shared cluster.
  - **Verify:** Run `kubectl get nodes` and confirm that multiple eligible nodes exist before testing distribution constraints.

## References

- [Kubernetes: Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes: Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes: Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes: Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes: Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes: Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Kubernetes: Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
