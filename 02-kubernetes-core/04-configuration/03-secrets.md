# Secrets

> Phase 2 — Kubernetes Core (highest priority) › Configuration

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A Kubernetes Secret is an object used to store sensitive information required by applications or Kubernetes components.

Examples include:

- Database passwords
- API tokens
- Authentication credentials
- TLS certificates and private keys
- Registry credentials
- Application secrets

Secrets allow sensitive values to be managed separately from application code, container images, and ordinary configuration.

For example, an application connecting to a database needs:

```text
Database hostname: postgres-service
Database username: app_user
Database password: <sensitive-password>
```

The hostname and other non-sensitive settings can be stored in a ConfigMap, while the password can be stored in a Secret.

### Why Secrets are useful

Without Secrets, credentials may be hardcoded into application source code, Dockerfiles, deployment manifests, or configuration files committed to Git.

This creates security risks:

- Credentials can accidentally be committed to source control.
- Credentials can be exposed in container images.
- Credentials can be shared with users who do not need access.
- Rotating credentials becomes difficult.
- A leaked credential may allow unauthorized access to databases or external systems.

Secrets provide a Kubernetes-native mechanism for managing sensitive values and making them available to authorized workloads.

### How Secrets work

A Secret stores sensitive data as key-value pairs.

Kubernetes workloads can consume Secret data through:

1. Environment variables.
2. Mounted files.
3. Application or controller integrations that read the Secret through the Kubernetes API.

For example, a Secret named `db-credentials` may contain:

```text
DB_USERNAME=app_user
DB_PASSWORD=<sensitive-password>
```

A Pod can reference the Secret and receive the required values.

```text
Kubernetes Secret
       |
       | Referenced by Pod
       v
Application Container
       |
       | Reads credentials
       v
Database Connection
```

### Secret data and Base64 encoding

In a Secret manifest, values under the `data` field are Base64-encoded.

For example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YXBwX3VzZXI=
  password: c2VjdXJlLXBhc3N3b3Jk
```

Base64 is an encoding format, not encryption.

Anyone who can read the Secret data can decode it:

```bash
echo 'c2VjdXJlLXBhc3N3b3Jk' | base64 --decode
```

Kubernetes also supports the `stringData` field, which allows you to provide plaintext values in a manifest. Kubernetes converts those values into Secret data when processing the object.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: app_user
  password: secure-password
```

Although `stringData` is convenient, plaintext credentials in a manifest can be exposed through source control, backups, command history, or other access paths.

### Secret types

Kubernetes supports several Secret types.

| Secret type | Purpose |
|---|---|
| `Opaque` | General-purpose arbitrary sensitive data |
| `kubernetes.io/tls` | TLS certificate and private key |
| `kubernetes.io/dockerconfigjson` | Docker registry authentication configuration |
| `kubernetes.io/basic-auth` | Username and password |
| `kubernetes.io/ssh-auth` | SSH authentication credentials |
| `kubernetes.io/service-account-token` | Service account token Secret, used in legacy or specific integrations |

The `Opaque` type is the common choice for application credentials.

TLS Secrets are commonly used to store certificates and private keys for Ingress or other TLS-enabled components.

### How applications consume Secrets

**Environment variables**

Secret values can be injected into a container's environment.

For example:

```text
DB_USERNAME=app_user
DB_PASSWORD=<sensitive-password>
```

The application reads these values using its normal environment-variable mechanism.

**Mounted files**

Secret values can also be mounted as files.

For example:

```text
/etc/secrets/db/username
/etc/secrets/db/password
```

This is useful when applications expect credentials in files, such as TLS certificates, private keys, or configuration files.

### Secrets vs ConfigMaps

| ConfigMap | Secret |
|---|---|
| Stores non-sensitive configuration | Stores sensitive information |
| Example: API endpoint | Example: API token |
| Example: log level | Example: database password |
| Not intended for confidential data | Intended for sensitive values, subject to security controls |

A Secret is not automatically secure simply because Kubernetes calls it a Secret.

By default, Secret values are Base64-encoded in the API representation. To protect Secret data at rest, configure encryption at rest for the Kubernetes datastore. Access should also be restricted using RBAC.

### Kubernetes Secrets vs external secret managers

In production environments, organizations often use external secret management platforms such as:

- Azure Key Vault
- AWS Secrets Manager
- HashiCorp Vault
- Google Cloud Secret Manager

These systems can provide centralized secret storage, auditing, access policies, and rotation workflows.

Kubernetes workloads can integrate with external secret stores through mechanisms such as the Secrets Store CSI Driver or provider-specific integrations.

This can reduce the need to store long-lived credentials directly in Kubernetes Secret objects.

## Key points

- Secrets store sensitive values such as credentials, passwords, tokens, certificates, and keys.
- Secrets separate sensitive data from application code and container images.
- Secrets are namespace-scoped Kubernetes resources.
- A Secret can be consumed by a Pod using environment variables or mounted files.
- Secret values in the `data` field are Base64-encoded.
- Base64 is encoding, not encryption.
- `stringData` allows plaintext values in manifests, but those manifests must still be protected.
- Kubernetes supports different Secret types, including `Opaque`, `kubernetes.io/tls`, and `kubernetes.io/dockerconfigjson`.
- Secrets should not be committed to public or ordinary application repositories.
- Secrets should not be baked into container images or Dockerfiles.
- Kubernetes Secret access is controlled through RBAC.
- Encryption at rest should be configured to protect Secret data stored in the Kubernetes datastore.
- Restrict access to Secrets using least-privilege permissions.
- A user or workload with permission to read Secrets may be able to retrieve sensitive values.
- Environment variables containing Secret values may be exposed through debugging tools, process inspection, logs, or application diagnostics.
- Mounted Secret files are generally preferable when the application supports file-based credential loading.
- Secret volume updates are eventually propagated, but applications must reload the values to use them.
- Secret values consumed as environment variables do not update in existing containers when the Secret changes.
- Secret rotation should be planned as part of application and infrastructure security.
- External secret managers can provide centralized credential storage and rotation.
- Secrets are not a substitute for application-level authentication, authorization, or encryption.

## Commands / manifests

### 1. Create a Secret using imperative commands

Create a Secret with username and password values:

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=app_user \
  --from-literal=password='example-password' \
  -n development
```

What these arguments mean:

- `create secret generic`: Creates a general-purpose Secret.
- `db-credentials`: Name of the Secret.
- `--from-literal`: Provides a key-value pair.
- `-n development`: Creates the Secret in the `development` namespace.

The password shown is a placeholder for this learning exercise. Do not use it as a real credential.

Inspect the Secret metadata:

```bash
kubectl get secret db-credentials -n development
kubectl describe secret db-credentials -n development
```

Avoid displaying Secret values in terminal output unless you have a specific reason and an appropriate secure environment.

### 2. Create a Secret from files

Suppose you have two local files:

```text
username.txt
password.txt
```

Create a Secret:

```bash
kubectl create secret generic db-credentials \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt \
  -n development
```

The Secret will contain keys named `username` and `password`.

This is useful when credentials are supplied through protected files rather than command-line literals.

Be careful with local file permissions and avoid storing credential files in repositories.

### 3. Create a Secret using a YAML manifest

Example `db-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: development
type: Opaque
stringData:
  username: app_user
  password: example-password
```

Apply it:

```bash
kubectl apply -f db-secret.yaml
```

This example demonstrates the manifest structure. For real credentials, avoid committing plaintext Secret manifests to source control.

For production environments, use an approved secret-management workflow, such as an external secret manager or a secure deployment mechanism.

### 4. Inject Secret values as environment variables

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-client
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database-client
  template:
    metadata:
      labels:
        app: database-client
    spec:
      containers:
        - name: app
          image: nginx:stable
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
```

Apply the Deployment:

```bash
kubectl apply -f database-client.yaml
```

The container receives the Secret values through its environment.

The NGINX image is used here to demonstrate the Kubernetes configuration. A real database client application would read these environment variables and use them to connect to the database.

### 5. Load all Secret keys using `envFrom`

You can inject all keys from a Secret:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-client
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database-client
  template:
    metadata:
      labels:
        app: database-client
    spec:
      containers:
        - name: app
          image: nginx:stable
          envFrom:
            - secretRef:
                name: db-credentials
```

This makes all Secret keys available as environment variables.

Use `secretKeyRef` when you need only specific keys.

Use `envFrom` when the application needs all keys from the Secret.

### 6. Mount a Secret as files

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-file-demo
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret-file-demo
  template:
    metadata:
      labels:
        app: secret-file-demo
    spec:
      containers:
        - name: app
          image: nginx:stable
          volumeMounts:
            - name: credentials
              mountPath: /etc/app-secrets
              readOnly: true
      volumes:
        - name: credentials
          secret:
            secretName: db-credentials
```

The container receives files:

```text
/etc/app-secrets/username
/etc/app-secrets/password
```

The application can read the values from those files.

This approach is useful for applications that support file-based credential loading.

### 7. Create a TLS Secret

A TLS Secret stores a certificate and its corresponding private key.

```bash
kubectl create secret tls web-tls \
  --cert=web.example.com.crt \
  --key=web.example.com.key \
  -n development
```

The Secret type is:

```text
kubernetes.io/tls
```

Inspect its metadata:

```bash
kubectl get secret web-tls -n development
kubectl describe secret web-tls -n development
```

TLS Secrets are commonly referenced by Ingress resources or other components that terminate HTTPS.

### 8. Create a Docker registry Secret

A registry Secret can be used when a Pod needs credentials to pull an image from a private registry.

```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=registry.example.com \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n development
```

Reference it in a Pod or Deployment:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: registry-credentials
      containers:
        - name: app
          image: registry.example.com/my-app:1.0
```

The credentials allow Kubernetes to authenticate to the registry when pulling the image.

Avoid entering actual passwords into shared terminals or shell history.

### 9. Update a Secret

Update a Secret using a protected manifest or approved secret-management workflow:

```bash
kubectl apply -f db-secret.yaml
```

For Secrets consumed as environment variables, restart the Deployment so new Pods receive the updated values:

```bash
kubectl rollout restart deployment/database-client \
  -n development
```

Check the rollout:

```bash
kubectl rollout status deployment/database-client \
  -n development
```

Mounted Secret files are updated eventually, but applications may need to reload them. Secret volumes mounted using `subPath` do not receive updates.

### 10. Clean up

```bash
kubectl delete secret db-credentials -n development
kubectl delete secret web-tls -n development
kubectl delete secret registry-credentials -n development
```

Deleting a Secret does not automatically guarantee that all copies of its values have disappeared from containers, logs, backups, or external systems.

## Common failures & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `secret not found` | Secret is missing or in the wrong namespace | Verify the Secret name and namespace |
| `key not found` | Pod references a key that does not exist in the Secret | Inspect the Secret keys and correct `secretKeyRef` |
| Pod fails to start due to missing Secret | Secret is required but unavailable | Create the Secret or correct the reference |
| `ImagePullBackOff` with a private registry | Registry credentials are incorrect or missing | Verify the registry Secret and `imagePullSecrets` |
| Application receives incorrect credentials | Wrong Secret key or stale environment variables | Verify the key mapping and recreate the Pod |
| Secret update is not reflected in application | Environment variables are not updated in running containers | Restart the workload or implement credential reload |
| Mounted Secret file is missing | Incorrect volume configuration or Secret reference | Inspect Pod events and verify the Secret name |
| TLS handshake failure | Invalid, expired, or mismatched certificate | Verify certificate validity, hostname, and key pair |
| `Unauthorized` when reading a Secret | RBAC does not permit access | Review Role, RoleBinding, ClusterRole, and ClusterRoleBinding |
| Sensitive values appear in Git | Plaintext Secret manifest was committed | Revoke and rotate the exposed credentials, remove the secret from repository history where appropriate, and use a secure secret-management workflow |
| Secret values are exposed in logs | Application or debugging command prints credentials | Remove sensitive logging and rotate exposed credentials |
| Secret values are stored in Base64 and mistaken for encrypted data | Base64 encoding was misunderstood | Treat Base64 values as readable data and configure encryption at rest |

### Troubleshooting sequence

Inspect Secret metadata:

```bash
kubectl get secrets -n development
```

Inspect the Secret's key names and type:

```bash
kubectl describe secret db-credentials -n development
```

Inspect Pod status:

```bash
kubectl get pods -n development
kubectl describe pod <pod-name> -n development
```

Check application logs:

```bash
kubectl logs <pod-name> -n development
```

Check whether the current user has permission to read Secrets:

```bash
kubectl auth can-i get secrets -n development
```

For production credentials, troubleshoot without unnecessarily printing the Secret values.

## References

- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Configure a Pod to Use a Secret](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-secret/)
- [Secret Best Practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/)
