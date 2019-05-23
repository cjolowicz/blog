--- 
date: 2019-05-23T09:08:59+02:00
title: "Incremental Docker builds for monolithic codebases"
description: "This post demonstrates how to use Docker to incrementally build and deploy multiple artifacts from large monolithic codebases."
tags:
  - docker
  - cmake
---

This post demonstrates how to use Docker to incrementally build and deploy
multiple artifacts from a large monolithic codebase during development. The
technique is useful for developers who need to run frequent integration tests on
their work in progress. Sample code is available in a [GitHub
repository](https://github.com/cjolowicz/docker-incremental-build-example).

##### Overview

1. [Introduction](#introduction)
1. [Example codebase](#example-codebase)
2. [Writing Dockerfiles](#writing-dockerfiles)
3. [Avoiding code duplication](#avoiding-code-duplication)
4. [Using Docker Compose for a builder image](#using-docker-compose-for-a-builder-image)
5. [Reducing image size](#reducing-image-size)
6. [Building incrementally](#building-incrementally)
7. [Summary](#summary)

## Introduction

Companies and open-source organizations often use a single source code
repository for their projects. These large monolithic codebases are also known
as [monorepos](https://en.wikipedia.org/wiki/Monorepo). As an extreme example,
Google's monorepo spanned 2 billion lines of code in early 2015, amounting to 85
terabytes of data.[^1]

[^1]: [Why Google Stores Billions of Lines of Code in a Single
    Repository](https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/fulltext),
    by Rachel Potvin and Josh Levenberg, in: Communications of the ACM, July
    2016, Vol. 59 No. 7, Pages 78-87.

How do you test changes that potentially affect a multitude of services built
from it? While unit tests are the primary means of checking code changes during
development, it is important to also run integration tests early on. They help
making sure that services start up and interact with each other as expected.

[Docker](https://www.docker.com/) is a convenient way of automating the build
and deployment process. What's more, it is not restricted to continuous
integration and production. Docker can also be used on a developer machine,
allowing you to test the running environment even before you push your changes
to a branch for CI and review.

Using Docker to build and deploy artifacts from a monolithic codebase faces
several challenges. This is especially true if you're a developer who needs to
frequently rebuild the codebase.

##### The goal

▹&nbsp;&nbsp;&nbsp;*Build and deploy artifacts from a monolithic repository during development*

* The solution must scale to large codebases.
* Images are built on, and deployed to, a developer machine.
* Builds and deployments can be repeated frequently.

##### Challenges

▹&nbsp;&nbsp;&nbsp;*How to avoid building the entire source tree each time?*

Every time a line of code is changed, Docker's build cache is invalidated,
causing it to rebuild the entire codebase from scratch. Typically this takes
anywhere from minutes to hours depending on codebase and build infrastructure.
How can you get Docker to build incrementally, reusing the intermediate build
artifacts from its last invocation?

▹&nbsp;&nbsp;&nbsp;*How to keep source and build trees out of the images?*

Another problem you encounter when writing Dockerfiles for a monolithic codebase
is the size of the Docker images resulting from it. The intermediate images
contain the entire source and build trees, including a heap of intermediate and
unrelated build artifacts, as well as the complete build toolchain.

[Multi-stage
builds](https://docs.docker.com/develop/develop-images/multistage-build/) are
commonly used to keep build dependencies out of the final Docker image: The
first stage imports the source tree, installs the build toolchain, and produces
the build artifact. The second stage extracts the build artifact and copies it
into a minimal base image. But how do you use multi-stage builds when the
initial build stage must be shared between all the images?

▹&nbsp;&nbsp;&nbsp;*How to use Docker Compose to build a shared intermediate image?*

Instructions to build and deploy a great number of services can easily become
quite complex. While Dockerfiles already help greatly with this, you can use
[Docker Compose](https://docs.docker.com/compose/) to encapsulate the entire
build and deployment process in a single declarative file. However, Docker
Compose assumes that only a single Dockerfile needs to be built for every
service. How can you get it to build a shared intermediate image?

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
single command:

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
separate Dockerfile:

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

The Dockerfile is used to build a common base image named `builder`, which can
be referenced by the other Dockerfiles like this:

```Dockerfile
FROM builder
RUN dpkg --install foobar-0.1.1-Linux-bar.deb
CMD ["bar"]
```

Docker Compose does not yet know about the builder image. Before the services
can be created and started, the builder image needs to be built explicitly using
a command like the following:

```sh
docker build --tag=builder .
```

## Using Docker Compose for a builder image

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/1cfa80e)**

Getting Docker Compose to build the base image is only possible by adding a
service for it to the Docker Compose file:

{{< highlight yaml "hl_lines=3-5" >}}
version: "3.7"
services:
  builder:
    image: builder
    build: .
  bar:
    build: bar
  baz:
    build: baz
{{< /highlight >}}

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
from it, and the complete build toolchain:

```sh
$ docker image ls --format="table {{.Repository}}\t{{.Size}}"
REPOSITORY                             SIZE
docker-incremental-build-example_baz   317MB
docker-incremental-build-example_bar   317MB 
builder                                317MB
```

Instead of deriving from the builder image, extract the required package from
the builder image using the `COPY --from` instruction, and copy it into a
minimal base image:

```Dockerfile
FROM debian:stretch-slim
COPY --from=builder /build/foobar-0.1.1-Linux-bar.deb /tmp/
RUN dpkg --install /tmp/foobar-0.1.1-Linux-bar.deb
CMD ["bar"]
```

Images now derive from a minimal base image, leaving source and build trees as
well as build dependencies behind. This reduces image sizes significantly, even
for our tiny example codebase:

```sh
$ docker image ls --format="table {{.Repository}}\t{{.Size}}"
REPOSITORY                             SIZE
docker-incremental-build-example_baz   55.4MB
docker-incremental-build-example_bar   55.4MB
builder                                317MB
```

This technique is related to [multi-stage
builds](https://docs.docker.com/develop/develop-images/multistage-build/). Due
to the monolithic nature of the codebase, the first stage---the build stage---is
shared between images and defined in a separate Dockerfile.

## Building incrementally

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/277915f)**

Anytime a line of source code is changed, Docker needs to rebuild the entire
codebase.

Copy the build tree from a previous Docker build, using the `COPY --from`
instruction. Now only targets with changed dependencies are rebuilt. For a
sizeable codebase, this can speed up Docker builds dramatically.

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

The `COPY --from=builder` instruction introduces a self-referentiality to the
Dockerfile for the builder image. The consequence of this self-referentiality is
that the initial build is now broken: there is no image to copy the build tree
from.

The following `Dockerfile.init` creates an initial builder image from which the
"real" builder image can copy the build directory. The build directory contains
only an empty placeholder file.

```Dockerfile
FROM scratch
COPY .keep /build/
```

For convenience, create this `docker-compose.init.yaml` to trigger the initial
build:

```yaml
version: "3.7"
services:
  builder:
    image: builder
    build:
      context: .
      dockerfile: Dockerfile.init
```

Docker builds can now be bootstrapped using the following command:

```sh
docker-compose --file=docker-compose.init.yml build
```

## Summary

The technique outlined here allows to build and deploy artifacts from large and
monolithic codebases.

* A single command builds and deploys Docker images.
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

```sh
docker-compose --file=docker-compose.init.yml build
```

From then on, images can be built and deployed using the command:

```sh
docker-compose up --build
```
