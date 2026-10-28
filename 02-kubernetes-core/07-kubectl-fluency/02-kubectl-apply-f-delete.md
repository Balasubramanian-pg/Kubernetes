# kubectl apply -f, delete

> Phase 2 — Kubernetes Core (highest priority) › kubectl fluency

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

`kubectl apply -f` creates or updates Kubernetes resources using configuration files. `kubectl delete` removes Kubernetes resources from the cluster.

These commands are fundamental to managing Kubernetes applications declaratively. Instead of manually creating individual resources, you define the desired state in YAML manifests and use `kubectl apply` to make the cluster match that state.

For example, you can define a Deployment in `deployment.yaml` with three replicas. Running `kubectl apply -f deployment.yaml` creates the Deployment if it does not exist or updates it if it already exists. Kubernetes controllers then work to achieve the desired state.

`kubectl delete` removes resources by specifying a manifest, resource type and name, or other supported selectors.

**Declarative management with `apply`:**

1. Write the desired configuration in a YAML file.
2. Run `kubectl apply -f` to submit the configuration.
3. Kubernetes creates or updates the resource.
4. Controllers reconcile the actual cluster state with the desired state.

**Deletion with `kubectl delete`:**

1. Identify the resource to remove.
2. Run `kubectl delete` with the appropriate resource reference.
3. Kubernetes processes the deletion request.
4. Controllers and dependent systems respond to the resource's removal.

A key distinction is that `kubectl apply` manages the desired configuration, while `kubectl delete` requests removal of resources. Neither command guarantees that every downstream effect has completed immediately.

## Key points

- **`kubectl apply -f`:** Creates resources that do not exist and updates existing resources based on the supplied configuration.
- **`kubectl delete -f`:** Deletes resources described by the specified manifest files. It does not delete only the fields that changed or merely undo the last `apply`.
- **Declarative vs imperative commands:**
  - `kubectl apply -f deployment.yaml` manages resources through configuration files.
  - `kubectl create deployment web --image=nginx` creates a Deployment directly through a command.
  - `kubectl delete deployment web` deletes a resource by type and name.
- **Applying changes:** Updating a Deployment's container image in its YAML and applying the file can trigger a rolling update.
- **Applying unchanged configuration:** Reapplying a manifest is generally safe and does not necessarily restart Pods. A new rollout occurs only when a relevant Pod template change or another triggering condition occurs.
- **Multiple manifests:** `kubectl apply -f` accepts a file, a directory containing manifests, or a URL to a manifest.
- **Namespaces:** Resources can be created, updated, or deleted in a specific namespace using `-n`. If no namespace is supplied, kubectl uses the namespace configured in the current context, which is often `default`.
- **Previewing changes:** `kubectl diff -f` displays differences between the manifest and the live configuration before applying.
- **Server-side dry run:** `kubectl apply --dry-run=server -f` asks the API server to validate and process the request without persisting the changes.
- **Deletion scope:** `kubectl delete -f` deletes the resources represented by the supplied files. It does not automatically delete every resource that was once created from a file but is no longer present in that file.
- **Removing a manifest from a directory:** If a resource is removed from a directory's manifests, applying that directory does not automatically delete the resource from the cluster. Use an appropriate deletion workflow.
- **Deletion is not always immediate:** Kubernetes may take time to terminate Pods, release storage, or complete finalizers. A resource can remain in `Terminating` while cleanup proceeds.
- **Graceful deletion:** Kubernetes normally allows a grace period for supported resources to shut down cleanly. Avoid using `--force` unless you understand the consequences.
- **Resource dependencies:** Deleting a Deployment typically deletes its managed ReplicaSets and Pods through owner references and garbage collection. Deleting a PVC can affect access to persistent data, depending on the storage configuration and reclaim policy.
- **Configuration management:** Keep manifests in version control so changes can be reviewed, reproduced, and rolled back deliberately.
- **`apply` is not a universal merge:** The exact update behavior depends on the resource type, field ownership, patch strategy, and whether client-side or server-side apply is used.
- **Server-side apply:** `kubectl apply --server-side -f` uses server-side field ownership and conflict detection. This is useful when multiple tools or teams manage different fields of the same resource.
- **Practical rule:** Preview changes when possible, apply the intended manifest, verify the resulting state, and confirm the exact target before deleting anything.

## Commands / manifests

### 1. Create a working directory

```bash
mkdir kubectl-apply-demo
cd kubectl-apply-demo
```

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
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
          image: nginx:1.28
          ports:
            - containerPort: 80
```

This manifest defines a Deployment named `web` that should maintain two replicas of an NGINX container.

Important fields:

- `apiVersion`: API version for the Kubernetes resource.
- `kind`: Resource type, in this case `Deployment`.
- `metadata.name`: Name of the Deployment.
- `spec.replicas`: Desired number of Pods.
- `spec.selector.matchLabels`: Identifies Pods managed by the Deployment.
- `spec.template`: Defines the Pod configuration created by the Deployment.
- `containers.image`: Container image used by each Pod.

The Deployment selector must match the labels in `spec.template.metadata.labels`.

### 2. Apply the manifest

```bash
kubectl apply -f deployment.yaml
```

Expected output on first creation:

```text
deployment.apps/web created
```

If the Deployment already exists and its configuration changes, the output will typically indicate that it was configured.

If no relevant change is detected, kubectl may report that it is unchanged.

### 3. Verify the resource

```bash
kubectl get deployments
kubectl get pods
kubectl describe deployment web
```

What these commands do:

- `kubectl get deployments`: Lists Deployments in the current namespace.
- `kubectl get pods`: Lists Pods in the current namespace.
- `kubectl describe deployment web`: Displays the Deployment's configuration, conditions, and events.

Wait until the Deployment reports that the desired replicas are available. Container image downloads and Pod startup can take time.

### 4. Preview changes before applying

```bash
kubectl diff -f deployment.yaml
```

This compares the configuration in the manifest with the live resource.

An exit code of `1` from `kubectl diff` normally means differences were found, not that the command failed. Other nonzero exit codes can indicate errors.

To validate the manifest against the API server without saving the change:

```bash
kubectl apply --dry-run=server -f deployment.yaml
```

This checks the request through the server-side validation and admission process without persisting the resource.

### 5. Update the Deployment

Change the replica count in `deployment.yaml` from `2` to `3`, then run:

```bash
kubectl apply -f deployment.yaml
kubectl get deployment web
kubectl get pods
```

Kubernetes will reconcile the Deployment toward three replicas.

To change the image in the manifest from `nginx:1.28` to another intended version, then apply it again:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/web
```

A change to the Pod template, such as the container image, normally triggers a Deployment rollout. A replica-count-only change adjusts scaling and does not normally create a new Deployment revision.

### 6. Apply multiple manifests

Suppose a directory contains:

```text
k8s/
├── deployment.yaml
└── service.yaml
```

Apply the supported manifests in the directory:

```bash
kubectl apply -f k8s/
```

This is useful when an application consists of multiple Kubernetes resources.

Be aware that applying a directory does not automatically remove resources whose manifest files have been deleted from that directory.

### 7. Apply resources to a specific namespace

```bash
kubectl apply -f deployment.yaml -n development
kubectl get deployments -n development
```

The namespace must exist unless it is created separately or included as a resource in an appropriate workflow.

For a namespace-specific manifest, you can also specify the namespace under `metadata`:

```yaml
metadata:
  name: web
  namespace: development
```

Using `-n` is useful for targeting a namespace, but be careful not to apply production resources to the wrong environment.

### 8. Use server-side apply

```bash
kubectl apply --server-side -f deployment.yaml
```

Server-side apply tracks field ownership on the API server. If another manager owns a field that your request attempts to change, Kubernetes can report a field ownership conflict.

Inspect field ownership and managed fields when diagnosing such conflicts. Do not automatically use `--force-conflicts` without understanding which manager's ownership will be overridden.

### 9. Delete using the manifest

```bash
kubectl delete -f deployment.yaml
```

This requests deletion of the resource described in `deployment.yaml`.

Verify the result:

```bash
kubectl get deployments
kubectl get pods
```

If the file describes multiple resources, kubectl attempts to delete those resources.

**Important:** Keep a copy of the manifest until you have confirmed which resources it describes. Deleting by file does not mean “undo only the latest apply”; it means delete the resources represented by that file.

### 10. Delete by resource type and name

```bash
kubectl delete deployment web
```

This deletes the Deployment named `web` in the current namespace.

To delete a Pod directly:

```bash
kubectl delete pod <pod-name>
```

If a Pod is managed by a Deployment or another controller, the controller may create a replacement Pod. Delete or scale down the controller if the goal is to stop the workload rather than replace one Pod.

### 11. Delete a resource from a specific namespace

```bash
kubectl delete deployment web -n development
```

Use the namespace explicitly when working across development, staging, and production environments.

Before a destructive operation, inspect the active context and target resource:

```bash
kubectl config current-context
kubectl get deployment web -n development
```

### 12. Delete resources using labels

```bash
kubectl delete pods -l app=web
```

This deletes Pods in the current namespace whose labels match `app=web`.

Label selectors can match multiple resources, so inspect the target set before deleting:

```bash
kubectl get pods -l app=web
```

Deleting Pods selected by a label does not necessarily stop an application. A Deployment may recreate them.

### 13. Wait for deletion to complete

```bash
kubectl delete deployment web --wait=true
```

The default behavior waits for the deletion to complete, subject to kubectl's waiting behavior and timeout settings.

For a resource that is taking too long to disappear:

```bash
kubectl get deployment web -o yaml
kubectl describe deployment web
```

For Pods stuck in `Terminating`:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

Inspect events, finalizers, storage detach operations, and controller behavior before considering force deletion.

### 14. Hands-on exercise

Perform this sequence in a development cluster:

1. Save the Deployment manifest as `deployment.yaml`.
2. Apply it with `kubectl apply -f deployment.yaml`.
3. Verify that two replicas become available.
4. Change `replicas` from `2` to `3`.
5. Run `kubectl diff -f deployment.yaml`.
6. Apply the updated manifest.
7. Verify that three replicas are available.
8. Change the image version in the manifest and apply it.
9. Monitor the rollout using `kubectl rollout status deployment/web`.
10. Delete the Deployment using `kubectl delete -f deployment.yaml`.
11. Verify that the Deployment and its managed Pods are removed.

Do this in a non-production namespace. If you are using a shared cluster, confirm the namespace and resource ownership before running deletion commands.

## Common failures & fixes

- **`error: the path "deployment.yaml" does not exist`**
  - Cause: The file is missing or kubectl is running from a different directory.
  - Fix: Run `pwd` and `ls`, then provide the correct relative or absolute file path.

- **`error: resource mapping not found`**
  - Cause: The manifest's API version or resource kind is unsupported or incorrect, or the required Custom Resource Definition is not installed.
  - Fix: Check the `apiVersion` and `kind`. Use `kubectl api-resources` to inspect supported resource types and install the required CRD when appropriate.

- **`no matches for kind ...`**
  - Cause: The API resource is unavailable, commonly because a CRD has not been installed or the manifest uses an invalid API version.
  - Fix: Verify that the API is registered and that the manifest uses the correct API version.

- **`AlreadyExists` when using `kubectl create -f`**
  - Cause: The resource already exists and `create` does not update it.
  - Fix: Use `kubectl apply -f` when the intention is to manage an existing resource declaratively.

- **`Forbidden` or `Unauthorized`**
  - Cause: The current identity lacks permission, or the credentials are invalid or expired.
  - Fix: Check `kubectl config current-context`, authenticate if necessary, and inspect permissions with `kubectl auth can-i create deployments` or the relevant verb and resource.

- **The manifest applies, but the Pods do not start**
  - Cause: Image pull failures, insufficient cluster capacity, invalid configuration, or other scheduling or runtime problems.
  - Fix: Run `kubectl get pods`, `kubectl describe pod <pod-name>`, and `kubectl get events --sort-by=.metadata.creationTimestamp`.

- **`Invalid` or schema validation errors**
  - Cause: Incorrect YAML structure, invalid field names, incompatible values, or a mismatch between the manifest and the API schema.
  - Fix: Check indentation and field names, use `kubectl apply --dry-run=server -f deployment.yaml`, and consult the documentation for the resource's API version.

- **Deployment selector does not match Pod labels**
  - Cause: `spec.selector.matchLabels` does not match `spec.template.metadata.labels`.
  - Fix: Make the selector and Pod-template labels consistent. Deployment selectors are generally immutable after creation, so recreating a resource may be necessary to correct an existing selector.

- **Changes do not appear after applying**
  - Cause: The wrong file or namespace was applied, the configuration did not change, or the changed field does not trigger the expected behavior.
  - Fix: Run `kubectl diff -f deployment.yaml`, inspect the live object with `kubectl get deployment web -o yaml`, and verify the namespace and current context.

- **Pods keep restarting after an image update**
  - Cause: The new image is invalid, the application crashes, or its configuration or probes fail.
  - Fix: Check `kubectl rollout status deployment/web`, `kubectl describe pod <pod-name>`, and `kubectl logs <pod-name>`. Roll back a Deployment if the previous revision is known to be healthy.

- **A resource remains in `Terminating`**
  - Cause: Graceful shutdown, finalizers, volume detachment, or other cleanup operations have not completed.
  - Fix: Inspect the resource YAML and events. Identify the reason before modifying finalizers or attempting force deletion.

- **A deleted Pod reappears**
  - Cause: A Deployment, StatefulSet, DaemonSet, Job, or another controller is maintaining the workload.
  - Fix: Manage the owning controller, such as scaling down or deleting the Deployment, rather than repeatedly deleting individual Pods.

- **`kubectl delete -f` deletes more than expected**
  - Cause: The manifest contains multiple resources or the file points to a broader set of resources than intended.
  - Fix: Review every `kind`, `metadata.name`, and `metadata.namespace` in the file before deletion. Use `kubectl get -f deployment.yaml` to inspect the resources identified by the manifest when supported.

- **PVC deletion threatens persistent data**
  - Cause: A PVC may be tied to a persistent volume whose reclaim policy or storage backend determines what happens to the underlying data.
  - Fix: Check the PVC, bound PV, storage class, reclaim policy, and backup requirements before deletion. Do not assume that deleting a PVC is harmless.

- **Server-side apply reports a field ownership conflict**
  - Cause: Another manager owns a field that the current apply request attempts to change.
  - Fix: Inspect `managedFields` and coordinate field ownership. Use `--force-conflicts` only when intentionally taking ownership.

- **A resource is deleted from a manifest directory, but remains in the cluster**
  - Cause: Ordinary `kubectl apply -f directory/` does not prune resources removed from the directory.
  - Fix: Explicitly delete the obsolete resource or use a deliberate lifecycle-management workflow. Understand the risks before enabling pruning or adopting a declarative management tool.

## References

- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [kubectl delete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/)
- [kubectl diff](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_diff/)
- [Declarative Management of Kubernetes Objects Using Configuration Files](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
