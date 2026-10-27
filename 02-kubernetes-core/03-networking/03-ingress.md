# Ingress

> Phase 2 — Kubernetes Core (highest priority) › Networking

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Ingress is a Kubernetes API resource that defines rules for routing external HTTP and HTTPS traffic to Services inside a Kubernetes cluster.

It allows multiple applications to be exposed through a shared entry point, using hostnames and URL paths to direct requests to the correct backend Service.

For example, an application platform may expose:

- `app.example.com` to a frontend application.
- `api.example.com` to a backend API.
- `app.example.com/admin` to an administrative application.

Instead of creating a separate external LoadBalancer Service for every application, an Ingress can consolidate HTTP and HTTPS routing through a common entry point.

### How Ingress works

Ingress requires an **Ingress controller** to implement the routing rules.

The Ingress resource itself does not handle network traffic. It declares the desired routing configuration, and the controller watches that configuration and configures the actual traffic-routing infrastructure.

Common Ingress controllers include:

- NGINX Ingress Controller
- Traefik
- HAProxy Ingress
- Cloud-provider-specific Ingress controllers

A typical traffic flow looks like this:

```text
External Client
      |
      | HTTP / HTTPS
      v
External Load Balancer
      |
      v
Ingress Controller
      |
      | Evaluates hostname and path
      v
Kubernetes Service
      |
      | Routes to backend endpoint
      v
Application Pod
```

The exact flow depends on the controller and cluster setup. For example, the Ingress controller may itself be exposed through a LoadBalancer Service or a NodePort Service.

### Ingress resource vs Ingress controller

These are two different things.

| Component | Responsibility |
|---|---|
| Ingress resource | Defines the desired routing rules |
| Ingress controller | Implements those rules and routes incoming traffic |
| Kubernetes Service | Provides a stable endpoint for backend Pods |
| Backend Pod | Runs the application that handles the request |

An Ingress resource without a compatible controller does not automatically expose an application.

### Host-based routing

Host-based routing sends requests to different Services based on the hostname.

For example:

```text
app.example.com     -> frontend-service
api.example.com    -> api-service
admin.example.com  -> admin-service
```

This allows several applications to share an external IP address while using different hostnames.

### Path-based routing

Path-based routing directs requests according to the URL path.

For example:

```text
example.com/        -> frontend-service
example.com/api     -> api-service
example.com/admin   -> admin-service
```

The controller's behaviour determines how paths are matched and whether path rewriting is supported.

### HTTPS and TLS termination

Ingress can be configured to handle HTTPS traffic using TLS certificates.

In a common configuration:

1. The client connects to the Ingress endpoint using HTTPS.
2. The Ingress controller terminates TLS using a configured certificate.
3. The controller forwards the request to the backend Service.

The connection between the controller and backend may use HTTP or HTTPS, depending on the configuration.

Kubernetes commonly stores TLS certificate and private key material in a Secret of type `kubernetes.io/tls`.

### Ingress vs Service type LoadBalancer

| Feature | Ingress | Service type LoadBalancer |
|---|---|---|
| Primary purpose | HTTP/HTTPS routing | Expose a Service through a load balancer |
| Host-based routing | Supported by compatible controllers | Not provided by the Service itself |
| Path-based routing | Supported by compatible controllers | Not provided by the Service itself |
| TLS termination | Supported through controller configuration | Depends on external infrastructure |
| Multiple applications sharing an entry point | Common use case | Usually requires additional routing infrastructure |
| Layer | Application-level HTTP routing | Service exposure and network load balancing |

Ingress is commonly used alongside Services rather than replacing them.

### Ingress vs Gateway API

Ingress is the older, widely supported Kubernetes HTTP routing API.

Gateway API is a newer, more expressive set of Kubernetes APIs for managing traffic routing. It supports a broader range of traffic-management use cases and separates infrastructure ownership from application routing more explicitly.

Ingress remains useful for straightforward HTTP and HTTPS routing, while Gateway API is worth evaluating for more advanced traffic management and standardized gateway configuration.

## Key points

- Ingress defines rules for routing HTTP and HTTPS traffic to Kubernetes Services.
- An Ingress resource does not route traffic on its own. A compatible Ingress controller must be installed and running.
- Ingress is primarily designed for HTTP and HTTPS routing, not arbitrary TCP or UDP traffic.
- Host-based routing uses the request hostname to choose a backend.
- Path-based routing uses the request URL path to choose a backend.
- Ingress normally routes to Services, not directly to individual Pods.
- The Ingress controller is commonly exposed through a LoadBalancer or NodePort Service.
- Ingress can support TLS termination using certificates stored in Kubernetes Secrets.
- The Ingress API's behaviour depends on the controller implementation and its supported annotations or configuration.
- Ingress resources should specify an `ingressClassName` when the cluster uses an IngressClass to identify the intended controller.
- Ingress is not a replacement for a Kubernetes Service. Both resources usually work together.
- A configured Ingress rule does not guarantee that the hostname is publicly reachable. DNS, external load balancer configuration, firewall rules, and controller availability also matter.
- Kubernetes Ingress API supports HTTP and HTTPS routing. For broader traffic management, consider Gateway API.

## Commands / manifests

### 1. Inspect available Ingress resources

```bash
kubectl get ingress -A
```

List IngressClasses:

```bash
kubectl get ingressclass
```

IngressClasses identify the available Ingress controller implementations or configurations.

Inspect an Ingress resource:

```bash
kubectl describe ingress <ingress-name> -n <namespace>
```

View its YAML:

```bash
kubectl get ingress <ingress-name> -n <namespace> -o yaml
```

### 2. Check whether an Ingress controller is installed

List controller Pods and Services:

```bash
kubectl get pods -A
kubectl get services -A
```

Look for the controller deployed in your cluster, such as an NGINX Ingress controller.

Check its logs:

```bash
kubectl logs <controller-pod> -n <controller-namespace>
```

The controller's namespace and Pod names depend on the installation.

### 3. Create a sample application

Create a namespace:

```bash
kubectl create namespace ingress-lab
```

Deploy an NGINX application:

```bash
kubectl create deployment web \
  --image=nginx:stable \
  -n ingress-lab
```

Expose the Deployment through a ClusterIP Service:

```bash
kubectl expose deployment web \
  --name=web-service \
  --port=80 \
  --target-port=80 \
  -n ingress-lab
```

Verify the application and Service:

```bash
kubectl get deployments,pods,services -n ingress-lab
```

### 4. Create an Ingress manifest

The following manifest assumes that an Ingress controller is already installed and that an IngressClass named `nginx` exists in the cluster.

Save as `web-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f web-ingress.yaml
```

What the manifest does:

- `apiVersion`: Uses the stable Kubernetes Ingress API.
- `kind: Ingress`: Creates an Ingress routing resource.
- `ingressClassName`: Identifies the intended controller configuration.
- `host`: Matches the requested hostname.
- `path`: Matches requests whose URL path starts with `/`.
- `pathType: Prefix`: Specifies prefix-based path matching.
- `backend.service.name`: Identifies the Service receiving the request.
- `backend.service.port.number`: Identifies the Service port.

Inspect the resource:

```bash
kubectl get ingress -n ingress-lab
kubectl describe ingress web-ingress -n ingress-lab
```

### 5. Test host-based routing

If the controller is reachable through an IP address, test the hostname using curl:

```bash
curl -H "Host: web.example.com" http://<ingress-ip>/
```

This sends an HTTP request with the specified hostname without requiring public DNS to be configured.

If the Ingress controller is exposed through a LoadBalancer Service:

```bash
kubectl get service -A
```

Find the controller's external IP or hostname.

If the controller is reachable only through a NodePort, use the appropriate node address and exposed port.

### 6. Configure multiple paths

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

Requests to `/api` are routed to `api-service`, while requests matching `/` are routed to `web-service`.

The backend application may need to support the original path. Path rewriting, if required, depends on the controller and its configuration.

### 7. Configure HTTPS

First create a TLS Secret containing a certificate and private key:

```bash
kubectl create secret tls web-tls \
  --cert=web.example.com.crt \
  --key=web.example.com.key \
  -n ingress-lab
```

Then configure the Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - web.example.com
      secretName: web-tls
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

The controller must support and be configured to use the TLS settings. The certificate must be valid for the hostname clients use.

### 8. Clean up

```bash
kubectl delete namespace ingress-lab
```

This removes the sample Deployment, Service, Ingress, and other resources in the namespace.

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Ingress has no address | Controller is missing, not ready, or not exposing an address | Verify the controller Pods and its Service |
| Ingress resource exists but traffic does not reach the application | No controller is handling the resource | Check the IngressClass and controller configuration |
| `404 Not Found` | Host or path does not match a routing rule, or the controller's default backend returned 404 | Verify the request hostname and path against the Ingress rules |
| `503 Service Unavailable` | Controller cannot reach a valid backend | Check the Service, its endpoints, and application readiness |
| `502 Bad Gateway` | Controller reached the backend but received an invalid response or could not complete the connection | Inspect backend port, protocol, logs, and application health |
| `connection refused` | Controller or backend is not accepting traffic on the configured port | Verify the Service port, `targetPort`, and application listener |
| DNS name does not resolve | DNS record is missing or points to the wrong endpoint | Configure the hostname to resolve to the Ingress controller's external address |
| HTTPS certificate warning | Certificate is missing, expired, untrusted, or does not match the hostname | Check the TLS Secret and certificate details |
| HTTPS returns the wrong certificate | Hostname does not match the configured TLS host or controller certificate selection | Inspect the TLS configuration and requested hostname |
| Ingress rule appears ignored | Wrong IngressClass or controller-specific configuration | Check `ingressClassName`, controller logs, and supported annotations |
| `/api` returns 404 | Backend expects a different path, or path rewriting is required | Verify path matching and configure rewriting if supported |
| Works inside cluster but not from internet | External load balancer, DNS, firewall, or network routing is missing or blocked | Verify the controller's external exposure and network path |

### Troubleshooting sequence

Use this order to isolate the problem:

```bash
kubectl get ingress -n ingress-lab
kubectl describe ingress web-ingress -n ingress-lab
kubectl get ingressclass
kubectl get services -n ingress-lab
kubectl get endpointslices -n ingress-lab
kubectl get pods -n ingress-lab
```

Then inspect the Ingress controller:

```bash
kubectl get pods -A
kubectl logs <controller-pod> -n <controller-namespace>
```

Confirm each layer:

1. The Ingress resource has the intended hostname and path rules.
2. The Ingress controller is installed and recognizes the IngressClass.
3. The controller is reachable from the client.
4. The backend Service exists and exposes the correct port.
5. The Service has eligible endpoints.
6. The backend application is listening and healthy.
7. DNS and TLS configuration are correct for external access.

## References

- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Ingress API Reference](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/ingress-v1/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
