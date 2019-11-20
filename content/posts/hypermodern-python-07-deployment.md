--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 7: Deployment"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - docker
---

In this appendix to the Hypermodern Python series, I'm going to discuss how
to deploy your project with Docker.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation)
- [Chapter 6: CI/CD](../hypermodern-python-06-ci-cd)
- [Appendix: Docker](../hypermodern-python-07-deployment)

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Building a Docker image](#building-a-docker-image)
- [The Dockerfile in detail](#the-dockerfile-in-detail)
- [Hosting images at Docker Hub](#hosting-images-at-docker-hub)

<!-- markdown-toc end -->

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Here is the link for the changes contained in this chapter:

▶ **[View code](https://github.com/cjolowicz/hypermodern-python/compare/chapter06...chapter07)**

## Building a Docker image

[Docker](https://www.docker.com/) enables you to build, share, and run your
application anywhere using *containers*. Containers are a lightweight
virtualization technology, packaging your application with its dependencies, and
allowing it to be executed in an isolated environment.

Download and install [Docker Desktop](https://docker.com/get-started). If on
Linux, download [Docker
Engine](https://hub.docker.com/search?type=edition&offering=community).

Add the following `Dockerfile` to the root of your project. (We will go through
the Dockerfile line by line in the next section.)

```Dockerfile
FROM python:3.8.0-alpine3.10 as base
FROM base as builder

WORKDIR /app
ENV POETRY_VERSION=1.0.0b8
RUN wget -qO- https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
RUN python -m venv /venv

COPY pyproject.toml poetry.lock ./
RUN /root/.poetry/bin/poetry export -f requirements.txt | /venv/bin/pip install -r /dev/stdin

COPY . ./
RUN /root/.poetry/bin/poetry build && /venv/bin/pip install dist/*.whl

FROM base as final

COPY --from=builder /venv /venv
ENV PYTHONUNBUFFERED=1
ENTRYPOINT ["/venv/bin/hypermodern-python"]
```

Build the Docker image using the following command:

```sh
docker build -t hypermodern-python .
```

Run your image like this:

```sh
docker run hypermodern-python
```

You can pass options to `hypermodern-python` by appending them to the end of the
command-line:

```sh
$ docker run hypermodern-python --count=4
Reticulating spline 1...
Reticulating spline 2...
Reticulating spline 3...
Reticulating spline 4...
```

## The Dockerfile in detail

The Dockerfile defines a so-called [multi-stage
build](https://docs.docker.com/develop/develop-images/multistage-build/), an
elegant way to reduce the size and complexity of your images:

- The *build stage* includes all the developer tooling needed to build your
  application (e.g. Poetry).
- The *final stage* is deliberately kept slim and contains only artifacts needed
  at runtime.

In the Python world, multi-stage builds are typically realized using virtual
environments. In a nutshell, the build stage installs everything into a virtual
environment, and the final stage simply copies the virtual environment over into
a small base image.

Let's dissect the Dockerfile bit by bit.

The image is based on the [official Python
image](https://hub.docker.com/_/python) for [Alpine](https://alpinelinux.org/),
a security-oriented, lightweight Linux distribution:

```Dockerfile
FROM python:3.8.0-alpine3.10 as base
```

The second `FROM` statement starts the build stage:

```Dockerfile
FROM base as builder
```

In the build stage, create the directory for your source code and make it the
working directory:

```Dockerfile
WORKDIR /app
```

Install Poetry:

```Dockerfile
ENV POETRY_VERSION=1.0.0b8
RUN wget -qO- https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
```

Create a virtual environment:

```Dockerfile
RUN python -m venv /venv
```

Copy the package configuration and lock file into the image:

```Dockerfile
COPY pyproject.toml poetry.lock ./
```

Use [poetry export](https://poetry.eustace.io/docs/cli/#export) to install your
pinned requirements:

```Dockerfile
RUN /root/.poetry/bin/poetry export -f requirements.txt | /venv/bin/pip install -r /dev/stdin
```

Installing the requirements before copying your code allows you to leverage the
[Docker build
cache](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache),
and never reinstall dependencies just because you changed a line in your code.
Now copy your source code:

```Dockerfile
COPY . ./
```

Use [poetry build](https://poetry.eustace.io/docs/cli/#build) to build a wheel. Then
pip-install the wheel into your virtualenv:

```Dockerfile
RUN /root/.poetry/bin/poetry build && /venv/bin/pip install dist/*.whl
```

Do not use [poetry install](https://poetry.eustace.io/docs/cli/#install) to
install your code, because it will perform an [editable
install](https://pip.pypa.io/en/stable/reference/pip_install/#editable-installs).
(An editable install doesn’t actually install your package into the virtual
environment. It creates a `.egg-link` file that links to your source code, and
this link would only be valid for the duration of the build stage.)

The third `FROM` statement introduces the final stage:

```Dockerfile
FROM base as final
```

The final stage copies the virtual environment from the build stage:

```Dockerfile
COPY --from=builder /venv /venv
```

Set the
[PYTHONUNBUFFERED](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONUNBUFFERED)
environment variable to disable buffering for the standard output stream. This
makes sense for Docker images as [stdout is typically used for log
messages](https://12factor.net/logs).

```Dockerfile
ENV PYTHONUNBUFFERED=1
```

Set the entrypoint to the console script, which is located in the virtual
environment:

```Dockerfile
ENTRYPOINT ["/venv/bin/hypermodern-python"]
```

## Hosting images at Docker Hub

[Docker Hub](https://hub.docker.com/) is the official Docker image registry.
Uploading your image to Docker Hub means other users can download and run your
image with a single command:

```sh
docker run hypermodern-python [options]
```

Sign up for Docker Hub. On your Account Settings, under Linked Accounts, link
your Docker ID to your GitHub user account. You will be asked to grant access to
the Docker GitHub app.

In the Repositories section on Docker Hub, create the `hypermodern-python`
repository. In the Builds tab of the new repository, select GitHub as the code
repository service, and select your GitHub repository as the source repository.
Click *Save and Build* to confirm.

Docker Hub will build a Docker image with every git push to your GitHub
repository. Your image now also has a public page on Docker Hub:

> https://hub.docker.com/r/&lt;user&gt;/hypermodern-python

Add the Docker Hub badge to your repository:

```markdown
# README.md
[![Docker Hub](https://img.shields.io/docker/cloud/build/<your-username>/hypermodern-python.svg)](https://hub.docker.com/r/<your-username>/hypermodern-python)
```

The badge looks like this: [![Docker
Hub](https://img.shields.io/docker/cloud/build/cjolowicz/hypermodern-python.svg)](https://hub.docker.com/r/cjolowicz/hypermodern-python)
