# Reading YAML manifests

> Phase 2 — Kubernetes Core (highest priority) › kubectl fluency

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A Kubernetes YAML manifest is a declarative configuration file that describes a Kubernetes resource and its desired state.

Instead of manually creating and configuring every component through commands, you define the resource in YAML and submit it to the Kubernetes API using commands such as `kubectl apply -f`.

For example, a Deployment manifest can specify:
- How many Pod replicas should run.
- Which container image each Pod should use.
- Which ports the containers use.
- What labels identify the Pods.
- What CPU and memory resources the containers request.

Kubernetes controllers then work to reconcile the actual cluster state with the desired state described by the resource.

**A typical Kubernetes manifest has four main fields:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
```

- `apiVersion`: Identifies the Kubernetes API group and version for the resource.
- `kind`: Specifies the resource type, such as `Pod`, `Deployment`, `Service`, or `ConfigMap`.
- `metadata`: Contains resource identification and metadata, such as its name, namespace, and labels.
- `spec`: Defines the desired configuration. Its structure depends on the resource type.

Many Kubernetes resources also expose a `status` section in the live object. Kubernetes controllers and other system components update this section to report observed state.

**Important distinction:**

- `spec` describes the desired configuration.
- `status` describes the observed state reported by Kubernetes.

For example, a Deployment may specify `replicas: 3`, while its status reports only two available replicas because one Pod has not started successfully.

You generally define `spec` in your manifest and let Kubernetes manage the live `status` field.

## Key points

- **YAML syntax:** YAML represents structured data using indentation, key-value pairs, lists, and nested objects.
- **Indentation matters:** YAML uses spaces to represent nesting. Use consistent spaces and avoid tabs for indentation.
- **Key-value pairs:** A colon separates a key from its value, such as `replicas: 3`.
- **Lists:** A hyphen introduces an item in a list, such as the containers defined under `spec.containers`.
- **Nested objects:** Indentation indicates which object owns a field. For example, `image` belongs to a container under `spec.containers`.
- **API version:** `apiVersion` must be supported by the cluster and appropriate for the resource kind.
- **Resource kind:** `kind` tells Kubernetes what type of object to create or update.
- **Metadata:** `metadata.name` identifies the resource within its namespace. `metadata.namespace` specifies the namespace for namespaced resources.
- **Labels:** Labels are key-value pairs used to organize and select resources.
- **Selectors:** Controllers such as Deployments use selectors to identify the Pods they manage. A Deployment's selector must match the labels in its Pod template.
- **Desired state:** `spec` defines what you want Kubernetes to maintain.
- **Observed state:** `status` reports what Kubernetes currently observes about the resource.
- **Container image:** `image` identifies the container image to run, including its registry, repository, and optionally a tag or digest.
- **Ports:** `containerPort` documents a port in the container specification. It does not start a server or automatically expose the application outside the Pod.
- **Environment variables:** `env` defines environment variables passed to a container. Sensitive values should generally be supplied through Kubernetes Secrets or an appropriate external secret-management solution.
- **Volumes:** `volumes` defines Pod-level storage sources, while `volumeMounts` specifies where containers mount them.
- **Resource requests and limits:** `resources.requests` specifies resources used for scheduling and allocation decisions. `resources.limits` specifies resource ceilings.
- **Probes:** `livenessProbe`, `readinessProbe`, and `startupProbe` control different aspects of container health and readiness behavior.
- **Multiple resources:** A YAML file can contain multiple Kubernetes resource documents separated by `---`.
- **Comments:** Lines beginning with `#` are comments and are ignored by the YAML parser.
- **Strings and numbers:** YAML supports multiple data types. Quote values when necessary to avoid unintended interpretation, particularly for values that look like booleans, numbers, or special strings.
- **Field names are case-sensitive:** Use the exact API field names, such as `matchLabels`, `containerPort`, and `imagePullPolicy`.
- **API schema validation:** Valid YAML is not necessarily a valid Kubernetes manifest. Kubernetes also validates resource kinds, field types, required fields, and other constraints.
- **Nested configuration:** The same field name may appear in different parts of a manifest. Always understand its location in the object hierarchy.
- **Manifest versus live resource:** The YAML file is your configuration input; the resource returned by `kubectl get -o yaml` is the live object and may contain generated fields, defaults, and runtime state.
- **Declarative updates:** Editing a manifest and running `kubectl apply -f` is a standard way to manage changes to resources.
- **Review before applying:** Use `kubectl diff -f` and server-side dry-run validation where appropriate before changing a shared or production environment.

## Commands / manifests

### 1. Read a basic Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: development
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.28
          ports:
            - containerPort: 80
```

Read the manifest from the outside inward.

**Level 1: Identify the resource**

```yaml
apiVersion: apps/v1
kind: Deployment
```

This tells you that the file defines a Deployment using the `apps/v1` API.

**Level 2: Identify the resource**

```yaml
metadata:
  name: web
  namespace: development
```

The Deployment is named `web` and belongs to the `development` namespace.

**Level 3: Understand the desired replica count**

```yaml
spec:
  replicas: 2
```

The Deployment should maintain two Pod replicas.

**Level 4: Understand the selector**

```yaml
selector:
  matchLabels:
    app: web
```

The Deployment manages Pods whose labels match `app: web`. The Pod template below must contain the matching label.

**Level 5: Understand the Pod template**

```yaml
template:
  metadata:
    labels:
      app: web
```

This defines the labels that Kubernetes places on Pods created by the Deployment.

**Level 6: Understand the container**

```yaml
containers:
  - name: nginx
    image: nginx:1.28
```

Each Pod contains a container named `nginx`, using the `nginx:1.28` image.

**Level 7: Understand the port declaration**

```yaml
ports:
  - containerPort: 80
```

This documents the container's intended port. The NGINX process must actually listen on port 80 for the application to work as expected.

### 2. Understand YAML indentation and lists

Consider this example:

```yaml
spec:
  containers:
    - name: api
      image: my-api:v1
    - name: worker
      image: my-worker:v1
```

The indentation shows that `containers` is a list containing two container objects.

The first container is named `api`. The second is named `worker`.

Both container objects belong to the same `spec.containers` list.

Compare it with:

```yaml
spec:
  containers:
    - name: api
      image: my-api:v1
      env:
        - name: LOG_LEVEL
          value: "INFO"
```

Here, `env` belongs to the `api` container, and `LOG_LEVEL` is an item in its environment-variable list.

A useful rule is to trace each field upward through its indentation until you reach the resource root.

### 3. Understand a Service manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: development
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

Read the important fields:

- `kind: Service`: Defines a Service.
- `type: ClusterIP`: Exposes the Service through an internal cluster IP by default.
- `selector.app: web`: Selects Pods carrying the `app: web` label.
- `port: 80`: The port exposed by the Service.
- `targetPort: 80`: The destination port on selected Pods.

The Service routes traffic to matching, eligible endpoints. The Pods must exist and be selected correctly for the Service to have usable endpoints.

A Service does not create the Pods itself.

### 4. Understand a ConfigMap manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  LOG_LEVEL: "INFO"
  API_TIMEOUT: "30"
```

This defines a ConfigMap containing two string values.

A Pod can consume these values through environment variables or mounted files.

Example environment-variable configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
  namespace: development
spec:
  containers:
    - name: app
      image: nginx:1.28
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

The container receives `LOG_LEVEL` from the `app-config` ConfigMap.

The ConfigMap must exist in the same namespace as the Pod. ConfigMaps are for non-confidential configuration, not passwords or other sensitive values.

### 5. Understand a resource with requests and limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: nginx:1.28
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Interpret the resource values:

- CPU request `250m`: One quarter of a CPU core.
- Memory request `128Mi`: 128 mebibytes of memory.
- CPU limit `500m`: Half of a CPU core.
- Memory limit `256Mi`: 256 mebibytes of memory.

These values apply to the container in this example. Kubernetes scheduling considers resource requests, while the runtime enforces limits according to the resource type and platform behavior.

A manifest can be syntactically valid but still fail to schedule if the cluster cannot satisfy the requested resources.

### 6. Read a manifest containing multiple resources

A YAML file can contain multiple Kubernetes resources separated by `---`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.28
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

This file defines a Deployment and a Service.

The separator `---` marks the beginning of another YAML document. It is not a shell command or a Kubernetes resource.

Apply both resources with:

```bash
kubectl apply -f web.yaml
```

The Service selector matches the Deployment's Pod labels, allowing the Service to route traffic to eligible Pods.

### 7. Validate a manifest before applying

Save a valid manifest as `deployment.yaml`.

Check for differences against the live resource:

```bash
kubectl diff -f deployment.yaml
```

Validate through the API server without saving the changes:

```bash
kubectl apply --dry-run=server -f deployment.yaml
```

Apply the manifest:

```bash
kubectl apply -f deployment.yaml
```

Check the result:

```bash
kubectl get deployment web
kubectl get pods
kubectl describe deployment web
```

These steps help separate configuration problems from runtime problems.

A successful server-side dry run does not guarantee that Pods will start successfully. Image availability, scheduling capacity, application configuration, and dependencies can still cause failures.

### 8. Inspect a live resource as YAML

```bash
kubectl get deployment web -o yaml
```

This prints the live Deployment object in YAML format.

You may see fields that were not explicitly written in your original manifest, including:

- `status`: The observed state reported by Kubernetes.
- `metadata.uid`: The resource's unique identifier.
- `metadata.resourceVersion`: The version identifier used by the API for concurrency and change tracking.
- `metadata.creationTimestamp`: When the object was created.
- `metadata.managedFields`: Information about field management.

Kubernetes may also add defaults to the live resource.

Do not blindly copy the entire live object back into a source manifest. Generated metadata and status fields are generally not part of the configuration you should manage manually.

### 9. Use `kubectl explain` while reading YAML

If you encounter an unfamiliar field, inspect its API documentation.

```bash
kubectl explain deployment.spec
kubectl explain deployment.spec.template
kubectl explain pod.spec.containers
kubectl explain service.spec.ports
```

For a broader view:

```bash
kubectl explain deployment --recursive
```

Use `kubectl explain` to understand field purpose and placement. Use the official documentation when you need more detailed examples, operational behavior, or version-specific guidance.

### 10. Practical YAML-reading exercise

Read the following manifest before running it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: development
  labels:
    team: data-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api-container
          image: nginx:1.28
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "200m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Answer these questions:

1. What type of resource does the file define?
2. What is its name and namespace?
3. How many Pod replicas should it maintain?
4. Which labels must the Pods have to match the Deployment selector?
5. What container image will each Pod use?
6. What port is declared for the container?
7. How much CPU and memory does each container request?
8. What CPU and memory limits are specified?
9. Does this manifest create a Service?
10. What could cause the Deployment to have fewer than three available replicas?

Verify your understanding by applying it in a development environment:

```bash
kubectl apply -f deployment.yaml
kubectl get deployment api -n development
kubectl get pods -n development -l app=api
kubectl describe deployment api -n development
```

The Deployment requests three replicas, but actual availability depends on successful scheduling and startup. This manifest does not create a Service.

Delete the test resource when finished:

```bash
kubectl delete -f deployment.yaml
```

## Common failures & fixes

- **YAML indentation errors**
  - Cause: Incorrect indentation or tabs used where indentation spaces are expected.
  - Fix: Use consistent spaces, inspect nested lists and objects, and validate the manifest before applying it.

- **`apiVersion` is unsupported**
  - Cause: The manifest uses an API version not served by the cluster.
  - Fix: Check the resource's supported API versions using `kubectl api-resources` and consult the documentation for the cluster's Kubernetes version.

- **`kind` is incorrect**
  - Cause: The resource kind is misspelled or not available in the cluster.
  - Fix: Verify the resource type and use `kubectl api-resources` to inspect available kinds.

- **A field is placed at the wrong indentation level**
  - Cause: A field is nested under the wrong parent object.
  - Fix: Trace the field's indentation and use `kubectl explain <resource>.<field-path>` to confirm the expected structure.

- **`unknown field` or validation errors**
  - Cause: The manifest contains an unsupported field, an invalid value, or a field in the wrong location.
  - Fix: Check spelling, API version, field path, and data type. Use `kubectl apply --dry-run=server -f <file.yaml>` to obtain API server validation feedback.

- **Deployment selector does not match Pod labels**
  - Cause: `spec.selector.matchLabels` differs from `spec.template.metadata.labels`.
  - Fix: Make the labels consistent. Deployment selectors are generally immutable, so correcting an existing Deployment may require recreating it.

- **The Deployment is created but no Pods become available**
  - Cause: Insufficient cluster capacity, invalid images, configuration problems, or failing health probes.
  - Fix: Inspect `kubectl describe deployment <name>`, `kubectl get pods`, and `kubectl describe pod <pod-name>`. Check the events and container logs.

- **Service has no usable endpoints**
  - Cause: The Service selector does not match Pod labels, or matching Pods are not ready.
  - Fix: Compare `spec.selector` in the Service with the Pod labels and inspect the Service's EndpointSlices.

- **A port is declared but the application is unreachable**
  - Cause: `containerPort` is only a declaration; it does not start a server or expose the application externally.
  - Fix: Verify the application is listening on the expected port, then check the Service, selectors, `port`, `targetPort`, and network access path.

- **ConfigMap value cannot be loaded**
  - Cause: The ConfigMap does not exist, the key is incorrect, or the Pod is in a different namespace.
  - Fix: Verify the ConfigMap and key with `kubectl get configmap <name> -o yaml -n <namespace>`. Ensure the Pod references the correct resource.

- **A manifest contains `status` copied from a live resource**
  - Cause: Runtime-generated information was copied into the source manifest.
  - Fix: Remove fields that should be managed by Kubernetes, especially `status` and generated metadata, while retaining the intended configuration.

- **The manifest is valid YAML but rejected by Kubernetes**
  - Cause: YAML syntax is valid, but the resource violates the Kubernetes API schema, validation rules, or admission policies.
  - Fix: Use server-side dry-run validation and review the error returned by the API server.

- **The manifest applies successfully but Pods remain Pending**
  - Cause: The configuration may be valid while scheduling is blocked by resource requests, node constraints, taints, or storage requirements.
  - Fix: Inspect Pod events and node capacity. A successful apply does not guarantee that the desired state can be achieved.

- **A change in the YAML does not appear in the cluster**
  - Cause: The wrong file or namespace was applied, the field was not changed in the intended object, or another manager controls the field.
  - Fix: Run `kubectl diff -f <file.yaml>`, inspect the live resource with `kubectl get <resource> <name> -o yaml`, and verify the active context and namespace.

## References

- [Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
- [Declarative Management of Kubernetes Objects Using Configuration Files](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/)
- [kubectl explain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/)
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
