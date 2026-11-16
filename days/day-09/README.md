# Day 09 --- Docker Resource Limits: CPU, Memory, and PIDs

## Overview

**Focus:** Apply and inspect resource constraints on Docker containers.

Day 08 introduced Linux cgroups. Today, use Docker's command-line
options to configure CPU, memory, and process-count limits, then inspect
those settings and monitor usage.

## Learning objectives

-   Explain why container resource limits matter.
-   Set memory, CPU, and PIDs limits.
-   Inspect configured values with `docker inspect`.
-   Monitor usage with `docker stats`.
-   Distinguish a configured limit from current usage.
-   Recognize what can happen when a workload reaches a limit.

## 1. Why resource limits?

Containers share the host's underlying resources. A memory leak,
CPU-intensive workload, or runaway process-creation loop can affect
other workloads. Resource limits help make resource consumption more
predictable.

Limits do not replace monitoring, capacity planning, or host-level
protection.

## 2. Memory limits

Docker's `--memory` option configures a container memory limit.

``` bash
docker run -d --name memory-lab --memory=128m alpine sleep 300
```

Inspect the configured value:

``` bash
docker inspect -f '{{.HostConfig.Memory}}' memory-lab
```

Monitor current usage:

``` bash
docker stats --no-stream memory-lab
```

Understand the difference: - **Memory limit:** Configured maximum
enforced through resource controls. - **Memory usage:** Amount currently
being used. - **Memory reservation:** A softer reservation concept, not
the same as a hard maximum.

When memory pressure exceeds what can be satisfied, the kernel may
reclaim memory or invoke OOM handling. A process may be killed. The
exact outcome depends on kernel behavior and workload.

## 3. CPU limits

Docker's `--cpus` option provides a convenient CPU quota configuration.

``` bash
docker run -d --name cpu-lab --cpus=0.5 alpine sleep 300
```

Inspect the configured quota:

``` bash
docker inspect -f '{{.HostConfig.NanoCpus}}' cpu-lab
```

Monitor:

``` bash
docker stats --no-stream cpu-lab
```

A value of `0.5` limits the container to approximately half of one CPU's
processing capacity over time. It does not pin the container to half of
a physical core, nor does it force the container to consume CPU. An idle
container should show low CPU usage.

### CPU quota vs CPU shares

-   **CPU quota/period:** Can impose an upper bound on CPU time over a
    scheduling period.
-   **CPU shares/weight:** Influences relative CPU allocation when
    workloads compete. It is not necessarily a hard cap.

For this lab, use the simpler `--cpus` option.

## 4. Process-count limits

Docker's `--pids-limit` configures a limit on the number of
processes/tasks in the container's cgroup.

``` bash
docker run -d --name pids-lab --pids-limit=64 alpine sleep 300
```

Inspect:

``` bash
docker inspect -f '{{.HostConfig.PidsLimit}}' pids-lab
```

Monitor:

``` bash
docker stats --no-stream pids-lab
docker top pids-lab
```

This helps reduce the impact of runaway process creation. Do not run a
fork bomb or intentionally exhaust the limit.

## 5. Monitor usage with docker stats

``` bash
docker stats
docker stats --no-stream
docker stats --no-stream <container-name>
```

Useful columns: - **CPU %:** CPU usage reported by Docker. - **MEM USAGE
/ LIMIT:** Current memory usage and displayed limit. - **MEM %:** Memory
usage as a percentage of the displayed limit. - **PIDS:** Process/task
count reported for the container. - **BLOCK I/O:** Block-device data
read and written. - **NET I/O:** Network data received and transmitted.

Monitoring shows observed usage; it does not configure a limit or
provide a complete performance diagnosis.

## 6. Hands-on labs

Use lightweight Alpine containers. Run one experiment at a time and
clean up after each.

### Experiment 1 --- Create and inspect a memory-limited container

``` bash
docker run -d --name memory-lab --memory=128m alpine sleep 300
docker ps --filter name=memory-lab
docker inspect -f '{{.HostConfig.Memory}}' memory-lab
docker stats --no-stream memory-lab
```

Record the configured limit and the usage reported by `docker stats`.

Clean up:

``` bash
docker rm -f memory-lab
```

### Experiment 2 --- Create a CPU-limited container

``` bash
docker run -d --name cpu-lab --cpus=0.5 alpine sleep 300
docker inspect -f '{{.HostConfig.NanoCpus}}' cpu-lab
docker stats --no-stream cpu-lab
```

The container is idle, so CPU usage should generally be low. The limit
is a ceiling, not a target.

Clean up:

``` bash
docker rm -f cpu-lab
```

### Experiment 3 --- Create a container with a PIDs limit

``` bash
docker run -d --name pids-lab --pids-limit=64 alpine sleep 300
docker inspect -f '{{.HostConfig.PidsLimit}}' pids-lab
docker stats --no-stream pids-lab
docker top pids-lab
```

Do not try to create dozens of processes to reach the limit. The goal is
to inspect the configured value and current count safely.

Clean up:

``` bash
docker rm -f pids-lab
```

### Experiment 4 --- Compare default and explicitly limited configuration

``` bash
docker run -d --name default-lab alpine sleep 300
docker run -d --name limited-lab --memory=128m --cpus=0.5 --pids-limit=64 alpine sleep 300
```

Inspect both:

``` bash
docker inspect -f 'Name={{.Name}} Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}} PidsLimit={{.HostConfig.PidsLimit}}' default-lab limited-lab
```

Compare:

``` bash
docker stats --no-stream default-lab limited-lab
```

An omitted container-specific limit does not guarantee unlimited access
to the physical machine. Docker Desktop's Linux VM, WSL configuration,
and parent cgroups may impose broader constraints.

Clean up:

``` bash
docker rm -f default-lab limited-lab
```

## 7. Compose resource settings --- awareness

Docker Compose supports resource-related service configuration, but
support and enforcement can depend on Compose version and deployment
mode. First understand the equivalent `docker run` options before moving
to Compose-specific resource configuration.

Reference: [Compose services
reference](https://docs.docker.com/reference/compose-file/services/)

## 8. Troubleshooting

**`docker stats` shows low CPU usage:** Expected for an idle container.
A CPU limit is a ceiling, not a target.

**A container exits unexpectedly:** Check its state, exit code, and
logs:

``` bash
docker ps -a
docker inspect <container-name>
docker logs <container-name>
```

Do not assume every exit is caused by a resource limit.

**An inspected field is zero or looks unexpected:** Inspect the full
configuration:

``` bash
docker inspect <container-name>
```

A zero value can indicate that no container-specific limit was
configured for that field. Parent limits may still apply.

**WSL and Docker Desktop show different resource information:** Linux
containers run inside Docker Desktop's Linux environment. WSL-visible
resource information and container-host information may describe
different layers.

## 9. Key terminology

-   **Resource constraint:** Configured restriction on resource
    consumption.
-   **CPU quota:** Allowance of CPU time over a scheduling period.
-   **CPU shares/weight:** Relative weight used when CPU is contended.
-   **Memory limit:** Maximum memory configured for a container's
    cgroup.
-   **OOM handling:** Kernel behavior when memory cannot be satisfied.
-   **PIDs limit:** Maximum number of processes/tasks in a cgroup.
-   **Monitoring:** Observing usage; distinct from configuring limits.

## 10. Completion checklist

-   [ ] I understand why resource limits matter.
-   [ ] I created a memory-limited container.
-   [ ] I created a CPU-limited container.
-   [ ] I created a container with a PIDs limit.
-   [ ] I inspected limits using `docker inspect`.
-   [ ] I observed usage using `docker stats`.
-   [ ] I can explain CPU quota vs CPU shares.
-   [ ] I removed all lab containers.

## References

-   [Docker --- Resource
    constraints](https://docs.docker.com/engine/containers/resource_constraints/)
-   [Docker --- docker
    run](https://docs.docker.com/reference/cli/docker/container/run/)
-   [Docker --- docker
    stats](https://docs.docker.com/reference/cli/docker/container/stats/)
-   [Docker --- docker
    inspect](https://docs.docker.com/reference/cli/docker/inspect/)
-   [Linux kernel --- cgroup
    v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)

## Day 09 practical task

Start with **Experiment 1 only**:

``` bash
docker run -d --name memory-lab --memory=128m alpine sleep 300
docker inspect -f '{{.HostConfig.Memory}}' memory-lab
docker stats --no-stream memory-lab
```

Share the exact output. We will interpret the configured limit versus
actual usage before moving to the CPU lab.

After recording the result, clean up:

``` bash
docker rm -f memory-lab
```
