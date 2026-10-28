# kubectl get pods -w

> Phase 2 — Kubernetes Core (highest priority) › kubectl fluency

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

`kubectl get pods -w` watches Pods for changes and continuously updates the command output as Kubernetes reports new events in their state.

The `-w` flag is shorthand for `--watch`.

Normally, `kubectl get pods` prints a snapshot of the current Pod state and exits. Adding `-w` keeps the command running so you can observe changes as they happen.

For example, after creating a Deployment, its Pods may move through different states while Kubernetes schedules them, pulls their container images, and starts the containers.

A typical progression might look like:

```text
NAME                    READY   STATUS              RESTARTS   AGE
web-7c9d8f6b5-abc12      0/1     Pending             0          0s
web-7c9d8f6b5-abc12      0/1     ContainerCreating   0          2s
web-7c9d8f6b5-abc12      1/1     Running             0          5s
```

This is an illustrative sequence. Actual output and intermediate states depend on the cluster, workload, and kubectl version.

Watching Pods is useful during deployments, scaling operations, troubleshooting, and testing configuration changes. It lets you observe changes without repeatedly running `kubectl get pods`.

**How it works:**

1. Run `kubectl get pods -w`.
2. Kubernetes returns the current Pod information and establishes a watch for subsequent changes.
3. When relevant updates arrive, kubectl displays the changed resource information.
4. Continue observing until the workload reaches the expected state or a problem becomes apparent.
5. Press `Ctrl+C` to stop watching.

A watch reports changes to Kubernetes resource objects. It does not stream application logs, and a Pod's `STATUS` column is not a complete health assessment.

## Key points

- **`-w` means watch:** It keeps kubectl running to display changes instead of returning only a one-time snapshot.
- **Initial listing:** The command normally displays the existing matching Pods before continuing to watch for updates.
- **Real-time troubleshooting:** It is useful for observing Pod creation, scheduling, startup, termination, and replacement.
- **Pod lifecycle:** A Pod's phase can be `Pending`, `Running`, `Succeeded`, `Failed`, or `Unknown`. Values shown in the `STATUS` column can include more descriptive states, such as `ContainerCreating` or `CrashLoopBackOff`.
- **`Running` does not mean healthy:** A Pod can be in the `Running` phase while its application is failing readiness checks or repeatedly restarting.
- **Readiness matters:** Check the `READY` column to see how many containers in a Pod are ready, but also inspect Deployment readiness and application health when needed.
- **Restarts matter:** The `RESTARTS` column helps identify containers that have restarted. Increasing counts may indicate crashes, probe failures, or other problems.
- **Watch one namespace:** `kubectl get pods -w` watches Pods in the current namespace.
- **Watch all namespaces:** `kubectl get pods -A -w` watches Pods across namespaces, subject to your permissions.
- **Watch a specific workload:** Use a label selector, such as `-l app=web`, to limit output to matching Pods.
- **Watch a single Pod:** Use `kubectl get pod <pod-name> -w` to monitor one Pod.
- **Watch Deployment changes:** Use `kubectl get deployments -w` to observe changes to Deployment replica counts and readiness.
- **Watch events separately:** `kubectl get events -w` watches event resources. It does not replace watching Pods because events and Pod object updates represent different information.
- **Watch does not stream logs:** Use `kubectl logs -f <pod-name>` to follow container logs.
- **Watch is not an application health monitor:** It shows API resource updates, not every internal application event or metric.
- **Output may vary:** Updates, columns, and intermediate statuses depend on Kubernetes behavior, kubectl version, and how quickly changes occur.
- **Use `Ctrl+C` to stop:** This stops the local watch command. It does not stop or delete the Pods.
- **Permissions are required:** Your Kubernetes identity must have permission to list and watch the relevant resources.
- **Useful deployment workflow:** Start watching Pods, apply a manifest, observe the rollout, and verify the final Deployment status.
- **Pods can be replaced:** A Deployment may terminate an old Pod and create a new Pod with a different name during a rollout. Watch the label-selected set rather than relying on one Pod name.
- **Watch connections can be interrupted:** Network problems or API server interruptions can terminate the watch. Rerun the command to establish a new watch.
- **Resource usage:** Watching a large number of resources can generate substantial output. Narrow the namespace or use label selectors when possible.

## Commands / manifests

### 1. Watch Pods in the current namespace

```bash
kubectl get pods -w
```

This displays the current Pods and continues showing updates.

Useful when you have just applied a manifest and want to observe the Pods being created.

Stop watching with `Ctrl+C`.

### 2. Watch Pods across all namespaces

```bash
kubectl get pods -A -w
```

This displays Pods from all namespaces and watches for changes.

Use it when troubleshooting a cluster-wide issue or when you do not know which namespace contains the workload.

You need the appropriate permissions to list and watch Pods across namespaces.

### 3. Watch Pods in a specific namespace

```bash
kubectl get pods -n development -w
```

This monitors Pods in the `development` namespace.

It is particularly useful when separating development, staging, and production workloads.

### 4. Watch a single Pod

```bash
kubectl get pod my-pod -w
```

This monitors the named Pod.

If the Pod is deleted and replaced by a controller, the replacement may have a different name. The command targeting the original Pod does not automatically become a watch of every replacement Pod.

### 5. Watch Pods using a label selector

```bash
kubectl get pods -l app=web -w
```

This monitors Pods matching the label selector `app=web`.

Label selectors are useful for tracking a workload whose Pod names change during deployments or scaling.

To check which Pods match before starting the watch:

```bash
kubectl get pods -l app=web
```

For a particular namespace:

```bash
kubectl get pods -n development -l app=web -w
```

### 6. Watch Deployment status

```bash
kubectl get deployments -w
```

This shows changes to Deployment information, including desired replicas, available replicas, and readiness.

For a particular Deployment:

```bash
kubectl get deployment web -w
```

To check whether its rollout has completed:

```bash
kubectl rollout status deployment/web
```

The watch helps you observe changes. The rollout status command helps determine whether the Deployment has completed its rollout successfully.

### 7. Watch Pods while applying a manifest

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
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

Start watching the matching Pods:

```bash
kubectl get pods -l app=web -w
```

In another terminal, apply the Deployment:

```bash
kubectl apply -f deployment.yaml
```

Observe the Pods as Kubernetes creates and starts them.

Once the Pods are ready, check the Deployment:

```bash
kubectl get deployment web
kubectl rollout status deployment/web
```

Using two terminals makes it easier to watch the output while running other commands.

### 8. Watch Pods during scaling

With the Deployment from the previous example, change the replica count to three or run:

```bash
kubectl scale deployment web --replicas=3
```

Watch the Pods:

```bash
kubectl get pods -l app=web -w
```

You should see Kubernetes create an additional Pod and eventually reach the desired replica count, assuming the cluster can schedule and start it.

Verify the result:

```bash
kubectl get deployment web
```

The Deployment should report three desired replicas and, once the workload is healthy, three available replicas.

### 9. Watch Pods during an image update

Update the image through the Deployment:

```bash
kubectl set image deployment/web nginx=nginx:1.29
```

Then monitor the Pods:

```bash
kubectl get pods -l app=web -w
```

Check the rollout:

```bash
kubectl rollout status deployment/web
kubectl get deployment web
```

A Deployment typically creates new Pods and gradually terminates old ones according to its rollout strategy. The exact sequence depends on the strategy, replica count, readiness, and available resources.

### 10. Watch events alongside Pod changes

In one terminal:

```bash
kubectl get pods -w
```

In another terminal:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp -w
```

The first command shows Pod resource changes. The second watches event resources.

This combination can help connect a Pod's changing state with scheduling failures, image-pull errors, volume problems, or probe failures.

Events are retained for a limited period, and their ordering is not guaranteed to be a perfect record of every operation.

### 11. Follow container logs

If a Pod is running but its application is failing, watch its logs:

```bash
kubectl logs -f <pod-name>
```

For a Pod with multiple containers:

```bash
kubectl logs -f <pod-name> -c <container-name>
```

To follow logs from the previous container instance when available:

```bash
kubectl logs <pod-name> -c <container-name> --previous
```

Use `kubectl get pods -w` for Kubernetes resource state changes and `kubectl logs -f` for application output.

### 12. Hands-on exercise

Perform the following exercise in a development environment:

1. Apply the `deployment.yaml` manifest.
2. Start `kubectl get pods -l app=web -w`.
3. Observe Pod creation and startup.
4. Scale the Deployment from two replicas to three.
5. Observe the new Pod appearing.
6. Update the container image.
7. Observe the rollout and replacement Pods.
8. Run `kubectl rollout status deployment/web` to verify completion.
9. Stop the watch using `Ctrl+C`.
10. Delete the test Deployment when finished.

```bash
kubectl apply -f deployment.yaml
kubectl get pods -l app=web -w
```

In a second terminal:

```bash
kubectl scale deployment web --replicas=3
kubectl rollout status deployment/web
```

After testing:

```bash
kubectl delete -f deployment.yaml
```

The `Ctrl+C` action stops only the watch process. The final delete command removes the Deployment and allows Kubernetes to clean up its managed Pods.

## Common failures & fixes

- **The watch shows no Pods**
  - Cause: No matching Pods exist in the current namespace, or the selector does not match any Pod labels.
  - Fix: Run `kubectl get pods -A` and verify the namespace. If using a selector, check it with `kubectl get pods -l app=web`.

- **The command exits instead of watching**
  - Cause: The `-w` flag was omitted, or the watch was interrupted by a connection or API error.
  - Fix: Run `kubectl get pods -w` again and inspect any error messages.

- **A Pod stays in `Pending`**
  - Cause: The scheduler cannot find a suitable node, or required scheduling or volume operations have not completed.
  - Fix: Run `kubectl describe pod <pod-name>` and inspect events for `FailedScheduling`, resource shortages, taints, affinity rules, or volume problems.

- **A Pod stays in `ContainerCreating`**
  - Cause: The container runtime is preparing the container, pulling an image, or mounting volumes.
  - Fix: Run `kubectl describe pod <pod-name>` and inspect recent events for image-pull, mount, or runtime errors.

- **A Pod shows `ImagePullBackOff`**
  - Cause: The image cannot be pulled, often because the image name or tag is incorrect or registry access is unavailable.
  - Fix: Check `kubectl describe pod <pod-name>`, verify the image reference, and confirm registry credentials and connectivity.

- **A Pod repeatedly enters `CrashLoopBackOff`**
  - Cause: The container process repeatedly exits or fails to start successfully.
  - Fix: Inspect `kubectl logs <pod-name> --previous`, run `kubectl describe pod <pod-name>`, and investigate configuration, startup commands, dependencies, and probes.

- **The Pod shows `Running`, but `READY` is `0/1`**
  - Cause: The container may be running but not ready to receive traffic, or its readiness probe may be failing.
  - Fix: Inspect `kubectl describe pod <pod-name>`, check readiness-probe configuration, and review application logs.

- **The Pod disappears during a rollout**
  - Cause: A controller may have terminated an old Pod and created a replacement with a different name.
  - Fix: Watch using a label selector, such as `kubectl get pods -l app=web -w`, and inspect the Deployment's rollout status.

- **The output does not update**
  - Cause: No matching resource changes have occurred, or the watch connection has stalled or been interrupted.
  - Fix: Verify the resource with a fresh `kubectl get pods` command. If necessary, restart the watch and inspect network or API server errors.

- **`Forbidden` or `Unauthorized` appears**
  - Cause: Your identity lacks permission to list or watch Pods, or the credentials are invalid or expired.
  - Fix: Check `kubectl config current-context` and `kubectl auth can-i watch pods -n <namespace>`. Reauthenticate or request appropriate permissions if needed.

- **Watching all namespaces produces too much output**
  - Cause: The cluster contains many Pods and workloads.
  - Fix: Narrow the watch to a namespace or use a label selector.

- **The watch stops after pressing `Ctrl+C`**
  - Cause: `Ctrl+C` terminates the local watch process.
  - Fix: Rerun the command to resume watching. Pressing `Ctrl+C` does not stop or delete the Kubernetes workloads.

- **A Pod looks healthy in the watch but the application fails**
  - Cause: Pod status alone does not confirm application correctness, successful requests, or downstream dependency health.
  - Fix: Check readiness, application logs, Service endpoints, and application-specific health checks.

## References

- [kubectl get command reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [kubectl rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
- [Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
