# Container vs VM

> Phase 1 — Linux, Docker & Networking › Containers

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

A **virtual machine (VM)** virtualizes hardware to run an independent guest operating system, typically using a hypervisor. A **container** isolates an application and its processes while sharing the host's operating system kernel.

Both technologies provide isolation and allow workloads to run in controlled environments, but they operate at different layers of the system.

A VM typically includes virtual hardware, a guest OS kernel, system libraries, and applications. A container typically includes an application, its dependencies, filesystem, and runtime configuration, but it uses the kernel provided by its host environment.

This difference affects startup time, resource overhead, isolation boundaries, portability, and operational management.

## Key points

- **Architecture:** A hypervisor creates and manages VMs with virtual CPUs, memory, storage, and network interfaces. Containers use operating-system mechanisms such as namespaces and cgroups to isolate processes and control resource usage.

- **Operating system:** Each conventional VM runs its own guest OS kernel. Containers on a Linux host generally share that host's Linux kernel, even when individual containers use different user-space distributions.

- **Isolation:** VMs provide a hardware-virtualization boundary, including a separate guest kernel. Containers isolate processes and resources but share a kernel, so a kernel vulnerability or misconfiguration can have broader consequences. Neither model is automatically secure without proper configuration.

- **Resource overhead:** VMs require resources for their guest operating systems and supporting services. Containers generally have less per-workload overhead because they do not each need a separate guest kernel, although containerized applications can still consume substantial CPU, memory, and storage.

- **Startup time:** Containers often start in seconds or less because they do not need to boot a separate guest OS. VM startup depends on the guest OS and configuration, although optimized VMs can also start quickly.

- **Portability:** Container images package application files and dependencies into a standardized image format. They can run across compatible container runtimes and operating systems, but kernel requirements, CPU architecture, runtime behavior, and host configuration still matter.

- **Kernel compatibility:** A Linux container requires a compatible Linux kernel environment. On Windows or macOS, Docker commonly runs Linux containers inside a lightweight Linux VM. A Windows container requires a compatible Windows container environment and kernel configuration.

- **Application dependencies:** Containers help standardize libraries, runtimes, and application files, reducing differences between development and production. They do not automatically eliminate configuration errors, external dependency failures, or environment-specific behavior.

- **Networking:** VMs usually receive virtual network interfaces managed by the hypervisor or cloud platform. Containers use runtime-managed networking, which may provide bridge networks, virtual interfaces, port mappings, or orchestration-managed networking.

- **Storage:** VM disks generally hold the guest OS and application data. Container filesystems often use layered images and a writable container layer. Data that must survive container replacement should be stored in a volume or another persistent storage system.

- **Resource control:** Containers can use cgroups to enforce or account for CPU, memory, and other resource usage. VMs receive virtualized hardware resources, while the hypervisor or platform manages resource allocation and limits.

- **Security trade-off:** Containers are efficient and convenient for application isolation, but share-kernel exposure matters. VMs can provide a stronger isolation boundary for workloads requiring separate kernels. Both need patching, access controls, network restrictions, and least-privilege configuration.

- **Deployment model:** VMs are common for running operating systems, legacy applications, and workloads requiring kernel-level control. Containers are common for microservices, CI/CD pipelines, batch jobs, and cloud-native applications.

- **Orchestration:** Docker can build and run containers on a single host. Kubernetes orchestrates containerized workloads across a cluster, handling scheduling, desired replica counts, service discovery, and recovery. Kubernetes nodes themselves commonly run on VMs or physical machines.

- **They are complementary:** Containers and VMs are not mutually exclusive. A common production architecture runs Kubernetes nodes inside cloud VMs, with containers running application workloads on those nodes.

- **Images versus running instances:** A container image is an immutable, versioned template containing filesystem layers and metadata. A container is a running or created instance of that image with its own runtime configuration and writable layer.

## Commands / manifests

### 1. Inspect the host operating system

```bash
# Identify the operating system and kernel
uname -a
cat /etc/os-release

# Inspect CPU and memory
lscpu
free -h

# Inspect filesystem capacity
df -h
```

These commands help establish which kernel and resources the current environment exposes. When run inside a container, some information may describe the shared host kernel rather than a separate guest OS.

### 2. Run and inspect a container

```bash
# Download an image and run a temporary container
docker run --rm alpine:3.22 echo "Hello from a container"

# Start an interactive Linux container
docker run --rm -it alpine:3.22 sh

# Run a container in the background
docker run -d --name container-demo alpine:3.22 sleep 3600

# Inspect the running container
docker ps
docker inspect container-demo

# Inspect processes inside the container
docker exec container-demo ps

# Inspect its resource usage
docker stats container-demo

# Inspect its configured filesystem mounts
docker inspect container-demo --format '{{json .Mounts}}'

# Remove the container
docker rm -f container-demo
```

The `--rm` flag automatically removes the container after it exits. The `container-demo` example remains running until you stop or remove it.

### 3. Compare container and host process visibility

```bash
# Start a container with a long-running process
docker run -d --name process-demo alpine:3.22 sleep 3600

# List processes visible inside the container
docker exec process-demo ps

# List processes visible from the host
ps aux

# Inspect the container's configured isolation and resource settings
docker inspect process-demo

# Clean up
docker rm -f process-demo
```

The process lists can differ because of process namespaces. A container's process IDs may also differ from the IDs assigned to those processes in the host's PID namespace.

### 4. Inspect container image layers

```bash
# Download an image
docker pull alpine:3.22

# Inspect image metadata
docker image inspect alpine:3.22

# Display image layers and sizes
docker history alpine:3.22

# List local images
docker images
```

Image layers are reused across images where possible. A container's writable layer is separate from the image's read-only layers.

### 5. Inspect VMs on a Linux host

If the host uses KVM and libvirt, the following commands can inspect available virtual machines:

```bash
# List virtual machines
virsh list --all

# Display information about a specific VM
virsh dominfo VM_NAME

# Inspect VM configuration
virsh dumpxml VM_NAME
```

These commands require a configured libvirt environment and suitable permissions. They will not work on every Linux machine or on systems that do not use libvirt.

### 6. Example architecture

```text
Physical Server or Cloud Host
|
+-- Hypervisor
    |
    +-- VM 1
    |   +-- Guest OS kernel
    |   +-- Container runtime
    |       +-- Container A: API service
    |       +-- Container B: Worker
    |
    +-- VM 2
        +-- Guest OS kernel
        +-- Container runtime
            +-- Container C: Database service
```

In this architecture, each VM has its own guest kernel, while the containers running inside a given VM share that VM's kernel.

## Common failures & fixes

- **A container behaves differently from a VM:** Containers share the host kernel and do not independently boot a full guest OS. Check kernel dependencies, filesystem assumptions, permissions, and runtime configuration.

- **A Linux container fails on Windows or macOS:** Verify that Docker's Linux container environment is running. Docker Desktop typically uses a Linux VM to provide the kernel needed for Linux containers.

- **A container exits immediately:** The container stops when its primary process exits. Inspect `docker ps -a` and `docker logs CONTAINER_NAME`. Run a long-lived process when testing interactively.

- **Container data disappears after removal:** Data stored only in the container's writable layer is not a durable persistence strategy. Use a named volume, bind mount, or external storage system for data that must survive replacement.

- **A container cannot access required hardware or kernel functionality:** Containers depend on the host kernel and available device access. Check kernel compatibility, runtime permissions, device configuration, and whether a VM or another isolation model is more appropriate.

- **A container consumes excessive memory or CPU:** Inspect usage with `docker stats` and configure suitable resource limits. Containers are not automatically lightweight if their applications consume substantial resources.

- **A container cannot communicate with another container:** Inspect Docker networks, network membership, DNS resolution, port mappings, firewall rules, and application bind addresses. Containers on separate networks may not be able to communicate directly.

- **A VM has poor performance:** Check CPU and memory allocation, disk I/O, virtualization support, host contention, and guest OS resource usage. Virtualization overhead is only one possible cause.

- **A container image runs on one machine but not another:** Check CPU architecture, image platform, kernel requirements, runtime version, and external configuration. An image does not guarantee identical behavior across all hosts.

- **A container is assumed to be a security boundary equivalent to a VM:** Review the threat model. Containers share a kernel, so apply least privilege, avoid unnecessary capabilities, use non-root users, and consider VM-based isolation for workloads requiring a stronger boundary.

- **A container's `uname -r` output matches the host kernel:** This is expected for ordinary Linux containers. They use the host kernel rather than booting their own independent kernel.

- **A VM cannot be inspected with `virsh`:** The system may not use libvirt, or the command may lack access to the hypervisor connection. Identify the virtualization platform and use its supported management tools.

## References

- [Docker: What is a container?](https://www.docker.com/resources/what-container/)
- [Docker: Containers](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Linux kernel documentation: Namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Linux kernel documentation: Control Groups](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [KVM: Kernel-based Virtual Machine](https://www.linux-kvm.org/page/Main_Page)
- [Kubernetes: Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes: Container Runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
