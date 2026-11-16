# Day 08 --- Linux cgroups: Resource Management

## Overview

**Focus:** Linux control groups (cgroups), the kernel mechanism used to
account for and control resource usage by groups of processes.

Day 07 covered namespaces: isolating what processes can see. Day 08
covers cgroups: managing how much of a resource a group can consume.

## Learning objectives

-   Explain why namespaces alone do not limit resource consumption.
-   Define cgroups and controllers.
-   Distinguish cgroups v1 from v2.
-   Describe CPU, memory, PIDs, and I/O controls.
-   Inspect cgroup information on your Linux environment.
-   Explain how Docker uses cgroups.

## 1. Why cgroups?

Containers share the host's underlying resources. One workload could
consume excessive CPU, memory, or process slots and affect other
workloads. Cgroups let the kernel organize processes into hierarchical
groups, account for their resource usage, and apply controls.

## 2. What is a cgroup?

A **control group (cgroup)** is a Linux kernel mechanism for organizing
processes into hierarchical groups and applying resource accounting and
control to those groups. Processes normally pass cgroup membership to
child processes unless a manager moves them.

Cgroup interfaces are commonly exposed through the virtual filesystem:

``` bash
/sys/fs/cgroup
```

## 3. Resource controllers

A **controller** handles accounting or control for a particular
resource.

  Controller/resource   Purpose
  --------------------- --------------------------------------------------------------
  CPU                   Accounts for CPU use and can control CPU bandwidth
  Memory                Tracks memory use and can enforce memory limits
  PIDs                  Limits the number of processes/tasks
  I/O                   Accounts for and can limit block-device I/O, where supported

**CPU:** A quota can constrain CPU time over a scheduling period. It
does not necessarily reserve a dedicated core.

**Memory:** A limit constrains memory charged to the cgroup. When memory
pressure exceeds what can be satisfied, the kernel may reclaim memory or
invoke OOM handling, potentially killing processes.

**PIDs:** The PIDs controller limits the number of processes/tasks in a
cgroup hierarchy. This helps mitigate runaway process creation.

**I/O:** I/O controls depend on the kernel, storage device, and
controller support.

## 4. cgroups v1 and v2

-   **cgroups v1:** Uses multiple hierarchies; controllers may be
    mounted separately.
-   **cgroups v2:** Uses a unified hierarchy and a more consistent
    interface.

On cgroups v2, files may include `cpu.max`, `cpu.stat`,
`memory.current`, `memory.max`, `pids.current`, and `pids.max`.
Availability depends on enabled controllers and the environment.

## 5. Namespaces vs cgroups

  -----------------------------------------------------------------------
  Aspect                  Namespaces              Cgroups
  ----------------------- ----------------------- -----------------------
  Main purpose            Isolate views of system Account for and control
                          resources               resource use

  Question                What can this process   How much can this group
                          see?                    consume?

  Examples                PID, mount, network,    CPU, memory, PIDs, I/O
                          UTS, IPC, user          
  -----------------------------------------------------------------------

Containers commonly use both. Namespaces isolate views; cgroups manage
resource consumption. Additional mechanisms such as capabilities,
seccomp, and AppArmor/SELinux contribute to security. Neither namespaces
nor cgroups alone form a complete security boundary.

## 6. How Docker uses cgroups

At a high level: 1. Docker receives container configuration. 2. The
runtime creates the container process environment. 3. The runtime places
container processes into the appropriate cgroup. 4. The Linux kernel
accounts for and enforces configured resource controls.

Docker exposes options such as `--memory`, `--cpus`, and `--pids-limit`.

**Docker Desktop note:** Linux containers run inside Docker Desktop's
Linux environment. WSL-visible cgroup information may describe the WSL
environment rather than the container's Linux host. Exact visibility
depends on your Docker context and setup.

## 7. Hands-on inspection

Run these commands in Ubuntu/WSL.

### Experiment 1 --- Identify the visible cgroup version

``` bash
stat -fc %T /sys/fs/cgroup
cat /proc/self/cgroup
```

A filesystem type of `cgroup2fs` generally indicates a cgroups v2 mount.

### Experiment 2 --- Explore the cgroup filesystem

``` bash
ls -la /sys/fs/cgroup
```

On v2, you may see files such as `cgroup.controllers`,
`cgroup.subtree_control`, `cpu.max`, `memory.current`, and `pids.max`.
Do not assume every file exists.

### Experiment 3 --- Read available resource information

``` bash
cat /sys/fs/cgroup/cgroup.controllers
cat /sys/fs/cgroup/memory.current
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/cpu.stat
```

If a file is absent, record that fact and inspect the directory. Do not
create missing kernel interface files.

### Experiment 4 --- Inspect a Docker container's cgroup membership

``` bash
docker run -d --name cgroup-lab alpine sleep 300
docker inspect -f '{{.State.Pid}}' cgroup-lab
```

If the returned PID is visible in the Linux environment hosting the
container, inspect its cgroup:

``` bash
cat /proc/<PID>/cgroup
```

Replace `<PID>` with the actual value. With Docker Desktop, the
container PID may belong to its Linux VM and not be directly visible
from your WSL distribution.

Clean up:

``` bash
docker rm -f cgroup-lab
```

## 8. Key terminology

-   **cgroup:** Hierarchical group of processes for resource accounting
    and control.
-   **Controller:** Component that handles a resource such as CPU or
    memory.
-   **Hierarchy:** Parent-child organization of cgroups.
-   **Quota:** Configured allowance, such as CPU time over a period.
-   **OOM:** Out-of-memory condition and the kernel's associated
    handling.
-   **cgroups v2:** Unified cgroup hierarchy and interface.

## 9. Common misconceptions

-   Namespaces do not, by themselves, impose memory limits.
-   A CPU quota does not mean a dedicated CPU core is assigned.
-   Cgroups are resource-management mechanisms, not a complete security
    boundary.
-   The visible cgroup files vary with kernel, version, controllers,
    permissions, and runtime environment.

## 10. Completion checklist

-   [ ] I can explain why namespaces alone do not control resource
    consumption.
-   [ ] I can define cgroups and controllers.
-   [ ] I can distinguish namespaces from cgroups.
-   [ ] I can describe CPU, memory, PIDs, and I/O controls.
-   [ ] I identified the cgroup version visible in my environment.
-   [ ] I inspected `/proc/self/cgroup` and `/sys/fs/cgroup`.

## References

-   [Linux kernel documentation --- cgroup
    v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
-   [Linux kernel documentation --- cgroup
    v1](https://docs.kernel.org/admin-guide/cgroup-v1/)
-   [Docker documentation --- Resource
    constraints](https://docs.docker.com/engine/containers/resource_constraints/)
-   [Docker documentation --- Runtime
    metrics](https://docs.docker.com/engine/containers/runmetrics/)

## Day 08 practical task

Start with **Experiment 1 only**:

``` bash
stat -fc %T /sys/fs/cgroup
cat /proc/self/cgroup
```

Share the exact output. We will interpret your environment before moving
to Experiment 2.
