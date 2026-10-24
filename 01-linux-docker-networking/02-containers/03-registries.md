# Registries

> Phase 1 — Linux, Docker & Networking › Containers

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **container registry** is a service that stores, manages, versions, and distributes container images. Instead of building an image separately on every machine, developers can push an image to a registry and allow other machines, CI/CD pipelines, or Kubernetes clusters to pull it when needed.

For example, after building a Docker image for a data processing application, you can push it to a registry. A Kubernetes cluster can then download that image and use it to create application containers on its worker nodes.

A registry typically stores:
- **Repositories:** Collections of related images, often organized by application or project.
- **Tags:** Human-readable references such as `1.0`, `v2.1.0`, or `latest`.
- **Image manifests:** Metadata describing an image and the content it references, including its filesystem layers.
- **Image layers:** Filesystem changes that make up the image.
- **Digests:** Content-based identifiers used to reference specific image content.
- **Access controls:** Permissions that determine who can push, pull, or manage images.
- **Artifacts and metadata:** Depending on the registry, these may include signatures, software bills of materials (SBOMs), vulnerability scan results, and other supply-chain information.

### How a registry fits into the container lifecycle

```text
Developer
    |
    | Write Dockerfile and application code
    v
Build container image
    |
    | docker push
    v
Container Registry
    |
    | docker pull / Kubernetes image pull
    v
Server or Kubernetes Worker Node
    |
    v
Container Runtime
    |
    v
Running Container
```

The registry stores and distributes images. It does not normally run your application containers. The container runtime on a server or Kubernetes node downloads the image and creates the containers.

### Major container registries

**1. Docker Hub**

Docker Hub is a widely used public container registry. It hosts official images, community images, and private repositories.

- Common use cases: Learning Docker, downloading base images, publishing open-source images, and sharing application images.
- Example image: `nginx:alpine`
- Typical reference: `docker.io/library/nginx:alpine`
- Authentication: `docker login`
- Best suited for: General-purpose container image distribution and development workflows.

**2. GitHub Container Registry (GHCR)**

GitHub Container Registry is part of GitHub Packages and supports storing container images alongside source code and GitHub Actions workflows.

- Common use cases: Publishing images from GitHub repositories and integrating image builds with GitHub Actions.
- Registry hostname: `ghcr.io`
- Example reference: `ghcr.io/OWNER/IMAGE:TAG`
- Authentication: `docker login ghcr.io`
- Best suited for: Projects that use GitHub for source control, CI/CD, and release management.

**3. Amazon Elastic Container Registry (ECR)**

Amazon Elastic Container Registry is AWS's managed container image registry. It integrates with AWS services, including Amazon Elastic Kubernetes Service (EKS), Amazon ECS, and AWS IAM.

- Common use cases: Storing production application images for AWS-hosted workloads.
- Registry hostname: Typically includes the AWS account ID, AWS Region, and `amazonaws.com`.
- Example reference: `ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/REPOSITORY:TAG`
- Authentication: Commonly uses AWS IAM and an ECR authorization token.
- Best suited for: AWS-native deployments and access control integrated with AWS identities.

**4. Google Artifact Registry**

Google Artifact Registry is Google's managed artifact repository service and supports container images as well as other artifact formats. It is the recommended general-purpose successor to Google Container Registry for new Google Cloud artifact repository workflows.

- Common use cases: Storing application images for Google Kubernetes Engine (GKE), Cloud Run, and other Google Cloud workloads.
- Registry hostname: Commonly `LOCATION-docker.pkg.dev`
- Example reference: `LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE:TAG`
- Authentication: Commonly uses Google Cloud IAM and the Google Cloud CLI credential helper.
- Best suited for: Google Cloud deployments and centralized artifact management.

**Terminology note:** Google Container Registry (GCR), historically accessed through `gcr.io`, is the legacy registry service. Google has transitioned users toward Artifact Registry. Existing GCR workflows and compatibility arrangements may still exist, but Artifact Registry is generally the appropriate choice for new implementations.

### Registry comparison

| Feature | Docker Hub | GHCR | Amazon ECR | Google Artifact Registry |
|---|---|---|---|---|
| Provider | Docker | GitHub | AWS | Google Cloud |
| Registry hostname | `docker.io` | `ghcr.io` | AWS account and Region-specific endpoint | `LOCATION-docker.pkg.dev` |
| Authentication | Docker credentials or supported access tokens | GitHub credentials or supported tokens | AWS IAM and ECR authorization | Google Cloud IAM and supported credentials |
| CI/CD integration | Broad tooling support | GitHub Actions | AWS CodeBuild and other CI/CD systems | Cloud Build and other CI/CD systems |
| Kubernetes integration | Any compatible cluster with registry access | Any compatible cluster with registry access | EKS and other compatible clusters | GKE and other compatible clusters |
| Typical strength | General-purpose image sharing | GitHub-centric workflows | AWS-native deployments | Google Cloud-native deployments |

The registries use broadly compatible container image formats, but their authentication, access-control, repository naming, networking, and lifecycle-management features differ.

## Key points

- **A registry is not an image.** The image is the packaged application environment. The registry is the service that stores and distributes it.

- **A repository is not the same as a registry.** A registry hosts repositories. A repository organizes images for a particular application or project. A repository can contain multiple tags.

- **Understand the full image reference.** A reference can include a registry hostname, namespace or project, repository name, and tag.

  Example:

  ```text
  ghcr.io/my-org/data-pipeline:1.2.0
  |       |       |             |
  |       |       |             +-- Tag
  |       |       +---------------- Repository
  |       +------------------------ Owner or namespace
  +-------------------------------- Registry
  ```

  The exact structure varies by registry. For example, AWS ECR includes an account-specific registry endpoint and a Region, while Google Artifact Registry includes a location and project.

- **Tags are mutable references.** A tag such as `latest` can point to different image content over time. For reproducible deployments, prefer versioned tags and, when appropriate, pin deployments to an image digest.

- **Image digests identify content.** A reference such as `nginx@sha256:...` identifies image content by digest. A digest is useful when deployments must use a specific image version rather than whichever content a tag currently references.

- **Pushing and pulling are different operations.** `docker push` uploads a local image to a registry. `docker pull` downloads an image from a registry to the local machine.

- **Authentication and authorization are separate concerns.** Authentication establishes which identity is making a request. Authorization determines whether that identity is allowed to push, pull, or manage the image.

- **Private repositories require access.** A successful image build does not guarantee that a registry push or pull will succeed. Credentials, repository permissions, and network access must also be configured.

- **Registries reduce duplicated build work.** A team can build an image once, publish it, and deploy that same image across testing, staging, and production environments.

- **Registries support CI/CD workflows.** A pipeline can build an image after a code change, tag it with a version or commit identifier, push it to a registry, scan it, and deploy it.

- **Kubernetes nodes need image access.** The node's container runtime must be able to retrieve the image. Private registry access may require Kubernetes `imagePullSecrets`, node-level identity, or a cloud-specific identity integration.

- **Image layers enable reuse.** Registries and runtimes can reuse existing layers when compatible content is already available, reducing some transfer and storage costs.

- **A successful push does not mean a deployment succeeded.** The image may exist in the registry while the application still fails because of configuration, resource constraints, networking, missing secrets, or an invalid startup command.

- **Security applies to both images and credentials.** Use trusted base images, scan for vulnerabilities, avoid embedding secrets in image layers, restrict push permissions, and protect registry credentials.

- **Registries can support retention and cleanup policies.** Old tags and unreferenced artifacts can consume storage. Retention policies help remove unwanted images, but should be configured carefully to avoid deleting images needed for rollback.

- **Registry selection should match the deployment environment.** Docker Hub works well for general distribution, GHCR fits GitHub-centric workflows, ECR fits AWS environments, and Artifact Registry fits Google Cloud environments. Cross-cloud and hybrid deployments may use any registry that the deployment environment can securely access.

## Commands / manifests

### 1. Log in to Docker Hub

```bash
docker login
```

Download an image:

```bash
docker pull nginx:alpine
```

List local images:

```bash
docker image ls
```

Tag an image for your own Docker Hub repository:

```bash
docker tag nginx:alpine YOUR_DOCKERHUB_USERNAME/my-nginx:1.0
```

Push it:

```bash
docker push YOUR_DOCKERHUB_USERNAME/my-nginx:1.0
```

Pull it using the published reference:

```bash
docker pull YOUR_DOCKERHUB_USERNAME/my-nginx:1.0
```

Replace the placeholder with your Docker Hub username and ensure the target repository exists or can be created under your account.

### 2. Log in to GitHub Container Registry

GHCR commonly uses a GitHub personal access token with the required package permissions when authenticating Docker from the command line. Follow your organization's authentication policy and use a token appropriate to the task.

```bash
export CR_PAT="YOUR_GITHUB_TOKEN"

echo "$CR_PAT" | docker login ghcr.io \
  -u YOUR_GITHUB_USERNAME \
  --password-stdin
```

Avoid committing tokens to source control or placing them directly in shell scripts. For automated workflows, use GitHub Actions secrets or the workflow's supported package permissions.

Tag an existing image:

```bash
docker tag nginx:alpine ghcr.io/YOUR_GITHUB_USERNAME/my-nginx:1.0
```

Push it:

```bash
docker push ghcr.io/YOUR_GITHUB_USERNAME/my-nginx:1.0
```

Pull it:

```bash
docker pull ghcr.io/YOUR_GITHUB_USERNAME/my-nginx:1.0
```

The token must have permission to publish packages for a push. Pulling a private package requires appropriate read access.

### 3. Authenticate with Amazon ECR

Configure AWS credentials using an approved method, such as an AWS CLI profile or an assigned IAM role.

Set the required values:

```bash
export AWS_REGION="ap-south-1"
export AWS_ACCOUNT_ID="YOUR_AWS_ACCOUNT_ID"
export ECR_REPOSITORY="my-nginx"
```

Create the repository if it does not already exist:

```bash
aws ecr create-repository \
  --repository-name "$ECR_REPOSITORY" \
  --region "$AWS_REGION"
```

If the repository already exists, this command may return a repository-exists error. In that case, use the existing repository.

Authenticate Docker with ECR:

```bash
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login \
      --username AWS \
      --password-stdin \
      "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
```

Tag the image:

```bash
docker tag nginx:alpine \
  "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:1.0"
```

Push the image:

```bash
docker push \
  "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:1.0"
```

Pull the image:

```bash
docker pull \
  "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:1.0"
```

The AWS identity needs the appropriate ECR permissions. The repository must be in the intended account and Region.

### 4. Authenticate with Google Artifact Registry

Authenticate using the Google Cloud CLI:

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
```

Enable the Artifact Registry API if it has not already been enabled:

```bash
gcloud services enable artifactregistry.googleapis.com
```

Create a Docker repository:

```bash
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=asia-south1
```

Configure Docker authentication for the relevant registry hostname:

```bash
gcloud auth configure-docker asia-south1-docker.pkg.dev
```

Tag the image:

```bash
docker tag nginx:alpine \
  asia-south1-docker.pkg.dev/YOUR_PROJECT_ID/my-docker-repo/my-nginx:1.0
```

Push the image:

```bash
docker push \
  asia-south1-docker.pkg.dev/YOUR_PROJECT_ID/my-docker-repo/my-nginx:1.0
```

Pull the image:

```bash
docker pull \
  asia-south1-docker.pkg.dev/YOUR_PROJECT_ID/my-docker-repo/my-nginx:1.0
```

Replace the project ID and location with your own values. The authenticated Google Cloud identity must have the required Artifact Registry permissions.

### 5. Inspect image references and digests

Display local image tags and IDs:

```bash
docker image ls
```

Inspect an image:

```bash
docker image inspect nginx:alpine
```

Display its repository digests, when available:

```bash
docker image inspect \
  --format '{{json .RepoDigests}}' nginx:alpine
```

For a remote registry, use the registry's supported inspection tooling or registry API to examine manifests and digests. A locally built image may not have a repository digest until it has been published or otherwise associated with a registry reference.

### 6. Build, tag, and push an application image

Create a `Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

Build the image:

```bash
docker build -t my-data-app:1.0 .
```

Tag it for the target registry:

```bash
docker tag my-data-app:1.0 \
  YOUR_REGISTRY_HOST/YOUR_NAMESPACE/my-data-app:1.0
```

Authenticate with the appropriate registry, then push:

```bash
docker push YOUR_REGISTRY_HOST/YOUR_NAMESPACE/my-data-app:1.0
```

This demonstrates the standard pattern. Replace the placeholder with a complete registry reference supported by the registry you use.

### 7. Deploy an image from a registry in Kubernetes

Example Pod manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: registry-demo
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
```

Apply the manifest:

```bash
kubectl apply -f registry-demo.yaml
```

Check the Pod:

```bash
kubectl get pods
kubectl describe pod registry-demo
```

Inspect the image reference:

```bash
kubectl get pod registry-demo \
  -o jsonpath='{.spec.containers[*].image}'
```

For a private registry, configure the appropriate credentials or workload identity. A common option is an `imagePullSecret`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-demo
spec:
  imagePullSecrets:
    - name: registry-credentials
  containers:
    - name: app
      image: registry.example.com/team/my-app:1.0
```

The secret named `registry-credentials` must already exist in the same namespace as the Pod and contain valid registry credentials. The example registry reference is a placeholder.

In production, use the appropriate cloud identity integration where available rather than distributing long-lived registry passwords unnecessarily.

## Common failures & fixes

- **`unauthorized: authentication required`**
  - **Cause:** Docker is not authenticated, credentials have expired, or the registry requires a different authentication mechanism.
  - **Fix:** Log in using the registry's supported method. For ECR, obtain a fresh authorization token. For GHCR, check the token permissions. For cloud registries, verify the active identity.

- **`denied: requested access to the resource is denied`**
  - **Cause:** The identity lacks permission to push to the repository, the repository path is incorrect, or the target namespace is not accessible.
  - **Fix:** Verify the complete image reference and repository permissions. Ensure the identity has push or write access rather than only pull or read access.

- **`manifest unknown` or `not found`**
  - **Cause:** The requested image repository, tag, or manifest does not exist at the specified location.
  - **Fix:** Verify the registry hostname, namespace, repository, and tag. Confirm that the push completed successfully.

- **The image exists locally but cannot be pulled by another machine.**
  - **Cause:** A locally built image has not been pushed to a registry, or the target machine lacks access to that registry.
  - **Fix:** Tag and push the image, then authenticate and pull it from the other machine.

- **A container keeps running an older version after an image is updated.**
  - **Cause:** Updating a registry tag does not modify an already-created container.
  - **Fix:** Pull the intended image and recreate the container. In Kubernetes, update the workload's image reference or trigger a rollout when using a mutable tag and an appropriate pull policy. Prefer versioned tags or digests for predictable deployments.

- **Kubernetes reports `ImagePullBackOff`.**
  - **Cause:** The image cannot be retrieved because of an invalid reference, missing credentials, insufficient permissions, registry connectivity issues, or rate limits.
  - **Fix:** Run `kubectl describe pod POD_NAME` and examine Events. Check the image reference, registry permissions, credentials, and network access.

- **Pushing an image fails with a tag or repository error.**
  - **Cause:** The image has not been tagged with the intended registry hostname and repository path.
  - **Fix:** Run `docker image ls`, use `docker tag SOURCE_IMAGE TARGET_IMAGE`, and push the exact target reference.

- **A container's files disappear after it is removed.**
  - **Cause:** The data was stored in the container's writable layer.
  - **Fix:** Store persistent data in a volume, bind mount, or external storage service. Rebuild the image if the files should be part of the application package.

- **An image is unexpectedly large.**
  - **Cause:** The build includes unnecessary files, development dependencies, caches, or oversized base-image layers.
  - **Fix:** Use a suitable minimal base image, a `.dockerignore` file, multi-stage builds where appropriate, and layer-efficient build instructions.

- **A registry tag points to unexpected image content.**
  - **Cause:** Tags can be reassigned to different image versions.
  - **Fix:** Verify the image digest and use immutable release tags or digest-pinned references where reproducibility is important.

- **The image works on one machine but fails on another architecture.**
  - **Cause:** The image was built for an incompatible CPU architecture, or it lacks a suitable platform-specific manifest.
  - **Fix:** Build and publish an image for the required platform or publish a multi-platform image using a tool such as `docker buildx`. Verify the target architecture and operating system.

## References

- [Docker Docs: What is an image?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)
- [Docker Docs: What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Docker Docs: Building, tagging, and pushing images](https://docs.docker.com/get-started/introduction/build-and-push-first-image/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Docker Docs: Image digests](https://docs.docker.com/dhi/core-concepts/digests/)
- [Kubernetes Docs: Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Kubernetes Docs: Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
