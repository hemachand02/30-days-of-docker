# Day 12 --- Docker Volumes and Persistent Data

## Overview

**Focus:** Understand container storage, writable layers, named volumes,
and bind mounts.

Containers are replaceable, but application data often must survive
container removal. Today we separate the lifetime of data from the
lifetime of a container.

## Learning objectives

-   Explain the container writable layer.
-   Distinguish writable layers, named volumes, and bind mounts.
-   Create, inspect, mount, and remove volumes.
-   Share a volume between containers.
-   Use bind mounts for host files and development.
-   Understand read-only mounts and safe cleanup.

## 1. Why persistent storage?

Databases, uploaded files, and application state must often outlive a
container. The key principle is: **manage the data lifecycle separately
from the container lifecycle.**

## 2. Container writable layer

An image consists of read-only layers. When Docker creates a container,
it adds a container-specific writable layer.

-   **Image layers:** Read-only filesystem layers from the image.
-   **Writable layer:** Stores changes made by that container outside
    mounted storage.
-   **Mounted storage:** Volumes or bind mounts that provide storage
    independently of the container's writable layer.

Files written to an ordinary directory inside a container usually go
into its writable layer. Removing the container removes that layer. Do
not store important persistent data only there.

## 3. Docker named volumes

A **volume** is Docker-managed storage that can be mounted into one or
more containers.

Create and inspect a volume:

``` bash
docker volume create day12-data
docker volume ls
docker volume inspect day12-data
```

Mount it into a container:

``` bash
docker run -it --name volume-writer \
  --mount source=day12-data,target=/data \
  alpine sh
```

Inside the container:

``` sh
echo "Persistent data from Day 12" > /data/message.txt
cat /data/message.txt
exit
```

Remove the container and read the file from a new one:

``` bash
docker rm volume-writer

docker run --rm \
  --mount source=day12-data,target=/data \
  alpine cat /data/message.txt
```

The file remains because it belongs to the volume, not the removed
container.

## 4. Bind mounts

A **bind mount** maps a selected host file or directory into a
container. The user chooses the host source path.

Create a host directory and file:

``` bash
mkdir -p ~/day12-bind
echo "File created on the host" > ~/day12-bind/host-message.txt
```

Read it from a container:

``` bash
docker run --rm \
  --mount type=bind,source="$HOME/day12-bind",target=/workspace \
  alpine cat /workspace/host-message.txt
```

A read-write bind mount normally exposes changes in both directions:
host changes can appear in the container, and container changes can
appear on the host. This is useful for source code and development
workflows.

Mount only the host paths a container needs. Use read-only mode when
writes are unnecessary.

## 5. Read-only mounts

Named volume:

``` bash
docker run --rm \
  --mount source=day12-data,target=/data,readonly \
  alpine cat /data/message.txt
```

Bind mount:

``` bash
docker run --rm \
  --mount type=bind,source="$HOME/day12-bind",target=/workspace,readonly \
  alpine ls -la /workspace
```

A read-only mount prevents writes through that mount.

## 6. Compare the storage options

  --------------------------------------------------------------------------
  Feature           Writable layer       Named volume      Bind mount
  ----------------- -------------------- ----------------- -----------------
  Location          Container-specific   Docker-managed    Chosen host path
                    layer                storage           

  Survives          No                   Yes, unless       Yes, host files
  container removal                      volume is removed remain

  Common use        Temporary container  Persistent        Source code and
                    changes              application data  host files

  Source path       No                   No                Yes
  selected by user                                         
  --------------------------------------------------------------------------

Choose: - **Named volume** for persistent application data, such as
database files. - **Bind mount** when a container needs a specific host
directory. - **Writable layer** for temporary changes that need not
survive container replacement.

## 7. Share a volume between containers

``` bash
docker volume create shared-data

docker run --rm \
  --mount source=shared-data,target=/data \
  alpine sh -c 'echo "Written by container A" > /data/shared.txt'

docker run --rm \
  --mount source=shared-data,target=/data \
  alpine cat /data/shared.txt
```

Both containers access the same volume. Docker does not automatically
coordinate application-level concurrent writes; applications must handle
shared-file access safely.

## 8. Volume lifecycle and cleanup

``` bash
docker volume ls
docker volume inspect day12-data
docker volume rm day12-data
```

A volume cannot normally be removed while a container is using it.

`docker volume prune` removes unused volumes and can permanently delete
data. Use it carefully; do not run it as routine cleanup if you need to
preserve data.

The `--rm` option on `docker run` removes the container after it exits.
It does not automatically delete a named volume mounted into it.

## 9. Hands-on experiments

### Experiment 1 --- Observe the writable layer

``` bash
docker run -it --name writable-layer-lab alpine sh
```

Inside:

``` sh
echo "Only in this container's writable layer" > /root/layer-file.txt
cat /root/layer-file.txt
exit
```

Remove the container and check from a new one:

``` bash
docker rm writable-layer-lab
docker run --rm alpine sh -c 'ls -l /root/layer-file.txt'
```

The file should not exist in the new container. Explain why.

### Experiment 2 --- Verify volume persistence

``` bash
docker volume create day12-lab-volume

docker run --rm \
  --mount source=day12-lab-volume,target=/data \
  alpine sh -c 'echo "Volume data survives container removal" > /data/proof.txt'

docker run --rm \
  --mount source=day12-lab-volume,target=/data \
  alpine cat /data/proof.txt

docker volume inspect day12-lab-volume
```

Record the output and explain why the file persists.

### Experiment 3 --- Verify a bind mount

``` bash
mkdir -p ~/day12-bind-lab
echo "Created on the WSL host" > ~/day12-bind-lab/host.txt

docker run --rm \
  --mount type=bind,source="$HOME/day12-bind-lab",target=/workspace \
  alpine cat /workspace/host.txt
```

Write a file from the container:

``` bash
docker run --rm \
  --mount type=bind,source="$HOME/day12-bind-lab",target=/workspace \
  alpine sh -c 'echo "Created by the container" > /workspace/container.txt'

cat ~/day12-bind-lab/container.txt
```

### Experiment 4 --- Test read-only behavior

``` bash
docker run --rm \
  --mount type=bind,source="$HOME/day12-bind-lab",target=/workspace,readonly \
  alpine sh -c 'cat /workspace/host.txt'
```

Attempt a write:

``` bash
docker run --rm \
  --mount type=bind,source="$HOME/day12-bind-lab",target=/workspace,readonly \
  alpine sh -c 'echo "This should fail" > /workspace/blocked.txt'
```

The write should fail. Record the error.

### Experiment 5 --- Share a volume

``` bash
docker volume create day12-shared

docker run --rm \
  --mount source=day12-shared,target=/data \
  alpine sh -c 'echo "Shared between containers" > /data/shared.txt'

docker run --rm \
  --mount source=day12-shared,target=/data \
  alpine cat /data/shared.txt
```

Explain why the second container can read the first container's file.

## 10. Docker Desktop and WSL notes

-   When running Docker commands from Ubuntu/WSL, use WSL paths such as
    `$HOME/day12-bind-lab` for bind mount sources.
-   Docker Desktop provides integration between WSL and its Linux
    container environment.
-   Named volumes are Docker-managed; they are not ordinary project
    directories under your WSL home directory.
-   Avoid manually editing a named volume's internal host path. Use
    Docker commands to manage it.

## 11. Troubleshooting

**A file disappears after container removal:** It may have been written
to the container writable layer instead of a mounted volume or bind
mount.

**A volume is in use:** Find containers with `docker ps -a`; remove the
relevant container only if safe.

**A bind source path is missing:** Verify it on the host with
`ls -la "$HOME/day12-bind-lab"`. The `--mount` form reports an error if
the bind source path does not exist.

**A read-only mount rejects writes:** That is expected. Remove
`readonly` only when write access is genuinely required.

## 12. Key terminology

-   **Writable layer:** Container-specific layer that stores filesystem
    changes.
-   **Volume:** Docker-managed storage mounted into containers.
-   **Bind mount:** A selected host path mounted into a container.
-   **Mount target:** The path inside the container where storage is
    attached.
-   **Read-only mount:** A mount that prevents writes through that
    mount.
-   **Persistence:** Retaining data beyond the lifetime of a container.

## 13. Completion checklist

-   [ ] I understand the container writable layer.
-   [ ] I verified writable-layer data does not survive container
    removal.
-   [ ] I created and inspected a named volume.
-   [ ] I verified volume data survives container removal.
-   [ ] I created and tested a bind mount.
-   [ ] I tested a read-only mount.
-   [ ] I shared a volume between two containers.
-   [ ] I can choose between a volume and a bind mount.
-   [ ] I cleaned up lab resources without deleting data
    unintentionally.

## References

-   [Docker ---
    Volumes](https://docs.docker.com/engine/storage/volumes/)
-   [Docker --- Bind
    mounts](https://docs.docker.com/engine/storage/bind-mounts/)
-   [Docker --- Storage
    overview](https://docs.docker.com/engine/storage/)
-   [Docker CLI --- docker
    volume](https://docs.docker.com/reference/cli/docker/volume/)

------------------------------------------------------------------------

## Day 12 practical task

Start with **Experiment 1 only**. Create the container, write
`/root/layer-file.txt`, remove the container, and check for the file in
a new container.

Share the commands and exact output. We will confirm why the writable
layer is tied to the container before moving to named volumes.

After completing the labs, remove only the resources you no longer need:

``` bash
docker volume rm day12-lab-volume day12-shared day12-data
rm -rf ~/day12-bind ~/day12-bind-lab
```

Do not run cleanup until you have confirmed that you no longer need the
data.
