# StorageClasses

> Phase 2 — Kubernetes Core (highest priority) › Storage

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **StorageClass** is a Kubernetes resource that describes how persistent storage should be provisioned. It allows developers to request storage without needing to know all the details of the underlying storage infrastructure.

For example, a cloud Kubernetes cluster might offer different storage options for:
- General-purpose application storage.
- High-performance disks for databases.
- Lower-cost storage for less frequently accessed data.
- Storage with specific performance, replication, or availability characteristics.

Instead of manually creating a PersistentVolume (PV) for every application, a developer creates a PersistentVolumeClaim (PVC) and specifies the required StorageClass. Kubernetes uses the configured provisioner to request suitable storage.

A StorageClass is essentially a **storage provisioning template**, not a storage volume itself.

### How StorageClass fits into Kubernetes storage

```text
Application Pod
      |
      | references
      v
PersistentVolumeClaim (PVC)
      |
      | requests
      v
StorageClass
      |
      | invokes
      v
Storage provisioner / CSI driver
      |
      | creates or allocates
      v
PersistentVolume (PV)
      |
      v
Underlying storage
```

Each component has a distinct responsibility:

| Component | Responsibility |
|---|---|
| StorageClass | Defines the storage provisioning policy and parameters |
| PVC | Requests storage capacity and access characteristics |
| Provisioner or CSI driver | Creates or allocates storage through the backend |
| PV | Represents the provisioned storage resource |
| Pod | Consumes the PVC by mounting its volume |

### Static vs dynamic provisioning

**Static provisioning**

An administrator creates a PV manually, and a PVC binds to a compatible existing PV. A StorageClass can still be used to identify or group storage, but automatic provisioning is not required.

**Dynamic provisioning**

A PVC requests a StorageClass, and the configured provisioner creates a suitable storage resource. Kubernetes then creates or associates a PV with the new storage.

Dynamic provisioning is the main reason StorageClasses are useful: applications can request storage without administrators manually creating a separate PV for every claim.

### Default StorageClass

A cluster can designate a StorageClass as the default. When a PVC does not specify `storageClassName`, Kubernetes can use that default class for dynamic provisioning.

For example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

If the cluster has a functioning default StorageClass, the claim can use it automatically.

If you need to request a specific class, specify it explicitly:

```yaml
storageClassName: fast-storage
```

The name `fast-storage` is only an example. Available StorageClass names depend on the cluster.

One important distinction: setting `storageClassName: ""` explicitly requests a PVC with no StorageClass. It prevents the claim from using the default StorageClass and is commonly used when binding to a manually provisioned PV with no class.

### StorageClass parameters

A StorageClass can include settings that determine how storage is provisioned.

Common fields include:

- `provisioner`: Identifies the provisioner or CSI driver responsible for creating storage.
- `parameters`: Backend-specific configuration, such as disk type or performance tier.
- `reclaimPolicy`: Determines what happens to dynamically provisioned storage when its PV is released.
- `allowVolumeExpansion`: Indicates whether claims using the class can request storage expansion, subject to backend support.
- `volumeBindingMode`: Controls when volume binding and provisioning occur.
- `mountOptions`: Specifies supported filesystem mount options.

The meaning of `parameters` depends on the provisioner. A parameter valid for one cloud provider or CSI driver might be invalid for another.

### Volume binding modes

Two common binding modes are:

- `Immediate`: Storage binding or provisioning begins as soon as the PVC is created.
- `WaitForFirstConsumer`: Binding or provisioning waits until a Pod uses the PVC, allowing Kubernetes to consider the Pod's scheduling constraints, such as node affinity and topology.

`WaitForFirstConsumer` is especially useful for storage that is restricted to particular zones or nodes. It helps avoid provisioning storage in a location where the consuming Pod cannot run.

## Key points

- **StorageClass is a cluster-scoped resource:** It is not created inside a namespace.
- **StorageClass is not a volume:** It describes how storage should be provisioned; the PV represents the resulting storage resource.
- **Dynamic provisioning:** A PVC can trigger automatic storage provisioning by requesting a compatible StorageClass.
- **Provisioner:** The provisioner must be available and correctly configured. A StorageClass definition alone does not create storage.
- **CSI drivers:** Many modern Kubernetes storage integrations use Container Storage Interface drivers to provision, attach, mount, and manage storage.
- **Default class:** A cluster may have a default StorageClass. If multiple classes are marked default, Kubernetes uses the most recently created default for PVCs that omit the class, but configuring one clear default is preferable.
- **No default class:** A PVC that omits `storageClassName` may remain `Pending` if there is no default class and no suitable static PV.
- **Explicit empty class:** `storageClassName: ""` means no StorageClass should be used for that claim. This is different from omitting the field.
- **StorageClass names:** Names such as `standard`, `premium`, or `fast-storage` are not universal Kubernetes defaults. Always inspect the actual cluster.
- **Reclaim policy:** `Delete` can cause dynamically provisioned backing storage to be deleted when the PV is released, if supported by the provisioner. `Retain` preserves the storage for manual recovery or cleanup.
- **Volume expansion:** `allowVolumeExpansion: true` permits expansion requests through the class when the underlying storage system supports them. It does not guarantee every expansion will succeed.
- **Binding mode:** `WaitForFirstConsumer` can keep a PVC in `Pending` until a consuming Pod is scheduled for storage placement.
- **Access modes are separate:** The PVC requests access modes such as `ReadWriteOnce` or `ReadWriteMany`. A StorageClass does not automatically guarantee that the backend supports every mode.
- **Performance is backend-dependent:** A class name such as `premium` does not itself guarantee a particular IOPS, latency, throughput, or durability. Verify the provider's specifications.
- **StorageClass parameters are driver-specific:** Use the documentation for the installed provisioner or CSI driver instead of copying parameters from another environment.
- **Storage is not backup:** Even high-performance or replicated storage needs an appropriate backup and recovery plan.
- **Production design:** Select StorageClasses based on workload requirements, performance, availability, topology, cost, security, and recovery objectives.

## Commands / manifests

### 1. Inspect available StorageClasses

```bash
# List all StorageClasses
kubectl get storageclass

# Short form
kubectl get sc

# Show detailed configuration for a class
kubectl describe storageclass <storage-class-name>

# Display the StorageClass configuration as YAML
kubectl get storageclass <storage-class-name> -o yaml

# List PersistentVolumes and PersistentVolumeClaims
kubectl get pv
kubectl get pvc -A
```

Example output from `kubectl get storageclass` might look like this:

```text
NAME                 PROVISIONER                 RECLAIMPOLICY   VOLUMEBINDINGMODE
standard (default)   example.csi.driver          Delete          WaitForFirstConsumer
fast-storage         example.csi.driver          Retain          Immediate
```

These names and provisioner values are illustrative. Your cluster may use completely different names and drivers.

### 2. Create a StorageClass

The following is a **template**, not a universally runnable manifest. Replace the provisioner and parameters with values supported by the storage driver installed in your cluster.

Create `storageclass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: example.csi.driver
parameters:
  type: fast
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

Apply it:

```bash
kubectl apply -f storageclass.yaml
kubectl get storageclass
kubectl describe storageclass fast-storage
```

Understand each field:

- `apiVersion`: Uses the Kubernetes StorageClass API.
- `kind`: Declares the resource type.
- `metadata.name`: Names the class.
- `provisioner`: Identifies the driver that handles provisioning.
- `parameters`: Passes driver-specific options to the provisioner.
- `reclaimPolicy`: Defines the default reclaim policy for PVs dynamically provisioned by this class.
- `allowVolumeExpansion`: Allows supported PVC expansion requests.
- `volumeBindingMode`: Delays provisioning until a Pod uses the claim, which can help with topology-aware scheduling.

**Important:** Kubernetes will accept the general structure only if the manifest is valid, but successful provisioning requires a real, functioning provisioner and valid parameters. Do not deploy `example.csi.driver` into a real cluster expecting it to create storage.

### 3. Request storage from a specific StorageClass

Create `app-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: fast-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Apply and inspect:

```bash
kubectl apply -f app-pvc.yaml
kubectl get pvc app-data
kubectl describe pvc app-data
```

The PVC requests 10 GiB from `fast-storage`. If the class and provisioner are configured correctly, storage can be provisioned dynamically.

If the class uses `WaitForFirstConsumer`, the claim may remain `Pending` until a Pod references it.

To consume the PVC, create a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-consumer
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: app-volume
          mountPath: /data
  volumes:
    - name: app-volume
      persistentVolumeClaim:
        claimName: app-data
```

Apply and check the result:

```bash
kubectl apply -f storage-consumer.yaml
kubectl get pod storage-consumer
kubectl get pvc app-data
kubectl get pv
```

This Pod assumes the `fast-storage` class refers to a working storage provisioner.

### 4. Create a PVC using the default StorageClass

If the cluster has a suitable default StorageClass, omit `storageClassName`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: default-storage-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

Apply it:

```bash
kubectl apply -f default-storage-pvc.yaml
kubectl get pvc default-storage-pvc
kubectl describe pvc default-storage-pvc
```

Kubernetes can use the default StorageClass to provision storage dynamically.

Check which class is marked as default:

```bash
kubectl get storageclass
```

The default is commonly indicated by `(default)` in the output.

### 5. Mark a StorageClass as the default

A cluster administrator can mark a class as the default using the annotation:

```bash
kubectl patch storageclass <storage-class-name> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify the change:

```bash
kubectl get storageclass
```

Be careful when changing the default in a shared or production cluster. PVCs that omit `storageClassName` can be affected by the default configuration.

To remove the default designation from a class:

```bash
kubectl patch storageclass <storage-class-name> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Use this only when you intend to change the cluster's default storage behavior.

### 6. Compare StorageClass settings

```bash
# Display names, provisioners, and binding modes
kubectl get storageclass \
  -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner,BINDING:.volumeBindingMode,RECLAIM:.reclaimPolicy

# Inspect one class's parameters
kubectl get storageclass <storage-class-name> \
  -o jsonpath='{.parameters}{"\n"}'

# Inspect its expansion setting
kubectl get storageclass <storage-class-name> \
  -o jsonpath='{.allowVolumeExpansion}{"\n"}'
```

These commands help compare classes before choosing one for an application.

### 7. Delete a StorageClass

```bash
kubectl delete storageclass <storage-class-name>
```

Deleting a StorageClass does not automatically delete all existing PVs or their data. However, existing claims may still depend on the storage it provisions, and future provisioning requests using that class will no longer be able to reference it.

Before deleting a class, check its associated resources:

```bash
kubectl get pvc -A
kubectl get pv
kubectl describe storageclass <storage-class-name>
```

Avoid deleting a StorageClass used by production workloads without assessing the impact first.

## Common failures & fixes

- **PVC remains `Pending`**
  - Cause: The requested StorageClass does not exist, the provisioner is unavailable, the requested capacity cannot be provisioned, or the storage backend is misconfigured.
  - Fix: Run `kubectl describe pvc <pvc-name>`, `kubectl get storageclass`, and `kubectl get events -A --sort-by=.metadata.creationTimestamp`. Review the Events section for the specific failure.

- **StorageClass exists, but dynamic provisioning fails**
  - Cause: The StorageClass points to an unavailable or incorrectly configured provisioner, or its parameters are invalid.
  - Fix: Run `kubectl describe storageclass <storage-class-name>`. Check the CSI controller or provisioner logs and verify the driver-specific parameter names and values.

- **PVC says the StorageClass does not exist**
  - Cause: The claim's `storageClassName` contains a typo or refers to a class that has not been installed.
  - Fix: Run `kubectl get sc` and use an exact existing class name. Do not assume that `standard` or `premium` exists in every cluster.

- **PVC without `storageClassName` remains `Pending`**
  - Cause: The cluster has no default StorageClass and no suitable static PV, or the default provisioner is failing.
  - Fix: Check `kubectl get sc`. Either configure a suitable default, specify an existing StorageClass, or arrange a compatible static PV.

- **PVC remains `Pending` with `WaitForFirstConsumer`**
  - Cause: The StorageClass intentionally delays provisioning until a Pod uses the PVC.
  - Fix: Create the consuming Pod and inspect its scheduling events. Check node selectors, affinity, resource availability, and storage topology.

- **Pod cannot be scheduled because of storage topology**
  - Cause: The volume is restricted to a zone or node that is incompatible with the Pod's scheduling requirements.
  - Fix: Inspect the PVC, PV, StorageClass binding mode, and Pod scheduling events. Use compatible topology constraints and allow Kubernetes to select a valid placement.

- **Storage provisioner rejects the parameters**
  - Cause: The StorageClass contains parameters unsupported by the installed driver.
  - Fix: Consult the relevant CSI driver's documentation. Replace example parameters with values supported by the actual storage backend.

- **StorageClass cannot be edited as expected**
  - Cause: Many StorageClass provisioning settings are immutable or cannot safely be changed after creation.
  - Fix: Review the API validation error. Create a new StorageClass with the desired settings and migrate workloads or claims where necessary. Do not assume existing volumes will be automatically migrated.

- **Storage expansion fails**
  - Cause: `allowVolumeExpansion` is unset or false, or the storage driver and backend do not support the requested expansion.
  - Fix: Inspect the StorageClass and driver documentation. Verify that the PVC requests a larger capacity and check events for controller or filesystem-resizing failures.

- **Data disappears after a claim is deleted**
  - Cause: The PV reclaim policy is `Delete`, and the provisioner removed the backing storage.
  - Fix: Check the reclaim policy before deleting PVCs. Use `Retain` where manual recovery is required and maintain independent backups.

- **A class named `fast-storage` performs poorly**
  - Cause: The name is only a label. Actual performance depends on the backend type, provisioner parameters, workload, and storage limits.
  - Fix: Verify the provider's documented performance characteristics and monitor latency, throughput, and IOPS. Do not infer performance from the StorageClass name alone.

## References

- [Kubernetes documentation: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes documentation: Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes documentation: PersistentVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
- [Kubernetes documentation: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes documentation: Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Kubernetes documentation: Container Storage Interface (CSI)](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
