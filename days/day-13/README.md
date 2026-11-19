# Day 13 --- Dockerfile Fundamentals

## Learning objectives

By the end of this day, you should be able to: - Explain what a
Dockerfile is and why it is used. - Understand `FROM`, `WORKDIR`,
`COPY`, `RUN`, `CMD`, `ENTRYPOINT`, and `EXPOSE`. - Build an image using
`docker build`. - Explain build context and `.dockerignore`. - Inspect
an image and run a container from it.

## 1. Why do we need a Dockerfile?

An application needs its source code, runtime, dependencies, filesystem
layout, and startup instructions. Manually preparing a container every
time is difficult to reproduce.

A **Dockerfile** is a text file containing instructions used to build a
Docker image.

Keep these three concepts distinct: - **Dockerfile:** instructions for
building an image. - **Image:** packaged, read-only template produced by
the build. - **Container:** a running or stopped instance created from
an image.

## 2. A first Dockerfile

``` dockerfile
FROM alpine:3.21

WORKDIR /app

COPY hello.sh .

RUN chmod +x hello.sh

CMD ["./hello.sh"]
```

This starts from Alpine, sets `/app` as the working directory, copies in
a script, makes it executable, and configures it as the default startup
command.

## 3. Core Dockerfile instructions

  -----------------------------------------------------------------------
  Instruction                         Purpose
  ----------------------------------- -----------------------------------
  `FROM`                              Selects the base image

  `WORKDIR`                           Sets the working directory for
                                      subsequent instructions

  `COPY`                              Copies files from the build context
                                      into the image

  `RUN`                               Executes a command during image
                                      build

  `CMD`                               Defines the default command or
                                      arguments at container startup

  `ENTRYPOINT`                        Defines the main executable for the
                                      container

  `EXPOSE`                            Documents the port the application
                                      is intended to listen on

  `ENV`                               Sets environment variables in the
                                      image

  `USER`                              Sets the user for subsequent build
                                      instructions and runtime

  `LABEL`                             Adds metadata to the image
  -----------------------------------------------------------------------

### `FROM`

``` dockerfile
FROM alpine:3.21
```

Selects the base image. It provides the initial filesystem and
environment for the new image.

### `WORKDIR`

``` dockerfile
WORKDIR /app
```

Sets the working directory for later instructions such as `RUN`, `COPY`,
and `CMD`. Docker creates the directory if it does not exist.

Using `WORKDIR` is clearer than relying on `RUN cd ...`; a `cd` in one
`RUN` instruction does not change the working directory of a later
instruction.

### `COPY`

``` dockerfile
COPY hello.sh .
```

Copies `hello.sh` from the build context into the current working
directory (`/app`). The source path is relative to the build context.

### `RUN`

``` dockerfile
RUN apk add --no-cache curl
```

Executes a command during image building. Its filesystem changes become
part of the resulting image.

### `CMD`

``` dockerfile
CMD ["./hello.sh"]
```

Defines the default command or arguments used when a container starts. A
command supplied to `docker run` normally replaces the image's default
`CMD`.

The JSON-array form is called **exec form** and is generally preferred
for predictable process and signal handling.

### `ENTRYPOINT` and `CMD`

-   `ENTRYPOINT` defines the main executable.
-   `CMD` supplies default arguments to that executable, or a default
    command if no entrypoint is set.

They can be combined. We will examine their startup behavior in a later
practical exercise.

### `EXPOSE`

``` dockerfile
EXPOSE 8080
```

Documents the port on which the application is expected to listen inside
the container. It does **not** publish the port to the host.

To publish a port, use `docker run -p`, for example:

``` bash
docker run -p 8080:8080 my-app
```

## 4. Build context

When you run:

``` bash
docker build -t my-app:1.0 .
```

The final `.` specifies the **build context**: the set of files made
available to the builder for instructions such as `COPY` and `ADD`.

-   `docker build` starts the build.
-   `-t my-app:1.0` assigns an image name and tag.
-   `.` selects the current directory as the build context.

The Dockerfile is normally named `Dockerfile` and placed in the context
directory. You can specify another Dockerfile with `-f`:

``` bash
docker build -f docker/Dockerfile -t my-app:1.0 .
```

The build context remains `.` in this example. The `-f` option selects
the Dockerfile; it does not change the context.

A `COPY` source must be available within the build context. Files
outside it cannot be copied into the image.

## 5. `.dockerignore`

A `.dockerignore` file excludes files and directories from the build
context.

Example:

``` dockerignore
.git
*.log
node_modules
.env
```

Why use it? - Exclude unnecessary files and reduce context transfer. -
Avoid including local development artifacts. - Reduce the risk of
accidentally including secrets.

Do not copy secrets into images. Excluding `.env` is useful, but
production secrets should be supplied through an appropriate runtime
configuration or secret mechanism.

## 6. First understanding of image layers and cache

A Dockerfile describes a sequence of build instructions. Filesystem
changes are represented through image layers, and Docker can reuse
cached build results when the relevant instruction and inputs have not
changed.

For example:

``` dockerfile
FROM alpine:3.21
RUN apk add --no-cache curl
WORKDIR /app
COPY hello.sh .
CMD ["./hello.sh"]
```

Changing an earlier instruction or its inputs can invalidate cache for
later build steps. We will study layer ordering and cache optimization
in more depth later.

------------------------------------------------------------------------

# Hands-on Labs

Use Ubuntu on WSL with Docker Desktop running. First verify that the
Docker CLI can reach the daemon.

## Experiment 1 --- Build your first custom image

**Goal:** Create a script, describe its environment in a Dockerfile, and
build an image.

### Step 1: Create a project directory

``` bash
mkdir -p ~/docker-day13
cd ~/docker-day13
```

### Step 2: Create the application script

``` bash
cat > hello.sh <<'EOF'
#!/bin/sh
echo "Hello from my first custom Docker image!"
echo "Container hostname: $(hostname)"
EOF
```

### Step 3: Create the Dockerfile

``` bash
cat > Dockerfile <<'EOF'
FROM alpine:3.21

WORKDIR /app

COPY hello.sh .

RUN chmod +x hello.sh

CMD ["./hello.sh"]
EOF
```

### Step 4: Build the image

``` bash
docker build -t day13-hello:1.0 .
```

Observe the build output and identify the steps corresponding to the
Dockerfile instructions.

### Step 5: Confirm the image exists

``` bash
docker images day13-hello
```

### Step 6: Run a container

``` bash
docker run --name day13-hello-container day13-hello:1.0
```

Observe the output.

### Step 7: Inspect image configuration

``` bash
docker image inspect day13-hello:1.0
```

Inspect selected fields:

``` bash
docker image inspect -f '{{.Config.WorkingDir}}' day13-hello:1.0
docker image inspect -f '{{json .Config.Cmd}}' day13-hello:1.0
```

### Step 8: Clean up

``` bash
docker rm day13-hello-container
docker rmi day13-hello:1.0
```

Keep the project directory for the next experiments.

### Record your observations

1.  Did the image build successfully?
2.  What output did the container print?
3.  What working directory was configured?
4.  What default command was configured?
5.  Which instruction copied the script into the image?

Do not mark this experiment complete until you have run it and checked
the output.

## Experiment 2 --- Understand build context and `.dockerignore`

**Goal:** Observe which files are available to a Docker build.

From `~/docker-day13`, create files:

``` bash
mkdir -p notes
echo "This file is part of the project" > notes/info.txt
echo "Temporary log data" > debug.log
```

Create `.dockerignore`:

``` bash
cat > .dockerignore <<'EOF'
*.log
.git
EOF
```

Temporarily add this instruction to your Dockerfile:

``` dockerfile
COPY . /context-check
```

Build a new image:

``` bash
docker build -t day13-context:1.0 .
```

Inspect the copied files:

``` bash
docker run --rm day13-context:1.0 find /context-check -maxdepth 3 -type f
```

Check whether `debug.log` appears. It should be excluded by
`.dockerignore`.

**Important:** This experiment changes the Dockerfile. Restore the
original Dockerfile from Experiment 1 when finished, or keep a separate
copy.

## Experiment 3 --- Observe `CMD` behavior

**Goal:** Understand that a command supplied to `docker run` replaces
the default `CMD`.

Build the original Dockerfile image if needed:

``` bash
docker build -t day13-hello:1.0 .
```

Run it normally:

``` bash
docker run --rm day13-hello:1.0
```

Now supply another command:

``` bash
docker run --rm day13-hello:1.0 echo "This command replaced CMD"
```

Compare the outputs.

**Think about it:** Why did the script not run in the second command?

## Experiment 4 --- Understand `EXPOSE` versus port publishing

**Goal:** Verify that documenting a port does not publish it
automatically.

Create a separate lab directory:

``` bash
mkdir -p ~/docker-day13/nginx-lab
cd ~/docker-day13/nginx-lab
```

Create a Dockerfile:

``` dockerfile
FROM nginx:alpine
EXPOSE 80
```

Build and run:

``` bash
docker build -t day13-nginx:1.0 .
docker run -d --name day13-nginx day13-nginx:1.0
docker ps
docker port day13-nginx
```

No host port mapping should be listed because you did not publish one.

Remove the container and run it with a published port:

``` bash
docker rm -f day13-nginx
docker run -d --name day13-nginx -p 8080:80 day13-nginx:1.0
```

Test from Ubuntu:

``` bash
curl -I http://localhost:8080
```

Clean up:

``` bash
docker rm -f day13-nginx
docker rmi day13-nginx:1.0
```

If host port `8080` is already in use, choose another host port and use
it consistently in the `-p` mapping and `curl` command.

------------------------------------------------------------------------

# Troubleshooting

### `docker build` cannot find the Dockerfile

-   Confirm that the file is named exactly `Dockerfile`.
-   Run the command from the intended directory.
-   Check with `pwd` and `ls -la`.
-   If using a different filename or location, specify it with `-f`.

### `COPY` failed or the source file was not found

-   Confirm that the file exists.
-   Remember that `COPY` source paths are relative to the build context.
-   Check whether `.dockerignore` excludes the file.
-   Make sure the build context is the intended directory.

### Docker daemon is unavailable

Start Docker Desktop, then check:

``` bash
docker context ls
docker info
```

Use the context connected to the Docker Desktop engine in your setup. Do
not assume the daemon is available when Docker Desktop is stopped.

### Port publishing fails

-   Check whether another process or container is using the host port.
-   Inspect mappings with `docker ps`.
-   Choose a different host port if needed.

------------------------------------------------------------------------

# Key terminology

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  Dockerfile                          Text file containing image build
                                      instructions

  Base image                          Starting image selected by `FROM`

  Build context                       Files made available to the image
                                      builder

  Build cache                         Reusable results from earlier build
                                      steps

  Image layer                         Filesystem change represented as
                                      part of an image

  `.dockerignore`                     File that excludes paths from the
                                      build context

  Build-time instruction              Instruction executed while
                                      building, such as `RUN`

  Runtime command                     Default command configured for
                                      startup, such as `CMD`

  Port publishing                     Mapping a host port to a container
                                      port using `-p`
  -----------------------------------------------------------------------

# Day 13 Completion Checklist

-   [ ] I understand why Dockerfiles are used.
-   [ ] I can explain the difference between an image and a container.
-   [ ] I understand `FROM`, `WORKDIR`, `COPY`, `RUN`, and `CMD`.
-   [ ] I understand the basic role of `ENTRYPOINT` and `EXPOSE`.
-   [ ] I can build an image using `docker build`.
-   [ ] I understand the build context.
-   [ ] I understand why `.dockerignore` is useful.
-   [ ] I can inspect an image and run a container from it.
-   [ ] I completed the experiments and recorded my actual observations.

# Official documentation

-   [Dockerfile
    reference](https://docs.docker.com/reference/dockerfile/)
-   [Build
    images](https://docs.docker.com/get-started/docker-concepts/building-images/)
-   [Build context](https://docs.docker.com/build/concepts/context/)
-   [`.dockerignore`
    reference](https://docs.docker.com/build/concepts/context/#dockerignore-files)
-   [Build cache](https://docs.docker.com/build/cache/)

------------------------------------------------------------------------

## Today's task

Start with **Experiment 1 only**. Build `day13-hello:1.0`, run the
container, and share:

``` bash
docker images day13-hello
docker image inspect -f '{{.Config.WorkingDir}}' day13-hello:1.0
docker image inspect -f '{{json .Config.Cmd}}' day13-hello:1.0
```

Also share the output from:

``` bash
docker run --name day13-hello-container day13-hello:1.0
```

We will verify your results before moving to Experiment 2.
