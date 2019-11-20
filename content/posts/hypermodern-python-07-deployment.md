--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 7: Deployment"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - sphinx
  - readthedocs
  - nox
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

[Docker](https://www.docker.com/) allows you to build your

Add the following `Dockerfile` to the root of your project:

```Dockerfile
FROM python:3.8.0-alpine3.10 as base

ENV PYTHONUNBUFFERED=1

WORKDIR /app

FROM base as builder

ENV PIP_DEFAULT_TIMEOUT=60 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    POETRY_VERSION=1.0.0b5

RUN pip install "poetry==$POETRY_VERSION"
RUN python -m venv /venv

COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt | /venv/bin/pip install -r /dev/stdin

COPY . .
RUN poetry build && /venv/bin/pip install dist/*.whl

FROM base as final

COPY --from=builder /venv /venv
CMD ["/venv/bin/hypermodern-python"]
```

You can build the Dockerfile using the following command:

```sh
docker build -t hypermodern-python .
```

Run a container like this:

```sh
docker run hypermodern-python [options]
```

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
