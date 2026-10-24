# Dockerfiles

> Phase 1 — Linux, Docker & Networking › Containers

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **Dockerfile** is a text file containing instructions that Docker uses to build a container image. It defines the base image, application files, dependencies, build-time operations, environment settings, and default command used when a container starts.

Instead of manually installing dependencies and configuring an application every time you create a container, you describe the process in a Dockerfile and build a repeatable image.

For example, a data engineering application might require Python, PySpark dependencies, application scripts, and a specific startup command. A Dockerfile packages the required environment so the application can run consistently across development machines, CI/CD pipelines, and deployment environments.

### How a Dockerfile works

```text
Dockerfile
    |
    | docker build
    v
Docker Build Engine
    |
    | Executes build instructions
    v
Container Image
    |
    | docker run
    v
Container
    |
    | Executes CMD / ENTRYPOINT
    v
Running Application
```

A Dockerfile contains instructions such as:

- `FROM`: Selects the base image.
- `WORKDIR`: Sets the working directory for subsequent instructions.
- `COPY`: Copies files from the build context into the image.
- `RUN`: Executes commands during the image build.
- `ENV`: Defines environment variables in the image configuration.
- `EXPOSE`: Documents the port the application is intended to use.
- `CMD`: Defines the default command or arguments used when the container starts.
- `ENTRYPOINT`: Defines the executable intended to run when the container starts.

These instructions have different purposes. Some contribute filesystem changes to the image, while others primarily configure the resulting image or its runtime behavior.

### Example Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

ENV APP_ENV=production

EXPOSE 5000

CMD ["python", "app.py"]
```

Explanation:

1. `FROM` selects a Python base image.
2. `WORKDIR` establishes `/app` as the working directory.
3. `COPY requirements.txt .` copies the dependency file into the image.
4. `RUN` installs the Python dependencies during the build.
5. `COPY app.py .` adds the application code.
6. `ENV` defines a default environment variable.
7. `EXPOSE` documents the intended application port.
8. `CMD` specifies the default process to start when the container runs.

The Dockerfile is not itself the container. It is the build definition used to create an image, from which containers can be created.

## Key points

- **Dockerfiles make builds repeatable.** They capture the steps required to package an application. A team can rebuild the image without manually repeating every installation and configuration step.

- **`FROM` defines the base image.** Every Dockerfile starts with a base image selection in the normal case, unless it uses a supported alternative such as `FROM scratch`. Choose a base image that provides the required runtime while minimizing unnecessary dependencies.

  ```dockerfile
  FROM python:3.12-slim
  ```

  This starts from a Python image based on a slim Linux distribution. The base image already contains its filesystem and Python runtime.

- **`RUN` executes commands during the build.** It is commonly used to install packages, compile software, and prepare files.

  ```dockerfile
  RUN apt-get update && apt-get install -y curl
  ```

  Prefer grouping related package-manager operations so that the package index and installation occur together. Remove unnecessary package-manager caches in the same instruction when appropriate. Pin dependencies where reproducibility matters.

- **`COPY` adds files from the build context.** It copies files and directories from the build context into the image.

  ```dockerfile
  COPY app.py /app/app.py
  ```

  The source path is relative to the build context, not necessarily relative to the directory from which the command is executed. A `.dockerignore` file can exclude unwanted files from the context.

- **`WORKDIR` establishes the working directory.** If the directory does not exist, Docker normally creates it.

  ```dockerfile
  WORKDIR /app
  ```

  Subsequent relative paths and commands generally use this directory unless another working directory is specified.

- **`CMD` specifies the default runtime command.** It is executed when a container starts unless overridden by the runtime's command configuration.

  ```dockerfile
  CMD ["python", "app.py"]
  ```

  The JSON-array form is called exec form. It runs the executable directly rather than invoking a shell implicitly.

- **`ENTRYPOINT` defines the main executable.** It is useful when the image is designed to behave like a specific application or utility.

  ```dockerfile
  ENTRYPOINT ["python", "app.py"]
  ```

  With exec form, arguments supplied through Docker's runtime command are normally appended to the entrypoint. The entrypoint can be overridden with `--entrypoint`.

- **`CMD` and `ENTRYPOINT` can be combined.** A common pattern is to define the executable with `ENTRYPOINT` and its default arguments with `CMD`.

  ```dockerfile
  ENTRYPOINT ["python", "app.py"]
  CMD ["--port", "5000"]
  ```

  Running the container without additional arguments starts `python app.py --port 5000`. Supplying another runtime command replaces the default `CMD` arguments while retaining the exec-form entrypoint.

- **Prefer exec form for application processes.** Use JSON-array syntax for `CMD` and `ENTRYPOINT` when possible. The application can receive operating-system signals directly, which helps with graceful shutdown.

  ```dockerfile
  CMD ["python", "app.py"]
  ```

  Shell form may run the command through a shell, which can affect signal handling and argument processing.

- **`EXPOSE` does not publish a port.** It documents the container port the application is intended to listen on.

  ```dockerfile
  EXPOSE 5000
  ```

  To make a port accessible through Docker's host-port mapping, use a runtime option such as `-p 8080:5000`.

- **`ENV` defines default environment variables.** These variables are available to later build instructions and to containers created from the image unless overridden.

  ```dockerfile
  ENV APP_ENV=production
  ```

  Do not use `ENV` to embed passwords, API keys, or other secrets into the image. Image metadata and layers can expose such values.

- **Build-time arguments differ from runtime environment variables.** `ARG` defines a build-time argument, while `ENV` defines an environment variable in the image configuration.

  ```dockerfile
  ARG APP_VERSION=1.0
  ENV APP_ENV=production
  ```

  Build arguments can be supplied with `--build-arg`. They are not a secure mechanism for passing secrets because values can be exposed through build metadata, logs, or other artifacts.

- **Instruction order affects caching.** Put stable dependency installation steps before frequently changing application files when possible.

  ```dockerfile
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY . .
  ```

  This allows Docker to reuse the dependency installation result when only application source files change.

- **Use `.dockerignore`.** Exclude files that are not needed during the build, such as `.git`, local virtual environments, caches, and local configuration files.

  ```text
  .git
  .venv
  __pycache__
  *.pyc
  .env
  ```

  Excluding `.env` from the build context helps prevent accidental inclusion, but secrets should also be handled through secure runtime configuration or supported build-secret mechanisms.

- **Use multi-stage builds when appropriate.** A build stage can contain compilers and development tools, while the final stage contains only the runtime and required artifacts.

  ```dockerfile
  FROM golang:1.24 AS build
  WORKDIR /src
  COPY . .
  RUN go build -o /out/app .

  FROM debian:bookworm-slim
  COPY --from=build /out/app /usr/local/bin/app
  ENTRYPOINT ["/usr/local/bin/app"]
  ```

  The example illustrates the pattern. Use a supported Go version and the appropriate dependencies for the actual project.

- **Run applications as non-root where possible.** Containers can provide process isolation, but running the application as an unprivileged user reduces the impact of some compromises.

  ```dockerfile
  RUN useradd --create-home appuser
  USER appuser
  ```

  Ensure the application can read and write the required directories before switching users.

- **A Dockerfile does not guarantee application correctness.** The image can build successfully while the application fails at runtime due to missing configuration, unavailable services, incorrect ports, permissions, or an invalid startup command.

## Commands / manifests

### 1. Create a simple Python application

Create a directory named `dockerfile-demo` containing these files.

`app.py`:

```python
import os
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        environment = os.getenv("APP_ENV", "development")
        body = f"Application running in {environment} mode\n".encode()

        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

server = HTTPServer(("0.0.0.0", 5000), Handler)
server.serve_forever()
```

This example uses Python's standard library, so no `requirements.txt` is needed.

### 2. Write the Dockerfile

Create a file named `Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY app.py .

ENV APP_ENV=production

EXPOSE 5000

CMD ["python", "app.py"]
```

### 3. Build the image

Run the following command from the `dockerfile-demo` directory:

```bash
docker build -t dockerfile-demo:1.0 .
```

Explanation:

- `docker build` starts the image build.
- `-t dockerfile-demo:1.0` assigns a repository name and tag.
- `.` specifies the current directory as the build context.

Verify that the image exists:

```bash
docker image ls dockerfile-demo
```

### 4. Run a container from the image

```bash
docker run -d \
  --name dockerfile-demo \
  -p 8080:5000 \
  dockerfile-demo:1.0
```

Test the application:

```bash
curl http://localhost:8080
```

Expected response:

```text
Application running in production mode
```

The application listens on port `5000` inside the container. Docker maps host port `8080` to that container port.

### 5. Override environment variables

```bash
docker run --rm \
  -e APP_ENV=testing \
  -p 8081:5000 \
  dockerfile-demo:1.0
```

In another terminal, run:

```bash
curl http://localhost:8081
```

Expected response:

```text
Application running in testing mode
```

The runtime value overrides the default `ENV APP_ENV=production` specified in the Dockerfile.

### 6. Understand `CMD` and command overrides

Run the container with its default command:

```bash
docker run --rm dockerfile-demo:1.0
```

The Python HTTP server starts and keeps the container running.

Override the default command:

```bash
docker run --rm dockerfile-demo:1.0 python --version
```

This replaces the image's default `CMD` with `python --version`. The container prints the Python version and exits.

### 7. Compare `CMD` and `ENTRYPOINT`

Create `Dockerfile.entrypoint`:

```dockerfile
FROM python:3.12-slim

ENTRYPOINT ["python", "--version"]
```

Build it:

```bash
docker build -f Dockerfile.entrypoint -t python-version-demo .
```

Run it:

```bash
docker run --rm python-version-demo
```

Override the entrypoint:

```bash
docker run --rm \
  --entrypoint python \
  python-version-demo -c "print('Hello from Python')"
```

The second command replaces the configured entrypoint with `python` and supplies the Python expression as its argument.

### 8. Inspect the image configuration

```bash
docker image inspect dockerfile-demo:1.0
```

Display selected fields:

```bash
docker image inspect \
  --format '{{.Config.WorkingDir}}' \
  dockerfile-demo:1.0

docker image inspect \
  --format '{{json .Config.Cmd}}' \
  dockerfile-demo:1.0

docker image inspect \
  --format '{{json .Config.Env}}' \
  dockerfile-demo:1.0
```

These commands inspect the configured working directory, default command, and environment variables.

### 9. Inspect build history

```bash
docker history dockerfile-demo:1.0
```

This displays the image's build history and associated layer information. Some Dockerfile instructions affect image configuration rather than adding filesystem layers.

### 10. Clean up

Stop the running container:

```bash
docker stop dockerfile-demo
```

Remove it:

```bash
docker rm dockerfile-demo
```

Remove the image when it is no longer needed:

```bash
docker image rm dockerfile-demo:1.0
```

Removing a container does not automatically remove its image. Other containers or image tags may also prevent image deletion.

## Common failures & fixes

- **`docker build` reports that it cannot find the Dockerfile.**
  - **Cause:** The build command is running from the wrong directory or the file has a different name.
  - **Fix:** Verify the current directory and use `docker build -f PATH_TO_DOCKERFILE -t IMAGE_NAME .` when the Dockerfile is elsewhere.

- **`COPY` reports that the source file cannot be found.**
  - **Cause:** The source path is outside the build context, is incorrect, or is excluded by `.dockerignore`.
  - **Fix:** Check the build context, file path, and `.dockerignore` rules. Docker generally cannot copy arbitrary files from outside the build context.

- **A dependency installation step fails.**
  - **Cause:** The dependency file is missing, the package version is unavailable, or the build environment cannot reach the package repository.
  - **Fix:** Verify the dependency file and package versions, check network access, and inspect the build output.

- **The container exits immediately.**
  - **Cause:** The command finishes, the application crashes, or the configured `CMD` or `ENTRYPOINT` is invalid.
  - **Fix:** Use `docker ps -a`, `docker logs CONTAINER_NAME`, and `docker inspect CONTAINER_NAME` to identify the failure. Ensure the container's main application process stays active when intended.

- **The application starts but cannot be reached from the host.**
  - **Cause:** The application listens on the wrong interface or port, or the port mapping is missing or incorrect.
  - **Fix:** Ensure the application listens on `0.0.0.0` inside the container, verify the container port, and publish it with `-p HOST_PORT:CONTAINER_PORT` when needed.

- **Changing application code does not appear in the container.**
  - **Cause:** The image was not rebuilt, or an existing container is still running the previous image.
  - **Fix:** Rebuild the image and recreate the container. Building a new image does not automatically replace a running container.

- **Environment variables have unexpected values.**
  - **Cause:** Runtime options override image defaults, or the application reads a different variable name.
  - **Fix:** Inspect the container configuration with `docker inspect` and verify the variable names and runtime options.

- **A container fails after switching to a non-root user.**
  - **Cause:** The user does not have permission to read application files, write temporary files, or access required directories.
  - **Fix:** Set correct file ownership and permissions before switching users. Grant only the access the application needs.

- **The image is unexpectedly large.**
  - **Cause:** The base image contains unnecessary packages, build tools are included in the final image, or temporary files remain in image layers.
  - **Fix:** Use an appropriate minimal base image, apply `.dockerignore`, clean up temporary files in the same build step, and consider multi-stage builds.

- **A build unexpectedly reuses old results.**
  - **Cause:** Docker reused cached results because the relevant inputs appeared unchanged.
  - **Fix:** Check the Dockerfile instruction order and build context. Use `docker build --no-cache -t IMAGE_NAME .` when you need to disable normal build-cache reuse.

## References

- [Docker Docs: Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Docker Docs: Building images](https://docs.docker.com/build/concepts/dockerfile/)
- [Docker Docs: Build cache optimization](https://docs.docker.com/build/cache/optimize/)
- [Docker Docs: Image layers](https://docs.docker.com/get-started/docker-concepts/building-images/understanding-image-layers/)
- [Docker Docs: Container lifecycle](https://docs.docker.com/engine/containers/run/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Kubernetes Docs: Images](https://kubernetes.io/docs/concepts/containers/images/)
