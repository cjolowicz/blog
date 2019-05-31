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

The post first shows a naive approach to building an entire codebase with
Docker, and then an interesting and novel solution which avoids code
duplication, reduces image size, and speeds up builds by reusing the
intermediate build artifacts from a previous Docker run. Each section
corresponds to a commit in the GitHub repository, linked to at the top of each
section.

##### Contents

1. [Introduction](#introduction)
2. [Example codebase](#example-codebase)
3. [Writing Dockerfiles](#writing-dockerfiles)
4. [Avoiding code duplication](#avoiding-code-duplication)
5. [Using Docker Compose for a builder image](#using-docker-compose-for-a-builder-image)
6. [Reducing image size](#reducing-image-size)
7. [Building incrementally](#building-incrementally)
8. [Summary](#summary)

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
make sure that services start up and interact with each other as expected.

[Docker](https://www.docker.com/) is a convenient way of automating the build
and deployment process. What's more, it is not restricted to continuous
integration and production. Docker can also be used on a developer machine,
allowing you to test the running environment even before you push your changes
to a branch for CI and review.

Using Docker to build and deploy artifacts from a monolithic codebase presents
several challenges. This is especially true if you're a developer who needs to
frequently rebuild the codebase.

## Example codebase

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/6c6da9e)**

I will use a small example codebase to explain how to use Docker for a
monolithic repository. It consists of a static library `foo` and two executables
`bar` and `baz`. The build system produces a Debian package for each executable.

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

Don't worry if you're not familiar with the C programming language or the
[CMake](https://cmake.org/) build system. This article does not assume any
knowledge about these. The technique shown here applies to any programming
language and build system, and should be straightforward to translate to your
favorite weapon of choice.

## Writing Dockerfiles

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/d7b93f1)**

Let's start by writing a Dockerfile for each of the two executables.

The Dockerfiles are almost identical: They install the build requirements, copy
the source tree, and build the binaries and packages. At the end, each
Dockerfile installs the package containing its executable, and sets it as the
command to be executed when running the image.

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

Instructions to build and deploy many services become complex, very fast. While
Dockerfiles already help greatly with this, you can use [Docker
Compose](https://docs.docker.com/compose/) to encapsulate the entire build and
deployment process in a single declarative file.

Create the following `docker-compose.yml` file:

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

This allows you to build and deploy the images with a single invocation of
`docker-compose up`.

Test everything using the following commands:

```sh
docker-compose up --build --detach
docker-compose logs --tail=10
docker-compose down
```

It works, but this solution has several shortcomings:

1. Each Dockerfile contains identical commands to build the source tree.
2. Each Docker image contains the entire source and build trees.
3. The entire codebase is built every time a line of code is changed.

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

The builder image is not yet registered in the Docker Compose file. Before the
services can be created and started, the builder image needs to be built
explicitly using a command such as the following:

```sh
docker build --tag=builder .
```

## Using Docker Compose for a builder image

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/1cfa80e)**

Docker Compose only builds a single image for every service, but these images
are derived from a common base image. How do you ensure the base image is
rebuilt any time the services are?

Getting Docker Compose to build the base image is possible, but it involves
adding a service for it at the top of the Docker Compose file:

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

The image name is specified explicitly because, by default, image names are
constructed from the basename of the directory and the service name. By default,
the image would end up being named `docker-incremental-build-example_builder`,
rather than `builder`.

Of course, you do not actually want Docker Compose to run a service with the
builder image. Ensure that the builder service exits immediately by changing its
command to `true`:

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

One problem you encounter when writing Dockerfiles for a monolithic codebase is
the size of the Docker images resulting from it. Each image contains the entire
source and build trees, including a heap of unrelated build artifacts, as well
as the complete build toolchain. You can verify this for your example codebase:

```sh
$ docker image ls --format="table {{.Repository}}\t{{.Size}}"
REPOSITORY                             SIZE
docker-incremental-build-example_baz   317MB
docker-incremental-build-example_bar   317MB 
builder                                317MB
```

##### How to keep source and build trees out of the images

[Multi-stage
builds](https://docs.docker.com/develop/develop-images/multistage-build/) are
commonly used to keep build dependencies out of the final Docker image: The
first stage imports the source tree, installs the build toolchain, and produces
the build artifact. The second stage extracts the build artifact and copies it
into a minimal base image. This is achieved using the `COPY --from` instruction,
which allows copying files from another image or build stage.

A typical multi-stage build looks like this:

```Dockerfile
FROM debian
RUN apt-get update && apt-get install -y build-essential \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /build
COPY . .
RUN make foo

FROM alpine
COPY --from=0 /build/foo /usr/bin/
RUN ["foo"]
```

This Dockerfile has two stages, each introduced by a `FROM` instruction. In this
example, the second stage uses an Alpine image. [Alpine
Linux](https://alpinelinux.org/) is a security-oriented, lightweight Linux
distribution and a popular choice for Docker images. Docker build stages are
numbered from zero, so `COPY --from=0` references the first stage. It copies the
build artifact from the first stage and sets it as the command to be executed
when the image is run.

But with a monolithic codebase, the build instructions for the first stage are
identical for all images. How do you use multi-stage builds when the initial
stage is shared between the images? This is actually rather simple. The `COPY
--from` instruction can also be used with the name of an external image, rather
than a build stage. In your case, this needs to be an image that builds the
codebase and produces all the build artifacts: the builder image.

Let's rewrite the Dockerfiles for `bar` and `baz` using the `COPY --from`
instruction. Instead of deriving the final images from the builder image, derive
them from a minimal base image. Then extract the Debian package from the builder
image, and install it into the image:

```Dockerfile
FROM debian:stretch-slim
COPY --from=builder /build/foobar-0.1.1-Linux-bar.deb /tmp/
RUN dpkg --install /tmp/foobar-0.1.1-Linux-bar.deb
CMD ["bar"]
```

Images now contain only the minimum required to run the service, leaving source
and build trees as well as build dependencies behind. This reduces image sizes
significantly, even for your tiny example codebase:

```sh
$ docker image ls --format="table {{.Repository}}\t{{.Size}}"
REPOSITORY                             SIZE
docker-incremental-build-example_baz   55.4MB
docker-incremental-build-example_bar   55.4MB
builder                                317MB
```

This technique is related to multi-stage builds, but---due to the monolithic
nature of the codebase---the first stage is shared between images and contained
in a separate Dockerfile.

## Building incrementally

▶ **[View code](https://github.com/cjolowicz/docker-incremental-build-example/commit/277915f)**

Every time a line of code is changed, Docker rebuilds the entire codebase from
scratch. Typically, this takes anywhere from minutes to hours, depending on
codebase and build infrastructure. Let's see why this happens.

The Dockerfile for the builder image imports the source tree into the image
using the `COPY` instruction. When encountering a `COPY` instruction, Docker
examines the contents of the copied files and calculates a checksum for each
file. If the checksum of each of the files matches its checksum in a previous
build, the image is retrieved from the [Docker build
cache](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache).

If, on the other hand, one of the files has changed, the build cache is
invalidated and Docker runs the remaining instructions in the Dockerfile without
using the cache. As a consequence, the build tools---cmake and make---cannot
reuse any artifacts from a previous run. The layers containing them appear after
the `COPY` instruction and are thus no longer available.

##### How to avoid building the entire source tree each time

How can you get Docker to build incrementally, reusing the intermediate build
artifacts from its last invocation? Luckily, there is an instruction that allows
you to copy the build artifacts into the image explicitly: the `COPY --from`
instruction, which you just used in the section [Reducing image
size](#reducing-image-size). At the time the Dockerfile is being executed, the
`builder` tag still references the last successful build of this image, so you
can use it to copy the build tree.

Let's add this instruction right after we copy the source tree:

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

Now the build tools can access the results of previous runs, so only targets
with changed dependencies need to be rebuilt. For a sizeable codebase, this can
speed up Docker builds dramatically.

##### Bootstrapping incremental builds

The `COPY --from=builder` instruction references the very image that is
currently being built: the builder image. The consequence of this
self-referentiality is that the initial build is now broken: there is no image
to copy the build tree from.

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

The technique outlined here allows you to build and deploy artifacts from large
and monolithic codebases.

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

I hope you found this useful, and I'm happy to answer questions or to learn
about how you've implemented this method in your own work.
