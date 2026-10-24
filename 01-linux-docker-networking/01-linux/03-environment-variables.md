# Environment variables

> Phase 1 — Linux, Docker & Networking › Linux

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Environment variables are named key-value pairs that processes use to access configuration and runtime information without hardcoding those values into application code. They commonly store executable paths, configuration settings, environment names, feature flags, and references to credentials.

For example, `PATH` tells Linux which directories to search when locating executable commands, while `HOME` identifies a user's home directory. An application might read `APP_ENV=development` to determine which configuration to load.

Environment variables are inherited through the process hierarchy. When a process starts a child process, the child generally receives a copy of the parent's environment. Changes made by the child do not modify the parent's environment or automatically propagate to other processes.

Environment variables are especially important in Docker and Kubernetes because they allow the same application image to run across development, testing, and production environments with different configuration values.

## Key points

- **Variables versus environment variables:** A shell variable exists within the current shell. An environment variable is exported so that child processes can inherit it. For example, `APP_ENV=dev` creates a shell variable, while `export APP_ENV=dev` makes it available to subsequently launched child processes.

- **Reading variables:** Use `$VARIABLE_NAME` or `${VARIABLE_NAME}` to expand a variable in the shell. Braces make boundaries explicit, as in `${APP_ENV}_config`.

- **Setting variables temporarily:** Prefixing a command with a variable assignment, such as `APP_ENV=test ./app`, sets the variable for that command's environment without permanently changing the current shell variable.

- **Variable scope:** Variables set in one terminal session are generally not available in another independent session. Child processes inherit the environment at launch, but changes do not flow back to the parent.

- **Exporting variables:** `export` marks a shell variable for inclusion in the environment of child processes. Exporting a variable does not make it globally available to every process on the system.

- **Removing variables:** `unset VARIABLE_NAME` removes a shell variable and its export attribute from the current shell. It does not retroactively change environments of already-running processes.

- **Default values:** Shell parameter expansion can supply defaults. `${APP_ENV:-development}` uses `development` if the variable is unset or empty, while `${APP_ENV-development}` uses the default only when it is unset.

- **Persistent configuration:** Shell startup files such as `~/.bashrc`, `~/.profile`, or `/etc/environment` may configure values for future sessions. The correct file depends on the shell, login mechanism, and operating system. Editing a startup file does not automatically update already-running processes.

- **The `PATH` variable:** `PATH` contains colon-separated directories searched for executables. An incorrect `PATH` can make commands appear unavailable. Use `command -v python` or `command -v kubectl` to identify the executable the shell resolves.

- **Environment variables are strings:** Values such as `PORT=8080` are usually passed as text. Applications must parse and validate them into the required data types.

- **Environment variables are not a complete configuration system:** They are useful for runtime configuration but do not inherently provide validation, type safety, or secret management.

- **Security and secrets:** Environment variables can expose sensitive values through process inspection, debugging, logs, crash reports, or diagnostic tools, depending on the operating system and execution context. Avoid printing secrets, committing them to source control, or embedding them in container images. Prefer a dedicated secret-management mechanism where appropriate.

- **Docker environment:** Docker can inject environment variables using the `-e` option, an environment file, or Dockerfile `ENV`. Values set with Docker's `ENV` instruction are part of the image configuration and may be visible to users who can inspect the image. Do not bake credentials into an image.

- **Kubernetes environment:** Kubernetes containers can receive variables through `env` and `envFrom`, using literal values, ConfigMaps, or Secrets. ConfigMaps are intended for non-confidential configuration; Secrets are intended for sensitive data but require appropriate access controls and cluster security.

- **Configuration precedence:** If the same setting is defined in several places, the application, shell, container runtime, or orchestration platform determines which value takes effect. Understand the actual configuration path instead of assuming one universal precedence rule.

- **Environment variables versus command-line arguments:** Environment variables are convenient for configuration shared across an application's process tree. Command-line arguments are useful for explicit per-invocation options, but both may be visible to process-inspection tools in some environments.

## Commands / manifests

```bash
# 1. Display a variable
echo "$HOME"
echo "$PATH"
printf '%s\n' "$USER"

# 2. Create a shell variable
APP_ENV=development
echo "$APP_ENV"

# 3. Export a variable to child processes
export APP_ENV=development
bash -c 'printf "APP_ENV=%s\n" "$APP_ENV"'

# 4. Demonstrate that an unexported variable is not inherited
LOCAL_SETTING=hello
bash -c 'echo "LOCAL_SETTING=${LOCAL_SETTING-unset}"'

# 5. Set a variable for one command only
APP_ENV=testing bash -c 'echo "$APP_ENV"'
echo "${APP_ENV-unset}"

# 6. Use braces and default values
APP_NAME=myapp
echo "${APP_NAME}_config"
echo "${MISSING_VAR:-default_value}"

# 7. List environment variables
printenv
printenv HOME
env

# 8. Remove a variable
export TEMP_SETTING=temporary
unset TEMP_SETTING
echo "${TEMP_SETTING-unset}"

# 9. Inspect command resolution and PATH
printf '%s\n' "$PATH"
command -v bash
command -v python
command -v kubectl

# 10. Add a directory to PATH for the current shell
export PATH="$HOME/bin:$PATH"

# 11. Set a variable for a child process and inspect it
APP_ENV=production bash -c 'env | grep "^APP_ENV="'

# 12. Inspect a process environment where permissions allow
# Replace <PID> with a process ID
tr '\0' '\n' < /proc/<PID>/environ
```

The `/proc/<PID>/environ` example is Linux-specific and may require suitable permissions. It exposes environment values, so do not inspect another process's environment without authorization.

### Persistent environment variables

For Bash, add a setting to `~/.bashrc` for interactive non-login shells:

```bash
# Add to ~/.bashrc
export APP_ENV=development
export PATH="$HOME/bin:$PATH"

# Reload the file in the current shell
source ~/.bashrc
```

For login environments, `~/.profile` may be more appropriate, depending on how the session starts. Use the startup file that matches your shell and login configuration.

### Docker example

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY app.py .

# Suitable for non-sensitive default configuration
ENV APP_ENV=production

CMD ["python", "app.py"]
```

Run the container with a different value:

```bash
# Override the image's default environment value
docker run --rm -e APP_ENV=development my-python-app

# Load variables from a local file
docker run --rm --env-file .env my-python-app

# Inspect the configured environment of a container
docker inspect CONTAINER_NAME --format '{{json .Config.Env}}'

# Display selected variables inside a running container
docker exec CONTAINER_NAME printenv APP_ENV
```

An environment file might contain:

```text
APP_ENV=development
LOG_LEVEL=debug
PORT=8000
```

Use `--env-file` for configuration that belongs in the environment, and keep files containing credentials out of source control. The `docker inspect` command can reveal configured environment values, including sensitive ones if you put them there.

### Kubernetes example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "printenv APP_ENV; printenv LOG_LEVEL; sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
```

Apply and inspect the example:

```bash
kubectl apply -f env-demo.yaml
kubectl get pod env-demo
kubectl exec env-demo -- printenv APP_ENV
kubectl exec env-demo -- printenv LOG_LEVEL
kubectl delete pod env-demo
```

You can also define an individual variable using `env`:

```yaml
env:
  - name: APP_ENV
    value: "staging"
```

Or reference a specific ConfigMap key:

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
```

For sensitive values, use a Kubernetes Secret reference rather than placing the credential directly in the manifest:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: DB_PASSWORD
```

The referenced Secret and key must exist. Kubernetes Secret data is not automatically safe simply because it is stored in a Secret object; configure suitable RBAC, encryption at rest, and secret-handling practices.

## Common failures & fixes

- **A variable prints as an empty string in a child process:** The variable may exist only in the shell and not be exported. Use `export VARIABLE_NAME=value` before starting the child process.

- **A variable works in one terminal but not another:** The value may have been set only in the original session. Configure the appropriate shell startup file or set the variable in the command, service, container, or orchestration configuration that launches the application.

- **A variable defined in `~/.bashrc` is unavailable to a service:** Services often run outside interactive Bash sessions. Configure the service's environment explicitly using its service manager or deployment configuration rather than relying on interactive shell startup files.

- **A variable disappears after restarting the shell:** An interactive assignment is temporary. Put the setting in the correct startup file or configuration mechanism if it needs to persist.

- **A variable contains spaces or special characters:** Quote expansions, such as `"$APP_NAME"`, to prevent unwanted word splitting and pathname expansion. Use `printf '%s\n' "$VARIABLE"` for predictable output.

- **A variable is unexpectedly empty despite having a default:** `${VAR-default}` and `${VAR:-default}` have different behavior. The colon form also uses the default when the variable is empty.

- **Commands stop working after changing `PATH`:** The new value may have replaced the existing path. Prefer `export PATH="$HOME/bin:$PATH"` to prepend a directory while preserving the existing entries.

- **A Docker container receives the wrong configuration:** Check the image's `ENV`, runtime `-e` options, `--env-file`, Compose configuration if used, and the application's own configuration-loading logic. Inspect the effective environment inside the container without exposing credentials.

- **Updating a Kubernetes ConfigMap does not change an application's existing environment variable:** Environment variables are established when a container starts. Recreate or restart the relevant Pods to load updated values. ConfigMap volumes behave differently and can update over time, but the application may still need to reload the files.

- **A Kubernetes Pod fails to start because a ConfigMap or Secret is missing:** Verify the resource exists in the same namespace as the Pod, the referenced key is correct, and the manifest has been applied successfully.

- **A secret appears in logs or diagnostic output:** Remove the logging of sensitive variables, restrict access to logs and process inspection, and rotate exposed credentials when necessary. Avoid sharing raw environment dumps in tickets or chat.

- **An application reads the wrong value:** Inspect the environment in the actual process or container and check for duplicate settings, defaults, entrypoint scripts, and application-level overrides. The value in your terminal may differ from the value received by the deployed process.

## References

- [GNU Bash manual: Environment](https://www.gnu.org/software/bash/manual/html_node/Environment.html)
- [GNU Bash manual: Shell Parameters](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameters.html)
- [Linux `printenv` manual](https://www.gnu.org/software/coreutils/manual/html_node/printenv-invocation.html)
- [Docker: Environment variables](https://docs.docker.com/build/building/variables/)
- [Docker CLI: `docker run`](https://docs.docker.com/reference/cli/docker/container/run/)
- [Kubernetes: Define environment variables for a container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
- [Kubernetes: ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
