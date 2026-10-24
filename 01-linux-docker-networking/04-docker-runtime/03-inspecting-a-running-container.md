# Inspecting a running container

> Phase 1 — Linux, Docker & Networking › Docker runtime

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Inspecting a running container means examining its logs, executing commands inside it, and retrieving its configuration and runtime metadata to understand its current state and troubleshoot problems.

Docker provides three essential commands for container inspection:

- `docker logs`: Reads the output written to a container's standard output (`stdout`) and standard error (`stderr`). It is commonly used to investigate application errors, startup failures, and unexpected behavior.
- `docker exec`: Runs a new process inside an existing, running container. It is useful for checking environment variables, examining files, testing connectivity, and debugging application behavior.
- `docker inspect`: Returns detailed JSON information about a Docker object, such as a container, image, network, or volume. For containers, this includes configuration, state, networking, mounts, and other metadata.

These commands answer different questions:

| Command | Primary question | Typical use |
|---|---|---|
| `docker logs` | What has the application printed? | Find exceptions and startup errors |
| `docker exec` | What can I check inside the container right now? | Examine files, processes, and connectivity |
| `docker inspect` | How is this container configured, and what is its state? | Check exit status, ports, mounts, and network settings |

A practical debugging workflow is to check the container's status, read its logs, inspect its configuration, and then execute targeted commands inside it if it is still running.

## Key points

- **`docker logs` retrieves application output**
  - By default, it displays the available logs for the specified container.
  - `-f` or `--follow` streams new log output as it arrives.
  - `--tail N` displays only the last `N` lines.
  - `--since` filters logs by a timestamp or duration, such as `10m` or an RFC 3339 timestamp.
  - `-t` or `--timestamps` adds timestamps to log entries.
  - Logs are useful only to the extent that the application writes useful information to `stdout` and `stderr` and the configured logging driver makes it available.
  - An application that writes exclusively to a file inside the container may not expose that file's contents through `docker logs`.

- **`docker exec` starts an additional process**
  - The container must be running.
  - The command runs in the existing container's namespaces and filesystem environment.
  - `docker exec -it CONTAINER sh` opens an interactive shell if a suitable shell exists in the image.
  - `-i` keeps standard input open, while `-t` allocates a pseudo-terminal.
  - `docker exec` does not restart the container or attach to the original application process.
  - Processes started using `docker exec` exist only while the container remains running. They are not automatically restarted when the container restarts.
  - A minimal or distroless image may not contain `sh`, `bash`, `curl`, or other debugging utilities.

- **`docker inspect` exposes detailed metadata**
  - It returns JSON describing the requested Docker object.
  - `--format` or `-f` uses Go templates to extract specific fields.
  - `.State.Status` reports the container's current state, such as `running`, `exited`, or `restarting`.
  - `.State.ExitCode` reports the last recorded process exit code.
  - `.NetworkSettings.Ports` contains configured port bindings.
  - `.Mounts` describes mounted volumes and bind mounts.
  - `.Config.Env` contains configured environment variables and may expose sensitive values.
  - Inspection output describes Docker's recorded configuration and state. It does not guarantee that the application itself is healthy.

- **Container states matter**
  - `created`: The container has been created but has not started.
  - `running`: Its main container process is running.
  - `paused`: Its processes are suspended.
  - `restarting`: Docker is restarting the container.
  - `exited`: Its main process has terminated.
  - `dead`: The container is in a terminal error state that Docker cannot recover from automatically.
  - `docker exec` generally requires the container to be running. `docker logs` and `docker inspect` can often still help diagnose an exited container.

- **Container ID versus container name**
  - Both can identify a container.
  - Names are usually easier to read in commands and scripts.
  - Use `docker ps` to list running containers and `docker ps -a` to include stopped containers.

- **Logs are not the same as health checks**
  - A container can be running while its application is failing.
  - A Docker health check can report `healthy` or `unhealthy` independently of the container's running state.
  - Use `docker inspect` to check health status when a health check is configured.

- **Use inspection commands carefully**
  - Container logs may contain credentials, personal information, or sensitive request data.
  - Inspection output may contain secrets in environment variables, command arguments, or labels.
  - Avoid sharing raw diagnostic output without reviewing it first.

## Commands / manifests

**1. Start a sample container for inspection**

```bash
docker run -d \
  --name inspect-demo \
  -e APP_ENV=development \
  -p 8080:80 \
  nginx:alpine
```

This starts an NGINX container in the background.

- `-d` runs the container in detached mode.
- `--name inspect-demo` assigns a readable name.
- `-e APP_ENV=development` supplies an environment variable.
- `-p 8080:80` maps host port `8080` to container port `80`.

The example uses NGINX because it produces accessible HTTP responses and normally writes access and error logs to the container's logging streams.

**2. List running and stopped containers**

```bash
docker ps
```

Include stopped containers:

```bash
docker ps -a
```

Display only the container ID, name, and status:

```bash
docker ps --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}'
```

Use this first when you do not know the exact container name or whether the container is still running.

**3. Read container logs**

```bash
docker logs inspect-demo
```

Display only the last 50 lines:

```bash
docker logs --tail 50 inspect-demo
```

Follow new log output:

```bash
docker logs -f inspect-demo
```

Follow logs and display timestamps:

```bash
docker logs -f -t inspect-demo
```

Show logs from the last 10 minutes:

```bash
docker logs --since 10m inspect-demo
```

Follow logs from the last five minutes:

```bash
docker logs --since 5m -f inspect-demo
```

Press `Ctrl+C` to stop following logs. This stops the log-following command, not the container.

To generate an NGINX access log, send a request from the host:

```bash
curl -i http://localhost:8080
```

Then check the logs:

```bash
docker logs --tail 20 inspect-demo
```

The request should produce an HTTP response and normally an access-log entry.

**4. Execute commands inside a running container**

Check the container's environment:

```bash
docker exec inspect-demo printenv APP_ENV
```

Expected output:

```text
development
```

Open an interactive shell:

```bash
docker exec -it inspect-demo sh
```

Inside the container, try:

```sh
printenv
ls -la /
cat /etc/os-release
exit
```

NGINX's Alpine-based image generally includes `sh`, but it may not include Bash or every common troubleshooting utility.

Run a command without opening a shell:

```bash
docker exec inspect-demo ls -la /etc/nginx
```

Inspect running processes, if the required utility is available:

```bash
docker exec inspect-demo ps
```

Execute a command as a different user when permissions allow:

```bash
docker exec -u 0 inspect-demo id
```

Here, `-u 0` runs the command as user ID `0`, typically root. Use elevated privileges only when necessary.

**5. Understand the shell-expansion difference**

This command runs `printenv` directly:

```bash
docker exec inspect-demo printenv APP_ENV
```

This command invokes a shell to interpret shell syntax:

```bash
docker exec inspect-demo sh -c 'echo "$APP_ENV"'
```

Use `sh -c` when you need shell features such as variable expansion, pipes, redirection, or multiple commands.

For example:

```bash
docker exec inspect-demo sh -c 'printenv APP_ENV; ls -la /etc/nginx'
```

Without a shell, Docker does not interpret shell operators such as `|`, `&&`, or `>` for you.

**6. Inspect the complete container configuration**

```bash
docker inspect inspect-demo
```

This returns a JSON array containing detailed metadata. To retrieve a single container's data, access the first array element when processing the JSON programmatically.

Inspect the current state:

```bash
docker inspect \
  --format '{{.State.Status}}' \
  inspect-demo
```

Inspect the exit code:

```bash
docker inspect \
  --format '{{.State.ExitCode}}' \
  inspect-demo
```

Inspect the configured environment variables:

```bash
docker inspect \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  inspect-demo
```

**Security note:** Review this output before sharing it. Environment variables can contain credentials or tokens.

Inspect port bindings:

```bash
docker inspect \
  --format '{{json .NetworkSettings.Ports}}' \
  inspect-demo
```

Inspect mounts:

```bash
docker inspect \
  --format '{{json .Mounts}}' \
  inspect-demo
```

Inspect the container's IP addresses:

```bash
docker inspect \
  --format '{{json .NetworkSettings.Networks}}' \
  inspect-demo
```

Inspect the image used to create the container:

```bash
docker inspect \
  --format '{{.Config.Image}}' \
  inspect-demo
```

Inspect the container's configured restart policy:

```bash
docker inspect \
  --format '{{json .HostConfig.RestartPolicy}}' \
  inspect-demo
```

These commands use Go-template expressions to select fields from Docker's inspection data.

**7. Check whether the application is healthy**

If the image defines a Docker health check, inspect its status:

```bash
docker inspect \
  --format '{{if .State.Health}}{{.State.Health.Status}}{{else}}No health check configured{{end}}' \
  inspect-demo
```

NGINX images do not necessarily define a Docker health check, so the output may report that none is configured.

List container health-check history when available:

```bash
docker inspect \
  --format '{{json .State.Health.Log}}' \
  inspect-demo
```

A `running` state means the container's main process is running. It does not by itself prove that the application can serve requests successfully.

**8. Investigate an exited container**

Run a container that exits with an error:

```bash
docker run --name failed-demo alpine:3.22 \
  sh -c 'echo "Application failed to start" >&2; exit 1'
```

Inspect all containers, including stopped ones:

```bash
docker ps -a
```

Read its logs:

```bash
docker logs failed-demo
```

Check its exit code and state:

```bash
docker inspect \
  --format 'Status={{.State.Status}} ExitCode={{.State.ExitCode}}' \
  failed-demo
```

Expected state includes `Status=exited` and `ExitCode=1`.

You cannot run `docker exec` against this container because its main process has terminated. Use logs and inspection first. If necessary, create a separate debugging container using the same image or inspect the stopped container's filesystem through appropriate Docker tooling.

Clean up the sample containers:

```bash
docker rm -f inspect-demo failed-demo
```

**9. Combine the commands into a debugging workflow**

```bash
# 1. Find the container and check its status
docker ps -a

# 2. Read the most recent application output
docker logs --tail 100 CONTAINER_NAME

# 3. Inspect the state and exit code
docker inspect \
  --format 'Status={{.State.Status}} ExitCode={{.State.ExitCode}}' \
  CONTAINER_NAME

# 4. If the container is running, inspect its environment
docker exec CONTAINER_NAME printenv

# 5. Inspect ports and mounts
docker inspect \
  --format 'Ports={{json .NetworkSettings.Ports}} Mounts={{json .Mounts}}' \
  CONTAINER_NAME
```

Replace `CONTAINER_NAME` with the actual name or ID. Run the `docker exec` command only if the container is running, and avoid displaying or sharing environment variables indiscriminately.

## Common failures & fixes

- **`No such container`**
  - **Cause:** The name or ID is incorrect, or the container was removed.
  - **Fix:** Run `docker ps -a` and copy the correct name or ID.
  - **Verify:** Retry the command using the listed identifier.

- **`docker exec` reports that the container is not running**
  - **Cause:** The main process has exited, the container is stopped, or it is restarting.
  - **Fix:** Run `docker logs` and `docker inspect` to investigate the failure before deciding whether to restart it.
  - **Verify:** Check `.State.Status` and `.State.ExitCode`.

- **`exec: "bash": executable file not found`**
  - **Cause:** The image does not include Bash. Minimal images often contain only a limited set of utilities.
  - **Fix:** Try `docker exec -it CONTAINER_NAME sh` if a POSIX shell is available.
  - **Verify:** Use a suitable shell or an approved debugging container if the image has no shell.

- **`docker logs` returns no output**
  - **Cause:** The application has not written to `stdout` or `stderr`, the relevant output is stored in files, or the logging configuration does not expose the expected records.
  - **Fix:** Check the application's logging configuration and Docker logging driver. Inspect application log files if appropriate.
  - **Verify:** Trigger a known request or event and confirm where the application writes its logs.

- **Logs show an application error, but the container is running**
  - **Cause:** The application may have encountered an error without terminating, or only part of the application is failing.
  - **Fix:** Check application health, dependencies, and recent error messages. Do not assume that a running container is healthy.
  - **Verify:** Send a test request and inspect the response, application logs, and health-check status.

- **`docker inspect` output is too large**
  - **Cause:** The command returns detailed JSON for the entire object.
  - **Fix:** Use `--format` to retrieve only the required fields.
  - **Verify:** Extract `.State.Status`, `.State.ExitCode`, `.NetworkSettings.Ports`, or `.Mounts` as appropriate.

- **A port is shown in inspection, but the application is unreachable**
  - **Cause:** The host port may not be published as expected, the application may be listening on the wrong interface, a firewall may block access, or the application may not be ready.
  - **Fix:** Check the port mapping with `docker port CONTAINER_NAME`, inspect the application logs, and test connectivity.
  - **Verify:** Run `curl -v http://localhost:HOST_PORT` from the Docker host when the service is expected to be reachable there.

- **A debugging command fails because a utility is missing**
  - **Cause:** The container image is designed to run the application, not provide a complete debugging environment.
  - **Fix:** Use available commands, application-specific diagnostics, or an approved temporary debugging container.
  - **Verify:** Confirm that the diagnostic method tests the same relevant network, permissions, or runtime conditions.

- **Container inspection exposes credentials**
  - **Cause:** Sensitive values were supplied through environment variables or embedded in container configuration.
  - **Fix:** Redact diagnostic output before sharing it, rotate exposed credentials when necessary, and use a secrets-management mechanism for sensitive values.
  - **Verify:** Review the output and any logs or tickets where it may have been copied.

## References

- [Docker CLI: `docker logs`](https://docs.docker.com/reference/cli/docker/container/logs/)
- [Docker CLI: `docker exec`](https://docs.docker.com/reference/cli/docker/container/exec/)
- [Docker CLI: `docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/)
- [Docker CLI: `docker ps`](https://docs.docker.com/reference/cli/docker/container/ls/)
- [Docker logging](https://docs.docker.com/engine/logging/)
- [Docker health checks in the Dockerfile reference](https://docs.docker.com/reference/dockerfile/#healthcheck)
