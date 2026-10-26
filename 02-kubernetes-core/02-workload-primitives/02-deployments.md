# Deployments

> Phase 2 — Kubernetes Core (highest priority) › Workload primitives

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **Deployment is a Kubernetes workload resource that declaratively manages replicated application Pods and supports controlled updates and rollbacks**.

Instead of manually creating Pods, restarting failed instances, or updating each instance individually, you declare the desired state in a Deployment manifest. Kubernetes controllers continuously work to make the actual state match that declaration.

For example, you can specify that your web application should have three replicas running a particular container image. Kubernetes then manages the ReplicaSets and Pods required to achieve that state.

### 1. Why Deployments exist

Creating a standalone Pod is useful for learning and debugging, but it is not sufficient for most long-running applications.

Consider an application that requires three running instances.

Without a Deployment, you would need to manage tasks such as:

- Creating the required Pods.
- Replacing Pods deleted or lost during node failures.
- Increasing or decreasing the number of replicas.
- Updating the application version.
- Recovering from an unsuccessful rollout.
- Maintaining the desired number of replicas.

A Deployment automates much of this through Kubernetes controllers.

You declare the desired state, and Kubernetes continuously reconciles the cluster toward it.

### 2. Deployment, ReplicaSet, and Pod

A Deployment does not directly create and manage application containers. It manages ReplicaSets, and ReplicaSets manage Pods.

The hierarchy is:

`Deployment → ReplicaSet → Pods → Containers`

Each layer has a distinct responsibility.

| Resource | Responsibility |
|---|---|
| Deployment | Declares the desired application state and manages rollouts and rollbacks |
| ReplicaSet | Maintains the desired number of matching Pods |
| Pod | Provides the execution environment for one or more containers |
| Container | Runs the application process |

Suppose a Deployment specifies:

```yaml
replicas: 3
```

The Deployment controller ensures that an appropriate ReplicaSet exists. The ReplicaSet controller then works toward maintaining three matching Pods.

If one Pod is deleted, the ReplicaSet controller attempts to create a replacement.

If you change the Pod template, such as updating the container image, the Deployment normally creates a new ReplicaSet and coordinates the rollout.

**Important:** ReplicaSets maintain replica counts, but Deployments add rollout management, revision history, and declarative update capabilities.

### 3. Declarative management

Kubernetes supports declarative configuration, where you describe the desired state rather than issuing every individual operational step.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:1.28
          ports:
            - containerPort: 80
```

This manifest describes a Deployment named `web-app` that should maintain three replicas using the specified container image.

Apply the manifest:

```bash
kubectl apply -f deployment.yaml
```

Kubernetes compares the desired configuration with the existing resource and applies the required changes.

This differs from an imperative command such as:

```bash
kubectl create deployment web-app --image=nginx:1.28
```

The imperative command creates a Deployment using command-line arguments. A declarative manifest is easier to version-control, review, reproduce, and manage as part of a broader application configuration.

`kubectl apply` is commonly used to manage declarative configuration, although it is not the only way to modify a Deployment.

### 4. The Deployment manifest

The most important fields are:

- `apiVersion: apps/v1`: Identifies the API group and version.
- `kind: Deployment`: Specifies the resource type.
- `metadata.name`: Identifies the Deployment within its namespace.
- `spec.replicas`: Declares the desired number of replicas.
- `spec.selector`: Identifies the Pods managed by the Deployment's ReplicaSets.
- `spec.template`: Defines the Pod template used to create replicas.
- `spec.template.metadata.labels`: Provides labels that must match the Deployment selector.
- `spec.template.spec.containers`: Defines the containers and their configuration.

The selector and Pod-template labels must be compatible. For an `apps/v1` Deployment, the selector is immutable after creation.

For example:

```yaml
selector:
  matchLabels:
    app: payments
```

must match labels in the Pod template:

```yaml
template:
  metadata:
    labels:
      app: payments
```

If the labels do not satisfy the selector, the Deployment manifest will be rejected.

### 5. Scaling a Deployment

Scaling changes the desired number of replicas.

Suppose a Deployment currently maintains three replicas:

```yaml
replicas: 3
```

You change the desired count to five:

```bash
kubectl scale deployment web-app --replicas=5
```

The Deployment's desired replica count becomes five, and the controllers work toward maintaining that count.

To scale back down:

```bash
kubectl scale deployment web-app --replicas=2
```

Scaling does not guarantee that all requested replicas immediately become ready. Available node resources, image availability, storage, scheduling constraints, and application startup can affect the result.

If you use a declarative manifest to manage the Deployment, update the manifest as well. Otherwise, a later application of the old manifest may restore the previous replica count.

### 6. Rolling updates

A Deployment can update its application version without deliberately stopping all existing replicas at once.

For example, the initial Pod template might use:

```yaml
image: nginx:1.28
```

You update it to:

```yaml
image: nginx:1.29
```

The changed Pod template causes the Deployment to create a new ReplicaSet and perform a rollout.

During a rolling update, Kubernetes gradually replaces old Pods with new ones according to the Deployment's update strategy.

The default strategy is `RollingUpdate`.

Two useful parameters are:

- `maxSurge`: The maximum number of Pods that may temporarily exist above the desired replica count during the rollout.
- `maxUnavailable`: The maximum number of Pods that may be unavailable during the rollout.

For example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

With three desired replicas, this configuration allows up to one additional Pod above the desired count and permits no unavailable replicas under the Deployment's rollout accounting.

However, the application must be configured correctly for this strategy to provide reliable availability. Readiness probes, resource capacity, application startup behavior, and dependency health all matter.

A Deployment rollout does not automatically guarantee zero downtime.

### 7. Rollout status and history

A Deployment tracks rollout progress and revisions.

Useful commands include:

```bash
kubectl rollout status deployment/web-app
kubectl rollout history deployment/web-app
```

The rollout status command waits for the Deployment's rollout to complete or fail to make progress within the applicable timeout.

The history command displays recorded revisions when available.

A new revision is generally created when the Pod template changes. Changing only fields such as the replica count does not normally create a new Deployment revision.

This distinction matters when diagnosing whether a change actually replaced the application's Pods.

### 8. Rollbacks

If an update introduces a problem, you can roll back to a previous Deployment revision when that revision is available.

```bash
kubectl rollout undo deployment/web-app
```

This restores the previous Pod template recorded in the Deployment's rollout history.

To inspect revision history:

```bash
kubectl rollout history deployment/web-app
```

To roll back to a particular revision:

```bash
kubectl rollout undo deployment/web-app --to-revision=2
```

Rollback restores the Deployment's previous Pod template, not every external dependency or change made outside the Deployment.

For example, it does not automatically reverse database schema changes, restore a deleted PersistentVolume, or undo changes to a separate ConfigMap unless those changes are managed separately.

### 9. Deployment availability and readiness

A Deployment can exist successfully while its application is not available.

Consider a Deployment with three desired replicas:

- Three Pods exist, but none is ready: the desired replica count exists, but the application is not available.
- Two Pods are ready and one is starting: the Deployment may be progressing toward its target.
- All three Pods are ready and available: the Deployment has achieved its desired availability, subject to its configured availability requirements.

Readiness probes help determine whether individual containers should receive traffic. Deployment status uses Pod readiness and availability to report rollout progress.

You can inspect the state using:

```bash
kubectl get deployment web-app
kubectl describe deployment web-app
kubectl get pods -l app=web-app
```

A Kubernetes Service can provide a stable endpoint for ready Pods selected by labels. A Deployment alone does not expose an application externally.

### 10. Deployment vs StatefulSet vs DaemonSet

These workload controllers serve different purposes.

| Workload | Best suited for | Typical example |
|---|---|---|
| Deployment | Stateless applications where replicas are interchangeable | Web APIs, frontend applications, stateless services |
| StatefulSet | Applications requiring stable Pod identities and ordered management, often with persistent storage | Distributed databases, stateful clusters |
| DaemonSet | Running a Pod on each eligible node, or a selected subset of nodes | Node monitoring agents, log collectors, networking agents |

A Deployment is usually the right starting point for stateless services. It is not the appropriate default for every workload.

## Key points

- A Deployment declaratively manages replicated application Pods and supports controlled rollouts and rollbacks.
- The resource hierarchy is `Deployment → ReplicaSet → Pods → Containers`.
- The Deployment controller manages ReplicaSets; ReplicaSet controllers maintain the desired number of Pods.
- `spec.replicas` declares the desired replica count.
- `spec.selector` identifies the Pods managed by the Deployment and must match the Pod-template labels.
- The `spec.template` defines the Pod specification used to create replicas.
- Updating the Pod template generally creates a new ReplicaSet and triggers a rollout.
- Scaling changes the desired number of replicas.
- Rolling updates gradually replace old Pods with new ones according to the update strategy.
- `maxSurge` controls temporary capacity above the desired replica count during a rollout.
- `maxUnavailable` controls how many replicas may be unavailable during a rollout.
- Readiness probes and sufficient capacity are important for reliable application updates.
- `kubectl rollout status` monitors rollout progress.
- `kubectl rollout history` displays available revision history.
- `kubectl rollout undo` rolls back the Pod template to a previous revision.
- Scaling alone does not normally create a new Deployment revision.
- A Deployment does not automatically create a Service or expose the application.
- A Deployment does not guarantee that every Pod is ready merely because the resource exists.
- Deleting a Deployment-managed Pod normally results in a replacement because the controllers maintain the desired replica count.
- A standalone Pod does not have this replacement behavior unless another controller manages it.
- Deployment rollback does not reverse unrelated database, storage, or external-system changes.
- Deployments are generally preferred for stateless workloads; StatefulSets and DaemonSets address different requirements.

## Commands / manifests

### 1. Create a Deployment manifest

Create a file named `web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-app
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

This manifest defines:

- A Deployment named `web-app`.
- Three desired replicas.
- A selector that matches Pods labelled `app: web-app`.
- A rolling-update strategy.
- A container image and resource requests and limits.

The example image tag is illustrative. Use a valid image tag available in your environment.

### 2. Apply the Deployment

```bash
kubectl apply -f web-deployment.yaml
```

Inspect the Deployment and its resources:

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -l app=web-app
```

Wait for rollout completion:

```bash
kubectl rollout status deployment/web-app
```

### 3. Inspect Deployment details

```bash
kubectl describe deployment web-app
```

This displays the replica counts, selector, update strategy, conditions, and events.

To see the Pod-template image:

```bash
kubectl get deployment web-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

To inspect rollout status:

```bash
kubectl get deployment web-app \
  -o jsonpath='{.status.replicas}{" replicas, "}{.status.readyReplicas}{" ready, "}{.status.availableReplicas}{" available\n"}'
```

If a status field is not yet populated, its output may be empty.

### 4. Scale the Deployment

Increase replicas to five:

```bash
kubectl scale deployment/web-app --replicas=5
```

Verify the result:

```bash
kubectl get deployment web-app
kubectl get pods -l app=web-app
```

Scale down to two:

```bash
kubectl scale deployment/web-app --replicas=2
```

For declarative management, also update `replicas` in `web-deployment.yaml` to the intended value.

### 5. Update the container image

Update the image using a command:

```bash
kubectl set image deployment/web-app web=nginx:1.29
```

The container name is `web`, as specified in the Pod template.

Monitor the rollout:

```bash
kubectl rollout status deployment/web-app
kubectl get replicasets
kubectl get pods -l app=web-app
```

For a declarative workflow, update the image in the manifest and apply it:

```bash
kubectl apply -f web-deployment.yaml
```

Remember that repeatedly applying a manifest with the old image can revert a command-line image update.

### 6. View rollout history

```bash
kubectl rollout history deployment/web-app
```

Inspect a specific revision:

```bash
kubectl rollout history deployment/web-app --revision=2
```

The requested revision must exist in the Deployment's recorded history.

### 7. Roll back an update

Return to the previous revision:

```bash
kubectl rollout undo deployment/web-app
```

Monitor the rollback:

```bash
kubectl rollout status deployment/web-app
kubectl get deployment web-app
kubectl get pods -l app=web-app
```

To roll back to a particular revision:

```bash
kubectl rollout undo deployment/web-app --to-revision=2
```

For declarative workflows, reconcile your manifest and version-control repository with the intended state after a rollback. Otherwise, a subsequent apply may restore the newer configuration.

### 8. Pause and resume a rollout

A Deployment rollout can be paused while you make related changes:

```bash
kubectl rollout pause deployment/web-app
```

Resume it:

```bash
kubectl rollout resume deployment/web-app
```

Pausing is useful in certain multi-step update workflows, but it should not replace a properly planned deployment process.

### 9. Check Deployment availability

```bash
kubectl get deployment web-app
kubectl describe deployment web-app
kubectl get pods -l app=web-app -o wide
```

Check recent events:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for image-pull errors, scheduling failures, failed probes, and other conditions preventing the rollout from completing.

### 10. Expose the application through a Service

A Deployment manages Pods, not a stable network endpoint. Create a Service to access the application inside the cluster:

```bash
kubectl expose deployment web-app \
  --name=web-app-service \
  --port=80 \
  --target-port=80
```

Inspect the Service and its backends:

```bash
kubectl get service web-app-service
kubectl get endpointslices \
  -l kubernetes.io/service-name=web-app-service
```

For a local test:

```bash
kubectl port-forward service/web-app-service 8080:80
```

In another terminal:

```bash
curl http://localhost:8080
```

Port forwarding is useful for development and debugging. It does not by itself expose the application publicly.

### 11. Clean up the practice resources

```bash
kubectl delete service web-app-service
kubectl delete deployment web-app
```

Or delete the Deployment from its manifest:

```bash
kubectl delete -f web-deployment.yaml
```

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Deployment has fewer Pods than requested | Scheduling constraints, resource shortage, or Pod creation errors | Inspect `kubectl describe deployment web-app` and Pod events |
| Deployment has the expected number of Pods, but some are not ready | Application startup failures, readiness probes, or dependency problems | Check Pod status, logs, and readiness configuration |
| `ImagePullBackOff` during rollout | Invalid image, inaccessible registry, or authentication issue | Verify the image tag and registry access |
| `FailedScheduling` | Insufficient node resources, affinity constraints, taints, or selectors | Inspect Pod events and node capacity |
| Deployment rollout does not complete | New Pods cannot become available or old Pods cannot be replaced under the strategy | Run `kubectl rollout status` and inspect the new ReplicaSet and Pods |
| `ProgressDeadlineExceeded` | The Deployment did not make sufficient rollout progress within its configured progress deadline | Inspect events, readiness, startup, scheduling, and image-pull errors |
| Manifest rejected due to selector errors | Selector and Pod-template labels do not match | Correct the labels; recreate the Deployment if an immutable selector must change |
| Image update does not create a new ReplicaSet | The Pod template may not have changed, or the image update targeted the wrong container | Check the Deployment template and container names |
| Rollback fails or has no expected effect | Revision is unavailable, or the reverted Pod template still has a problem | Inspect rollout history and the target revision |
| Deployment reverts to an older image after `kubectl apply` | The manifest still declares the older image | Update the manifest and apply the intended configuration |
| Pods repeatedly restart | Application failure, incorrect configuration, or liveness probe failures | Inspect current and previous container logs and Pod events |
| Service has no endpoints | Service selector mismatch or no ready matching Pods | Check Service selectors, Pod labels, readiness, and EndpointSlices |
| Scaling command succeeds but fewer replicas are ready | Desired replica count changed, but Pods cannot become ready | Check scheduling, image availability, application health, and resource capacity |
| Rollback restores the application image but the app still fails | External state or dependencies were not rolled back | Inspect database compatibility, configuration, storage, and downstream services |

### Troubleshooting workflow

Start with the Deployment:

```bash
kubectl get deployment web-app
kubectl describe deployment web-app
kubectl rollout status deployment/web-app
```

Inspect the ReplicaSets:

```bash
kubectl get replicasets -l app=web-app
```

Inspect the Pods:

```bash
kubectl get pods -l app=web-app -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

Use this sequence to narrow down the problem:

1. **Desired vs actual replicas:** Check whether Kubernetes has created the expected number of Pods.
2. **Scheduling:** Look for `FailedScheduling` events.
3. **Image retrieval:** Check for `ErrImagePull` or `ImagePullBackOff`.
4. **Container startup:** Inspect container states, exit codes, and logs.
5. **Readiness:** Check whether the application passes its readiness probe.
6. **Rollout strategy:** Check whether `maxUnavailable`, `maxSurge`, or available node capacity is preventing progress.
7. **Service routing:** Verify that the Service selects the correct Pods and has usable endpoints.

Avoid deleting the Deployment as the first troubleshooting step. Doing so removes the controller responsible for maintaining the application. Inspect the resource and its events before making changes.

## References

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [kubectl scale](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/)
- [kubectl rollout](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
- [Performing a Rolling Update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
- [Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
- [Deployment Rollbacks](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Debugging Applications](https://kubernetes.io/docs/tasks/debug/)
