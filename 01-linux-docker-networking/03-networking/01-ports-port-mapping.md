# Ports & port mapping

> Phase 1 — Linux, Docker & Networking › Networking

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Port mapping connects a port on the Docker host to a port inside a container, allowing clients to reach an application running inside that container.

The syntax is:

```bash
-p HOST_PORT:CONTAINER_PORT
```

For example:

```bash
docker run -d -p 8080:80 nginx
```

This maps port `8080` on the Docker host to port `80` inside the NGINX container.

When a client accesses `http://localhost:8080`, Docker forwards the incoming connection to port `80` in the container, where NGINX listens for HTTP requests.

**Example:**

<text>Client → Host port 8080 → Docker port forwarding → Container port 80 → NGINX</text>

There are two different ports involved:

- **Host port:** The port exposed on the machine running Docker. Clients connect to this port.
- **Container port:** The port on which the application listens inside the container's network namespace.

These port numbers do not have to be identical. For example, host port `3000` can forward traffic to container port `80`.

Port mapping is especially important when running multiple containers that expose the same internal port. Each container can use port `80` internally while using a different available host port.

## Key points

- **Port mapping syntax**

  ```bash
  docker run -p HOST_PORT:CONTAINER_PORT IMAGE
  ```

  Example:

  ```bash
  docker run -d -p 8080:80 nginx
  ```

  The host receives connections on port `8080` and forwards them to port `80` in the container.

- **Host ports and container ports are independent**

  | Host port | Container port | Typical use |
  |---:|---:|---|
  | 8080 | 80 | HTTP application |
  | 8443 | 443 | HTTPS application |
  | 3000 | 3000 | Node.js development server |
  | 5433 | 5432 | PostgreSQL database |
  | 6380 | 6379 | Redis instance |

  These are examples, not fixed assignments. The container port must match the port on which the application actually listens.

- **Port mapping is not the same as exposing a port**

  - `EXPOSE 80` in a Dockerfile documents that the application uses port `80`. It does not publish the port to the host.
  - `-p 8080:80` publishes a host port and configures forwarding to the container port.
  - `-P` publishes ports declared through `EXPOSE` to automatically assigned host ports.

- **Publishing a port does not make an application listen on it**

  If an application listens only on `127.0.0.1` inside its container, Docker's forwarding may not reach it through the container's network interface.

  For most containerized web applications, configure the application to listen on `0.0.0.0` inside the container.

- **Multiple containers can use the same container port**

  For example:

  ```bash
  docker run -d --name web1 -p 8081:80 nginx
  docker run -d --name web2 -p 8082:80 nginx
  ```

  Both containers use port `80` internally, but clients access them through different host ports.

- **Host port conflicts**

  Two containers generally cannot publish the same host IP and port combination simultaneously when the existing binding conflicts.

  For example, if one container already uses host port `8080`, another container cannot normally publish the same host port and address.

- **Binding to a specific host interface**

  By default, published ports are generally exposed on all host interfaces, subject to Docker's networking implementation and firewall configuration.

  To bind a published port only to the host's IPv4 loopback interface:

  ```bash
  docker run -d -p 127.0.0.1:8080:80 nginx
  ```

  This restricts access through that published binding to the host itself. It is useful for local development services that should not be directly reachable from other machines.

- **Container-to-container communication**

  Containers attached to the same user-defined Docker network can generally communicate using their container names and internal listening ports. They do not usually need published host ports for this.

  For example, an application container can connect to a PostgreSQL container at `db:5432` when both containers share a suitable Docker network and `db` resolves to the database container.

- **Port mapping is not a complete security control**

  Publishing a port can make a service reachable beyond the local machine, depending on the host interface, firewall rules, routing, and Docker configuration. Use appropriate network restrictions, authentication, and firewall policies for sensitive services.

## Commands / manifests

**1. Run a container with port mapping**

```bash
docker run -d --name web -p 8080:80 nginx
```

Explanation:

- `-d`: Runs the container in the background.
- `--name web`: Assigns the container the name `web`.
- `-p 8080:80`: Maps host port `8080` to container port `80`.
- `nginx`: Specifies the image.

Access the application at:

```text
http://localhost:8080
```

**2. Verify published ports**

```bash
docker ps
```

Example output:

```text
CONTAINER ID   IMAGE   PORTS                  NAMES
abc123         nginx   0.0.0.0:8080->80/tcp   web
```

The `0.0.0.0:8080->80/tcp` entry indicates that host port `8080` is published on the host's IPv4 interfaces and forwarded to container port `80`.

Inspect a specific container:

```bash
docker port web
```

Example output:

```text
80/tcp -> 0.0.0.0:8080
```

Inspect the container's complete configuration:

```bash
docker inspect web
```

**3. Test the application**

```bash
curl -I http://localhost:8080
```

The `-I` option requests response headers. A functioning NGINX server commonly returns an HTTP response such as `200 OK`.

You can also open `http://localhost:8080` in a browser on the Docker host.

**4. Map a different host port**

```bash
docker run -d --name web-alt -p 8081:80 nginx
```

Access it at:

```text
http://localhost:8081
```

Both containers can run simultaneously because they use different host ports.

**5. Publish a port only on localhost**

```bash
docker run -d --name local-web -p 127.0.0.1:8082:80 nginx
```

Test it from the host:

```bash
curl -I http://127.0.0.1:8082
```

This is useful when a service should be accessed only locally through that host binding.

**6. Let Docker choose a host port**

```bash
docker run -d --name dynamic-web -p 80 nginx
```

Here, Docker publishes container port `80` on an automatically assigned host port.

Find the assigned port:

```bash
docker port dynamic-web
```

Example output:

```text
80/tcp -> 0.0.0.0:32768
```

The exact host port is assigned at runtime and may differ between runs.

**7. Publish multiple ports**

```bash
docker run -d \
  --name multi-port-app \
  -p 8080:80 \
  -p 8443:443 \
  nginx
```

This publishes host port `8080` to container port `80`, and host port `8443` to container port `443`.

The application must actually listen on the corresponding container ports for these mappings to be useful.

**8. Run two containers using the same internal port**

```bash
docker run -d --name web1 -p 8081:80 nginx
docker run -d --name web2 -p 8082:80 nginx
```

Test both:

```bash
curl -I http://localhost:8081
curl -I http://localhost:8082
```

Both containers run NGINX on port `80` internally, while the host uses separate ports.

Clean up the containers:

```bash
docker rm -f web1 web2
```

**9. Check which process is listening on a host port**

On Linux:

```bash
sudo ss -lntp
```

To inspect TCP listeners associated with port `8080`:

```bash
sudo ss -lntp 'sport = :8080'
```

Depending on the Docker networking implementation, a published port may be handled by Docker's networking and packet-forwarding mechanisms without a conventional userspace process appearing as the listener. Therefore, also inspect Docker's published-port configuration:

```bash
docker ps
docker port web
```

**10. Understand `EXPOSE` versus `-p`**

Dockerfile:

```dockerfile
FROM nginx:alpine
EXPOSE 80
```

Build and run:

```bash
docker build -t nginx-demo .
docker run -d --name nginx-demo nginx-demo
```

The Dockerfile's `EXPOSE 80` does not publish port `80` on the host. To publish it:

```bash
docker rm -f nginx-demo
docker run -d --name nginx-demo -p 8080:80 nginx-demo
```

Now the application is accessible through `http://localhost:8080`, assuming the application is listening correctly and the host networking configuration permits access.

## Common failures & fixes

- **Problem: `Bind for 0.0.0.0:8080 failed: port is already allocated`.**
  - **Cause:** Another container or process is already using the requested host port and address.
  - **Fix:** Choose another host port, such as `8081:80`, or identify and stop the conflicting service if appropriate.

- **Problem: The container is running, but `localhost:8080` does not respond.**
  - **Cause:** The container may not be listening, the application may have crashed, the wrong port may be mapped, or a firewall or network rule may block traffic.
  - **Fix:** Check `docker ps`, inspect logs with `docker logs web`, verify the mapping using `docker port web`, and test the application from inside the container if necessary.

- **Problem: Docker reports a successful port mapping, but connections are refused.**
  - **Cause:** The application may be listening on the wrong port or bound only to the container's loopback interface.
  - **Fix:** Check the application configuration and listening sockets inside the container. For a web server, configure it to listen on `0.0.0.0` and confirm that its port matches the container port in `-p`.

- **Problem: The application works inside the container but cannot be reached from another machine.**
  - **Cause:** The port may be published only on `127.0.0.1`, or the host firewall, cloud security group, routing, or network policy may block external connections.
  - **Fix:** Inspect the published address with `docker port`, verify firewall and cloud network rules, and publish on an appropriate host interface if external access is intended. Do not expose sensitive services unnecessarily.

- **Problem: `EXPOSE 80` does not make the application accessible through the host.**
  - **Cause:** `EXPOSE` documents the container port but does not publish it.
  - **Fix:** Run the container with `-p 8080:80` or use `-P` to publish declared ports on automatically assigned host ports.

- **Problem: Two containers cannot both publish host port `8080`.**
  - **Cause:** Their published bindings conflict.
  - **Fix:** Use different host ports, such as `8081:80` and `8082:80`. Alternatively, use a reverse proxy when multiple applications need to share a public entry point.

- **Problem: A database container is exposed to the network unintentionally.**
  - **Cause:** Publishing a database port without specifying a host interface may expose it on the host's network interfaces, depending on networking and firewall configuration.
  - **Fix:** For local-only access, publish on `127.0.0.1`, for example `-p 127.0.0.1:5433:5432`. For application-to-database communication, prefer a shared user-defined Docker network without publishing the database port unless external access is required.

- **Problem: The host port is correct, but the container port is wrong.**
  - **Cause:** The mapping targets a port on which the application is not listening.
  - **Fix:** Check the application's configuration and listening sockets, then correct the container-side port. Remember that `-p 8080:80` means host port `8080` forwards to container port `80`, not the reverse.

- **Problem: A container is reachable on the host but not from a container on another Docker network.**
  - **Cause:** Published host ports and container-to-container connectivity are separate concerns. Containers on isolated networks may not be able to resolve or reach each other.
  - **Fix:** Attach the containers to an appropriate shared user-defined network and connect using the service or container name and internal port.

## References

- [Docker documentation: Published ports](https://docs.docker.com/engine/network/#published-ports)
- [Docker documentation: Port publishing and mapping](https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/)
- [Docker documentation: Networking](https://docs.docker.com/engine/network/)
- [Docker documentation: Docker run reference](https://docs.docker.com/reference/cli/docker/container/run/)
- [Linux manual: `ss`](https://man7.org/linux/man-pages/man8/ss.8.html)
