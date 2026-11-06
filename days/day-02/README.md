# Day 02 — Traditional Deployment vs Virtual Machines vs Containers

**Phase:** Containerization Fundamentals  
**Day:** 02 / 30  
**Focus:** Understanding deployment architectures, virtualization and containerization  
**Learning Type:** Conceptual Learning + Hands-on Experiments

---

## 1. Learning Objectives

The objectives of this session are to:

- Understand traditional application deployment.
- Understand hardware virtualization and hypervisors.
- Differentiate host operating systems and guest operating systems.
- Understand containerization and shared kernel architecture.
- Compare resource allocation, isolation and overhead across deployment models.
- Understand how Docker Desktop uses WSL 2 to run Linux containers.
- Experiment with Linux kernel visibility and container resource limits.

## 2. Traditional Deployment

### 2.1 What Is Traditional Deployment?

Traditional deployment involves installing and running applications directly on the host operating system.

Applications use the host operating system's kernel, system resources, libraries and installed dependencies.

### 2.2 Architecture

```text
Physical Hardware
       |
       v
Host Operating System
       |
       +---- Application A (Java 17)
       |
       +---- Application B (Java 21)
       |
       +---- Application C (Python)
```

All applications execute within the same host operating system environment and share its kernel.

### 2.3 How It Works

1. The host operating system boots and initializes its kernel.
2. Applications are installed with their required dependencies.
3. Applications execute as processes.
4. The operating system kernel manages CPU scheduling, memory, filesystems and other system resources.

### 2.4 Limitations

- Dependency conflicts between applications.
- Environment inconsistencies.
- Manual dependency and configuration management.
- Potential resource contention.
- Difficulty reproducing identical environments across machines.

Traditional deployment is still useful for applications that require direct host access or simple deployment environments.

## 3. Virtualization — Virtual Machines

### 3.1 What Is Virtualization?

Hardware virtualization allows multiple virtual computers to run on a single physical computer.

Each virtual machine (VM) behaves like an independent computer with virtual hardware resources.

A VM can have its own:

- Virtual CPU.
- Virtual memory.
- Virtual disk.
- Virtual network interface.
- Guest operating system.

### 3.2 What Is a Hypervisor?

A hypervisor is software responsible for creating, running and managing virtual machines.

It manages the virtual hardware presented to each VM and coordinates access to the physical machine's resources.

### 3.3 Types of Hypervisors

| Type | Description | Example |
|---|---|---|
| Type 1 | Runs directly on physical hardware (bare-metal hypervisor). | VMware ESXi, Microsoft Hyper-V |
| Type 2 | Runs as an application on a host operating system. | Oracle VirtualBox |

### 3.4 Host OS vs Guest OS

**Host Operating System:** The operating system running on the physical machine.

**Guest Operating System:** The operating system installed and running inside a virtual machine.

For example:

- Physical machine: Windows.
- Hypervisor: VirtualBox.
- VM 1: Ubuntu Linux.
- VM 2: Windows Server.

Each VM has its own operating system kernel.

### 3.5 Virtual Machine Architecture

```text
Physical Hardware
       |
       v
    Hypervisor
       |
       +----------------------+
       |                      |
       v                      v
 Virtual Machine 1       Virtual Machine 2
       |                      |
 Virtual Hardware        Virtual Hardware
       |                      |
 Guest OS 1              Guest OS 2
       |                      |
 Application A           Application B
```

### 3.6 Resource Allocation

The hypervisor provides virtual resources to virtual machines.

Example:

| Resource | VM 1 | VM 2 |
|---|---:|---:|
| vCPU | 2 | 2 |
| RAM | 2 GB | 4 GB |
| Virtual Disk | 20 GB | 40 GB |

Each guest operating system requires resources for its kernel, system services and applications.

### 3.7 Advantages

- Each VM has an independent guest operating system and kernel.
- Different operating systems can run on the same physical machine.
- Strong separation between virtual machine environments.
- Useful for legacy applications and workloads requiring different operating systems.

### 3.8 Disadvantages

- Higher resource consumption due to guest operating systems.
- More memory and storage requirements.
- Additional operating-system maintenance.
- VM startup requires booting the guest operating system.

## 4. Containerization

### 4.1 What Is Containerization?

Containerization is an operating-system-level virtualization technique that packages applications and their required user-space dependencies into isolated execution environments called containers.

Unlike conventional virtual machines, Linux containers do not boot independent Linux kernels.

Instead, they share the underlying host Linux kernel.

### 4.2 Container Architecture

```text
Physical Hardware
       |
       v
Host Operating System
       |
Linux Kernel
       |
Container Runtime / Engine
       |
       +----------------------+
       |          |           |
       v          v           v
 Container 1  Container 2  Container 3
       |          |           |
      App A      App B       App C
       |          |           |
    Java 17     Java 21      Python
```

All Linux containers use the same underlying Linux kernel.

Each container can have its own user-space filesystem and application dependencies.

### 4.3 How Containers Work

1. A container image provides the application and its user-space dependencies.
2. The container runtime creates an isolated container environment.
3. The application runs as a process managed by the host Linux kernel.
4. Kernel isolation mechanisms separate process, network and filesystem views.
5. Resource-control mechanisms manage CPU, memory and other resource consumption.

### 4.4 Linux Kernel Mechanisms

Two important mechanisms are used to implement container isolation and resource management.

#### A. Namespaces — Isolation

Namespaces control what processes can see.

| Namespace | Responsibility |
|---|---|
| PID | Isolates process IDs and process visibility. |
| Network | Isolates network interfaces, routes and network stacks. |
| Mount | Isolates filesystem mount views. |
| UTS | Isolates hostname and domain-name views. |
| IPC | Isolates inter-process communication resources. |
| User | Isolates user and group ID mappings. |

#### B. Control Groups (cgroups) — Resource Management

Control groups account for and control resource consumption by groups of processes.

They can manage:

- CPU consumption.
- Memory consumption.
- I/O usage.

Namespaces and cgroups have different responsibilities: namespaces provide isolation, while cgroups provide resource accounting and control.

### 4.5 Advantages

- Lower overhead than running a complete guest OS for every application.
- Faster container startup.
- Efficient resource utilization.
- Consistent application packaging.
- Easy creation of multiple application instances.
- Useful for microservices, development environments and CI/CD workflows.

### 4.6 Limitations

- Containers share the host kernel.
- They do not provide the same kernel separation as conventional VMs.
- Kernel vulnerabilities and incorrect privilege configurations can create security risks.
- Container portability still depends on kernel compatibility, CPU architecture and external configuration.

## 5. Comparison of Deployment Models

| Feature | Traditional Deployment | Virtual Machines | Containers |
|---|---|---|---|
| Application execution | Directly on host OS | Inside guest OS | Inside isolated environment |
| Kernel | Shared host kernel | Separate guest kernel per VM | Shared host kernel |
| Guest OS | No | Yes | No |
| Isolation | OS process and permission boundaries | Hypervisor and guest OS boundary | Kernel namespaces and other security controls |
| Resource overhead | Usually low | Generally higher | Generally lower than full VMs |
| Startup | Application startup | Guest OS boot + application startup | Usually fast |
| Portability | Environment dependent | VM and hypervisor compatibility | Image and kernel/runtime compatibility |
| Resource management | Host OS | Hypervisor and guest OS | Host kernel and container runtime |
| Suitable for | Direct host workloads | Different OS environments | Application packaging and deployment |

**Important:** These are general comparisons. Actual isolation, performance and overhead depend on implementation and configuration.

## 6. Resource Management and Isolation

Consider three applications running on a physical computer.

### Traditional Deployment

Applications execute directly on the host operating system.

They share the host kernel and system environment.

### Virtual Machines

Each VM has its own guest OS, kernel and system services.

This provides separate operating-system environments but consumes additional resources.

### Containers

Containers share the host kernel but have isolated execution environments.

They can be configured with CPU and memory limits using cgroups.

### Key Difference

VMs virtualize hardware resources and run independent guest operating systems.

Containers use operating-system-level isolation while sharing the underlying kernel.

## 7. Docker Desktop with WSL 2

### 7.1 Understanding the Architecture

Docker Desktop on Windows uses WSL 2 to provide a Linux environment for running Linux containers.

The approximate architecture is:

```text
Windows Host Operating System
          |
          v
       WSL 2
   (Lightweight VM)
          |
          v
     Linux Kernel
          |
          v
      Docker Engine
          |
          +------------------+
          |                  |
          v                  v
     Container A        Container B
```

Docker Desktop uses its own WSL distribution to run the Docker Engine.

The Ubuntu WSL distribution can communicate with Docker Desktop through its integration.

### 7.2 Important Observations

- Windows is the physical machine's host operating system.
- WSL 2 provides a Linux kernel through a lightweight virtual-machine environment.
- Docker Engine runs in Docker Desktop's Linux environment.
- Linux containers share the Linux kernel provided by that environment.
- Virtualization and containerization can work together.

This demonstrates that containers and VMs are not mutually exclusive technologies.

## 8. Hands-on Experiments

The following experiments are designed to investigate kernel visibility and container resource boundaries using Docker Desktop and Ubuntu WSL.

### Experiment 1 — Identify the Environment

**Objective:** Identify the Linux kernel and active Docker context.

Commands:

```bash
uname -r
```

```bash
docker context show
```

```bash
docker info --format 'Operating System: {{.OperatingSystem}}
Kernel Version: {{.KernelVersion}}
CPUs: {{.NCPU}}
Memory: {{.MemTotal}}'
```

**Expected observations:**

- Identify the Linux kernel visible to Ubuntu WSL.
- Identify the active Docker context.
- Observe the operating system and kernel reported by Docker Engine.
- Observe the CPU and memory resources available to Docker Engine.

**Execution Status:** Pending.

### Experiment 2 — Investigate Container Kernel Visibility

**Objective:** Observe the kernel version visible inside a Linux container.

Command:

```bash
docker run --rm alpine:latest uname -r
```

Compare this output with:

```bash
uname -r
```

and the kernel version reported by:

```bash
docker info
```

**Expected observation:**

Linux containers do not boot independent kernels. They use the kernel provided by their Docker Engine's Linux environment.

The kernel versions visible to different WSL distributions and Docker may not necessarily be identical in every setup.

**Execution Status:** Pending.

### Experiment 3 — Investigate Container Resource Limits

**Objective:** Understand how Docker applies memory and CPU limits.

Command:

```bash
docker run --rm \
  --memory=128m \
  --cpus=0.5 \
  alpine:latest \
  sh -c 'echo "Memory limit:"; cat /sys/fs/cgroup/memory.max; echo "CPU quota:"; cat /sys/fs/cgroup/cpu.max'
```

**Explanation:**

- `--memory=128m` sets a 128 MiB memory limit.
- `--cpus=0.5` limits the container to half a CPU's worth of CPU time.
- `memory.max` reports the cgroup memory limit.
- `cpu.max` reports the CPU quota and period.

On a cgroup v2 system, the memory limit may be displayed as `134217728` bytes.

**Expected observation:**

The container can have resource restrictions without requiring an independent guest operating system.

**Execution Status:** Pending.

## 9. Key Technical Observations

1. Traditional deployment runs applications directly on the host operating system.
2. Virtual machines provide virtual hardware and independent guest operating systems.
3. Every conventional VM has its own guest OS kernel.
4. Linux containers share the underlying Linux kernel.
5. Namespaces isolate process and resource views.
6. Cgroups account for and control resource consumption.
7. Containers generally have less overhead than full VMs because they do not require separate guest operating systems.
8. Docker Desktop can use WSL 2 virtualization to provide the Linux environment required for Linux containers.
9. Containerization and virtualization can work together.

## 10. Key Takeaways

The major distinction between these deployment models is their architecture and isolation boundary.

- **Traditional deployment:** Applications run directly on the host OS.
- **Virtualization:** Multiple guest operating systems run through a hypervisor.
- **Containerization:** Applications run in isolated environments while sharing the underlying kernel.

Virtual machines provide independent operating-system environments, while containers provide lightweight application isolation.

Understanding this distinction is essential before exploring Docker architecture and Linux container internals.

## 11. References

- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Overview](https://docs.docker.com/get-started/docker-overview/)
- [Docker Desktop WSL 2 Backend](https://docs.docker.com/desktop/features/wsl/)
- [Microsoft — What is WSL?](https://learn.microsoft.com/en-us/windows/wsl/about)
- [Docker — Running Containers](https://docs.docker.com/engine/containers/run/)
- [Linux Kernel — Namespaces](https://docs.kernel.org/admin-guide/namespaces/)
- [Linux Kernel — Control Groups](https://docs.kernel.org/admin-guide/cgroup-v2.html)

---

**Next Session:** Day 03 — Docker Architecture: Client, Daemon, Engine and Registry

**Learning Journey:** 30 Days of Docker