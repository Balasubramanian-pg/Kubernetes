# ConfigMaps

> Phase 2 — Kubernetes Core (highest priority) › Configuration

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A ConfigMap is a Kubernetes object used to store non-sensitive configuration data separately from application container images.

It allows you to configure an application without hardcoding configuration values into the application code or rebuilding the container image whenever a configuration value changes.

For example, an application might need:

- Database hostname
- Application environment
- API endpoint URL
- Log level
- Feature flags
- Configuration files

Instead of embedding these values inside the Docker image, you can store them in a ConfigMap and make them available to the application at runtime.

### Why ConfigMaps are useful

Consider an application that uses different database hostnames across environments:

```text
Development: postgres.dev.svc.cluster.local
Testing:    postgres.test.svc.cluster.local
Production: postgres.prod.svc.cluster.local
```

Without ConfigMaps, the configuration might be hardcoded into the application or container image.

With ConfigMaps, the same application image can be deployed across environments while receiving different configuration values.

```text
                  Same Application Image
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
          Dev Pod       Test Pod      Prod Pod
             |             |             |
             v             v             v
         Dev Config    Test Config    Prod Config
         ConfigMap     ConfigMap      ConfigMap
```

### How ConfigMaps work

A ConfigMap stores configuration as key-value pairs or configuration file content.

Kubernetes can make ConfigMap data available to containers through:

1. Environment variables.
2. Environment variables populated from individual ConfigMap keys.
3. Mounted files in a container's filesystem.

The application reads the configuration using its normal configuration-loading mechanism.

### ConfigMap as environment variables

For example, a ConfigMap might contain:

```text
APP_ENV=development
LOG_LEVEL=debug
API_URL=https://dev.example.com
```

Kubernetes can inject these values into the container environment.

The application can then access them using environment-variable mechanisms provided by its programming language or runtime.

### ConfigMap as a mounted configuration file

ConfigMaps can also store entire configuration files.

For example:

```yaml
server:
  port: 8080
logging:
  level: INFO
```

This configuration can be mounted into the container as a file.

The application can read the file using its normal configuration parser.

This is useful for applications that expect configuration in formats such as YAML, JSON, properties files, or other text-based formats.

### ConfigMaps and Secrets

ConfigMaps are intended for non-sensitive configuration.

| ConfigMap | Secret |
|---|---|
| Stores non-sensitive configuration | Stores sensitive information |
| Example: application log level | Example: database password |
| Suitable for environment variables and config files | Suitable for credentials, tokens, and keys |
| Not intended to provide confidentiality | Requires appropriate access controls and security configuration |

A Secret is not automatically secure simply because it is a Secret. Kubernetes Secrets require appropriate RBAC, encryption-at-rest configuration, and careful handling.

Do not store passwords, access tokens, private keys, or other sensitive values in ConfigMaps.

### ConfigMap updates and application behaviour

Updating a ConfigMap does not necessarily cause an application to reload its configuration.

The behaviour depends on how the ConfigMap is consumed:

- **Environment variables:** Existing containers do not receive updated environment variables. A Pod restart or replacement is generally required.
- **Mounted ConfigMap volume:** Kubernetes updates the projected files eventually, but the application must reload or reread them to use the new values.
- **ConfigMap mounted using `subPath`:** Updates are not propagated to the mounted file.

For production workloads, configuration changes may need to be coordinated with a Deployment rollout or an application-specific reload mechanism.

## Key points

- ConfigMaps store non-sensitive application configuration.
- They separate application configuration from container images.
- The same container image can be deployed across different environments with different ConfigMaps.
- ConfigMaps can store key-value pairs and configuration file content.
- ConfigMap data can be exposed as environment variables or mounted files.
- ConfigMaps are namespace-scoped resources.
- A Pod can reference a ConfigMap in its own namespace.
- ConfigMaps are not intended for confidential or sensitive data.
- Use Secrets for credentials, tokens, passwords, and other sensitive values.
- ConfigMaps can be created using YAML manifests, imperative commands, files, or directories.
- A ConfigMap can be referenced by a Pod through `env`, `envFrom`, or a volume.
- Updating a ConfigMap does not automatically update environment variables inside existing containers.
- Mounted ConfigMap files can update eventually, but applications may need to reload their configuration.
- A ConfigMap mounted with `subPath` does not receive updates to the underlying ConfigMap.
- ConfigMaps have a size limit, commonly around 1 MiB per object. They should not be used to store large application data or binaries.
- ConfigMaps are not a replacement for persistent storage, application databases, or external configuration management systems.

## Commands / manifests

### 1. Create a ConfigMap using imperative commands

Create a ConfigMap with individual key-value pairs:

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=development \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=API_URL=https://dev.example.com \
  -n development
```

What these arguments mean:

- `create configmap`: Creates a ConfigMap object.
- `app-config`: Name of the ConfigMap.
- `--from-literal`: Adds a configuration key and value.
- `-n development`: Creates the ConfigMap in the `development` namespace.

Inspect it:

```bash
kubectl get configmap app-config -n development
kubectl describe configmap app-config -n development
```

View its YAML:

```bash
kubectl get configmap app-config \
  -n development \
  -o yaml
```

### 2. Create a ConfigMap using a YAML manifest

Save the following as `app-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  APP_ENV: "development"
  LOG_LEVEL: "debug"
  API_URL: "https://dev.example.com"
```

Apply it:

```bash
kubectl apply -f app-config.yaml
```

The `data` section contains the configuration key-value pairs.

### 3. Inject ConfigMap values as environment variables

Create a Deployment that consumes the ConfigMap:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-demo
  template:
    metadata:
      labels:
        app: config-demo
    spec:
      containers:
        - name: app
          image: nginx:stable
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
            - name: API_URL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: API_URL
```

Apply the Deployment:

```bash
kubectl apply -f config-demo.yaml
```

Inspect the environment variables inside the container:

```bash
kubectl exec deployment/config-demo \
  -n development \
  -- printenv APP_ENV LOG_LEVEL API_URL
```

Expected output:

```text
development
debug
https://dev.example.com
```

### 4. Load all ConfigMap keys using `envFrom`

Instead of defining each environment variable separately, you can inject all keys from a ConfigMap.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-demo
  template:
    metadata:
      labels:
        app: config-demo
    spec:
      containers:
        - name: app
          image: nginx:stable
          envFrom:
            - configMapRef:
                name: app-config
```

This exposes the ConfigMap keys as environment variables in the container.

Use `configMapKeyRef` when you want to select specific keys.

Use `envFrom` when you want to inject all keys.

### 5. Create a ConfigMap from a configuration file

Suppose you have a local file named `application.yaml`:

```yaml
server:
  port: 8080

logging:
  level: INFO

application:
  environment: development
```

Create a ConfigMap from that file:

```bash
kubectl create configmap application-config \
  --from-file=application.yaml \
  -n development
```

The ConfigMap contains a key named `application.yaml` whose value is the contents of the file.

Inspect it:

```bash
kubectl describe configmap application-config -n development
```

### 6. Mount a ConfigMap as a file

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-file-demo
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-file-demo
  template:
    metadata:
      labels:
        app: config-file-demo
    spec:
      containers:
        - name: app
          image: nginx:stable
          volumeMounts:
            - name: app-config-volume
              mountPath: /etc/application-config
              readOnly: true
      volumes:
        - name: app-config-volume
          configMap:
            name: application-config
```

The file will be available inside the container at:

```text
/etc/application-config/application.yaml
```

Inspect the mounted file:

```bash
kubectl exec deployment/config-file-demo \
  -n development \
  -- cat /etc/application-config/application.yaml
```

This example mounts the configuration file into an NGINX container for demonstration. A real application would need to read and use the configuration file.

### 7. Mount a specific ConfigMap key

You can use `items` to select specific keys and control the filenames.

```yaml
volumes:
  - name: app-config-volume
    configMap:
      name: app-config
      items:
        - key: APP_ENV
          path: environment
        - key: LOG_LEVEL
          path: logging-level
```

The resulting files are:

```text
/etc/application-config/environment
/etc/application-config/logging-level
```

This requires the volume to be mounted at `/etc/application-config`.

### 8. Update a ConfigMap

Edit the ConfigMap directly:

```bash
kubectl edit configmap app-config -n development
```

Or apply an updated manifest:

```bash
kubectl apply -f app-config.yaml
```

Inspect the updated values:

```bash
kubectl get configmap app-config \
  -n development \
  -o yaml
```

If the application consumes the values through environment variables, restart the Deployment to recreate the Pods:

```bash
kubectl rollout restart deployment/config-demo -n development
```

Check the rollout:

```bash
kubectl rollout status deployment/config-demo -n development
```

### 9. Delete a ConfigMap

```bash
kubectl delete configmap app-config -n development
```

Deleting a ConfigMap referenced by a running Pod can affect the workload. Pods using the ConfigMap as a volume may encounter problems, while environment variables already injected into running containers remain in their existing process environments.

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `configmap not found` | ConfigMap is missing or in the wrong namespace | Run `kubectl get configmap -n <namespace>` and verify the name |
| Pod fails to start because a ConfigMap is missing | Pod references a ConfigMap that does not exist | Create the ConfigMap or correct the reference |
| `configmap key not found` | The referenced key does not exist | Inspect the ConfigMap and correct the key name |
| Environment variable is empty or missing | Incorrect key reference or optional ConfigMap/key | Verify `configMapKeyRef` and the key name |
| Updated ConfigMap value is not reflected in the application | Environment variables in existing Pods are unchanged | Restart the Deployment or implement application-level reload |
| Mounted configuration file is outdated | Application is using `subPath`, or has not reread the updated file | Avoid `subPath` for dynamic updates and reload configuration as needed |
| Application cannot parse the configuration file | Incorrect file format, syntax, or mount path | Validate the file and confirm the application expects that configuration format |
| Pod cannot mount the ConfigMap | ConfigMap name, key, or namespace is incorrect | Inspect the Pod events and verify the referenced ConfigMap |
| ConfigMap exceeds size limit | Too much data stored in one ConfigMap | Reduce the configuration size or use an appropriate external configuration or storage mechanism |
| Sensitive data is exposed | Credentials or tokens were stored in a ConfigMap | Move sensitive values to a Secret and apply proper access controls |

### Troubleshooting sequence

Inspect ConfigMaps:

```bash
kubectl get configmaps -n development
kubectl describe configmap app-config -n development
```

Inspect the Deployment:

```bash
kubectl describe deployment config-demo -n development
```

Inspect Pod status and events:

```bash
kubectl get pods -n development
kubectl describe pod <pod-name> -n development
```

Check the container's environment:

```bash
kubectl exec <pod-name> -n development -- printenv
```

Check mounted files:

```bash
kubectl exec <pod-name> -n development -- ls -l /etc/application-config
kubectl exec <pod-name> -n development -- cat /etc/application-config/application.yaml
```

Inspect application logs:

```bash
kubectl logs <pod-name> -n development
```

## References

- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [ConfigMaps as Environment Variables](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-container-environment-variables-using-configmap-data)
- [ConfigMaps as Mounted Volumes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#add-configmap-data-to-a-volume)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
