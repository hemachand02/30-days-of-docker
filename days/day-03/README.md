# Day 03 — Understanding Docker Architecture

**Phase:** Containerization Fundamentals  
**Day:** 03 / 30  
**Focus:** Docker Client, Docker Daemon, Docker Engine, API and Registry  
**Learning Type:** Conceptual Learning + Hands-on Experiments

---

## 1. Learning Objectives

The objectives of this session are to:

- Understand the overall architecture of Docker.
- Understand the responsibilities of Docker Client and Docker Daemon.
- Understand how the Docker Client communicates with the Docker Daemon.
- Understand the role of the Docker Engine and Docker API.
- Understand how Docker interacts with image registries.
- Trace the execution workflow of a `docker run` command.
- Inspect Docker architecture using practical commands.

## 2. Introduction to Docker Architecture

Docker follows a **client-server architecture**.

This means Docker separates the component that accepts user commands from the component that performs container-management operations.

For example, when we execute:

```bash
docker run nginx
```

The Docker CLI does not directly create a container by itself.

Instead, it communicates with the Docker Daemon, which performs the requested operations.

### Main components

Docker architecture consists of these important components:

1. Docker Client
2. Docker Daemon
3. Docker Engine API
4. Docker Registry

### Architecture Overview

```text
                USER
                  |
                  v
          +---------------+
          | Docker Client |
          |   (Docker CLI)|
          +---------------+
                  |
                  | Docker API
                  | REST API requests
                  v
          +----------------+
          | Docker Daemon  |
          |    dockerd     |
          +----------------+
                  |
       +----------+----------+
       |          |          |
       v          v          v
    Images    Containers   Networks
       |
       |
       v
  Docker Registry
  (Docker Hub)
```

The client-server communication is fundamental to understanding how Docker works.

## 3. Docker Client

### 3.1 What Is the Docker Client?

The Docker Client is the interface through which users communicate with Docker.

The most commonly used client is the Docker Command Line Interface (CLI).

It accepts commands from the user and sends corresponding requests to the Docker Daemon through the Docker API.

### 3.2 Examples of Docker Client Commands

```bash
docker run nginx
docker pull alpine
docker images
docker ps
docker stop container_name
docker rm container_name
```

These commands are entered through the Docker CLI.

The CLI translates the requested operation into API requests understood by the Docker Daemon.

### 3.3 Important Responsibilities

- Accepts commands from users.
- Communicates with the Docker Daemon.
- Sends requests through the Docker API.
- Displays responses received from the daemon.
- Can communicate with a local or remote Docker Engine.

**Important:** The Docker Client and Docker Daemon do not necessarily need to run on the same machine.

A client can communicate with a Docker Engine running on a remote server.

## 4. Docker Daemon — dockerd

### 4.1 What Is the Docker Daemon?

The Docker Daemon is a background service responsible for managing Docker objects and responding to Docker API requests.

Its executable is called:

```text
dockerd
```

It runs continuously and waits for requests from Docker clients.

### 4.2 Responsibilities

The Docker Daemon manages:

- Docker images.
- Containers.
- Networks.
- Volumes.
- Image building.
- Container creation and execution.
- Container lifecycle operations.

For example, when the client sends a request to create a container, the daemon coordinates the necessary operations.

### 4.3 What Happens Internally?

Consider this command:

```bash
docker run alpine
```

The daemon performs operations such as:

1. Checking whether the requested image is available locally.
2. Pulling the image if it is unavailable.
3. Creating a container using the image.
4. Preparing the required filesystem and network configuration.
5. Starting the container process.

The daemon is responsible for managing these operations.

**Important:** The daemon is not the same thing as an individual container runtime. Docker Engine uses lower-level components to manage container execution.

## 5. Docker Engine

### 5.1 What Is Docker Engine?

Docker Engine is the core containerization technology that provides the functionality to build and run containers.

It follows a client-server architecture.

Its major components include:

- Docker Daemon (`dockerd`).
- Docker Engine API.
- Docker CLI (`docker`).

The CLI acts as the client, while the daemon performs the management operations through the API.

### 5.2 Docker Engine Architecture

```text
              Docker Engine
        +------------------------+
        |                        |
        |     Docker Client      |
        |         CLI            |
        |           |            |
        |           v            |
        |      Docker API        |
        |           |            |
        |           v            |
        |     Docker Daemon      |
        |        dockerd         |
        |           |            |
        |           v            |
        |   Docker Management   |
        |   Images / Containers |
        |   Networks / Volumes  |
        |                        |
        +------------------------+
```

The Docker Engine API provides the communication interface between the client and daemon.

Reference: [Docker Engine](https://docs.docker.com/engine/)

## 6. Docker API

### 6.1 What Is the Docker API?

The Docker Engine API is an interface that allows clients and other software to communicate with the Docker Daemon.

Docker CLI commands use this API to request operations.

The API uses HTTP-based communication and exposes endpoints for managing Docker resources.

For example, container operations have API endpoints for:

- Creating containers.
- Starting containers.
- Stopping containers.
- Inspecting containers.
- Removing containers.

### 6.2 How Does Communication Happen?

The Docker Client sends requests to the daemon through a configured endpoint.

Common communication mechanisms include:

**Unix Domain Socket**

On many Linux systems, Docker uses:

```text
/var/run/docker.sock
```

This is a local communication endpoint.

**TCP Socket**

A Docker client can also communicate with a remote daemon using a TCP endpoint.

For example:

```text
tcp://hostname:2376
```

The actual endpoint and security configuration depend on the Docker environment.

### 6.3 Important Security Consideration

Access to the Docker Daemon is highly privileged.

A user with unrestricted access to the Docker socket may effectively gain powerful control over the host.

Therefore:

- Do not expose the Docker API to an unauthenticated network.
- Protect Docker socket permissions.
- Use secure authentication and TLS when remote access is required.

## 7. Docker Registry

### 7.1 What Is a Docker Registry?

A Docker Registry is a service that stores and distributes Docker images.

It allows developers to upload, download and share container images.

A commonly used public registry is Docker Hub.

Other registry examples include:

- GitHub Container Registry.
- Amazon Elastic Container Registry (ECR).
- Google Artifact Registry.
- Azure Container Registry.

### 7.2 Why Do We Need a Registry?

Imagine creating a Java application and building its Docker image.

You want to deploy that application on another server.

Instead of manually transferring all application files and recreating the environment, you can push the image to a registry.

The destination server can then pull the image and create a container.

### 7.3 Image Distribution Workflow

```text
Developer Machine
       |
       v
  Build Image
       |
       v
  Docker Image
       |
       | docker push
       v
  Docker Registry
       |
       | docker pull
       v
 Destination Machine
       |
       v
 Create Container
```

### 7.4 Important Registry Terminology

| Term | Meaning |
|---|---|
| Registry | Service that stores and distributes images. |
| Repository | Collection of related images under a repository name. |
| Image Tag | Label used to identify a particular image version or variant. |
| Image Digest | Content-addressable identifier based on the image manifest's cryptographic digest. |
| Docker Hub | A popular public Docker image registry. |

### 7.5 Example Commands

Download an image:

```bash
docker pull nginx:latest
```

List locally available images:

```bash
docker images
```

Search Docker Hub:

```bash
docker search nginx
```

Push an image to a registry:

```bash
docker push username/repository:tag
```

The push command requires the image to be appropriately tagged and the user to have permission to publish it.

Reference: [Docker Hub](https://hub.docker.com/)

## 8. Understanding the Complete `docker run` Workflow

This is one of the most important parts of today's lesson.

Consider:

```bash
docker run -d -p 8080:80 --name my-nginx nginx:latest
```

Let's understand what happens behind this single command.

### Step 1 — Docker CLI Receives the Command

The Docker CLI receives the command from the user.

It identifies the requested operation and its parameters.

### Step 2 — CLI Communicates with Docker Daemon

The CLI sends the appropriate API request to the Docker Daemon.

### Step 3 — Image Availability Check

The daemon checks whether the requested `nginx:latest` image exists locally.

If it is unavailable, Docker attempts to retrieve it from the configured registry.

### Step 4 — Image Download

The daemon downloads the required image content from the registry.

The image is stored locally for future use.

### Step 5 — Container Creation

The daemon creates a container using the downloaded image.

It prepares the container's filesystem and required configuration.

### Step 6 — Network Configuration

Docker configures the container's network connectivity.

The `-p 8080:80` option publishes host port 8080 to container port 80.

### Step 7 — Container Startup

Docker starts the container's main process.

The `-d` option requests detached mode, so the container runs in the background.

### Step 8 — CLI Displays the Result

The CLI displays the container ID or relevant output returned by the daemon.

### Complete Workflow

```text
User
 |
 v
docker run nginx
 |
 v
Docker CLI
 |
 | API Request
 v
Docker Daemon
 |
 v
Check Local Image
 |
 +---- Image Available? ----+
 |                          |
 Yes                        No
 |                          |
 |                    Pull from Registry
 |                          |
 +------------+-------------+
              |
              v
       Create Container
              |
              v
       Configure Network
              |
              v
       Start Container
              |
              v
       Return Container ID
```

**Key observation:** A single Docker CLI command can trigger multiple operations across the Docker Engine and Registry.

## 9. Hands-on Experiments

These experiments are designed for Ubuntu WSL with Docker Desktop running.

### Experiment 1 — Inspect Docker Client and Server

**Objective:** Understand the difference between the Docker Client and Docker Server.

Command:

```bash
docker version
```

Observe:

- Client version.
- Server version.
- API version.
- Operating system and architecture information.

The Client section represents the Docker CLI.

The Server section represents the Docker Engine endpoint responding to the client.

**Execution Status:** Pending.

### Experiment 2 — Inspect Docker Engine Information

**Objective:** Understand the Docker Engine environment.

Command:

```bash
docker info
```

Observe:

- Server version.
- Storage driver.
- Operating system.
- Kernel version.
- Number of CPUs.
- Total memory.
- Container and image information.

This command provides useful information about the Docker Engine.

**Execution Status:** Pending.

### Experiment 3 — Understand Docker Contexts

**Objective:** Identify how the Docker CLI selects the Docker Engine endpoint.

Commands:

```bash
docker context ls
```

```bash
docker context show
```

```bash
docker context inspect
```

Observe:

- Available Docker contexts.
- Currently selected context.
- Docker Engine endpoint associated with that context.

A Docker context stores connection information used by the Docker CLI to communicate with a Docker Engine.

This is especially useful when working with multiple Docker environments or remote servers.

**Execution Status:** Pending.

### Experiment 4 — Run an Nginx Container

**Objective:** Observe the complete Docker workflow.

Command:

```bash
docker run -d -p 8080:80 --name day03-nginx nginx:latest
```

Inspect running containers:

```bash
docker ps
```

Inspect container details:

```bash
docker inspect day03-nginx
```

Check container logs:

```bash
docker logs day03-nginx
```

Test the web server from Ubuntu WSL:

```bash
curl http://localhost:8080
```

Expected observation:

Nginx should return an HTML response when the container is running and the published port is accessible.

Stop the container:

```bash
docker stop day03-nginx
```

Remove the container:

```bash
docker rm day03-nginx
```

**Execution Status:** Pending.

## 10. Key Technical Observations

1. Docker follows a client-server architecture.
2. Docker CLI is the primary command-line client.
3. Docker Daemon (`dockerd`) manages Docker resources.
4. Docker Engine provides the API and core container-management functionality.
5. Docker Client communicates with Docker Daemon through the Docker API.
6. Docker Registry stores and distributes container images.
7. Docker Hub is a popular public registry.
8. A `docker run` command can involve image retrieval, container creation, network configuration and process startup.
9. Docker contexts determine which Docker Engine endpoint the CLI communicates with.
10. Access to the Docker Daemon must be protected because it provides powerful control over the host environment.

## 11. Key Takeaways

Docker is more than a command-line tool.

It is a client-server platform in which the Docker CLI communicates with the Docker Daemon through the Docker Engine API.

The daemon manages images, containers, networks and volumes, while registries provide a mechanism for distributing images between environments.

Understanding this architecture helps explain what happens behind everyday Docker commands.

## 12. References

- [Docker Architecture Overview](https://docs.docker.com/get-started/docker-overview/)
- [Docker Engine Documentation](https://docs.docker.com/engine/)
- [Docker Engine API](https://docs.docker.com/reference/api/engine/)
- [Docker CLI Reference](https://docs.docker.com/reference/cli/docker/)
- [Docker Context Documentation](https://docs.docker.com/engine/manage-resources/contexts/)
- [Docker Hub](https://hub.docker.com/)
- [Docker Registry Documentation](https://docs.docker.com/registry/)

---

**Next Session:** Day 04 — Docker Images, Layers and Image Management

**Learning Journey:** 30 Days of Docker