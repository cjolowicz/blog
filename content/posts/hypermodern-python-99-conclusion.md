--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 9: Conclusion"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - sphinx
  - readthedocs
  - nox
---

## Building a Docker image

Add the following `Dockerfile` to the root of your project:

```Dockerfile
FROM python:3.8.0-alpine3.10 as base

ENV PYTHONFAULTHANDLER=1 \
    PYTHONHASHSEED=random \
    PYTHONUNBUFFERED=1

WORKDIR /app

FROM base as builder

ENV PIP_DEFAULT_TIMEOUT=100 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    POETRY_VERSION=1.0.0b2

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

## Conclusion

Thank you for reading this far.

This article is dedicated to my father who introduced me to programming in the
1980s, and who is an avid chess book collector. I saw a copy of *Die
hypermoderne Schachpartie* (The hypermodern chess game) in his bookshelf,
written by [Savielly
Tartakower](https://en.wikipedia.org/wiki/Savielly_Tartakower) in 1925 to
modernize chess theory. That inspired the title of this guide.
