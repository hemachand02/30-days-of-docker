# Day 05 --- Docker Container Creation, Execution and Lifecycle

**30 Days of Docker · Day 05 · `days/day-05/README.md`**

Today we move from understanding Docker images to understanding what
happens when an image becomes a running container.

The central idea:

> An image is a template. A container is an instance created from that
> template. Its lifecycle is primarily controlled by the lifecycle of
> its main process.

## 1. Learning objectives

By the end of this lesson, you should be able to:

-   Explain how Docker creates a container from an image.
-   Distinguish `docker create`, `docker start`, and `docker run`.
-   Identify the major container states and understand their
    transitions.
-   Explain the role of the container's main process.
-   Start, stop, pause, unpause, restart, and remove containers.
-   Understand graceful shutdown, `SIGTERM`, and `SIGKILL`.
-   Run containers in foreground and detached modes.
-   Inspect container state, logs, and exit codes.
-   Troubleshoot common container lifecycle problems.

## 2. Image vs container: what actually gets created?

A container is built from an image, but it is not simply the image
itself. Four components are important:

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Docker image                        Read-only template containing the
                                      application filesystem and required
                                      user-space dependencies

  Container configuration             Runtime settings such as command,
                                      environment variables, networking,
                                      and resource limits

  Writable container layer            Stores filesystem changes made
                                      inside the container outside
                                      mounted volumes

  Main process                        The primary process started when
                                      the container runs
  -----------------------------------------------------------------------

Conceptually:

``` text
Docker Image (read-only)
        |
        + Container Configuration
        |
        + Writable Container Layer
        |
        v
     Container
        |
        v
 Main Process (PID 1)
```

When Docker creates a container, it prepares a container-specific
configuration and writable layer on top of the image. When the container
is started, Docker launches its configured main process.

**Important:** The writable layer is not the same as a Docker volume.
Removing a container removes its writable layer, but data stored in
separately managed volumes can persist.

## 3. Container lifecycle and states

A container moves through different states as Docker and its main
process perform operations.

``` text
                    docker create
   [Image] ----------------------------> [Created]
                                            |
                                            | docker start
                                            v
                                         [Running]
                                          /  |  \
                                         /   |   \
                          docker pause /    |    \ Main process exits
                                       v     |     v
                                   [Paused]  |  [Exited]
                                       |     |      |
                          docker       |     |      | docker start
                          unpause      |     |      +------> [Running]
                                       +-----+
                                             |
                                  docker stop / kill
                                             v
                                          [Exited]
                                             |
                                          docker rm
                                             v
                                          [Removed]
```

This is a simplified conceptual diagram, not every possible transition.
Restart policies, errors, and removal operations introduce additional
states.

### Main container states

  -----------------------------------------------------------------------
  State                               Meaning
  ----------------------------------- -----------------------------------
  `created`                           Container exists, but its main
                                      process has not started

  `running`                           Main process is executing

  `paused`                            Container processes are temporarily
                                      frozen

  `restarting`                        Docker is restarting the container,
                                      commonly due to a restart policy

  `exited`                            Main process has terminated; the
                                      container still exists

  `removing`                          Docker is in the process of
                                      removing the container

  `dead`                              An unusual failure state in which
                                      Docker could not complete normal
                                      container management
  -----------------------------------------------------------------------

### Important distinctions

**Created vs running**

-   `created` means Docker has prepared the container but has not
    started its main process.
-   `running` means the main process has started and the container is
    executing.

**Paused vs stopped**

-   `paused` means the container's processes are frozen.
-   `exited` means the main process has terminated.
-   A paused container still occupies resources; pausing is not a way to
    shut it down.

**Exited vs removed**

-   `exited` means the container has stopped but its metadata and
    writable layer still exist.
-   Removed means the container object has been deleted.
-   Once removed, you cannot inspect that container using its old name
    or ID.

**Restarting vs running**

-   `restarting` is generally a transient state during an automatic
    restart-policy cycle.
-   A successful manual `docker restart` normally finishes with the
    container running.

## 4. Commands that control the lifecycle

### 4.1 `docker create` --- create without starting

``` bash
docker create --name my-container alpine:latest
```

What happens:

1.  Docker locates the image, pulling it if necessary.
2.  Docker prepares the container configuration.
3.  Docker creates the container's writable layer.
4.  Docker assigns a container ID.
5.  The container remains in the `created` state.

It does not start the container's main process.

### 4.2 `docker start` --- start an existing container

``` bash
docker start my-container
```

Docker starts the configured main process of an existing created or
stopped container.

To attach to its output:

``` bash
docker start -a my-container
```

### 4.3 `docker run` --- create and start

``` bash
docker run alpine:latest
```

`docker run` combines container creation and startup. It creates a new
container and starts it immediately.

For example:

``` bash
docker run alpine:latest echo "Hello Docker"
```

The command prints its message and exits. Since the main process has
completed, the container enters the `exited` state.

To list stopped containers too:

``` bash
docker ps -a
```

**Important:** Running `docker run` again normally creates another
container. It does not restart the previous container.

### 4.4 `docker pause` --- temporarily freeze processes

``` bash
docker pause my-container
```

Docker pauses the processes inside the container using the Linux cgroups
freezer functionality.

-   Container processes stop executing.
-   The container is not terminated.
-   Its existing state and writable layer remain.
-   The container can be resumed.

### 4.5 `docker unpause` --- resume execution

``` bash
docker unpause my-container
```

This resumes the paused processes. The container returns to the running
state.

### 4.6 `docker stop` --- gracefully stop a container

``` bash
docker stop my-container
```

Docker generally follows this sequence:

1.  Sends `SIGTERM` to the container's main process.
2.  Waits for the configured grace period.
3.  If the process has not exited, sends `SIGKILL`.
4.  The container enters the `exited` state.

Customize the waiting period:

``` bash
docker stop --time 5 my-container
```

This gives the main process five seconds to shut down before Docker
forcefully terminates it.

### 4.7 `docker kill` --- send a signal immediately

``` bash
docker kill my-container
```

By default, Docker sends `SIGKILL`. Unlike `docker stop`, the default
`docker kill` operation does not provide a graceful shutdown period.

You can also choose a signal:

``` bash
docker kill --signal SIGTERM my-container
```

### 4.8 `docker restart` --- stop and start again

``` bash
docker restart my-container
```

Docker stops the existing container and starts it again. The existing
container is reused; Docker does not create a completely new container,
and its writable layer is retained.

Customize the stop timeout:

``` bash
docker restart --time 5 my-container
```

### 4.9 `docker rm` --- remove a container

``` bash
docker rm my-container
```

This removes a stopped container and its writable layer. It does not
normally remove the image from which the container was created.

To force-remove a running container:

``` bash
docker rm -f my-container
```

This forcefully terminates the running container before removing it. It
is not the preferred graceful shutdown method.

To automatically remove a container when it exits:

``` bash
docker run --rm alpine:latest echo "Temporary container"
```

### 4.10 `docker inspect` --- examine container details

``` bash
docker inspect my-container
```

This returns detailed JSON information, including:

-   Container ID and name.
-   Image reference.
-   Runtime configuration.
-   Network settings and mounts.
-   State and exit code.
-   Creation and execution timestamps.

Retrieve specific information using Go template formatting:

``` bash
docker inspect --format '{{.State.Status}}' my-container
docker inspect --format '{{.State.Running}}' my-container
docker inspect --format '{{.State.ExitCode}}' my-container
docker inspect --format '{{.State.StartedAt}}' my-container
docker inspect --format '{{.State.FinishedAt}}' my-container
```

These commands help you inspect the actual state instead of guessing
from terminal output.

## 5. Graceful shutdown: SIGTERM vs SIGKILL

A Linux signal is a notification delivered to a process to communicate
an event or request an action.

Two signals matter particularly during container shutdown:

  -----------------------------------------------------------------------
  Signal                  Meaning                 Can the process handle
                                                  it?
  ----------------------- ----------------------- -----------------------
  `SIGTERM`               Request termination     Yes. A process can
                                                  handle it or ignore it.

  `SIGKILL`               Force termination       No. It cannot be
                                                  caught, blocked, or
                                                  ignored.
  -----------------------------------------------------------------------

### What happens during `docker stop`?

``` text
docker stop
     |
     v
Docker sends SIGTERM
     |
     v
Application gets time to shut down
     |
     +---- Process exits ----> Container stopped
     |
     +---- Timeout expires --> Docker sends SIGKILL
                                  |
                                  v
                              Container stopped
```

Why does graceful shutdown matter?

A server application may need to:

-   Stop accepting new requests.
-   Complete or safely terminate ongoing work.
-   Close database connections.
-   Flush important buffers.
-   Release resources.

If it is forcefully terminated, these cleanup steps may not complete.

**Technical detail:** A container's main process is normally PID 1
inside the container's process namespace. The application's process and
signal-handling behavior matter. For example, using the shell form of a
Dockerfile `ENTRYPOINT` can introduce a shell process that affects
signal delivery. The exec form is generally preferable for applications
that should receive container stop signals directly.

## 6. Foreground mode vs detached mode

### Foreground mode

``` bash
docker run --rm alpine:latest sh -c 'echo "Started"; sleep 5; echo "Finished"'
```

The terminal remains attached while the command executes.

-   Output is visible in the terminal.
-   The command normally returns after the main process exits.
-   Useful for debugging and short-lived commands.

### Detached mode

``` bash
docker run -d --name web-test nginx:latest
```

The `-d` option means detached mode.

-   Docker starts the container in the background.
-   The terminal returns without waiting for the container to finish.
-   Useful for web servers, databases, and other long-running services.

Check running containers:

``` bash
docker ps
```

View output:

``` bash
docker logs web-test
```

Stop it:

``` bash
docker stop web-test
```

Detached does not mean the container is guaranteed to remain running. If
its main process exits, the container stops.

## 7. Exit codes and container logs

When a container's main process terminates, Docker records its exit
code.

Inspect it:

``` bash
docker inspect --format '{{.State.ExitCode}}' my-container
```

Common interpretations:

  Exit code   Common interpretation
  ----------- ---------------------------------------------------
  `0`         Process completed successfully
  Non-zero    Application or command reported an error
  `137`       Commonly associated with termination by `SIGKILL`
  `143`       Commonly associated with termination by `SIGTERM`

The values `137` and `143` commonly follow the shell convention of
`128 + signal number`. They are useful clues, not definitive proof of
why a process exited: applications can also explicitly return exit
codes.

### View container logs

``` bash
docker logs my-container
```

Follow logs continuously:

``` bash
docker logs -f my-container
```

Show only the last few lines:

``` bash
docker logs --tail 20 my-container
```

Show logs with timestamps:

``` bash
docker logs -t my-container
```

Docker logs normally display output written by the application to
standard output (`stdout`) and standard error (`stderr`). They do not
automatically display every log file created inside the container.

## 8. Hands-on experiments

> **Status: Pending.** All experiments below are practical tasks, not
> claimed results. Run them and record your actual outputs and
> observations. The expected results are predictions to compare against
> your observations.

**Environment:** Ubuntu on WSL, with Docker Desktop running and the
Docker CLI connected to the intended Docker Engine.

### Experiment 1 --- Create, start, stop, inspect and remove

**Objective:** Observe the basic lifecycle of one container.

Download the lightweight Alpine image:

``` bash
docker pull alpine:latest
```

Create a container without starting it:

``` bash
docker create --name day05-lifecycle alpine:latest sh -c 'echo "Container main process started"; sleep 300'
```

Check all containers:

``` bash
docker ps -a --filter name=day05-lifecycle
```

Inspect its state:

``` bash
docker inspect --format 'Status={{.State.Status}} Running={{.State.Running}} ExitCode={{.State.ExitCode}}' day05-lifecycle
```

Start it:

``` bash
docker start day05-lifecycle
```

Check running containers:

``` bash
docker ps --filter name=day05-lifecycle
```

View its output:

``` bash
docker logs day05-lifecycle
```

Stop it:

``` bash
docker stop --time 5 day05-lifecycle
```

Inspect the final state:

``` bash
docker inspect --format 'Status={{.State.Status}} ExitCode={{.State.ExitCode}} FinishedAt={{.State.FinishedAt}}' day05-lifecycle
```

Remove it:

``` bash
docker rm day05-lifecycle
```

**Expected observations:**

1.  After `docker create`, the container should be in `created` state.
2.  After `docker start`, it should appear in `docker ps`.
3.  The logs should contain the message printed by the container's
    command.
4.  After `docker stop`, it should be in `exited` state.
5.  After `docker rm`, it should no longer appear in `docker ps -a`.

The exact container ID, timestamps, and exit code depend on actual
execution.

### Experiment 2 --- Foreground vs detached execution

**Objective:** Understand how the terminal behaves in both modes.

Run in foreground:

``` bash
docker run --rm alpine:latest sh -c 'echo "Started"; sleep 3; echo "Finished"'
```

Run in detached mode:

``` bash
docker run -d --name day05-detached alpine:latest sh -c 'echo "Background process started"; sleep 30'
```

Check the container:

``` bash
docker ps
```

Read its logs:

``` bash
docker logs day05-detached
```

Clean up:

``` bash
docker stop day05-detached
docker rm day05-detached
```

**Expected observations:**

-   Foreground execution keeps the terminal attached until the command
    completes.
-   Detached execution returns control to the terminal while the
    container continues running.
-   The detached container can be inspected and its logs viewed
    independently.

### Experiment 3 --- Pause and unpause

**Objective:** Understand the difference between pausing and stopping.

Start a container that prints a heartbeat:

``` bash
docker run -d --name day05-pause alpine:latest sh -c 'while true; do echo "heartbeat"; sleep 2; done'
```

Pause it:

``` bash
docker pause day05-pause
```

Inspect it:

``` bash
docker inspect --format '{{.State.Status}}' day05-pause
```

Wait briefly and inspect its logs:

``` bash
sleep 5
docker logs --tail 5 day05-pause
```

Resume it:

``` bash
docker unpause day05-pause
```

Clean up:

``` bash
docker stop day05-pause
docker rm day05-pause
```

**Expected observations:**

-   The state should change to `paused`.
-   The heartbeat process should stop making progress while paused.
-   After unpausing, the process should continue.

### Experiment 4 --- Compare stop, kill and restart

**Objective:** Understand the different ways to terminate and restart a
container.

Create a long-running container:

``` bash
docker run -d --name day05-stop alpine:latest sh -c 'echo "Ready"; sleep 300'
```

Stop it gracefully:

``` bash
docker stop --time 5 day05-stop
```

Inspect its exit information:

``` bash
docker inspect --format 'Status={{.State.Status}} ExitCode={{.State.ExitCode}}' day05-stop
```

Remove it:

``` bash
docker rm day05-stop
```

Now test forceful termination:

``` bash
docker run -d --name day05-kill alpine:latest sh -c 'sleep 300'
docker kill day05-kill
docker inspect --format 'Status={{.State.Status}} ExitCode={{.State.ExitCode}}' day05-kill
docker rm day05-kill
```

Finally, test restarting an existing container:

``` bash
docker run -d --name day05-restart alpine:latest sh -c 'while true; do sleep 2; done'
docker restart --time 5 day05-restart
docker inspect --format '{{.State.Status}}' day05-restart
docker stop day05-restart
docker rm day05-restart
```

**Expected observations:**

-   `docker stop` requests graceful termination before forcefully
    killing an unresponsive process.
-   `docker kill` sends `SIGKILL` by default.
-   `docker restart` stops and starts the same container.
-   A successfully restarted container should return to the running
    state.

## 9. Troubleshooting checklist

  -----------------------------------------------------------------------
  Problem                             What to investigate
  ----------------------------------- -----------------------------------
  Container exits immediately         Check its configured command and
                                      main process

  Container is missing from           Run `docker ps -a` to include
  `docker ps`                         stopped containers

  Cannot start a container            Inspect its state and read the
                                      Docker error

  Container appears stuck             Check logs, process behavior, and
                                      resource usage

  `docker rm` refuses to remove it    Stop it first, or intentionally use
                                      `docker rm -f`

  No output from `docker logs`        Check whether the application
                                      writes to stdout/stderr

  Unexpected exit code                Inspect application output,
                                      `.State.Error`, `.State.ExitCode`,
                                      and timestamps

  Docker commands cannot connect      Check Docker Desktop, WSL
                                      integration, and the active Docker
                                      context
  -----------------------------------------------------------------------

Useful diagnostic commands:

``` bash
docker ps -a
docker inspect my-container
docker logs my-container
docker stats
docker events
docker context show
docker info
```

Do not delete containers or images while troubleshooting until you have
captured the information you need.

## 10. Key takeaways

1.  An image is a template; a container is an instance created from it.
2.  `docker create` creates a container without starting it.
3.  `docker start` starts an existing container.
4.  `docker run` creates and starts a new container.
5.  A container's lifecycle is closely tied to its main process.
6.  `docker pause` freezes processes; `docker stop` terminates them.
7.  `docker stop` normally sends `SIGTERM` first and `SIGKILL` after the
    grace period if necessary.
8.  `docker kill` sends `SIGKILL` by default.
9.  `docker restart` reuses the existing container.
10. `docker rm` removes the container, not its source image.
11. `docker inspect` reveals detailed state and configuration
    information.

## 11. Official documentation

-   [Docker --- Run
    containers](https://docs.docker.com/engine/containers/run/)
-   [Docker CLI ---
    Create](https://docs.docker.com/reference/cli/docker/container/create/)
-   [Docker CLI ---
    Start](https://docs.docker.com/reference/cli/docker/container/start/)
-   [Docker CLI ---
    Run](https://docs.docker.com/reference/cli/docker/container/run/)
-   [Docker CLI ---
    Pause](https://docs.docker.com/reference/cli/docker/container/pause/)
-   [Docker CLI ---
    Unpause](https://docs.docker.com/reference/cli/docker/container/unpause/)
-   [Docker CLI ---
    Stop](https://docs.docker.com/reference/cli/docker/container/stop/)
-   [Docker CLI ---
    Kill](https://docs.docker.com/reference/cli/docker/container/kill/)
-   [Docker CLI ---
    Restart](https://docs.docker.com/reference/cli/docker/container/restart/)
-   [Docker CLI ---
    Remove](https://docs.docker.com/reference/cli/docker/container/rm/)
-   [Docker CLI ---
    Inspect](https://docs.docker.com/reference/cli/docker/inspect/)
-   [Docker CLI ---
    Logs](https://docs.docker.com/reference/cli/docker/container/logs/)

## Day 05 completion tracker

-   [ ] Understand image vs container.
-   [ ] Understand the container lifecycle states.
-   [ ] Complete Experiment 1.
-   [ ] Complete Experiment 2.
-   [ ] Complete Experiment 3.
-   [ ] Complete Experiment 4.
-   [ ] Record actual outputs and observations.
-   [ ] Update this README with verified findings.

------------------------------------------------------------------------

**GitHub destination:** `days/day-05/README.md`

**First task:** Run only **Experiment 1** in your Ubuntu WSL terminal
and share the command outputs. We will examine the actual state
transitions together before moving on to the next experiment.
