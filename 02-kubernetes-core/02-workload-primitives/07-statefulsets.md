# StatefulSets

> Phase 2 — Kubernetes Core (highest priority) › Workload primitives

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **StatefulSet** is a Kubernetes workload resource used to manage applications that require stable Pod identities, predictable ordering, or persistent storage associated with individual Pods.

Unlike a Deployment, where Pods are generally interchangeable, a StatefulSet gives each Pod a stable identity that persists across rescheduling and replacement. When a Pod is recreated, Kubernetes can recreate it with the same ordinal identity and reconnect it to its associated persistent storage, subject to storage and scheduling constraints.

StatefulSets are commonly used for:
- Distributed databases such as Cassandra.
- Stateful data stores such as MongoDB when deployed using an appropriate architecture.
- Distributed messaging systems such as Kafka.
- Applications that require stable network identities.
- Stateful services that need persistent volumes associated with individual replicas.

For data engineering, you will more often interact with managed databases, cloud data platforms, and batch processing workloads than build StatefulSets yourself. However, understanding them helps when working with self-hosted Kafka, distributed databases, or other stateful infrastructure running on Kubernetes.

### Why StatefulSets exist

Consider an application with three replicas.

With a Deployment:
- Pods are designed to be interchangeable.
- A replacement Pod can receive a different name.
- Persistent storage is not automatically tied to a unique replica identity.

With a StatefulSet:
- Each Pod receives a stable ordinal identity, such as `database-0`, `database-1`, and `database-2`.
- A replacement for `database-1` retains the `database-1` identity.
- Each replica can have its own persistent volume claim when `volumeClaimTemplates` are configured.
- Pod creation and termination can follow defined ordering rules.

This matters when an application needs to know which replica it is, maintain a stable endpoint, or associate data with a specific replica.

### How a StatefulSet works

Suppose you deploy a StatefulSet named `database` with three replicas.

```text
StatefulSet: database
|
|-- database-0 --> PersistentVolumeClaim: data-database-0
|
|-- database-1 --> PersistentVolumeClaim: data-database-1
|
|-- database-2 --> PersistentVolumeClaim: data-database-2
```

The StatefulSet controller manages the desired replicas and their identities.

Key behaviors:

1. **Stable Pod identity:** Pods receive predictable names based on their ordinal index.
2. **Stable network identity:** With a suitable governing headless Service, each Pod can have a stable DNS identity.
3. **Persistent storage association:** With `volumeClaimTemplates`, each replica gets its own PersistentVolumeClaim.
4. **Ordered creation:** The default `OrderedReady` policy creates Pods sequentially and waits for each to become ready before proceeding.
5. **Ordered termination:** Under the default policy, scale-down and termination follow ordinal ordering.
6. **Replacement behavior:** If a Pod is lost, Kubernetes can recreate the same ordinal Pod and attach its associated storage when the infrastructure and storage configuration permit.

A StatefulSet does not itself implement database replication, leader election, data consistency, or failover. Those behaviors must be provided by the application or an appropriate operator.

### StatefulSet vs Deployment

| Feature | StatefulSet | Deployment |
|---|---|---|
| Main purpose | Manage workloads with stable identities and storage associations | Manage interchangeable application replicas |
| Pod identity | Stable ordinal names | Pod names can change when Pods are replaced |
| Network identity | Can provide stable per-Pod DNS through a governing Service | Usually accessed through a common Service |
| Persistent storage | Supports per-replica claims through `volumeClaimTemplates` | Can use volumes, but per-replica storage identity is not built into the Deployment model |
| Startup order | Ordered by default | No equivalent ordinal startup guarantee |
| Typical workloads | Stateful databases and distributed systems | APIs, web applications, stateless workers |

A StatefulSet is not automatically more reliable than a Deployment. It provides the identity and lifecycle behavior required by certain applications, but application-level replication, backup, recovery, and storage availability remain separate concerns.

### StatefulSet vs DaemonSet vs Job

- **StatefulSet:** Runs replicas with stable identities, often with persistent storage.
- **DaemonSet:** Runs a Pod on each eligible node.
- **Job:** Runs finite work until the required successful completion count is reached.
- **CronJob:** Creates Jobs according to a schedule.
- **Deployment:** Maintains a desired number of interchangeable application replicas.

For most ordinary data engineering scripts, ETL tasks, and scheduled data quality checks, a Job or CronJob is more appropriate than a StatefulSet.

## Key points

- **Stable identity:** StatefulSet Pods have predictable ordinal names such as `app-0` and `app-1`.
- **Stable network identity:** A governing headless Service can provide stable DNS names for individual Pods.
- **Persistent storage:** `volumeClaimTemplates` creates a separate PersistentVolumeClaim for each StatefulSet replica.
- **Pod replacement preserves identity:** A replacement Pod can reuse the same ordinal identity and associated claim.
- **Storage is not the same as backup:** Persistent volumes help preserve data across Pod replacement, but they do not replace backups or disaster recovery.
- **Ordered deployment:** The default `OrderedReady` policy creates Pods in ordinal order and waits for readiness.
- **Parallel management:** `podManagementPolicy: Parallel` allows Pods to be created or deleted in parallel during scaling, rather than enforcing the default sequential behavior.
- **Ordered rolling updates:** StatefulSets support rolling updates, generally proceeding through replicas in descending ordinal order.
- **Readiness matters:** A Pod that never becomes ready can block later Pods during an ordered rollout or scale-up.
- **Headless Service:** A Service with `clusterIP: None` is commonly used to provide stable DNS identities for StatefulSet Pods.
- **Not a database operator:** StatefulSet manages Pods and related resources, but does not automatically configure database clustering, replication, or failover.
- **Storage depends on the environment:** Reattaching a volume depends on the storage provisioner, access mode, node placement, and availability of the underlying storage.
- **Scaling down can have data implications:** PVCs are not necessarily deleted when a StatefulSet is scaled down or deleted. Retention depends on the configured PVC retention policy and related resource lifecycle settings.
- **Not commonly needed for basic DE workloads:** Use StatefulSets when you specifically need stable replica identities or storage associations, rather than simply because an application writes data.

## Commands / manifests

### 1. Create a basic StatefulSet

Save this manifest as `statefulset.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 3
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
          image: nginx:1.28
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f statefulset.yaml
```

What this manifest demonstrates:

- `kind: Service`: Creates the network resource used by the StatefulSet.
- `clusterIP: None`: Makes the Service headless, allowing DNS records for individual Pods.
- `kind: StatefulSet`: Creates the StatefulSet.
- `serviceName: web`: Associates the StatefulSet with its governing Service.
- `replicas: 3`: Requests three Pods.
- `selector.matchLabels`: Identifies the Pods managed by the StatefulSet.
- `template.metadata.labels`: Provides matching labels for the Pods.
- The container runs NGINX, keeping the example simple.

The resulting Pods should be named `web-0`, `web-1`, and `web-2`. This example demonstrates stable identity and networking, but it does not configure persistent storage.

### 2. Inspect StatefulSet Pods

```bash
kubectl get statefulsets
kubectl get pods -o wide
kubectl describe statefulset web
```

Check the Pod names:

```bash
kubectl get pods -l app=web
```

You should see ordinal names such as:

```text
web-0
web-1
web-2
```

Actual readiness and creation times depend on cluster resources and scheduling.

### 3. Inspect stable DNS identities

```bash
kubectl get service web
kubectl get endpointslice
kubectl exec web-0 -- nslookup web
```

The headless Service is named `web`, and the governing Service gives Pods stable DNS identities. A Pod's fully qualified DNS name typically follows this pattern:

```text
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

For example, if the StatefulSet and Service are in the `default` namespace:

```text
web-0.web.default.svc.cluster.local
```

The full DNS suffix depends on the cluster's DNS configuration. The `nslookup` command also depends on the diagnostic utilities available in the container image; if the image lacks them, use a temporary diagnostic Pod.

### 4. Add persistent storage

For stateful workloads, a `volumeClaimTemplates` section can create one PersistentVolumeClaim per replica.

Example StatefulSet manifest:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 2
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
        - name: database
          image: busybox:1.36
          command:
            - sh
            - -c
            - 'while true; do sleep 3600; done'
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

This is a storage demonstration, not a production database deployment. It also requires a working default StorageClass or another supported way to provision or bind a PersistentVolume.

Apply it:

```bash
kubectl apply -f database-statefulset.yaml
```

Inspect the claims:

```bash
kubectl get pvc
kubectl get pods -l app=database
kubectl describe pvc data-database-0
```

Expected claim names include `data-database-0` and `data-database-1`.

Each replica gets its own claim. `ReadWriteOnce` generally permits a volume to be mounted read-write by a single node at a time; it does not necessarily mean only one Pod on that node can use it. The supported access modes and attachment behavior depend on the storage provider.

### 5. Scale a StatefulSet

```bash
kubectl scale statefulset web --replicas=5
```

Verify:

```bash
kubectl get statefulset web
kubectl get pods -l app=web
```

The StatefulSet should aim to maintain five replicas with ordinal names from `web-0` through `web-4`.

With the default `OrderedReady` policy, scale-up occurs sequentially. When scaling down, the highest ordinal Pods are normally removed first.

### 6. Check rollout status

```bash
kubectl rollout status statefulset/web
kubectl rollout history statefulset/web
kubectl describe statefulset web
```

StatefulSets can use rolling updates, but a rollout may pause if a Pod cannot become ready. Inspect Pod events and logs before assuming the controller is stuck.

### 7. Update the container image

```bash
kubectl set image statefulset/web nginx=nginx:1.28
```

Check the rollout:

```bash
kubectl rollout status statefulset/web
kubectl get pods -l app=web
```

This command uses the same image as the example manifest. Replace it with the intended version when performing an actual update.

### 8. Delete a StatefulSet

```bash
kubectl delete statefulset web
```

Deleting the StatefulSet does not necessarily delete its associated PVCs. This behavior helps protect persistent data from accidental loss, but you should verify the storage lifecycle and retention policy before deleting resources.

Inspect the remaining claims:

```bash
kubectl get pvc
```

Do not delete PVCs unless you understand whether the underlying data must be retained.

### 9. Useful inspection commands

```bash
kubectl get statefulsets -A
kubectl describe statefulset <statefulset-name>
kubectl get pods -o wide
kubectl get pvc
kubectl describe pvc <pvc-name>
kubectl get services
kubectl get events --sort-by=.metadata.creationTimestamp
```

Use these commands to investigate replica status, storage provisioning, network identity, scheduling, and rollout issues.

## Common failures & fixes

| Problem | Likely cause | Fix |
|---|---|---|
| StatefulSet Pods remain `Pending` | Insufficient resources, storage problems, or scheduling constraints | Inspect Pod events using `kubectl describe pod <pod-name>` |
| Only the first Pod is created | The default ordered policy is waiting for an earlier Pod to become ready | Inspect the first unready Pod, its events, and its logs |
| PVC remains `Pending` | No suitable StorageClass or PersistentVolume is available | Check `kubectl get storageclass` and `kubectl describe pvc <pvc-name>` |
| Pod cannot mount its volume | Storage attachment, access mode, node placement, or filesystem problems | Inspect Pod events and PVC/PV details; check the storage provider |
| Pod names are unexpected | The workload is not managed by the expected StatefulSet, or the name is being interpreted incorrectly | Inspect `kubectl get statefulsets` and the Pod owner references |
| Per-Pod DNS does not resolve | Governing Service is missing or misconfigured, or DNS has not propagated | Verify `serviceName`, the headless Service selector, Pod labels, and cluster DNS |
| StatefulSet rollout is stuck | A new Pod cannot become ready or a readiness probe fails | Inspect the rollout status, Pod events, and container logs |
| Replacement Pod cannot start | Persistent volume cannot be attached or the node is unavailable | Inspect PVC/PV state, node status, and storage events |
| Application data is lost | Data was stored in the container filesystem instead of persistent storage, or the volume itself was deleted | Mount a suitable persistent volume and maintain independent backups |
| Scaling down affects an application | A replica was removed without considering cluster membership or data replication | Follow the application's safe scale-down procedure before changing replicas |
| Deleting a StatefulSet leaves PVCs behind | PVC retention behavior preserves claims | Inspect the claims and delete them only if the data is no longer required |
| A database does not replicate automatically | StatefulSet manages workload lifecycle, not database replication logic | Configure the database's replication mechanism or use an appropriate operator |
| A Pod cannot become ready | The application depends on another replica, has invalid configuration, or its readiness check is failing | Inspect logs and readiness probes; verify the application startup and clustering design |

### Practical debugging sequence

When a StatefulSet does not behave as expected:

1. Inspect the StatefulSet's desired and ready replica counts.
2. Identify the first Pod that is not ready.
3. Review its events and container logs.
4. If storage is involved, inspect the PVC and PV states.
5. Verify the governing headless Service and Pod DNS.
6. Check readiness probes, resource requests, and scheduling constraints.
7. Confirm the application's own replication and recovery behavior before scaling or deleting replicas.

Remember: StatefulSets preserve Pod identity and can associate replicas with persistent storage, but they do not guarantee that application data is replicated, backed up, or recoverable.

## References

- [Kubernetes documentation: StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes documentation: Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes documentation: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes documentation: Services and networking](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes documentation: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
