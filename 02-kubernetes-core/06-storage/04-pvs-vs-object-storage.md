# PVs vs object storage

> Phase 2 — Kubernetes Core (highest priority) › Storage

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

PersistentVolumes (PVs) and object storage are two different ways to store data that needs to survive beyond the lifecycle of a container or Pod. They solve different problems, and choosing between them depends on how an application accesses its data.

- **PersistentVolumes (PVs):** Provide storage that Kubernetes workloads can consume through PersistentVolumeClaims (PVCs). The storage is typically presented to the application as a mounted filesystem or block device.
- **Object storage:** Stores data as objects inside buckets or containers. Applications typically access these objects through an API, SDK, or HTTP interface rather than mounting a conventional filesystem.

Examples of object storage services include Amazon S3, Azure Blob Storage, and Google Cloud Storage.

### 1. How PersistentVolumes work

A PV represents storage that a Pod can consume through a PVC. The underlying storage might be a cloud disk, a network filesystem, or local storage.

```text
Kubernetes Pod
      |
      v
PersistentVolumeClaim (PVC)
      |
      v
PersistentVolume (PV)
      |
      v
Storage backend
      |
      v
Mounted filesystem or block device
```

The application reads and writes files through normal filesystem operations, such as opening a file, appending records, or reading a directory.

**Use PV-backed storage when the application expects filesystem or block-storage semantics.**

Common examples:
- Database data directories.
- Stateful applications that need persistent files.
- Applications that require filesystem access.
- Workloads that need persistent working directories.
- Applications that depend on filesystem operations or low-latency random reads and writes.

The actual capabilities depend on the storage driver, access mode, filesystem, and backend.

### 2. How object storage works

Object storage organizes data into buckets and objects. Each object contains data, metadata, and an object key used to identify it.

```text
Kubernetes Pod
      |
      v
Application code / SDK / HTTP API
      |
      v
Object storage service
      |
      v
Bucket
      |
      +---- Object: raw-data.parquet
      |
      +---- Object: curated-data.parquet
      |
      +---- Object: model-artifacts.zip
```

An application uploads, downloads, or manages objects through an API rather than relying on a conventional mounted filesystem.

**Use object storage when you need scalable storage for files and datasets accessed as complete objects or through supported object operations.**

Common examples:
- Data lake raw and curated datasets.
- Parquet, CSV, JSON, and Avro files.
- Machine learning training datasets and model artifacts.
- Application uploads and archived reports.
- Logs, backups, and historical data.
- Data exchanged between independent processing jobs.

Object storage is especially common in data engineering because large datasets can be stored independently of the compute clusters that process them.

### 3. The fundamental difference

The most important distinction is the **data access model**, not simply the storage capacity.

| Characteristic | PV-backed storage | Object storage |
|---|---|---|
| Access model | Mounted filesystem or block device | API, SDK, or HTTP requests |
| Data organization | Files, directories, or blocks | Buckets and objects |
| Kubernetes integration | PVC mounted into a Pod | Application connects to the storage API |
| Typical workloads | Databases, stateful applications, filesystem-dependent tools | Data lakes, archives, backups, datasets |
| Random file updates | Often suitable, depending on backend | Usually handled through object operations; partial updates may not behave like filesystem writes |
| Sharing | Depends on storage type and access mode | Multiple authorized clients can access objects concurrently |
| Scaling | Depends on storage backend and provisioning | Often designed for large-scale object capacity |
| Compute independence | Storage can persist independently of Pods | Storage is generally independent of the Kubernetes cluster |
| Performance | Depends on disk, filesystem, network, and driver | Depends on service, request patterns, object sizes, and network |
| Typical interface | Filesystem calls or block I/O | Object APIs such as `GET`, `PUT`, and `DELETE` |
| Lifecycle | PV, PVC, reclaim policy, and backend lifecycle | Bucket policies, object lifecycle rules, versioning, and retention policies |

Neither option is universally better. A well-designed system may use both.

### 4. When should you use a PV?

Choose a PV when your application needs storage that behaves like a filesystem or block device.

**Example: PostgreSQL running inside Kubernetes**

PostgreSQL maintains database files, write-ahead logs, and other files that require database-controlled access and consistency.

A common design is:

```text
PostgreSQL Pod
      |
      v
PVC
      |
      v
PV
      |
      v
Persistent disk
```

The database accesses its files through the mounted storage. The volume can remain available when the Pod is replaced, subject to the storage backend's availability and attachment constraints.

Other examples:
- Stateful applications that maintain files in a working directory.
- Tools that expect a POSIX-style filesystem.
- Applications that need persistent local working storage.
- Workloads that need a storage interface supported by an appropriate CSI driver.

A PV does not automatically provide database high availability, replication, backups, or protection from storage failure. Those require separate design decisions.

### 5. When should you use object storage?

Choose object storage when data is naturally represented as objects and should be accessed independently of a particular Kubernetes node or Pod.

**Example: A PySpark data engineering pipeline**

Suppose a pipeline processes healthcare claims in Parquet format.

```text
Source systems
      |
      v
Object storage: raw/
      |
      v
Spark processing
      |
      v
Object storage: curated/
      |
      v
Analytics / BI / downstream pipelines
```

The raw and curated datasets live in object storage. Spark workers access the datasets through compatible connectors or cloud storage APIs.

If the Spark cluster is terminated and recreated, the datasets remain in the storage service. The storage lifecycle is independent of the compute lifecycle.

Common uses:
- Data lake storage.
- Batch-processing inputs and outputs.
- Model training data and artifacts.
- Long-term backups and archives.
- Large collections of files shared across different compute environments.

Object storage is not inherently limited to large files, but very small objects can create overhead from request latency, metadata operations, and listing patterns.

### 6. A practical data engineering example

Imagine a healthcare analytics platform processing claims and provider data.

| Requirement | Recommended storage | Why |
|---|---|---|
| PostgreSQL database files | PV-backed storage | The database requires an appropriate persistent filesystem or block device |
| Raw CMS claims files | Object storage | Files can be stored independently of compute and accessed by multiple jobs |
| Curated Parquet datasets | Object storage | Suitable for large analytical datasets and distributed processing |
| Temporary Spark shuffle data | Local or ephemeral storage, depending on workload | Intermediate data can often be regenerated; requirements vary |
| Long-term pipeline output archives | Object storage | Supports durable storage, lifecycle management, and independent access |
| Application configuration files | ConfigMaps or Secrets for configuration; storage only when appropriate | Configuration and persistent datasets are different concerns |
| Stateful service requiring a mounted data directory | PV-backed storage | The application expects filesystem access |

A common architecture is to use PVs for stateful services that require mounted storage and object storage for the analytical data lake.

### 7. Can object storage be mounted as a filesystem?

Sometimes, yes. Tools and drivers can expose object storage through a filesystem-like interface, often using a FUSE driver or a specialized CSI integration.

However, **a filesystem-like mount does not make object storage behave exactly like a traditional filesystem**.

Potential differences include:
- Rename operations may be implemented as copy-and-delete.
- File locking and atomic filesystem operations may be limited or unavailable.
- Random writes and append operations may be unsupported or inefficient.
- Metadata operations and directory listings may have different performance characteristics.
- Consistency, caching, and concurrency behavior depend on the service and mount implementation.

For example, a FUSE-based object storage mount may work for reading datasets or accessing files through compatible tools, but it may be unsuitable for a database that expects standard filesystem semantics.

Prefer a native object-storage connector or SDK when the application supports it. Use a mount only after verifying the driver's supported operations and performance characteristics.

## Key points

- **PVs are Kubernetes storage resources; object storage is a storage service model.** They are not directly interchangeable.
- **PVs are consumed through PVCs.** Pods typically mount the claim into a container.
- **Object storage is usually accessed through APIs.** Applications use SDKs, HTTP requests, or compatible data-processing connectors.
- **PVs do not always mean local disks.** A PV may represent network-attached or cloud-backed storage.
- **Object storage is independent of the Kubernetes cluster.** Data can remain available to other compute platforms, subject to access controls and service availability.
- **Storage capacity alone does not determine the choice.** Consider access semantics, latency, throughput, concurrency, durability, cost, and application requirements.
- **Database workloads often need PV-backed storage.** Select a supported backend with appropriate latency, durability, recovery, and attachment characteristics.
- **Analytical datasets often fit object storage well.** Distributed processing engines can access datasets without mounting a dedicated volume into every worker.
- **PVs can be shared, but only when supported.** `ReadWriteMany` may allow multi-node mounting when the backend and driver support it. `ReadWriteOnce` permits read-write mounting by one node at a time, not necessarily just one Pod.
- **Object storage can support concurrent clients.** Applications still need to account for service-specific consistency, concurrency, and update semantics.
- **Object storage is not a database replacement.** It is not generally designed to provide the same transaction and random-update semantics as a database engine.
- **A PVC is not a backup.** A persistent volume can survive Pod deletion but can still be lost through backend failure, accidental deletion, or misconfiguration.
- **Object storage is not automatically a backup either.** Versioning, retention, replication, access controls, and recovery procedures must be configured appropriately.
- **Security matters for both options.** PVs need appropriate filesystem and storage permissions. Object storage needs identity-based access control, suitable credentials, and encryption settings.
- **Cost models differ.** PV-backed storage may incur provisioned-capacity and performance charges. Object storage often has capacity, request, retrieval, and transfer charges. Compare the actual provider's pricing.
- **Use both when appropriate.** A database can store operational state on a PV while exporting backups and analytical datasets to object storage.

## Commands / manifests

### 1. Inspect Kubernetes persistent storage

```bash
# List PersistentVolumes
kubectl get pv

# List PersistentVolumeClaims
kubectl get pvc -A

# Inspect StorageClasses
kubectl get storageclass

# Inspect a specific claim
kubectl describe pvc <pvc-name>

# Inspect a specific volume
kubectl describe pv <pv-name>
```

These commands inspect Kubernetes-managed persistent storage. They do not list the contents of an external object storage bucket.

### 2. Create a PVC for a filesystem-based workload

The following example requests a 10 GiB volume using a StorageClass named `standard`. Replace the class name with one available in your cluster.

Create `app-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Apply it:

```bash
kubectl apply -f app-pvc.yaml
kubectl get pvc app-data
kubectl describe pvc app-data
```

The claim may remain `Pending` if the StorageClass is unavailable, provisioning fails, or binding is waiting for a consumer Pod.

### 3. Mount the PVC into a Pod

Create `app-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: filesystem-demo
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
        claimName: app-data
```

Apply and test:

```bash
kubectl apply -f app-pod.yaml

kubectl exec filesystem-demo -- sh -c \
  'echo "stored on persistent volume" > /data/example.txt'

kubectl exec filesystem-demo -- cat /data/example.txt
```

The file is written to the mounted volume rather than only to the container's writable layer.

### 4. Access object storage from a Kubernetes application

Object storage access depends on the cloud provider and application. A common pattern is to use an SDK or a command-line client with credentials supplied through the platform's identity mechanism.

For example, an application might upload a dataset to an object storage bucket using a provider SDK:

```python
# Illustrative Python example using boto3 and Amazon S3.
# Requires boto3, suitable AWS credentials, and an existing bucket.

import boto3

s3 = boto3.client("s3")

s3.upload_file(
    "claims.parquet",
    "my-data-lake-bucket",
    "raw/claims.parquet",
)
```

Here:
- `claims.parquet` is a local file available to the application.
- `my-data-lake-bucket` is an existing S3 bucket.
- `raw/claims.parquet` is the object's key.

The application authenticates to the object storage service and uploads the object through the API. It does not need a PV solely to access the bucket.

In Kubernetes, prefer workload identity or the cloud provider's supported identity integration over hardcoding access keys in source code. The example uses Amazon S3; other providers have different SDKs, endpoints, and identity configurations.

### 5. Compare the two approaches in a workload

**PV-backed filesystem**

```text
Pod
 |
 v
PVC
 |
 v
PV
 |
 v
Mounted path: /data
 |
 v
Read/write files
```

Typical operations:

```bash
ls -lah /data
cat /data/example.txt
echo "new record" >> /data/example.txt
```

These are normal filesystem operations on the mounted volume.

**Object storage API**

```text
Pod
 |
 v
Application SDK
 |
 v
Object storage API
 |
 v
Bucket / object key
```

Typical operations using an S3-compatible CLI, if installed and configured:

```bash
# List objects in a bucket
aws s3 ls s3://my-data-lake-bucket/

# Upload a file
aws s3 cp claims.parquet \
  s3://my-data-lake-bucket/raw/claims.parquet

# Download a file
aws s3 cp \
  s3://my-data-lake-bucket/raw/claims.parquet \
  ./claims.parquet
```

These commands require the AWS CLI, valid credentials, suitable permissions, and an existing bucket. The CLI examples are specific to Amazon S3 and compatible services.

### 6. Validate which storage model your application needs

Before choosing a storage backend, ask:

```text
Does the application require a mounted filesystem
or block device?
       |
       +-- Yes --> Evaluate PV-backed storage
       |
       +-- No
            |
            v
Can the application access data through an object API?
       |
       +-- Yes --> Evaluate object storage
       |
       +-- No --> Investigate the application's storage requirements
```

Then evaluate:
- Required read/write semantics.
- Latency and throughput.
- Single-node versus multi-node access.
- Data size and object count.
- Durability, backup, and recovery requirements.
- Security and identity integration.
- Total cost, including requests and data transfer.

## Common failures & fixes

- **A database fails when using object storage as its data directory**
  - Cause: The database expects filesystem or block-storage semantics that object storage does not provide.
  - Fix: Use a supported persistent volume for the database files. Export backups or analytical snapshots to object storage separately.

- **A Pod cannot mount a PVC**
  - Cause: The claim is not bound, the CSI driver is unavailable, or the storage backend cannot attach or mount the volume.
  - Fix: Run `kubectl describe pvc <pvc-name>` and `kubectl describe pod <pod-name>`. Inspect events and storage-driver health.

- **An application cannot access an object storage bucket**
  - Cause: Credentials are missing, the workload identity is misconfigured, or the application lacks permissions.
  - Fix: Verify the workload's identity, bucket or container name, endpoint, and minimum required permissions. Avoid putting long-lived credentials directly into source code.

- **A mounted object storage filesystem behaves differently from a normal filesystem**
  - Cause: The mount driver translates filesystem operations into object API calls and may not support all filesystem semantics.
  - Fix: Review the driver's documented limitations. Use a native SDK or data-processing connector when appropriate, particularly for workloads that require atomic updates, file locking, or random writes.

- **A data lake pipeline is slow when reading many small objects**
  - Cause: Large numbers of object requests, listing operations, and request latency create overhead.
  - Fix: Where appropriate, compact small files into larger columnar files such as Parquet, partition datasets sensibly, and use efficient listing and query strategies.

- **Object uploads fail with access-denied errors**
  - Cause: The workload identity lacks the necessary bucket or object permissions, or a bucket policy blocks the operation.
  - Fix: Inspect the cloud provider's authorization and audit logs. Grant only the permissions needed for the intended operations.

- **A PV is lost after a Pod is deleted**
  - Cause: The application may have written data to the container filesystem instead of the mounted path, or the backing storage may have been deleted or lost.
  - Fix: Verify the container's `volumeMounts` and `mountPath`, inspect the PVC and PV, and check the reclaim policy. Maintain independent backups.

- **Object data disappears after an application overwrite or deletion**
  - Cause: The application replaced or deleted an object, or a lifecycle rule removed it.
  - Fix: Review application logs, object versioning, lifecycle rules, retention settings, and audit logs. Configure versioning or retention when appropriate and test recovery.

- **The same object is read differently by multiple applications**
  - Cause: The applications may use different object keys, cached data, or service-specific consistency and update behavior.
  - Fix: Confirm the exact bucket and key, review caching and update semantics, and use an application-level coordination strategy where needed.

- **Storage costs are unexpectedly high**
  - Cause: Provisioned disk capacity, performance tiers, object requests, retrieval charges, replication, or data transfer may exceed expectations.
  - Fix: Review billing and usage metrics. Select a storage tier suited to access patterns, implement lifecycle policies where appropriate, and remove unused resources safely.

## References

- [Kubernetes documentation: Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes documentation: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes documentation: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Amazon S3 documentation: What is Amazon S3?](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Microsoft Azure documentation: Introduction to Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction)
- [Google Cloud documentation: Cloud Storage overview](https://cloud.google.com/storage/docs/introduction)
- [Kubernetes documentation: Container Storage Interface (CSI)](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
