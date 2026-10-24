# Volumes

> Phase 1: Linux, Docker & Networking › Docker runtime

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Docker volumes provide persistent storage for data generated or used by containers. A container's writable filesystem is tied to its lifecycle, so data stored only inside that filesystem can be lost when the container is removed. Volumes and bind mounts allow data to exist independently of a container and, in some cases, to be shared between multiple containers.

Docker supports two common approaches to mounting host storage into containers: **named volumes** and **bind mounts**.

- **Named volumes:** Docker manages the volume's storage location and lifecycle. You reference it by name rather than by a host directory path. Named volumes are useful for databases, persistent application data, and storage that should be managed separately from a container.
- **Bind mounts:** You explicitly map a file or directory from the host machine into a container. They are useful when developing applications locally, because changes made to source files on the host can appear inside the container.

### How container storage works

By default, a container has a writable layer associated with its image. Changes made to that layer are specific to the container and are not automatically shared with other containers.

When you mount storage into a container, the mounted path is backed by the specified host directory or Docker volume rather than the container's ordinary writable layer.

For example, mounting a volume at `/var/lib/mysql` allows database files to persist outside the container's writable layer. Removing and recreating the container does not automatically remove the named volume.

### Named volumes vs bind mounts

| Feature | Named volume | Bind mount |
|---|---|---|
| Storage location | Managed by Docker | Explicit host path |
| Configuration | Volume name | Host file or directory path |
| Best suited for | Persistent application and database data | Source code, configuration files, local development |
| Host path knowledge | Usually unnecessary | Required |
| Portability | Generally easier to configure across environments | Depends on host paths and filesystem layout |
| Data persistence | Survives container removal unless the volume is deleted | Files remain in the host directory independently of the container |
| Host access | Possible, but Docker manages the storage location | Direct access through the mapped host path |
| Permissions | Depends on the container process and volume contents | Depends on host permissions, ownership, and filesystem behavior |

### When to use each

Use a **named volume** when the application needs persistent data and Docker should manage its storage location. A database is a common example.

Use a **bind mount** when the container needs access to a specific host directory or file. Application source code, local configuration, and development files are common examples.

For production workloads, the storage strategy should also account for backups, access permissions, data recovery, and host failure. A local Docker volume is not automatically a replicated or backed-up storage system.

## Key points

- **Container lifecycle and data lifecycle are different.** Removing a container does not automatically delete its named volumes or bind-mounted host files.
- **Named volumes are managed by Docker.** Create and reference them by name instead of relying on a specific host filesystem path.
- **Bind mounts use explicit host paths.** They are useful when the host and container need to work with the same files.
- **Mount destinations matter.** Mounting a volume over a directory hides the image's existing contents at that path while the mount is active. Those files are not necessarily deleted; they are obscured by the mounted filesystem.
- **An empty named volume may be initialized with image contents.** When mounted over a non-empty directory in the container, Docker normally copies the existing contents into a new empty volume unless the `volume-nocopy` option is used.
- **Bind mounts do not copy image contents into the host directory.** The host directory's contents appear at the container mount destination, potentially hiding files already present in the image.
- **Mounts can be read-only.** Use `readonly` or `ro` when a container should read files without modifying them.
- **Permissions still apply.** A process running as a non-root user inside the container may not be able to write to a mounted directory if filesystem permissions do not allow it.
- **Named volumes can be shared.** Multiple containers can mount the same volume, but concurrent access must be safe for the application and storage format.
- **A named volume is not the same as a backup.** Deleting a volume, losing the Docker host, or corrupting application data can still cause data loss.
- **`docker compose down` normally preserves named volumes.** Adding `--volumes` or `-v` removes the project's declared named volumes and attached anonymous volumes.
- **Anonymous volumes are different from named volumes.** Docker assigns anonymous volumes generated identifiers. They can persist beyond container removal but are harder to identify and reuse intentionally.
- **Mounts are not network storage by default.** A standard local Docker volume is typically stored on the Docker host. Sharing data across machines requires an appropriate network or distributed storage solution.

## Commands / manifests

### 1. Create and inspect a named volume

```bash
# Create a named volume
docker volume create app-data

# List Docker volumes
docker volume ls

# Inspect volume metadata
docker volume inspect app-data
```

`docker volume inspect` returns metadata including the volume's name, driver, and mountpoint. On a typical Linux Docker Engine installation, the mountpoint is under Docker's data directory. Docker Desktop uses a managed Linux environment, so the host-visible storage layout may differ.

### 2. Persist data with a named volume

```bash
# Start a container and mount the named volume
docker run -d \
  --name volume-demo \
  --mount source=app-data,target=/data \
  alpine:3.22 \
  sh -c "while true; do sleep 3600; done"

# Write a file into the mounted directory
docker exec volume-demo sh -c \
  "echo 'Persistent Docker data' > /data/message.txt"

# Read the file
docker exec volume-demo cat /data/message.txt

# Remove the container
docker rm -f volume-demo

# Recreate a container using the same volume
docker run --rm \
  --mount source=app-data,target=/data \
  alpine:3.22 \
  cat /data/message.txt
```

Expected output:

```text
Persistent Docker data
```

The file survives container removal because it is stored in the named volume, not only in the container's writable layer.

### 3. Use a bind mount

The following example creates a directory on the host and mounts it into a container.

```bash
# Create a directory and a file on the host
mkdir -p "$PWD/app-config"
echo "environment=development" > "$PWD/app-config/app.conf"

# Mount the host directory into the container
docker run --rm \
  --mount type=bind,source="$PWD/app-config",target=/config,readonly \
  alpine:3.22 \
  cat /config/app.conf
```

Expected output:

```text
environment=development
```

The `--mount` options mean:

- `type=bind`: Use a host directory or file as the storage source.
- `source="$PWD/app-config"`: Specify the host path.
- `target=/config`: Specify the path inside the container.
- `readonly`: Prevent the container from writing through this mount.

The host directory must exist. Unlike some `-v` bind-mount usage, `--mount type=bind` normally returns an error when the source path does not exist.

### 4. Compare named volumes and bind mounts

```bash
# Named volume
docker run --rm \
  --mount source=app-data,target=/data \
  alpine:3.22 \
  sh -c "echo named-volume > /data/named.txt"

# Bind mount
mkdir -p "$PWD/shared-data"

docker run --rm \
  --mount type=bind,source="$PWD/shared-data",target=/data \
  alpine:3.22 \
  sh -c "echo bind-mount > /data/bind.txt"

# Verify the bind-mounted file on the host
cat "$PWD/shared-data/bind.txt"

# Verify the named-volume file in another container
docker run --rm \
  --mount source=app-data,target=/data \
  alpine:3.22 \
  cat /data/named.txt
```

Expected output from the final two commands:

```text
bind-mount
named-volume
```

The bind-mounted file is directly visible in `shared-data` on the host. The named-volume file is managed in Docker's volume storage.

### 5. Run a database with persistent storage

This example demonstrates the volume pattern for PostgreSQL. Use a suitable password for real environments and avoid placing production secrets directly in shell history or committed configuration.

```bash
docker volume create postgres-data

docker run -d \
  --name postgres-demo \
  -e POSTGRES_PASSWORD=local-dev-password \
  -e POSTGRES_DB=appdb \
  --mount source=postgres-data,target=/var/lib/postgresql/data \
  postgres:17
```

Inspect the container logs:

```bash
docker logs postgres-demo
```

After PostgreSQL has initialized, remove and recreate the container using the same volume and compatible image version and configuration:

```bash
docker rm -f postgres-demo

docker run -d \
  --name postgres-demo \
  -e POSTGRES_PASSWORD=local-dev-password \
  -e POSTGRES_DB=appdb \
  --mount source=postgres-data,target=/var/lib/postgresql/data \
  postgres:17
```

The database files remain in `postgres-data`. For a meaningful persistence test, create a table and insert a record before removing the first container, then query the record after recreating it.

**Important:** Persistent database storage does not guarantee compatibility across arbitrary database versions. Follow the database's upgrade and backup procedures.

### 6. Use volumes in Docker Compose

Create a `compose.yaml` file:

```yaml
services:
  app:
    image: alpine:3.22
    command: ["sh", "-c", "while true; do sleep 3600; done"]
    volumes:
      - app-data:/data
      - ./app-config:/config:ro

volumes:
  app-data:
```

Start the service:

```bash
docker compose up -d
```

Inspect the container and its mounts:

```bash
docker compose ps
docker inspect "$(docker compose ps -q app)"
```

Stop and remove the Compose containers and network:

```bash
docker compose down
```

The named volume `app-data` is normally retained. To remove the Compose project's declared named volumes as well:

```bash
docker compose down --volumes
```

Be careful with `--volumes`: removing a volume can permanently delete the data stored in it unless you have a separate backup.

### 7. Inspect mounts on a container

```bash
# Show all mounts for a container
docker inspect volume-demo --format '{{json .Mounts}}'

# Show a specific container's mount information
docker inspect volume-demo
```

The mount information includes fields such as:

- `Type`: `volume` or `bind`.
- `Source`: The volume name's storage location or the host path.
- `Destination`: The path inside the container.
- `RW`: Whether the mount is writable.

For a quick check, run `docker inspect` against a currently existing container. A removed container can no longer be inspected.

### 8. Remove unused volumes safely

```bash
# List volumes
docker volume ls

# Inspect a volume before deletion
docker volume inspect app-data

# Remove a specific volume
docker volume rm app-data

# Remove unused volumes
docker volume prune
```

Only remove a volume when its data is no longer needed. `docker volume prune` removes unused volumes according to Docker's pruning rules. By default, it targets unused anonymous volumes; use `docker volume prune --all` to include unused named volumes where supported by the installed Docker version.

## Common failures & fixes

- **Data disappears after a container is removed.**
  - **Cause:** The data was written only to the container's writable layer, or the volume was removed.
  - **Fix:** Mount a named volume or bind mount at the path where the application writes persistent data. Verify the actual application data directory before configuring the mount.

- **The container cannot see files that exist in the image.**
  - **Cause:** A volume or bind mount is mounted over the directory containing those files.
  - **Fix:** Choose a mount destination that does not hide required image contents. For an empty named volume, review Docker's copy-up behavior and the `volume-nocopy` option.

- **A bind mount is empty or contains unexpected files.**
  - **Cause:** The wrong host path was mounted, or the host directory was empty and obscured the image's files.
  - **Fix:** Check `pwd`, confirm the source directory's contents, and inspect the container's mounts with `docker inspect`.

- **Docker reports that a bind-mount source path does not exist.**
  - **Cause:** The host path is incorrect or missing.
  - **Fix:** Create the directory first and use an absolute path where practical. Remember that `--mount type=bind` normally requires the source path to exist.

- **The application receives a permission-denied error.**
  - **Cause:** The container process lacks the required permissions on the mounted files or directory.
  - **Fix:** Check the container's effective user and the host file ownership and permissions. Grant only the permissions needed by the application rather than defaulting to broad permissions such as `777`.

- **A read-only mount prevents application writes.**
  - **Cause:** The mount was configured with `readonly` or `ro`.
  - **Fix:** Remove the read-only setting only if the application genuinely needs write access. Keep configuration files read-only when possible.

- **A database starts with an empty data directory after mounting a volume.**
  - **Cause:** The database is using a new volume, the wrong volume, or a mount destination that differs from the database's configured data directory.
  - **Fix:** Inspect the container's mount configuration and verify the volume name and database data path. Do not delete or recreate volumes until the original data location has been checked.

- **A named volume is missing after running `docker compose down`.**
  - **Cause:** The command may have included `--volumes`, or the volume was created under a different Compose project name.
  - **Fix:** Run `docker volume ls` and inspect the Compose configuration and project name. Use an explicitly named external volume when it must be managed independently of a Compose project.

- **Two containers corrupt or overwrite shared data.**
  - **Cause:** Both containers write to the same volume without coordination, or the application does not support concurrent access.
  - **Fix:** Review the application's concurrency guarantees. Use appropriate locking, a single-writer design, or a storage system designed for concurrent access.

- **A container works on one machine but not another.**
  - **Cause:** The bind mount depends on a host-specific path or the required directory permissions differ.
  - **Fix:** Prefer named volumes when Docker-managed storage is appropriate. For bind mounts, document the required directory structure and permissions.

- **A volume exists but its data cannot be recovered after host failure.**
  - **Cause:** A local volume stores data on the Docker host and is not automatically replicated or backed up.
  - **Fix:** Implement tested backups and recovery procedures, or use storage with the durability and availability guarantees required by the application.

- **A volume cannot be removed because it is in use.**
  - **Cause:** One or more containers still reference the volume.
  - **Fix:** Identify the containers using it, stop and remove them if appropriate, and retry. Do not force removal without understanding the impact on running workloads.

## References

- [Docker documentation: Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Docker documentation: Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [Docker documentation: Storage overview](https://docs.docker.com/engine/storage/)
- [Docker documentation: `docker volume` CLI](https://docs.docker.com/reference/cli/docker/volume/)
- [Docker documentation: Compose services and volumes](https://docs.docker.com/reference/compose-file/services/)
- [Docker documentation: `docker compose down`](https://docs.docker.com/reference/cli/docker/compose/down/)
- [PostgreSQL documentation](https://www.postgresql.org/docs/)
