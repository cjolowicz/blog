--- 
date: 2019-10-16T09:12:59+02:00
title: "The hypermodern Python project"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - poetry
  - nox
  - GitHub Actions
  - pytest
  - coverage.py
  - Codecov
  - PyPI
  - black
  - pyenv
---

<!--
TODO:

- release-drafter
- Dockerfile
- Docker Hub
- cookiecutter
-->

Welcome to the whirlwind tour of the Python ecosystem in late 2019!

[Python 3.8](https://docs.python.org/3/whatsnew/3.8.html) has been officially
released this month, and the [Python 2
sunset](https://www.python.org/doc/sunset-python-2/) will occur on new year
2020, after more than a decade of coexistence with Python 3.

The Python landscape has changed drastically over the last decade, with a host
of new tools and best practices improving the Python developer experience. At
the same time, their adoption has lagged behind, due to the constraints of
legacy support. Time to show how to build a Python project for
*hypermodernists*, from scratch.

This post is aimed both at beginners who are keen to learn best practises from
the start, and seasoned Python developers whose workflows are affected by
boilerplate and workarounds required by the legacy toolbox. The focus is on
simplicity and minimalism.

<!--
This post has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

> *You need a recent Linux, Unix, or Mac system with
> [bash](https://www.gnu.org/software/bash/), [curl](https://curl.haxx.se) and
> [git](https://www.git-scm.com) for this tutorial.*

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Static type checking with pytype](#static-type-checking-with-pytype)
- [Creating documentation with Sphinx](#creating-documentation-with-sphinx)
- [Hosting documentation at Read the Docs](#hosting-documentation-at-read-the-docs)
- [Reticulating splines](#reticulating-splines)
- [Mocking with pytest-mock](#mocking-with-pytest-mock)
- [Increasing type coverage with flake8-annotations](#increasing-type-coverage-with-flake8-annotations)
- [Documenting code using Python docstrings](#documenting-code-using-python-docstrings)
- [Linting code documentation with flake8-docstrings](#linting-code-documentation-with-flake8-docstrings)
- [Linting docstrings against function signatures with darglint](#linting-docstrings-against-function-signatures-with-darglint)
- [Running tests in docstrings with xdoctest](#running-tests-in-docstrings-with-xdoctest)
- [Generating API documentation with autodoc](#generating-api-documentation-with-autodoc)
- [Validating docstrings with flake8-rst-docstrings](#validating-docstrings-with-flake8-rst-docstrings)
- [Building a Docker image](#building-a-docker-image)
- [Awesome flake8 extensions](#awesome-flake8-extensions)
- [Conclusion](#conclusion)

<!-- markdown-toc end -->

## Static type checking with pytype

With the advent of [type hints](https://docs.python.org/3/library/typing.html),
several static type checkers for Python have come into existence:
[mypy](http://mypy-lang.org/), Google's
[pytype](https://google.github.io/pytype/), and Microsoft's
[pyright](https://github.com/microsoft/pyright).

Add the following session to `noxfile.py` to add `pytype`:

```python
@nox.session(python=["3.8", "3.7"])
def pytype(session):
    """Run the static type checker."""
    session.install("pytype")
    session.run("pytype", "--config=pytype.cfg", *locations)
```

Update `nox.options.session` to include static type checking in the default Nox
sessions:

```python
nox.options.sessions = "lint", "pytype", "tests"
```

Create the `pytype.cfg` configuration file and disable import errors. Some
third-party packages are still distributed without type annotations:

```ini
# pytype.cfg
[pytype]
disable = import-error
```

You could also add a `# pytype: disable=import-error` comment to the `import`
statements of offending packages, but this can get pretty repetitive.

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

For good measure, include `docs/conf.py` in the linting and type checking
sessions:

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

## Reticulating splines

Right now, our package doesn't do anything useful yet. Let's change this and add
a module that can be used to reticulate splines:

```python
# src/hypermodern_python/splines.py
import time
from typing import Iterator


def reticulate(count: int = -1) -> Iterator[int]:
    spline: int = 0
    while count < 0 or spline < count:
        time.sleep(1)
        spline += 1
        yield spline
```

This `splines` module defines a single function `splines.reticulate`. If you
wonder what the `variable: type = value` constructs mean, read about type
annotations. If `yield` is new to you, read about generators. If you wonder what
the `import` statement in the first line does, how the heck did you get this
far?

Add a test case:

```python
# tests/test_splines.py
from hypermodern_python import splines


def test_reticulate_yields_count_times():
    iterator = splines.reticulate(2)
    assert sum(1 for _ in iterator) == 2
```

Call the module from the CLI:

```python
# src/hypermodern_python/console.py
 import click
 
-from . import __version__
+from . import __version__, splines
 
 
 @click.command()
 @click.version_option(version=__version__)
 def main():
     """The hypermodern Python project."""
+    for spline in splines.reticulate():
+        click.echo(f"Reticulating spline {spline}...")
```

This runs forever. Add an option to control the number of splines to reticulate:

```python
# src/hypermodern_python/console.py
...
 @click.command()
+@click.option("-n", "--count", default=-1, help="Number of splines to reticulate")
 @click.version_option(version=__version__)
-def main():
+def main(count):
     """The hypermodern Python project."""
-    for spline in splines.reticulate():
+    for spline in splines.reticulate(count):
         click.echo(f"Reticulating spline {spline}...")
```

Add a test for the new console functionality:

```python
# tests/test_console.py
def test_main_prints_progress_message(runner, mock_sleep):
    result = runner.invoke(console.main, ["--count=1"])
    assert result.output == "Reticulating spline 1...\n"
```

## Mocking with pytest-mock

Tests are slow. 

The [unittest.mock](https://docs.python.org/3/library/unittest.mock.html)
standard library allows you to replace parts of your system under test with mock
objects and make assertions about how they have been used.

Add [pytest-mock](https://github.com/pytest-dev/pytest-mock), a `pytest` plugin
wrapping the `unittest.mock` API with its `mocker` fixture:

```sh
poetry add --dev pytest-mock
```

Mock `time.sleep` using `mocker`. Put it into `conftest.py` to make it
accessible to all tests without importing:

```python
# tests/conftest.py
import pytest


@pytest.fixture
def mock_sleep(mocker):
    return mocker.patch("time.sleep")
```

Use the mock in the test case:

```python
def test_reticulate_yields_count_times(mock_sleep):
    ...
```
    
Now the test passes immediately, without invoking the actual `time.sleep`.

You can also check that the mock was called by the function under test:

```python
def test_reticulate_sleeps(mock_sleep):
    for _ in splines.reticulate():
        break
    assert mock_sleep.called
```

## Increasing type coverage with flake8-annotations

Install plugin:

```python
# noxfile.py
session.install("flake8", "flake8-black", "flake8-annotations")
```

Enable warnings:

```ini
# .flake8
select = C,E,F,TYP,W
```

Add type annotations:

```python
# src/hypermodern_python/console.py
def main(count: int) -> None:

# noxfile.py
def black(session: nox.sessions.Session) -> None:
def lint(session: nox.sessions.Session) -> None:
def tests(session: nox.sessions.Session) -> None:
def coverage(session: nox.sessions.Session) -> None:

# tests/conftest.py
import unittest.mock
def mock_sleep(mocker: pytest_mock.MockFixture) -> unittest.mock.Mock:

# tests/test_console.py
import unittest.mock
def runner() -> click.testing.CliRunner:
def test_help_succeeds(runner: click.testing.CliRunner) -> None:
def test_main_prints_progress_message(
    runner: click.testing.CliRunner, mock_sleep: unittest.mock.Mock
) -> None:

# tests/test_splines.py
import unittest.mock
def test_reticulate_sleeps(mock_sleep: unittest.mock.Mock) -> None:
def test_reticulate_yields_count_times(mock_sleep: unittest.mock.Mock) -> None:
```

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

## Awesome flake8 extensions

There are many [awesome extensions for
flake8](https://github.com/DmytroLitvinov/awesome-flake8-extensions).

**flake8-bugbear**

The [flake8-bugbear](https://pypi.org/project/flake8-bugbear/) plugin helps you
finding bugs and design problems in your program. Add the plugin to the linter
session in your `noxfile.py`:

```python
# noxfile.py
...
session.install("flake8", "flake8-bugbear")
...
```

Enable Bugbear's warnings in the `.flake8` configuration file. These warnings are
prefixed with `B`:

```ini
# .flake8
[flake8]
select = B,B9,C,E,F,W
...
```

This also enables Bugbear's opinionated warnings (`B9`), which are disabled by
default. In particular, `B950` checks the maximum line length like the built-in
`E501`, but with a tolerance margin of 10%. Ignore the built-in error `E501` and
set the maximum line length to a sane value:

```ini
# .flake8
[flake8]
...
ignore = E501
max-line-length = 80
```

**flake8-import-order**

The [flake8-import-order](https://github.com/PyCQA/flake8-import-order) plugin
checks whether the order of import statements is consistent and [PEP
8](https://www.python.org/dev/peps/pep-0008/#imports)-compliant. Install the
plugin in the linter session:

```python
# noxfile.py
...
session.install("flake8", "flake8-bugbear", "flake8-black", "flake8-import-order")
...
```

Enable the warnings emitted by the plugin, which are prefixed by `I`. Also
inform the plugin about the local package name, which affects sorting order:

```ini
# .flake8
[flake8]
select = B,B9,C,E,F,I,W
...
application-import-names = hypermodern_python,tests
```

**flake8-docstrings and flake8-rst-docstrings**

The [flake8-docstrings](https://gitlab.com/pycqa/flake8-docstrings) plugin
checks that docstrings are compliant with [PEP
257](https://www.python.org/dev/peps/pep-0257/) using
[pydocstyle](https://github.com/pycqa/pydocstyle).

The [flake8-rst-docstrings](https://github.com/peterjc/flake8-rst-docstrings)
plugin validates docstrings as reStructuredText (RST).

**darglint**

https://github.com/terrencepreilly/darglint: checks that the docstring description matches the definition.

**flake8-bandit**

https://github.com/tylerwince/flake8-bandit: Automated security testing using
bandit and flake8

https://github.com/PyCQA/bandit: Find common security issues in Python code

**flake8-annotations**

https://github.com/python-discord/flake8-annotations: detects the absence of PEP 3107-style function annotations and PEP 484-style type comments

<!-- TODO -->

## Conclusion

Thank you for reading this far.

This article is dedicated to my father who introduced me to programming in the
1980s, and who is an avid chess book collector. I saw a copy of *Die
hypermoderne Schachpartie* (The hypermodern chess game) in his bookshelf,
written by [Savielly
Tartakower](https://en.wikipedia.org/wiki/Savielly_Tartakower) in 1925 to
modernize chess theory. That inspired the title of this blog post.

<!--
{{< figure src="http://www.vintagecomputer.net/ctc/3300/CTC_DataPoint-3300_pic3.jpg" caption="Fun fact: Consoles have supported dark mode since 1969, exactly half a century before iOS 13." alt="DataPoint 3300 (1969)" link="https://www.youtube.com/watch?v=dEGlKpIBujc" width="80%" class="centered" >}}
-->

<!--
{{< figure src="../../images/hypermodern-python-github.png" width="90%" alt="Create a GitHub repository" class="centered" >}}
-->
