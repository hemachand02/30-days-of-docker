# Day 10 --- Docker Networking Fundamentals

## Overview

**Focus:** Understand how containers communicate with one another, with
the host, and with external networks.

So far, we have studied how containers isolate processes and manage
resource consumption. Today, we begin Docker networking: how a container
gets network connectivity, how Docker networks connect containers, and
how published ports make container services reachable from outside.

## Learning objectives

By the end of this day, you should be able to:

-   Explain why containers need network connectivity.
-   Understand the role of a Docker network.
-   Describe the default bridge network.
-   Distinguish the default bridge from a user-defined bridge network.
-   Inspect container network configuration and IP addresses.
-   Explain port publishing and the difference between `EXPOSE` and
    `-p`.
-   Test communication between containers on a user-defined bridge.
-   Clean up networks and containers created for experiments.

------------------------------------------------------------------------

## 1. Why does Docker need networking?

Containers are isolated environments, but real applications rarely
operate alone.

Consider a simple web application:

-   A browser sends a request to a web server.
-   The web server communicates with an application backend.
-   The backend queries a database.

These components may run in separate containers. They need a way to
communicate while retaining useful network isolation.

Docker networking provides the connectivity mechanisms that let
containers communicate with each other, the host, and external networks.

## 2. Essential networking terminology

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  Network interface                   A software or hardware interface
                                      through which a system sends and
                                      receives network traffic

  IP address                          An address used to identify an
                                      endpoint on an IP network

  Subnet                              A logical range of IP addresses

  Gateway                             A device or endpoint used to reach
                                      destinations outside the local
                                      subnet

  Bridge                              A virtual Layer 2 network device
                                      that connects network interfaces

  Port                                A transport-layer endpoint used by
                                      applications, such as TCP port 80

  Port publishing                     Mapping a host port to a port in a
                                      container

  DNS                                 A naming system that resolves names
                                      to addresses
  -----------------------------------------------------------------------

A container can have its own network interface and IP configuration
within the network namespace that Docker configures for it.

## 3. Docker network drivers

A **network driver** determines how a Docker network provides
connectivity.

Common drivers include:

-   **bridge:** Connects containers on a single Docker host. Common for
    local development and multi-container applications.
-   **host:** Removes network isolation between the container and the
    Docker host's network stack on supported Linux environments. Port
    publishing is not used in the usual way with host networking.
-   **none:** Disables normal external network connectivity for the
    container, leaving only loopback.
-   **overlay:** Connects services across multiple Docker hosts,
    typically in a swarm context.
-   **macvlan:** Gives containers addresses on a physical network using
    MAC-address-based connectivity.

This learning track focuses first on **bridge networking**, especially
user-defined bridge networks.

## 4. The default bridge network

Docker normally provides a default network named `bridge`.

List available networks:

``` bash
docker network ls
```

Inspect the default bridge:

``` bash
docker network inspect bridge
```

When a container is started without an explicit network, Docker
generally connects it to the default bridge network.

Inspect a running container's network configuration:

``` bash
docker run -d --name net-default alpine sleep 300
docker inspect net-default
```

For a compact view:

``` bash
docker inspect -f '{{json .NetworkSettings.Networks}}' net-default
```

The container receives network configuration from Docker, including an
IP address when applicable.

Clean up:

``` bash
docker rm -f net-default
```

### A limitation of the default bridge

Containers attached to the default bridge do not get the same automatic
container-name DNS resolution that user-defined bridge networks provide.

Legacy links exist, but they are not the recommended approach for new
applications.

## 5. User-defined bridge networks

A **user-defined bridge network** is a bridge network that you create
and manage explicitly.

Create one:

``` bash
docker network create app-network
```

Inspect it:

``` bash
docker network inspect app-network
```

Start two containers on it:

``` bash
docker run -dit --name app-one --network app-network alpine
docker run -dit --name app-two --network app-network alpine
```

Because both containers share a user-defined bridge network, Docker's
embedded DNS can resolve container names on that network.

Test name resolution and connectivity from `app-one`:

``` bash
docker exec app-one ping -c 3 app-two
```

If the `ping` command is unavailable in the selected image, install the
appropriate package only if needed, or use another available network
diagnostic tool. Do not interpret a missing utility as proof that the
network is broken.

Inspect network membership:

``` bash
docker network inspect app-network
```

You should see the connected containers in the network details.

Clean up:

``` bash
docker rm -f app-one app-two
docker network rm app-network
```

## 6. Port publishing: making a container service reachable

A process listening inside a container is not automatically reachable
through a host port.

Docker's `-p` or `--publish` option maps a port on the Docker host to a
port in the container.

General form:

``` bash
docker run -p HOST_PORT:CONTAINER_PORT IMAGE
```

For example, start an Nginx web server:

``` bash
docker run -d --name web-lab -p 8080:80 nginx:alpine
```

This maps host port `8080` to container port `80`.

Check the mapping:

``` bash
docker port web-lab
docker ps
```

Open this address in a browser on the same machine:

``` text
http://localhost:8080
```

You should see the Nginx welcome page if the container started
successfully and the port is reachable in your environment.

Clean up:

``` bash
docker rm -f web-lab
```

### Binding to a particular host address

You can specify the host IP address as part of the mapping:

``` bash
docker run -d --name web-local-only -p 127.0.0.1:8081:80 nginx:alpine
```

This requests publishing on the host's loopback interface. It is useful
when the service should be reachable locally rather than exposed on
every host interface. Actual reachability can vary with Docker Desktop
and host networking.

Clean up:

``` bash
docker rm -f web-local-only
```

## 7. `EXPOSE` vs `-p`

These are often confused.

### `EXPOSE`

`EXPOSE` is Dockerfile metadata documenting the port that an application
is intended to listen on.

Example:

``` dockerfile
FROM nginx:alpine
EXPOSE 80
```

By itself, `EXPOSE` does **not** publish the port on the Docker host.

### `-p` / `--publish`

The `-p` option configures an actual host-to-container port mapping when
a container is created.

Example:

``` bash
docker run -d --name web-lab -p 8080:80 nginx:alpine
```

Remember:

-   `EXPOSE 80`: documents the intended container port.
-   `-p 8080:80`: publishes host port 8080 to container port 80.
-   `-P`: publishes exposed ports to randomly assigned host ports.

## 8. Hands-on labs

Use the experiments in order. These labs use small containers and do not
require privileged mode.

### Experiment 1 --- Inspect Docker networks

Run:

``` bash
docker network ls
docker network inspect bridge
```

Record: - The network names and drivers shown by `docker network ls`. -
The subnet and gateway shown for the default bridge, if available. - The
purpose of the default bridge network.

No cleanup is required for this experiment.

### Experiment 2 --- Inspect a container's network settings

Create a container:

``` bash
docker run -d --name network-inspect-lab alpine sleep 300
```

Inspect its network details:

``` bash
docker inspect -f '{{json .NetworkSettings.Networks}}' network-inspect-lab
```

Also inspect the container's network namespace from inside it:

``` bash
docker exec network-inspect-lab ip addr
```

If `ip` is not installed in the Alpine image, do not assume the
container's network is unavailable. You can inspect the Docker metadata
with `docker inspect`.

Clean up:

``` bash
docker rm -f network-inspect-lab
```

### Experiment 3 --- Test communication using a user-defined bridge

Create a network:

``` bash
docker network create day10-network
```

Start two containers:

``` bash
docker run -dit --name node-a --network day10-network alpine
docker run -dit --name node-b --network day10-network alpine
```

Test name resolution and connectivity:

``` bash
docker exec node-a ping -c 3 node-b
```

Inspect the network:

``` bash
docker network inspect day10-network
```

Record whether the container name resolved and whether packets were
returned.

Clean up:

``` bash
docker rm -f node-a node-b
docker network rm day10-network
```

### Experiment 4 --- Publish a web server port

Start Nginx:

``` bash
docker run -d --name day10-web -p 8080:80 nginx:alpine
```

Inspect the mapping:

``` bash
docker port day10-web
docker ps --filter name=day10-web
```

From your browser, visit:

``` text
http://localhost:8080
```

Or test from the terminal if `curl` is available:

``` bash
curl http://localhost:8080
```

Record the result and explain the meaning of `8080:80`.

Clean up:

``` bash
docker rm -f day10-web
```

------------------------------------------------------------------------

## 9. Troubleshooting

### The published web page does not load

Check whether the container is running:

``` bash
docker ps -a
docker logs day10-web
docker port day10-web
```

Check whether the host port is already in use. If so, choose another
host port, such as `8082:80`.

### A container name does not resolve

Check that both containers are connected to the same user-defined
network:

``` bash
docker network inspect day10-network
```

Container-name DNS resolution is supported on user-defined bridge
networks. Do not assume the default bridge behaves the same way.

### `ping` is not found

A minimal image may not include diagnostic utilities. A missing `ping`
executable is not itself a networking failure. Inspect Docker's network
metadata or use an image that includes the required diagnostic tool.

### WSL and Docker Desktop networking behave differently

With Docker Desktop, Linux containers run inside Docker Desktop's Linux
environment. Host-to-container connectivity is integrated with Windows
and WSL, so the exact network interfaces and addresses visible inside
WSL may differ from those inside the container environment.

Prefer testing the published service through
`http://localhost:<published-port>` before drawing conclusions from an
IP address alone.

------------------------------------------------------------------------

## 10. Key concepts to remember

-   Containers have network configuration managed through network
    namespaces and Docker networking.
-   A Docker network connects container endpoints.
-   The default bridge and a user-defined bridge have different
    name-resolution behavior.
-   User-defined bridge networks provide automatic DNS resolution
    between attached containers by name.
-   Port publishing maps a host port to a container port.
-   `EXPOSE` documents a port; it does not publish it.
-   Network isolation and port publishing are separate concerns.

## 11. Completion checklist

-   [ ] I listed Docker networks using `docker network ls`.
-   [ ] I inspected the default bridge network.
-   [ ] I inspected a container's network settings.
-   [ ] I created a user-defined bridge network.
-   [ ] I tested container-to-container communication by name.
-   [ ] I published a container port using `-p`.
-   [ ] I tested the published service through localhost.
-   [ ] I removed all lab containers and networks.

## References

-   [Docker documentation --- Networking
    overview](https://docs.docker.com/engine/network/)
-   [Docker documentation --- Bridge network
    driver](https://docs.docker.com/engine/network/drivers/bridge/)
-   [Docker documentation --- Port
    publishing](https://docs.docker.com/engine/network/port-publishing/)
-   [Docker documentation ---
    `docker network`](https://docs.docker.com/reference/cli/docker/network/)
-   [Docker documentation --- Dockerfile
    `EXPOSE`](https://docs.docker.com/reference/dockerfile/#expose)

------------------------------------------------------------------------

## Day 10 practical task

Start with **Experiment 1 only**. Run:

``` bash
docker network ls
docker network inspect bridge
```

Share the output of `docker network ls` and the relevant `bridge`
network details. We will identify the network driver, subnet, and
gateway before moving on to container-to-container communication.
