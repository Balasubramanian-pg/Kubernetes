# Image layers & caching

> Phase 1 — Linux, Docker & Networking › Containers

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Docker image layers are filesystem changes that are assembled to form a container image. Instead of storing every image as one completely independent filesystem, Docker can reuse layers shared by multiple images.

**Image caching** is the process of reusing previously built image layers and build results when building an image again. If a Dockerfile instruction and its relevant inputs have not changed, Docker can often reuse the cached result instead of executing that build step again.

This improves build speed, reduces redundant work, and can lower image distribution costs.

For example, consider a Python application with a `requirements.txt` file containing its dependencies.

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

Docker processes the Dockerfile instructions in sequence. Conceptually, each filesystem-changing instruction contributes a layer to the resulting image, while some other instructions affect image configuration rather than creating filesystem layers.

```text
Base image: python:3.12-slim
        |
        v
Set working directory: /app
        |
        v
Copy requirements.txt
        |
        v
Install Python dependencies
        |
        v
Copy application source code
        |
        v
Final image
```

When only `app.py` changes, Docker can often reuse the cached base image, working directory, dependency file, and installed dependencies. It then repeats the application-copy step and any subsequent affected steps.

If `requirements.txt` changes, the dependency installation step must normally be rebuilt because its input has changed.

### How image layers work

A built image is composed of a sequence of filesystem layers and image configuration metadata.

- **Base image:** The starting filesystem and configuration, such as `python:3.12-slim`.
- **Build layers:** Filesystem changes introduced by build instructions.
- **Read-only image content:** The stored image layers are not modified by normal container writes.
- **Container writable layer:** When a container is created, the runtime provides a separate writable layer above the image's read-only filesystem layers, subject to the storage driver and runtime implementation.
- **Layer reuse:** Identical or compatible layers can be reused instead of downloaded or built again.

A useful distinction is that a Dockerfile instruction does not necessarily correspond to one filesystem layer. Instructions such as `RUN`, `COPY`, and `ADD` generally create filesystem changes, while instructions such as `CMD`, `ENTRYPOINT`, and `ENV` primarily affect image configuration.

### How Docker build caching works

During a build, Docker evaluates the Dockerfile from top to bottom. For each instruction, it determines whether an existing build result can be reused.

Conceptually:

1. Docker evaluates the instruction and relevant inputs.
2. It checks whether a suitable cached result is available.
3. If a match exists, it reuses the result.
4. If the cache cannot be reused, it executes the instruction again.
5. Subsequent instructions may also need to be rebuilt because their inputs or parent results have changed.

The exact cache behavior depends on the builder, instruction type, build context, mount configuration, and relevant input metadata. Modern Docker builds commonly use BuildKit, which provides more advanced caching and build features.

### Example: inefficient versus efficient Dockerfile

An inefficient Dockerfile:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

CMD ["python", "app.py"]
```

The entire build context is copied before the dependency installation step. If any copied file changes, the cache for the `COPY . .` instruction is invalidated, so the dependency installation step will generally run again.

An improved Dockerfile:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

The dependency file is copied and processed separately from the application source code. If only application code changes and `requirements.txt` remains unchanged, Docker can usually reuse the cached dependency installation.

The improvement comes from arranging instructions according to how frequently their inputs change.

## Key points

- **Layers are reusable filesystem changes.** Image layers represent changes that contribute to the final image filesystem. Multiple images can share compatible layers, particularly when they use the same base image.

- **Caching avoids repeated work.** A cache hit allows Docker to reuse a previous build result. A cache miss means the relevant step must be processed again, although the builder may still reuse cached results for other steps.

- **Dockerfile instruction order matters.** Instructions that change frequently should generally appear after instructions whose inputs change infrequently. This preserves the cache for stable dependencies when application source code changes.

- **Separate dependency files from source code.** Copy files such as `requirements.txt`, `package-lock.json`, or `pom.xml` before copying the rest of the application. This lets dependency installation remain cached when only application code changes.

- **Cache invalidation can affect later instructions.** When a step cannot reuse its cached result, subsequent steps that depend on the new result may also need to execute again.

- **Build context matters.** The build context is the set of files made available to the builder. Copying the entire context too early can cause unnecessary cache invalidation when unrelated files change.

- **Use `.dockerignore`.** Exclude unnecessary files such as `.git`, local virtual environments, caches, temporary outputs, and local secrets. This reduces the context sent to the builder and helps prevent irrelevant files from affecting builds.

- **`RUN`, `COPY`, and `ADD` generally produce filesystem changes.** They commonly contribute filesystem layers in the resulting image. Configuration instructions such as `ENV`, `CMD`, and `ENTRYPOINT` primarily affect image metadata or runtime configuration.

- **Combine commands carefully.** Combining related commands into one `RUN` instruction can reduce the number of resulting filesystem layers, but it does not automatically make an image smaller. Delete temporary files and package-manager caches in the same layer where they are created if they should not remain in the final image.

- **Deleting a file in a later layer does not necessarily shrink the image.** If a file was introduced in an earlier layer and deleted in a subsequent layer, the earlier layer still contains the file's content. The later layer records the deletion. Avoid introducing unnecessary files into earlier layers.

- **Multi-stage builds can reduce final image size.** Compile or build the application in an intermediate stage, then copy only the required artifacts into a smaller runtime stage. Build-stage tools and dependencies do not need to be included in the final image.

- **Build cache and image layers are related but distinct.** Image layers are part of the resulting image filesystem. Build cache stores reusable build results. A build can reuse cached steps without requiring every cached intermediate result to appear in the final image.

- **`--no-cache` forces Dockerfile build steps to be processed again.** It disables normal build-layer cache reuse for the build. It does not automatically mean every base image will be downloaded again. Use `--pull` when you also want Docker to check for a newer version of the referenced base image.

- **BuildKit supports advanced caching.** BuildKit provides features such as cache mounts, secret mounts, parallel execution where possible, and external cache import/export. Cache mounts can preserve downloaded package data between builds without including that cache data in the final image.

- **Caching is not a substitute for reproducibility.** Pin base images and dependencies where appropriate. Mutable image tags and unpinned dependencies can produce different results even when the Dockerfile looks unchanged.

- **Build caches are not automatically shared across every machine.** A developer's laptop, a CI runner, and a Kubernetes node may have different caches. CI workflows can import and export supported build caches to accelerate builds across runners.

- **Runtime caching is different from build caching.** Docker build cache speeds up image construction. A container runtime may reuse already-downloaded image layers when pulling an image, but that is a separate mechanism from reusing Dockerfile build steps.

- **Layer sharing does not mean shared container state.** Containers may share read-only image layers while maintaining independent writable layers. Modifying a file in one container does not normally change that file in another container created from the same image.

## Commands / manifests

### 1. Create a sample application

Create a directory named `image-cache-demo` with these files.

`requirements.txt`:

```text
flask==3.1.0
```

`app.py`:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Image layer caching demo"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Create an inefficient Dockerfile named `Dockerfile.inefficient`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

CMD ["python", "app.py"]
```

Create an optimized Dockerfile named `Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
```

The optimized version copies dependency definitions separately from application code.

### 2. Add a `.dockerignore` file

Create `.dockerignore` in the same directory:

```text
.git
.venv
__pycache__
*.pyc
.env
dist
build
```

This prevents the listed files and directories from being included in the build context. Keep in mind that `.env` is excluded from the context here, but secrets should also never be embedded into image layers through other build steps.

### 3. Build the image and observe caching

Build the optimized image:

```bash
docker build -t image-cache-demo:1.0 .
```

Run the same command again without changing the inputs:

```bash
docker build -t image-cache-demo:1.0 .
```

The second build should be able to reuse cached results for unchanged steps. Build output commonly identifies reused steps as `CACHED` when using BuildKit.

Now change the response in `app.py`:

```python
return "Image caching makes builds faster"
```

Rebuild:

```bash
docker build -t image-cache-demo:1.0 .
```

The application-copy step should be rebuilt. Earlier steps, including dependency installation, can generally remain cached because their inputs have not changed.

### 4. Compare inefficient and optimized builds

Build the inefficient version:

```bash
docker build \
  -f Dockerfile.inefficient \
  -t image-cache-demo:inefficient .
```

Modify only `app.py`, then build it again:

```bash
docker build \
  -f Dockerfile.inefficient \
  -t image-cache-demo:inefficient .
```

Because `COPY . .` precedes dependency installation, changing a copied file can invalidate that copy step and cause the later `RUN pip install` step to execute again.

Compare with the optimized build:

```bash
docker build -t image-cache-demo:1.0 .
```

With the optimized Dockerfile, a change limited to `app.py` should generally leave the dependency-installation step cached.

For a meaningful comparison, inspect the build output and look for which steps are marked `CACHED`.

### 5. Inspect image history and layers

List images:

```bash
docker image ls
```

Inspect the image history:

```bash
docker history image-cache-demo:1.0
```

Inspect image metadata:

```bash
docker image inspect image-cache-demo:1.0
```

Compare the reported sizes:

```bash
docker image ls image-cache-demo
```

`docker history` displays the image's build history and associated layer sizes. It is useful for identifying large build steps, but it does not provide a complete picture of shared on-disk storage or the build cache.

### 6. Force a rebuild

Build while disabling normal build-cache reuse:

```bash
docker build --no-cache -t image-cache-demo:1.0 .
```

To also check for a newer version of the base image:

```bash
docker build --pull --no-cache -t image-cache-demo:1.0 .
```

The first command reprocesses the Dockerfile build steps without normal cache reuse. The second additionally asks Docker to check the referenced base image for a newer version.

### 7. Use a cache mount for package downloads

With BuildKit, a cache mount can preserve package-download data between builds:

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
```

Build it:

```bash
docker build -t image-cache-demo:cache-mount .
```

The cache mount can retain downloaded package data between builds, making dependency installation faster when the package cache is reusable. The mounted cache itself is not committed to the resulting image layer.

### 8. Inspect and clean up build cache

Display Docker disk usage:

```bash
docker system df
```

Display detailed build-cache usage:

```bash
docker buildx du
```

Remove unused build cache:

```bash
docker builder prune
```

For Buildx-managed builders, you can also prune their cache using the supported Buildx commands. Pruning removes reusable cache entries and may make future builds slower because steps must be rebuilt or cache data downloaded again.

Use cleanup commands carefully, particularly on shared CI runners or development machines.

### 9. Run the image and verify the application

Run the application:

```bash
docker run -d \
  --name image-cache-demo \
  -p 5000:5000 \
  image-cache-demo:1.0
```

Test it:

```bash
curl http://localhost:5000
```

Inspect its logs:

```bash
docker logs image-cache-demo
```

Remove the container when finished:

```bash
docker rm -f image-cache-demo
```

The built image remains available locally:

```bash
docker image ls image-cache-demo
```

## Common failures & fixes

- **Every build step executes again even when little has changed.**
  - **Cause:** An early instruction changes frequently, a copied file has changed, or the build context contains unnecessary files.
  - **Fix:** Move stable dependency instructions earlier, copy dependency manifests separately, and use `.dockerignore`.

- **Changing application code triggers dependency installation again.**
  - **Cause:** The Dockerfile copies all application files before installing dependencies.
  - **Fix:** Copy dependency manifests first, install dependencies, then copy application code.

- **The image remains large after deleting unnecessary files.**
  - **Cause:** The files were introduced in an earlier layer and deleted in a later layer, so their original contents remain in the earlier layer.
  - **Fix:** Avoid adding unnecessary files, remove temporary files in the same `RUN` instruction that creates them, and use multi-stage builds when appropriate.

- **`--no-cache` does not download a newer base image.**
  - **Cause:** Disabling the build cache does not itself force a fresh pull of the base image.
  - **Fix:** Use `docker build --pull --no-cache ...` when you want both a cache-free build and a check for a newer base image.

- **A new container does not contain files created in another container.**
  - **Cause:** Each container has its own writable layer, even when both use the same image.
  - **Fix:** Put shared or persistent files in a volume or external storage service.

- **A container change is lost when the container is removed.**
  - **Cause:** The change was stored in the container's writable layer.
  - **Fix:** Rebuild the image for application changes or use persistent storage for data that must survive container replacement.

- **Docker reports that an image is up to date, but the application still uses old code.**
  - **Cause:** An existing container may still use the previously created image, or the image tag may resolve to unexpected content.
  - **Fix:** Verify the image ID and digest, rebuild or pull the intended image, and recreate the container. Updating a local image does not automatically replace an existing container.

- **A cache mount does not improve build speed.**
  - **Cause:** The cache may be empty, the package manager may not use the mounted directory, or the build is running on a fresh builder without a retained cache.
  - **Fix:** Verify the cache mount path and builder configuration. Remember that a cache mount is different from the normal Dockerfile instruction cache.

- **Build cache is missing on a CI runner.**
  - **Cause:** The runner uses a fresh environment or a different builder whose cache is not retained.
  - **Fix:** Configure a supported BuildKit cache import/export workflow or use a persistent builder where appropriate.

- **A cached build uses an unexpected dependency version.**
  - **Cause:** Dependencies are not pinned adequately, or the build inputs do not capture a changing external dependency source.
  - **Fix:** Pin dependencies and use lockfiles where available. Design builds to make external inputs explicit, and invalidate or refresh the relevant cache when appropriate.

## References

- [Docker Docs: About images](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)
- [Docker Docs: About containers](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Docker Docs: Build cache](https://docs.docker.com/build/cache/)
- [Docker Docs: Cache optimization](https://docs.docker.com/build/cache/optimize/)
- [Docker Docs: Cache storage backends](https://docs.docker.com/build/cache/backends/)
- [Docker Docs: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Kubernetes Docs: Images](https://kubernetes.io/docs/concepts/containers/images/)
