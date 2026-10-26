# kubectl context & config

> Phase 2 — Kubernetes Core (highest priority) › Cluster model

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

`kubectl` is the command-line tool used to communicate with the Kubernetes API server. Before it can interact with a cluster, it needs to know **which cluster to contact, which identity to use, and how to authenticate**.

This information is typically stored in a kubeconfig file.

A kubeconfig file organizes connection information into three main concepts:

- **Cluster:** The Kubernetes API server endpoint and its certificate authority configuration.
- **User:** The identity and authentication configuration used to access the cluster.
- **Context:** A combination of a cluster, a user, and optionally a default namespace.

The active context determines the default cluster, user, and namespace used by `kubectl` commands when you do not explicitly override them.

### 1. Cluster

A cluster entry tells `kubectl` where the Kubernetes API server is and how to verify its TLS certificate.

Example:

```yaml
clusters:
  - name: development-cluster
    cluster:
      server: https://192.168.1.100:6443
      certificate-authority: /path/to/ca.crt
```

Key fields:

- `name`: A local identifier for this cluster entry in the kubeconfig file.
- `server`: The API server URL.
- `certificate-authority`: A file containing the CA certificate used to verify the API server's certificate.

Other configurations may use `certificate-authority-data`, which embeds the CA certificate data directly in the kubeconfig file.

The cluster name is a local alias. It does not have to match the actual cluster's name in the infrastructure provider.

### 2. User

A user entry specifies how `kubectl` authenticates to the API server.

Depending on the environment, authentication may use:

- Client certificates and private keys.
- Bearer tokens.
- An external credential plugin.
- Cloud-provider authentication integrations.
- Other supported authentication mechanisms.

Example using client certificate files:

```yaml
users:
  - name: developer-user
    user:
      client-certificate: /path/to/client.crt
      client-key: /path/to/client.key
```

The `name` field is a local identity alias in the kubeconfig. It does not automatically create a Kubernetes user or grant permissions.

Authentication and authorization are different:

- **Authentication:** Who are you?
- **Authorization:** What are you allowed to do?

Even if authentication succeeds, Kubernetes can reject an operation with `Forbidden` when the identity lacks the required permissions.

### 3. Context

A context connects a cluster entry to a user entry and can specify a default namespace.

Example:

```yaml
contexts:
  - name: development
    context:
      cluster: development-cluster
      user: developer-user
      namespace: development
```

This context tells `kubectl` to:

- Connect to `development-cluster`.
- Authenticate using `developer-user`.
- Use the `development` namespace by default for namespaced operations.

The context does not contain the cluster endpoint or credentials directly. It references the corresponding entries by name.

### 4. Current context

A kubeconfig file can contain multiple clusters, users, and contexts. The `current-context` field identifies the context that `kubectl` uses by default.

Example:

```yaml
current-context: development
```

Suppose your kubeconfig contains contexts for development, staging, and production. Switching the current context changes the default target for subsequent commands.

This is useful when working across multiple environments, but it introduces a risk: **a valid command can still be dangerous if it targets the wrong cluster**.

Before performing an important operation, verify the active context.

### 5. How kubeconfig is selected

By default, `kubectl` looks for a kubeconfig file at:

```text
$HOME/.kube/config
```

On Linux and macOS, this is commonly `~/.kube/config`. On Windows, the equivalent default location is typically `%USERPROFILE%\.kube\config`.

You can override the default configuration in two main ways:

- Set the `KUBECONFIG` environment variable.
- Pass `--kubeconfig` to an individual `kubectl` command.

For example:

```bash
kubectl --kubeconfig=/path/to/staging-config.yaml get nodes
```

This command uses the specified configuration file for that invocation.

When `KUBECONFIG` contains multiple file paths, `kubectl` can merge those configurations. The path separator is platform-specific:

- Linux and macOS: `:`
- Windows: `;`

For example, on Linux or macOS:

```bash
export KUBECONFIG="$HOME/.kube/config:$HOME/.kube/staging-config"
```

When multiple files are merged, overlapping entries and precedence rules can affect which configuration values are used. Inspect the merged result before relying on it.

### 6. Namespace vs context

A namespace logically separates resources within a cluster. A context selects a cluster and identity, and may also set a default namespace.

These concepts solve different problems:

| Concept | Purpose | Example |
|---|---|---|
| Cluster | Identifies an API server configuration | `development-cluster` |
| User | Defines an authentication identity | `developer-user` |
| Context | Combines cluster, user, and optional namespace | `development` |
| Namespace | Organizes namespaced resources within one cluster | `development` |
| Current context | Determines the default context for `kubectl` | `development` |

A namespace is not a separate Kubernetes cluster. Switching namespaces does not switch the API server or change your identity.

For example:

```bash
kubectl config set-context --current --namespace=testing
```

This changes the default namespace in the active context. It does not create the `testing` namespace, switch clusters, or grant access to resources there.

You can always override the default namespace for a command:

```bash
kubectl get pods -n production
```

### 7. What happens when you run kubectl?

Consider this command:

```bash
kubectl get pods
```

The process is broadly:

1. `kubectl` loads its kubeconfig settings.
2. It determines the active context, cluster endpoint, and authentication configuration.
3. It connects securely to the Kubernetes API server.
4. The API server authenticates the request and checks authorization.
5. Kubernetes returns the Pod information that the identity is allowed to access in the selected namespace.
6. `kubectl` formats the response for display.

If the context is wrong, the command may query a different cluster. If authentication fails, the request may be rejected. If authorization fails, the API server may return a `Forbidden` error.

### 8. Context safety in production

When working with multiple environments, adopt a deliberate verification routine:

- Check the current context before making changes.
- Use descriptive context names, such as `company-prod` and `company-dev`.
- Use explicit namespaces for important operations.
- Review the resource and namespace before deleting or modifying it.
- Treat kubeconfig files as sensitive because they may contain credentials or references to credential plugins.
- Avoid committing kubeconfig files, client private keys, or access tokens to Git.
- Do not assume that a context named `production` necessarily points to production. Verify its cluster endpoint and associated configuration.

For especially sensitive operations, explicitly specify the context:

```bash
kubectl --context=company-prod get pods -n production
```

This reduces reliance on whichever context happens to be active.

## Key points

- **Kubeconfig** stores cluster connection details, authentication configuration, and context definitions.
- A **cluster entry** identifies an API server endpoint and its certificate verification settings.
- A **user entry** specifies how a client authenticates to Kubernetes.
- A **context** combines a cluster, a user, and optionally a default namespace.
- The **current context** is the default target for `kubectl`.
- `kubectl config get-contexts` lists available contexts.
- `kubectl config use-context` changes the active context.
- `kubectl config view` displays configuration information. Use `--minify` to focus on the active context.
- `kubectl config current-context` shows the active context name.
- `kubectl config set-context --current --namespace=<namespace>` changes the default namespace of the active context.
- `-n <namespace>` overrides the default namespace for a command.
- `--context=<context-name>` overrides the active context for an individual command.
- `--kubeconfig=<path>` specifies the configuration file for an individual command.
- `KUBECONFIG` can specify one or more configuration files.
- **Context switching does not grant permissions.** Access is still controlled by the identity's authorization.
- **Namespace switching does not switch clusters.** It only changes the default namespace for applicable operations.
- **Authentication and authorization are different.** Valid credentials do not guarantee permission to perform an operation.
- Kubeconfig files may contain sensitive credentials. Do not share or commit them carelessly.
- A context name is only a local label. Verify the actual API server endpoint before performing production operations.

## Commands / manifests

### 1. Check the current context

```bash
kubectl config current-context
```

Displays the name of the currently active context.

### 2. List all contexts

```bash
kubectl config get-contexts
```

Shows configured contexts, their clusters, associated users, and default namespaces. The current context is marked in the output.

To display only the context names:

```bash
kubectl config get-contexts -o name
```

### 3. Inspect the kubeconfig

```bash
kubectl config view
```

Displays the merged configuration that `kubectl` sees, subject to its normal redaction behavior.

To focus on the active context and its referenced configuration:

```bash
kubectl config view --minify
```

To display the active context's namespace:

```bash
kubectl config view --minify \
  -o jsonpath='{..namespace}'
```

The output may be empty if the active context has no explicit namespace configured. In that case, namespaced commands normally default to the `default` namespace.

Avoid printing or sharing unredacted credentials. `kubectl config view --raw` can expose sensitive credential material and should only be used when necessary in a controlled environment.

### 4. Switch contexts

```bash
kubectl config use-context development
```

Changes the current context to `development`.

Verify the change:

```bash
kubectl config current-context
kubectl get nodes
```

Replace `development` with a context name that actually exists in your configuration.

### 5. Use a context without switching the default

```bash
kubectl --context=development get pods
```

Runs the command against the specified context without changing the current context.

To query a particular namespace:

```bash
kubectl --context=development get pods -n testing
```

This is useful when you need to run a command against a specific environment without changing your default target.

### 6. Change the default namespace

Set the default namespace for the active context:

```bash
kubectl config set-context --current --namespace=development
```

Verify it:

```bash
kubectl config view --minify
kubectl get pods
```

The second command now uses `development` as its default namespace, provided the resource type is namespaced.

To use a different namespace for just one command:

```bash
kubectl get pods -n testing
```

### 7. Create a new context

You can create a context that references existing cluster and user entries:

```bash
kubectl config set-context development-testing \
  --cluster=development-cluster \
  --user=developer-user \
  --namespace=testing
```

This creates or updates a context definition. It does not create a cluster, create a user, authenticate you, or grant permissions.

Inspect the result:

```bash
kubectl config get-contexts
```

If you want to make it active:

```bash
kubectl config use-context development-testing
```

The cluster and user names must refer to entries that exist in the effective kubeconfig.

### 8. Specify a kubeconfig file

```bash
kubectl --kubeconfig="$HOME/.kube/staging-config" get nodes
```

Uses the specified file for this command.

To set the configuration file for subsequent commands in the current Linux or macOS shell session:

```bash
export KUBECONFIG="$HOME/.kube/staging-config"
```

Confirm the selected context:

```bash
kubectl config current-context
```

On Windows PowerShell, a session-scoped equivalent is:

```powershell
$env:KUBECONFIG = "$HOME\.kube\staging-config"
```

### 9. Inspect the selected API server

```bash
kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}'
```

Prints the API server URL associated with the active context's cluster configuration.

For a safety check, inspect both the context and endpoint:

```bash
kubectl config current-context
kubectl config view --minify
```

A familiar context name alone is not sufficient evidence that you are connected to the intended environment.

### 10. Rename or delete a context

Rename a context:

```bash
kubectl config rename-context old-name new-name
```

Delete a context entry:

```bash
kubectl config delete-context old-name
```

Deleting a context does not delete the actual Kubernetes cluster or its resources. It removes the local context definition from the effective configuration being modified.

Before deleting a context, check whether it is the current context and whether any scripts or workflows depend on it.

### 11. Practice exercise

Use an existing development or local Kubernetes cluster.

1. Display the current context.
2. List all configured contexts.
3. Inspect the active context's cluster and namespace.
4. List the Pods in the default namespace.
5. Switch to another context only if one is available and safe to use.
6. Run `kubectl get nodes` with an explicit context.
7. Set a default namespace on a practice context.
8. Verify the result and explain the difference between changing a context and changing a namespace.

Useful commands:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config view --minify
kubectl get pods
kubectl get nodes
kubectl config set-context --current --namespace=default
kubectl config view --minify
```

Do not switch to or modify a production context just to complete this exercise.

## Common failures & fixes

| Error or symptom | Likely cause | Fix |
|---|---|---|
| `current-context is not set` | No active context is configured | Inspect `kubectl config get-contexts` and select a valid context |
| `context "..." does not exist` | The requested context name is incorrect or missing | Run `kubectl config get-contexts` and use an exact name |
| `The connection to the server ... was refused` | Incorrect endpoint, unreachable API server, or unavailable cluster | Check the active context, API server URL, network connectivity, and cluster status |
| `Unable to connect to the server` | Network, DNS, VPN, proxy, or endpoint issue | Verify connectivity to the configured API server and confirm the correct network or VPN is active |
| `Unauthorized` | Credentials are missing, invalid, or expired | Refresh credentials or rerun the appropriate provider login process |
| `Forbidden` | The authenticated identity lacks the required permissions | Check RBAC permissions and the target namespace |
| `certificate signed by unknown authority` | The CA configuration is missing or incorrect, or the server certificate cannot be validated | Verify the cluster CA configuration and endpoint |
| `You must be logged in to the server` | The configured authentication mechanism cannot provide valid credentials | Check token expiry, credential plugins, or provider-specific authentication |
| Commands target the wrong cluster | An unintended current context is active | Check `kubectl config current-context` and use `--context` explicitly |
| Commands return no resources | Wrong namespace or no matching resources | Check the active context namespace and use `-n <namespace>` or `-A` |
| Namespace change appears ineffective | The command explicitly specifies a namespace, or the operation is cluster-scoped | Review command flags and resource scope |
| `KUBECONFIG` appears ignored | A different configuration source or shell environment is in use | Inspect the environment variable and command-line `--kubeconfig` options |
| A context references a missing cluster or user | The kubeconfig is incomplete or was merged incorrectly | Inspect `kubectl config view` and restore the required entries |
| `kubectl config use-context` works, but API calls fail | Switching contexts changes local selection, not cluster availability or permissions | Verify endpoint reachability, authentication, and authorization |

### Troubleshooting workflow

When `kubectl` cannot reach a cluster, check the configuration before changing it:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config view --minify
kubectl cluster-info
kubectl get nodes
```

Interpret the results:

- If there is no current context, select an existing valid context.
- If the endpoint is incorrect, inspect the relevant kubeconfig entry.
- If the endpoint is correct but unreachable, investigate networking, DNS, VPN, firewall, and cluster availability.
- If the server responds with `Unauthorized`, investigate credentials.
- If the server responds with `Forbidden`, investigate authorization.
- If the API works but expected resources are missing, verify the namespace and resource selectors.

Do not immediately delete the kubeconfig or regenerate credentials. First identify which part of the configuration or connection is failing.

## References

- [Kubernetes: Configure Access to Multiple Clusters](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
- [kubectl config](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/)
- [kubectl config view](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_view/)
- [kubectl config use-context](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_use-context/)
- [Kubernetes Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
