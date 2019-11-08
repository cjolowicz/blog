--- 
date: 2019-11-07T12:52:59+02:00
title: "The hypermodern Python project, pt. 2"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - pytype
  - nox
  - GitHub Actions
---

<!--
TODO:

- release-drafter
- Dockerfile
- Docker Hub
- cookiecutter
-->

In this second installment of the Hypermodern Python series, I'm going to
discuss how to add documentation to your project.

For your reference, below is a list of the articles in this series.

- Chapter 1: Setup
- Chapter 2: Documentation
- Chapter 3: Type Checking

<!--
This post has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Creating documentation with Sphinx](#creating-documentation-with-sphinx)
- [Hosting documentation at Read the Docs](#hosting-documentation-at-read-the-docs)
- [Documenting code using Python docstrings](#documenting-code-using-python-docstrings)
- [Linting code documentation with flake8-docstrings](#linting-code-documentation-with-flake8-docstrings)
- [Linting docstrings against function signatures with darglint](#linting-docstrings-against-function-signatures-with-darglint)
- [Running tests in docstrings with xdoctest](#running-tests-in-docstrings-with-xdoctest)
- [Generating API documentation with autodoc](#generating-api-documentation-with-autodoc)
- [Validating docstrings with flake8-rst-docstrings](#validating-docstrings-with-flake8-rst-docstrings)
- [Building a Docker image](#building-a-docker-image)
- [Conclusion](#conclusion)

<!-- markdown-toc end -->

## Creating documentation with Sphinx

[Sphinx](http://www.sphinx-doc.org/) is the documentation tool used by the
official Python documentation and many open-source projects. Sphinx
documentation is commonly written using
[reStructuredText](http://docutils.sourceforge.net/rst.html), although Markdown
is also supported. 

Create a directory `docs` and place the text below in the file `docs/index.rst`:

```rst
The hypermodern Python project
==============================

Installation
------------

To install the hypermodern Python project, run this command in your terminal:

.. code-block:: console

   $ pip install hypermodern-python

Usage
-----

Hypermodern Python's usage looks like:

.. code-block:: console

    $ hypermodern-python [OPTIONS]

.. option:: --version

    Display the version and exit.

.. option:: --help

    Display a short usage message and exit.
```

Create the Sphinx configuration file `docs/conf.py`, in the same directory. This
provides meta information about your project, and applies the theme
[sphinx-rtd-theme](https://github.com/readthedocs/sphinx_rtd_theme):


```python
# docs/conf.py
project = "hypermodern-python"
author = "Your Name"
copyright = f"2019, {author}"
extensions = ["sphinx_rtd_theme"]
html_theme = "sphinx_rtd_theme"
```

Add a Nox session to build the documentation:

```python
# noxfile.py
@nox.session(python="3.8")
def docs(session):
    """Build the documentation."""
    session.install("sphinx", "sphinx-rtd-theme")
    session.run("sphinx-build", "docs", "docs/_build")
```

For good measure, include `docs/conf.py` in the linting session:

```python
# noxfile.py
...
locations = "src", "tests", "noxfile.py", "docs/conf.py"
...
```

Run the Nox session:

```sh
nox -rs docs
```

You can now open the file `docs/_build/index.html` in your browser to view your
documentation offline.

## Hosting documentation at Read the Docs

[Read the Docs](https://readthedocs.org/) hosts documentation for countless
open-source Python projects. 

Create the `.readthedocs.yml` configuration file:

```yaml
version: 2
sphinx:
  configuration: docs/conf.py
formats: all
python:
  version: 3.7
  install:
    - requirements: docs/requirements.txt
```

Ensure that a recent Sphinx version is used, by adding this
`docs/requirements.txt` file:

```python
sphinx==2.2.0
sphinx-rtd-theme==0.4.3
```

Let's also adapt the `docs` session to use this same requirements file:

```python
# noxfile.py
...
session.install("-r", "docs/requirements.txt")"
```

Sign up at Read the Docs, and import your GitHub repository, using the button
*Import a Project*. Read the Docs automatically starts building your
documentation. When the build has completed, your documentation will have a
public URL like this:

> https://hypermodern-python.readthedocs.io/

You can display the documentation link on PyPI by including it in your package
metainfo:

```toml
# pyproject.toml
[tool.poetry]
...
documentation = "https://hypermodern-python.readthedocs.io"
```

Let's also add the link to the GitHub repository page, by adding a Read the Docs
badge to `README.md`:

```markdown
[![Read the Docs](https://readthedocs.org/projects/hypermodern-python/badge/)](https://hypermodern-python.readthedocs.io/)
```

The badge looks like this: [![Read the
Docs](https://readthedocs.org/projects/hypermodern-python/badge/)](https://hypermodern-python.readthedocs.io/)

## Documenting code using Python docstrings

```python
#   docs/conf.py
"""Sphinx configuration."""
...

# noxfile.py
"""Nox sessions."""
...

# src/hypermodern_python/__init__.py
"""The hypermodern Python project."""
...

# src/hypermodern_python/console.py
"""Command-line interface for the hypermodern Python project."""
...

# src/hypermodern_python/splines.py
"""Utilities for spline manipulation."""
...

def reticulate(count: int = -1) -> Iterator[int]:
    """Reticulate splines."""
    ...

# tests/__init__.py
"""Test cases for the hypermodern_python package."""
...

# tests/conftest.py
"""Package-wide test fixtures."""
...

@pytest.fixture
def mock_sleep(mocker):
    """Mock for time.sleep."""
    ...
# tests/test_console.py
"""Test cases for the console module."""
...

@pytest.fixture
def runner():
    """Fixture for invoking command-line interfaces."""
    ...
 
 
def test_help_succeeds(runner):
    """Test if --help succeeds."""
    ... 
 

def test_main_prints_progress_message(runner, mock_sleep):
    """Test if main prints a progress message."""
    ...

# tests/test_splines.py
"""Test cases for the splines module."""
...
 
 
def test_reticulate_sleeps(mock_sleep):
    """Test if reticulate sleeps."""
    ... 

 
def test_reticulate_yields_count_times(mock_sleep):
    """Test if reticulate yields <count> times."""
    ...
```

## Linting code documentation with flake8-docstrings

The [flake8-docstrings](https://gitlab.com/pycqa/flake8-docstrings) plugin
checks that docstrings are compliant with [PEP
257](https://www.python.org/dev/peps/pep-0257/) using
[pydocstyle](https://github.com/pycqa/pydocstyle).

Add `flake8-docstrings` to the `lint` session:

```python
# noxfile.py
...
session.install("flake8", "flake8-black", "flake8-docstrings")
```

Enable the plugin warnings (`D`) and use Google conventions for docstrings:

```ini
# .flake8
select = C,D,E,F,W
docstring-convention = google
```

## Linting docstrings against function signatures with darglint

https://github.com/terrencepreilly/darglint checks that the docstring description matches the definition.

```python
# noxfile.py
...
session.install("flake8", "flake8-black", "flake8-docstrings", "flake8-rst-docstrings", "darglint")
```

Configure darglint to accept short docstrings:

```init
# .darglint
[darglint]
strictness=short
```

TODO Enable darglint warnings?

Document function arguments and return value for `splines.reticulate` function:

```python
# src/hypermodern_python/splines.py
...

def reticulate(count: int = -1) -> Iterator[int]:
    """Reticulate splines.

    Args:
        count: Number of splines to reticulate

    Yields:
        Reticulated splines

    """
    ...
```

## Running tests in docstrings with xdoctest

A good way to explain how to use your function is to include an example in your
docstring:

```python
# src/hypermodern_python/splines.py
...

def reticulate(count: int = -1) -> Iterator[int]:
    """Reticulate splines.

    Args:
        count: Number of splines to reticulate

    Yields:
        Reticulated splines

    Example:
        >>> from hypermodern_python import splines
        >>> a, b = splines.reticulate(2)
        >>> a, b
        (1, 2)

    """
    ...
```

The `xdoctest` package runs the examples in your docstrings and compares the
actual output to the expected output as per the docstring. Add `xdoctest` to
your developer dependencies:

```sh
poetry add --dev xdoctest
```

The `xdoctest` package integrates with pytest as a plugin. Invoke `pytest` with
the `--xdoctest` option to activate the plugin:

```python
# noxfile.py
...
def tests(session):
    ...
    session.run("pytest", "--cov", "--xdoctest", *session.posargs)
```

## Generating API documentation with autodoc

- autodoc
- napoleon
- sphinx-autodoc-typehints

```requirements
# docs/requirements.txt
sphinx==2.2.0
sphinx-rtd-theme==0.4.3
sphinx-autodoc-typehints==1.8.0
```

Add the extensions to your Sphinx configuration:

```python
# docs/conf.py
extensions = [
    "sphinx_rtd_theme",
    "sphinx.ext.autodoc",
    "sphinx.ext.napoleon",
    "sphinx_autodoc_typehints",
]
```

Install the package when building your documentation:

```python
# noxfile.py
...
def docs(session):
    ...
    env = {"VIRTUAL_ENV": session.virtualenv.location}
    session.run("poetry", "install", external=True, env=env)
```

Add documentation for the `splines` module:

```rst
# docs/splines.rst
Splines
=======
.. py:module:: hypermodern_python.splines

.. autofunction:: reticulate()
```

Add `toctree` directive to `index.rst`:

```rst
# docs/index.rst
...

.. toctree::
  :hidden:
  :maxdepth: 2

  splines
```

## Validating docstrings with flake8-rst-docstrings

The [flake8-rst-docstrings](https://github.com/peterjc/flake8-rst-docstrings)
plugin validates docstrings as reStructuredText (RST).

```python
# noxfile.py
...
session.install("flake8", "flake8-black", "flake8-docstrings", "flake8-rst-docstrings")
```

Enable flake8-rst-docstrings warnings:

```ini
# .flake8
select = C,D,E,F,RST,W
```

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

<!--
{{< figure src="http://www.vintagecomputer.net/ctc/3300/CTC_DataPoint-3300_pic3.jpg" caption="Fun fact: Consoles have supported dark mode since 1969, exactly half a century before iOS 13." alt="DataPoint 3300 (1969)" link="https://www.youtube.com/watch?v=dEGlKpIBujc" width="80%" class="centered" >}}
-->

<!--
{{< figure src="../../images/hypermodern-python-github.png" width="90%" alt="Create a GitHub repository" class="centered" >}}
-->
