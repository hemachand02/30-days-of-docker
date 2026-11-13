# Day 06 --- Linux Processes and Container Processes

**30 Days of Docker · Day 06 · `days/day-06/README.md`**

Day 05 explored the container lifecycle: create, start, pause, stop,
restart, and remove. To understand *why* a container starts or stops, we
now need to understand the Linux processes running inside it.

> A container is not a virtual machine with its own kernel. It is an
> isolated group of processes running on the host's Linux kernel, with
> isolation and resource controls provided by Linux features.

## 1. Learning objectives

By the end of this lesson, you should be able to:

-   Explain what a Linux process is and how a process is identified.
-   Distinguish a program from a running process.
-   Understand PID, PPID, PID 1, and a process tree.
-   Explain how a container's main process relates to its lifecycle.
-   Inspect processes inside a container using `ps`, `top`, and
    `docker top`.
-   Run an additional process inside a running container with
    `docker exec`.
-   Distinguish `docker exec` from `docker attach`.
-   Understand, at a high level, how PID namespaces isolate process IDs.
-   Observe what happens when a container's main process exits.

## 2. What is a process?

A **program** is a set of instructions stored on disk. A **process** is
a running instance of a program.

For example:

-   `/usr/bin/sleep` is an executable program.
-   Running `sleep 100` creates a process that executes that program.
-   Running the same command twice normally creates two separate
    processes.

When Linux starts a process, the kernel maintains information about it,
including its process ID, state, memory mappings, open file descriptors,
credentials, and scheduling information.

### Important process terminology

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  PID                                 Process ID: an identifier assigned
                                      to a process

  PPID                                Parent Process ID: the PID of the
                                      process that created it

  PID 1                               The process assigned ID 1 within a
                                      PID namespace

  Process tree                        The parent-child relationships
                                      between processes

  Foreground process                  A process associated with the
                                      terminal's foreground job

  Background process                  A process running without occupying
                                      the terminal's foreground job

  Signal                              A notification sent to a process,
                                      often to request an action such as
                                      termination
  -----------------------------------------------------------------------

PIDs are identifiers within a PID namespace. The same process can have
different visible PIDs from different PID namespaces.

## 3. Parent and child processes

Processes can create other processes. The creator is the **parent
process**, and the newly created process is the **child process**.

For example, when a shell runs:

``` bash
sleep 100
```

the shell typically creates a child process to execute `sleep`.

Conceptually:

``` text
Shell
  |
  +---- sleep 100
```

You can inspect process relationships with:

``` bash
ps -ef
```

Useful columns include:

-   `UID`: user that owns the process.
-   `PID`: process ID.
-   `PPID`: parent process ID.
-   `CMD`: command associated with the process.

To display a process tree:

``` bash
ps -ef --forest
```

The exact output and available formatting can vary between Linux
distributions and `ps` implementations.

## 4. Why PID 1 matters in a container

When Docker starts a container, it starts the container's configured
main process. Within the container's PID namespace, that process
normally appears as **PID 1**.

For example:

``` bash
docker run --name process-demo alpine:latest sleep 300
```

Inside this container, `sleep` is the main process and normally has PID
1.

Check it from another terminal:

``` bash
docker exec process-demo ps
```

The process list should show the `sleep` process. Its PID should be `1`
inside the container's PID namespace.

### Why is PID 1 special?

On Linux, PID 1 has special responsibilities:

1.  It is the initial process in its PID namespace.
2.  It can become responsible for adopting orphaned child processes.
3.  It should reap terminated child processes (collect their exit
    status) so they do not remain as zombies.
4.  Signal handling has special semantics for PID 1. In particular,
    default signal actions that would normally terminate a process may
    not work as expected for PID 1 unless the process installs
    appropriate handlers.

This matters when you run applications inside containers. A process that
works correctly as an ordinary Linux process may need correct signal
handling and child-process management when it runs as PID 1.

### What is a zombie process?

A **zombie** is a child process that has terminated, but whose parent
has not yet collected its exit status using a wait operation.

A zombie is no longer executing, but a small process-table entry remains
until it is reaped.

A well-behaved parent process should collect the exit status of its
terminated children. A container init process can help manage child
processes when the application does not do so itself.

## 5. The container's main process controls its lifecycle

This is one of the most important connections between Linux processes
and Docker.

A container's configured main process is often called its **PID 1
process** from inside the container.

Consider:

``` bash
docker run --name short-task alpine:latest echo "Task completed"
```

What happens?

1.  Docker creates the container.
2.  Docker starts the configured command.
3.  `echo` runs as the container's main process.
4.  It prints the message and exits.
5.  The main process has terminated.
6.  The container enters the `exited` state.

The container does not remain running just because its filesystem still
exists.

Now compare:

``` bash
docker run -d --name long-task alpine:latest sleep 300
```

Here, `sleep` remains active for 300 seconds, so the container remains
running during that time unless it is stopped or otherwise terminated.

**Remember:** Docker does not keep a container alive merely because you
want to inspect its filesystem. The main process must still be running
for the container to remain in the running state.

## 6. How Linux process isolation relates to containers

Containers share the host's Linux kernel, but Linux provides mechanisms
that isolate what processes can see and do.

One important mechanism is the **PID namespace**.

A PID namespace provides processes with a separate view of process IDs
and process relationships.

Conceptually:

``` text
Host PID namespace
|
+-- Host process: Docker-related processes
|
+-- Container A
|     PID 1  -> application
|     PID 2  -> worker
|
+-- Container B
      PID 1  -> another application
      PID 2  -> helper process
```

The diagram is conceptual. Actual host PIDs and container PIDs differ,
and the Docker daemon or container runtime may be hosted in a separate
environment when using Docker Desktop with WSL.

Important points:

-   Different containers can each have a process identified as PID 1 in
    their own PID namespaces.
-   A process can have a different PID when viewed from the host and
    from inside its container.
-   PID namespaces isolate process visibility and PID numbering; they do
    not provide a separate kernel.
-   Other Linux mechanisms, including mount, network, IPC, UTS, and user
    namespaces, isolate different system resources.

We will study namespaces and cgroups in greater depth in their dedicated
lessons. For now, remember that PID namespaces explain why a container
can have its own process tree and PID 1 while sharing the host kernel.

## 7. Inspecting processes: Linux and Docker commands

### 7.1 `ps` --- display processes

Inside a Linux environment:

``` bash
ps
ps -ef
ps aux
```

These display process information in different formats.

Inside a container:

``` bash
docker exec process-demo ps
```

This runs `ps` as an additional process inside the existing container.

### 7.2 `top` --- monitor processes

``` bash
top
```

`top` provides a continuously updating view of processes and resource
usage.

If the container image includes `top`, you can run:

``` bash
docker exec -it process-demo top
```

Minimal images such as Alpine may not include every diagnostic utility
by default.

### 7.3 `docker top` --- inspect container processes from Docker

``` bash
docker top process-demo
```

This displays processes associated with the container from the Docker
host's perspective.

The displayed PIDs can differ from the PIDs visible inside the container
because the host and container may see different PID namespaces.

### 7.4 `docker exec` --- run another process in a running container

``` bash
docker exec -it process-demo sh
```

This starts a new `sh` process inside the already-running container.

Breakdown:

-   `docker exec`: execute a new command in an existing running
    container.
-   `-i`: keep standard input open.
-   `-t`: allocate a pseudo-terminal.
-   `sh`: command to execute.

Exit the shell with:

``` bash
exit
```

This exits the shell started by `docker exec`. It does **not** normally
stop the container's original main process.

### 7.5 `docker attach` --- attach to the main process streams

``` bash
docker attach process-demo
```

`docker attach` connects your terminal to the existing container's main
process input/output streams. It does not start a new shell or a new
application process.

Be careful: sending an interrupt from an attached terminal can affect
the main process, depending on the application and terminal behavior.

To detach without stopping the container, use the default key sequence:

``` text
Ctrl-p, then Ctrl-q
```

This is different from `docker exec`, which starts a separate process.

### `docker exec` vs `docker attach`

  -----------------------------------------------------------------------
  Command                 What it does            Typical use
  ----------------------- ----------------------- -----------------------
  `docker exec`           Starts a new process    Open a shell, run
                          inside a running        diagnostics, execute a
                          container               command

  `docker attach`         Connects to the         Interact with the main
                          existing main process   process
                          streams                 
  -----------------------------------------------------------------------

## 8. Hands-on experiments

> **Status: Pending.** These are tasks to run and document. No
> experiment is claimed to have been performed. The expected
> observations are predictions; compare them with your actual terminal
> output.

**Environment:** Ubuntu on WSL, with Docker Desktop running and the
Docker CLI connected to the intended Docker Engine.

### Experiment 1 --- Observe PID 1 inside a container

**Objective:** Identify the container's main process.

Start a long-running container:

``` bash
docker run -d --name day06-pid alpine:latest sleep 300
```

Inspect the process from inside the container:

``` bash
docker exec day06-pid ps
```

Inspect it from Docker's host-side view:

``` bash
docker top day06-pid
```

Inspect the container's configured command:

``` bash
docker inspect --format 'Path={{.Path}} Args={{json .Args}}' day06-pid
```

Check its state:

``` bash
docker inspect --format 'Status={{.State.Status}}' day06-pid
```

Clean up:

``` bash
docker stop day06-pid
docker rm day06-pid
```

**Expected observations:**

-   The container should be running while `sleep 300` is active.
-   Inside the container, `sleep` should normally appear as PID 1.
-   `docker top` should show the container's process from the Docker
    host-side view. Its PID may differ from the container PID.

### Experiment 2 --- `docker exec` creates another process

**Objective:** Understand that `docker exec` does not replace the main
process.

Start a container:

``` bash
docker run -d --name day06-exec alpine:latest sleep 300
```

Inspect its processes:

``` bash
docker exec day06-exec ps
```

Run another command:

``` bash
docker exec day06-exec echo "This is a separate process"
```

Open an interactive shell:

``` bash
docker exec -it day06-exec sh
```

Inside the container, run:

``` sh
ps
echo "I am inside the container"
exit
```

Check the container state from Ubuntu:

``` bash
docker inspect --format '{{.State.Status}}' day06-exec
```

Clean up:

``` bash
docker stop day06-exec
docker rm day06-exec
```

**Expected observations:**

-   `sleep` remains the container's main process.
-   `echo` and the interactive shell are additional processes started by
    `docker exec`.
-   Exiting the interactive shell should not stop the original `sleep`
    process.
-   The container should remain running until its main process exits or
    the container is stopped.

### Experiment 3 --- Observe what happens when the main process exits

**Objective:** Connect process termination to the container lifecycle.

Run:

``` bash
docker run --name day06-exit alpine:latest sh -c 'echo "Main process is running"; sleep 3; echo "Main process is exiting"'
```

Check all containers:

``` bash
docker ps -a --filter name=day06-exit
```

Inspect the final state and exit code:

``` bash
docker inspect --format 'Status={{.State.Status}} ExitCode={{.State.ExitCode}}' day06-exit
```

Read its logs:

``` bash
docker logs day06-exit
```

Clean up:

``` bash
docker rm day06-exit
```

**Expected observations:**

-   The command runs in the foreground and prints both messages.
-   After the main process exits, the container should be in the
    `exited` state.
-   The successful shell command should normally produce exit code `0`.
-   The container remains listed by `docker ps -a` until it is removed.

### Experiment 4 --- Compare `docker exec` and `docker attach`

**Objective:** Understand the difference between starting a process and
attaching to an existing process.

Start a container with a long-running foreground process:

``` bash
docker run -d --name day06-attach alpine:latest sh -c 'while true; do echo "main process heartbeat"; sleep 3; done'
```

Read its logs:

``` bash
docker logs --tail 5 day06-attach
```

Start a separate shell:

``` bash
docker exec -it day06-attach sh
```

Inside the shell, run:

``` sh
ps
exit
```

Now attach to the main process streams:

``` bash
docker attach day06-attach
```

Detach using `Ctrl-p`, then `Ctrl-q`.

Check that the container is still running:

``` bash
docker ps --filter name=day06-attach
```

Clean up:

``` bash
docker stop day06-attach
docker rm day06-attach
```

**Expected observations:**

-   `docker exec` creates a separate process, such as a shell.
-   `docker attach` connects to the existing main process streams.
-   Detaching with the documented key sequence should leave the
    container running.
-   Stopping the container terminates its main process.

## 9. Troubleshooting checklist

  -----------------------------------------------------------------------
  Problem                             What to investigate
  ----------------------------------- -----------------------------------
  `docker exec` says the container is Check `docker ps -a` and the
  not running                         container's state

  `ps` command is not found inside a  The image may not include the
  minimal image                       process-inspection utility

  `docker top` shows different PIDs   Compare host and container PID
                                      namespaces

  Container exits immediately         Inspect its configured main command
                                      and process exit code

  Exiting an exec shell stops the     Check whether the shell was
  container unexpectedly              actually the main process or an
                                      exec process

  `docker attach` appears to hang     It is attached to the main process
                                      streams; inspect the application's
                                      behavior

  Container does not respond to       Investigate signal handling and the
  `docker stop` promptly              configured stop timeout
  -----------------------------------------------------------------------

Useful commands:

``` bash
docker ps -a
docker top <container>
docker inspect <container>
docker logs <container>
docker exec -it <container> sh
docker stats
```

Replace `<container>` with the actual container name or ID.

## 10. Key takeaways

1.  A program is stored instructions; a process is a running instance of
    a program.
2.  Every process has a PID and normally has a parent process identified
    by PPID.
3.  The container's main process normally appears as PID 1 inside its
    PID namespace.
4.  The container's lifecycle is closely tied to its main process.
5.  PID namespaces provide isolated views of process IDs and process
    trees.
6.  Containers share the host's Linux kernel; they do not each run a
    separate kernel.
7.  `docker exec` starts an additional process inside an existing
    running container.
8.  `docker attach` connects to the existing main process streams.
9.  PID 1 has special signal-handling and child-reaping
    responsibilities.
10. When the main process exits, the container normally enters the
    `exited` state.

## 11. Official documentation and references

-   [Docker --- Running
    containers](https://docs.docker.com/engine/containers/run/)
-   [Docker CLI ---
    `docker exec`](https://docs.docker.com/reference/cli/docker/container/exec/)
-   [Docker CLI ---
    `docker attach`](https://docs.docker.com/reference/cli/docker/container/attach/)
-   [Docker CLI ---
    `docker top`](https://docs.docker.com/reference/cli/docker/container/top/)
-   [Docker CLI ---
    `docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/)
-   [Linux man page ---
    `ps`](https://man7.org/linux/man-pages/man1/ps.1.html)
-   [Linux man page ---
    `pid_namespaces`](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)
-   [Linux man page ---
    `signal`](https://man7.org/linux/man-pages/man7/signal.7.html)
-   [Docker Engine --- Container
    runtime](https://docs.docker.com/engine/)

## Day 06 completion tracker

-   [ ] Understand programs, processes, PIDs, and PPIDs.
-   [ ] Understand PID 1 and why it matters in containers.
-   [ ] Understand the relationship between the main process and
    container lifecycle.
-   [ ] Understand PID namespaces at a conceptual level.
-   [ ] Complete Experiment 1.
-   [ ] Complete Experiment 2.
-   [ ] Complete Experiment 3.
-   [ ] Complete Experiment 4.
-   [ ] Record actual outputs and observations.
-   [ ] Update this README with verified findings.

------------------------------------------------------------------------

**GitHub destination:** `days/day-06/README.md`

**First task:** Run only **Experiment 1** in your Ubuntu WSL terminal
and share the outputs of `docker exec ... ps`, `docker top`, and
`docker inspect`. We will interpret the process IDs together before
moving to Experiment 2.
