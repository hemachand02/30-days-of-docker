# Day 07 --- Linux Namespaces: How Containers Isolate Processes

**30 Days of Docker · Day 07 · `days/day-07/README.md`**

In Day 06, we learned that a container is a group of Linux processes and
that its main process normally appears as PID 1 inside the container.
This raises an important question: if containers share the host's Linux
kernel, how can each container have its own process view, hostname,
network stack, and filesystem mount view?

The answer includes **Linux namespaces**.

> Namespaces isolate a process's view of selected system resources. They
> help make a group of processes behave as though it has its own
> environment, without giving it a separate kernel.

## 1. Learning objectives

By the end of this lesson, you should be able to:

-   Explain why containers need isolation.
-   Define Linux namespaces and describe what they isolate.
-   Identify the main namespace types used by containers.
-   Explain PID, mount, network, UTS, IPC, user, and cgroup namespaces.
-   Understand that namespaces isolate views, not physical machines or
    kernels.
-   Inspect namespace information through Linux and Docker.
-   Run safe namespace-observation experiments using Alpine.
-   Explain the limits of namespaces and why they are not the entire
    container security model.

## 2. Why do containers need namespaces?

Imagine two applications running on the same Linux host:

-   Application A runs inside Container A.
-   Application B runs inside Container B.

You generally want Container A not to see Container B's processes, each
container to have its own hostname and network view, and each container
to have an appropriately isolated filesystem mount view.

Linux namespaces provide separate views of selected resources.

``` text
                 Host Linux Kernel
                        |
          +-------------+-------------+
          |                           |
      Container A                 Container B
          |                           |
      PID namespace               PID namespace
      Mount namespace             Mount namespace
      Network namespace           Network namespace
      UTS namespace               UTS namespace
          |                           |
      Application A               Application B
```

This is conceptual. The actual namespaces used depend on Docker's
configuration and the host environment.

## 3. What is a Linux namespace?

A **Linux namespace** is a kernel feature that wraps a global system
resource so that processes in different namespaces see different
instances or views of that resource.

A process belongs to one namespace of each supported type. It can
therefore belong simultaneously to a PID namespace, mount namespace,
network namespace, UTS namespace, IPC namespace, user namespace, and
cgroup namespace.

Namespaces are created and managed through Linux system calls and
interfaces such as `clone()`, `unshare()`, and `setns()`. Docker and its
runtime arrange the required namespaces; ordinary Docker users do not
need to call these system calls manually.

## 4. Main Linux namespace types

  -----------------------------------------------------------------------
  Namespace               Isolates or provides a  Why it matters
                          separate view of        
  ----------------------- ----------------------- -----------------------
  PID                     Process IDs and process Processes see their own
                          tree                    process hierarchy

  Mount (MNT)             Filesystem mount points A container can have
                                                  its own filesystem
                                                  mount view

  Network (NET)           Network devices,        Containers can have
                          routes, IP addresses,   separate network
                          ports, and network      interfaces and routing
                          stack                   

  UTS                     Hostname and NIS domain A container can have
                          name                    its own hostname

  IPC                     System V IPC objects    IPC resources can be
                          and POSIX message       isolated
                          queues                  

  User (USER)             User and group ID       A process can have a
                          mappings                different identity view
                                                  inside a container

  Cgroup (CGROUP)         View of the cgroup      A process can see a
                          hierarchy               cgroup hierarchy rooted
                                                  at its namespace
  -----------------------------------------------------------------------

### 4.1 PID namespace --- process isolation

A PID namespace provides a separate view of process IDs and the process
hierarchy.

-   The container's main process normally appears as PID 1 inside its
    PID namespace.
-   Processes in one PID namespace are isolated from the process view of
    another.
-   A process can have different PID values when viewed from nested PID
    namespaces.

``` text
Host PID namespace
|
+-- Host PID 4200: container main process
|       |
|       +-- Container PID 1: application
|               |
|               +-- Container PID 2: worker
|
+-- Host PID 5100: another container process
        |
        +-- Container PID 1: another application
```

The numbers are illustrative only.

**Why useful?** An application in one container normally cannot
enumerate another container's processes through its own `/proc` process
view.

### 4.2 Mount namespace --- filesystem mount isolation

A mount namespace gives a process its own view of filesystem mount
points. A mount point is a location in the directory tree where a
filesystem is attached.

``` text
Container A mount view       Container B mount view
/                            /
+-- app/                     +-- app/
+-- data/ -> Volume A        +-- data/ -> Volume B
```

The mount namespace isolates the mount arrangement, not necessarily the
underlying data. Docker images, writable layers, bind mounts, and
volumes work together with mount namespaces to construct a container's
filesystem view.

**Important:** A mount namespace does not automatically make every file
private. Host directories can intentionally be exposed using bind
mounts, and volumes can be shared.

### 4.3 Network namespace --- network isolation

A network namespace provides a separate network stack. It can contain
its own:

-   Network interfaces.
-   IP addresses.
-   Routing tables.
-   Firewall rules and network-related configuration.
-   Network sockets and port-number space.

``` text
Container A network namespace     Container B network namespace
    eth0                              eth0
    IP: 172.18.0.2                    IP: 172.18.0.3
    Routes                            Routes
```

The addresses are illustrative. Actual addresses depend on Docker
networking configuration.

Docker commonly connects container network namespaces to Docker networks
using virtual Ethernet interfaces and other networking components.

**Important:** Network namespace isolation does not mean a container has
no network access. Docker can connect it to other containers, the host,
or external networks according to its network configuration.

### 4.4 UTS namespace --- hostname isolation

UTS stands for UNIX Time-sharing System. The UTS namespace isolates the
hostname and NIS domain name.

``` bash
docker run --rm --hostname day07-lab alpine:latest hostname
```

Expected output:

``` text
day07-lab
```

The output is an expectation, not a recorded experiment. A container's
hostname can differ from the host's hostname because it has a separate
UTS namespace.

### 4.5 IPC namespace --- inter-process communication isolation

IPC means **Inter-Process Communication**. Linux IPC mechanisms include:

-   System V shared memory.
-   System V message queues.
-   System V semaphore arrays.
-   POSIX message queues.

An IPC namespace isolates these resources so processes in different IPC
namespaces do not automatically share the same IPC objects.

### 4.6 User namespace --- user and group identity isolation

The user namespace isolates user IDs and group IDs and supports mappings
between namespace IDs and host IDs.

For example, a process can appear as UID `0` (root) inside a user
namespace while being mapped to a non-zero, less privileged UID on the
host, when configured.

``` text
Inside user namespace        On the host
UID 0 (root)          --->    UID 100000
UID 1                 --->    UID 100001
UID 2                 --->    UID 100002
```

These numbers are examples only; actual mappings depend on host and
Docker configuration.

User namespaces can reduce the impact of container process privileges on
the host. Do not assume they are enabled in every Docker installation or
container. Container root is not automatically equivalent to
unrestricted host root, but running as root without user namespace
remapping can still carry significant risk.

### 4.7 Cgroup namespace --- cgroup hierarchy view

A cgroup namespace virtualizes the view of the cgroup hierarchy shown to
processes.

Cgroups (control groups) are different from namespaces:

-   Namespaces isolate what a process can see.
-   Cgroups organize processes into groups and control or account for
    resource usage.

We will study cgroups and resource controls in a dedicated lesson.

## 5. Namespaces vs cgroups

  -----------------------------------------------------------------------
  Feature                 Main question           Example
  ----------------------- ----------------------- -----------------------
  Namespaces              What can this process   Which processes,
                          see?                    network interfaces,
                                                  mounts, and hostnames
                                                  are visible?

  Cgroups                 How much resource can   CPU, memory, and
                          this group use, and how process-count limits
                          is usage accounted for? 
  -----------------------------------------------------------------------

A useful mental model:

-   **Namespaces = isolation of views.**
-   **Cgroups = resource organization, accounting, and control.**

Neither feature alone provides the complete container security model.
Docker also uses capabilities, security profiles, filesystem controls,
and other kernel mechanisms.

## 6. How Docker uses namespaces

At a high level:

1.  Docker receives the container configuration.
2.  The runtime prepares the container's filesystem and settings.
3.  The runtime configures namespaces and other isolation mechanisms.
4.  The container's main process is started in that environment.
5.  The process sees the resource views provided by its namespaces.

The implementation involves components such as containerd and an OCI
runtime (commonly `runc` on Linux-based installations). Docker Desktop
adds another layer because the Linux Docker Engine runs in a Linux
environment managed by Docker Desktop.

You do not need to manually create namespaces for ordinary Docker usage.
Docker handles this configuration.

## 7. Inspect namespace information on Linux

Linux exposes namespace information through the `/proc` filesystem. For
a process, namespace links are commonly visible under:

``` bash
ls -l /proc/<PID>/ns/
```

Replace `<PID>` with a real process ID.

Inspect the current process:

``` bash
ls -l /proc/self/ns/
```

Read selected namespace identifiers:

``` bash
readlink /proc/self/ns/pid
readlink /proc/self/ns/net
readlink /proc/self/ns/mnt
readlink /proc/self/ns/uts
```

A namespace link often displays an identifier similar to:

``` text
pid:[402653xxxx]
```

The number is illustrative. It is a kernel namespace inode identifier,
not a container ID.

Inspect namespace links inside a container:

``` bash
docker run --rm alpine:latest sh -c 'ls -l /proc/self/ns/ && readlink /proc/self/ns/pid && readlink /proc/self/ns/net && readlink /proc/self/ns/mnt && readlink /proc/self/ns/uts'
```

Namespace IDs can help compare whether processes share a namespace. Do
not assume every namespace will differ: Docker can intentionally share
namespaces, and settings can be configured differently.

## 8. Hands-on experiments

> **Status: Pending.** These are practical tasks to run and document. No
> experiment is claimed to have been performed. Expected observations
> are predictions, not recorded results.

**Environment:** Ubuntu on WSL, with Docker Desktop running and the
Docker CLI connected to the intended Docker Engine.

The experiments use Alpine and are designed to be read-only or
short-lived. They do not require privileged containers or changes to
host namespaces.

### Experiment 1 --- Compare host and container hostnames

**Objective:** Observe UTS hostname isolation.

Display the hostname in Ubuntu:

``` bash
hostname
```

Run a container with an explicit hostname:

``` bash
docker run --rm --hostname day07-container alpine:latest hostname
```

Run another container with a different hostname:

``` bash
docker run --rm --hostname second-container alpine:latest hostname
```

**Expected observations:**

-   Each container command should print the hostname assigned to that
    container.
-   The two container hostnames should differ.
-   The host's hostname should remain unchanged.

### Experiment 2 --- Compare PID views

**Objective:** Observe that a container has its own process-ID view.

Start a long-running container:

``` bash
docker run -d --name day07-pid alpine:latest sleep 300
```

Inspect processes inside it:

``` bash
docker exec day07-pid ps
```

Inspect the container's processes from Docker:

``` bash
docker top day07-pid
```

Inspect the main process's host PID from container metadata:

``` bash
docker inspect --format 'Host PID={{.State.Pid}}' day07-pid
```

Clean up:

``` bash
docker stop day07-pid
docker rm day07-pid
```

**Expected observations:**

-   The main `sleep` process should normally appear as PID 1 inside the
    container.
-   `docker top` displays process information from the Docker host-side
    view.
-   The host PID reported by inspection may differ from the PID visible
    inside the container.

With Docker Desktop, the host-side process view refers to the Linux
environment running the Docker Engine, not necessarily the Windows
host's native process list.

### Experiment 3 --- Inspect namespace identifiers

**Objective:** Read namespace identifiers from `/proc`.

Inspect the current Ubuntu environment:

``` bash
ls -l /proc/self/ns/
```

Read selected namespace identifiers:

``` bash
readlink /proc/self/ns/pid
readlink /proc/self/ns/net
readlink /proc/self/ns/mnt
readlink /proc/self/ns/uts
```

Run the same checks in a container:

``` bash
docker run --rm alpine:latest sh -c 'ls -l /proc/self/ns/ && readlink /proc/self/ns/pid && readlink /proc/self/ns/net && readlink /proc/self/ns/mnt && readlink /proc/self/ns/uts'
```

**Expected observations:**

-   Both environments should expose namespace links through `/proc`.
-   Namespace identifiers may differ between the Ubuntu environment and
    the container.
-   Do not expect every identifier to differ in every configuration.
    Compare actual outputs rather than assuming isolation based only on
    container names.

### Experiment 4 --- Observe a container's network configuration

**Objective:** See network information visible inside a container.

Try:

``` bash
docker run --rm alpine:latest ip addr
```

If `ip` is not installed in the image, do not treat that as a Docker
failure. Inspect Docker network metadata instead:

``` bash
docker network ls
```

Create a temporary container:

``` bash
docker run -d --name day07-network alpine:latest sleep 300
```

Inspect its network settings:

``` bash
docker inspect --format '{{json .NetworkSettings.Networks}}' day07-network
```

Clean up:

``` bash
docker stop day07-network
docker rm day07-network
```

**Expected observations:**

-   The container should have a network configuration assigned by
    Docker.
-   Exact interfaces, addresses, and routes depend on the Docker network
    driver and environment.
-   Docker Desktop may add networking layers between the container and
    the physical network.

## 9. Troubleshooting checklist

  -----------------------------------------------------------------------
  Problem                             What to investigate
  ----------------------------------- -----------------------------------
  `/proc/self/ns` is missing          Confirm that `/proc` is mounted and
                                      available

  `readlink` is not found             Check whether the image includes
                                      the utility; BusyBox commonly
                                      provides it

  `ip` is not found                   Minimal images may not include
                                      iproute2

  Host and container namespace IDs    Check which process and namespace
  look the same                       you inspected; some namespaces can
                                      be shared

  Container hostname does not match   A container name and hostname are
  its name                            separate settings

  Container has no expected IP        Inspect its Docker network, driver,
  address                             and network mode

  Docker commands cannot connect      Check Docker Desktop, WSL
                                      integration, and
                                      `docker context show`
  -----------------------------------------------------------------------

Useful diagnostic commands:

``` bash
docker context show
docker info
docker ps -a
docker inspect <container>
docker network ls
docker network inspect bridge
```

Replace `<container>` with the actual container name or ID.

## 10. Security boundaries and limitations

Namespaces are important, but they are not a complete security boundary
by themselves.

A container shares the host kernel. A kernel vulnerability or unsafe
configuration can potentially undermine isolation.

Container security also depends on:

-   **Linux capabilities:** which privileged operations a process can
    perform.
-   **Seccomp:** restricting the system calls a process can make.
-   **AppArmor or SELinux:** mandatory access-control policies.
-   **User namespaces:** UID/GID mappings.
-   **Read-only filesystems and mounts:** limiting filesystem changes
    and exposure.
-   **Cgroups:** resource controls and accounting.
-   Avoiding unnecessary privileged mode and unnecessary host mounts.

**Practical rule:** Do not use `--privileged` simply to make a container
command work. It grants broad additional access and weakens isolation.

## 11. Key takeaways

1.  Containers share the host's Linux kernel; they do not each run a
    separate kernel.
2.  Linux namespaces isolate views of selected system resources.
3.  PID namespaces provide separate process-ID and process-tree views.
4.  Mount namespaces provide separate filesystem mount views.
5.  Network namespaces provide separate network stacks.
6.  UTS namespaces isolate hostnames and NIS domain names.
7.  IPC namespaces isolate IPC resources.
8.  User namespaces provide user and group ID mappings.
9.  Cgroup namespaces virtualize the view of the cgroup hierarchy.
10. Namespaces answer "what can this process see?"; cgroups answer "how
    are resources organized and controlled?"
11. Namespaces are one part of container isolation, not the complete
    security model.

## 12. Official documentation and references

-   [Linux man page ---
    namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
-   [Linux man page ---
    pid_namespaces(7)](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)
-   [Linux man page ---
    mount_namespaces(7)](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)
-   [Linux man page ---
    network_namespaces(7)](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)
-   [Linux man page ---
    user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)
-   [Linux man page ---
    uts_namespaces(7)](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html)
-   [Linux man page ---
    ipc_namespaces(7)](https://man7.org/linux/man-pages/man7/ipc_namespaces.7.html)
-   [Linux man page ---
    cgroup_namespaces(7)](https://man7.org/linux/man-pages/man7/cgroup_namespaces.7.html)
-   [Docker Engine security](https://docs.docker.com/engine/security/)
-   [Docker --- Run
    containers](https://docs.docker.com/engine/containers/run/)

## Day 07 completion tracker

-   [ ] Understand why containers need isolation.
-   [ ] Explain what a Linux namespace is.
-   [ ] Understand PID namespaces.
-   [ ] Understand mount namespaces.
-   [ ] Understand network namespaces.
-   [ ] Understand UTS and IPC namespaces.
-   [ ] Understand user and cgroup namespaces.
-   [ ] Explain namespaces vs cgroups.
-   [ ] Complete Experiment 1.
-   [ ] Complete Experiment 2.
-   [ ] Complete Experiment 3.
-   [ ] Complete Experiment 4.
-   [ ] Record actual outputs and observations.
-   [ ] Update this README with verified findings.

------------------------------------------------------------------------

**GitHub destination:** `days/day-07/README.md`

**First task:** Run only **Experiment 1** in your Ubuntu WSL terminal
and share the output of `hostname` and the two container hostname
commands. We will interpret the results together before moving on.
