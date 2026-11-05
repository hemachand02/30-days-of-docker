# Day 01 — Introduction to Containerization

**Phase:** Containerization Fundamentals  
**Day:** 01 / 30  
**Focus:** Understanding containerization and the problems it solves  
**Learning Type:** Conceptual Learning + Hands-on Experiments

---

## 1. Learning Objectives

The objectives of this session are to:

- Understand traditional application deployment challenges.
- Identify dependency and environment inconsistencies.
- Understand containerization and its purpose.
- Explore how containers address deployment problems.
- Understand the role of Docker in modern application development.
- Perform basic experiments using Docker containers.

## 2. Understanding the Deployment Problem

Consider a Java application developed on a developer's machine.

The application requires:

- Java Runtime Environment (JRE).
- MySQL database.
- External Java libraries.
- Environment variables.
- Application configuration files.

The application works correctly on the developer's machine. However, when deployed to another machine, it fails.

Possible reasons include:

- Different Java versions.
- Incompatible database versions.
- Missing dependencies.
- Different environment variables.
- Operating system configuration differences.

This creates an environment inconsistency problem.

A common expression used to describe this situation is:

**"It works on my machine!"**

The underlying issue is that the application depends on an execution environment that has not been consistently reproduced across machines.

### Major deployment challenges

| Problem | Explanation |
|---|---|
| Dependency conflicts | Different applications may require incompatible dependency versions. |
| Environment inconsistency | Development, testing and production environments may differ. |
| Configuration management | Applications require specific environment variables and configuration files. |
| Deployment complexity | Setting up application dependencies manually can be time-consuming. |
| Resource management | Multiple applications may compete for shared system resources. |
| Portability | Moving applications between different environments can require additional configuration. |

## 3. What Is Containerization?

Containerization is an operating-system-level virtualization technique that packages an application and its required user-space dependencies into an isolated execution environment called a **container**.

A container provides an environment in which an application can execute with its required dependencies and configurations.

A containerized application can be deployed across compatible environments without manually reproducing all its application-level dependencies on every machine.

### Important terminology

| Terminology | Description |
|---|---|
| Application | Software that performs a particular task. |
| Dependency | Software, libraries or components required by an application. |
| Container | An isolated execution environment for an application. |
| Container Image | A packaged, reusable template used to create containers. |
| Container Runtime | Software responsible for creating and running containers. |
| Host Operating System | The operating system providing the underlying kernel and system resources. |
| Container Isolation | Mechanisms that separate container processes and resources from other environments. |

### Important technical distinction

A Linux container does not contain and boot an independent Linux kernel.

Instead, Linux containers share the host's Linux kernel while using kernel isolation and resource-control mechanisms.

This is an important distinction between containers and traditional virtual machines.

## 4. Problems Solved by Containerization

### 4.1 Dependency Conflicts

Consider two Java applications:

- Application A requires Java 17.
- Application B requires Java 21.

Managing both applications on the same host may introduce dependency and configuration conflicts.

Containers allow each application to have its own user-space environment with the required Java runtime.

### 4.2 Environment Inconsistency

Development, testing and production environments may have different configurations.

Container images help standardize application dependencies and runtime environments across these stages.

### 4.3 Application Isolation

Applications running directly on the same operating system may interact through shared processes, files, network resources and system resources.

Containers use Linux kernel mechanisms such as namespaces and cgroups to provide isolation and resource management.

### 4.4 Deployment Complexity

Without containerization, administrators may need to manually configure application runtimes, dependencies and supporting services.

Container images provide a standardized way to package and distribute application environments.

### 4.5 Portability

Container images can be transferred between compatible environments and used to create containers without manually installing every application dependency again.

Portability still depends on factors such as CPU architecture, kernel compatibility, runtime support and external configuration.

### 4.6 Scalability

Multiple container instances can be created from the same image.

This makes it easier to run additional application instances when application demand increases.

Containerization alone does not automatically provide load balancing or orchestration; additional infrastructure may be required.

## 5. How Docker Helps

Docker is a containerization platform that provides tools to build, distribute and run containerized applications.

It simplifies the container workflow by providing a consistent interface for working with images and containers.

### Important Docker components

| Component | Responsibility |
|---|---|
| Docker Client | Sends commands and requests to the Docker Engine. |
| Docker Daemon | Manages Docker objects such as images, containers, networks and volumes. |
| Docker Engine | Core platform responsible for building and running containers. |
| Docker Image | Template used to create containers. |
| Docker Container | Running or stopped instance created from an image. |
| Docker Registry | Stores and distributes container images. |
| Docker Compose | Defines and manages multi-container applications. |

These components will be explored in detail in upcoming sessions.

Reference: [Docker Overview](https://docs.docker.com/get-started/docker-overview/)

## 6. Traditional vs Containerized Deployment

### Traditional deployment

Applications are installed directly into the host operating system or deployed inside virtual machines.

The application uses the host environment and its installed dependencies.

### Containerized deployment

Applications run inside isolated container environments managed by a container runtime.

Each container can have its own application dependencies and user-space environment while sharing the underlying host kernel.

| Aspect | Traditional deployment | Containerized deployment |
|---|---|---|
| Application dependencies | Installed and managed in the host environment | Packaged in container images |
| Isolation | Depends on host configuration and deployment model | Provided through container isolation mechanisms |
| Deployment | Often requires manual environment setup | Uses standardized container images |
| Resource overhead | Depends on deployment architecture | Generally lower than full VM-based deployment |
| Portability | Environment recreation may be necessary | Images provide a more consistent deployment unit |
| Scalability | Requires deployment and infrastructure configuration | Multiple instances can be created from images |

**Note:** Traditional deployment is a broad category. It can include bare-metal installations and virtual-machine deployments, which have different isolation and resource characteristics.

## 7. Hands-on Experiments

The following experiments explore Docker's basic container workflow and the relationship between container user space and the host kernel.

### Experiment 1 — Docker Environment Verification

**Objective:** Verify the Docker client and Engine connectivity.

Command:

```bash
docker version
```

This command displays information about the Docker client and server.

Additional commands:

```bash
docker context ls
docker context show
```

These commands identify the available Docker contexts and the currently selected context.

**Expected observation:**

The Docker CLI communicates with the Docker Engine through the configured endpoint.

**Execution status:** Pending.

### Experiment 2 — Running the Hello World Container

**Objective:** Understand the basic image-to-container execution workflow.

Command:

```bash
docker run --rm hello-world
```

Execution workflow:

1. Docker checks whether the image exists locally.
2. If the image is unavailable, Docker pulls it from a registry.
3. Docker creates a container using the image.
4. The container executes its configured program.
5. The program prints a confirmation message.
6. The container is removed after execution because of the `--rm` option.

**Expected observation:**

A successful execution prints a message confirming that Docker is working.

**Execution status:** Pending.

### Experiment 3 — Investigating Alpine Linux

**Objective:** Observe the user-space operating system environment inside a container.

Command:

```bash
docker run --rm alpine:latest cat /etc/os-release
```

This displays information about the Alpine Linux distribution inside the container.

Second command:

```bash
docker run --rm alpine:latest uname -r
```

This displays the kernel version visible to the container.

**Important observation:**

The container provides its own user-space distribution, but it does not boot an independent Linux kernel.

Linux containers share the underlying host kernel.

**Execution status:** Pending.

## 8. Key Technical Observations

The following concepts are important takeaways from this session:

1. Applications depend on their execution environments, not just their source code.
2. Dependency and configuration inconsistencies can cause deployment failures.
3. Containerization provides standardized application environments.
4. Container images are used to create container instances.
5. Linux containers share the host kernel rather than booting independent kernels.
6. Docker simplifies the process of building, distributing and running containers.
7. Containerization improves portability but does not eliminate every environment compatibility issue.

## 9. Key Takeaways

Containerization addresses many challenges associated with application deployment and environment consistency.

It allows applications and their required user-space dependencies to be packaged into standardized images and executed in isolated environments.

Docker provides an accessible ecosystem for managing this workflow.

Understanding the underlying Linux mechanisms is essential to understanding how containers actually work.

## 10. References

- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Overview](https://docs.docker.com/get-started/docker-overview/)
- [Docker Containers](https://docs.docker.com/engine/containers/)
- [Docker Images](https://docs.docker.com/engine/images/)
- [Linux Kernel Documentation — Namespaces](https://docs.kernel.org/admin-guide/namespaces/)

---

**Next Session:** Day 02 — Traditional Deployment vs Virtual Machines vs Containers

**Learning Journey:** 30 Days of Docker
