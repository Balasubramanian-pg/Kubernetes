# Building, tagging, pushing an image

> Phase 1 — Linux, Docker & Networking › Docker runtime

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Building, tagging, and pushing a Docker image is the process of packaging an application into a reusable image, assigning it a name and version, and uploading it to a container registry so it can be distributed and deployed.

- **Build:** Converts a Dockerfile and its build context into an image using `docker build`.
- **Tag:** Assigns a human-readable name and version identifier to an image using `docker tag`. Tagging does not rebuild or duplicate the image.
- **Push:** Uploads an image and its required layers to a registry using `docker push`.
- **Pull:** Downloads an image from a registry using `docker pull`.

A container registry is a service that stores and distributes container images. Examples include Docker Hub, GitHub Container Registry (GHCR), Azure Container Registry (ACR), and Amazon Elastic Container Registry (ECR).

The typical workflow is:

1. Write the application and Dockerfile.
2. Build the image locally.
3. Tag it with the target registry, repository, and version.
4. Authenticate to the registry.
5. Push the image.
6. Pull and run the image from the registry, either locally or in a deployment environment.

An image reference generally follows this structure:

```text
[REGISTRY/]NAMESPACE/IMAGE_NAME:TAG
```

For example:

```text
docker.io/baludev/data-api:v1.0.0
```

- `docker.io` is the registry.
- `baludev` is the namespace or account.
- `data-api` is the repository name.
- `v1.0.0` is the tag.

The registry hostname may be omitted for Docker Hub references in common Docker CLI workflows. For other registries, include the registry hostname, such as `ghcr.io` or `myregistry.azurecr.io`.

## Key points

- **An image is not a running container**
  - An image is a packaged template containing application files, dependencies, metadata, and configuration.
  - A container is a runtime instance created from an image.
  - Building or pushing an image does not start the application.

- **The build context matters**
  - `docker build -t data-api:v1.0.0 .` uses the current directory as the build context.
  - Docker can access files in the build context when processing `COPY` and `ADD` instructions.
  - `.dockerignore` excludes unnecessary files from the context, reducing transfer size and preventing accidental inclusion of files such as local credentials.
  - The Dockerfile and build context are separate concepts. Specify a different Dockerfile with `-f` when needed.

- **Tags identify image versions**
  - A tag is a human-readable reference to an image, such as `v1.0.0`, `v1.1.0`, or `latest`.
  - `latest` is just a conventional tag. It does not guarantee that the image contains the newest application version.
  - Tags are mutable in many registries, meaning the same tag can be reassigned to another image.
  - A digest, such as `sha256:...`, identifies a specific image manifest and is useful when reproducibility matters.
  - Use meaningful release tags instead of relying exclusively on `latest`.

- **Tagging does not rebuild an image**
  - `docker tag SOURCE TARGET` creates another reference to the same local image.
  - Multiple tags can point to the same image ID.
  - Changing the tag does not modify the image contents.

- **Registry naming determines the push destination**
  - `docker push data-api:v1.0.0` attempts to push to the registry implied by that image reference, usually Docker Hub for an unqualified name.
  - `docker tag data-api:v1.0.0 ghcr.io/OWNER/data-api:v1.0.0` creates a reference suitable for GHCR.
  - The target repository must exist or be creatable under the registry's policies, and the authenticated account must have permission to push.

- **Authentication is required for private repositories**
  - Use `docker login REGISTRY_HOST` to authenticate.
  - For Docker Hub, use `docker login`.
  - For GHCR, use `docker login ghcr.io`.
  - For ACR, use the appropriate Azure authentication flow, such as `az acr login --name REGISTRY_NAME`.
  - Use an access token or other supported credential when required. Avoid embedding passwords in shell history or scripts.
  - Registry credentials are typically stored by Docker's credential helper or in the Docker configuration directory. Protect that configuration.

- **Pushing does not always upload every layer**
  - Docker images are composed of layers.
  - When the registry already contains matching layers, Docker may reuse them instead of uploading them again.
  - Build caching can also reduce the work required to rebuild an image.

- **Builds should be reproducible and secure**
  - Use a suitable base image and pin versions where practical.
  - Avoid copying secrets into image layers or passing sensitive values through build arguments.
  - Use BuildKit secret mounts for build-time credentials when necessary.
  - Scan images for known vulnerabilities and minimize unnecessary packages.
  - Consider immutable release references, image signing, and provenance attestations for production workloads.

- **Architecture compatibility matters**
  - Images may target a specific operating system and CPU architecture, such as `linux/amd64` or `linux/arm64`.
  - An image built for one architecture may not run natively on another.
  - Multi-platform images can be built and published with `docker buildx build --platform ... --push`.

## Commands / manifests

**1. Create a sample application**

Create a directory:

```bash
mkdir docker-image-demo
cd docker-image-demo
```

Create a file named `app.py`:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import os

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        message = f"Hello from {os.getenv('APP_ENV', 'development')}!"

        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        self.wfile.write(message.encode())

HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
```

Create a `Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY app.py .

ENV APP_ENV=production

EXPOSE 8080

CMD ["python", "app.py"]
```

Create a `.dockerignore` file:

```text
.git
.env
__pycache__/
*.pyc
```

This small application responds to HTTP requests on port `8080`. The Dockerfile selects a Python base image, copies the application, sets a default environment variable, documents the application port, and defines the startup command.

`EXPOSE 8080` documents the container port but does not publish it to the host. Port publishing is configured when running the container.

**2. Build the image**

```bash
docker build -t data-api:v1.0.0 .
```

Explanation:

- `docker build` builds an image from the Dockerfile.
- `-t data-api:v1.0.0` assigns the image name and version tag.
- `.` specifies the current directory as the build context.

List local images:

```bash
docker images
```

Inspect the image:

```bash
docker image inspect data-api:v1.0.0
```

Display its image ID:

```bash
docker image inspect \
  --format '{{.Id}}' \
  data-api:v1.0.0
```

The image ID is distinct from its tag. Multiple tags can reference the same image.

**3. Run and test the image locally**

```bash
docker run -d \
  --name data-api-test \
  -p 8080:8080 \
  data-api:v1.0.0
```

Test the application:

```bash
curl http://localhost:8080
```

Expected response:

```text
Hello from production!
```

Check the container logs:

```bash
docker logs data-api-test
```

Stop and remove the test container:

```bash
docker rm -f data-api-test
```

**4. Tag the image for Docker Hub**

Replace `DOCKERHUB_USERNAME` with your Docker Hub username:

```bash
docker tag \
  data-api:v1.0.0 \
  DOCKERHUB_USERNAME/data-api:v1.0.0
```

Confirm that both references exist:

```bash
docker images
```

The local image now has two references: `data-api:v1.0.0` and `DOCKERHUB_USERNAME/data-api:v1.0.0`. They point to the same image until one of the references is updated.

**5. Authenticate and push to Docker Hub**

Authenticate interactively:

```bash
docker login
```

Use the credentials or supported access token for your Docker Hub account.

Push the tagged image:

```bash
docker push DOCKERHUB_USERNAME/data-api:v1.0.0
```

Docker uploads the image manifest and any required layers that the registry does not already have.

The image can now be pulled by another machine with access to the repository:

```bash
docker pull DOCKERHUB_USERNAME/data-api:v1.0.0
```

Run it on that machine:

```bash
docker run -d \
  --name data-api \
  -p 8080:8080 \
  DOCKERHUB_USERNAME/data-api:v1.0.0
```

The Docker Hub repository must be accessible to the account performing the push. For a private repository, the machine pulling the image must authenticate with sufficient access.

**6. Push to GitHub Container Registry (GHCR)**

GHCR uses the `ghcr.io` registry hostname. Replace `GITHUB_USERNAME` with the account or organization that owns the package.

Authenticate:

```bash
docker login ghcr.io
```

When prompted, use a GitHub username and a personal access token with the required package permissions, such as `write:packages` for pushing. Token requirements can depend on the account and organization configuration.

Tag the image:

```bash
docker tag \
  data-api:v1.0.0 \
  ghcr.io/GITHUB_USERNAME/data-api:v1.0.0
```

Push it:

```bash
docker push ghcr.io/GITHUB_USERNAME/data-api:v1.0.0
```

Pull it:

```bash
docker pull ghcr.io/GITHUB_USERNAME/data-api:v1.0.0
```

Package visibility and access permissions are managed through GitHub's package settings. A successful login alone does not guarantee permission to push to every package.

**7. Build and push directly with Docker Buildx**

For a straightforward single-platform build, Buildx can build and push an image in one command:

```bash
docker buildx build \
  -t DOCKERHUB_USERNAME/data-api:v1.0.0 \
  --push \
  .
```

The `--push` option sends the resulting image to the registry instead of merely leaving it in the local image store.

For a multi-platform image:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t DOCKERHUB_USERNAME/data-api:v1.0.0 \
  --push \
  .
```

This publishes a multi-platform image reference when the selected Buildx builder supports the required platforms. Emulation or a multi-node builder may be needed for cross-platform builds.

**8. Tag and push a new release**

After changing the application code, build a new version:

```bash
docker build -t data-api:v1.1.0 .
```

Tag it for the registry:

```bash
docker tag \
  data-api:v1.1.0 \
  DOCKERHUB_USERNAME/data-api:v1.1.0
```

Push the release:

```bash
docker push DOCKERHUB_USERNAME/data-api:v1.1.0
```

A versioned release makes it easier to identify which application version a deployment is using and to roll back to a previous version when needed.

**9. Inspect the published image**

View the image's local metadata:

```bash
docker image inspect \
  DOCKERHUB_USERNAME/data-api:v1.0.0
```

For registry-side information, use the registry's interface or tooling. Docker Buildx also provides metadata and build options for release workflows.

To verify that the published tag can be pulled, remove the local registry-tagged reference and pull it again:

```bash
docker image rm DOCKERHUB_USERNAME/data-api:v1.0.0
docker pull DOCKERHUB_USERNAME/data-api:v1.0.0
```

Only remove the reference after confirming that doing so is safe for your local workflow. If another tag still references the same image, Docker may retain the underlying image data.

**10. Clean up**

Remove the test container if it still exists:

```bash
docker rm -f data-api-test
```

Remove local image references if no longer needed:

```bash
docker image rm data-api:v1.0.0
docker image rm DOCKERHUB_USERNAME/data-api:v1.0.0
```

Removing a local image does not delete the image from the remote registry. Registry-side deletion is a separate operation governed by the registry's permissions and retention policies.

## Common failures & fixes

- **`docker build` cannot find the Dockerfile**
  - **Cause:** The command is running in the wrong directory or the Dockerfile has a different name.
  - **Fix:** Run the build from the directory containing the Dockerfile, or specify it with `-f`.
  - **Verify:** Use `ls` to confirm the file exists, then run `docker build -f path/to/Dockerfile -t data-api:v1.0.0 path/to/context`.

- **`COPY` fails because a file cannot be found**
  - **Cause:** The file is outside the build context, the path is incorrect, or `.dockerignore` excludes it.
  - **Fix:** Correct the path, adjust the build context, or update `.dockerignore`.
  - **Verify:** Check that the required file is inside the selected context and is not excluded.

- **The image builds but the application exits**
  - **Cause:** The startup command fails, dependencies are missing, or the application is configured incorrectly.
  - **Fix:** Check `docker ps -a`, `docker logs CONTAINER_NAME`, and the Dockerfile's `CMD` or `ENTRYPOINT`.
  - **Verify:** Run the image locally and confirm the process remains running.

- **`docker push` reports `denied` or `unauthorized`**
  - **Cause:** The user is not authenticated, the token lacks required permissions, or the image tag points to a repository the account cannot write to.
  - **Fix:** Log in to the correct registry and confirm the namespace, repository, and token permissions.
  - **Verify:** Run `docker image inspect` on the tagged image and retry the push after correcting access.

- **The image is pushed to the wrong registry**
  - **Cause:** The image reference uses an unintended registry or omits the intended registry hostname.
  - **Fix:** Tag the image with the complete destination reference, such as `ghcr.io/OWNER/data-api:v1.0.0`.
  - **Verify:** Check the full tag with `docker images` before pushing.

- **The push reports that the repository does not exist**
  - **Cause:** The namespace or repository is incorrect, or the registry requires the repository to be created first.
  - **Fix:** Correct the image reference and create the repository if required by that registry.
  - **Verify:** Confirm the destination repository exists and the authenticated account can push to it.

- **The pushed image is not visible to another user**
  - **Cause:** The repository is private or the other user lacks package permissions.
  - **Fix:** Grant the required access or configure visibility according to the organization's policy.
  - **Verify:** Authenticate as the intended consumer and try `docker pull`.

- **The new image appears to be an old version**
  - **Cause:** The image was not rebuilt, the wrong tag was pushed, or a deployment continues using an older image reference.
  - **Fix:** Rebuild the image, assign a distinct release tag, push that tag, and update the deployment.
  - **Verify:** Compare the image digest and application version between the local image, registry, and deployed workload.

- **The image runs on one machine but fails on another**
  - **Cause:** The image targets an incompatible architecture, relies on environment-specific configuration, or depends on unavailable external services.
  - **Fix:** Publish the correct platform variants, provide the required runtime configuration, and validate dependencies.
  - **Verify:** Inspect the image platform and test the same release in the target environment.

- **The push is unexpectedly slow**
  - **Cause:** Large layers, poor network connectivity, or many changed layers increase the amount of data that must be uploaded.
  - **Fix:** Reduce unnecessary files in the build context, use `.dockerignore`, optimize Dockerfile layer reuse, and avoid copying build artifacts that are not required.
  - **Verify:** Review the build output and layer sizes, then compare the next build and push.

- **Credentials are exposed in an image**
  - **Cause:** Secrets were copied into the build context, written into a layer, or embedded in Dockerfile instructions.
  - **Fix:** Remove the secret from the build process, use BuildKit secret mounts where appropriate, rotate exposed credentials, and rebuild the image.
  - **Verify:** Inspect the Dockerfile, build context, image history, and image contents. Deleting a secret in a later layer does not remove it from earlier layers.

## References

- [Docker: Build an image](https://docs.docker.com/build/building/)
- [Docker CLI: `docker build`](https://docs.docker.com/reference/cli/docker/image/build/)
- [Docker CLI: `docker tag`](https://docs.docker.com/reference/cli/docker/image/tag/)
- [Docker CLI: `docker push`](https://docs.docker.com/reference/cli/docker/image/push/)
- [Docker CLI: `docker pull`](https://docs.docker.com/reference/cli/docker/image/pull/)
- [Docker Hub](https://hub.docker.com/)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Docker Buildx](https://docs.docker.com/build/buildx/)
- [Docker build secrets](https://docs.docker.com/build/building/secrets/)
