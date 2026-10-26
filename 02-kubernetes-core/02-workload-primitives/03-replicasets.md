# ReplicaSets

> Phase 2 — Kubernetes Core (highest priority) › Workload primitives

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **ReplicaSet is a Kubernetes workload resource that maintains a specified number of running Pod replicas matching a selector**.

Its primary responsibility is to ensure that the desired number of matching Pods exists. If a managed Pod is deleted or becomes unavailable, the ReplicaSet controller works to create a replacement and restore the desired replica count.

ReplicaSets are most commonly managed by Deployments rather than created directly.

### 1. Why ReplicaSets exist

Imagine that your application should run three replicas:

```yaml
replicas: 3
```

If you manually create three Pods, Kubernetes does not automatically know that you intend to maintain exactly three copies unless a controller manages them.

A ReplicaSet expresses that requirement. Its controller continuously compares the desired replica count with the actual number of matching Pods.

For example:

| Event | Desired replicas | Actual replicas | Controller behavior |
|---|---:|---:|---|
| Initial deployment | 3 | 0 | Creates Pods |
| All Pods are created | 3 | 3 | Maintains the replica count |
| One Pod is deleted | 3 | 2 | Creates a replacement Pod |
| A fourth matching Pod appears | 3 | 4 | Attempts to reduce the count to three |
| Desired count is scaled to five | 5 | 3 | Creates additional Pods |

The controller works toward the desired state, but recovery is not necessarily instantaneous. Scheduling constraints, resource shortages, image-pull failures, and other problems can prevent replacement Pods from becoming ready.

**Important:** A ReplicaSet maintains the number of matching Pods, not necessarily the number of healthy, ready application instances. Readiness and availability are separate considerations.

### 2. Where ReplicaSets fit in Kubernetes

The relationship between workload resources is:

`Deployment → ReplicaSet → Pods → Containers`

Each resource has a distinct responsibility:

| Resource | Responsibility |
|---|---|
| Deployment | Manages application rollouts, revisions, and the desired application configuration |
| ReplicaSet | Maintains the desired number of matching Pods |
| Pod | Provides the environment in which containers run |
| Container | Runs the application process |

Suppose you create a Deployment with three replicas.

1. The Deployment controller creates a ReplicaSet for the specified Pod template.
2. The ReplicaSet controller creates Pods until the desired replica count is reached.
3. If a Pod is deleted, the ReplicaSet controller works to create a replacement.
4. If you update the Deployment's Pod template, the Deployment controller creates a new ReplicaSet for the updated template.
5. During a rolling update, the Deployment coordinates scaling between the old and new ReplicaSets.

This separation lets Kubernetes maintain replica counts while providing a higher-level mechanism for application updates.

### 3. How a ReplicaSet identifies Pods

A ReplicaSet uses a **label selector** to identify the Pods it manages.

Example:

```yaml
selector:
  matchLabels:
    app: payments
```

This selector matches Pods labelled:

```yaml
metadata:
  labels:
    app: payments
```

The ReplicaSet also defines a Pod template describing how to create new Pods.

Example:

```yaml
template:
  metadata:
    labels:
      app: payments
  spec:
    containers:
      - name: app
        image: nginx:1.28
```

For an `apps/v1` ReplicaSet, the Pod-template labels must satisfy its selector.

A ReplicaSet's selector is important because it determines which Pods count toward its desired replica count. Poorly designed selectors can cause unexpected behavior if multiple controllers are configured to match the same Pods.

**Best practice:** Use specific, intentional labels and selectors. Avoid creating overlapping ReplicaSets with selectors that unintentionally match the same Pods.

### 4. ReplicaSet manifest

A ReplicaSet can be defined directly using a manifest:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-rs
  template:
    metadata:
      labels:
        app: web-rs
    spec:
      containers:
        - name: web
          image: nginx:1.28
          ports:
            - containerPort: 80
```

This manifest specifies:

- A ReplicaSet named `web-rs`.
- Three desired Pod replicas.
- A selector matching Pods labelled `app: web-rs`.
- A Pod template defining the container image and port.

When you apply the manifest, the ReplicaSet controller works toward creating three matching Pods.

The image tag is illustrative. Use a valid tag available in your environment.

Direct ReplicaSet creation is useful for learning the controller's behavior. For most application workloads, a Deployment is preferred because it manages ReplicaSets and supports rollouts and rollbacks.

### 5. How Deployments use ReplicaSets

A Deployment manages ReplicaSets to support controlled application updates.

Suppose the initial Deployment uses:

```yaml
image: nginx:1.28
replicas: 3
```

Kubernetes creates an associated ReplicaSet that manages three Pods using that template.

Later, you update the image:

```yaml
image: nginx:1.29
```

The Deployment controller detects the change to the Pod template and creates a new ReplicaSet for the updated version.

During a rolling update, the Deployment adjusts the old and new ReplicaSets according to its rollout strategy.

For example:

| Stage | Old ReplicaSet | New ReplicaSet | Total desired replicas |
|---|---:|---:|---:|
| Before update | 3 | 0 | 3 |
| Rollout starts | 2 | 1 | 3 |
| Rollout progresses | 1 | 2 | 3 |
| Rollout completes | 0 | 3 | 3 |

These numbers illustrate a rollout with no surge and no unavailable replicas being enforced as a strict requirement. Actual intermediate counts depend on the Deployment's strategy, readiness, scheduling, and available capacity.

The old ReplicaSet can remain in the cluster with zero replicas so that the Deployment can retain revision history and support rollback.

**Key distinction:** A ReplicaSet does not independently perform Deployment-style rolling updates. The Deployment controller coordinates the transition between ReplicaSets.

### 6. ReplicaSet scaling

The `spec.replicas` field specifies the desired number of Pods.

You can change the replica count using a manifest or an imperative command.

For example:

```bash
kubectl scale replicaset/web-rs --replicas=5
```

The ReplicaSet controller then works toward maintaining five matching Pods.

However, when a ReplicaSet is managed by a Deployment, manually scaling the ReplicaSet is generally not the correct way to scale the application. The Deployment controller manages the ReplicaSet's desired size and may overwrite manual changes.

For Deployment-managed applications, scale the Deployment instead:

```bash
kubectl scale deployment/web-app --replicas=5
```

This lets the Deployment controller manage the appropriate ReplicaSet or ReplicaSets.

### 7. What happens when a Pod is deleted?

Suppose a ReplicaSet maintains three Pods:

```text
ReplicaSet: web-rs
Desired replicas: 3

Pod A: Running
Pod B: Running
Pod C: Running
```

You delete Pod B:

```bash
kubectl delete pod <pod-b-name>
```

The ReplicaSet controller observes that fewer matching Pods exist than desired and creates a replacement.

The replacement is a new Pod object with a new name and normally a new UID. It may receive a different IP address and may be scheduled onto a different node.

This is a reconciliation process, not an immediate restoration guarantee. The replacement may remain Pending or fail to start if the cluster cannot run it successfully.

Deleting the Pod does not delete the ReplicaSet. The ReplicaSet continues to exist and maintains the configured replica count.

### 8. ReplicaSet vs Deployment

| Capability | ReplicaSet | Deployment |
|---|---|---|
| Maintain a desired Pod replica count | Yes | Indirectly, through ReplicaSets |
| Create replacement Pods when replicas fall below the target | Yes | Through its ReplicaSets |
| Manage application rollouts | No independent rollout orchestration | Yes |
| Maintain rollout revision history | Not as a Deployment revision history | Yes |
| Roll back an application update | No Deployment-style rollback mechanism | Yes, when the relevant revision is available |
| Recommended for most stateless application deployments | Usually not directly | Yes |

ReplicaSets are essential to understanding how Deployments work, but you will usually create a Deployment rather than a standalone ReplicaSet.

## Key points

- A ReplicaSet maintains a desired number of matching Pod replicas.
- Its controller continuously reconciles the actual Pod count toward `spec.replicas`.
- A ReplicaSet uses a label selector to identify the Pods it manages.
- Its Pod template defines how replacement Pods are created.
- The selector must match the labels in the Pod template.
- Deleting a managed Pod normally causes the ReplicaSet controller to create a replacement.
- A replacement Pod is a new Kubernetes object and may have a different name, UID, IP address, and node.
- A ReplicaSet maintains replica count, not guaranteed application health or readiness.
- A ReplicaSet does not independently manage rolling updates between application versions.
- A Deployment creates and manages ReplicaSets to coordinate rollouts and rollbacks.
- During a Deployment update, old and new ReplicaSets may coexist temporarily.
- Old ReplicaSets may remain with zero replicas to preserve Deployment revision history.
- For a Deployment-managed application, scale the Deployment rather than manually scaling its ReplicaSet.
- Creating overlapping ReplicaSet selectors can lead to unexpected Pod ownership and reconciliation behavior.
- Direct ReplicaSets are useful for learning and specialized use cases, but Deployments are the standard choice for most stateless applications.

## Commands / manifests

### 1. Create a ReplicaSet

Save this as `web-replicaset.yaml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-rs
  template:
    metadata:
      labels:
        app: web-rs
    spec:
      containers:
        - name: web
          image: nginx:1.28
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f web-replicaset.yaml
```

Inspect the ReplicaSet and Pods:

```bash
kubectl get replicasets
kubectl get pods -l app=web-rs
```

### 2. Inspect ReplicaSet details

```bash
kubectl describe replicaset web-rs
```

Review:

- Desired and current replicas.
- Ready and available replicas.
- Label selector.
- Pod template.
- Events related to Pod creation or scheduling.

To inspect the resource configuration:

```bash
kubectl get replicaset web-rs -o yaml
```

### 3. Scale a standalone ReplicaSet

Increase replicas to five:

```bash
kubectl scale replicaset/web-rs --replicas=5
```

Verify:

```bash
kubectl get replicaset web-rs
kubectl get pods -l app=web-rs
```

Scale it back to two:

```bash
kubectl scale replicaset/web-rs --replicas=2
```

### 4. Observe automatic Pod replacement

List the Pods managed by the ReplicaSet:

```bash
kubectl get pods -l app=web-rs
```

Choose one Pod name and delete that Pod:

```bash
kubectl delete pod <pod-name>
```

Immediately check the replica count:

```bash
kubectl get replicaset web-rs
kubectl get pods -l app=web-rs -w
```

The controller should work toward restoring three replicas. The replacement may take time to schedule and start.

### 5. Inspect Pod ownership

```bash
kubectl get pods -l app=web-rs \
  -o custom-columns=NAME:.metadata.name,OWNER:.metadata.ownerReferences[0].name,PHASE:.status.phase
```

The owner reference should identify the ReplicaSet responsible for the Pods.

For a Deployment-managed Pod, the direct owner is normally a ReplicaSet, while the ReplicaSet is itself owned by the Deployment.

### 6. Observe how a Deployment manages ReplicaSets

Create a Deployment:

```bash
kubectl create deployment web-app --image=nginx:1.28
```

Scale it to three replicas:

```bash
kubectl scale deployment/web-app --replicas=3
```

Inspect the hierarchy:

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -l app=web-app
```

Update the image:

```bash
kubectl set image deployment/web-app nginx=nginx:1.29
```

Here, `nginx` is the container name generated by `kubectl create deployment` in this example. If you use a different manifest, substitute the actual container name.

Inspect the old and new ReplicaSets:

```bash
kubectl get replicasets -l app=web-app
kubectl rollout status deployment/web-app
```

After the rollout, the new ReplicaSet should maintain the desired replicas, while the old ReplicaSet may remain at zero replicas.

### 7. Inspect a Deployment's ReplicaSet ownership

```bash
kubectl get replicasets -l app=web-app \
  -o custom-columns=NAME:.metadata.name,DESIRED:.spec.replicas,CURRENT:.status.replicas,READY:.status.readyReplicas
```

This helps you observe how the Deployment's rollout changes the desired replica counts across ReplicaSets.

### 8. Clean up the practice resources

Delete the standalone ReplicaSet:

```bash
kubectl delete -f web-replicaset.yaml
```

Delete the Deployment:

```bash
kubectl delete deployment web-app
```

Deleting a ReplicaSet also deletes the Pods it manages. Deleting a Deployment removes its managed ReplicaSets and associated Pods as part of workload cleanup.

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| ReplicaSet has fewer Pods than requested | Pod creation or scheduling failures | Run `kubectl describe replicaset <name>` and inspect Pod events |
| ReplicaSet shows three replicas, but some Pods are not ready | Containers are failing, starting slowly, or failing readiness probes | Inspect Pod descriptions, events, and logs |
| ReplicaSet creates no Pods | Invalid Pod template, selector mismatch, or controller-related issue | Inspect the ReplicaSet manifest and events |
| `selector does not match template labels` | Pod-template labels do not satisfy the ReplicaSet selector | Make the selector and template labels consistent |
| Pods remain `Pending` | Insufficient resources, node constraints, or storage issues | Run `kubectl describe pod <pod-name>` |
| Pods show `ImagePullBackOff` | Invalid image, inaccessible registry, or authentication problem | Check the image reference and Pod events |
| ReplicaSet creates more Pods than expected | Selector matches additional Pods or multiple controllers overlap | Inspect labels, selectors, and Pod owner references |
| Manually scaled ReplicaSet returns to its previous size | A Deployment controller manages that ReplicaSet | Scale the Deployment instead |
| Old ReplicaSets remain after an update | Deployment revision history is being retained | Inspect the Deployment's rollout history; this is normal |
| Deleted Pod is not immediately replaced | Controller reconciliation or Pod creation is delayed, or the controller cannot create a viable replacement | Inspect ReplicaSet events, Pod status, and cluster health |
| A new Pod appears but the application is still unavailable | Replica count is restored, but the new Pod is not ready or healthy | Check readiness, logs, and Service endpoints |
| ReplicaSet update does not change existing Pods' configuration | ReplicaSet changes do not provide Deployment-style rollout orchestration | Update the workload through a Deployment and let it create a new ReplicaSet |

### Troubleshooting sequence

Start by inspecting the ReplicaSet:

```bash
kubectl get replicaset <replicaset-name>
kubectl describe replicaset <replicaset-name>
```

Then inspect the matching Pods:

```bash
kubectl get pods -l app=<label-value> -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

For Pods that have restarted, inspect the previous container instance where applicable:

```bash
kubectl logs <pod-name> --previous
```

Check the following in order:

1. Is the desired replica count correct?
2. Does the selector match the intended Pod labels?
3. Is the ReplicaSet creating Pods?
4. Are the Pods being scheduled successfully?
5. Are the containers starting successfully?
6. Are the Pods ready and available?
7. Is a Deployment controlling this ReplicaSet and reconciling its replica count?

Avoid manually editing a Deployment-managed ReplicaSet to fix an application rollout. Make the intended change to the Deployment instead.

## References

- [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [kubectl scale](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/)
- [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Debugging Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
