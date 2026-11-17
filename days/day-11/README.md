# Day 11 --- Docker Networking in Practice

## Overview

**Focus:** Container name resolution, Docker DNS, network isolation,
aliases, and connecting running containers to networks.

Day 10 introduced bridge networks and port publishing. Today, build on
that foundation by examining how containers discover one another and how
network membership controls connectivity.

## Learning objectives

-   Explain Docker's embedded DNS on user-defined bridge networks.
-   Reach a containerized service by container name rather than a
    hard-coded IP.
-   Distinguish container-to-container traffic from host-to-container
    access.
-   Inspect network membership and resolver configuration.
-   Connect and disconnect a running container from a network.
-   Use network aliases and test basic network isolation.

## 1. Container name resolution

When two containers share a **user-defined bridge network**, Docker
provides embedded DNS-based name resolution. A client container can
generally reach a web container using its name, for example:

``` text
http://web:80
```

This is preferable to hard-coding an IP address. Container IP addresses
can change when containers are recreated or reconnected to networks,
while a container name or network alias can remain the application's
stable reference.

Name resolution is scoped to the network. Containers need to share a
suitable user-defined network for this behavior.

## 2. Container-to-container vs host-to-container

**Container-to-container:** Containers on the same user-defined bridge
can communicate using the destination container's name and listening
port. This does not require publishing the destination port to the host.

**Host-to-container:** To make a service reachable through a host port,
publish a mapping such as:

``` bash
docker run -d --name web -p 8080:80 nginx:alpine
```

This maps host port `8080` to container port `80`.

Remember: - A shared Docker network provides a path for
container-to-container communication. - `-p` publishes a container port
through a host port. - Publishing is not required just because another
container needs to access the service on the same user-defined network.

## 3. Network membership and isolation

A container can join one or more Docker networks. Containers attached to
separate user-defined bridge networks are isolated from one another at
the network level by default. Connecting a container to a shared network
gives it a path to communicate with other members, subject to listening
services and other controls.

This can help separate application tiers. For example, a frontend and
application service may share one network, while the application service
and database share another. The database need not join the frontend
network.

Network separation is useful, but it is not a complete security boundary
by itself.

## 4. Build a two-container network

Create a user-defined bridge:

``` bash
docker network create day11-app-net
```

Start Nginx without publishing a host port:

``` bash
docker run -d --name day11-web --network day11-app-net nginx:alpine
```

Start an Alpine client:

``` bash
docker run -dit --name day11-client --network day11-app-net alpine
```

Install curl in this disposable lab client:

``` bash
docker exec day11-client sh -c "apk add --no-cache curl"
```

Request Nginx using its container name:

``` bash
docker exec day11-client curl -I http://day11-web
```

A successful response commonly includes `HTTP/1.1 200 OK`. The key point
is that the client used `day11-web`, not a manually copied IP address.

You can also test name resolution/connectivity with:

``` bash
docker exec day11-client ping -c 3 day11-web
```

If `ping` is missing, that does not prove networking is broken. Minimal
images omit diagnostic utilities.

## 5. Inspect DNS and network membership

Inspect the network:

``` bash
docker network inspect day11-app-net
```

Look for its driver, subnet, gateway, and connected containers.

Inspect each container's network settings:

``` bash
docker inspect -f '{{json .NetworkSettings.Networks}}' day11-web
docker inspect -f '{{json .NetworkSettings.Networks}}' day11-client
```

Inspect the client's resolver configuration:

``` bash
docker exec day11-client cat /etc/resolv.conf
```

Docker's embedded DNS is commonly available at `127.0.0.11` inside
containers on user-defined networks. Details may vary by environment.

## 6. Network aliases

A **network alias** is an additional name by which a container can be
discovered on a particular network.

Create another Nginx container with an alias:

``` bash
docker run -d \
  --name day11-api \
  --network day11-app-net \
  --network-alias api-service \
  nginx:alpine
```

Test the alias from the client:

``` bash
docker exec day11-client curl -I http://api-service
```

The container name is `day11-api`; its additional network alias is
`api-service`. If multiple containers share an alias, DNS may return
multiple addresses. An alias is not itself a load balancer.

## 7. Connect and disconnect a running container

Create a second network:

``` bash
docker network create day11-observer-net
```

Connect the running client:

``` bash
docker network connect day11-observer-net day11-client
```

Inspect its network membership:

``` bash
docker inspect -f '{{json .NetworkSettings.Networks}}' day11-client
```

Disconnect it:

``` bash
docker network disconnect day11-observer-net day11-client
```

Inspect again to confirm the change.

## 8. Hands-on experiments

Complete these in order. Use the names shown so the cleanup commands
work as written.

### Experiment 1 --- Verify container-name DNS and HTTP

``` bash
docker network create day11-lab-net

docker run -d \
  --name day11-web \
  --network day11-lab-net \
  nginx:alpine

docker run -dit \
  --name day11-client \
  --network day11-lab-net \
  alpine

docker exec day11-client sh -c "apk add --no-cache curl"
docker exec day11-client curl -I http://day11-web
```

Record the HTTP status line and explain which name the client used to
reach the server.

### Experiment 2 --- Inspect network membership

``` bash
docker inspect -f '{{json .NetworkSettings.Networks}}' day11-web
docker inspect -f '{{json .NetworkSettings.Networks}}' day11-client
docker network inspect day11-lab-net
docker exec day11-client cat /etc/resolv.conf
```

Explain why both containers being members of the same user-defined
bridge matters.

### Experiment 3 --- Test network isolation

Create a separate network and a client attached only to it:

``` bash
docker network create day11-isolated-net

docker run -dit \
  --name day11-isolated-client \
  --network day11-isolated-net \
  alpine

docker exec day11-isolated-client sh -c "apk add --no-cache curl"
```

Try to reach the web container by name:

``` bash
docker exec day11-isolated-client curl --connect-timeout 3 -I http://day11-web
```

This should not work through the isolated network because the client is
not attached to the web container's network. Record the actual error.

Now connect the client to the shared network:

``` bash
docker network connect day11-lab-net day11-isolated-client
```

Retry:

``` bash
docker exec day11-isolated-client curl --connect-timeout 3 -I http://day11-web
```

Explain what changed: the client gained membership in a network shared
with the web container.

### Experiment 4 --- Test a network alias

``` bash
docker run -d \
  --name day11-api \
  --network day11-lab-net \
  --network-alias api-service \
  nginx:alpine

docker exec day11-client curl -I http://api-service
docker network inspect day11-lab-net
```

Explain the difference between the container name and the network alias.

## 9. Troubleshooting

**Name resolution fails:** Confirm both containers are attached to the
same user-defined network using `docker network inspect`.

**DNS works but HTTP fails:** Name resolution does not prove that the
application is listening. Check the container state and logs:

``` bash
docker ps
docker logs day11-web
```

**`curl` or `ping` is missing:** Minimal images omit many utilities.
Install a tool in a disposable lab container or use Docker metadata. A
missing executable is not evidence of a broken network.

**A container cannot reach another network:** Inspect network
membership. Connect it to the intended network only if that access is
part of your design.

**A fixed IP stops working:** Container addresses can change. Prefer
container names or network aliases for service discovery.

## 10. Key terminology

-   **Embedded DNS:** Docker-provided name resolution for containers on
    user-defined networks.
-   **Service discovery:** Finding a service endpoint through a name or
    registration mechanism.
-   **Network alias:** An additional network-scoped name for a
    container.
-   **Network membership:** The Docker networks to which a container is
    connected.
-   **Endpoint:** A container's connection to a Docker network.
-   **Network isolation:** Restricting connectivity by separating
    network membership.
-   **Port publishing:** Mapping a host port to a container port.

## 11. Completion checklist

-   [ ] I created a user-defined bridge network.
-   [ ] I connected two containers to the same network.
-   [ ] I reached Nginx using its container name.
-   [ ] I inspected network membership and resolver configuration.
-   [ ] I tested access from a separate network.
-   [ ] I connected a running container to an additional network.
-   [ ] I tested a network alias.
-   [ ] I understand why application configuration should avoid
    hard-coded container IPs.
-   [ ] I cleaned up all lab resources.

## References

-   [Docker networking
    overview](https://docs.docker.com/engine/network/)
-   [Bridge network
    driver](https://docs.docker.com/engine/network/drivers/bridge/)
-   [Docker networking and
    DNS](https://docs.docker.com/engine/network/#dns-services)
-   [`docker network connect`](https://docs.docker.com/reference/cli/docker/network/connect/)
-   [`docker network disconnect`](https://docs.docker.com/reference/cli/docker/network/disconnect/)

------------------------------------------------------------------------

## Day 11 practical task

Start with **Experiment 1 only**. Run the commands to create
`day11-lab-net`, `day11-web`, and `day11-client`, then test:

``` bash
docker exec day11-client sh -c "apk add --no-cache curl"
docker exec day11-client curl -I http://day11-web
```

Share the exact output. We will verify name resolution and HTTP
connectivity before moving to network isolation.

When you finish all experiments, remove the lab resources:

``` bash
docker rm -f day11-web day11-client day11-api day11-isolated-client
docker network rm day11-lab-net day11-isolated-net day11-app-net day11-observer-net
```

If you created only the resources in Experiment 1, remove just those:

``` bash
docker rm -f day11-web day11-client
docker network rm day11-lab-net
```
