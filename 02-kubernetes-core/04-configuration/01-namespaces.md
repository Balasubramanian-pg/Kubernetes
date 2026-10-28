# Namespaces

> Phase 2 — Kubernetes Core (highest priority) › Configuration

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A Kubernetes Namespace is a logical grouping mechanism used to organize and isolate resources within a single Kubernetes cluster.

Namespaces allow teams to separate workloads into different environments, projects, or organizational units without requiring a separate cluster for every environment.

For example, a team may organize its workloads using:

- `development`: Applications under active development.
- `testing`: Applications undergoing validation.
- `production`: Applications serving live workloads.

A cluster can contain multiple namespaces, each with its own Deployments, Pods, Services, ConfigMaps, Secrets, and other namespaced resources.

```text
Kubernetes Cluster
|
+-- development
|   +-- frontend Deployment
|   +-- backend Deployment
|   +-- database Service
|
+-- testing
|   +-- frontend Deployment
|   +-- backend Deployment
|   +-- database Service
|
+-- production
    +-- frontend Deployment
    +-- backend Deployment
    +-- database Service
```

Namespaces help organize resources and provide boundaries for applying access controls, resource quotas, and policies.

### Why namespaces are useful

Without namespaces, resources of the same type must have unique names within the same namespace. Namespaces allow different environments to use the same resource names.

For example, both `development` and `production` can contain a Deployment named `web`.

```text
development/web
production/web
```

These are separate Kubernetes resources because they belong to different namespaces.

### Namespaces and environment separation

Namespaces can be used to separate development, testing, and production workloads.

For example:

| Namespace | Purpose |
|---|---|
| `dev` | Development workloads and experimental deployments |
| `test` | Integration testing and validation |
| `staging` | Pre-production testing |
| `prod` | Production workloads |

Each namespace can have its own resource quotas, access permissions, and policies.

However, **Namespaces alone do not provide complete environment isolation.**

For example, workloads in different namespaces can still communicate over the network unless network policies or other controls restrict them. Namespaces also share the same underlying cluster infrastructure.

For stronger isolation, organizations may use separate clusters, dedicated node pools, network policies, RBAC, and other security controls.

### Namespaced vs cluster-scoped resources

Not every Kubernetes resource belongs to a namespace.

**Namespaced resources** exist within a particular namespace.

Examples:

- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- Jobs
- CronJobs
- PersistentVolumeClaims

**Cluster-scoped resources** exist at the cluster level and are not assigned to individual namespaces.

Examples:

- Nodes
- Namespaces
- PersistentVolumes
- StorageClasses
- ClusterRoles
- ClusterRoleBindings
- CustomResourceDefinitions

This distinction matters because a namespace cannot be used to organize or isolate a cluster-scoped resource in the same way as a Deployment or Pod.

### Default namespaces

Kubernetes creates several namespaces when a cluster is initialized.

| Namespace | Purpose |
|---|---|
| `default` | Namespace used when no other namespace is specified |
| `kube-system` | Resources used by Kubernetes system components |
| `kube-public` | Namespace intended for publicly readable cluster information |
| `kube-node-lease` | Stores Lease objects used for node heartbeat and availability tracking |

The `default` namespace does not mean production or development. It is simply the namespace used when a command or workload does not specify another namespace.

For real projects, creating explicit namespaces is generally better than placing all workloads in `default`.

### Namespace and resource naming

The same resource name can exist in multiple namespaces.

For example:

```text
dev/web-service
test/web-service
prod/web-service
```

When referencing a resource, Kubernetes commands can specify the namespace using:

```bash
-n prod
```

or:

```bash
--namespace=prod
```

For Service DNS, a Service can be referenced using a namespace-qualified name:

```text
web-service.prod
```

Or a fully qualified Service name:

```text
web-service.prod.svc.cluster.local
```

The cluster domain may differ from `cluster.local`.

## Key points

- Namespaces provide logical organization and scoped names for resources inside a Kubernetes cluster.
- Namespaces are commonly used to separate environments, teams, applications, or projects.
- Resource names only need to be unique within their namespace, not across the entire cluster.
- Namespaces can be used with RBAC, ResourceQuotas, LimitRanges, and NetworkPolicies.
- A namespace is not a separate Kubernetes cluster.
- Namespaces alone do not guarantee network isolation or complete security isolation.
- Namespaced resources belong to a particular namespace.
- Cluster-scoped resources do not belong to any namespace.
- Kubernetes uses the `default` namespace when no namespace is specified.
- `kubectl -n <namespace>` scopes a command to a particular namespace.
- A Pod's namespace can be specified in its manifest using `metadata.namespace`.
- A Deployment creates Pods in the namespace where the Deployment exists.
- Services in different namespaces can use the same name.
- Service DNS supports cross-namespace discovery using namespace-qualified names.
- Resource quotas can restrict the total resource consumption of a namespace.
- RBAC can restrict which users or service accounts can access resources in a namespace.
- NetworkPolicies can restrict network communication between Pods and namespaces when supported by the cluster's networking implementation.
- Deleting a namespace deletes its namespaced resources, so namespace deletion should be treated as a potentially destructive operation.
- Namespaces are useful for organizing environments, but separate clusters may be necessary for stronger isolation.

## Commands / manifests

### 1. List namespaces

```bash
kubectl get namespaces
```

Short form:

```bash
kubectl get ns
```

This lists the namespaces in the current cluster.

### 2. Create a namespace

```bash
kubectl create namespace development
```

Create additional environment namespaces:

```bash
kubectl create namespace testing
kubectl create namespace production
```

### 3. Get resources in a namespace

```bash
kubectl get pods -n development
```

List multiple resource types:

```bash
kubectl get deployments,services,pods -n development
```

List Pods across all namespaces:

```bash
kubectl get pods -A
```

### 4. Set the current namespace for a context

First inspect the current context:

```bash
kubectl config current-context
```

Set the namespace for the current context:

```bash
kubectl config set-context --current --namespace=development
```

Now commands such as:

```bash
kubectl get pods
```

will use the `development` namespace by default for that context.

This changes the namespace setting in the local kubeconfig context. It does not create or delete a namespace.

To switch back to `default`:

```bash
kubectl config set-context --current --namespace=default
```

### 5. Deploy an application into a namespace

Create a Deployment:

```bash
kubectl create deployment web \
  --image=nginx:stable \
  -n development
```

Expose it through a Service:

```bash
kubectl expose deployment web \
  --name=web-service \
  --port=80 \
  --target-port=80 \
  -n development
```

Verify:

```bash
kubectl get deployments,pods,services -n development
```

### 6. Create a namespace using YAML

Save the following manifest as `development-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

Apply it:

```bash
kubectl apply -f development-namespace.yaml
```

Using a manifest is useful when namespace creation needs to be version-controlled and repeatable.

### 7. Deploy resources into a namespace using YAML

Save the following as `web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: development
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f web-deployment.yaml
```

The Deployment and its Pods will be created in the `development` namespace.

### 8. Create a namespace with resource quotas

A ResourceQuota can limit the total resource consumption of a namespace.

Example `resource-quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "20"
```

Apply it:

```bash
kubectl apply -f resource-quota.yaml
```

Inspect quota usage:

```bash
kubectl get resourcequota -n development
kubectl describe resourcequota development-quota -n development
```

This quota limits aggregate resource requests, limits, and Pod count in the namespace. Individual containers may also need appropriate resource requests and limits.

### 9. Inspect namespace details

```bash
kubectl describe namespace development
```

Check labels:

```bash
kubectl get namespace development --show-labels
```

Add a label:

```bash
kubectl label namespace development environment=dev
```

Labels can be used by policies, automation, and organizational tooling.

### 10. Delete a namespace

```bash
kubectl delete namespace development
```

This deletes the namespace and its namespaced resources.

Use caution in production. Namespace deletion can remove Deployments, Pods, Services, Secrets, ConfigMaps, Jobs, and PersistentVolumeClaims contained in the namespace.

PersistentVolumes and other cluster-scoped resources are not automatically namespaced, though deleting a PersistentVolumeClaim may affect the associated storage depending on its reclaim policy and storage configuration.

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `namespaces "dev" not found` | Namespace does not exist or name is incorrect | Run `kubectl get namespaces` and create the namespace if needed |
| `pods not found` | Command is running against the wrong namespace | Specify `-n <namespace>` or inspect all namespaces using `kubectl get pods -A` |
| Resource created in `default` unexpectedly | Namespace was not specified in the command or manifest | Use `-n <namespace>` or set `metadata.namespace` in YAML |
| Cannot access resources in another namespace | RBAC permissions are restricted | Check the user's RoleBindings, ClusterRoleBindings, and namespace permissions |
| Pod creation fails due to quota | Namespace ResourceQuota has been exceeded | Inspect `kubectl describe resourcequota -n <namespace>` and adjust resource consumption or quota |
| Service cannot discover another Service | Incorrect Service name or namespace | Use the namespace-qualified DNS name, such as `web-service.prod` |
| Namespace remains in `Terminating` state | Resources or finalizers are preventing deletion | Inspect namespace conditions and remaining resources before resolving finalizers |
| Cannot apply a manifest | Namespace is missing and the resource is namespaced | Create the namespace first or include a Namespace manifest in the deployment workflow |
| Same resource name causes confusion | Identical names exist in different namespaces | Include namespace in commands and resource references |
| Workloads in different namespaces can still communicate | Namespace boundaries do not automatically block network traffic | Configure NetworkPolicies and verify CNI support |
| Production workloads can be modified by developers | RBAC permissions are too broad | Use namespace-scoped Roles and RoleBindings with least privilege |

### Troubleshooting sequence

List namespaces:

```bash
kubectl get namespaces
```

List resources in the target namespace:

```bash
kubectl get all -n development
```

Inspect the namespace:

```bash
kubectl describe namespace development
```

Check quota:

```bash
kubectl get resourcequota -n development
```

Check permissions:

```bash
kubectl auth can-i get pods -n development
kubectl auth can-i create deployments -n development
```

Check whether the current context uses the expected namespace:

```bash
kubectl config view --minify
```

## References

- [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Namespace Management](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
