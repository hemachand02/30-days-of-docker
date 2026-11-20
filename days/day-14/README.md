# Day 14 --- Building Docker Images and Optimizing Image Layers

## Learning objectives

By the end of this lesson, you should be able to:

-   Build a Docker image from a Dockerfile using `docker build`.
-   Explain the build context and why it should be kept small.
-   Understand image layers and build-cache reuse.
-   Use `.dockerignore` to exclude unnecessary files.
-   Inspect image history and size.
-   Explain why Dockerfile instruction ordering affects build
    efficiency.

## 1. Why image-building practices matter

A Dockerfile describes how an image should be built. The `docker build`
command sends a build context to the builder, processes the Dockerfile
instructions, and produces an image.

A good image should be small, repeatable, efficient to build, and easy
to understand. Image size and build speed affect storage, registry
transfer time, deployment time, and the amount of software that must be
maintained.

## 2. Build context

The **build context** is the set of files the builder can access during
a build. It is commonly the directory passed to `docker build`.

``` bash
docker build -t my-app:1.0 .
```

The final `.` means "use the current directory as the build context."

An instruction such as `COPY . /app` can copy files from that context
into the image. It cannot normally copy arbitrary files from outside the
context.

Keep the context small. Avoid sending Git history, local dependencies,
temporary files, logs, and secrets to the builder.

## 3. Image layers

Many Dockerfile instructions create filesystem layers. Layers can be
cached and reused by later builds when the relevant instruction and its
inputs have not changed.

Example:

``` dockerfile
FROM alpine:3.21
WORKDIR /app
COPY message.txt .
CMD ["cat", "message.txt"]
```

An image is an immutable template made from filesystem layers and image
configuration. A container is a runnable instance of an image with its
own writable layer. A build cache stores reusable results from previous
build steps; it is not the same thing as a running container.

## 4. Build cache and instruction ordering

Docker can reuse a cached result when a build step and the inputs it
depends on have not changed. If an early instruction changes, later
steps may need to be rebuilt.

For applications with dependencies, a common pattern is:

1.  Copy dependency definition files.
2.  Install dependencies.
3.  Copy frequently changing application source files.
4.  Define the runtime command.

This can keep dependency installation cached when only application
source code changes. The exact pattern depends on the language and
package manager.

## 5. `.dockerignore`

A `.dockerignore` file tells Docker which paths to exclude from the
build context.

``` dockerignore
.git
.gitignore
*.log
tmp/
.env
node_modules/
```

Only add entries that make sense for your project. Excluding `.env` is a
useful safeguard, but production secrets should be supplied at runtime
or through an appropriate secrets mechanism. Never copy secrets into an
image.

## 6. Useful commands

``` bash
docker build -t day14-app:1.0 .
docker image ls
docker image inspect day14-app:1.0
docker image history day14-app:1.0
docker build --no-cache -t day14-app:debug .
```

Use `--no-cache` for comparison or troubleshooting, not as the default
for every build.

------------------------------------------------------------------------

# Hands-on experiments

Complete these in order. Start with Experiment 1 and verify its output
before moving on.

## Experiment 1 --- Build your first image and inspect its history

Create a working directory:

``` bash
mkdir -p ~/docker-day14/experiment-1
cd ~/docker-day14/experiment-1
```

Create `message.txt`:

``` text
Hello from my Day 14 Docker image!
```

Create a file named `Dockerfile` (capital D, no file extension):

``` dockerfile
FROM alpine:3.21
WORKDIR /app
COPY message.txt .
CMD ["cat", "message.txt"]
```

Build and run the image:

``` bash
docker build -t day14-app:1.0 .
docker run --rm day14-app:1.0
```

Inspect its history:

``` bash
docker image history day14-app:1.0
```

**Observe:**

-   What does the `.` at the end of the build command represent?
-   Which instruction selects the base image?
-   Which instruction copies `message.txt` into the image?
-   What does `CMD` specify?
-   What information does `docker image history` show?

Do not move on until the image builds and prints the expected message.

## Experiment 2 --- See the effect of `.dockerignore`

From the same directory, create files that should not be included in the
build context:

``` bash
mkdir -p tmp
echo "temporary data" > tmp/debug.txt
echo "sample log" > debug.log
```

Create `.dockerignore`:

``` dockerignore
tmp/
*.log
.git
```

Build again:

``` bash
docker build -t day14-app:1.1 .
```

Review the build output. The displayed context size can vary by builder
and version, so focus on whether ignored files are excluded rather than
expecting an exact byte count.

**Observe:**

-   Which paths are excluded?
-   Why is excluding unnecessary files useful?
-   Why should credentials and secret files never be copied into an
    image?

## Experiment 3 --- Observe build-cache reuse

Build the image twice without changing files:

``` bash
docker build -t day14-cache:1.0 .
docker build -t day14-cache:1.0 .
```

Compare the output. Depending on the builder, cached steps may be marked
`CACHED` or shown with equivalent cache messages.

Now change only `message.txt`:

``` bash
echo "The message has changed." > message.txt
docker build -t day14-cache:1.0 .
```

**Observe:**

-   Which steps were reused on the unchanged build?
-   Which step needed new work after `message.txt` changed?
-   Why can stable instructions placed before frequently changing files
    improve build efficiency?

## Experiment 4 --- Compare image history and size

``` bash
docker image ls day14-app
docker image history day14-app:1.0
docker image inspect day14-app:1.0
```

Record the image size and note the history entries.

**Observe:**

-   What image size is shown by `docker image ls`?
-   What does the history reveal about the build?
-   Why should you avoid installing tools that the application does not
    need at runtime?

------------------------------------------------------------------------

## Troubleshooting

### `Dockerfile not found`

Check that the file is named exactly `Dockerfile` and that you are in
the intended directory:

``` bash
pwd
ls -la
docker build -t day14-app:1.0 .
```

### `COPY failed: file not found`

The source file must exist inside the build context and must not be
excluded by `.dockerignore`.

``` bash
ls -la
cat .dockerignore
```

### Docker daemon is unavailable

Make sure Docker Desktop is running, then check:

``` bash
docker version
docker info
docker context ls
```

On this Windows + WSL setup, confirm that the active Docker context
points to the Docker Desktop engine you intend to use.

### Build appears to reuse old output

Docker may be using its build cache. For a diagnostic rebuild:

``` bash
docker build --no-cache -t day14-app:debug .
```

------------------------------------------------------------------------

## Key terminology

  Term              Meaning
  ----------------- -----------------------------------------------------
  Build context     Files made available to the builder for a build
  Dockerfile        Instructions used to build an image
  Image layer       A filesystem change represented in an image
  Build cache       Reusable results from previous build steps
  `.dockerignore`   Rules for excluding paths from the build context
  Base image        Starting image selected by `FROM`
  Image history     Metadata describing image build layers and commands

## Completion checklist

-   [ ] I built an image using `docker build`.
-   [ ] I ran a container from my own image.
-   [ ] I inspected the image using `docker image history`.
-   [ ] I understand what the build context is.
-   [ ] I created and tested a `.dockerignore` file.
-   [ ] I observed cache reuse across repeated builds.
-   [ ] I can explain why Dockerfile instruction ordering matters.

## Cleanup

After recording your observations, remove the images created for this
lesson if you no longer need them:

``` bash
docker image rm day14-app:1.0 day14-app:1.1 day14-cache:1.0
```

If Docker reports that an image is being used by a container, remove
that container first. The `--rm` option used in Experiment 1
automatically removes its container when it exits.

## Official documentation

-   [Docker build](https://docs.docker.com/build/)
-   [Build context](https://docs.docker.com/build/concepts/context/)
-   [Dockerfile
    reference](https://docs.docker.com/reference/dockerfile/)
-   [Build cache
    optimization](https://docs.docker.com/build/cache/optimize/)
-   [Docker image
    history](https://docs.docker.com/reference/cli/docker/image/history/)

------------------------------------------------------------------------

## Day 14 --- First task

Run **Experiment 1 only**. Build `day14-app:1.0`, run it, and inspect
its history.

Share the output of:

``` bash
docker build -t day14-app:1.0 .
docker run --rm day14-app:1.0
docker image history day14-app:1.0
```

We will verify the result and discuss what each build step did before
proceeding to Experiment 2.
