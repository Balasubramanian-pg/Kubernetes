# Environment-based configuration

> Phase 1 — Linux, Docker & Networking › Docker runtime

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Environment-based configuration means supplying application settings through environment variables instead of hardcoding them into application code or container images.

Environment variables are commonly used to configure:

- Database connection details
- API endpoints
- Application runtime modes such as `development`, `testing`, and `production`
- Logging levels
- Feature flags
- Service ports and timeouts

Docker provides two common ways to pass environment variables when starting a container:

- `-e` or `--env`: Pass individual variables directly through the `docker run` command.
- `--env-file`: Load multiple variables from a file.

Environment variables can also be defined in a Dockerfile using `ENV`, although values intended to change between deployments are generally better supplied at runtime.

**Example:** The same application image can connect to a development database in one environment and a production database in another without rebuilding the image.

## Key points

- **Build-time versus runtime configuration**
  - `ARG` defines build-time variables that can be used while building an image.
  - `ENV` defines environment variables in the image that are available to processes in the container.
  - `docker run -e` and `docker run --env-file` can supply or override environment variables when a container starts.
  - Runtime configuration allows the same image to be deployed across multiple environments.

- **Using `-e` and `--env`**
  - Both options specify environment variables for a container.
  - Assign a value explicitly using `-e APP_ENV=production`.
  - Pass through a variable from the host using `-e APP_ENV`, provided it is set in the host shell.
  - Repeat the option to supply multiple variables.

- **Using `--env-file`**
  - Stores multiple environment variables in a text file.
  - Each entry normally follows the `KEY=value` format.
  - Use `--env-file .env` to load the file when creating a container.
  - Docker does not automatically load a `.env` file just because it exists. You must explicitly pass it to `docker run`.
  - The file is read by the Docker client and its values are supplied to the container. It is not automatically mounted as a file inside the container.

- **Variable precedence**
  - For a variable defined in the image and also supplied at container creation, the runtime value takes precedence.
  - When the same variable is supplied through multiple runtime sources, an explicit `-e KEY=value` takes precedence over the corresponding value in `--env-file`.
  - A variable explicitly passed through `-e KEY` can inherit its value from the host environment.

- **Configuration versus secrets**
  - Environment variables are suitable for non-sensitive configuration such as application modes and service URLs.
  - Database passwords, API tokens, and other credentials require careful handling.
  - Environment variables can be exposed through container inspection, process diagnostics, application logs, or accidental error messages.
  - Do not commit real `.env` files containing credentials to source control.
  - For production deployments, prefer an appropriate secrets-management mechanism, such as a cloud secrets manager or orchestrator-managed secrets.

- **Environment variables are strings**
  - Values such as `false`, `8080`, and `3.14` are passed as strings.
  - The application must parse them into the correct types.
  - An unset variable and a variable set to an empty string are not necessarily equivalent.

- **Environment variables do not automatically configure the application**
  - Docker passes variables to the container process, but the application must read and use them.
  - The variable names and expected formats must match what the application supports.

## Commands / manifests

**1. Pass a variable directly using `-e`**

```bash
docker run --rm \
  -e APP_ENV=development \
  -e LOG_LEVEL=debug \
  alpine:3.22 \
  sh -c 'echo "Environment: $APP_ENV"; echo "Log level: $LOG_LEVEL"'
```

Explanation:

- `--rm` removes the container after it exits.
- `-e APP_ENV=development` sets the container's `APP_ENV` variable.
- `-e LOG_LEVEL=debug` sets another variable.
- The shell inside the container reads the variables and prints their values.

**2. Pass an existing host variable into a container**

Linux shell example:

```bash
export APP_ENV=testing

docker run --rm \
  -e APP_ENV \
  alpine:3.22 \
  sh -c 'echo "$APP_ENV"'
```

When `-e APP_ENV` has no explicit assignment, Docker passes the host's value for that variable if it exists in the client environment.

If the variable is not set on the host, do not rely on this form to provide a useful value. Supply an explicit assignment or validate the variable before starting the container.

**3. Create an environment file**

Create a file named `.env`:

```dotenv
APP_ENV=development
LOG_LEVEL=debug
APP_PORT=8080
```

Run a container using that file:

```bash
docker run --rm \
  --env-file .env \
  alpine:3.22 \
  sh -c 'echo "$APP_ENV"; echo "$LOG_LEVEL"; echo "$APP_PORT"'
```

Expected output:

```text
development
debug
8080
```

**4. Override a value from the environment file**

Suppose `.env` contains:

```dotenv
APP_ENV=development
LOG_LEVEL=debug
```

Run:

```bash
docker run --rm \
  --env-file .env \
  -e APP_ENV=production \
  alpine:3.22 \
  sh -c 'echo "$APP_ENV"; echo "$LOG_LEVEL"'
```

Expected output:

```text
production
debug
```

The explicit runtime assignment overrides `APP_ENV` from the file, while `LOG_LEVEL` retains its file-defined value.

**5. Inspect environment variables in a running container**

Start a container that remains running:

```bash
docker run -d \
  --name config-demo \
  -e APP_ENV=testing \
  alpine:3.22 \
  sh -c 'sleep 600'
```

Inspect the container configuration:

```bash
docker inspect \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  config-demo
```

Filter the displayed values if needed:

```bash
docker inspect \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  config-demo | grep '^APP_ENV='
```

Execute a command inside the container:

```bash
docker exec config-demo sh -c 'echo "$APP_ENV"'
```

Clean up:

```bash
docker rm -f config-demo
```

**Security note:** `docker inspect` can reveal environment variables, including credentials supplied this way. Avoid placing real secrets in diagnostic output.

**6. Define defaults in a Dockerfile**

```dockerfile
FROM alpine:3.22

ENV APP_ENV=production
ENV LOG_LEVEL=info

CMD ["sh", "-c", "echo \"Environment: $APP_ENV\"; echo \"Log level: $LOG_LEVEL\""]
```

Build and run:

```bash
docker build -t config-demo .
docker run --rm config-demo
```

Override the defaults at runtime:

```bash
docker run --rm \
  -e APP_ENV=development \
  config-demo
```

The image supplies defaults, while runtime configuration can override them.

**Important:** Do not use Dockerfile `ENV` or build arguments to embed credentials in an image. Values embedded in image metadata or build history may be discoverable.

**7. Use environment variables with Docker Compose**

Create a `compose.yaml` file:

```yaml
services:
  app:
    image: alpine:3.22
    environment:
      APP_ENV: ${APP_ENV:-development}
      LOG_LEVEL: ${LOG_LEVEL:-info}
    command:
      - sh
      - -c
      - 'echo "Environment: $$APP_ENV"; echo "Log level: $$LOG_LEVEL"'
```

Start the service:

```bash
docker compose up
```

Compose substitutes `${APP_ENV:-development}` using its interpolation rules and uses `development` as the default if `APP_ENV` is unset or empty.

Create a `.env` file beside `compose.yaml`:

```dotenv
APP_ENV=testing
LOG_LEVEL=debug
```

Run the service again:

```bash
docker compose up
```

The application receives `APP_ENV=testing` and `LOG_LEVEL=debug`.

There are two distinct concepts here:

- **Compose interpolation:** `${APP_ENV}` substitutes a value while Compose processes the configuration.
- **Container environment:** `environment:` passes the resulting values into the container.

A Compose `.env` file is primarily an input for interpolation. It does not automatically make every variable available inside every container. Explicitly map variables through `environment:` or use the service-level `env_file:` option.

The `$$APP_ENV` syntax prevents Compose from substituting the variable itself, allowing the shell inside the container to expand it instead.

To use a different interpolation file:

```bash
docker compose --env-file .env.testing up
```

To inspect the resolved Compose configuration:

```bash
docker compose config
```

Be careful when sharing this output because interpolated configuration may contain sensitive values.

**8. Keep local configuration out of Git**

Add the following to `.gitignore`:

```gitignore
.env
.env.*
!.env.example
```

Create a safe example file named `.env.example`:

```dotenv
APP_ENV=development
LOG_LEVEL=info
APP_PORT=8080
```

The example file documents the required configuration without including real credentials. Team members can copy it to `.env` and supply their own values.

## Common failures & fixes

- **The container cannot read a variable**
  - **Cause:** The variable was defined on the host but never passed into the container.
  - **Fix:** Use `-e KEY=value`, `-e KEY`, `--env-file`, or the appropriate Compose configuration.
  - **Verify:** Run `docker exec CONTAINER_NAME printenv KEY`.

- **The `--env-file` file cannot be found**
  - **Cause:** The path is incorrect or the command is being executed from a different working directory.
  - **Fix:** Use the correct relative path or an absolute path.
  - **Verify:** Check the file with `ls -l .env` before running the container.

- **A variable contains an unexpected value**
  - **Cause:** A runtime assignment, environment file, image default, or Compose interpolation value has overridden the expected value.
  - **Fix:** Check each configuration source and its precedence.
  - **Verify:** Use `docker inspect` to inspect the container's configured environment, without exposing secrets.

- **The application ignores the variable**
  - **Cause:** The application expects a different variable name, reads configuration from another source, or requires a restart.
  - **Fix:** Check the application's configuration documentation and ensure it reads the variable at startup.
  - **Verify:** Test with a known variable name and confirm the application uses the expected value.

- **A number or Boolean is interpreted incorrectly**
  - **Cause:** Environment variables are passed as strings.
  - **Fix:** Parse and validate the value in application code. For example, convert a port to an integer and parse Boolean values explicitly.
  - **Verify:** Test valid, missing, empty, and invalid inputs.

- **Compose substitutes an empty value**
  - **Cause:** A variable exists but is empty, or the expected `.env` file is not being used.
  - **Fix:** Use a default such as `${APP_ENV:-development}` and verify the interpolation inputs.
  - **Verify:** Run `docker compose config` and inspect the resolved configuration carefully.

- **A variable appears in Compose but is missing inside the container**
  - **Cause:** The variable is available for Compose interpolation but has not been mapped into the service environment.
  - **Fix:** Add it under `environment:` or use service-level `env_file:`.
  - **Verify:** Run `docker compose exec app printenv APP_ENV`.

- **Changes to the environment file do not affect an existing container**
  - **Cause:** The container was created using the previous configuration.
  - **Fix:** Recreate the container after changing its configuration. For Compose, use `docker compose up -d --force-recreate`.
  - **Verify:** Inspect the recreated container's environment.

- **A secret is accidentally committed**
  - **Cause:** A real `.env` file was added to version control.
  - **Fix:** Remove it from tracking, rotate any exposed credentials, and use an appropriate secrets-management solution. Adding the file to `.gitignore` alone does not remove it from Git history or invalidate the exposed secret.
  - **Verify:** Check repository history and credential usage, then confirm the replacement credentials work.

## References

- [Docker run: Environment variables](https://docs.docker.com/reference/cli/docker/container/run/)
- [Dockerfile reference: `ENV` and `ARG`](https://docs.docker.com/reference/dockerfile/)
- [Docker Compose: Environment variables](https://docs.docker.com/compose/how-tos/environment-variables/)
- [Docker Compose: Set environment variables in a container](https://docs.docker.com/compose/how-tos/environment-variables/set-environment-variables/)
- [Docker security](https://docs.docker.com/engine/security/)
