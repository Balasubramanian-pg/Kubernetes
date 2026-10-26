# Services

> Phase 2 — Kubernetes Core (highest priority) › Networking

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **Service** is a Kubernetes resource that provides a stable network endpoint for accessing a group of Pods.

Pods are temporary. They can be deleted, recreated, rescheduled to different nodes, and assigned new IP addresses. If another application connects directly to a Pod IP, that connection may break when the Pod is replaced.

A Service solves this problem by providing a stable endpoint and directing traffic to eligible backend Pods selected through labels.

For example, suppose you deploy three replicas of a Python API. Instead of making clients connect to each Pod individually, you create a Service that targets all three Pods. Clients connect to the Service, and Kubernetes networking routes the traffic to an eligible backend.

Services are essential for:
- Allowing applications to communicate with other applications inside a cluster.
- Exposing applications to clients outside the cluster.
- Distributing traffic across replicated Pods.
- Providing stable DNS names for applications.
- Decoupling application access from individual Pod IP addresses.

### How a Service works

Consider an API application with three Pods.

```text
Client Application
        |
        v
  Kubernetes Service
        |
        |-- API Pod 1
        |-- API Pod 2
        |-- API Pod 3
```

The general process is:

1. You create a Service with a selector, such as `app: api`.
2. Kubernetes identifies matching Pods that are eligible to receive traffic.
3. Kubernetes maintains EndpointSlices describing the Service's backend endpoints.
4. The cluster's networking implementation routes traffic addressed to the Service to an eligible backend.
5. Clients continue using the same Service endpoint even when individual Pods are replaced.

The Service's stable endpoint does not mean every request is guaranteed to succeed. The backend must be healthy, correctly configured, and reachable through the cluster's networking implementation.

### Service selectors and labels

A Service usually identifies backend Pods using labels.

For example, a Pod may have:

```yaml
metadata:
  labels:
    app: api
```

The Service selects it using:

```yaml
spec:
  selector:
    app: api
```

If the selector does not match the Pod labels, the Service will not discover those Pods as its backends.

A Service can also be created without a selector, for example when you want to represent an external endpoint manually. In that case, Kubernetes does not automatically discover matching Pods through the selector.

### The three main Service types

Kubernetes supports several Service types. The three most important for this topic are `ClusterIP`, `NodePort`, and `LoadBalancer`.

#### 1. ClusterIP

`ClusterIP` exposes the Service through an internal cluster IP address. It is the default Service type.

Use it when an application needs to be accessed by other workloads inside the cluster, but does not need direct external exposure.

Examples:
- A backend API accessed by a frontend Pod.
- An internal authentication service.
- A data processing service accessed by another workload.
- A database service that should not be directly exposed to external clients.

Traffic flow:

```text
Frontend Pod
     |
     v
ClusterIP Service
     |
     v
Backend Pods
```

A ClusterIP Service is normally reachable from within the cluster. External access is possible through additional routing or networking arrangements, but ClusterIP alone does not provision an external endpoint.

#### 2. NodePort

`NodePort` exposes the Service on a port across eligible cluster nodes, typically using a port from the configured NodePort range.

Traffic can enter through a node's IP address and the assigned NodePort, then be forwarded to a backend Pod.

Example:

```text
External Client
      |
      v
Node IP : NodePort
      |
      v
Kubernetes Service
      |
      v
Backend Pod
```

Use NodePort when you need node-level access to a Service, often for development, testing, or environments where an external load balancer is configured separately.

Important considerations:
- The default NodePort range is `30000-32767`, although the cluster can be configured differently.
- The Service's `port` is not necessarily the same as its `nodePort`.
- A NodePort does not automatically create a public IP address or configure firewall rules.
- The node and network must be reachable from the client.

#### 3. LoadBalancer

`LoadBalancer` requests an external load-balancing endpoint through the cluster's infrastructure or an installed load-balancer implementation.

In a cloud environment, Kubernetes can integrate with the provider to provision a load balancer, subject to the cluster's configuration and permissions.

Traffic flow:

```text
External Client
      |
      v
External Load Balancer
      |
      v
Kubernetes Service
      |
      v
Backend Pods
```

Use it when clients outside the cluster need a stable external endpoint and the environment supports load balancing.

Important considerations:
- An external IP or hostname may take time to appear.
- A compatible cloud integration or load-balancer controller may be required.
- The Service's external reachability depends on networking, firewall rules, and cloud configuration.
- A LoadBalancer Service is not automatically a full HTTP ingress or API gateway.

### Service type comparison

| Feature | ClusterIP | NodePort | LoadBalancer |
|---|---|---|---|
| Primary access | Internal cluster endpoint | Node IP and assigned port | External load-balancer endpoint |
| Default type | Yes | No | No |
| External endpoint provisioned automatically | No | No | Requested through supported infrastructure |
| Typical use | Internal APIs and backends | Development or node-level access | Externally accessible applications |
| Requires matching backend Pods | Usually, for Pod-backed traffic | Usually, for Pod-backed traffic | Usually, for Pod-backed traffic |
| Stable Service endpoint | Yes | Yes | Yes, with external endpoint availability dependent on infrastructure |

### Important distinction: port, targetPort, and nodePort

These fields describe different parts of the traffic path.

- `port`: The port exposed by the Service.
- `targetPort`: The port on the backend Pod that receives the traffic.
- `nodePort`: The port exposed on cluster nodes for a NodePort Service.

For example:

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

The meaning is:

- Clients address the Service on port `80`.
- Traffic is forwarded to port `8080` on the selected Pod.
- For a NodePort Service, clients can also enter through a node's IP address on port `30080`.

`nodePort` is only applicable to Service types that support it, such as `NodePort` and `LoadBalancer` under the standard configuration.

### Service DNS

Kubernetes normally provides DNS records for Services through cluster DNS.

A Service can generally be addressed using:

```text
<service-name>.<namespace>.svc.cluster.local
```

For example, a Service named `backend` in the `default` namespace can be addressed as:

```text
backend.default.svc.cluster.local
```

Within the same namespace, a workload can often access it simply by using:

```text
backend
```

DNS resolution depends on the cluster's DNS configuration and the client's namespace and search domains.

## Key points

- **A Service provides a stable endpoint for changing Pods.** Clients do not need to track individual Pod IP addresses.
- **Selectors connect Services to Pods.** The Service's selector must match the intended Pod labels.
- **EndpointSlices describe backend endpoints.** If the Service has no eligible backends, traffic cannot be routed to the intended application Pods.
- **ClusterIP is the default type.** It is the normal choice for internal application-to-application communication.
- **NodePort exposes a node-level port.** It does not automatically make the application publicly accessible.
- **LoadBalancer requests an external load-balancing endpoint.** Successful provisioning depends on infrastructure support.
- **A Service is not a Pod.** It is a networking abstraction that directs traffic to backend endpoints.
- **A Service is not an Ingress.** An Ingress or Gateway can provide additional HTTP routing capabilities, such as host-based or path-based routing, when supported by the relevant controller.
- **A Service does not guarantee application health.** Readiness probes and endpoint management help prevent traffic from being sent to Pods that are not ready.
- **`port` and `targetPort` are different.** The Service port is the client-facing port; the target port is the backend destination.
- **NodePort is not the same as `targetPort`.** It is an additional node-level entry point.
- **Service DNS simplifies communication.** Applications can use stable names instead of hardcoded Pod IP addresses.
- **A selector mismatch can leave a Service without backends.** Always verify the labels and EndpointSlices when debugging.
- **A Service usually load-balances connections, not individual HTTP requests.** The actual traffic behavior depends on the protocol and networking implementation.
- **Services are commonly used with Deployments.** The Deployment manages application replicas, while the Service provides a stable way to reach them.
- **StatefulSets can also use Services.** A normal Service can distribute traffic across replicas, while a headless Service can support stable per-Pod DNS identities.

## Commands / manifests

### 1. Deploy a sample application

Create a Deployment for a simple NGINX web server.

Save this manifest as `web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
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

Apply it:

```bash
kubectl apply -f web-deployment.yaml
kubectl get deployments
kubectl get pods -l app=web -o wide
```

The Deployment attempts to maintain three replicas. The `containerPort` documents the port the container is expected to use; it does not expose the application outside the Pod by itself.

### 2. Create a ClusterIP Service

Save this manifest as `clusterip-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-internal
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

Apply it:

```bash
kubectl apply -f clusterip-service.yaml
kubectl get services
kubectl describe service web-internal
```

The Service selects Pods labeled `app: web` and exposes port `80` internally.

Test it from a temporary Pod:

```bash
kubectl run test-client \
  --image=curlimages/curl \
  --restart=Never \
  --rm -it \
  -- curl -sS http://web-internal
```

This command creates a temporary client Pod, sends an HTTP request to the Service, and removes the Pod when the command finishes. If the client image cannot be pulled or the cluster has no connectivity, inspect the resulting errors and Pod events.

### 3. Create a NodePort Service

Save this manifest as `nodeport-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
      protocol: TCP
```

Apply it:

```bash
kubectl apply -f nodeport-service.yaml
kubectl get service web-nodeport
kubectl describe service web-nodeport
```

If your cluster permits access to the node IP and port, the endpoint follows this pattern:

```text
http://<node-ip>:30080
```

The request succeeds only if the node is reachable, the NodePort is allowed through relevant firewalls, and the Service has working backend endpoints.

If the node IP is not reachable from your machine, you can still test the Service from inside the cluster using a temporary client Pod.

### 4. Create a LoadBalancer Service

Save this manifest as `loadbalancer-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-external
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

Apply it:

```bash
kubectl apply -f loadbalancer-service.yaml
kubectl get service web-external
kubectl describe service web-external
```

Inspect the `EXTERNAL-IP` or hostname:

```bash
kubectl get service web-external -w
```

In a supported cloud environment, the infrastructure integration may provision the external endpoint. In a local cluster or an environment without a load-balancer implementation, the external address may remain pending until the necessary support is configured.

### 5. Inspect Service endpoints

```bash
kubectl get endpoints web-internal
kubectl get endpointslices \
  -l kubernetes.io/service-name=web-internal
```

EndpointSlices are the modern resource used to represent Service backends. The older Endpoints resource may still be visible, but EndpointSlices are the preferred resource to inspect.

If the Service has no backend endpoints, check:

- Whether the Service selector matches the Pod labels.
- Whether the Pods exist in the same namespace.
- Whether the Pods are ready.
- Whether any additional Service configuration affects endpoint eligibility.

### 6. Inspect labels and selectors

List Pod labels:

```bash
kubectl get pods --show-labels
```

Inspect the Service selector:

```bash
kubectl get service web-internal -o yaml
```

If the Service selects `app: web`, the intended Pods must have the label `app=web`.

You can also list matching Pods directly:

```bash
kubectl get pods -l app=web
```

### 7. Test Service DNS

Run a temporary client:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -- nslookup web-internal
```

For a Service in another namespace, use its namespace-qualified name:

```bash
nslookup web-internal.<namespace>.svc.cluster.local
```

Replace `<namespace>` with the namespace where the Service exists. DNS troubleshooting should distinguish between a missing DNS record, an unreachable Service, and an application that is not listening on the expected port.

### 8. Port-forward a Service for local testing

```bash
kubectl port-forward service/web-internal 8080:80
```

Then access the application locally at:

```text
http://localhost:8080
```

The command forwards local port `8080` to Service port `80`. This is useful for development and debugging, but it is not a production exposure method.

### 9. Useful inspection commands

```bash
kubectl get services -A
kubectl describe service <service-name>
kubectl get endpointslices
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get events --sort-by=.metadata.creationTimestamp
```

Use these commands to investigate Service configuration, backend selection, DNS, and connectivity.

## Common failures & fixes

| Problem | Likely cause | Fix |
|---|---|---|
| Service has no backend endpoints | Selector does not match Pod labels, Pods are missing, or Pods are not eligible | Compare Service selectors with Pod labels and inspect EndpointSlices |
| Service DNS does not resolve | DNS configuration problem, incorrect Service name, or wrong namespace | Verify the Service exists and test its fully qualified DNS name |
| DNS resolves but requests fail | Backend port, routing, application, or network policy problem | Test connectivity and inspect the Service, Pods, and application logs |
| `Connection refused` | Nothing is listening on the target port, or the connection is actively rejected | Verify the container's listening port and the Service's `targetPort` |
| Requests time out | Network path, firewall, network policy, or backend health issue | Inspect routing, relevant policies, node reachability, and Pod readiness |
| NodePort is unreachable | Node IP is inaccessible or firewall rules block the assigned port | Check node reachability, security rules, the NodePort, and backend endpoints |
| LoadBalancer external address stays pending | The cluster lacks a working load-balancer integration or provisioning failed | Inspect Service events and verify cloud integration or load-balancer controller configuration |
| Wrong application responds | The Service selector matches unintended Pods | Use precise labels and verify the EndpointSlices |
| Only some requests succeed | One or more backend Pods are unhealthy or misconfigured | Inspect all backend Pods, readiness probes, and application logs |
| `targetPort` is wrong | The Service forwards traffic to a port the container does not serve | Align `targetPort` with the application's actual listening port |
| Service exposes the wrong port | `port`, `targetPort`, or `nodePort` was confused | Verify each port's purpose and correct the manifest |
| Service is not reachable across namespaces | Incorrect DNS name, namespace, or network policy | Use the namespace-qualified Service name and inspect policies |
| Port-forward works but external access fails | The application works through the API-server tunnel, but the external networking path is broken | Debug NodePort, LoadBalancer, firewall, and routing configuration separately |
| Service selects no ready Pods after a rollout | Deployment Pods are not ready or labels changed | Check Deployment status, readiness probes, and Service selectors |

### Practical debugging sequence

When a Service is unreachable:

1. Confirm the Service exists in the expected namespace.
2. Inspect its `port`, `targetPort`, type, and selector.
3. Check whether the selector matches the intended Pod labels.
4. Inspect EndpointSlices to confirm that backend endpoints exist.
5. Verify that the application is listening on the expected container port.
6. Test Service DNS and in-cluster connectivity.
7. For NodePort or LoadBalancer, inspect external routing, firewall rules, and infrastructure provisioning.
8. Review application logs, Pod events, and network policies.

Remember: a Service provides a stable way to reach backend endpoints, but it cannot fix an application that is not listening, a Pod that is not ready, or a network path that is blocked.

## References

- [Kubernetes documentation: Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes documentation: Service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
- [Kubernetes documentation: EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes documentation: DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes documentation: Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes documentation: Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
