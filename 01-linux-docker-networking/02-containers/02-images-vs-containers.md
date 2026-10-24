# Images vs containers

> Phase 1 — Linux, Docker & Networking › Containers

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **container image** is an immutable template containing the filesystem, application code, runtime, libraries, dependencies, and configuration defaults required to run an application.

A **container** is a running or created instance of an image. It adds a writable container layer, runtime configuration, and isolated processes and resources.

The simplest way to understand the relationship is:

- **Image:** The packaged blueprint used to create containers.
- **Container:** An instance created from that blueprint.
- **Registry:** A repository service used to store and distribute images.
- **Container runtime:** Software that creates and manages containers from images, such as containerd or Docker Engine.

For example, `nginx:alpine` is an image reference. When you run it with `docker run`, Docker creates a container from that image and starts the NGINX web server.

```text
Dockerfile
    |
    | docker build
    v
Container Image
    |
    | docker run
    v
Container 1       Container 2       Container 3
    |                  |                  |
 NGINX process     NGINX process     NGINX process
```

Multiple containers can be created from the same image. Each container has its own writable layer, process environment, and runtime configuration, although containers can intentionally share networks, volumes, and other resources.

### Image lifecycle

1. **Build:** Create an image from a Dockerfile or another supported build source.
2. **Tag:** Assign a reference such as `myapp:1.0`.
3. **Push:** Upload the image to a registry.
4. **Pull:** Download the image to another machine.
5. **Create:** Create a container from the image.
6. **Start:** Run the container's configured process.
7. **Stop and remove:** Stop the container and optionally delete its container metadata and writable layer.

An image does not need to be running. It can remain stored locally or in a registry without any containers being created from it.

A container can also exist without currently running. For example, a stopped container still exists until it is removed.

## Key points

- **Images are templates; containers are instances.** An image describes the filesystem and application environment. A container is an isolated runtime instance created from that image.

- **Images are composed of layers.** Image layers represent filesystem changes introduced by build instructions. Shared layers can be reused across images, reducing storage and download costs.

- **Images are immutable by design.** Once an image is built, its content is not modified in place during normal use. A change to the application or its dependencies generally requires building a new image. Image tags can be reassigned to different image versions, so a tag alone does not guarantee immutable identity.

- **Containers have a writable layer.** When a container is created, the runtime adds a writable layer above the image's read-only layers. Files written into the container's writable layer belong to that container and are not automatically saved back into the image.

- **Container changes are usually temporary.** Removing a container also removes its writable layer. Data that must survive container replacement should be stored in a volume, bind mount, or external data service.

- **One image can create many containers.** You can run multiple instances of the same image with different names, ports, environment variables, resource limits, and network settings.

- **Containers do not automatically share writable files.** Two containers created from the same image have separate writable layers. They can share files through explicitly configured volumes or bind mounts.

- **Building and running are different operations.** `docker build` creates an image. `docker run` creates and starts a container from an image, creating the image locally first if necessary.

- **Pulling an image does not start an application.** `docker pull nginx:alpine` downloads the image. A separate container creation and start operation is needed to run NGINX.

- **Image tags are references, not guaranteed permanent identities.** Tags such as `latest`, `1.0`, or `alpine` can point to different image content over time. Image digests identify specific image content more reliably.

- **A container can be stopped, running, or exited.** The image remains available independently of the container's runtime state.

- **Containers share the host kernel.** An image contains the user-space filesystem and application dependencies, not a separate guest operating system kernel. The container runtime uses Linux kernel features such as namespaces and cgroups to isolate processes and control resources.

- **The container's main process matters.** When the process configured as the container's primary process exits, the container typically stops. A container is not simply a miniature virtual machine that remains running regardless of its application process.

- **Kubernetes schedules containers using images.** A Pod specification references container images. Kubernetes nodes pull the required images when needed and use the configured container runtime to create and run the containers.

## Commands / manifests

### 1. Pull an image without running it

```bash
docker pull nginx:alpine
```

This downloads the image to the local machine. It does not start an NGINX container.

List locally available images:

```bash
docker images
```

Or use the newer image-management command:

```bash
docker image ls
```

Inspect an image:

```bash
docker image inspect nginx:alpine
```

Display its repository, tag, image ID, layers, default command, environment settings, and other metadata.

### 2. Create and start a container

```bash
docker run -d --name web-1 -p 8080:80 nginx:alpine
```

Explanation:

- `docker run`: Create and start a container.
- `-d`: Run in detached mode.
- `--name web-1`: Assign a container name.
- `-p 8080:80`: Map host port `8080` to container port `80`.
- `nginx:alpine`: Select the image used to create the container.

Test the web server:

```bash
curl http://localhost:8080
```

List running containers:

```bash
docker ps
```

List all containers, including stopped ones:

```bash
docker ps -a
```

### 3. Create multiple containers from the same image

```bash
docker run -d --name web-1 -p 8080:80 nginx:alpine
docker run -d --name web-2 -p 8081:80 nginx:alpine
```

Both containers use the same image, but each has a separate writable layer and a different host port mapping.

Verify their image references:

```bash
docker inspect --format '{{.Name}} {{.Config.Image}}' web-1 web-2
```

Check their running processes and resource usage:

```bash
docker top web-1
docker top web-2

docker stats web-1 web-2
```

### 4. Compare an image with a container

Inspect the image:

```bash
docker image inspect nginx:alpine
```

Inspect the container:

```bash
docker container inspect web-1
```

Display the container's image ID:

```bash
docker inspect --format '{{.Image}}' web-1
```

Display the container's writable-layer changes:

```bash
docker diff web-1
```

`docker diff` lists filesystem changes made relative to the image, including additions, modifications, and deletions. It does not provide a complete record of every runtime change, and it does not show data written into separate mounted volumes as changes to the container's writable layer.

### 5. Understand image layers and writable layers

Display an image's build history:

```bash
docker history nginx:alpine
```

The image consists of read-only filesystem layers. A container adds a writable layer on top.

Test the writable layer:

```bash
docker exec web-1 sh -c 'echo "container-specific data" > /tmp/example.txt'
```

Read the file:

```bash
docker exec web-1 cat /tmp/example.txt
```

Create another container from the same image:

```bash
docker run -d --name web-3 nginx:alpine
```

Try reading the file in the new container:

```bash
docker exec web-3 cat /tmp/example.txt
```

The file will normally be absent because it was created in `web-1`'s writable layer, not in the image. The second container has its own writable layer.

### 6. Demonstrate what happens when a container is removed

Remove the test container:

```bash
docker rm -f web-1
```

Check whether the image still exists:

```bash
docker image ls nginx
```

The image remains available even though the container has been removed.

Create another container from the same image:

```bash
docker run -d --name web-1 -p 8080:80 nginx:alpine
```

The container is recreated from the image. Changes made only to the previous container's writable layer are not restored.

### 7. Build an image from a Dockerfile

Create a file named `Dockerfile`:

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

Create `index.html` in the same directory:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Image vs Container</title>
</head>
<body>
    <h1>Hello from my custom image</h1>
</body>
</html>
```

Build the image:

```bash
docker build -t my-nginx:1.0 .
```

Run a container from it:

```bash
docker run -d --name custom-web -p 8082:80 my-nginx:1.0
```

Test it:

```bash
curl http://localhost:8082
```

The Dockerfile is the build definition, `my-nginx:1.0` is the resulting image, and `custom-web` is a container created from that image.

### 8. Tag and push an image to a registry

Tag an image for Docker Hub:

```bash
docker tag my-nginx:1.0 YOUR_DOCKERHUB_USERNAME/my-nginx:1.0
```

Authenticate:

```bash
docker login
```

Push the image:

```bash
docker push YOUR_DOCKERHUB_USERNAME/my-nginx:1.0
```

On another machine, pull the image:

```bash
docker pull YOUR_DOCKERHUB_USERNAME/my-nginx:1.0
```

Create a container from the downloaded image:

```bash
docker run -d --name remote-web -p 8083:80 YOUR_DOCKERHUB_USERNAME/my-nginx:1.0
```

Replace `YOUR_DOCKERHUB_USERNAME` with your registry username. Authentication and registry permissions must allow the push or pull operation.

### 9. Preserve data outside the container's writable layer

Create a named volume:

```bash
docker volume create web-data
```

Run a container using that volume:

```bash
docker run -d \
  --name persistent-web \
  -p 8084:80 \
  -v web-data:/usr/share/nginx/html \
  nginx:alpine
```

The volume stores data independently of the container's writable layer. Removing the container does not automatically remove the named volume.

Inspect the volume:

```bash
docker volume inspect web-data
```

This is useful when container replacement should not destroy application data. Note that mounting a volume over a directory can hide the files that were already present at that location in the image.

### 10. Kubernetes image reference

A Kubernetes Pod can reference an image as follows:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
```

Save this as `nginx-pod.yaml`, then create the Pod:

```bash
kubectl apply -f nginx-pod.yaml
```

Inspect the Pod:

```bash
kubectl get pods
kubectl describe pod nginx-demo
kubectl get pod nginx-demo -o yaml
```

The `image` field identifies the image Kubernetes should use. The container runtime on the selected node obtains the image if it is not already available and creates the container.

Delete the Pod when finished:

```bash
kubectl delete -f nginx-pod.yaml
```

For reproducible deployments, use an intentional image version or digest instead of relying on a mutable tag such as `latest`.

## Common failures & fixes

- **`docker run` fails because the image cannot be found.**
  - **Cause:** The image name or tag is incorrect, the image is unavailable locally, or the registry cannot be reached.
  - **Fix:** Check `docker image ls`, verify the image reference, and run `docker pull IMAGE:TAG` if necessary. Check registry authentication and network connectivity for private or remote images.

- **A container exits immediately after starting.**
  - **Cause:** Its primary process exits, the application crashes, or the configured command is invalid.
  - **Fix:** Run `docker ps -a` and inspect the container with `docker logs CONTAINER_NAME` and `docker inspect CONTAINER_NAME`. Correct the command, configuration, or application error.

- **Changes made inside a container disappear after recreation.**
  - **Cause:** The changes existed only in the old container's writable layer.
  - **Fix:** Rebuild the image if the change belongs in the application package, or use a volume or external storage for persistent data. Avoid relying on manual modifications inside production containers.

- **A new container does not contain files created in another container.**
  - **Cause:** Containers created from the same image have independent writable layers.
  - **Fix:** Put shared or persistent files in a volume, bind mount, or external storage service.

- **A rebuilt image does not appear to contain the latest application code.**
  - **Cause:** The build context may be wrong, the relevant files may be excluded by `.dockerignore`, or a cached build step may be reused when the build inputs have not changed.
  - **Fix:** Verify the `COPY` source paths and build context. Inspect `.dockerignore`, rebuild the image, and use `docker build --no-cache -t myapp:1.0 .` when you need to rule out build-cache reuse.

- **A container runs an older image than expected.**
  - **Cause:** The container was created from an earlier image, or a mutable tag now points to different content.
  - **Fix:** Inspect the container's image ID and the local image ID. Pull the intended image and recreate the container. Pulling or rebuilding an image does not automatically replace an existing container.

- **Docker reports that a container name is already in use.**
  - **Cause:** A container with that name already exists, even if it is stopped.
  - **Fix:** Run `docker ps -a`, then start the existing container, remove it if no longer needed, or choose a different name.

- **Docker reports that a host port is already allocated.**
  - **Cause:** Another container or host process is already using the requested host port.
  - **Fix:** Inspect `docker ps` and change the host-side port mapping, for example from `-p 8080:80` to `-p 8085:80`.

- **A pushed image cannot be pulled on another machine.**
  - **Cause:** The image was not pushed, the registry reference is incorrect, credentials are missing, or the repository is private.
  - **Fix:** Verify the registry URL and image tag, check `docker push` output, and authenticate with `docker login` when required.

- **Kubernetes reports `ImagePullBackOff` or `ErrImagePull`.**
  - **Cause:** The image reference is invalid, the registry is unavailable, access is denied, or the image cannot be pulled by the node.
  - **Fix:** Run `kubectl describe pod POD_NAME` and inspect the Events section. Correct the image name or tag, configure registry credentials when required, and verify registry connectivity.

- **A container's writable-layer changes are missing from `docker diff`.**
  - **Cause:** The files may be inside a mounted volume rather than the container's writable layer.
  - **Fix:** Inspect the container's mounts using `docker inspect CONTAINER_NAME`. Check the mounted volume or bind-mount source directly.

## References

- [Docker Docs: What is an image?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)
- [Docker Docs: What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Docker Docs: Building, tagging, and pushing images](https://docs.docker.com/get-started/introduction/build-and-push-first-image/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Docker Docs: Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Kubernetes Docs: Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Kubernetes Docs: Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
