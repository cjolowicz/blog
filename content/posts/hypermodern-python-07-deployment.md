--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 7: Deployment"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - docker
---

In this seventh and last installment of the Hypermodern Python series, I'm going
to discuss various options to deploy your project.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation)
- [Chapter 6: CI/CD](../hypermodern-python-06-ci-cd)
- [Chapter 7: Deployment](../hypermodern-python-07-deployment)

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Building a Docker image](#building-a-docker-image)
- [Hosting images at Docker Hub](#hosting-images-at-docker-hub)
- [Conclusion](#conclusion)

<!-- markdown-toc end -->

## Building a Docker image

[Docker](https://www.docker.com/) enables you to build, share, and run your
application anywhere using containers. Containers are a lightweight
virtualization technology, packaging your application with its dependencies, and
allowing it to be executed in an isolated environment.

Add the following `Dockerfile` to the root of your project:

```Dockerfile
FROM python:3.8.0-alpine3.10 as base
FROM base as builder

WORKDIR /app
ENV POETRY_VERSION=1.0.0b5
RUN wget -qO- https://raw.githubusercontent.com/sdispater/poetry/master/get-poetry.py | python
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

This Dockerfile defines a so-called [multi-stage
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

Here are a few more observations about the Dockerfile:

- The image is based on the official Python image for
  [Alpine](https://alpinelinux.org/), a security-oriented, lightweight Linux
  distribution.
- Use `poetry export` to install your pinned requirements first, before copying
  your code. This will allow you to leverage the [Docker build
  cache](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache),
  and never reinstall dependencies just because you changed a line in your code.
- Use `poetry build` to build a wheel, and then pip-install that into your
  virtualenv. Do not use `poetry install` to install your code, because it will
  perform an [editable
  install](https://pip.pypa.io/en/stable/reference/pip_install/#editable-installs).
- The final stage sets the
  [PYTHONUNBUFFERED](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONUNBUFFERED)
  environment variable to disable buffering for the standard output stream. This
  makes sense for Docker images as [stdout is typically used for log
  messages](https://12factor.net/logs).

Build the Docker image using the following command:

```sh
docker build -t hypermodern-python .
```

Run your image like this:

```sh
docker run hypermodern-python [options]
```

You can pass options to `hypermodern-python` by appending them to the end of the
command-line.

## Hosting images at Docker Hub

## Conclusion

Thank you for reading this far. This chapter concludes the Hypermodern Python
series.

This series of articles is dedicated to my father who introduced me to
programming in the 1980s, and who is an avid chess book collector. I saw a copy
of *Die hypermoderne Schachpartie* (The hypermodern chess game) in his
bookshelf, written by [Savielly
Tartakower](https://en.wikipedia.org/wiki/Savielly_Tartakower) in 1925 to
modernize chess theory. That inspired the title of this guide.

<!--
{{< figure src="http://www.vintagecomputer.net/ctc/3300/CTC_DataPoint-3300_pic3.jpg" caption="Fun fact: Consoles have supported dark mode since 1969, exactly half a century before iOS 13." alt="DataPoint 3300 (1969)" link="https://www.youtube.com/watch?v=dEGlKpIBujc" width="80%" class="centered" >}}
-->
