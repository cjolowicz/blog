--- 
date: 2020-01-28T06:56:59+02:00
title: "Hypermodern Python Chapter 5: Documentation"
description: "A guide to modern Python tooling with a focus on simplicity and minimalism."
draft: true
tags:
  - python
  - nox
  - sphinx
  - flake8
  - darglint
  - xdoctest
---

[Read this article on Medium](https://medium.com/@cjolowicz/hypermodern-python-5-documentation-xxxxxxxxxxxx)

{{< figure
    src="/images/hypermodern-python-05/robida01.jpg" 
    link="/images/hypermodern-python-05/robida01.jpg"
>}}

In this fifth installment of the Hypermodern Python series,
I'm going to discuss
how to add documentation to your project.[^1]
In the [previous chapter](../hypermodern-python-04-typing), we discussed
how to add type annotations and type checking.
(If you start reading here,
you can also 
[download the code](https://github.com/cjolowicz/hypermodern-python/archive/chapter04.zip) 
for the previous chapter.)

[^1]: The images in this chapter are details from
      the photogravures
      *Paris la nuit*
      (Paris by night) and
      *Départ d'un ballon transatlantique*
      (Departure of a transatlantic balloon), and
      the wood engraving
      *Station centrale des aéronefs à Notre-Dame*
      (Central aircraft station at Notre-Dame),
      by Albert Robida, 1883
      (via [Old Book Illustrations](https://www.oldbookillustrations.com/writers/robida-albert/)).

Here are the topics covered in this chapter on documentation:

- [Documenting code with Python docstrings](#documenting-code-with-python-docstrings)
- [Linting code documentation with flake8-docstrings](#linting-code-documentation-with-flake8-docstrings)
- [Validating docstrings against function signatures with darglint](#validating-docstrings-against-function-signatures-with-darglint)
- [Running examples in docstrings with xdoctest](#running-examples-in-docstrings-with-xdoctest)
- [Creating documentation with Sphinx](#creating-documentation-with-sphinx)
- [Generating API documentation with autodoc](#generating-api-documentation-with-autodoc)

Here is a full list of the articles in this series:

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation) (this article)
- *Chapter 6: CI/CD*
- *Appendix: Docker*

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Each article in the guide corresponds to a set of commits in the GitHub repository:

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/chapter04...chapter05)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter05.zip)

## Documenting code with Python docstrings

{{< figure
    src="/images/hypermodern-python-05/robida02.jpg" 
    link="/images/hypermodern-python-05/robida02.jpg"
>}}

[Documentation strings](https://www.python.org/dev/peps/pep-0257/#what-is-a-docstring),
also known as *docstrings*,
allow you to embed documentation directly into your code.
An example of a docstring is the first line of `console.main`,
used by Click to
generate the usage message of your command-line interface:

```python
# src/hypermodern_python/console.py
def main(language: str) -> None:
    """The hypermodern Python project."""
```

Documentation strings communicate
the purpose and usage of a module, class, or function
to other developers reading your code.
Unlike comments, the Python bytecode compiler does not throw them away,
but adds them to the `__doc__` attribute of documented objects.
This allows tools like [Sphinx](http://www.sphinx-doc.org/)
to generate API documentation from your code.

You can document your entire package
by adding a docstring to the top of `__init__.py`:

```python
# src/hypermodern_python/__init__.py
"""The Hypermodern Python project."""
```

Document the modules in your package by adding docstrings
to the top of their respective source files.
Here's one for the `console` module:

```python
# src/hypermodern_python/console.py
"""Command-line interface."""
```

And here's a docstring for the `wikipedia` module:

```python
# src/hypermodern_python/wikipedia.py
"""Client for the Wikipedia REST API, version 1."""
```

So far we have used one-line docstrings.
Docstrings can also consist of multiple lines.
By convention, the first line is separated from the rest of the docstring by a blank line.
This structures the docstring into a summary line and a more elaborate description.

Docstring summaries can include useful information about
class attributes, function parameters, return values, and other things.
A common format for this with good readability and tooling support is the
[Google docstring style](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings).
Other options are the
[Sphinx](https://sphinx-rtd-tutorial.readthedocs.io/en/latest/docstrings.html) and
[NumPy](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_numpy.html#example-numpy)
formats. In this chapter, we adopt the Google style.

The following example shows a multi-line docstring for the `wikipedia.Page` class.
The summary describes class attributes in Google style.
Docstrings for classes are added to the first line of the class definition:

```python
# src/hypermodern_python/wikipedia.py
@dataclass
class Page:
    """Page resource.

    Attributes:
        title: The title of the Wikipedia page.
        extract: A plain text summary.
    """

    title: str
    extract: str
```

Similarly, you can document functions by 
adding docstrings to the first line of the function body.
Here is a multi-line docstring for the `wikipedia.random_page` function:

```python
# src/hypermodern_python/wikipedia.py
def random_page(language: str = "en") -> Page:
    """Return a random page.

    Performs a GET request to the /page/random/summary endpoint.

    Args:
        language: The Wikipedia language edition. By default, the English
            Wikipedia is used ("en").

    Returns:
        A page resource.

    Raises:
        ClickException: The HTTP request failed or the HTTP response
            contained an invalid body.
    """
```

## Linting code documentation with flake8-docstrings

{{< figure
    src="/images/hypermodern-python-05/robida03.jpg" 
    link="/images/hypermodern-python-05/robida03.jpg"
>}}

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

{{< figure
    src="/images/hypermodern-python-05/robida04.jpg" 
    link="/images/hypermodern-python-05/robida04.jpg"
>}}

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

{{< figure
    src="/images/hypermodern-python-05/robida05.jpg" 
    link="/images/hypermodern-python-05/robida05.jpg"
>}}

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

The [darglint](https://github.com/terrencepreilly/darglint) tool checks that the
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

{{< figure
    src="/images/hypermodern-python-05/robida06.jpg" 
    link="/images/hypermodern-python-05/robida06.jpg"
>}}

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

The [xdoctest](https://github.com/Erotemic/xdoctest) package runs the examples
in your docstrings and compares the actual output to the expected output as per
the docstring. Add `xdoctest` to your developer dependencies:

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

{{< figure
    src="/images/hypermodern-python-05/robida07.jpg" 
    link="/images/hypermodern-python-05/robida07.jpg"
>}}

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

{{< figure
    src="/images/hypermodern-python-05/robida08.jpg" 
    link="/images/hypermodern-python-05/robida08.jpg"
>}}

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

Update the lock file by invoking [poetry
lock](https://poetry.eustace.io/docs/cli/#lock).

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

## Thanks for reading!

The next chapter is about Continuous Integration and Delivery. It will be
published on February 5, 2020.

{{< figure src="/images/hypermodern-python-05/trolley.jpg" class="centered" >}}
<!-- 
{{< figure src="/images/hypermodern-python-05/trolley.jpg" link="../hypermodern-python-06-ci-cd" class="centered" >}}
<span class="centered">[Continue to the next chapter](../hypermodern-python-06-ci-cd)</span>
-->
