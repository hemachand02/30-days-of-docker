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
