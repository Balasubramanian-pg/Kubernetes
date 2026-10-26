# Cluster, nodes, control plane

> Phase 2 — Kubernetes Core (highest priority) › Cluster model

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A Kubernetes cluster is a group of machines that work together to run containerized applications. Kubernetes coordinates where applications run, monitors their state, and works to maintain the desired configuration declared by the user.

A Kubernetes cluster consists of two major parts:

- **Control plane:** Makes cluster-wide decisions, exposes the Kubernetes API, and coordinates the desired state of the cluster.
- **Worker nodes:** Run application workloads, primarily through Pods managed by the Kubernetes control plane.

A **node** is a machine that participates in the cluster. It can be a physical machine or a virtual machine. Each node provides CPU, memory, storage, and networking resources for the workloads assigned to it.

The control plane does not normally run application workloads in the same way worker nodes do, although small development clusters and some specialized configurations can run workloads on control-plane nodes.

### How the components work together

Suppose you deploy a data processing application that needs three replicas.

1. You submit a Kubernetes manifest describing three replicas of your application.
2. `kubectl` sends the request to the Kubernetes API server.
3. The API server validates the request and persists the desired state in the cluster's backing store, typically etcd.
4. The scheduler selects suitable nodes for Pods that have not yet been assigned to a node.
5. The control plane's controllers continuously compare the actual state with the desired state.
6. On each selected worker node, the kubelet works with the container runtime to ensure the assigned Pods run.
7. The container runtime starts the containers, while networking components enable communication as configured.
8. Kubernetes continues monitoring the state and attempts to reconcile differences, such as replacing a failed Pod when its managing controller requires the replica count to be maintained.

This is a reconciliation-based system: Kubernetes repeatedly works toward the desired state rather than relying on a single command to perform every step.

### Control plane components

- **kube-apiserver:** The front end of the Kubernetes control plane. It exposes the Kubernetes API, validates and processes requests, and coordinates access to cluster state.
- **etcd:** A distributed key-value store that holds Kubernetes cluster data. It is a critical component for preserving the cluster's state.
- **kube-scheduler:** Selects a suitable node for each newly created Pod that has not yet been scheduled. It considers resource requests, constraints, affinity rules, topology, and other scheduling requirements.
- **kube-controller-manager:** Runs controllers that monitor cluster resources and reconcile actual state with desired state. Examples include controllers that maintain replica counts and manage node-related behavior.
- **cloud-controller-manager:** Integrates Kubernetes with supported cloud-provider functionality, such as cloud load balancers, node information, and cloud storage or networking integrations. It is optional in environments that do not need it.

### Worker node components

- **kubelet:** The node agent that communicates with the API server and ensures Pods assigned to that node are running as specified.
- **Container runtime:** The software responsible for running containers. Kubernetes commonly uses a CRI-compatible runtime such as containerd or CRI-O.
- **kube-proxy:** Implements Service networking rules on many Kubernetes installations. Some cluster networking implementations replace its functionality, so it is not mandatory in every cluster.
- **CNI network plugin:** Provides Pod networking and, depending on the implementation, network policy enforcement. CNI refers to the Container Network Interface ecosystem used by network plugins.

A worker node may run additional components for observability, security, storage, and networking.

## Key points

- **A cluster is not a single machine**
  - A cluster is the overall Kubernetes environment containing the control plane and nodes.
  - The control plane coordinates operations across the cluster.
  - Worker nodes supply the resources used to run application Pods.
  - In a local learning environment, the cluster may consist of a single virtual machine or a single physical machine running all the required components.

- **The control plane manages desired state**
  - Users declare what they want, such as three replicas of an application.
  - Kubernetes controllers observe the current state and take action when it differs from the desired state.
  - The control plane coordinates scheduling and lifecycle management.
  - Kubernetes does not guarantee that every request succeeds. Resource shortages, scheduling constraints, image failures, or infrastructure problems can prevent the desired state from being achieved.

- **The API server is the main entry point**
  - `kubectl` communicates with the Kubernetes API server.
  - The API server authenticates requests, applies authorization and admission controls, and validates API operations.
  - Most cluster management operations go through the API server.
  - A functioning API server does not automatically mean that every worker node or application is healthy.

- **etcd stores cluster state**
  - Kubernetes uses etcd as its backing key-value store in standard control-plane architectures.
  - The API server interacts with etcd to persist cluster data.
  - etcd availability is critical to normal cluster management operations.
  - etcd is not an application database intended to hold business records generated by your workloads.
  - Production clusters require secure etcd access, backups, and a reliable recovery strategy.

- **The scheduler chooses where Pods run**
  - The scheduler evaluates eligible nodes and selects one for an unscheduled Pod.
  - It considers resource requests, node selectors, affinity and anti-affinity, taints and tolerations, topology constraints, and other scheduling rules.
  - The scheduler does not normally start the container itself. It assigns the Pod to a node, after which the node's kubelet and container runtime work to run it.
  - A Pod can remain in `Pending` if no node satisfies its scheduling requirements.

- **Controllers continuously reconcile state**
  - Controllers observe resources through the Kubernetes API.
  - A Deployment controller helps maintain the desired ReplicaSet, and a ReplicaSet controller maintains the requested number of matching Pods.
  - If a managed Pod disappears, the relevant controller may create a replacement.
  - A replacement Pod is a new Pod object, not necessarily the original Pod restored in place.
  - Controllers do not guarantee application-level correctness. A Pod can be running while its application is malfunctioning.

- **The kubelet operates at node level**
  - The kubelet receives information about Pods assigned to its node.
  - It works with the container runtime to start and monitor containers.
  - It reports node and Pod status back through the Kubernetes API.
  - If a node becomes unavailable, the control plane may mark it unhealthy and initiate recovery behavior, depending on the workload controller and configured timing.

- **The container runtime executes containers**
  - Kubernetes does not directly run containers by itself.
  - The kubelet communicates with the runtime through the Container Runtime Interface (CRI).
  - Common runtimes include containerd and CRI-O.
  - Docker Engine is not itself the standard runtime integration used by modern Kubernetes through the CRI, although Docker-built images can run on compatible Kubernetes runtimes.

- **A Service is different from a node**
  - A node is a machine in the cluster.
  - A Pod is the smallest deployable Kubernetes workload unit.
  - A Service provides a stable network endpoint for a logical set of Pods.
  - A Deployment manages replicated application Pods through a ReplicaSet.
  - Do not confuse the node's IP address, a Pod IP, and a Service IP.

- **Control-plane availability affects cluster management**
  - If the API server is unavailable, `kubectl` operations that require it may fail.
  - If etcd is unavailable, the API server may be unable to read or persist cluster state normally.
  - Existing containers may continue running temporarily even when the control plane is unavailable, but scheduling, reconciliation, and management capabilities can be impaired.
  - High-availability control planes typically use multiple control-plane instances and an appropriately designed etcd deployment.

- **Worker node failure and application failure are different**
  - An application process can fail while its node remains healthy.
  - A node can fail while the application was previously healthy.
  - Kubernetes handles these scenarios through different mechanisms, including container restart behavior, workload controllers, node health monitoring, and scheduling.
  - Recovery is not always immediate and depends on probes, controllers, resource availability, and cluster configuration.

- **Managed Kubernetes hides some infrastructure**
  - Services such as Amazon EKS, Azure Kubernetes Service (AKS), and Google Kubernetes Engine (GKE) manage some or all control-plane infrastructure for the user.
  - Users still need to understand nodes, Pods, scheduling, workloads, networking, and resource constraints.
  - The amount of direct access to control-plane components and etcd varies by provider and cluster configuration.

## Commands / manifests

**1. Check whether `kubectl` can reach the cluster**

```bash
kubectl version
```

Display the current context:

```bash
kubectl config current-context
```

List configured contexts:

```bash
kubectl config get-contexts
```

Check cluster endpoints:

```bash
kubectl cluster-info
```

These commands help verify that your local Kubernetes client is configured to communicate with the intended cluster.

**2. List cluster nodes**

```bash
kubectl get nodes
```

Display additional information:

```bash
kubectl get nodes -o wide
```

Typical output:

```text
NAME           STATUS   ROLES           AGE   VERSION
control-plane  Ready    control-plane   10d   v1.xx.x
worker-1       Ready    <none>          10d   v1.xx.x
worker-2       Ready    <none>          10d   v1.xx.x
```

This is illustrative output, not a guarantee of the names or versions in your cluster.

Interpret the columns:

- `NAME`: The Kubernetes node name.
- `STATUS`: Whether the node is reported as `Ready`, `NotReady`, or another status.
- `ROLES`: Labels used to identify roles such as `control-plane`; worker nodes may show `<none>`.
- `AGE`: How long the node object has existed.
- `VERSION`: The kubelet version reported for that node.

A `Ready` node is considered ready by Kubernetes, but that alone does not guarantee sufficient resources for every workload or prove that all applications on the node are healthy.

**3. Inspect a specific node**

```bash
kubectl describe node NODE_NAME
```

Replace `NODE_NAME` with a name returned by `kubectl get nodes`.

The output includes:

- Node labels and annotations
- Taints
- Capacity and allocatable resources
- Conditions such as `Ready`, memory pressure, and disk pressure
- Pod resource requests and limits
- Events associated with the node

Check node labels:

```bash
kubectl get nodes --show-labels
```

Retrieve node capacity and allocatable resources as YAML:

```bash
kubectl get node NODE_NAME -o yaml
```

Capacity represents the node's total reported resources. Allocatable represents the resources available for Pods after accounting for resources reserved for the operating system, Kubernetes components, and other configured reservations.

**4. List Pods and identify the nodes running them**

```bash
kubectl get pods -A -o wide
```

The `-A` option includes all namespaces, while `-o wide` displays additional information such as the Pod IP and node name.

List Pods in a specific namespace:

```bash
kubectl get pods -n default -o wide
```

Inspect a Pod:

```bash
kubectl describe pod POD_NAME -n NAMESPACE
```

The output helps investigate scheduling failures, container states, image-pull errors, and recent events.

**5. Examine control-plane-related Pods**

In many self-managed clusters, control-plane components run as static Pods in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system -o wide
```

You may see Pods associated with the API server, scheduler, controller manager, and etcd.

This layout is common but not universal. Managed Kubernetes providers may hide control-plane Pods because the provider operates the control plane outside the worker nodes visible to the user.

Check the cluster's available API resources:

```bash
kubectl api-resources
```

Check API server reachability:

```bash
kubectl get --raw='/readyz'
```

A successful readiness response indicates that the API server reports itself ready. It does not prove that every node or workload is healthy.

**6. Understand cluster components through a Deployment**

Create a file named `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
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
kubectl apply -f nginx-deployment.yaml
```

Check the Deployment:

```bash
kubectl get deployment nginx-demo
```

Check the ReplicaSet:

```bash
kubectl get replicasets
```

Check the Pods and their node assignments:

```bash
kubectl get pods -l app=nginx-demo -o wide
```

Describe the Deployment:

```bash
kubectl describe deployment nginx-demo
```

The `replicas: 3` setting declares that the Deployment should maintain three replicas. The scheduler assigns eligible Pods to nodes, and node agents work to run them. Actual placement depends on available resources, scheduling rules, and cluster configuration.

The resource requests in this example influence scheduling. The limits constrain container resource usage according to Kubernetes' resource-management behavior. They do not reserve the entire limit as guaranteed capacity on a node.

Clean up the Deployment:

```bash
kubectl delete -f nginx-deployment.yaml
```

**7. Inspect node resource usage**

If the Kubernetes Metrics API is available, commonly through Metrics Server:

```bash
kubectl top nodes
```

Inspect Pod resource usage:

```bash
kubectl top pods -A
```

These commands report observed CPU and memory usage, not the full capacity or every resource constraint in the cluster. They may fail if the metrics provider is unavailable or not yet reporting data.

For configured resource requests and limits, inspect the workload manifest or describe the Pod:

```bash
kubectl describe pod POD_NAME -n NAMESPACE
```

**8. Practice with a local Kubernetes cluster**

For learning, tools such as Minikube and kind can create local clusters.

With Minikube:

```bash
minikube start
kubectl get nodes -o wide
kubectl cluster-info
```

With kind, after installing it:

```bash
kind create cluster --name learning
kubectl get nodes -o wide
```

A kind cluster commonly runs its Kubernetes nodes as Docker containers. A single-node local cluster can still demonstrate Kubernetes APIs, scheduling, Pods, and controllers, but it does not provide the same failure isolation as a production cluster with multiple independent machines.

Clean up when finished:

```bash
minikube delete
```

Or, for kind:

```bash
kind delete cluster --name learning
```

Use the cleanup command only for the local cluster you intend to remove.

## Common failures & fixes

- **`kubectl get nodes` reports that it cannot connect to the server**
  - **Cause:** The cluster is stopped, the current context is incorrect, the API endpoint is unreachable, or authentication has failed.
  - **Fix:** Check `kubectl config current-context`, `kubectl config get-contexts`, and `kubectl cluster-info`. Start the intended local cluster if necessary.
  - **Verify:** Run `kubectl get nodes` again after confirming the endpoint and credentials.

- **A node is in `NotReady` state**
  - **Cause:** The kubelet is unhealthy or disconnected, networking is broken, resource pressure exists, or the node has an underlying infrastructure problem.
  - **Fix:** Run `kubectl describe node NODE_NAME` and inspect node conditions and events. Investigate kubelet and runtime logs through the node's operating system or provider tools.
  - **Verify:** Confirm that the node becomes `Ready` and that its conditions and recent events no longer indicate the underlying problem.

- **A Pod remains in `Pending`**
  - **Cause:** No eligible node satisfies the Pod's requirements, resource requests cannot be accommodated, or scheduling constraints prevent placement.
  - **Fix:** Run `kubectl describe pod POD_NAME -n NAMESPACE` and review scheduling events, resource requests, node selectors, affinity rules, taints, and tolerations.
  - **Verify:** Confirm that the scheduler assigns the Pod to a node and that the Pod progresses toward `Running`.

- **A Pod is assigned to a node but does not start**
  - **Cause:** The image cannot be pulled, the container command fails, required configuration is missing, or storage cannot be mounted.
  - **Fix:** Inspect the Pod with `kubectl describe pod` and check container logs using `kubectl logs`.
  - **Verify:** Confirm that the container starts successfully and the Pod reaches the expected state.

- **A Deployment has fewer ready replicas than requested**
  - **Cause:** Pods are unschedulable, containers are crashing, readiness probes fail, or the cluster lacks resources.
  - **Fix:** Check `kubectl describe deployment`, `kubectl get pods -o wide`, and the affected Pods' events and logs.
  - **Verify:** Confirm that the Deployment's ready replica count reaches the desired count.

- **The control plane is reachable but workloads are failing**
  - **Cause:** API server availability does not guarantee healthy nodes, functioning networking, available storage, or healthy application dependencies.
  - **Fix:** Check node status, Pod status, events, resource usage, and application logs separately.
  - **Verify:** Test the affected workload and its dependencies rather than relying only on `kubectl cluster-info`.

- **`kubectl top nodes` fails**
  - **Cause:** Metrics Server or another supported metrics provider is missing, unavailable, or not yet reporting data.
  - **Fix:** Check whether the cluster's metrics provider is installed and healthy.
  - **Verify:** Retry after the provider begins returning metrics. Use `kubectl describe node` to inspect capacity and allocatable resources in the meantime.

- **Control-plane Pods are not visible**
  - **Cause:** The cluster is managed by a cloud provider that does not expose its control-plane components as Pods in the user's cluster view.
  - **Fix:** Use the provider's cluster health and diagnostics tools alongside the Kubernetes API.
  - **Verify:** Confirm the cluster's management model before assuming that missing control-plane Pods indicate a failure.

- **The cluster cannot recover a workload after a node failure**
  - **Cause:** Recovery may be delayed by node-failure detection, controller timing, unavailable capacity, restrictive scheduling rules, or workload configuration.
  - **Fix:** Inspect node conditions, Pod events, Deployment replica status, and the availability of eligible nodes.
  - **Verify:** Confirm that replacement Pods are scheduled and become ready. Stateful workloads may require additional storage and recovery considerations.

## References

- [Kubernetes: Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes: Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes: Control Plane and Node Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes: kube-apiserver](https://kubernetes.io/docs/concepts/architecture/#kube-apiserver)
- [Kubernetes: etcd](https://kubernetes.io/docs/concepts/architecture/#etcd)
- [Kubernetes: kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Kubernetes: Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Kubernetes: kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Minikube: Start a Cluster](https://minikube.sigs.k8s.io/docs/start/)
- [kind: Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
