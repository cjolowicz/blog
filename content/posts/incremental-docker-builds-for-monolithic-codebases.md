+++ 
date = 2019-05-23T09:08:59+02:00
title = "Incremental Docker builds for monolithic codebases"
description = "This post demonstrates how to use Docker to incrementally build and deploy multiple artifacts from large monolithic codebases."
tags = ["docker"]
+++

This post demonstrates how to use [Docker](https://www.docker.com/) to
incrementally build and deploy multiple artifacts from large monolithic
codebases. The technique is useful for developers who need to run frequent
integration tests on their work in progress.

**The goal**: 

* Build and deploy multiple Docker images from a monolithic codebase.
* The solution must work on a developer machine using [Docker
  Compose](https://docs.docker.com/compose/).
* The solution must scale to large codebases and allow frequent rebuilds.

**The challenges**:

1. How to avoid building the entire source tree each time? 
2. How to keep source and build trees out of the images?
3. How to use Docker Compose to build a shared builder image?

The code is available on GitHub:

* [cjolowicz/docker-incremental-build-example](https://github.com/cjolowicz/docker-incremental-build-example)

## Example codebase

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/6c6da9e)**

The example codebase consists of a static library `foo` and two executables
`bar` and `baz`, written in C. The project is built using
[CMake](https://cmake.org/) and produces a Debian package for each executable.

```
.
├── CMakeLists.txt
├── bar
│   ├── CMakeLists.txt
│   └── bar.c
├── baz
│   ├── CMakeLists.txt
│   └── baz.c
└── foo
    ├── CMakeLists.txt
    ├── foo.c
    └── foo.h

3 directories, 8 files
```

## Writing Dockerfiles

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/d7b93f1)**

The Dockerfiles for the two executables are almost identical: They install the
build requirements, copy the source tree, and build the binaries and packages.
At the end, each Dockerfile installs the package containing its executable, and
sets it as the command to be executed when running the image.

```Dockerfile
FROM debian:stretch-slim
RUN apt-get update && apt-get install -y \
    cmake \
    dpkg-dev \
    gcc \
    make \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /build
RUN cmake /src
RUN make package
RUN dpkg --install foobar-0.1.1-Linux-bar.deb
CMD ["bar"]
```

The `docker-compose.yml` file allows building and deploying the images with a
single command.

```yaml
version: "3.7"
services:
  bar:
    build:
      context: .
      dockerfile: bar/Dockerfile
  baz:
    build:
      context: .
      dockerfile: baz/Dockerfile
```

Test this solution using the following commands:

```sh
docker-compose up --build --detach
docker-compose logs --tail=10
docker-compose down
```

There are several problems with this approach:

1. Each Dockerfile contains the code to build the source tree.
2. Each Docker image contains the entire source and build trees.
3. The entire tree is built every time a line of code is changed.

## Avoiding code duplication

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/5e66dec)**

Code duplication is easily avoided by moving the build instructions to a
separate Dockerfile.

```Dockerfile
FROM debian:stretch-slim
RUN apt-get update && apt-get install -y \
    cmake \
    dpkg-dev \
    gcc \
    make \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /build
RUN cmake /src
RUN make package
```

This is used to build a common base image named `builder`, which can be
referenced by the other Dockerfiles.

```Dockerfile
FROM builder
RUN dpkg --install foobar-0.1.1-Linux-bar.deb
CMD ["bar"]
```

This introduces another problem, however: Docker Compose does not know about the
builder image. Before you can invoke `docker-compose`, the builder image needs
to be built explicitly using a command like the following:

```sh
docker build --tag=builder .
```

## Using Docker Compose for a builder image

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/1cfa80e)**

Getting Docker Compose to build the base image is only possible by adding a
service for it to the Docker Compose file.

```yaml
version: "3.7"
services:
  builder:
    image: builder
    build: .
  bar:
    build: bar
  baz:
    build: baz
```

Ensure that the builder service exits immediately by changing its command to
`true`.

```diff
diff --git a/Dockerfile b/Dockerfile
index 2202b3f..70a4be8 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -9,3 +9,4 @@ COPY . /src
 WORKDIR /build
 RUN cmake /src
 RUN make package
+CMD ["true"]
```

## Reducing image size

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/ca1c97a)**

Each image still contains the entire codebase, all the build artifacts produced
from it, and the complete build toolchain.

```sh
$ docker image ls --format="table {{.Repository}}\t{{.Size}}"
REPOSITORY                             SIZE
docker-incremental-build-example_baz   317MB
docker-incremental-build-example_bar   317MB 
builder                                317MB
```

Instead of deriving from the builder image, extract only the required package
from it using the `COPY --from` instruction. Images can now derive from a
minimal base image, leaving source and build trees as well as build dependencies
behind.

```Dockerfile
FROM debian:stretch-slim
COPY --from=builder /build/foobar-0.1.1-Linux-bar.deb /tmp/
RUN dpkg --install /tmp/foobar-0.1.1-Linux-bar.deb
CMD ["bar"]
```

This reduces image sizes significantly.

```sh
$ docker image ls --format="table {{.Repository}}\t{{.Size}}"
REPOSITORY                             SIZE
docker-incremental-build-example_baz   55.4MB
docker-incremental-build-example_bar   55.4MB
builder                                317MB
```

This technique is related to [multi-stage
builds](https://docs.docker.com/develop/develop-images/multistage-build/). Due
to the monolithic nature of the codebase, the first stage (the build stage) is
shared between images and defined in a separate Dockerfile.

## Building incrementally

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/277915f)**

Anytime a line of source code is changed, Docker needs to rebuild the entire
codebase.

Copy the build tree from a previous Docker build, using the `COPY --from`
instruction. Now only targets with changed dependencies are rebuilt. For large
codebases, this can speed up the Docker builds dramatically.

```diff
diff --git a/Dockerfile b/Dockerfile
index 70a4be8..82469c0 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -6,6 +6,7 @@ RUN apt-get update && apt-get install -y \
     make \
     && rm -rf /var/lib/apt/lists/*
 COPY . /src
+COPY --from=builder /build /build
 WORKDIR /build
 RUN cmake /src
 RUN make package
```

##### Bootstrapping incremental builds

The self-referentiality this introduces to the Dockerfile means that the initial
Docker build is broken: there is no image to copy the build tree from.

Build an initial builder image from which the "real" Docker build can copy the
build directory. Create this image using the following `Dockerfile.init`. The
build directory contains only an empty placeholder file.

```Dockerfile
FROM scratch
COPY .keep /build/
```

For convenience, create this `docker-compose.init.yaml` to trigger the initial
build.

```yaml
version: "3.7"
services:
  builder:
    image: builder
    build:
      context: .
      dockerfile: Dockerfile.init
```

Bootstrap the Docker builds using the following command:

```
docker-compose --file=docker-compose.init.yml build
```

## Summary

The technique outlined here allows to build and deploy artifacts from large and
monolithic codebases.

* Use a single command to build and deploy Docker images.
* Docker builds are fast due to the use of incremental builds.
* Image sizes are small due to the use of multi-stage builds.

At the heart of this technique is the `COPY --from` instruction. It is used for
two purposes:

* Copy the build tree from one builder image to the next.
* Copy the build artifacts into the final images.

```Dockerfile
# Copy the build tree from one builder image to the next.
COPY --from=builder /build /build

# Copy the build artifact from builder image to final image.
COPY --from=builder /build/foobar-0.1.1-Linux-bar.deb /tmp/
```

Builds are bootstrapped using the following command:

```
docker-compose --file=docker-compose.init.yml build
```

From then on, images can be built and deployed using the command:

```
docker-compose up --build
```
