# Container networking

> Phase 1 — Linux, Docker & Networking › Networking

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Container networking defines how containers communicate with one another, the host machine, and external systems.

Containers have isolated network environments, including network interfaces, IP addresses, routing tables, and ports. Container networking connects these environments through mechanisms such as virtual Ethernet interfaces, bridges, routing, network address translation (NAT), and overlay networks.

For example, consider a web application that uses a database:

```text
                 External client
                       |
                       v
                 Host port 8080
                       |
                 Docker networking
                       |
                       v
              +------------------+
              | Web container    |
              | Port 80           |
              +------------------+
                       |
                Docker bridge
                       |
                       v
              +------------------+
              | Database         |
              | Port 5432         |
              +------------------+
```

The client connects to the published host port, while the web container connects directly to the database container over their shared Docker network.

The database does not need a published host port if only other containers on the internal network need to access it.

Docker provides several network drivers, each designed for a different connectivity requirement.

- **Bridge:** Connects containers through a virtual network on a single Docker host. This is a common choice for applications running multiple containers on one machine.
- **Host:** Removes the usual container network isolation by sharing the host's network namespace on supported Linux configurations. The container uses the host's network interfaces and ports directly.
- **Overlay:** Connects containers across multiple Docker hosts through an overlay network. Docker's overlay driver is commonly used with Docker Swarm services.
- **None:** Disables normal container networking, leaving the container without a regular external network connection.
- **Macvlan:** Gives containers addresses on a physical network, making them appear as separate devices on that network when the surrounding infrastructure supports it.

These modes are not interchangeable. The correct choice depends on whether the application needs local container communication, direct host networking, cross-host connectivity, or integration with an existing network.

## Key points

- **Bridge networking**

  Bridge networking connects containers through a virtual Layer 2 network on a Docker host.

  Docker's default bridge network is created automatically. Containers attached to it can generally communicate using IP addresses, subject to network and firewall configuration. Automatic container-name resolution is a key difference between the default bridge and a user-defined bridge network.

  A user-defined bridge is generally preferable for multi-container applications because it provides container-name resolution and better network isolation.

  Example:

  ```text
  Host
   |
   +-- Docker bridge: app-net
         |
         +-- web: 172.x.x.2
         |
         +-- db:  172.x.x.3
  ```

  The IP addresses are illustrative. Actual addresses depend on the network configuration.

  Key characteristics:

  - Designed for containers on the same Docker host.
  - Provides network isolation between separate bridge networks.
  - Supports container-to-container communication on the same network.
  - Supports container-name resolution on user-defined bridge networks.
  - Can publish container ports to the host using `-p`.

- **Host networking**

  Host networking allows a container to use the host's network namespace rather than receiving a separate network namespace with its own container IP.

  Example:

  ```bash
  docker run --network host nginx
  ```

  On supported Linux systems, NGINX uses the host's networking directly. If it listens on port `80`, that port is the host's port `80`.

  Key characteristics:

  - Avoids the usual separate container network namespace.
  - Does not require normal `-p` port publishing because the container shares the host's network namespace.
  - Can be useful for workloads that need direct host-network access or have specific networking performance requirements.
  - Reduces network isolation and increases the risk of host-port conflicts.
  - Behaves differently on Docker Desktop and other environments where the Docker daemon runs inside a virtual machine.

  With host networking, `-p` is generally ignored on supported Linux Docker configurations because the container does not have a separate network namespace to which Docker can forward published ports.

- **Overlay networking**

  Overlay networking connects containers running on different Docker hosts through a virtual network that spans those hosts.

  Example:

  ```text
  Docker host A                     Docker host B
  +----------------+                +----------------+
  | Web container  |                | API container  |
  |                |                |                |
  +-------+--------+                +--------+-------+
          |                                  |
          +----------- Overlay --------------+
                     network
  ```

  Key characteristics:

  - Enables communication across multiple Docker hosts.
  - Commonly used with Docker Swarm services.
  - Encapsulates traffic between participating hosts.
  - Requires appropriate host connectivity, firewall rules, and overlay configuration.
  - Is different from Kubernetes networking, which is provided through the cluster's container network interface (CNI) and related networking components.

  Docker Swarm overlay networks may require specific host-to-host connectivity, including TCP port `2377` for cluster management, TCP and UDP port `7946` for node communication, and UDP port `4789` for the default VXLAN overlay data path. Actual requirements depend on the configuration.

- **Network isolation**

  Docker networks provide logical separation between groups of containers. Containers on different user-defined bridge networks cannot generally communicate directly unless an appropriate connection or routing path exists.

  Network isolation is not a complete security boundary. Host access, published ports, firewall rules, privileged containers, and other configuration can affect the actual security posture.

- **Container IP addresses are not stable identifiers**

  A container can receive a different IP address when it is recreated or reattached to a network.

  Applications should generally use service names or container names on user-defined Docker networks instead of hardcoding container IP addresses.

  For example, an application can connect to a database using:

  ```text
  db:5432
  ```

  Here, `db` is the database container's name or network alias, and `5432` is the database's internal listening port.

- **Container-to-container communication does not normally require published ports**

  Containers on the same user-defined bridge network can communicate through their internal ports.

  For example:

  ```text
  Web container  --->  db:5432  --->  Database container
  ```

  The database does not need `-p 5432:5432` for this connection. Publishing the database port is only necessary when clients outside the relevant Docker network need access through the host.

- **Published ports and container ports are different**

  ```bash
  docker run -d -p 8080:80 nginx
  ```

  This maps host port `8080` to container port `80`.

  Port publishing makes the application reachable through the host's published address and port, subject to firewall and network configuration. It is not required for every container-to-container connection.

- **DNS is part of container networking**

  Docker provides embedded DNS on user-defined networks. Containers can resolve connected container names and network aliases without hardcoding their IP addresses.

  This simplifies service discovery and makes applications more resilient to container recreation.

- **Networking depends on the platform**

  Docker Engine on Linux, Docker Desktop on macOS or Windows, and managed container environments may implement networking differently.

  In particular, host networking, routing, published ports, and access to host interfaces can behave differently across platforms.

- **Docker networking versus Kubernetes networking**

  Docker networking primarily concerns Docker containers and Docker network drivers. Kubernetes networking adds cluster-level concepts such as Pods, Services, and NetworkPolicies.

  Kubernetes commonly uses a CNI-based networking implementation to provide Pod connectivity. A Kubernetes Service provides a stable access point for a group of Pods, even when individual Pod IP addresses change.

  The concepts are related, but Docker bridge and overlay networking should not be treated as direct substitutes for every Kubernetes networking feature.

## Commands / manifests

**1. List Docker networks**

```bash
docker network ls
```

This lists networks known to the Docker daemon, including the default `bridge`, `host`, and `none` networks where applicable.

Inspect a particular network:

```bash
docker network inspect bridge
```

The output includes information such as the driver, subnet configuration, connected containers, and network options.

**2. Create a user-defined bridge network**

```bash
docker network create app-net
```

Confirm that the network exists:

```bash
docker network ls
docker network inspect app-net
```

A user-defined bridge provides a dedicated network for related containers.

**3. Connect two containers using a bridge network**

Start a web container:

```bash
docker run -d \
  --name web \
  --network app-net \
  -p 8080:80 \
  nginx:alpine
```

Start a diagnostic container on the same network:

```bash
docker run --rm \
  --network app-net \
  busybox \
  wget -qO- http://web
```

The diagnostic container resolves `web` through Docker's embedded DNS and sends an HTTP request to port `80`.

The host can access NGINX at `http://localhost:8080`. The diagnostic container does not need to use that published host port because both containers share `app-net`.

Inspect the network:

```bash
docker network inspect app-net
```

**4. Demonstrate network isolation**

Create a second bridge network:

```bash
docker network create isolated-net
```

Start a container on the second network:

```bash
docker run -d \
  --name isolated-web \
  --network isolated-net \
  nginx:alpine
```

Try resolving it from the first network:

```bash
docker run --rm \
  --network app-net \
  busybox \
  nslookup isolated-web
```

The name should not resolve through Docker's embedded DNS on `app-net` because `isolated-web` is connected only to `isolated-net`.

This demonstrates network separation, not an absolute security guarantee. Host access, additional network attachments, and routing configuration can change connectivity.

Clean up:

```bash
docker rm -f web isolated-web
docker network rm app-net isolated-net
```

**5. Use host networking**

On a supported Linux Docker Engine:

```bash
docker run -d \
  --name host-web \
  --network host \
  nginx:alpine
```

Check the logs:

```bash
docker logs host-web
```

If NGINX listens on port `80`, access it through the host's port `80`, assuming no conflict and the relevant interface is reachable.

Inspect the container's networking configuration:

```bash
docker inspect host-web
```

Unlike bridge networking, host networking does not normally require `-p 8080:80`. Verify platform support before relying on this behavior.

Clean up:

```bash
docker rm -f host-web
```

**6. Use an overlay network with Docker Swarm**

Initialize Swarm on a suitable test host:

```bash
docker swarm init
```

Create an attachable overlay network:

```bash
docker network create \
  --driver overlay \
  --attachable \
  app-overlay
```

Run a test service:

```bash
docker service create \
  --name web \
  --network app-overlay \
  --replicas 2 \
  nginx:alpine
```

Inspect the network and service:

```bash
docker network inspect app-overlay
docker service ls
docker service ps web
```

An overlay network is useful for cross-host communication in Swarm. To test actual cross-host connectivity, deploy the service across participating Swarm nodes and ensure the required network ports and routing are configured.

Clean up:

```bash
docker service rm web
docker network rm app-overlay
docker swarm leave --force
```

Run the cleanup command only on a disposable test node where leaving Swarm is appropriate.

**7. Inspect a container's network configuration**

```bash
docker inspect web
```

To display only its network information:

```bash
docker inspect \
  --format '{{json .NetworkSettings.Networks}}' \
  web
```

To inspect the container's IP configuration from inside a Linux container:

```bash
docker exec web ip addr
docker exec web ip route
```

These commands require the relevant utilities to exist in the container. Minimal images may not include `ip`; use a diagnostic container or an available alternative when necessary.

**8. Check listening ports and connectivity**

On a Linux host:

```bash
ss -lnt
```

This lists listening TCP sockets. Depending on the Docker networking implementation, published ports may be implemented through forwarding rules without a conventional userspace listener appearing in this output.

Test an HTTP service:

```bash
curl -v http://localhost:8080
```

Check the container's logs:

```bash
docker logs web
```

These checks help distinguish port-publishing problems from application failures.

**9. Define container networking with Docker Compose**

Example `compose.yaml`:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    networks:
      - app-net

  diagnostic:
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

Start the services:

```bash
docker compose up -d
```

Test container-to-container DNS and HTTP connectivity:

```bash
docker compose exec diagnostic wget -qO- http://web
```

The Compose network lets the `diagnostic` service resolve the `web` service by name. Only the `web` service publishes a host port.

Inspect the network:

```bash
docker compose ps
docker network ls
```

Clean up:

```bash
docker compose down
```

**10. Understand Kubernetes networking conceptually**

A Kubernetes Pod receives its own IP address through the cluster networking implementation. Containers within the same Pod share the Pod's network namespace and can communicate through `localhost`.

A Kubernetes Service provides a stable network endpoint for a set of matching Pods.

Example Service manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP
```

In this example:

- `port: 80` is the port exposed by the Service.
- `targetPort: 8080` is the port targeted on the selected Pods.
- `type: ClusterIP` makes the Service available through a cluster-internal IP by default.

This Service requires matching Pods labelled `app: web` with an application listening on port `8080`. The manifest alone does not create those Pods.

## Common failures & fixes

- **Problem: Containers cannot communicate using their names.**
  - **Cause:** They may be connected to different networks, or they may be using the default bridge where automatic name resolution is not provided in the same way as on a user-defined bridge.
  - **Fix:** Create a user-defined bridge network and connect both containers to it. Use the container name or network alias as the hostname.

- **Problem: A container can communicate with another container but cannot be reached from the host.**
  - **Cause:** Container-to-container connectivity does not automatically publish a port on the host.
  - **Fix:** Publish the required port using `-p HOST_PORT:CONTAINER_PORT` and verify that the application listens on the container port.

- **Problem: A published port is reachable locally but not from another machine.**
  - **Cause:** The port may be bound to loopback, or a firewall, security group, routing rule, or host-network restriction may block external traffic.
  - **Fix:** Inspect the published address, firewall rules, and routing. Expose the port on an appropriate interface only if remote access is intended.

- **Problem: Two containers cannot use the same host port.**
  - **Cause:** Their published host-port bindings conflict.
  - **Fix:** Use different host ports or place a reverse proxy in front of the applications. Containers can still use the same internal port when they have separate network endpoints.

- **Problem: Host networking does not behave as expected on macOS or Windows.**
  - **Cause:** Docker Desktop uses a different networking architecture from a native Linux Docker Engine, and host networking support depends on the environment and configuration.
  - **Fix:** Check the platform-specific Docker documentation. Use bridge networking with published ports when that better matches the intended workflow.

- **Problem: An overlay network does not provide cross-host connectivity.**
  - **Cause:** The participating nodes may not have the required connectivity, firewall ports may be blocked, or the network may not be configured correctly.
  - **Fix:** Verify Swarm membership, overlay network attachment, node reachability, and the required firewall rules. Confirm that the containers are deployed on participating hosts.

- **Problem: A container cannot reach the internet.**
  - **Cause:** DNS, host routing, NAT, firewall rules, proxy settings, or upstream network restrictions may prevent outbound connectivity.
  - **Fix:** Test DNS resolution separately from IP connectivity. Inspect the container's routes and resolver configuration, then check the host's networking and firewall setup.

- **Problem: A database is unexpectedly accessible from outside the host.**
  - **Cause:** Its port may have been published on all host interfaces.
  - **Fix:** Avoid publishing the database port if only application containers need access. Otherwise, bind it to an appropriate interface and apply firewall and authentication controls.

- **Problem: A Kubernetes Service has no working backend.**
  - **Cause:** The Service selector may not match any Pods, the Pods may not be ready, or the application may not be listening on the target port.
  - **Fix:** Inspect the Service, Pods, and endpoints or EndpointSlices. Confirm the labels, readiness status, target port, and application health.

- **Problem: A container's IP address changes after recreation.**
  - **Cause:** Container IP addresses are assigned through network configuration and are not stable service identifiers.
  - **Fix:** Use container names or network aliases on Docker user-defined networks. In Kubernetes, use Services instead of relying on Pod IP addresses.

## References

- [Docker documentation: Networking](https://docs.docker.com/engine/network/)
- [Docker documentation: Bridge network driver](https://docs.docker.com/engine/network/drivers/bridge/)
- [Docker documentation: Host network driver](https://docs.docker.com/engine/network/drivers/host/)
- [Docker documentation: Overlay network driver](https://docs.docker.com/engine/network/drivers/overlay/)
- [Docker documentation: Compose networking](https://docs.docker.com/compose/how-tos/networking/)
- [Kubernetes documentation: Cluster networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Kubernetes documentation: Services](https://kubernetes.io/docs/concepts/services-networking/service/)
