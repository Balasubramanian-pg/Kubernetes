# Service DNS

> Phase 2 — Kubernetes Core (highest priority) › Networking

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Kubernetes Service DNS allows Pods to discover and communicate with Services using DNS names instead of hardcoding Pod IP addresses.

Pod IPs can change when Pods are recreated, rescheduled, or scaled. A Kubernetes Service provides a stable network endpoint, while cluster DNS allows workloads to resolve that Service by name.

Kubernetes typically uses **CoreDNS** to provide DNS-based service discovery. The cluster DNS service receives queries from workloads and resolves names associated with Kubernetes Services and other configured DNS records.

For example, suppose an application consists of two components:

- `api`: A backend application exposed through a Kubernetes Service named `api-service`.
- `postgres`: A database exposed through a Service named `postgres-service`.

Instead of connecting to a database Pod's changing IP address, the API can connect using:

```text
postgres-service
```

If both Services are in the same namespace, this short name normally resolves to the corresponding Service IP.

### How Service DNS works

1. A Pod attempts to connect to a hostname such as `postgres-service`.
2. The container's DNS resolver sends a query to the DNS server configured in the Pod's `/etc/resolv.conf`.
3. The cluster DNS service, typically CoreDNS, looks up the Kubernetes Service DNS record.
4. DNS returns the Service's ClusterIP for a normal ClusterIP Service.
5. The application connects to that IP and the Service's networking rules route the traffic to an eligible backend Pod.

```text
API Pod
   |
   | Connects to postgres-service
   v
Pod DNS resolver
   |
   | DNS query
   v
CoreDNS
   |
   | Resolves Service name
   v
Service ClusterIP
   |
   | Service networking
   v
Database Pod
```

DNS resolution and traffic routing are separate operations. DNS resolves the hostname to an address; Kubernetes Service networking handles forwarding traffic to eligible endpoints.

### Kubernetes Service DNS naming convention

The fully qualified domain name (FQDN) for a Service generally follows this structure:

```text
<service-name>.<namespace>.svc.<cluster-domain>
```

For example:

```text
postgres-service.production.svc.cluster.local
```

Here:

- `postgres-service`: The Kubernetes Service name.
- `production`: The namespace containing the Service.
- `svc`: Identifies the Service DNS hierarchy.
- `cluster.local`: The common default cluster domain. A cluster can use a different domain.

The following names illustrate how DNS names depend on namespace:

| DNS name | Typical use |
|---|---|
| `postgres-service` | Service in the same namespace as the requesting Pod |
| `postgres-service.production` | Service in the `production` namespace |
| `postgres-service.production.svc.cluster.local` | Fully qualified Service DNS name |

The short name `postgres-service` is resolved using the search domains configured for the requesting Pod. It does not automatically mean that Kubernetes searches every namespace.

### Service DNS versus Pod IP addresses

| Approach | Behaviour | Recommendation |
|---|---|---|
| Hardcoded Pod IP | Can become invalid when the Pod is replaced | Avoid for normal service discovery |
| Service DNS name | Stable name for reaching a Service | Preferred for application-to-application communication |
| Service ClusterIP | Stable virtual IP for the Service's lifetime | Useful when an IP is specifically required |
| External DNS name | Resolves according to external or configured DNS records | Used to reach external services |

### DNS for different Service types

**ClusterIP Service**

A normal ClusterIP Service gets a DNS record that resolves to its ClusterIP. Connections to that IP are forwarded to eligible Service endpoints.

**Headless Service**

A headless Service is configured with `clusterIP: None`. Instead of resolving to a virtual ClusterIP, its DNS records normally return the IP addresses of its eligible endpoints.

Headless Services are commonly used with StatefulSets when clients need to discover individual Pods.

For example:

```text
database-headless.default.svc.cluster.local
```

Depending on the Service configuration and endpoint readiness, the DNS response can contain multiple Pod IP addresses.

**ExternalName Service**

An ExternalName Service returns a DNS CNAME pointing to the configured external hostname. It does not proxy network traffic or allocate a ClusterIP in the same way as a normal ClusterIP Service.

### Important distinction: DNS does not guarantee connectivity

Successfully resolving a Service name does not prove that the application is reachable. A connection can still fail because:

- The Service has no eligible endpoints.
- The Service's `port` or `targetPort` is incorrect.
- The application is not listening on the expected port.
- A NetworkPolicy blocks the traffic.
- The destination application is unhealthy.
- The Service selector does not match the intended Pods.

Troubleshoot DNS resolution first, then verify Service endpoints, port configuration, and network connectivity.

## Key points

- Kubernetes Service DNS enables name-based discovery instead of relying on changing Pod IP addresses.
- CoreDNS is the default DNS implementation in many Kubernetes clusters, but the implementation can vary.
- A Service's fully qualified DNS name follows the pattern `<service>.<namespace>.svc.<cluster-domain>`.
- The default cluster domain is commonly `cluster.local`, but it is configurable.
- Short Service names generally work within the same namespace because of the Pod's DNS search domains.
- Use a namespace-qualified name when connecting to a Service in another namespace.
- A normal ClusterIP Service resolves to its ClusterIP, not directly to an individual backend Pod.
- A headless Service resolves to endpoint addresses rather than a virtual ClusterIP.
- StatefulSets can use headless Services to support stable per-Pod DNS identities.
- DNS resolution and traffic routing are separate. A successful lookup does not guarantee a successful connection.
- Kubernetes DNS records are managed dynamically as Services and their endpoints change.
- A Service name is usually the right choice for application configuration because it remains stable when backend Pods are replaced.

## Commands / manifests

### 1. Create a namespace and a test application

Create a namespace for the exercise:

```bash
kubectl create namespace dns-lab
```

Deploy an NGINX application and expose it through a Service:

```bash
kubectl create deployment web --image=nginx:stable -n dns-lab

kubectl expose deployment web \
  --name=web-service \
  --port=80 \
  --target-port=80 \
  -n dns-lab
```

What these commands do:

- `kubectl create deployment`: Creates a Deployment that manages the NGINX Pods.
- `kubectl expose deployment`: Creates a Service selecting the Pods belonging to the Deployment.
- `--port=80`: Sets the port exposed by the Service.
- `--target-port=80`: Sets the port on which the application container receives traffic.
- `-n dns-lab`: Runs the command in the `dns-lab` namespace.

Verify the resources:

```bash
kubectl get deployments,pods,services -n dns-lab
kubectl get endpointslices -n dns-lab
```

EndpointSlices show the backend addresses associated with the Service.

### 2. Test DNS from inside a Pod

Create a temporary Pod with DNS diagnostic tools:

```bash
kubectl run dns-client \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600 \
  -n dns-lab
```

Wait until the Pod is ready:

```bash
kubectl wait \
  --for=condition=Ready pod/dns-client \
  --timeout=60s \
  -n dns-lab
```

Run a DNS lookup from inside the Pod:

```bash
kubectl exec -it dns-client -n dns-lab -- nslookup web-service
```

Because the client and Service are in the same namespace, the short name `web-service` should resolve to the Service's ClusterIP.

Try the namespace-qualified name:

```bash
kubectl exec -it dns-client -n dns-lab -- \
  nslookup web-service.dns-lab
```

Then query the fully qualified name:

```bash
kubectl exec -it dns-client -n dns-lab -- \
  nslookup web-service.dns-lab.svc.cluster.local
```

The final command assumes the cluster domain is `cluster.local`. If your cluster uses a different domain, substitute the configured domain.

### 3. Test HTTP connectivity using the Service name

```bash
kubectl exec dns-client -n dns-lab -- \
  wget -qO- http://web-service
```

Expected result: HTML returned by the NGINX welcome page.

This verifies more than DNS resolution alone: it also tests HTTP connectivity through the Service.

You can test the Service using its full DNS name as well:

```bash
kubectl exec dns-client -n dns-lab -- \
  wget -qO- http://web-service.dns-lab.svc.cluster.local
```

### 4. Discover a Service in another namespace

Create a second namespace:

```bash
kubectl create namespace app-lab
```

A Pod in `app-lab` cannot normally use the short name `web-service` to discover the Service in `dns-lab`, because its DNS search domains are based on its own namespace.

Use the namespace-qualified name instead:

```text
web-service.dns-lab
```

Or use the fully qualified name:

```text
web-service.dns-lab.svc.cluster.local
```

You can test the cross-namespace lookup by creating a client Pod in `app-lab` and running `nslookup` against the fully qualified name. Actual connectivity can still be restricted by NetworkPolicies or other network controls.

### 5. Inspect a Pod's DNS configuration

```bash
kubectl exec dns-client -n dns-lab -- cat /etc/resolv.conf
```

A typical configuration contains:

- A `nameserver` entry pointing to the cluster DNS Service IP.
- Search domains for the Pod's namespace and cluster DNS hierarchy.
- An `options ndots` setting that affects how names are tried with search domains.

The exact output depends on the cluster's DNS configuration and Pod settings.

To inspect the DNS configuration of a Pod through the Kubernetes API:

```bash
kubectl get pod dns-client -n dns-lab -o yaml
```

Look for the `dnsPolicy` and `dnsConfig` fields.

Common `dnsPolicy` values include:

- `ClusterFirst`: Uses cluster DNS for normal workloads and forwards non-cluster queries according to DNS configuration.
- `Default`: Inherits DNS configuration from the node.
- `None`: Uses explicitly specified Pod DNS settings.
- `ClusterFirstWithHostNet`: Commonly used for Pods running with `hostNetwork: true` that still need cluster DNS behaviour.

### 6. Inspect Services and their endpoints

```bash
kubectl get services -n dns-lab
kubectl describe service web-service -n dns-lab
kubectl get endpointslices -n dns-lab
kubectl get pods -n dns-lab --show-labels
```

These commands help determine whether the Service exists, whether its selector matches the intended Pods, and whether eligible endpoints are available.

### 7. Example Service manifest

The following manifest explicitly defines a ClusterIP Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: dns-lab
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

Save it as `web-service.yaml` and apply it:

```bash
kubectl apply -f web-service.yaml
```

The selector must match the labels on the actual Pods. Verify them with:

```bash
kubectl get pods -n dns-lab --show-labels
```

### 8. Clean up the exercise

```bash
kubectl delete namespace dns-lab
kubectl delete namespace app-lab
```

Deleting these namespaces removes the test resources created inside them.

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `nslookup: can't resolve 'web-service'` | Service name is wrong, the Service is missing, or DNS is failing | Run `kubectl get svc -n dns-lab`, verify the name, then inspect the Pod's DNS configuration |
| Short name works in one namespace but not another | The name is being resolved relative to the client's namespace | Use `web-service.dns-lab` or the full Service DNS name |
| DNS lookup returns an IP, but HTTP fails | DNS works, but the Service or application may be unreachable | Inspect the Service, EndpointSlices, port configuration, and NetworkPolicies |
| Service has no endpoints | Selector does not match Pod labels, or no eligible Pods are available | Compare `kubectl describe svc web-service -n dns-lab` with `kubectl get pods --show-labels -n dns-lab` |
| `connection refused` | The destination is reachable but nothing is accepting the connection on that port, or the connection is actively rejected | Check `targetPort`, application listener configuration, and Pod logs |
| `connection timed out` | Traffic may be blocked or the destination may not be responding | Check NetworkPolicies, CNI networking, Pod readiness, and application health |
| All Service lookups fail across the cluster | CoreDNS may be unhealthy or cluster DNS networking may be broken | Check CoreDNS Pods, logs, and the DNS Service |
| External names fail, but Service names work | Upstream DNS forwarding or external DNS access may be broken | Inspect CoreDNS configuration, upstream DNS, and network egress |
| A headless Service does not return the expected Pod IPs | Endpoints may be missing, or readiness and Service settings may affect published records | Inspect EndpointSlices and review the Service's `publishNotReadyAddresses` and selector settings |
| Fully qualified name fails with `cluster.local` | The cluster uses a different cluster domain, or the name is malformed | Inspect DNS configuration and use the actual cluster domain |

### Diagnose CoreDNS

Check the DNS Service and Pods:

```bash
kubectl get service -n kube-system
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

The DNS Service is commonly named `kube-dns`, even when CoreDNS provides the implementation.

Inspect logs:

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

If the selector does not match the cluster's DNS Pods, list the available Pods and inspect their labels:

```bash
kubectl get pods -n kube-system --show-labels
```

Also verify that the client Pod can reach the DNS Service IP and that cluster networking permits DNS traffic, typically UDP and TCP port 53.

Do not restart or modify CoreDNS immediately in a production cluster. First establish whether the failure is isolated to one Pod, one namespace, one DNS record, or the entire cluster.

## References

- [Kubernetes DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- [DNS for StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#stable-network-id)
- [Pod DNS Configuration](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)
