# PersistentVolumes

> Phase 2 — Kubernetes Core (highest priority) › Storage

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **PersistentVolume (PV)** is a Kubernetes resource that represents a piece of storage available to a cluster. It allows applications to store data beyond the lifecycle of an individual Pod.

By default, a container's writable filesystem is ephemeral. If a container is replaced, its local writable data is lost. A Pod can also be deleted and recreated on another node, so data stored only inside the container is not reliable for long-term use.

PersistentVolumes separate storage from the Pods that consume it.

For example, a data engineering application might need to store:
- Database files that must survive Pod restarts.
- Application uploads and generated reports.
- Intermediate data that must persist across container replacements.
- Shared files that multiple workloads need to access, if the storage system supports it.

A PV is the **storage resource**, while a PersistentVolumeClaim (PVC) is a **request for storage** made by an application.

The typical relationship is:

`Pod → PersistentVolumeClaim → PersistentVolume → Storage system`

The storage system could be a cloud disk, a network file system, a storage appliance, or local storage attached to a node.

### PV vs PVC vs Volume

| Concept | Purpose | Example |
|---|---|---|
| PersistentVolume (PV) | Represents available persistent storage | A 10 GiB storage volume |
| PersistentVolumeClaim (PVC) | Requests storage and access characteristics | Request 5 GiB with `ReadWriteOnce` |
| Volume | Storage mounted into a Pod | A volume mounted at `/var/lib/app` |
| StorageClass | Defines how storage can be provisioned | A class backed by a cloud disk driver |

A PVC does not directly create storage in every configuration. With **dynamic provisioning**, Kubernetes can provision a PV through a StorageClass when a PVC requests it. With **static provisioning**, an administrator creates the PV beforehand.

### Static vs dynamic provisioning

- **Static provisioning:** An administrator creates a PV manually. A matching PVC can bind to it.
- **Dynamic provisioning:** A PVC references a StorageClass, and the relevant storage provisioner creates storage automatically when needed.
- **Local storage:** A PV can represent storage on a specific node. The workload may be constrained to that node.
- **Network or cloud storage:** A PV can represent storage that is accessible through a storage driver, subject to that driver's capabilities and access-mode restrictions.

Dynamic provisioning is common in managed Kubernetes environments, but the available StorageClasses depend on the cluster configuration.

### Persistent storage lifecycle

1. A storage administrator creates a PV, or a PVC triggers dynamic provisioning.
2. An application creates a PVC requesting a capacity and access mode.
3. Kubernetes binds the PVC to a suitable PV.
4. A Pod references the PVC in its specification.
5. Kubernetes and the storage driver arrange for the volume to be made available to the Pod.
6. The container mounts the volume at a path and reads or writes data.
7. If the Pod is recreated, the PVC can be referenced again so the application can access the persistent data, provided the storage remains available and compatible.

A PV is a cluster-scoped resource, whereas a PVC is namespaced. A Pod generally references a PVC in its own namespace.

## Key points

- **Persistence:** A PV allows data to outlive an individual container or Pod, but persistence depends on the backing storage and its lifecycle.
- **Binding:** A PVC binds to a PV based on factors such as requested capacity, access modes, StorageClass, and volume selectors.
- **Capacity:** A PVC requests storage capacity, such as `5Gi`. A PV advertises the capacity it provides. Kubernetes does not automatically guarantee that every storage backend enforces a hard usage limit exactly as expected, so verify the backend's behavior.
- **Access modes:** These describe how a volume can be mounted by nodes or workloads. They are not universal guarantees that an application can safely write concurrently.
  - `ReadWriteOnce` (RWO): The volume can be mounted read-write by one node at a time. Multiple Pods on that same node may be able to use it, depending on the driver and workload configuration.
  - `ReadOnlyMany` (ROX): The volume can be mounted read-only by multiple nodes, if supported.
  - `ReadWriteMany` (RWX): The volume can be mounted read-write by multiple nodes, if supported.
  - `ReadWriteOncePod` (RWOP): Restricts read-write mounting to a single Pod, subject to Kubernetes version and CSI driver support.
- **Access modes are not application-level locking:** They do not replace database concurrency controls, file locking, or coordination between writers.
- **StorageClass:** Determines the storage provisioning behavior and can define parameters such as the storage type and provisioner.
- **Reclaim policy:** Controls what happens to the backing storage when the associated PVC is released.
  - `Retain`: Keeps the PV and underlying storage for manual recovery or cleanup.
  - `Delete`: Requests deletion of the PV's backing storage when supported by the provisioner.
  - `Recycle`: Deprecated and not recommended.
- **PVC deletion is not the same as Pod deletion:** Deleting a Pod does not normally delete its PVC. Deleting a PVC releases its binding and can trigger reclaim-policy behavior for the PV.
- **Volume expansion:** Some StorageClasses and storage drivers support expansion. The PVC and storage configuration must allow it.
- **Storage is not automatically backed up:** Persistent storage protects against some Pod and container lifecycle events, but it does not replace backups, replication, or disaster recovery.
- **PV and PVC namespaces:** PVs are cluster-scoped; PVCs are namespaced.
- **Production consideration:** Databases and stateful data platforms need appropriate storage performance, backup, recovery, access control, and availability planning.

## Commands / manifests

### 1. Inspect available storage resources

```bash
# List PersistentVolumes across the cluster
kubectl get pv

# List PVCs in the current namespace
kubectl get pvc

# List PVCs across all namespaces
kubectl get pvc -A

# Inspect StorageClasses
kubectl get storageclass

# Inspect a particular PV
kubectl describe pv <pv-name>

# Inspect a particular PVC
kubectl describe pvc <pvc-name>
```

Use `kubectl describe` when a PVC is stuck in `Pending`. Its events often explain why Kubernetes cannot find or provision suitable storage.

### 2. Create a static PersistentVolume

The following example creates a local PV for a lab cluster. It uses a directory on a specific node, so it is **not a portable production storage solution**.

First, create the directory on the intended Kubernetes node:

```bash
sudo mkdir -p /mnt/data-lab
```

Create `pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lab-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  local:
    path: /mnt/data-lab
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - worker-1
```

Replace `worker-1` with the actual hostname label of the node that contains the directory:

```bash
kubectl get nodes -L kubernetes.io/hostname
```

Apply the manifest:

```bash
kubectl apply -f pv.yaml
kubectl get pv
kubectl describe pv lab-pv
```

Important details:
- The directory must exist on the specified node.
- Local PVs require node affinity because the storage is tied to a particular node.
- The PV's `1Gi` capacity is declared metadata; it does not automatically limit the directory's disk usage.
- The `manual` StorageClass name is a label for matching this PV to a claim. It does not create a StorageClass automatically.

### 3. Create a PersistentVolumeClaim

Create `pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
```

Apply and inspect:

```bash
kubectl apply -f pvc.yaml
kubectl get pv,pvc
kubectl describe pvc lab-pvc
```

The claim requests `500Mi`, and the static PV advertises `1Gi`. Their storage class and access mode match, so the claim can bind to the PV if no other binding constraint prevents it.

Expected state after successful binding:

```text
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
persistentvolume/lab-pv  1Gi  RWO          Retain           Bound

NAME                            STATUS   VOLUME   CAPACITY
persistentvolumeclaim/lab-pvc   Bound    lab-pv   1Gi
```

Exact output formatting varies by Kubernetes version.

### 4. Mount the PVC into a Pod

Create `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do sleep 3600; done"]
      volumeMounts:
        - name: app-storage
          mountPath: /data
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: lab-pvc
```

Apply and verify:

```bash
kubectl apply -f pod.yaml
kubectl get pod storage-demo
kubectl describe pod storage-demo
```

The PVC is declared in `spec.volumes`, then mounted into the container at `/data` using `volumeMounts`.

### 5. Verify that data survives Pod recreation

Write a file to the mounted storage:

```bash
kubectl exec storage-demo -- sh -c \
  'echo "persistent-data-test" > /data/test.txt'

kubectl exec storage-demo -- cat /data/test.txt
```

Delete the Pod:

```bash
kubectl delete pod storage-demo
```

Recreate it:

```bash
kubectl apply -f pod.yaml
kubectl wait --for=condition=Ready pod/storage-demo --timeout=120s
kubectl exec storage-demo -- cat /data/test.txt
```

If the local PV is mounted successfully and the Pod runs on its permitted node, the file should still exist. The Pod was deleted, but the PV and PVC were not.

### 6. Dynamic provisioning with a StorageClass

In a cluster with a working default StorageClass, a PVC can trigger dynamic provisioning without an administrator creating a PV manually.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Apply and inspect:

```bash
kubectl apply -f dynamic-pvc.yaml
kubectl get pvc dynamic-pvc
kubectl get pv
kubectl describe pvc dynamic-pvc
```

If the cluster has a default StorageClass and its provisioner is functioning, Kubernetes can provision a suitable PV. Otherwise, the claim may remain `Pending`.

To request a specific StorageClass, add this field under `spec`:

```yaml
storageClassName: <storage-class-name>
```

Find the names available in your cluster with:

```bash
kubectl get storageclass
```

Do not assume that a particular cloud storage class exists across every cluster.

### 7. Understand reclaim policies

Inspect the reclaim policy:

```bash
kubectl get pv lab-pv \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,RECLAIM:.spec.persistentVolumeReclaimPolicy
```

With `Retain`, deleting the claim generally releases the PV for manual recovery and cleanup rather than automatically deleting the backing data. With `Delete`, a supported dynamic provisioner may delete the backing storage when the claim is deleted.

**Do not delete a PVC containing important data just to test reclaim behavior.** Use disposable storage for experiments.

## Common failures & fixes

- **PVC stays `Pending`**
  - Cause: No matching PV exists, the requested capacity is too large, access modes or StorageClass do not match, or dynamic provisioning is unavailable.
  - Fix: Run `kubectl describe pvc <pvc-name>` and inspect the Events section. Compare the claim against available PVs using `kubectl get pv` and check StorageClasses using `kubectl get storageclass`.

- **PVC is waiting for a consumer**
  - Cause: A StorageClass may use `WaitForFirstConsumer`, which delays binding or provisioning until a Pod that uses the PVC is scheduled.
  - Fix: Create the consuming Pod and inspect its scheduling events. Do not assume that every `Pending` PVC is misconfigured.

- **Pod cannot mount the volume**
  - Cause: The storage driver is unavailable, the volume cannot attach to the selected node, permissions are incorrect, or the backend has a problem.
  - Fix: Run `kubectl describe pod <pod-name>`, inspect events, and check the storage driver's health. For local PVs, confirm that the directory exists on the correct node.

- **Local PV Pod is stuck in `Pending`**
  - Cause: The node selected by the PV's node affinity is unavailable, or the Pod's scheduling constraints conflict with it.
  - Fix: Check `kubectl get nodes -o wide`, inspect the PV's node affinity, and review the Pod's scheduling events.

- **Application receives `Permission denied`**
  - Cause: The mounted directory's ownership or permissions do not allow the container's runtime user to write.
  - Fix: Check directory permissions on the storage backend and the container's security context. Use a suitable `runAsUser`, `runAsGroup`, or `fsGroup` where supported by the volume driver. Avoid blindly using `chmod 777`.

- **Data disappears after a claim is deleted**
  - Cause: The PV's reclaim policy is `Delete`, and the provisioner removed the backing storage.
  - Fix: Check the reclaim policy before deleting claims. Use `Retain` where manual recovery is required, and maintain independent backups for important data.

- **A volume cannot be mounted by multiple Pods**
  - Cause: The storage backend or driver does not support the required sharing pattern, or the access mode does not permit the requested mounts.
  - Fix: Confirm the driver's supported access modes. `ReadWriteOnce` means read-write mounting by one node, not necessarily one Pod. If multiple nodes need concurrent read-write access, use a backend and access mode that support it, such as RWX where available.

- **The volume is full even though its PV declares a capacity**
  - Cause: The backend may not enforce the declared capacity as a hard filesystem quota, or the underlying disk is full.
  - Fix: Check actual filesystem usage and storage-backend metrics. Use a storage solution that enforces quotas if strict capacity isolation is required.

- **PVC expansion fails**
  - Cause: The StorageClass does not permit expansion, or the storage driver does not support it.
  - Fix: Inspect the StorageClass and storage-driver documentation. Expansion typically requires `allowVolumeExpansion: true` and compatible backend support. Do not assume that reducing a PVC's requested capacity is supported.

- **Application data becomes inconsistent despite persistent storage**
  - Cause: Persistent storage preserves data across certain lifecycle events but does not guarantee application-level consistency or provide backups by itself.
  - Fix: Use application-aware backups, restore testing, and suitable replication and recovery mechanisms. For databases, follow the database's own consistency and recovery guidance.

## References

- [Kubernetes documentation: Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes documentation: PersistentVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
- [Kubernetes documentation: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes documentation: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes documentation: Local volumes](https://kubernetes.io/docs/concepts/storage/volumes/#local)
- [Kubernetes documentation: Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
