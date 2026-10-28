# kubectl port-forward

> Phase 2 — Kubernetes Core (highest priority) › kubectl fluency

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

`kubectl port-forward` establishes a temporary connection between a port on your local machine and a port on a Kubernetes resource, such as a Pod, Deployment, or Service.

It lets you access an application running inside a Kubernetes cluster through `localhost` without exposing the application publicly or creating an external-facing Service.

For example, suppose an application runs inside a Pod and listens on port `8080`. You can forward local port `8080` to the application's port and access it through `http://localhost:8080`.

The connection is initiated from your machine through Kubernetes API access. You do not need to configure an Ingress, a LoadBalancer Service, or a NodePort Service for this temporary access.

**How it works:**

1. You execute `kubectl port-forward` from your local machine.
2. `kubectl` uses your current Kubernetes context and credentials to establish the forwarding connection.
3. Kubernetes forwards traffic from the selected local port to the specified port on the target Pod.
4. You access the application through your local address, typically `localhost`.
5. When you stop the command, the forwarding session ends.

**Example:**

A web application runs inside a Pod on port `8080`.

```bash
kubectl port-forward pod/my-app 8080:8080
```

Open `http://localhost:8080` in your browser.

The first port, `8080`, is on your local machine. The second port, `8080`, is the destination port inside the Pod.

Port forwarding is primarily intended for development, debugging, and temporary administrative access. It is not a replacement for a Kubernetes Service, Ingress, or a production networking solution.

## Key points

- **Local-to-Pod forwarding:** The basic syntax is `kubectl port-forward <resource-type>/<resource-name> <local-port>:<remote-port>`.
- **Port mapping:** In `8080:80`, local port `8080` forwards to destination port `80`. The two ports do not have to match.
- **Supported targets:** You can forward to a Pod directly or use supported higher-level resources such as a Deployment, ReplicaSet, or Service. When targeting a controller or Service, kubectl selects a Pod to forward to.
- **Pod selection:** Forwarding to a Deployment or Service does not distribute traffic across all its Pods like a normal Service. The session forwards through a selected Pod.
- **Local address:** By default, kubectl listens on `localhost`, usually resolving to both IPv4 and IPv6 loopback addresses where supported.
- **Session lifetime:** The command runs in the foreground. Press `Ctrl+C` to stop the forwarding session.
- **Temporary access:** Port forwarding does not permanently expose an application to the internet or create a new Kubernetes Service.
- **Namespace targeting:** Use `-n` to specify the namespace containing the resource.
- **Resource port:** The destination port must be reachable on the selected Pod. It does not always have to be explicitly declared under `containerPort` in the Pod manifest, although the application must actually listen on that port.
- **Background processes:** You can run port forwarding in the background, but managing the process and detecting failures becomes your responsibility. For learning, keep it in the foreground.
- **Local port conflicts:** If another application already uses the chosen local port, select a different local port.
- **Network restrictions:** The Kubernetes API connection must be available, and your identity must have the required permissions.
- **Pod readiness:** The target Pod must be available for forwarding. If the Pod terminates or the connection fails, the session may stop and need to be restarted.
- **Security:** By default, the forwarded port is available only through loopback interfaces on your machine. Avoid binding it to all interfaces unless you explicitly need that access and understand the security implications.
- **No public URL:** Other machines cannot normally access your `localhost` address. They would need their own access path, and exposing the listener beyond loopback requires additional care.
- **Not a permanent endpoint:** Port forwarding is useful for testing an application, inspecting a dashboard, or connecting a local database client. It is not designed for stable application-to-application communication.
- **Service vs port-forward:** A Service provides a stable Kubernetes networking endpoint for workloads. Port forwarding creates a temporary tunnel from your local machine to a selected workload.
- **Application binding:** The application must listen on an address and port reachable inside the Pod's network namespace. A process listening only on an unexpected interface or a different port can cause connection failures.
- **Local and remote ports:** If local port `9000` forwards to remote port `8080`, use `http://localhost:9000`, not `http://localhost:8080`.
- **RBAC:** Depending on the cluster's authorization configuration, port forwarding commonly requires permission for the `pods/portforward` subresource, as well as permission to discover or access the target resource.
- **Connection interruptions:** A network interruption, Pod replacement, or API connection failure can break the session. Restart the command after confirming that the target resource is healthy.

## Commands / manifests

### 1. Check your Kubernetes context

```bash
kubectl config current-context
kubectl get pods
```

The first command shows which cluster and context kubectl is using. The second lists Pods in the current namespace.

If the application is in another namespace:

```bash
kubectl get pods -n development
```

Always verify the context and namespace before forwarding to a workload, especially when working with multiple environments.

### 2. Forward a local port to a Pod

```bash
kubectl port-forward pod/my-app 8080:8080
```

This forwards local port `8080` to port `8080` on the selected Pod.

Expected output resembles:

```text
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Open this URL locally:

```text
http://localhost:8080
```

The application must be running and listening on the destination port.

### 3. Use different local and remote ports

```bash
kubectl port-forward pod/my-app 9000:8080
```

This maps:

| Port | Purpose |
|---|---|
| `9000` | Port on your local machine |
| `8080` | Destination port on the Pod |

Access the application at:

```text
http://localhost:9000
```

This is useful when the destination port is already occupied on your machine.

### 4. Forward a port in a specific namespace

```bash
kubectl port-forward pod/my-app 8080:8080 -n development
```

The Pod must exist in the `development` namespace.

You can verify the target first:

```bash
kubectl get pod my-app -n development
```

### 5. Forward a port to a Deployment

```bash
kubectl port-forward deployment/my-app 8080:8080
```

Kubernetes selects a Pod associated with the Deployment and forwards traffic through that Pod.

This is convenient when you know the Deployment name but do not want to identify a specific Pod manually.

Remember that this does not load-balance your local requests across every replica. The forwarding session uses a selected Pod.

### 6. Forward a port to a Service

```bash
kubectl port-forward service/my-service 8080:80
```

This forwards local port `8080` to the Service's port `80`. Kubectl resolves the Service target and forwards through a selected Pod.

The distinction between Service ports is important:

- `port`: The port exposed by the Service inside the cluster.
- `targetPort`: The destination port on the Pod.
- Local port: The port on your computer.

For example, if a Service exposes port `80` and maps it to Pod port `8080`, forwarding to `service/my-service 8080:80` makes the Service's port `80` accessible at local port `8080`.

Inspect the Service configuration if unsure:

```bash
kubectl get service my-service -o yaml
kubectl describe service my-service
```

### 7. Let kubectl select an available local port

```bash
kubectl port-forward pod/my-app :8080
```

An empty local-port value tells kubectl to choose an available local port and forward it to Pod port `8080`.

The output displays the selected local port. Use that port when accessing the application.

This is useful when you do not care which local port is assigned or want to avoid a conflict with an existing process.

### 8. Bind to a specific local address

```bash
kubectl port-forward --address 127.0.0.1 pod/my-app 8080:8080
```

This explicitly binds the listener to the IPv4 loopback address.

To listen on all local network interfaces:

```bash
kubectl port-forward --address 0.0.0.0 pod/my-app 8080:8080
```

Use the second command only when required. Binding to `0.0.0.0` can make the forwarded application reachable from other machines that can access your computer's network interface and port. Access is still subject to host firewall and network rules, but this can expose sensitive services.

### 9. Forward a database port for local development

Suppose PostgreSQL is exposed by a Service named `postgres` on port `5432`.

```bash
kubectl port-forward service/postgres 5432:5432 -n development
```

Configure your local database client with:

| Setting | Value |
|---|---|
| Host | `localhost` |
| Port | `5432` |
| Database | Your database name |
| Username | Your configured database username |
| Password | Your configured database password |

The database must be running and accessible through the selected Pod. Port forwarding does not bypass database authentication.

If PostgreSQL is listening on port `5432` but your local machine already uses that port, choose a different local port:

```bash
kubectl port-forward service/postgres 15432:5432 -n development
```

Then connect your database client to `localhost:15432`.

### 10. Hands-on example using a Deployment

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
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

Apply the manifest:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/web
kubectl get pods
```

Forward local port `8080` to the Deployment's Pod port `80`:

```bash
kubectl port-forward deployment/web 8080:80
```

Open:

```text
http://localhost:8080
```

You should see the NGINX welcome page.

When finished, stop the port-forwarding process with `Ctrl+C`.

Remove the test Deployment:

```bash
kubectl delete -f deployment.yaml
```

This example uses port forwarding directly to the Deployment. A Service is not required for this particular local access pattern.

### 11. Check the command's available options

```bash
kubectl port-forward --help
```

This displays supported resource types, port syntax, address options, and other flags for the kubectl version installed on your machine.

## Common failures & fixes

- **`address already in use`**
  - Cause: Another local process is already listening on the selected port.
  - Fix: Choose a different local port, such as `9000:8080`, or identify the process using the port.

- **`pod not found`**
  - Cause: The Pod name is incorrect, the Pod is in another namespace, or it no longer exists.
  - Fix: Run `kubectl get pods -n <namespace>` and verify the resource name and namespace.

- **`unable to forward port` or `connection refused`**
  - Cause: The application is not listening on the destination port, the target port is wrong, or the selected Pod is not ready.
  - Fix: Inspect the Pod using `kubectl describe pod <pod-name>` and check application logs with `kubectl logs <pod-name>`. Confirm the actual listening port inside the container.

- **The command starts, but the browser cannot load the page**
  - Cause: The wrong local port or protocol is being used, the application is unhealthy, or the request is reaching a port where no service is listening.
  - Fix: Check the forwarding output and use the displayed local port. Confirm whether the application expects HTTP or HTTPS and inspect its logs.

- **`Forbidden` or an authorization error**
  - Cause: Your Kubernetes identity lacks the permissions needed to access the target resource or create a port-forward session.
  - Fix: Check `kubectl auth can-i get pods -n <namespace>` and `kubectl auth can-i create pods/portforward -n <namespace>`. Ask the cluster administrator for the required least-privilege access if needed.

- **The connection stops unexpectedly**
  - Cause: The Pod was terminated or replaced, the network connection was interrupted, or access to the Kubernetes API was lost.
  - Fix: Verify the Pod and Deployment status, then restart the port-forward command. Investigate repeated interruptions rather than assuming the tunnel is permanent.

- **Forwarding to a Service fails**
  - Cause: The Service has no ready endpoints, its port is incorrect, or the selected Pod is not serving the expected port.
  - Fix: Run `kubectl describe service <service-name>` and `kubectl get endpointslices -l kubernetes.io/service-name=<service-name>`. Check the Service selector and Pod labels.

- **The application is listening on a different port**
  - Cause: The remote port in the command does not match the port used by the application.
  - Fix: Inspect the application configuration and Pod specification. Remember that `containerPort` documents a port in the Pod spec but does not make an application listen on that port automatically.

- **A Deployment has multiple replicas, but traffic reaches only one Pod**
  - Cause: Port forwarding to a Deployment selects a Pod for the session rather than load-balancing across all replicas.
  - Fix: Use a Kubernetes Service for normal in-cluster traffic, or establish a separate forwarding session when specifically testing an individual Pod.

- **Other machines cannot access the forwarded application**
  - Cause: The default listener binds to loopback addresses on your computer.
  - Fix: Keep loopback binding when access should remain local. If remote access is genuinely required, evaluate a secure access method before considering `--address 0.0.0.0`.

- **The Pod is running, but forwarding fails after a restart**
  - Cause: The previous Pod was replaced and the existing forwarding session was tied to the old Pod.
  - Fix: Restart the command against the Deployment or Service, or select the current Pod. A new session may be required.

- **HTTPS or TLS errors occur**
  - Cause: The application expects HTTPS, uses a certificate for another hostname, or requires specific TLS configuration.
  - Fix: Use the correct protocol and inspect the application's TLS requirements. Port forwarding transports connections but does not rewrite certificates, hostnames, or application protocols.

## References

- [kubectl port-forward command reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes API access and authentication](https://kubernetes.io/docs/concepts/security/controlling-access/)
- [Kubernetes RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
