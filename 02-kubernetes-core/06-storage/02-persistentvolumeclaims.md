# PersistentVolumeClaims

> Phase 2 — Kubernetes Core (highest priority) › Storage

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **PersistentVolumeClaim (PVC)** is a Kubernetes resource through which an application requests persistent storage.

Instead of requiring a developer to locate and configure a specific storage volume manually, a PVC declares the storage requirements. Kubernetes then attempts to bind the claim to a suitable PersistentVolume (PV) or dynamically provision storage through a StorageClass.

Think of it this way:

- **PVC:** What storage does my application need?
- **PV:** What storage resource is available?
- **StorageClass:** How should storage be provisioned?
- **Pod:** Where should that storage be mounted?

For example, a data engineering application may need 5 GiB of persistent storage to store generated files, checkpoints, or application state. The application can request this capacity through a PVC without directly managing the underlying storage infrastructure.

The relationship looks like this:

```text
Application Pod
      |
      | references
      v
PersistentVolumeClaim (PVC)
      |
      | binds to
      v
PersistentVolume (PV)
      |
      | backed by
      v
Storage system
```

The backing storage might be a cloud disk, network file system, storage appliance, or local disk, depending on the cluster configuration.

### How a PVC works

1. A developer creates a PVC specifying the required capacity, access mode, and optionally a StorageClass.
2. Kubernetes searches for a suitable existing PV or requests dynamic provisioning through a compatible StorageClass.
3. Kubernetes binds the PVC to a suitable PV when the storage requirements and binding constraints are satisfied.
4. A Pod references the PVC in its volume configuration.
5. Kubernetes and the storage driver make the volume available to the Pod.
6. The container mounts the volume at a specified path and reads or writes data.

Once bound, a PVC represents a claim on a particular PV. It is not a direct path to a disk, and creating a PVC does not guarantee immediate provisioning or successful mounting.

### PVC vs PV

| Feature | PersistentVolumeClaim (PVC) | PersistentVolume (PV) |
|---|---|---|
| Purpose | Requests storage | Represents storage capacity |
| Created by | Usually an application developer or operator | An administrator or a dynamic provisioner |
| Scope | Namespaced | Cluster-scoped |
| Capacity | Specifies requested capacity | Advertises available capacity |
| Storage backend | Does not directly define the physical storage location | Represents storage provided by a backend |
| Lifecycle | Can exist before storage is bound | Can exist before a claim requests it |
| Pod usage | Referenced by a Pod | Normally consumed through a PVC |

A PVC is namespaced. A Pod generally must reference a PVC in its own namespace.

### Static and dynamic provisioning

**Static provisioning**

An administrator creates a PV first. A PVC then requests storage that matches the PV's properties.

```text
Administrator creates PV
          |
          v
Developer creates PVC
          |
          v
PVC binds to matching PV
```

**Dynamic provisioning**

A PVC requests storage through a StorageClass, and Kubernetes invokes the configured provisioner to create suitable storage.

```text
Developer creates PVC
          |
          v
PVC references StorageClass
          |
          v
Provisioner creates storage
          |
          v
PV is created and bound
```

Dynamic provisioning is common in managed Kubernetes clusters, but it depends on the availability and configuration of a compatible StorageClass and provisioner.

## Key points

- **Storage request:** A PVC declares the desired storage capacity, access modes, and optionally a StorageClass or volume selector.
- **Binding:** A PVC normally binds to one PV. Kubernetes evaluates compatibility rather than simply choosing any available volume.
- **Capacity matching:** A PV must provide sufficient capacity for the request. A PVC requesting `5Gi` cannot bind to a PV advertising only `2Gi`.
- **Access modes:** These describe the mounting capabilities supported by the volume.
  - `ReadWriteOnce` (RWO): Read-write mounting by one node at a time. Multiple Pods on that node may use the volume if the driver and workload support it.
  - `ReadOnlyMany` (ROX): Read-only mounting by multiple nodes, if supported.
  - `ReadWriteMany` (RWX): Read-write mounting by multiple nodes, if supported.
  - `ReadWriteOncePod` (RWOP): Restricts read-write mounting to a single Pod, subject to Kubernetes version and storage-driver support.
- **Access modes do not guarantee application-level safety:** They do not replace file locking, database concurrency controls, or coordination between writers.
- **StorageClass:** If specified, the PVC requests storage from that class. If omitted, Kubernetes may use the default StorageClass, depending on cluster configuration. If there is no applicable default, dynamic provisioning may not occur.
- **Existing PV vs dynamic provisioning:** A PVC can bind to an existing PV or trigger provisioning. A PVC does not always create a new PV.
- **Binding states:** Common PVC phases include `Pending`, `Bound`, and `Lost`.
  - `Pending`: The claim has not yet bound to a volume. It may be waiting for storage, provisioning, or a consumer Pod.
  - `Bound`: The claim is associated with a PV.
  - `Lost`: The previously bound volume is no longer available to the claim.
- **Delayed binding:** Some StorageClasses use `WaitForFirstConsumer`. The PVC can remain `Pending` until a Pod references it, allowing storage placement to account for scheduling constraints.
- **Pod deletion:** Deleting a Pod does not normally delete its PVC. A replacement Pod can use the same claim if the storage remains available and compatible.
- **PVC deletion:** Deleting a claim can release the PV and trigger its reclaim policy. With `Delete`, a supported provisioner may delete the underlying storage. With `Retain`, the storage is kept for manual recovery or cleanup.
- **Expansion:** A PVC may be expandable if the StorageClass allows expansion and the storage driver supports it.
- **Capacity vs actual quota:** The capacity requested through a PVC does not necessarily enforce a hard usage quota on every backend.
- **Production practice:** Persistent storage is not a backup. Important data needs an independent backup and recovery strategy.

## Commands / manifests

### 1. Inspect PVCs, PVs, and StorageClasses

```bash
# List PVCs in the current namespace
kubectl get pvc

# List PVCs across all namespaces
kubectl get pvc -A

# List PersistentVolumes
kubectl get pv

# List StorageClasses
kubectl get storageclass

# Inspect a particular PVC
kubectl describe pvc <pvc-name>

# Inspect a particular PV
kubectl describe pv <pv-name>

# Check PVC status, bound volume, and requested capacity
kubectl get pvc <pvc-name> \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,VOLUME:.spec.volumeName,REQUEST:.spec.resources.requests.storage
```

Use `kubectl describe pvc` to investigate a claim that remains in `Pending`. The Events section often contains the reason binding or provisioning has not completed.

### 2. Create a PVC using dynamic provisioning

This example requests 5 GiB of persistent storage. It assumes the cluster has a working default StorageClass or another applicable provisioning configuration.

Create `app-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Apply the manifest:

```bash
kubectl apply -f app-pvc.yaml
kubectl get pvc app-pvc
kubectl describe pvc app-pvc
```

If the cluster has a compatible default StorageClass, Kubernetes can dynamically provision storage. Otherwise, the PVC may remain `Pending`.

To specify a particular StorageClass, add the following under `spec`:

```yaml
storageClassName: <storage-class-name>
```

Discover available classes with:

```bash
kubectl get storageclass
kubectl describe storageclass <storage-class-name>
```

Replace the placeholder with a real StorageClass name from your cluster. StorageClass names are cluster-specific.

### 3. Create a PVC that binds to a pre-created PV

For a static provisioning example, suppose an administrator has already created a PV named `lab-pv` with:
- Capacity of `1Gi`
- Access mode `ReadWriteOnce`
- StorageClass `manual`
- Reclaim policy `Retain`

Create `lab-pvc.yaml`:

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

Apply and verify:

```bash
kubectl apply -f lab-pvc.yaml
kubectl get pv,pvc
kubectl describe pvc lab-pvc
```

The claim requests `500Mi`, and the PV advertises `1Gi`. The requested access mode and StorageClass also match, so the PVC can bind to the PV if no other binding constraints prevent it.

Check the binding:

```bash
kubectl get pvc lab-pvc \
  -o jsonpath='{.status.phase}{"\n"}{.spec.volumeName}{"\n"}'
```

Expected output after successful binding:

```text
Bound
lab-pv
```

The exact PV selected depends on the cluster's available storage and binding constraints.

### 4. Mount a PVC inside a Pod

A PVC does not mount itself into a container. A Pod must reference the claim through `spec.volumes`, then mount that volume inside a container.

Create `storage-pod.yaml`:

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

Apply and inspect:

```bash
kubectl apply -f storage-pod.yaml
kubectl get pod storage-demo
kubectl describe pod storage-demo
```

The `claimName` must match an existing PVC in the Pod's namespace. In this example, the PVC and Pod must both be in the current namespace.

### 5. Test persistence after Pod recreation

Write a file into the mounted volume:

```bash
kubectl exec storage-demo -- sh -c \
  'echo "PVC persistence test" > /data/test.txt'

kubectl exec storage-demo -- cat /data/test.txt
```

Delete the Pod:

```bash
kubectl delete pod storage-demo
```

Recreate it:

```bash
kubectl apply -f storage-pod.yaml
kubectl wait --for=condition=Ready pod/storage-demo --timeout=120s
kubectl exec storage-demo -- cat /data/test.txt
```

If the volume remains available and is mounted successfully, the file should still contain:

```text
PVC persistence test
```

This demonstrates the difference between a container's ephemeral writable filesystem and storage provided through a PVC.

The test assumes the PVC is bound and the storage backend supports the required remounting behavior. With a local PV, the Pod may need to run on the node associated with the volume.

### 6. Inspect PVC details using JSONPath

```bash
# PVC phase
kubectl get pvc lab-pvc \
  -o jsonpath='{.status.phase}{"\n"}'

# Bound PV name
kubectl get pvc lab-pvc \
  -o jsonpath='{.spec.volumeName}{"\n"}'

# Requested storage capacity
kubectl get pvc lab-pvc \
  -o jsonpath='{.spec.resources.requests.storage}{"\n"}'

# Requested access modes
kubectl get pvc lab-pvc \
  -o jsonpath='{.spec.accessModes}{"\n"}'
```

These commands are useful when troubleshooting claims or checking storage configuration in scripts.

### 7. Delete a PVC safely

Inspect the claim and its associated PV before deleting it:

```bash
kubectl get pvc lab-pvc
kubectl describe pvc lab-pvc
kubectl get pv
```

If the claim is no longer needed and its data can be discarded, delete it:

```bash
kubectl delete pvc lab-pvc
```

Be aware that the PV's reclaim policy determines what happens to the underlying storage after the claim is released. A `Delete` policy can result in data deletion through a supported provisioner. A `Retain` policy generally leaves the storage for manual recovery or cleanup.

Never treat PVC deletion as a harmless cleanup operation when the volume contains important data.

## Common failures & fixes

- **PVC remains `Pending`**
  - Cause: No suitable PV exists, the requested capacity is too large, access modes do not match, or no usable StorageClass or provisioner is available.
  - Fix: Run `kubectl describe pvc <pvc-name>`. Compare the request with `kubectl get pv` and `kubectl get storageclass`.

- **PVC remains `Pending` because no Pod consumes it**
  - Cause: The StorageClass may use `WaitForFirstConsumer`, delaying binding or provisioning until a Pod references the PVC.
  - Fix: Create the intended consuming Pod and inspect its scheduling events. A `Pending` PVC is not always a storage configuration error.

- **PVC cannot bind to a specific PV**
  - Cause: The PV is already bound, has insufficient capacity, uses incompatible access modes, has a different StorageClass, or conflicts with a selector or volume binding constraint.
  - Fix: Inspect both resources using `kubectl describe pv <pv-name>` and `kubectl describe pvc <pvc-name>`. Check the PV's `claimRef`, capacity, class, and access modes.

- **PVC is bound, but the Pod cannot start**
  - Cause: Binding only confirms the claim-to-volume association. The volume may still fail to attach or mount, the node may be unsuitable, or the storage driver may be unhealthy.
  - Fix: Run `kubectl describe pod <pod-name>` and review the Events section for attachment or mount errors.

- **Pod reports that the PVC does not exist**
  - Cause: The claim name is misspelled or the PVC exists in a different namespace.
  - Fix: Check `kubectl get pvc` in the Pod's namespace and verify `spec.volumes[].persistentVolumeClaim.claimName`.

- **PVC is bound but data is missing**
  - Cause: The Pod may be mounting a different claim, writing to another directory, or using a different volume than expected.
  - Fix: Inspect the Pod manifest and `volumeMounts`. Verify the mounted path using `kubectl exec <pod-name> -- mount` and confirm the expected file path.

- **Application receives `Permission denied`**
  - Cause: The container's runtime user does not have suitable permissions on the mounted storage.
  - Fix: Check storage ownership, directory permissions, and the Pod's security context. Use an appropriate `runAsUser`, `runAsGroup`, or `fsGroup` where supported by the storage driver.

- **PVC capacity cannot be increased**
  - Cause: The StorageClass may not allow expansion, or the backend does not support it.
  - Fix: Inspect the StorageClass and storage-driver documentation. Expansion commonly requires `allowVolumeExpansion: true`. Filesystem resizing may also require support from the driver and filesystem.

- **A claim becomes `Lost`**
  - Cause: The previously bound PV is no longer available or the storage association has been disrupted.
  - Fix: Inspect `kubectl describe pvc <pvc-name>` and `kubectl get pv`. Check the underlying storage and provisioner before attempting recovery.

- **A volume cannot be shared across multiple Pods**
  - Cause: The storage backend may not support the required multi-node access pattern, or the configured access mode is unsuitable.
  - Fix: Check the driver's supported access modes. `ReadWriteOnce` allows read-write mounting by one node at a time, not necessarily only one Pod. For multi-node read-write access, use a compatible backend and access mode such as RWX where available.

- **Data disappears after PVC deletion**
  - Cause: The PV's reclaim policy was `Delete`, and the provisioner removed the underlying storage.
  - Fix: Inspect reclaim policies before deleting claims. Use `Retain` when manual recovery is needed and maintain separate backups for critical data.

- **PVC is stuck during deletion**
  - Cause: Kubernetes may be waiting for a Pod to stop using the claim, or a storage controller may be performing cleanup.
  - Fix: Inspect `kubectl describe pvc <pvc-name>`, the Pods using the claim, and relevant storage-controller events. Do not remove finalizers manually unless you understand the consequences and have confirmed the storage state.

## References

- [Kubernetes documentation: Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes documentation: PersistentVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
- [Kubernetes documentation: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes documentation: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes task: Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
