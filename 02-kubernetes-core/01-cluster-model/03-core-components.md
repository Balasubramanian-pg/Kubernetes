# Core components

> Phase 2 — Kubernetes Core (highest priority) › Cluster model

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Kubernetes is made up of **control plane components** that manage the cluster and **worker node components** that run workloads. Each component has a specific responsibility, and they coordinate through the Kubernetes API.

A useful mental model is:

- **API server:** The entry point for Kubernetes operations.
- **etcd:** The persistent key-value store for cluster state.
- **Scheduler:** Decides which node should run a newly created Pod.
- **Controller manager:** Continuously works to make actual cluster state match the desired state.
- **Kubelet:** Ensures the Pods assigned to its node are running as specified.
- **Kube-proxy:** Implements Service networking behavior on nodes in many Kubernetes installations.

### 1. API server (`kube-apiserver`)

The API server is the **front door to the Kubernetes control plane**. Tools such as `kubectl`, controllers, and other Kubernetes clients communicate with the cluster through its API.

Its responsibilities include:

- Exposing the Kubernetes REST API.
- Authenticating requests to establish who is making them.
- Authorizing requests using policies such as Role-Based Access Control (RBAC).
- Validating API requests and resource specifications.
- Applying admission controls, which can validate or modify requests before persistence.
- Reading and writing Kubernetes resource state through the configured storage layer, normally etcd.

For example, when you run:

```bash
kubectl apply -f deployment.yaml
```

`kubectl` sends the Deployment definition to the API server. The API server processes the request and persists the accepted resource state. Other components then react to that state.

**Important:** The API server does not directly start your application container. It exposes the API through which the components responsible for scheduling and execution coordinate.

### 2. etcd

etcd is a distributed, consistent key-value store used by Kubernetes to persist its cluster state.

It stores Kubernetes API data, including resource definitions and metadata such as:

- Deployments, ReplicaSets, and Pods.
- Services, ConfigMaps, and Secrets.
- Namespaces and RBAC objects.
- Cluster configuration and other API-managed state.

The API server uses etcd as its backing store. Kubernetes clients normally interact with etcd indirectly through the API server rather than connecting to etcd themselves.

**Why it matters:**

- It provides persistent storage for cluster state.
- Its consistency properties help the control plane coordinate changes reliably.
- Backups are important for disaster recovery because etcd contains critical cluster metadata and configuration.

**Important distinction:** etcd stores the desired and recorded state of Kubernetes resources. It does not store the actual running application process or serve as the storage location for your application's ordinary container filesystem.

If etcd becomes unavailable, API operations that require storage access can fail or become unavailable. Existing workloads may continue running temporarily, but the control plane's ability to manage and recover the cluster can be impaired.

### 3. Scheduler (`kube-scheduler`)

The scheduler selects an appropriate worker node for a newly created Pod that does not yet have a node assignment.

It considers factors such as:

- CPU and memory requests.
- Node readiness and available resources.
- Node selectors and node affinity.
- Taints and tolerations.
- Pod affinity and anti-affinity.
- Topology spread constraints.
- Scheduling policies and other configured constraints.

The scheduler typically works in two broad stages:

1. **Filtering:** Identify nodes that satisfy the Pod's scheduling requirements.
2. **Scoring:** Rank suitable nodes and select one according to the scheduling framework.

After choosing a node, the scheduler records the assignment through the API server. The kubelet on that node can then act on the Pod specification.

**Important:** The scheduler selects a node. It does not download the image, start the container, or monitor the application's health. Those responsibilities belong primarily to the node's execution components and other Kubernetes controllers.

If no node satisfies the scheduling requirements, the Pod can remain in the `Pending` phase with an event such as `FailedScheduling`.

### 4. Controller manager (`kube-controller-manager`)

The controller manager runs multiple controllers that continuously compare the current state of the cluster with the desired state declared through the Kubernetes API.

A controller follows a reconciliation loop:

1. Observe the current state.
2. Compare it with the desired state.
3. Identify the difference.
4. Take action to reduce that difference.
5. Repeat as the cluster changes.

For example, if a Deployment specifies three replicas, the Deployment controller and ReplicaSet controller work together to maintain that replica count. If a Pod disappears, the controllers detect the difference and work toward creating a replacement.

Other controllers manage concerns such as node health, namespaces, service accounts, and endpoints, depending on the Kubernetes configuration.

**Important:** The controller manager is not one single controller. It runs multiple controllers, each responsible for particular resource types or cluster behaviors.

### 5. Kubelet

The kubelet is the node-level agent that runs on each worker node, and commonly on control-plane nodes as well.

Its responsibilities include:

- Registering the node and reporting node status.
- Watching for Pod specifications assigned to its node.
- Working with the container runtime to create and manage containers.
- Ensuring the containers in assigned Pods are running according to their specifications.
- Running configured startup, liveness, and readiness probes.
- Reporting Pod and container status to the API server.

The kubelet uses the container runtime through the Container Runtime Interface (CRI). Common runtimes include containerd and CRI-O.

For example, when a Pod is assigned to a node, the kubelet processes its specification and coordinates with the runtime to pull images when necessary and start the containers.

**Important:** The kubelet does not decide which node should receive a Pod. The scheduler makes that decision. The kubelet is responsible for carrying out the assigned work on its own node.

### 6. Kube-proxy

Kube-proxy is a node-level component that implements Kubernetes Service networking behavior in many clusters. It watches relevant Service and EndpointSlice information and configures network rules so traffic can reach the appropriate backends.

For example, a Service may provide a stable virtual IP and port for a group of application Pods. Kube-proxy can configure packet-forwarding rules that direct traffic sent to that Service toward an eligible backend Pod.

Depending on the installation, kube-proxy can use networking mechanisms such as:

- `iptables`
- IPVS, where supported and configured

**Important:** Kube-proxy does not run application containers, schedule Pods, or act as a conventional HTTP reverse proxy. It primarily implements Service traffic forwarding at the network level.

Some Kubernetes distributions replace kube-proxy's functionality with an alternative networking implementation, so a cluster may not run kube-proxy at all.

### 7. Container runtime

The container runtime is the software responsible for running containers on a node.

Examples include:

- containerd
- CRI-O

Its responsibilities include pulling and managing container images, creating containers, starting and stopping them, and managing their execution environment.

The kubelet communicates with the runtime through the CRI. The runtime then works with lower-level components, such as an OCI runtime, to execute containers.

**Remember the distinction:**

- The scheduler chooses the node.
- The kubelet manages the assigned Pod on that node.
- The container runtime runs the containers.

### How the components work together

Suppose you submit a Deployment requesting three replicas of a web application.

1. You submit a manifest using `kubectl`.
2. The API server authenticates, authorizes, validates, and processes the request.
3. The accepted resource state is persisted in etcd.
4. The Deployment controller observes the Deployment and ensures the required ReplicaSet exists.
5. The ReplicaSet controller works toward creating three Pod objects.
6. The scheduler assigns each unscheduled Pod to a suitable node.
7. The kubelet on each selected node observes the assigned Pod.
8. The kubelet coordinates with the container runtime to start the containers.
9. The kubelet reports status and probe results.
10. The control plane continues reconciling state if Pods fail or the desired replica count changes.
11. If a Service selects the application Pods, the cluster's Service networking implementation routes traffic to eligible backends. In installations using kube-proxy, it helps implement that forwarding behavior.

The components coordinate through the API server, but they do not all execute as a single linear program. Controllers and node agents continuously observe changes and react to them.

## Key points

- **Control plane vs worker node:** The control plane manages the cluster; worker nodes run application workloads. Some installations also schedule workloads on control-plane nodes.
- **API server:** The main API endpoint through which Kubernetes components and clients interact with cluster resources.
- **etcd:** The persistent backing store for Kubernetes API state.
- **Scheduler:** Selects a suitable node for each unscheduled Pod.
- **Controller manager:** Runs controllers that reconcile current state toward desired state.
- **Kubelet:** Manages assigned Pods on its node and coordinates with the container runtime.
- **Kube-proxy:** Implements Service networking behavior in installations that use it.
- **Container runtime:** Pulls images and runs containers on the node.
- **Desired state vs actual state:** A manifest expresses what should exist; Kubernetes controllers continually work toward making reality match it.
- **Declarative management:** You usually declare the desired configuration through Kubernetes resources instead of manually instructing every component how to execute each step.
- **Components communicate through the API:** The API server is the coordination point for most Kubernetes control-plane interactions.
- **Scheduling is not execution:** A successfully scheduled Pod can still fail if its image cannot be pulled, its volume cannot mount, or the container cannot start.
- **`Pending` does not always mean scheduling failure:** A Pod can remain Pending while waiting for scheduling, storage setup, or other prerequisites.
- **Kube-proxy is not mandatory in every cluster:** Some networking implementations replace its Service-forwarding functionality.
- **etcd is critical infrastructure:** Restrict access to it, protect its credentials, and maintain tested backups.
- **Node components can be healthy while an application is unhealthy:** Node readiness, Pod readiness, and application health are different signals.

### Quick comparison

| Component | Primary responsibility | Typical location |
|---|---|---|
| API server | Kubernetes API, validation, access control | Control plane |
| etcd | Persistent cluster state | Control plane |
| Scheduler | Selects a node for unscheduled Pods | Control plane |
| Controller manager | Reconciles resources toward desired state | Control plane |
| Kubelet | Manages assigned Pods | Each node |
| Kube-proxy | Implements Service forwarding rules, when used | Each node |
| Container runtime | Runs containers | Each node |

## Commands / manifests

### 1. Inspect cluster nodes

```bash
kubectl get nodes
```

Lists the nodes known to the API server and their reported readiness status.

```bash
kubectl describe node <node-name>
```

Displays detailed node information, including capacity, allocatable resources, taints, conditions, and events.

Replace `<node-name>` with an actual name from `kubectl get nodes`.

### 2. Inspect Kubernetes system components

```bash
kubectl get pods -n kube-system
```

Lists Pods in the `kube-system` namespace. Depending on the distribution, this may include CoreDNS, kube-proxy, and control-plane components.

```bash
kubectl get pods -n kube-system -o wide
```

Also displays information such as the node on which each Pod is running.

**Note:** In kubeadm-based clusters, many control-plane components run as static Pods. Managed Kubernetes services and other distributions may operate some control-plane components outside the visible cluster Pod list.

### 3. Inspect API resources

```bash
kubectl cluster-info
```

Displays cluster control-plane and core service information.

```bash
kubectl api-resources
```

Lists the resource types exposed by the Kubernetes API.

```bash
kubectl get componentstatuses
```

This is a legacy command and is deprecated in modern Kubernetes. Do not rely on it for comprehensive control-plane health checks.

### 4. Inspect a Pod and its events

```bash
kubectl get pods -A -o wide
```

Lists Pods across namespaces and shows which nodes they are assigned to.

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Shows the Pod specification, scheduling information, container state, probes, and recent events.

```bash
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Lists namespace events in creation-time order, helping you identify recent scheduling, image-pull, and startup failures.

### 5. Create a test Deployment

Save the following manifest as `web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-demo
  template:
    metadata:
      labels:
        app: web-demo
    spec:
      containers:
        - name: web
          image: nginx:1.28
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
```

Apply it:

```bash
kubectl apply -f web-deployment.yaml
```

Check the Deployment, ReplicaSet, and Pods:

```bash
kubectl get deployment web-demo
kubectl get replicasets
kubectl get pods -l app=web-demo -o wide
```

Watch Pod status as Kubernetes works toward the desired state:

```bash
kubectl get pods -l app=web-demo -w
```

Press `Ctrl+C` to stop watching.

The image tag is an example. If that tag is unavailable in your environment, choose a valid tag from the official NGINX image repository.

### 6. Observe scheduling

```bash
kubectl describe pod <pod-name>
```

Look for the `Node` field and the Events section. A scheduled Pod has a node assignment. An unscheduled Pod may show an event explaining why no suitable node was found.

You can also inspect node placement with:

```bash
kubectl get pods -l app=web-demo -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,PHASE:.status.phase
```

### 7. Inspect container runtime information

```bash
kubectl describe node <node-name>
```

Check the node's runtime information and conditions.

If you have authorized shell access to a node with `crictl` installed, you can inspect the CRI runtime directly:

```bash
sudo crictl info
sudo crictl ps
sudo crictl pods
```

These commands require node-level access and a correctly configured CRI endpoint. They are not normally run from an ordinary workstation.

### 8. Inspect Service networking

Create a Service that selects the Deployment's Pods:

```bash
kubectl expose deployment web-demo \
  --name=web-demo-svc \
  --port=80 \
  --target-port=80
```

Inspect it:

```bash
kubectl get service web-demo-svc
kubectl get endpointslices \
  -l kubernetes.io/service-name=web-demo-svc
```

EndpointSlices show the backend endpoints associated with the Service. If the Service has no usable endpoints, investigate its selector, Pod labels, and readiness status.

For a local test, forward a local port to the Service:

```bash
kubectl port-forward service/web-demo-svc 8080:80
```

In another terminal, run:

```bash
curl http://localhost:8080
```

This tests access through `kubectl port-forward`. It does **not** prove that external Service routing or kube-proxy is working, because port forwarding uses a different access path.

### 9. Clean up the test resources

```bash
kubectl delete service web-demo-svc
kubectl delete deployment web-demo
```

This removes the resources created in the exercise.

## Common failures & fixes

| Symptom | Likely cause | What to check |
|---|---|---|
| `kubectl` cannot connect to the cluster | Incorrect kubeconfig, unreachable API server, or cluster unavailable | `kubectl config current-context`, `kubectl cluster-info`, network connectivity |
| `Forbidden` error | The authenticated identity lacks permission | RBAC Roles, ClusterRoles, RoleBindings, ClusterRoleBindings |
| `Pending` Pod with `FailedScheduling` | Insufficient resources, node selector mismatch, untolerated taint, or affinity constraints | `kubectl describe pod <pod-name>` and `kubectl describe nodes` |
| `Pending` Pod without a scheduling event | Scheduling may be complete but storage or another prerequisite may be blocking progress | `kubectl describe pod <pod-name>`, PersistentVolumeClaims, events |
| `ImagePullBackOff` | Invalid image name or tag, registry authentication failure, or network problem | Pod events, image reference, registry credentials, node connectivity |
| `CrashLoopBackOff` | Container repeatedly exits or fails to start | `kubectl logs <pod-name>`, `kubectl logs <pod-name> --previous`, container configuration |
| Node is `NotReady` | Kubelet failure, network problem, resource pressure, or runtime issue | `kubectl describe node <node-name>`, node conditions, node service logs |
| Deployment has fewer ready replicas than requested | Pods cannot schedule, start, or pass readiness checks | `kubectl describe deployment web-demo`, `kubectl get pods`, Pod events and logs |
| Service has no usable backends | Selector mismatch or all matching Pods are unready | `kubectl describe service <service-name>`, Pod labels, EndpointSlices |
| API operations fail while workloads still run | Control-plane or API server problem, possibly involving etcd | API server health, control-plane logs, cluster-specific monitoring |
| `kubectl get pods -n kube-system` does not show a control-plane component | The distribution may run that component outside the cluster's visible Pod resources | Consult the distribution's control-plane architecture and administrative tools |
| No kube-proxy Pod exists | The cluster may use an alternative Service networking implementation | Check the cluster distribution and networking provider configuration |

### Troubleshooting commands

**Check API access and context:**

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

**Investigate scheduling:**

```bash
kubectl describe pod <pod-name>
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl describe node <node-name>
```

**Investigate container startup:**

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl describe pod <pod-name>
```

For a Pod with multiple containers, specify the container using `-c <container-name>`.

**Check control-plane health:**

```bash
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
```

These requests query API server health endpoints. Access depends on your credentials and cluster configuration. A successful API server health response does not guarantee that every control-plane component or application is healthy.

In a kubeadm-managed cluster where you have control-plane host access, additional checks may include:

```bash
sudo crictl ps
sudo journalctl -u kubelet --since "15 minutes ago"
```

Use the platform's own logging and monitoring tools for managed Kubernetes services. Avoid restarting control-plane services or modifying etcd directly without understanding the impact and following the cluster operator's procedures.

## References

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
- [etcd documentation](https://etcd.io/docs/)
- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- [Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/containers/cri/)
- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [Debugging Kubernetes Clusters](https://kubernetes.io/docs/tasks/debug/)
