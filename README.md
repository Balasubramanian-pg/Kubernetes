# Kubernetes

Everything from the roadmap, flattened into a checklist.

## Phase 1 — Linux, Docker & Networking (Week 1)

**Linux**
- [ ] Processes (how they start, run, get killed)
- [ ] Permissions (`chmod`, `chown`, users, groups)
- [ ] Environment variables (setting, reading, scoping)
- [ ] Files & directories (`ls`, `cp`, `mv`, `rm`, `find`, `grep`)
- [ ] Shell commands & pipes (`|`, `>`, `>>`, `&&`, `;`)
- [ ] Reading logs (`tail -f`, `journalctl` basics)

**Containers**
- [ ] What a container actually is vs a VM
- [ ] Images vs containers
- [ ] Registries (Docker Hub, GHCR, ECR, GCR)
- [ ] Image layers & caching
- [ ] Dockerfiles (writing, `FROM`, `RUN`, `COPY`, `CMD`, `ENTRYPOINT`)
- [ ] Multi-stage builds
- [ ] `.dockerignore`

**Networking**
- [ ] Ports & port mapping (`-p host:container`)
- [ ] DNS basics (how names resolve)
- [ ] HTTP vs TCP/IP
- [ ] Container networking (bridge, host, overlay — at least conceptually)

**Docker runtime**
- [ ] Volumes (bind mounts vs named volumes)
- [ ] Environment-based configuration (`-e`, `--env-file`)
- [ ] Inspecting a running container (`docker logs`, `docker exec`, `docker inspect`)
- [ ] Building, tagging, pushing an image

**Hands-on:** Containerize a Python ETL (CSV → Pandas → Parquet). Pass config via env vars, inspect logs.

## Phase 2 — Kubernetes Core (Weeks 2–3)  *highest priority*

**Cluster model**
- [ ] Cluster / nodes / control plane
- [ ] Where workloads actually run
- [ ] API server, etcd, scheduler, kubelet, kube-proxy (what each does)
- [ ] `kubectl` context & config

**Workload primitives**
- [ ] **Pods** — smallest deployable unit
- [ ] **Deployments** — declarative pod management
- [ ] **ReplicaSets** — how Deployments use them
- [ ] **Jobs** — run-to-completion batch work
- [ ] **CronJobs** — scheduled batch work
- [ ] **DaemonSets** — one pod per node (concept only)
- [ ] **StatefulSets** — concept only (rarely needed in DE)

**Networking**
- [ ] **Services** — ClusterIP, NodePort, LoadBalancer
- [ ] Service DNS (how pods find each other)
- [ ] Ingress (concept)
- [ ] `port-forward` for debugging

**Configuration**
- [ ] **Namespaces** — dev / test / prod separation
- [ ] **ConfigMaps** — non-secret config
- [ ] **Secrets** — credentials, tokens, keys
- [ ] Mounting config as env vars vs files

**Resources & health**
- [ ] **Requests** — guaranteed CPU/memory
- [ ] **Limits** — max CPU/memory
- [ ] **Liveness probes** — restart if dead
- [ ] **Readiness probes** — don't send traffic until ready
- [ ] **Startup probes** — slow-starting apps

**Storage**
- [ ] **PersistentVolumes (PV)**
- [ ] **PersistentVolumeClaims (PVC)**
- [ ] StorageClasses (concept)
- [ ] When you need PVs vs object storage

**kubectl fluency**
- [ ] `kubectl get`, `describe`, `logs`, `exec`
- [ ] `kubectl apply -f`, `delete`
- [ ] `kubectl port-forward`
- [ ] `kubectl get events`, `kubectl explain`
- [ ] `kubectl get pods -w` (watch)
- [ ] Reading YAML manifests

**Hands-on:** Deploy Python ETL as a `Job`, schedule it with a `CronJob`, deliberately break config, troubleshoot.

## Phase 3 — Data Workloads on Kubernetes (Weeks 4–5)

**A. Python ETL & batch**
- [ ] Containerizing reusable ETL code
- [ ] Passing job parameters (env vars, args, ConfigMaps)
- [ ] Managing Python dependencies in images
- [ ] Writing to object storage (S3/GCS/ADLS) from a pod
- [ ] Making retries **idempotent** (safe to re-run)

**B. Apache Spark on Kubernetes**
- [ ] Spark driver vs executors
- [ ] Executor pods (how Spark creates them on K8s)
- [ ] `spark-submit` on Kubernetes
- [ ] Cluster mode vs client mode
- [ ] Resource allocation (driver memory, executor memory, cores)
- [ ] Reading driver logs & executor logs
- [ ] Service account for the driver
- [ ] Dynamic allocation (concept)
- [ ] Where Spark writes output (object storage vs PVC)

**C. Airflow integration**
- [ ] How Airflow schedules tasks (DAG → task → executor)
- [ ] `KubernetesExecutor` — one pod per task
- [ ] `KubernetesPodOperator` — run arbitrary container as a task
- [ ] Task-specific resources (CPU/memory per task)
- [ ] Passing XComs / parameters to pods
- [ ] Log retrieval from pod logs

**Hands-on:** PySpark transformation on K8s → Parquet to object storage → orchestrated from Airflow.

## Phase 4 — Production Readiness (Weeks 6–7)

**Resource management**
- [ ] Requests vs limits in depth
- [ ] CPU throttling (what it looks like, how to spot it)
- [ ] `OOMKilled` — causes, diagnosis, fix
- [ ] QoS classes (Guaranteed / Burstable / BestEffort)

**Security**
- [ ] RBAC — Roles, RoleBindings, ClusterRoles
- [ ] ServiceAccounts
- [ ] Secrets — best practices (not baked into images)
- [ ] Least-privilege access
- [ ] Workload identity (IRSA / Workload Identity / Managed Identity) for object storage access

**Storage & data access**
- [ ] Persistent storage — when it's actually needed
- [ ] Object storage access from pods (S3/GCS/ADLS SDKs, credentials)
- [ ] Workload identity vs static keys

**Health & reliability**
- [ ] Liveness / readiness / startup probes (deep dive)
- [ ] Probe tuning — thresholds, periods, failure limits

**Observability**
- [ ] Metrics (what to collect from data workloads)
- [ ] Logs (centralized collection)
- [ ] Events (`kubectl get events`)
- [ ] Prometheus basics
- [ ] Grafana basics
- [ ] Debugging a `CrashLoopBackOff`
- [ ] Debugging a `Pending` pod
- [ ] Debugging an `OOMKilled` job

**Deployment & operations**
- [ ] Helm charts — structure (`Chart.yaml`, `values.yaml`, `templates/`)
- [ ] Release configuration
- [ ] Rolling updates
- [ ] Rollbacks

**Job reliability**
- [ ] Retry policies (`backoffLimit`, `restartPolicy`)
- [ ] Idempotency patterns
- [ ] Failure recovery
- [ ] Job cleanup (`ttlSecondsAfterFinished`)

**Hands-on:** Diagnose `CrashLoopBackOff`, `Pending` pod, `OOMKilled` job. Explain root cause + fix each.

## Phase 5 — Capstone (Week 8)

**Build a mini lakehouse ingestion pipeline that uses every layer:**

- [ ] CSV / JSON source data
- [ ] Dockerized Python ingestion
- [ ] Kubernetes `Job` **or** Spark application
- [ ] Object storage output — Parquet / Delta
- [ ] Airflow orchestration
- [ ] Logs + monitoring

**Deliverables you must produce:**
- [ ] Dockerfile + dependency definitions (`requirements.txt` / `pyproject.toml`)
- [ ] Kubernetes YAML manifests **or** Helm chart
- [ ] Batch pipeline with retries + parameterized runs
- [ ] Data-quality checks
- [ ] Failure handling
- [ ] `README` — architecture, setup, commands, troubleshooting

## Explicitly **SKIP** (for now)

- ❌ CKA certification prep
- ❌ Service mesh (Istio, Linkerd)
- ❌ Advanced networking (CNI plugins, network policies in depth)
- ❌ Writing custom Kubernetes operators / CRDs
- ❌ Installing a production-grade multi-node cluster
- ❌ Kubernetes admin / control-plane management

## The one distinction that matters

> **Kubernetes orchestrates containers and their resources.** It does **not** replace:
> - Spark's distributed execution model
> - Airflow's workflow orchestration
> - A lakehouse's storage & table formats

If you remember only one thing: **K8s = container scheduler + resource manager + service discovery**. Everything else is Spark's job, Airflow's job, or the storage layer's job.

## Minimum viable milestone (end of week 2)

Run a Python ETL inside a Kubernetes `Job`, pass input/output paths via config, inspect logs, diagnose a failed run. That's the bar — not memorizing 50 resource types.
