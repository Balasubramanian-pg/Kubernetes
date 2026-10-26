# Pods

> Phase 2 — Kubernetes Core (highest priority) › Workload primitives

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **Pod is the smallest deployable unit in Kubernetes**. It represents one or more containers that are scheduled together onto the same node and share certain resources, such as networking and optionally storage.

Kubernetes does not normally manage individual application containers directly. Instead, it manages Pods, which provide the execution environment for those containers.

### 1. Why Pods exist

Containers package applications and their dependencies, but real workloads sometimes need multiple tightly coupled containers to cooperate.

A Pod provides a shared environment for these containers:

- They run on the same node.
- They share the same network namespace and IP address.
- They can communicate using `localhost`.
- They can share storage through mounted volumes.
- They are scheduled as one unit.

For example, a Pod could contain an application container and a sidecar container that collects logs or proxies network traffic.

However, **one application container per Pod is the normal starting point**. Multiple containers belong in the same Pod when they need to be tightly coupled and share the same lifecycle or resources.

### 2. Pod vs container

A container is the running instance of an application image. A Pod is the Kubernetes object that groups containers and provides the environment in which they run.

| Aspect | Container | Pod |
|---|---|---|
| Purpose | Runs an application process | Provides a deployable environment for one or more containers |
| Scheduling | Not independently scheduled by Kubernetes | Scheduled as a unit onto a node |
| Network identity | Uses the networking environment provided by its container runtime | Containers share the Pod's network namespace and IP |
| Storage | Can use mounted filesystems and volumes | Can expose shared volumes to multiple containers |
| Lifecycle | Managed by the container runtime and kubelet | Managed through Kubernetes Pod lifecycle and controllers |
| Scaling | Container count is not the standard Kubernetes scaling unit | Workloads typically scale by creating more Pod replicas |

A Pod is not a virtual machine. It shares the node's operating system kernel through its containers, while Kubernetes manages the Pod's placement and lifecycle.

### 3. Anatomy of a Pod

A typical Pod specification contains:

- `apiVersion`: The Kubernetes API version for the resource.
- `kind`: The resource type, which is `Pod`.
- `metadata`: The Pod's name, namespace, labels, and annotations.
- `spec`: The desired configuration, including containers, images, ports, resource requests, and volumes.
- `status`: Runtime information reported by Kubernetes, such as Pod phase, assigned node, and container states.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
spec:
  containers:
    - name: web
      image: nginx:1.28
      ports:
        - containerPort: 80
```

Here:

- `web-pod` is the Pod's name.
- `app: web` is a label that other Kubernetes resources can use to select the Pod.
- `nginx:1.28` identifies the container image.
- `containerPort: 80` documents the port the container is expected to use. It does not, by itself, publish the port outside the Pod.

The example image tag is illustrative. If it is unavailable in your environment, select a valid tag from the official NGINX image repository.

### 4. How a Pod gets scheduled and runs

When you create a Pod, Kubernetes coordinates several components to get it running.

1. You submit a Pod manifest using `kubectl apply`.
2. The API server validates and processes the request.
3. The accepted Pod object is stored in the cluster's backing store.
4. The scheduler selects a suitable node if the Pod has not already been assigned one.
5. The kubelet on that node observes the Pod assignment.
6. The kubelet coordinates with the container runtime to prepare the Pod and start its containers.
7. Kubernetes reports container and Pod status.
8. The kubelet continues monitoring the containers and executes configured probes.

If the container exits, the kubelet's behavior depends on the Pod's restart policy. For ordinary application Pods, the default is `Always`, so the kubelet attempts to restart failed containers.

The Pod itself is not automatically recreated with a new identity simply because a container restarts. Container restarts and Pod replacement are different events.

### 5. Pod lifecycle and phases

A Pod's `status.phase` provides a high-level summary of its lifecycle.

| Phase | Meaning |
|---|---|
| `Pending` | The Pod has been accepted, but its containers are not all running. Scheduling or setup may still be in progress. |
| `Running` | The Pod has been bound to a node, and its containers have been created. At least one container is running, starting, or restarting. |
| `Succeeded` | All containers have terminated successfully and will not be restarted. |
| `Failed` | All containers have terminated, and at least one terminated unsuccessfully. |
| `Unknown` | Kubernetes cannot determine the Pod's state, commonly because node communication has failed. |

**Important:** `CrashLoopBackOff` and `ImagePullBackOff` are not Pod phases. They are container waiting-state reasons that you commonly see in `kubectl` output.

A Pod in `Running` phase is not necessarily ready to receive traffic. Readiness probes and container readiness status help determine whether the application should receive traffic through a Service.

### 6. Pod IP and networking

Every ordinary Pod receives its own IP address within the cluster network.

Containers inside the same Pod share the network namespace, so they share:

- The Pod IP address.
- The network interfaces.
- The port space.
- The loopback interface.

This means containers within the same Pod can communicate through `localhost`.

For example, if one container listens on port `8080`, another container in the same Pod can reach it through `localhost:8080`.

However, two containers in the same Pod cannot independently bind to the same IP address and port combination. If both attempt to listen on `0.0.0.0:8080`, they can conflict.

Different Pods generally have different IP addresses. Containers in different Pods communicate using the Pod network, subject to network policies and the cluster's networking implementation.

A Pod IP is not normally a durable application endpoint because the Pod may be replaced and receive a different IP. A Kubernetes Service provides a more stable way to access a changing group of Pods.

### 7. Labels and selectors

Labels are key-value pairs attached to Kubernetes objects. Selectors allow other resources to identify groups of Pods.

Example Pod labels:

```yaml
metadata:
  labels:
    app: payments
    environment: development
```

A Service can select Pods using labels:

```yaml
selector:
  app: payments
```

The Service can then route traffic to eligible Pods matching that selector.

Labels are also used by Deployments and ReplicaSets to identify the Pods they manage.

**Important:** A label does not automatically expose a Pod or create networking rules. It is metadata that Kubernetes resources can use for selection and organization.

### 8. Pod storage

A Pod can mount volumes into one or more containers.

Common volume types include:

- `emptyDir`: Temporary storage created for the Pod and shared by containers that mount it. Its contents survive individual container restarts but are lost when the Pod is removed from the node.
- `persistentVolumeClaim`: Requests storage through Kubernetes persistent storage resources.
- `configMap`: Exposes non-sensitive configuration.
- `secret`: Exposes sensitive configuration data, subject to the cluster's security configuration.

Containers can mount the same volume at different paths. This is useful when one container produces data and another consumes it.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage-pod
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "date > /shared/time.txt; sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /shared
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /shared
  volumes:
    - name: shared-data
      emptyDir: {}
```

Both containers can access `/shared/time.txt` because they mount the same `emptyDir` volume.

The example is for learning. In production, container commands, image versions, permissions, and resource settings should be selected deliberately.

### 9. Why Pods are usually managed by controllers

You can create a Pod directly, but standalone Pods are not the best way to run most long-lived applications.

If a standalone Pod is deleted, Kubernetes does not automatically create a replacement just because that Pod existed.

A Deployment provides higher-level management:

- Maintains the desired number of replicas through ReplicaSets.
- Replaces Pods when necessary.
- Supports rolling updates and rollbacks.
- Allows scaling the number of replicas.

The relationship is:

`Deployment → ReplicaSet → Pods → Containers`

For a long-running application, create a Deployment rather than manually creating multiple standalone Pods.

Directly created Pods are still useful for debugging, learning, and short-lived troubleshooting tasks.

### 10. Pod readiness, liveness, and startup probes

Probes help Kubernetes understand different aspects of an application's health.

- **Readiness probe:** Determines whether a container is ready to receive traffic. A failed readiness probe removes the Pod from the set of ready Service backends, where applicable.
- **Liveness probe:** Determines whether a container should be restarted because it is considered unhealthy.
- **Startup probe:** Gives a slow-starting application additional time to initialize before liveness and readiness checks begin.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probed-web-pod
spec:
  containers:
    - name: web
      image: nginx:1.28
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
      livenessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 10
```

In this example, the startup probe can allow roughly five minutes for startup failures to be tolerated before the startup check is considered unsuccessful, based on the configured period and failure threshold.

Once startup succeeds, Kubernetes begins the liveness and readiness checks.

Do not use identical probe logic automatically for every application. A liveness check that fails during a temporary dependency outage can trigger unnecessary restarts, while a readiness check should reflect whether the application can actually handle requests.

## Key points

- A Pod is the smallest deployable unit in Kubernetes.
- A Pod contains one or more containers that share networking and can share mounted volumes.
- Containers within one Pod share the same Pod IP and port space.
- Containers in different Pods generally communicate over the cluster network.
- The scheduler assigns a Pod to a node; the kubelet on that node coordinates its execution.
- The container runtime actually runs the containers.
- The default Pod restart policy is `Always`, but this can be changed for appropriate workload types.
- A container restart does not necessarily mean the Pod has been replaced.
- Deleting a standalone Pod does not cause Kubernetes to recreate it automatically.
- Deployments manage ReplicaSets, which maintain the desired number of Pod replicas.
- Labels and selectors connect Pods to resources such as Services and ReplicaSets.
- `containerPort` documents a container's intended port; it does not expose the Pod externally.
- Pod IPs are not durable application endpoints. Services provide stable access to selected backends.
- `Running` does not mean an application is ready to receive traffic.
- `CrashLoopBackOff` and `ImagePullBackOff` are container states or reasons, not Pod phases.
- `emptyDir` survives container restarts within the same Pod but is removed when the Pod is removed.
- Readiness, liveness, and startup probes serve different purposes.
- Pod resource requests affect scheduling; resource limits constrain resource consumption.
- A Pod is bound to one node and is not itself a mechanism for distributing its containers across multiple nodes.

## Commands / manifests

### 1. Create a simple Pod

Save this manifest as `nginx-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.28
      ports:
        - containerPort: 80
```

Create the Pod:

```bash
kubectl apply -f nginx-pod.yaml
```

Check its status:

```bash
kubectl get pods
kubectl get pod nginx-pod -o wide
kubectl describe pod nginx-pod
```

`kubectl get pod -o wide` includes the Pod IP and assigned node, where available.

### 2. Inspect Pod details

```bash
kubectl get pod nginx-pod -o yaml
```

Displays the Pod resource, including its specification and status.

To see its labels:

```bash
kubectl get pod nginx-pod --show-labels
```

To list only Pods matching a label:

```bash
kubectl get pods -l app=nginx
```

### 3. Read container logs

```bash
kubectl logs nginx-pod
```

For a Pod with multiple containers:

```bash
kubectl logs <pod-name> -c <container-name>
```

To inspect logs from the previous instance of a container after it has restarted:

```bash
kubectl logs nginx-pod --previous
```

The `--previous` option is useful for diagnosing a crash that occurred before the current container instance started.

### 4. Execute a command inside a container

```bash
kubectl exec -it nginx-pod -- sh
```

This opens a shell inside the container if the image includes `sh`.

For a single command without opening an interactive shell:

```bash
kubectl exec nginx-pod -- nginx -v
```

For multi-container Pods, specify the target container:

```bash
kubectl exec -it <pod-name> -c <container-name> -- sh
```

### 5. Test Pod networking

Check the Pod IP:

```bash
kubectl get pod nginx-pod -o wide
```

Forward a local port to the Pod:

```bash
kubectl port-forward pod/nginx-pod 8080:80
```

In another terminal:

```bash
curl http://localhost:8080
```

This tests access through port forwarding. It does not make the Pod publicly accessible.

### 6. Create a Pod with resource requests and limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: nginx:1.28
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 500m
          memory: 128Mi
```

Save it as `resource-demo.yaml` and apply it:

```bash
kubectl apply -f resource-demo.yaml
kubectl describe pod resource-demo
```

Interpretation:

- `100m` CPU means one-tenth of a CPU core.
- `64Mi` memory means 64 mebibytes.
- Requests help the scheduler determine whether a node can accommodate the Pod.
- Limits constrain resource consumption. CPU limits generally throttle CPU use, while exceeding a memory limit can result in an out-of-memory termination.

These values are examples, not universal recommendations.

### 7. Observe Pod lifecycle

```bash
kubectl get pods -w
```

Watch Pod status changes in real time.

To inspect recent events:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

To view the phase directly:

```bash
kubectl get pod nginx-pod -o jsonpath='{.status.phase}'
```

### 8. Compare standalone Pods and Deployment-managed Pods

Create a Deployment:

```bash
kubectl create deployment nginx-deployment --image=nginx:1.28
```

Scale it to three replicas:

```bash
kubectl scale deployment nginx-deployment --replicas=3
```

Inspect the resulting resources:

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -l app=nginx-deployment
```

Delete one of the Deployment-managed Pods:

```bash
kubectl delete pod <deployment-pod-name>
```

The Deployment's controllers work toward restoring the desired replica count.

By contrast, deleting `nginx-pod`, the standalone Pod created earlier, does not cause a replacement to be created.

### 9. Clean up

```bash
kubectl delete pod nginx-pod
kubectl delete pod resource-demo
kubectl delete deployment nginx-deployment
```

Alternatively, remove resources created from a manifest:

```bash
kubectl delete -f nginx-pod.yaml
kubectl delete -f resource-demo.yaml
```

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Pod remains `Pending` | Insufficient resources, scheduling constraints, or storage setup delay | Run `kubectl describe pod <pod-name>` and inspect Events |
| `ImagePullBackOff` | Invalid image name or tag, inaccessible registry, or authentication issue | Check the image reference and Pod events |
| `ErrImagePull` | Initial image pull failed | Inspect the specific registry or image error in `kubectl describe pod` |
| `CrashLoopBackOff` | Container repeatedly exits or fails its health checks | Inspect current and previous logs, commands, environment variables, and probes |
| Pod is `Running` but not `Ready` | Readiness probe failure or application not ready | Check readiness conditions, application logs, and probe configuration |
| `CreateContainerConfigError` | Missing ConfigMap, Secret, or invalid container configuration | Inspect Pod events and referenced configuration resources |
| `OOMKilled` | Container exceeded its effective memory limit or was terminated due to memory pressure | Review memory usage, limits, and application behavior |
| `FailedScheduling` | No suitable node satisfies resource or placement requirements | Check CPU and memory requests, node selectors, affinity, taints, and tolerations |
| Pod cannot reach another Pod | Network policy, application listener, routing, or DNS issue | Check connectivity, Pod IPs, listening ports, and network policies |
| `kubectl exec` fails | Container is not running, shell is absent, or the target container is wrong | Check Pod status and container name; use a command available in the image |
| Pod cannot be reached through a Service | Selector mismatch or no ready endpoints | Inspect Service selectors, Pod labels, readiness, and EndpointSlices |
| Pod is repeatedly restarted | Application exits, liveness probe fails, or node/runtime issues occur | Run `kubectl describe pod` and `kubectl logs --previous` |
| Deleting a standalone Pod leaves no replacement | No controller manages that Pod | Create a Deployment or another suitable workload controller |
| Pod gets a different IP after replacement | A replacement Pod receives its own network identity | Access the workload through a Service instead of depending on a Pod IP |

### Troubleshooting sequence

Use this sequence when a Pod does not behave as expected:

```bash
kubectl get pod <pod-name> -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl get events --sort-by=.metadata.creationTimestamp
```

For Pods with multiple containers, add `-c <container-name>` to the log or exec command where required.

Read the output in this order:

1. **Phase and readiness:** Determine whether the Pod is pending, running, or failing.
2. **Events:** Look for scheduling, image-pull, volume-mount, and probe errors.
3. **Logs:** Identify application-level failures.
4. **Container state:** Check restart counts, exit codes, and termination reasons.
5. **Configuration:** Verify image, command, environment variables, resources, probes, and volume mounts.
6. **Networking:** Check whether the application is listening on the expected interface and port.

Do not assume that every `Pending` Pod has a scheduling problem or that every `Running` Pod is healthy. Use the events and container status to identify the actual failure.

## References

- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Pod Networking](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Configure a Pod to Use a Volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/)
- [Container Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Debugging Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
