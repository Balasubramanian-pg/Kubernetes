# Port-forward for debugging

> Phase 2 — Kubernetes Core (highest priority) › Networking

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

`kubectl port-forward` allows you to access a Kubernetes Pod or Service through a port on your local machine without exposing the application externally.

It creates a temporary connection between a local port and a port inside a Kubernetes Pod.

This is particularly useful when debugging applications that are only accessible inside the cluster, such as:

- Databases running inside Kubernetes.
- Internal APIs and backend services.
- Applications exposed only through ClusterIP Services.
- Development dashboards and monitoring tools.
- Applications that do not have an Ingress or external LoadBalancer.

For example, suppose a PostgreSQL database is running inside a Kubernetes cluster and is exposed through a Service named `postgres-service` on port `5432`.

Instead of exposing PostgreSQL publicly, you can run:

```bash
kubectl port-forward service/postgres-service 5432:5432
```

You can then connect to the database from your local machine using:

```text
localhost:5432
```

The connection is forwarded through Kubernetes to the backend Pod.

### How port-forward works

```text
Local Machine
      |
      | localhost:5432
      v
kubectl port-forward
      |
      | Secure connection to Kubernetes API server
      v
Kubernetes API Server
      |
      | Port-forward connection
      v
Target Pod
      |
      v
Application listening on port 5432
```

The local port is not exposed to the internet by default. It is usually bound to `localhost`, making it accessible from your own machine.

### Port-forwarding a Pod vs a Service

You can forward traffic directly to a Pod or through a Service.

| Target | Example | Behaviour |
|---|---|---|
| Pod | `kubectl port-forward pod/web-abc123 8080:80` | Forwards directly to the selected Pod |
| Service | `kubectl port-forward service/web-service 8080:80` | Selects a backend Pod associated with the Service |
| Deployment | `kubectl port-forward deployment/web 8080:80` | Selects a Pod managed by the Deployment |

When forwarding to a Service, `kubectl` selects a Pod behind that Service and establishes the port-forward to that Pod.

Port-forwarding to a Service does not create a permanent load-balanced connection across all replicas. The forwarding session is associated with a selected Pod.

### Understanding the port mapping

The command format is:

```text
kubectl port-forward <resource> <local-port>:<remote-port>
```

For example:

```bash
kubectl port-forward pod/web-abc123 8080:80
```

This means:

- `8080`: Port on your local machine.
- `80`: Port inside the target Pod.

Your local application connects to:

```text
http://localhost:8080
```

The traffic is forwarded to port `80` inside the Pod.

The local and remote ports do not need to be the same.

### Port-forwarding is a debugging tool, not a production exposure mechanism

Port-forwarding is useful for temporary access during development and troubleshooting.

It is not a replacement for:

- Kubernetes Services.
- Ingress controllers.
- LoadBalancer Services.
- Production networking configuration.

The command runs as a foreground process. Closing the terminal or stopping the command ends the forwarding session.

## Key points

- `kubectl port-forward` creates a temporary local connection to a port inside a Kubernetes Pod.
- It is useful for debugging applications that are not externally exposed.
- It can target a Pod, Service, Deployment, ReplicaSet, or StatefulSet.
- The syntax is `kubectl port-forward <resource> <local-port>:<remote-port>`.
- The local port can differ from the remote Pod port.
- The target Pod must be running and have the requested port available.
- Port-forwarding does not require an Ingress or LoadBalancer.
- Forwarding to a Service selects a backend Pod for the session.
- It does not provide normal Service load balancing across all replicas.
- Port-forwarding does not modify the Kubernetes Service or Deployment.
- The command typically binds to `localhost` by default.
- Port-forwarding is temporary and ends when the process stops or the connection is interrupted.
- It can be used to debug HTTP applications, databases, dashboards, and other TCP-based services.
- It is not a production network exposure strategy.
- Port-forwarding does not bypass application authentication or authorization. Applications may still require credentials.
- Access control still depends on Kubernetes permissions, including permission to create port-forward connections to Pods.

## Commands / manifests

### 1. Create a test application

Create a namespace:

```bash
kubectl create namespace port-forward-lab
```

Deploy an NGINX application:

```bash
kubectl create deployment web \
  --image=nginx:stable \
  -n port-forward-lab
```

Expose it internally through a ClusterIP Service:

```bash
kubectl expose deployment web \
  --name=web-service \
  --port=80 \
  --target-port=80 \
  -n port-forward-lab
```

Verify the resources:

```bash
kubectl get pods,services -n port-forward-lab
```

### 2. Forward a local port to a Service

```bash
kubectl port-forward service/web-service 8080:80 \
  -n port-forward-lab
```

Expected output resembles:

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Keep the command running in the terminal.

Open a second terminal and access the application:

```bash
curl http://localhost:8080
```

You should receive the NGINX welcome page HTML.

You can also open:

```text
http://localhost:8080
```

in a browser.

### 3. Forward directly to a Pod

Get the Pod name:

```bash
kubectl get pods -n port-forward-lab
```

Forward local port `8080` to Pod port `80`:

```bash
kubectl port-forward pod/<pod-name> 8080:80 \
  -n port-forward-lab
```

Then access the application:

```bash
curl http://localhost:8080
```

This is useful when you want to debug a specific Pod rather than the Service.

### 4. Forward to a Deployment

```bash
kubectl port-forward deployment/web 8080:80 \
  -n port-forward-lab
```

This selects a Pod managed by the Deployment and forwards traffic to it.

If you need to inspect a particular replica, target its Pod explicitly.

### 5. Use the same local and remote port

```bash
kubectl port-forward service/web-service 80:80 \
  -n port-forward-lab
```

You can now access the application using:

```bash
curl http://localhost
```

This requires local port `80` to be available and may require elevated privileges depending on your operating system.

### 6. Bind to a specific local address

By default, `kubectl port-forward` binds to localhost.

You can specify a local address:

```bash
kubectl port-forward \
  --address 127.0.0.1 \
  service/web-service \
  8080:80 \
  -n port-forward-lab
```

You can also bind to all local interfaces:

```bash
kubectl port-forward \
  --address 0.0.0.0 \
  service/web-service \
  8080:80 \
  -n port-forward-lab
```

Binding to `0.0.0.0` can make the forwarded port reachable from other machines that can access your computer. Avoid doing this unless you explicitly need it and understand the security implications.

### 7. Forward a database port

Suppose a PostgreSQL Service is named `postgres-service` and exposes port `5432`.

```bash
kubectl port-forward service/postgres-service 15432:5432 \
  -n database
```

Connect from your local machine using:

```text
Host: localhost
Port: 15432
```

For example, with `psql`:

```bash
psql -h localhost -p 15432 -U postgres
```

This can be useful for connecting from local database clients such as SSMS alternatives, DBeaver, pgAdmin, or command-line clients, depending on the database type.

### 8. Forward multiple ports

You can forward more than one port in the same command:

```bash
kubectl port-forward pod/<pod-name> \
  8080:80 \
  8443:443 \
  -n port-forward-lab
```

This maps:

- Local port `8080` to Pod port `80`.
- Local port `8443` to Pod port `443`.

### 9. Check Kubernetes access

Port-forwarding requires access to the Kubernetes cluster and appropriate RBAC permissions.

Verify your current context:

```bash
kubectl config current-context
```

Check whether you can access the target Pod:

```bash
kubectl get pods -n port-forward-lab
```

### 10. Clean up

Stop the port-forward command using:

```text
Ctrl + C
```

Delete the test namespace:

```bash
kubectl delete namespace port-forward-lab
```

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `unable to listen on port` | Local port is already in use | Choose another local port, such as `8081:80` |
| `pod not found` | Wrong Pod name or namespace | Run `kubectl get pods -n <namespace>` |
| `service not found` | Wrong Service name or namespace | Run `kubectl get services -n <namespace>` |
| `connection refused` | Application is not listening on the target port | Check the Pod's application port and logs |
| `pod is not running` | Target Pod is Pending, Failed, or not ready | Inspect Pod status and events |
| `error forwarding port` | Connection to the target Pod failed | Check Pod health, networking, and kubelet/API connectivity |
| Port-forward starts but HTTP fails | Wrong remote port or application protocol | Verify the actual container listening port |
| Forwarding stops unexpectedly | Pod was deleted, restarted, or the connection was interrupted | Restart the command and inspect the Pod lifecycle |
| Service port-forward targets an unexpected Pod | Service selector matches multiple backend Pods | Inspect Service selectors and endpoints; target a specific Pod when debugging |
| Cannot access from another machine | Forwarding is bound to localhost | Use an explicit address only if remote access is needed and safe |
| Permission denied | Kubernetes RBAC does not allow port-forwarding | Ask for the appropriate permission, commonly `pods/portforward` |

### Troubleshooting sequence

Inspect the target Pod:

```bash
kubectl get pods -n port-forward-lab
kubectl describe pod <pod-name> -n port-forward-lab
kubectl logs <pod-name> -n port-forward-lab
```

Inspect the Service:

```bash
kubectl describe service web-service -n port-forward-lab
kubectl get endpointslices -n port-forward-lab
```

Check local port availability:

```bash
netstat -ano
```

On Linux, you can also use:

```bash
ss -ltnp
```

Confirm the remote application port is correct. A Kubernetes Service might expose port `80`, while the container application listens on another port. When forwarding to a Service, specify its Service port. When forwarding directly to a Pod, specify the Pod's actual listening port.

## References

- [kubectl port-forward reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Debugging applications in Kubernetes](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
