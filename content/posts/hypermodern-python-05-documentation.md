--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 5: Documentation"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - sphinx
  - readthedocs
  - nox
---

In this fifth installment of the Hypermodern Python series, I'm going to discuss
how to add documentation to your project.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation)
- [Chapter 6: CI/CD](../hypermodern-python-06-ci-cd)

<!--
This post has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Documenting code with Python docstrings](#documenting-code-with-python-docstrings)
- [Linting code documentation with flake8-docstrings](#linting-code-documentation-with-flake8-docstrings)
- [Validating docstrings against function signatures with darglint](#validating-docstrings-against-function-signatures-with-darglint)
- [Running examples in docstrings with xdoctest](#running-examples-in-docstrings-with-xdoctest)
- [Creating documentation with Sphinx](#creating-documentation-with-sphinx)
- [Generating API documentation with autodoc](#generating-api-documentation-with-autodoc)

<!-- markdown-toc end -->

## Documenting code with Python docstrings

[Documentation
strings](https://www.python.org/dev/peps/pep-0257/#what-is-a-docstring), also
known as *docstrings*, allow you to embed documentation directly into your code.
An example of a docstring is the first line of `console.main`, used by `click`
to generate the usage message of your command-line interface:

```python
# src/hypermodern_python/console.py
def main(count: int) -> None:
    """The hypermodern Python project."""
    for spline in splines.reticulate(count):
        click.echo(f"Reticulating spline {spline}...")
```

More commonly, documentation strings communicate the purpose and usage of a
module, class, or function to other developers reading your code. As we shall
see at the end of this chapter, they can also be used to generate API
documentation.

Document your entire package by adding a docstring to the top of `__init__.py`:

```python
# src/hypermodern_python/__init__.py
"""The hypermodern Python project."""
...
```

Document the modules in the package by adding docstrings to the top of their
respective source files:

```python
# src/hypermodern_python/console.py
"""Command-line interface for the hypermodern Python project."""
...
```

```python
# src/hypermodern_python/splines.py
"""Utilities for spline manipulation."""
...
```

Document functions by adding docstrings to the first line of the function body:

```python
# src/hypermodern_python/splines.py
def reticulate(count: int = -1) -> Iterator[int]:
    """Reticulate splines."""
    ...

```

## Linting code documentation with flake8-docstrings

The [flake8-docstrings](https://gitlab.com/pycqa/flake8-docstrings) plugin uses
the tool [pydocstyle](https://github.com/pycqa/pydocstyle) to check that
docstrings are compliant with the style recommendations of [PEP
257](https://www.python.org/dev/peps/pep-0257/). Warnings range from missing
docstrings to issues with whitespace, quoting, and docstring content.

Add `flake8-docstrings` to the `lint` session:

```python
# noxfile.py
@nox.session(python=["3.8", "3.7"])
def lint(session: Session) -> None:
    """Lint using flake8."""
    args = session.posargs or locations
    session.install(
        "flake8",
        "flake8-annotations",
        "flake8-bandit",
        "flake8-black",
        "flake8-bugbear",
        "flake8-docstrings",
        "flake8-import-order",
    )
    session.run("flake8", *args)
```

Configure Flake8 to enable the plugin warnings (`D` for docstring) and adopt the
[Google docstring
style](http://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings):
 
```ini
# .flake8
select = B,B9,BLK,C,D,E,F,I,S,TYP,W
docstring-convention = google
```

## Adding more docstrings to the project

Running `nox -rs lint` finds missing docstrings outside of the package, such as
in `noxfile.py` itself. Let's fix this one first:

```python
# noxfile.py
"""Nox sessions."""

def black(session: Session) -> None:
    """Run black code formatter."""

def lint(session: Session) -> None:
    """Lint using flake8."""

def mypy(session: Session) -> None:
    """Type-check using mypy."""

def pytype(session: Session) -> None:
    """Type-check using pytype."""

def tests(session: Session) -> None:
    """Run the test suite."""
```

Next, improve the readability of the test suite by adding docstrings to its
modules, test cases, and fixtures.

```python
# tests/__init__.py
"""Test suite for the hypermodern_python package."""
```

```python
# tests/conftest.py
"""Package-wide test fixtures."""

@pytest.fixture
def mock_sleep(mocker: MockFixture) -> Mock:
    """Mock for time.sleep."""
```

```python
# tests/test_console.py
"""Test cases for the console module."""

@pytest.fixture
def runner() -> CliRunner:
    """Fixture for invoking command-line interfaces."""
 
@pytest.fixture
def mock_splines_reticulate(mocker: MockFixture) -> Mock:
    """Mock for splines.reticulate."""

def test_main_succeeds(runner: CliRunner, mock_splines_reticulate: Mock) -> None:
    """Test if console.main succeeds."""
 
def test_main_prints_progress_message(runner: CliRunner, mock_sleep: Mock) -> None:
    """Test if console.main prints a progress message."""
```

```python
# tests/test_splines.py
"""Test cases for the splines module."""
 
def test_reticulate_yields_count_times(mock_sleep: Mock) -> None:
    """Test if reticulate yields <count> times."""
 
def test_reticulate_sleeps(mock_sleep: Mock) -> None:
    """Test if reticulate sleeps."""
```

## Validating docstrings against function signatures with darglint

Documentation strings can include useful information about parameters accepted
by a function, its return value, and any exceptions raised. A common format for
this with good tooling support is the [Google docstring
style](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings).
Other options are the
[Sphinx](https://sphinx-rtd-tutorial.readthedocs.io/en/latest/docstrings.html)
and
[NumPy](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_numpy.html#example-numpy)
formats.

Let's add usage information for the `splines.reticulate` function. This function
accepts the number of splines as an optional parameter, and yields splines at
each iteration:

```python
# src/hypermodern_python/splines.py
def reticulate(count: int = -1) -> Iterator[int]:
    """Reticulate splines.

    Args:
        count: Number of splines to reticulate

    Yields:
        A reticulated spline

    """
```

[darglint](https://github.com/terrencepreilly/darglint) checks that the
docstring description matches the definition, and integrates with Flake8 as a
plugin. Add the plugin to the lint session:

```python
# noxfile.py
@nox.session(python=["3.8", "3.7"])
def lint(session: Session) -> None:
    """Lint using flake8."""
    args = session.posargs or locations
    session.install(
        "flake8",
        "flake8-annotations",
        "flake8-bandit",
        "flake8-black",
        "flake8-bugbear",
        "flake8-docstrings",
        "flake8-import-order",
        "darglint",
    )
    session.run("flake8", *args)
```

Unlike other plugins, darglint enables most of its warnings by default. For
consistency and to future-proof your Flake8 configuration, enable the plugin
warnings explicitly (`DAR` like *darglint*):

```ini
# .flake8
[flake8]
select = B,B9,BLK,C,D,DAR,E,F,I,S,TYP,W
```

Sometimes one-line docstrings are quite sufficient. Configure darglint to accept
short docstrings, using the `.darglint` configuration file:

```ini
# .darglint
[darglint]
strictness=short
```

## Running examples in docstrings with xdoctest

A good way to explain how to use your function is to include an example in your
docstring:

```python
# src/hypermodern_python/splines.py
def reticulate(count: int = -1) -> Iterator[int]:
    """Reticulate splines.

    Args:
        count: Number of splines to reticulate

    Yields:
        A reticulated spline

    Example:
        >>> from hypermodern_python import splines
        >>> a, b = splines.reticulate(2)
        >>> a, b
        (1, 2)

    """
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
@nox.session(python=["3.8", "3.7"])
def tests(session: Session) -> None:
    """Run the test suite."""
    args = session.posargs or ["--cov", "--xdoctest"]
    session.run("poetry", "install", external=True)
    session.run("pytest", *args)
```

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

.. option:: -n <count>, --count <count>

    Number of splines to reticulate. By default, the program reticulates
    splines until interrupted by Ctrl+C.

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
"""Sphinx configuration."""
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
locations = "src", "tests", "noxfile.py", "docs/conf.py"
```

Run the Nox session:

```sh
nox -rs docs
```

You can now open the file `docs/_build/index.html` in your browser to view your
documentation offline.

## Generating API documentation with autodoc

In this section, you are going to use Sphinx to generate API documentation from
the documentation strings and type annotations in your package, using three
Sphinx extensions:

- [autodoc](https://www.sphinx-doc.org/en/master/usage/extensions/autodoc.html)
  enables Sphinx to generate API documentation from the docstrings in your
  package.
- [napoleon](https://www.sphinx-doc.org/en/master/usage/extensions/napoleon.html)
  pre-processes Google-style docstrings to reStructuredText.
- [sphinx-autodoc-typehints](https://github.com/agronholm/sphinx-autodoc-typehints)
  allows Sphinx to include type annotations in the generated API documentation.

For Sphinx to be able to read the documentation strings and type annotations in
your package, you need to install your package before building the
documentation. As the documentation dependencies include the package itself, it
makes sense to declare them as *extra dependencies* of your package, rather than
simply installing them in the Nox session. This has the following advantages:

- You can use Poetry to manage dependencies.
- You can build the documentation from within CI, using the same environment.
- Extra dependencies are kept separate from the main package dependencies.

Add the documentation dependencies with Poetry using the `--optional` option,
like this:

```sh
poetry add --optional sphinx sphinx-rtd-theme sphinx-autodoc-typehints
```

In case you wonder, the `autodoc` and `napoleon` extensions are distributed with
Sphinx, so there is no need to install them explicitly.

Declare the documentation dependencies in `pyproject.toml`, as an extra named
`docs`:

```toml
# pyproject.toml
[tool.poetry.extras]
docs = ["sphinx", "sphinx-rtd-theme", "sphinx-autodoc-typehints"]
```

You can now simply install your package with the `--extras=docs` option. Update
the Nox session:

```python
# noxfile.py
@nox.session(python="3.8")
def docs(session: Session) -> None:
    """Build the documentation."""
    session.run("poetry", "install", "--extras=docs", external=True)
    session.run("sphinx-build", "docs", "docs/_build")
```

Activate the extensions by declaring them in the Sphinx configuration file:

```python
# docs/conf.py
extensions = [
    "sphinx_rtd_theme",
    "sphinx.ext.autodoc",
    "sphinx.ext.napoleon",
    "sphinx_autodoc_typehints",
]
```

From now on, you can reference docstrings in your Sphinx documentation using
directives such as `autoclass` and `autofunction`.

Create the file `docs/splines.rst`, with API documentation for the `splines`
module:

```rst
Splines
=======
.. py:module:: hypermodern_python.splines

.. autofunction:: reticulate()
```

Include the new file in a `toctree` directive at the end of `docs/index.rst`:

```rst
.. toctree::
  :hidden:
  :maxdepth: 2

  splines
```

Rebuild the documentation using `nox -rs docs`, and open the file
`docs/_build/index.html` in your browser. Navigate to the `splines` module to
view its API documentation.

<center>[Continue to the next chapter](../hypermodern-python-06-ci-cd)</center>
