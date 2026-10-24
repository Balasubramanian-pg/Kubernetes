# Multi-stage builds

> Phase 1 — Linux, Docker & Networking › Containers

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **multi-stage build** is a Docker image-building technique that uses multiple `FROM` instructions in a single Dockerfile. Each `FROM` starts a new build stage, allowing different stages to use different base images and contain different tools, dependencies, and files.

The main purpose is to separate the environment required to **build an application** from the environment required to **run it**.

For example, compiling a Java, Go, or C++ application may require compilers, development libraries, build tools, and source code. The final application may need only the compiled binary or packaged artifact and a suitable runtime.

Without multi-stage builds, all these tools may end up in the final image. With multi-stage builds, you can compile the application in one stage and copy only the required output into a smaller final stage.

### How multi-stage builds work

```text
Source code
    |
    v
Build stage
    |
    | Compiler, SDK, build tools
    v
Compiled application
    |
    | COPY --from=build
    v
Final stage
    |
    | Runtime + application artifact
    v
Production image
    |
    v
Running container
```

The build stage can contain temporary files, source code, compilers, and development dependencies. The final stage receives only the files explicitly copied into it, along with anything already provided by its base image.

This reduces the final image's contents and can reduce its size, attack surface, and deployment transfer time.

### Example: Go application

```dockerfile
# Stage 1: Build the application
FROM golang:1.24 AS build

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

# Stage 2: Run the application
FROM scratch

COPY --from=build /out/app /app

ENTRYPOINT ["/app"]
```

The first stage contains the Go compiler and development environment. The second stage starts from `scratch`, an empty base image, and copies in the compiled binary.

This example assumes the project has a valid Go module, builds successfully as a standalone executable, and does not require additional runtime files or shared libraries. Applications that require certificates, time-zone data, shared libraries, or other operating-system files need those dependencies included in the final stage.

### Example: Python application

Python applications usually need a Python runtime in the final image, so the final stage is not necessarily empty.

```dockerfile
# Stage 1: Install dependencies
FROM python:3.12-slim AS build

WORKDIR /build

COPY requirements.txt .

RUN pip install \
    --no-cache-dir \
    --prefix=/install \
    -r requirements.txt

# Stage 2: Runtime
FROM python:3.12-slim AS runtime

WORKDIR /app

COPY --from=build /install /usr/local

COPY app.py .

CMD ["python", "app.py"]
```

The first stage installs the Python dependencies. The second stage copies the installed dependencies and application code into a clean Python runtime image.

This pattern works for many pure-Python packages. Packages containing native extensions may require shared libraries in the final stage, and build dependencies must be compatible with the runtime environment.

## Key points

- **Multiple `FROM` instructions define separate stages.** Each `FROM` starts a new stage with its own base image and build environment.

  ```dockerfile
  FROM node:22 AS build
  FROM nginx:alpine AS runtime
  ```

  The first stage can build a frontend application, while the second serves the generated static files.

- **Stage names make references readable.** Assign names with `AS`, such as `AS build` or `AS runtime`, and reference them using `COPY --from=build`.

  ```dockerfile
  COPY --from=build /app/dist /usr/share/nginx/html
  ```

  Naming stages is easier to maintain than referring to them by numeric index.

- **`COPY --from` transfers artifacts between stages.** It copies files from another build stage, an external image, or a supported image source into the current stage.

  ```dockerfile
  COPY --from=build /out/app /usr/local/bin/app
  ```

  Only the specified files and directories are copied. The source stage's entire filesystem is not included automatically.

- **Only the final stage is the default output.** In a typical multi-stage Dockerfile, the last stage becomes the final image when you run `docker build` without specifying a target.

- **Intermediate stages do not automatically enter the final image.** A compiler installed in the build stage is excluded unless its files are copied into the final stage or are otherwise required by the final base image.

- **Build stages can use different base images.** For example, a Node.js image can compile frontend assets and an NGINX image can serve them.

  ```dockerfile
  FROM node:22 AS build
  # Build frontend assets

  FROM nginx:alpine AS runtime
  # Copy the generated assets
  ```

- **Multi-stage builds reduce unnecessary dependencies.** Build tools, test utilities, source files, and development packages can be kept out of the production image.

- **A smaller image is not guaranteed in every case.** The final image may still be large if the application requires substantial runtime dependencies or if a large base image is selected.

- **Multi-stage builds can improve security.** Excluding unnecessary tools and files reduces the software included in the runtime image. It does not automatically make the application secure, and secrets must not be copied into image layers.

- **Build and runtime dependencies are different.** A compiler may be required to build an application but not to execute it. Shared libraries, runtime packages, certificates, and configuration may still be required at runtime.

- **The runtime stage must be compatible with the build output.** The operating system, CPU architecture, runtime ABI, shared libraries, and executable format must be appropriate for the final image.

- **Multi-stage builds work with build caching.** Docker can reuse cached build steps when their inputs remain unchanged. Keeping dependency installation separate from frequently changing source code can improve rebuild speed.

- **You can build a specific stage.** Use `--target` to select an intermediate stage, which is useful for debugging, testing, or creating development images.

  ```bash
  docker build --target build -t myapp:build .
  ```

- **Stages can be used for testing and production.** A Dockerfile can contain separate stages for compiling, linting, testing, and packaging. The final production image can include only the required runtime artifacts.

- **Multi-stage builds are not the same as running multiple containers.** The stages are part of the image build process. Containers are created later from the resulting image.

- **Multi-stage builds do not replace volumes or external storage.** They optimize image construction and packaging. Runtime data that must persist across container replacement still belongs in persistent storage.

- **BuildKit can optimize stage execution.** Modern Docker builds commonly use BuildKit, which can execute independent stages efficiently and supports additional caching and build features.

## Commands / manifests

### 1. Create a sample Go application

Create a directory named `multi-stage-demo`.

Create `go.mod`:

```go
module example.com/multi-stage-demo

go 1.24
```

Create `main.go`:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello from a multi-stage Docker build")
}
```

This application has no third-party dependencies.

### 2. Create the multi-stage Dockerfile

Create a file named `Dockerfile`:

```dockerfile
# Stage 1: Build
FROM golang:1.24 AS build

WORKDIR /src

COPY go.mod ./
COPY main.go ./

RUN CGO_ENABLED=0 go build -o /out/app .

# Stage 2: Runtime
FROM scratch

COPY --from=build /out/app /app

ENTRYPOINT ["/app"]
```

The build stage compiles the application. The runtime stage copies only the executable into an empty base image.

### 3. Build the image

Run the following command from the `multi-stage-demo` directory:

```bash
docker build -t multi-stage-demo:1.0 .
```

List the resulting image:

```bash
docker image ls multi-stage-demo
```

Inspect its configuration:

```bash
docker image inspect multi-stage-demo:1.0
```

### 4. Run the container

```bash
docker run --rm multi-stage-demo:1.0
```

Expected output:

```text
Hello from a multi-stage Docker build
```

Because the program exits after printing the message, the container exits too. The `--rm` option removes the container after it exits, while the image remains available.

### 5. Build only a specific stage

Build the intermediate stage as the target:

```bash
docker build \
  --target build \
  -t multi-stage-demo:build .
```

Inspect the image:

```bash
docker image inspect multi-stage-demo:build
```

The target image includes the Go build environment and the files available in that stage. It is useful for debugging or development but is not the minimal runtime image.

### 6. Compare the build and runtime images

```bash
docker image ls multi-stage-demo
```

Inspect the image history:

```bash
docker history multi-stage-demo:1.0
docker history multi-stage-demo:build
```

The build image contains the Go toolchain and build-stage filesystem. The final image contains only the copied executable and its image metadata because it uses `scratch`.

Actual image sizes depend on the application, base images, build outputs, and image format.

### 7. Build a frontend and serve it with NGINX

A common multi-stage pattern is to build static frontend assets using Node.js and serve them using NGINX.

Example Dockerfile:

```dockerfile
# Stage 1: Build frontend assets
FROM node:22 AS build

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 2: Serve the static files
FROM nginx:alpine AS runtime

COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
```

Build the image:

```bash
docker build -t frontend-app:1.0 .
```

Run it:

```bash
docker run -d \
  --name frontend-app \
  -p 8080:80 \
  frontend-app:1.0
```

Test it:

```bash
curl http://localhost:8080
```

This example assumes the frontend build command generates static assets in `/app/dist`. Some frameworks use a different output directory, which must be reflected in the `COPY --from=build` instruction.

### 8. Use a test stage

A Dockerfile can include a stage dedicated to testing.

```dockerfile
FROM python:3.12-slim AS base

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

FROM base AS test

RUN python -m compileall -q .

FROM base AS runtime

CMD ["python", "app.py"]
```

Build the test stage:

```bash
docker build --target test -t myapp:test .
```

Build the runtime stage:

```bash
docker build --target runtime -t myapp:runtime .
```

This is a simplified example. A real test stage should run the project's actual test suite and ensure that required test dependencies are installed. The runtime stage can be separated further to exclude test-only tools and files.

### 9. Inspect the final image's contents

Run a shell in a shell-capable image:

```bash
docker run --rm -it python:3.12-slim sh
```

For the Go example based on `scratch`, there is no shell available by default. You cannot use `docker exec` to open a shell inside that image unless you deliberately include one or use a separate debugging approach.

This illustrates a trade-off: minimal runtime images can reduce unnecessary components, but they may be less convenient to debug.

### 10. Clean up

Remove the running frontend container:

```bash
docker rm -f frontend-app
```

Remove example images when they are no longer needed:

```bash
docker image rm multi-stage-demo:1.0
docker image rm multi-stage-demo:build
docker image rm frontend-app:1.0
```

Docker will report an error if an image is still referenced by a container or if the specified image does not exist.

## Common failures & fixes

- **`COPY --from=build` reports that the source path does not exist.**
  - **Cause:** The build stage did not generate the expected artifact, or the source path is incorrect.
  - **Fix:** Verify the build command and output directory. For example, check that `go build -o /out/app .` actually creates `/out/app` before copying it.

- **The final container fails with `exec format error`.**
  - **Cause:** The executable was built for a different CPU architecture or in an incompatible executable format.
  - **Fix:** Build for the target platform and verify the binary architecture. Use BuildKit's platform support when producing images for multiple architectures.

- **The final image cannot find a required shared library.**
  - **Cause:** The executable depends on a shared library available in the build stage but absent from the final stage.
  - **Fix:** Include the required runtime libraries in the final image, choose a compatible runtime base image, or build a suitable self-contained executable when possible.

- **A `scratch`-based image fails because a file is missing.**
  - **Cause:** `scratch` provides no operating-system filesystem, shell, certificates, or other runtime files by default.
  - **Fix:** Copy all required runtime artifacts into the final image or use a minimal base image that provides the necessary components.

- **The final image is not significantly smaller.**
  - **Cause:** The runtime stage uses a large base image, copies unnecessary build artifacts, or needs substantial runtime dependencies.
  - **Fix:** Use an appropriate runtime base image and copy only required files. Compare image sizes and inspect image history before assuming the build is optimized.

- **Build dependencies are missing in the final image.**
  - **Cause:** A package was installed only in the build stage, but the application requires it at runtime.
  - **Fix:** Distinguish build-time dependencies from runtime dependencies. Install or copy the runtime requirements into the final stage.

- **The build stage is rebuilt every time.**
  - **Cause:** Frequently changing files are copied before dependency installation, invalidating cached steps.
  - **Fix:** Copy dependency manifests first, install dependencies, and copy the remaining source code afterward. Use supported cache mounts or external caches for CI when appropriate.

- **A test stage is skipped during production builds.**
  - **Cause:** The production target does not depend on the test stage, so Docker may not execute it when building that target.
  - **Fix:** Run the test target explicitly in CI before building or deploying the production image, or structure the build pipeline to enforce the required test dependency.

- **A build succeeds but the container fails at runtime.**
  - **Cause:** The final stage is missing configuration, runtime libraries, certificates, or required files.
  - **Fix:** Test the final image itself, not only the build stage. Verify its entrypoint, permissions, environment variables, and runtime dependencies.

- **A secret appears in the final image or build artifacts.**
  - **Cause:** The secret was copied into an image layer or exposed through build instructions.
  - **Fix:** Remove secrets from the build context, use supported BuildKit secret mounts for build-time credentials, and provide runtime secrets through an appropriate secret-management mechanism. Multi-stage builds alone do not guarantee that secrets are excluded from every image or build artifact.

## References

- [Docker Docs: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Docs: Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Docker Docs: Build cache optimization](https://docs.docker.com/build/cache/optimize/)
- [Docker Docs: Image layers](https://docs.docker.com/get-started/docker-concepts/building-images/understanding-image-layers/)
- [Docker Docs: Container lifecycle](https://docs.docker.com/engine/containers/run/)
- [Docker Buildx documentation](https://docs.docker.com/build/buildx/)
- [Kubernetes Docs: Images](https://kubernetes.io/docs/concepts/containers/images/)
