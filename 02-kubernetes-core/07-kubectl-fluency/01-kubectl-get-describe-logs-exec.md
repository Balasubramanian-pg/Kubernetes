# kubectl get, describe, logs, exec

> Phase 2 — Kubernetes Core (highest priority) › kubectl fluency

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

`kubectl` is the command-line tool used to communicate with the Kubernetes API server. It allows you to inspect resources, investigate failures, read application logs, and execute commands inside running containers.

Four commands form the foundation of day-to-day Kubernetes troubleshooting:

- **`kubectl get`**: Lists resources and displays their current status.
- **`kubectl describe`**: Shows detailed resource information, configuration, conditions, and events.
- **`kubectl logs`**: Reads the output written by a container to its standard output and standard error.
- **`kubectl exec`**: Executes a command inside a container in a running Pod.

These commands complement each other. `get` tells you what is happening, `describe` helps explain why, `logs` reveals what the application reported, and `exec` lets you inspect the container's runtime environment.

### The troubleshooting workflow

```text
Something is wrong with the application
                  |
                  v
          kubectl get
       Check resource status
                  |
                  v
        kubectl describe
    Inspect conditions and events
                  |
                  v
          kubectl logs
     Read application output
                  |
                  v
           kubectl exec
 Inspect container and dependencies
                  |
                  v
       Identify and fix the cause
```

This is a practical starting workflow, not a mandatory sequence. For example, you might inspect logs first if the Pod is already running but returning application errors.

### 1. `kubectl get`: What exists, and what is its status?

`kubectl get` retrieves Kubernetes resources and displays a concise summary.

For example:

```bash
kubectl get pods
```

Possible output:

```text
NAME             READY   STATUS             RESTARTS   AGE
api-7f6c9d8b-x2   1/1     Running            0          12m
worker-5d8c4a-p9  0/1     CrashLoopBackOff   5          8m
```

The output helps you identify the Pod that needs investigation.

Common fields include:
- `READY`: Number of ready containers compared with the total containers in the Pod.
- `STATUS`: The displayed Pod phase or a more specific status such as `CrashLoopBackOff`.
- `RESTARTS`: Number of container restarts reported for the Pod.
- `AGE`: How long the resource has existed.

The `STATUS` column is a useful summary, but it is not always the exact Pod phase. For example, `CrashLoopBackOff` describes a container restart backoff condition, not a distinct Pod phase.

Use `get` when you need a quick overview, when comparing resources, or when monitoring a rollout.

### 2. `kubectl describe`: Why is this resource behaving this way?

`kubectl describe` retrieves detailed information about a resource, including its configuration, current state, conditions, and associated events.

```bash
kubectl describe pod <pod-name>
```

Useful sections often include:
- **Labels and annotations:** Metadata attached to the resource.
- **Containers:** Image, ports, environment variables, resource requests, and limits.
- **State and Last State:** Current or previous container state and termination information.
- **Conditions:** Scheduling, readiness, and other status information.
- **Events:** Recent scheduling, image-pulling, mounting, probe, and container lifecycle events.

Example event:

```text
Warning  FailedScheduling  default-scheduler
0/3 nodes are available: insufficient memory.
```

This points toward a scheduling problem rather than an application exception.

Use `describe` when a Pod is stuck in `Pending`, a container repeatedly restarts, an image cannot be pulled, a volume fails to mount, or a readiness probe fails.

Events are not a permanent audit trail. They can expire, and some failures may not produce a useful event, so inspect logs and controller status as well.

### 3. `kubectl logs`: What did the application report?

`kubectl logs` retrieves a container's standard output and standard error.

```bash
kubectl logs <pod-name>
```

For a web application, logs might show:

```text
Starting API server on port 8080
Connected to database
ERROR: connection timeout
```

Logs are useful for investigating:
- Application exceptions.
- Failed database connections.
- Configuration errors.
- Startup failures.
- Request processing errors.
- Dependency timeouts.

By default, `kubectl logs` reads the logs of the selected container's current instance. If the container has restarted, use `--previous` to inspect the previous instance's logs, if they are still available.

Kubernetes does not automatically make every application's logs durable. Log retention depends on the container runtime, node configuration, and any centralized logging system.

### 4. `kubectl exec`: What is happening inside the container?

`kubectl exec` runs a command inside a running container.

```bash
kubectl exec <pod-name> -- <command>
```

For example:

```bash
kubectl exec api-pod -- printenv
```

You can use `exec` to:
- Inspect environment variables.
- Check files and mounted configuration.
- Test DNS resolution and connectivity.
- Inspect application processes.
- Verify whether a file exists at a mounted volume path.

To open an interactive shell:

```bash
kubectl exec -it <pod-name> -- sh
```

The `-i` option keeps standard input open, while `-t` allocates a terminal.

A shell such as `bash` may not exist in minimal container images. Try `sh` if it is available.

**Important:** `exec` requires a running container and an executable that exists inside it. It is not a way to run commands on the Kubernetes node itself. Avoid making untracked changes inside production containers, because those changes may disappear when the container is recreated.

## Key points

- **Use the right command for the question:**
  - `get`: What resources exist, and what is their status?
  - `describe`: What are the detailed state and events?
  - `logs`: What did the application output?
  - `exec`: What can I inspect or test from inside the container?
- **Namespaces matter:** A command without `-n` uses the current namespace from your kubectl context, often `default`.
- **Resource names matter:** `kubectl describe pod api-pod` inspects a Pod named `api-pod`; it does not search every Pod with a similar name.
- **Labels help with groups:** `-l app=api` selects resources carrying the matching label.
- **Pods can contain multiple containers:** Specify `-c <container-name>` when logs or execution should target a particular container.
- **Previous logs are important:** `kubectl logs --previous` is useful when a container crashes before you can inspect its current output.
- **Follow logs live:** `kubectl logs -f` streams new log lines as they are written.
- **`get -o yaml` reveals configuration:** It is useful when the concise table omits fields you need to inspect.
- **`get -o wide` adds details:** For Pods, it commonly includes the node and Pod IP.
- **Events provide clues, not guarantees:** Events can expire, and the absence of an event does not prove that no failure occurred.
- **A running Pod can still be unhealthy:** The application may return errors, fail readiness checks, or lose connectivity to dependencies.
- **`exec` is not always available:** A container may terminate quickly, have no shell, or run as a user without sufficient permissions.
- **A Pod is not the same as a Deployment:** Inspect the Pod to investigate a particular container, and inspect the Deployment or other controller to understand replicas, rollout status, and desired configuration.
- **Use the right permissions:** Kubernetes RBAC can allow reading Pod status while denying log access or command execution.
- **Be cautious in production:** `exec` can expose secrets, alter files, or change application state. Prefer read-only inspection and follow your team's access controls.

## Commands / manifests

### 1. Set the Kubernetes context and namespace

Before troubleshooting, make sure you are connected to the intended cluster and namespace.

```bash
# Display the current context
kubectl config current-context

# List available contexts
kubectl config get-contexts

# List namespaces
kubectl get namespaces

# List Pods in a specific namespace
kubectl get pods -n staging

# Set a default namespace for the current context
kubectl config set-context --current --namespace=staging
```

Changing the current context's namespace affects subsequent commands that omit `-n`. Verify the cluster and namespace before running commands that could modify resources.

### 2. Use `kubectl get`

```bash
# List Pods in the current namespace
kubectl get pods

# List Pods in all namespaces
kubectl get pods -A

# List Deployments
kubectl get deployments

# List Services
kubectl get services

# List Pods with node and IP information
kubectl get pods -o wide

# Display labels
kubectl get pods --show-labels

# Filter by a label
kubectl get pods -l app=api

# Continuously refresh resource output
kubectl get pods -w

# Display full resource configuration and status as YAML
kubectl get pod <pod-name> -o yaml

# Display selected fields
kubectl get pod <pod-name> \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,NODE:.spec.nodeName,IP:.status.podIP
```

Use `-w` when you want to observe changes such as a Pod moving from `Pending` to `Running`. Press `Ctrl+C` to stop watching.

### 3. Use `kubectl describe`

```bash
# Describe a Pod
kubectl describe pod <pod-name>

# Describe a Pod in a specific namespace
kubectl describe pod <pod-name> -n staging

# Describe a Deployment
kubectl describe deployment <deployment-name>

# Describe a Service
kubectl describe service <service-name>

# Describe a Node
kubectl describe node <node-name>

# Inspect recent namespace events
kubectl get events -n staging \
  --sort-by=.metadata.creationTimestamp
```

When inspecting events, look for:
- `FailedScheduling`: The scheduler cannot place the Pod on a suitable node.
- `FailedMount`: Kubernetes cannot mount a required volume.
- `FailedAttachVolume`: A volume could not be attached.
- `ErrImagePull` or `ImagePullBackOff`: The container image could not be retrieved.
- Probe failures: A configured liveness, readiness, or startup probe is failing.
- `BackOff`: Kubernetes is delaying another attempt after repeated failures.

Event reasons and messages depend on the underlying failure.

### 4. Use `kubectl logs`

```bash
# Read the current logs
kubectl logs <pod-name>

# Read logs from a specific namespace
kubectl logs <pod-name> -n staging

# Select a container in a multi-container Pod
kubectl logs <pod-name> -c <container-name>

# Follow logs continuously
kubectl logs -f <pod-name>

# Show the last 100 lines
kubectl logs <pod-name> --tail=100

# Show logs from the previous container instance
kubectl logs <pod-name> --previous

# Show logs from the previous instance of a specific container
kubectl logs <pod-name> -c <container-name> --previous

# Show logs from a recent time window
kubectl logs <pod-name> --since=10m

# Include timestamps
kubectl logs <pod-name> --timestamps

# Read logs from Pods selected by a label
kubectl logs -l app=api --all-containers=true --tail=100
```

For a Deployment with several replicas, label-based log retrieval can help gather output from multiple matching Pods. The results are not necessarily a single chronologically ordered application-wide log stream.

### 5. Use `kubectl exec`

```bash
# Run a command inside a container
kubectl exec <pod-name> -- printenv

# Open an interactive shell
kubectl exec -it <pod-name> -- sh

# Select a container
kubectl exec -it <pod-name> -c <container-name> -- sh

# Inspect the current directory
kubectl exec <pod-name> -- pwd

# List files
kubectl exec <pod-name> -- ls -lah /app

# Read a configuration file
kubectl exec <pod-name> -- cat /app/config.yaml

# Inspect environment variables
kubectl exec <pod-name> -- printenv

# Check a mounted storage path
kubectl exec <pod-name> -- ls -lah /data
```

Commands such as `ls`, `cat`, and `printenv` must exist in the container image. A minimal image may not contain a shell or common diagnostic utilities.

### 6. Work with multi-container Pods

A Pod may contain an application container, a sidecar, and an initialization container. Choose the correct container when inspecting logs.

```bash
# List container names in a Pod
kubectl get pod <pod-name> \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'

# Read logs from a specific container
kubectl logs <pod-name> -c app

# Read logs from all containers
kubectl logs <pod-name> --all-containers=true

# Execute a command in the application container
kubectl exec <pod-name> -c app -- printenv
```

Regular `kubectl logs` targets application containers. Init-container logs can be retrieved by specifying the init container's name with `-c`.

### 7. Hands-on troubleshooting exercise

Create a sample Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: troubleshooting-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: troubleshooting-demo
  template:
    metadata:
      labels:
        app: troubleshooting-demo
    spec:
      containers:
        - name: web
          image: nginx:stable
          ports:
            - containerPort: 80
```

Save it as `troubleshooting-demo.yaml`, then apply it:

```bash
kubectl apply -f troubleshooting-demo.yaml
```

**Step 1: Check whether the Pod exists and is running.**

```bash
kubectl get deployment troubleshooting-demo
kubectl get pods -l app=troubleshooting-demo -o wide
```

**Step 2: Inspect the Pod's detailed state and events.**

```bash
kubectl describe pod -l app=troubleshooting-demo
```

**Step 3: Read the web server logs.**

First retrieve the Pod name:

```bash
kubectl get pods -l app=troubleshooting-demo
```

Then use the returned name:

```bash
kubectl logs <pod-name>
```

**Step 4: Inspect the container.**

```bash
kubectl exec <pod-name> -- nginx -v
kubectl exec <pod-name> -- cat /etc/nginx/nginx.conf
```

These commands inspect the NGINX version and configuration. They do not change the configuration.

**Step 5: Clean up the exercise.**

```bash
kubectl delete deployment troubleshooting-demo
```

This exercise demonstrates the four commands without deliberately breaking a workload.

## Common failures & fixes

- **`kubectl get pods` shows no resources**
  - Cause: You may be using the wrong namespace or cluster context.
  - Fix: Run `kubectl config current-context`, `kubectl get namespaces`, and `kubectl get pods -A`. Specify the intended namespace with `-n`.

- **`kubectl describe pod` reports that the resource was not found**
  - Cause: The Pod name is incorrect, the Pod has already been deleted, or the command targets the wrong namespace.
  - Fix: Run `kubectl get pods -n <namespace>` and copy the exact Pod name.

- **`kubectl logs` returns no output**
  - Cause: The application may not have written anything to standard output or standard error, logs may have rotated, or the wrong container may have been selected.
  - Fix: Check `kubectl describe pod <pod-name>`, select the correct container with `-c`, and inspect the application's logging configuration. Check centralized logging if the environment uses it.

- **`kubectl logs` says the container is waiting to start**
  - Cause: The container has not started successfully, the image is still being pulled, or the Pod is waiting on another prerequisite.
  - Fix: Run `kubectl describe pod <pod-name>` and inspect its Events section for image, scheduling, volume, or configuration errors.

- **The container crashed, and its current logs are empty**
  - Cause: The current container instance may not have produced output, while the useful output belongs to the previous instance.
  - Fix: Run `kubectl logs <pod-name> --previous`. If the Pod has multiple containers, add `-c <container-name>`.

- **`kubectl exec` reports that the container is not running**
  - Cause: The container is waiting, has terminated, or is repeatedly crashing.
  - Fix: Check `kubectl get pod <pod-name>` and `kubectl describe pod <pod-name>`. Use `kubectl logs --previous` if the container has restarted.

- **`kubectl exec -it <pod-name> -- bash` fails**
  - Cause: The image may not include Bash.
  - Fix: Try `kubectl exec -it <pod-name> -- sh`, provided the image contains a shell. Minimal or distroless images may contain neither.

- **`kubectl exec` fails with `Forbidden`**
  - Cause: Kubernetes RBAC does not authorize the requested operation, or your identity lacks permission to access the resource.
  - Fix: Check permissions with `kubectl auth can-i get pods`, `kubectl auth can-i get pods/log`, and `kubectl auth can-i create pods/exec` in the intended namespace. Request the appropriate access from the cluster administrator.

- **`kubectl describe` shows probe failures**
  - Cause: The application may not be ready, may be listening on a different port, or may not respond to the configured health-check path.
  - Fix: Inspect the probe configuration in the Pod specification, read application logs, and verify the endpoint and port from the container where possible.

- **`kubectl get pods` shows `CrashLoopBackOff`**
  - Cause: The container repeatedly exits, often because of application errors, invalid configuration, missing dependencies, or an incorrect startup command.
  - Fix: Run `kubectl describe pod <pod-name>` and `kubectl logs <pod-name> --previous`. Inspect environment variables, resource limits, and dependency connectivity.

- **`kubectl get pods` shows `ImagePullBackOff`**
  - Cause: The image name or tag is invalid, the registry is unavailable, or image-pull credentials are missing or incorrect.
  - Fix: Inspect the Pod events with `kubectl describe pod <pod-name>`. Verify the image reference, registry access, and image-pull configuration.

- **`kubectl get pods` shows `Pending`**
  - Cause: The Pod cannot yet be scheduled or a required resource is not ready.
  - Fix: Use `kubectl describe pod <pod-name>` to look for insufficient CPU or memory, node constraints, unbound PVCs, or scheduling restrictions.

- **Logs are incomplete or disappear after a restart**
  - Cause: The container runtime may retain only a limited amount of local log data, or the container may have restarted multiple times.
  - Fix: Retrieve `--previous` logs promptly and use a centralized logging system for durable, searchable logs across Pods and nodes.

- **A command targets the wrong cluster**
  - Cause: The active kubectl context is not the intended environment.
  - Fix: Check `kubectl config current-context` and `kubectl config get-contexts` before troubleshooting or making changes. Use `--context` when explicitly targeting another configured context.

## References

- [Kubernetes documentation: kubectl overview](https://kubernetes.io/docs/reference/kubectl/)
- [Kubernetes documentation: kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [Kubernetes documentation: kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [Kubernetes documentation: kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
- [Kubernetes documentation: kubectl exec](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/)
- [Kubernetes documentation: Debugging Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes documentation: Debugging Applications](https://kubernetes.io/docs/tasks/debug/)
