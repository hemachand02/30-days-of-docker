# Day 04 — Docker Images, Layers and Image Management

**Phase:** Containerization Fundamentals  
**Day:** 04 / 30  
**Focus:** Understanding Docker images, image layers, writable container layers and image management  
**Learning Type:** Conceptual Learning + Hands-on Experiments

---

## 1. Learning Objectives

The objectives of this session are to:

- Understand what a Docker image is and why it is required.
- Understand the difference between Docker images and containers.
- Understand image layers and their relationship with Dockerfiles.
- Explore how Docker reuses image layers.
- Understand the writable container layer.
- Learn about image tags and digests.
- Perform basic image management operations using Docker CLI.

## 2. What Is a Docker Image?

A Docker image is an immutable, packaged template used to create containers.

It contains the application and the required user-space components needed to run it.

Depending on the image, these components may include:

- Application source code or executable files.
- Runtime environments.
- System libraries.
- Application dependencies.
- Filesystem content.
- Configuration and metadata.

For example, the official Nginx image contains the files and configuration required to run an Nginx web server.

### 2.1 Why Do We Need Images?

Imagine deploying a Java application on multiple servers.

Without a standardized package, each server may require manual installation of:

- A compatible Java runtime.
- Application dependencies.
- Application files.
- Configuration.
- Supporting tools.

A Docker image packages the required user-space environment into a reusable artifact.

Instead of manually reproducing the application environment on every machine, we can distribute the image and create containers from it.

### 2.2 Important Characteristics

| Characteristic | Explanation |
|---|---|
| Immutable | An existing image is not modified when a container writes data. |
| Reusable | Multiple containers can be created from the same image. |
| Portable | Images can be distributed across compatible environments. |
| Layered | Images are composed of filesystem layers. |
| Versioned | Tags and digests help identify image versions and content. |

**Important:** An image is a template, not a running application.

A container is created from an image.

## 3. Docker Image vs Docker Container

These two concepts are closely related but serve different purposes.

### Docker Image

- A packaged template.
- Contains application files and dependencies.
- Used to create containers.
- Its filesystem layers are immutable.

### Docker Container

- An instance created from an image.
- Runs an application process.
- Has an isolated execution environment.
- Uses the image's filesystem layers plus a writable container layer.

### Comparison

| Feature | Docker Image | Docker Container |
|---|---|---|
| Purpose | Template for creating containers | Executes an application |
| State | Immutable artifact | Can be running, stopped or paused |
| Filesystem | Read-only image layers | Image layers plus writable layer |
| Creation | Pulled or built | Created from an image |
| Reusability | Can create multiple containers | Represents an individual instance |

### Relationship

```text
             Docker Image
                  |
       +----------+----------+
       |          |          |
       v          v          v
  Container 1 Container 2 Container 3
       |          |          |
    Process A  Process B  Process C
```

One image can be used to create multiple independent container instances.

## 4. Understanding Docker Image Layers

This is one of the most important concepts in Docker.

### 4.1 What Is an Image Layer?

A Docker image is composed of multiple filesystem layers.

Each layer represents a set of filesystem changes.

For example, consider an image containing:

1. A minimal Linux user-space filesystem.
2. Required system libraries.
3. An application runtime.
4. Application files.

These components can be represented as a sequence of filesystem layers.

### 4.2 Example Image Architecture

```text
       Docker Image
            |
     +--------------+
     | Application  |
     | Files Layer  |
     +--------------+
     | Java Runtime |
     | Layer        |
     +--------------+
     | System       |
     | Libraries    |
     +--------------+
     | Base Image   |
     | Layer        |
     +--------------+
```

Each layer contributes filesystem content to the final image.

The image's filesystem is presented as a unified view when a container is created.

### 4.3 Are Layers Independent Filesystems?

No.

An image is not simply a collection of independent operating systems.

Its filesystem layers are combined to provide one coherent filesystem view.

Container storage drivers or snapshotters manage how these layers are made available.

### 4.4 Why Does Docker Use Layers?

Layering provides several benefits.

**1. Reusability**

Different images can share common layers.

For example, two application images may use the same base image.

Docker does not need to store duplicate copies of identical content when it can reuse the same image layers.

**2. Efficient Distribution**

When pulling an image, Docker can reuse layers that are already available locally.

Only missing image content needs to be downloaded.

**3. Build Caching**

Docker can reuse previously built layers when the relevant build instructions and their dependencies have not changed.

This can reduce image build time.

**4. Storage Efficiency**

Shared layers reduce unnecessary duplication of image data.

## 5. Image Layers and Dockerfile Instructions

A Dockerfile is a text file containing instructions used to build a Docker image.

Example:

```dockerfile
FROM alpine:latest

RUN apk add --no-cache curl

WORKDIR /app

COPY app.sh .

CMD ["sh", "app.sh"]
```

Let's understand how these instructions contribute to an image.

| Instruction | Purpose |
|---|---|
| `FROM` | Selects the base image. |
| `RUN` | Executes commands during image building. |
| `WORKDIR` | Sets the working directory. |
| `COPY` | Copies files into the image. |
| `CMD` | Defines the default command when a container starts. |

Filesystem-changing instructions such as `RUN` and `COPY` can produce image layers.

Instructions such as `CMD` and `WORKDIR` primarily contribute image configuration or metadata rather than necessarily creating filesystem layers.

**Important:** Not every Dockerfile instruction creates a filesystem layer.

We will study Dockerfiles and image-building optimization in depth during the image-building phase.

## 6. Image Layers vs Writable Container Layer

This distinction is extremely important.

When Docker creates a container from an image, it adds a writable layer above the image's read-only layers.

This allows the container to modify its own filesystem without modifying the original image.

### Architecture

```text
        Container Filesystem
       +----------------------+
       | Writable Layer       |
       | Container Changes    |
       +----------------------+
       | Image Layer 3        |
       | Application Files    |
       +----------------------+
       | Image Layer 2        |
       | Runtime Dependencies |
       +----------------------+
       | Image Layer 1        |
       | Base Filesystem      |
       +----------------------+
```

### 6.1 Read-Only Image Layers

These layers belong to the image.

They are not modified by normal container filesystem writes.

### 6.2 Writable Container Layer

This layer stores changes made by the container.

Examples:

- Creating a new file.
- Modifying an existing file.
- Deleting a file from the container's filesystem view.

These changes belong to that container's writable layer.

### 6.3 Copy-on-Write (CoW)

Copy-on-Write is a technique used by storage systems to avoid immediately copying shared data when it is not necessary.

In traditional Docker storage-driver implementations, when a container modifies a file that exists in a lower read-only layer, the storage driver may copy that file into the writable layer before modifying it.

This allows the container to modify its own filesystem view without changing the image.

The exact implementation depends on the storage driver or snapshotter.

### 6.4 What Happens When a Container Is Deleted?

When a container is removed, its writable layer is removed as well.

The original image remains available unless it is separately deleted.

This is why data that must survive container removal should be stored using Docker volumes or bind mounts.

We will study persistent storage in a later phase.

## 7. Docker Image Tags and Digests

Images need identifiers so developers can reference specific versions.

Two important concepts are tags and digests.

### 7.1 Image Tags

A tag is a human-readable label associated with an image.

Example:

```bash
docker pull nginx:latest
docker pull nginx:1.27
```

Here:

- `nginx` is the repository name.
- `latest` and `1.27` are image tags.

A tag is a convenient reference to an image version or variant.

**Important:** The `latest` tag does not necessarily mean the newest release. It is simply a tag that a publisher can move to another image.

### 7.2 Image Digests

An image digest is a content-addressable identifier based on a cryptographic hash.

Example format:

```text
nginx@sha256:<digest>
```

A digest identifies specific content, such as an image manifest.

Unlike a mutable tag, a digest provides a stable reference to that content.

### 7.3 Tags vs Digests

| Feature | Tag | Digest |
|---|---|---|
| Format | Human-readable label | Cryptographic content identifier |
| Mutability | Can be moved to another image | Identifies specific content |
| Purpose | Convenient version reference | Precise content identification |
| Reproducibility | Depends on tag stability | More reliable for pinning content |

For reproducible deployments, digest-based references can help ensure that the intended image content is used.

## 8. Docker Image Management Commands

Docker provides image-management commands through the Docker CLI.

### 8.1 Pull an Image

Downloads an image from a registry.

```bash
docker pull alpine:latest
```

### 8.2 List Local Images

Displays images available locally.

```bash
docker images
```

Modern equivalent:

```bash
docker image ls
```

### 8.3 Inspect an Image

Displays detailed image metadata.

```bash
docker image inspect alpine:latest
```

This provides information such as:

- Image ID.
- Repository tags.
- Architecture.
- Operating system.
- Image configuration.
- Root filesystem layer information.

To display only the image's layer identifiers:

```bash
docker image inspect alpine:latest \
  --format '{{json .RootFS.Layers}}'
```

### 8.4 View Image History

Displays the image's build history.

```bash
docker image history alpine:latest
```

This helps investigate the commands and layers associated with an image.

**Note:** Image history is useful for understanding build history, but it is not necessarily a complete representation of every filesystem detail.

### 8.5 Remove an Image

```bash
docker image rm alpine:latest
```

The image cannot normally be removed while it is being used by existing containers.

Remove dependent containers first when appropriate.

### 8.6 Remove Unused Images

```bash
docker image prune
```

This removes dangling images.

For broader cleanup:

```bash
docker image prune -a
```

This can remove all images not referenced by existing containers.

Use cleanup commands carefully because images may need to be downloaded again.

## 9. Hands-on Experiments

These experiments are designed for Ubuntu WSL with Docker Desktop running.

### Experiment 1 — Pull and Inspect an Image

**Objective:** Understand how Docker downloads and stores images.

Commands:

```bash
docker pull alpine:latest
```

```bash
docker image ls
```

```bash
docker image inspect alpine:latest
```

Observe:

- Image repository.
- Image tag.
- Image ID.
- Image size.
- Architecture.
- Operating system.

**Execution Status:** Pending.

### Experiment 2 — Investigate Image Layers

**Objective:** Explore the filesystem layers of a Docker image.

Commands:

```bash
docker image inspect alpine:latest \
  --format '{{json .RootFS.Layers}}'
```

Then:

```bash
docker image history alpine:latest
```

Observe:

- Layer identifiers.
- Image history entries.
- How Docker represents image filesystem changes.

Different images may contain different numbers of layers.

**Execution Status:** Pending.

### Experiment 3 — Observe the Writable Container Layer

**Objective:** Understand how a container modifies its own filesystem without modifying the original image.

First, create a container:

```bash
docker run -dit --name day04-layer-test alpine:latest sh
```

Create a file inside the container:

```bash
docker exec day04-layer-test sh -c \
  'echo "Container-specific data" > /test.txt'
```

Verify the file:

```bash
docker exec day04-layer-test cat /test.txt
```

Now inspect the container's filesystem changes:

```bash
docker diff day04-layer-test
```

Look for the `/test.txt` entry.

Docker uses these change indicators:

| Indicator | Meaning |
|---|---|
| A | File or directory added |
| C | File or directory changed |
| D | File or directory deleted |

Now remove the container:

```bash
docker rm -f day04-layer-test
```

The container's writable layer is removed.

The Alpine image remains available:

```bash
docker image ls alpine
```

**Execution Status:** Pending.

### Experiment 4 — Understand Image Reuse

**Objective:** Observe that multiple containers can be created from the same image.

Create two containers:

```bash
docker run -dit --name alpine-one alpine:latest sh
```

```bash
docker run -dit --name alpine-two alpine:latest sh
```

Inspect them:

```bash
docker ps
```

Observe:

- Both containers use the same image.
- They have different container IDs.
- Each container has its own writable layer.

Clean up:

```bash
docker rm -f alpine-one alpine-two
```

The shared image remains available.

**Execution Status:** Pending.

## 10. Key Technical Observations

1. Docker images are immutable templates used to create containers.
2. Containers are instances created from images.
3. Images consist of read-only filesystem layers.
4. Multiple images can reuse common layers.
5. Docker can reuse available image layers during image pulls.
6. Build caching can reuse previously generated build results.
7. Containers have an additional writable layer above their image layers.
8. Container filesystem modifications do not normally modify the original image.
9. Removing a container removes its writable layer.
10. Tags are convenient image references, while digests identify specific content.
11. Persistent application data should not depend solely on the container's writable layer.

## 11. Key Takeaways

Docker images provide reusable and distributable application environments.

Their layered filesystem architecture helps reduce unnecessary duplication and enables efficient image distribution and build caching.

When a container is created, Docker combines the image's read-only layers with a separate writable layer.

Understanding this architecture is essential before learning how to build optimized Docker images using Dockerfiles.

## 12. References

- [Docker — Images](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)
- [Docker — Image Management](https://docs.docker.com/reference/cli/docker/image/)
- [Docker — Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Docker — Storage Drivers](https://docs.docker.com/engine/storage/drivers/)
- [Docker — Container Filesystems](https://docs.docker.com/engine/storage/)
- [Docker — Image Digests](https://docs.docker.com/dhi/core-concepts/digests/)

---

**Next Session:** Day 05 — Container Creation, Execution and Lifecycle

**Learning Journey:** 30 Days of Docker