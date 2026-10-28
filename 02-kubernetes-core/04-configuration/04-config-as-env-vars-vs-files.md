# Config as env vars vs files

> Phase 2 — Kubernetes Core (highest priority) › Configuration

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Kubernetes applications often need configuration values such as database hostnames, log levels, feature flags, API endpoints, and application settings.

Kubernetes supports two common ways to provide configuration to a container:

1. **Environment variables:** Kubernetes injects configuration values into the container's environment when the container starts.
2. **Mounted files:** Kubernetes makes configuration available as files inside the container's filesystem.

Configuration is commonly stored in a Kubernetes `ConfigMap` for non-sensitive values or a `Secret` for sensitive values. Both can be exposed as environment variables or mounted files.

### Environment variables

Environment variables are useful for simple configuration values that an application reads when it starts.

For example:

```text
APP_ENV=production
LOG_LEVEL=INFO
DB_HOST=postgres.default.svc.cluster.local
```

The application reads these values from its process environment.

**Advantages**
- Simple for applications that already use environment variables.
- Convenient for individual settings such as log levels, hostnames, and feature flags.
- Avoids requiring the application to read a configuration file.

**Limitations**
- Values are generally fixed for the lifetime of the container process.
- Updating a ConfigMap or Secret does not update environment variables inside an already-running container.
- Large or structured configuration can become difficult to manage as individual variables.

### Mounted configuration files

Kubernetes can mount a ConfigMap or Secret into a container as files.

For example, a ConfigMap might contain:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    app.environment=production
    app.log_level=INFO
    app.feature_x_enabled=true
```

When mounted, the application can read the file from a path such as:

```text
/etc/app-config/app.properties
```

**Advantages**
- Suitable for structured configuration such as JSON, YAML, properties files, and application-specific config files.
- Keeps related settings together.
- ConfigMap and Secret volume updates are eventually reflected in the mounted files.
- Works well for applications that already load configuration from files.

**Limitations**
- The application must know where the file is and how to parse it.
- Updates are not necessarily picked up by the application automatically. The application may need to reload the file or restart.
- A `subPath` mount does not receive automatic updates from the underlying ConfigMap or Secret.

### How to choose

| Scenario | Recommended approach |
|---|---|
| Single values such as `LOG_LEVEL` or `APP_ENV` | Environment variables |
| Database hostname or service endpoint | Environment variables or a config file |
| Multiple related settings | Mounted configuration file |
| Application expects a JSON, YAML, or properties file | Mounted configuration file |
| Configuration must be reloaded without restarting | Mounted file, if the application supports reload |
| Passwords, tokens, API keys | Kubernetes Secret, exposed as env vars or files |
| Configuration that changes frequently | Mounted files can help, but application reload behaviour must be considered |

**Important:** Environment variables and mounted files are delivery mechanisms, not security boundaries. Secrets require appropriate RBAC permissions and encryption-at-rest configuration. Base64-encoding a Kubernetes Secret does not encrypt its contents.

## Key points

- `ConfigMap` stores non-sensitive configuration.
- `Secret` stores sensitive configuration, such as passwords and tokens.
- Kubernetes can expose ConfigMaps and Secrets through environment variables or volume mounts.
- Environment variables are set when a container starts. Existing processes do not automatically receive changed values.
- ConfigMap and Secret volume projections update eventually when the source object changes, subject to kubelet propagation.
- An application must reload a changed file to use its new contents. Kubernetes does not guarantee that the application will reload it.
- A volume mount using `subPath` does not receive automatic updates.
- Using `envFrom` imports all eligible keys from a ConfigMap or Secret as environment variables.
- Using `configMapKeyRef` or `secretKeyRef` imports a specific key.
- Use `items` in a volume definition to control which keys are mounted and what filenames they use.
- Avoid putting sensitive values in ConfigMaps, command-line arguments, container images, or source control.
- Updating a ConfigMap used as environment variables usually requires a Pod restart or rollout to make the new values available to the application.
- A Deployment rollout can be triggered by changing its Pod template, for example by updating an annotation or using a configuration checksum.

## Commands / manifests

### 1. Create a ConfigMap

Create a ConfigMap from literal values:

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=INFO
```

Inspect the ConfigMap:

```bash
kubectl get configmap app-config
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

### 2. Inject ConfigMap values as environment variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-config-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env && sleep 3600"]
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

Save this as `env-config-pod.yaml` and apply it:

```bash
kubectl apply -f env-config-pod.yaml
```

Inspect the environment variables:

```bash
kubectl exec env-config-pod -- printenv APP_ENV
kubectl exec env-config-pod -- printenv LOG_LEVEL
```

Expected output:

```text
production
INFO
```

### 3. Import all ConfigMap keys using `envFrom`

Instead of defining each environment variable individually:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-config-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env && sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
```

This imports the ConfigMap's keys as environment variables.

Use `env` when you need precise control over individual variables. Use `envFrom` when you want to import multiple keys together.

### 4. Mount ConfigMap values as files

Create a ConfigMap containing a configuration file:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-file-config
data:
  app.properties: |
    app.environment=production
    app.log_level=INFO
  feature-flags.json: |
    {
      "feature_x_enabled": true,
      "feature_y_enabled": false
    }
```

Create a Pod that mounts the ConfigMap:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: file-config-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "cat /etc/app-config/app.properties && sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app-config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-file-config
```

Apply the manifests:

```bash
kubectl apply -f app-file-config.yaml
kubectl apply -f file-config-pod.yaml
```

Read the mounted files:

```bash
kubectl exec file-config-pod -- \
  cat /etc/app-config/app.properties

kubectl exec file-config-pod -- \
  cat /etc/app-config/feature-flags.json
```

The ConfigMap keys become filenames inside `/etc/app-config`.

### 5. Mount only selected ConfigMap keys

Use `items` when the application should receive only specific files:

```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-file-config
      items:
        - key: app.properties
          path: application.properties
```

The file will be available at:

```text
/etc/app-config/application.properties
```

This maps the ConfigMap key `app.properties` to the filename `application.properties`.

### 6. Expose a Secret as an environment variable

Create a Secret:

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USERNAME=appuser \
  --from-literal=DB_PASSWORD='change-me'
```

Reference its values in a Pod:

```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: DB_USERNAME
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: DB_PASSWORD
```

For production workloads, avoid placing real credentials directly in shell history or committed YAML files. Use an approved secret-management workflow.

### 7. Mount a Secret as files

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-file-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/db-credentials
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
```

The Secret keys will appear as files:

```text
/etc/db-credentials/DB_USERNAME
/etc/db-credentials/DB_PASSWORD
```

The application can read the files directly. Be careful when using commands such as `cat` to inspect Secret files because they may expose credentials in terminal output or logs.

### 8. Update configuration and restart a workload

Update a ConfigMap:

```bash
kubectl edit configmap app-config
```

For environment-variable-based configuration, restart the Deployment so newly created Pods receive the updated values:

```bash
kubectl rollout restart deployment/my-app
kubectl rollout status deployment/my-app
```

For mounted files, Kubernetes eventually updates the projected files, but the application must reload them. If the application cannot reload configuration, a rollout restart may still be required.

### 9. Clean up the practice resources

```bash
kubectl delete pod env-config-pod
kubectl delete pod envfrom-config-pod
kubectl delete pod file-config-pod
kubectl delete pod secret-file-pod

kubectl delete configmap app-config
kubectl delete configmap app-file-config
kubectl delete secret db-credentials
```

## Common failures & fixes

| Failure | Likely cause | Fix |
|---|---|---|
| Pod fails to start because a ConfigMap or Secret is missing | The referenced object does not exist in the Pod's namespace | Create the object in the same namespace or correct its name |
| `CreateContainerConfigError` | A referenced key is missing or the ConfigMap/Secret cannot be found | Check `kubectl describe pod <pod>` and verify the key names |
| Environment variable is empty or missing | The key name is incorrect, or an optional reference is used | Inspect the ConfigMap/Secret and check the `env` or `envFrom` configuration |
| Updated ConfigMap value is not visible in the application | Environment variables are set at container startup | Restart the Pod or trigger a Deployment rollout |
| Mounted configuration file does not change | The application is reading a different path, or the mount uses `subPath` | Verify the mount path; avoid `subPath` when automatic projected updates are required |
| File changes but the application still uses the old setting | The application does not reload configuration automatically | Implement reload logic or restart the workload |
| Application cannot read a mounted file | File permissions, mount path, or container user mismatch | Inspect the Pod spec and file permissions; configure suitable `defaultMode` and application access |
| Sensitive configuration is exposed | Secret values are logged, committed, or accessible to excessive identities | Restrict RBAC, avoid logging secrets, and use encryption at rest or an external secret manager |
| ConfigMap key contains an invalid environment variable name | The key cannot be represented as a valid environment variable name | Use a valid key name or mount the value as a file instead |

Useful debugging commands:

```bash
kubectl describe pod <pod-name>
kubectl get configmap <configmap-name> -o yaml
kubectl get secret <secret-name> -o yaml
kubectl logs <pod-name>
kubectl exec <pod-name> -- printenv
kubectl exec <pod-name> -- ls -l <mount-path>
```

## References

- [Kubernetes: ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes: Define environment variables for a container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
- [Kubernetes: Configure a Pod to use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Kubernetes: Distribute credentials securely using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
