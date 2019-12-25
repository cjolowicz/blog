--- 
date: 2020-01-15T10:04:00+01:00
title: "Hypermodern Python Chapter 4: Typing"
description: "A guide to modern Python tooling with a focus on simplicity and minimalism."
draft: true
tags:
  - python
  - nox
  - mypy
  - pytype
  - flake8
---

[Read this article on Medium](https://medium.com/@cjolowicz/hypermodern-python-4-typing-xxxxxxxxxxxx)

In this fourth installment of the Hypermodern Python series, I'm going to
discuss how to add type annotations and static type checking to your
project.[^1] Previously, we discussed [how to add linting, static analysis, and
code formatting](../hypermodern-python-03-linting).

[^1]: The images in this chapter are details from illustrations to the book *A
    Journey in Other Worlds: A Romance of the Future* by John Jacob Astor, 1894
    (source: [The Internet
    Archive](https://archive.org/details/journeyinotherwo00astouoft)).

Here are the topics covered in this chapter on Typing in Python:

- [Static type checkers and type annotations](#static-type-checkers-and-type-annotations)
- [Static type checking with mypy](#static-type-checking-with-mypy)
- [Static type checking with pytype](#static-type-checking-with-pytype)
- [Adding type annotations to the package](#adding-type-annotations-to-the-package)
- [Increasing type coverage with flake8-annotations](#increasing-type-coverage-with-flake8-annotations)
- [Adding type annotations to the test suite](#adding-type-annotations-to-the-test-suite)

Here is a full list of the articles in this series:

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing) (this article)
- *Chapter 5: Documentation*
- *Chapter 6: CI/CD*
- *Appendix: Docker*

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Each article in the guide corresponds to a set of commits in the GitHub
repository:

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/chapter03...chapter04)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter04.zip)

## Type annotations and type checkers

[Type annotations](https://docs.python.org/3/library/typing.html), first
introduced in Python 3.5, are a way to annotate functions and variables with
types. Combined with tooling that understands them, they can make your programs
easier to understand, debug, and maintain. Here are two simple examples of type
annotations:

```python
# This is a variable holding an integer.
answer: int = 42


# This is function which accepts and returns an integer.
def increment(number: int) -> int:
    return number + 1
```

The Python runtime does not enforce type annotations. Python is a dynamically
typed language: it only verifies the types of your program at runtime, and uses
*duck typing* to do so ("if it walks and quacks like a duck, it is a duck"). A
static type checker, by contrast, can use type annotations and type inference to
verify the type correctness of your program without executing it, helping you
discover many bugs that would have otherwise gone unnoticed.

<!--

Static type checkers analyze your source code for type errors, like treating a
string as an integer. 

Type annotations are entirely optional: They do not interfere when running your
program, and type checkers do not require your code to be fully annotated. Some
type checkers are able to infer types for unannotated code, but all support the
coexistence of dynamically and statically typed Python.

-->

The introduction of type annotations has paved the way for an entire generation
of static type checkers: [mypy](http://mypy-lang.org/) can be considered the
pioneer and *de facto* reference implementation of Python type checking; several
core Python developers are involved in its development. Google, Facebook, and
Microsoft have each announced their own static type checkers for Python:
[pytype](https://google.github.io/pytype/), [pyre](https://pyre-check.org/), and
[pyright](https://github.com/microsoft/pyright), respectively. JetBrain's Python
IDE [PyCharm](https://www.jetbrains.com/pycharm/) also ships with a static type
checker. In this chapter, we introduce mypy and pytype.

## Static type checking with mypy

Add [mypy](http://mypy-lang.org/) as a development dependency:

```sh
poetry add --dev mypy
```

Add the following Nox session to type-check your source code with mypy:

```python
@nox.session(python=["3.8", "3.7"])
def mypy(session: Session) -> None:
    args = session.posargs or locations
    install_with_constraints(session, "mypy")
    session.run("mypy", *args)
```

Include mypy in the default Nox sessions:

```python
nox.options.sessions = "lint", "mypy", "tests"
```

Run the Nox session using the following command:

```sh
nox -rs mypy
```

Mypy raises an error if it cannot find any type definitions for a Python package
used by your program. Unless you are going to write these type definitions
yourself (which you can), you should disable the error using the `mypy.ini`
configuration file:

```ini
# mypy.ini
[mypy]

[mypy-nox.*,pytest,pytest_mock]
ignore_missing_imports = True
```

Specifying the packages explicitly helps you keep track which of your
dependencies are currently out of scope of the type checker. You may soon be
able to cut down this list, as many projects are actively working on typing
support.

## Static type checking with pytype

Add [pytype](https://google.github.io/pytype/) as a development dependency:

```sh
poetry add --dev pytype
```

Add the following session to `noxfile.py` to run pytype:

```python
# noxfile.py
@nox.session(python="3.7")
def pytype(session):
    """Run the static type checker."""
    args = session.posargs or ["--disable=import-error", *locations]
    install_with_constraints(session, "pytype")
    session.run("pytype", *args)
```

Run the Nox session using the following command:

```sh
nox -rs pytype
```

The session runs in Python 3.7 only, because Python 3.8 is [not yet
supported](https://github.com/google/pytype/issues/440) in pytype. We also pass
the command-line option `--disable=import-error`. Like mypy, pytype reports
import errors for third-party packages without type annotations.

Update `nox.options.session` to include static type checking with pytype in the
default Nox sessions:

```python
nox.options.sessions = "lint", "mypy", "pytype", "tests"
```

## Adding type annotations to the package

Let's add type annotations to your package! Looking at `console.main`, do not be
distracted by the decorators applied to it: This is just a simple function
accepting a `str`, and returning `None` by "falling off its end":

```python
# src/hypermodern_python/console.py
def main(language: str) -> None:
    ...
```

In the `wikipedia` module, the `API_URL` constant is a string:

```python
API_URL: str = "https://{language}.wikipedia.org/api/rest_v1/page/random/summary"
```

The `wikipedia.random_page` function accepts an optional parameter of type
`str`:

```python
# src/hypermodern_python/wikipedia.py
def random_page(language: str = "en"):
```

The function returns the JSON object received from the Wikipedia API. JSON
objects are represented in Python using built-in types such as `dict`, `list`,
`str`, and `int`, and can be arbitrarily nested. Due to the recursive nature of
JSON objects, their type is still [hard to
express](https://github.com/python/typing/issues/182) in Python, and is usually
given as [Any](https://docs.python.org/3/library/typing.html#the-any-type):

```python
# src/hypermodern_python/wikipedia.py
from typing import Any


def random_page(language: str = "en") -> Any:
```

You can think of the enigmatic
`Any` type as a box
which can hold *any* type on the inside, and behaves like *all* of the types on
the outside. It is the most permissive kind of type you can apply to a variable,
parameter, or return type in your program.

## Runtime type validation using Desert and Marshmallow

Returning `Any` is somewhat unsatisfactory, because we know [quite
precisely](https://en.wikipedia.org/api/rest_v1/#/) what we expect the Wikipedia
API to return. An API contract is not a type guarantee, but you can turn it into
one by validating the JSON you receive. This will also demonstrate some great
ways to use type annotations *at runtime*.

The first step on the road is defining the target type for the validation. The
API returns a dictionary with several keys, of which we are only interested in
`"title"` and `"extract"`. But your program can do better than operating on a
dictionary, so let's turn this into a proper Python object with `title` and
`extract` attributes.

The standard [dataclasses](https://docs.python.org/3/library/dataclasses.html)
module is a great choice for defining the data types of a program. It makes it
easy to define full-featured data types in a concise way, and is an example of
how useful type annotations can be at runtime. Here is the definition of the
`wikipedia.Page` type:

```python
# src/hypermodern_python/wikipedia.py
from dataclasses import dataclass


@dataclass
class Page:
    title: str
    extract: str
```

We are going to return this data type from `wikipedia.random_page`:

```python
# src/hypermodern_python/wikipedia.py
def random_page(language: str = "en") -> Page:
```

The [https://github.com/marshmallow-code/marshmallow](marshmallow) library
allows you to define schemas to serialize, deserialize and validate data. You
use it to define

- marshmallow allows data validation, serialization, deserialization using schemas
- desert generates schemas from type annotations

[marshmallow]() is a library which validates low-level data structures and
converts them to application-level objects, using schemas. [desert]() is a
front-end to marshmallow which uses type annotations to generate schemas
automatically.

Add desert to your dependencies using Poetry:

```sh
poetry add desert
```

You can generate a schema for `Page` like this:

```python
# src/hypermodern_python/wikipedia.py
import desert


schema = desert.schema(Page)
```

Actually, we need to tell schema that it should simply ignore unknown fields:

```python
# src/hypermodern_python/wikipedia.py
import marshmallow


schema = desert.schema(Page, meta={"unknown": marshmallow.EXCLUDE})
```

Using this schema, we can implement `random_page`:

```python
# src/hypermodern_python/wikipedia.py
def random_page(language: str = "en") -> Page:
    url = API_URL.format(language=language)

    try:
        with requests.get(url) as response:
            response.raise_for_status()
            data = response.json()
            return schema.load(data)
    except requests.RequestException as error:
        message = str(error)
        raise click.ClickException(message)
```

Thanks to the data type, we can simplify `console.main`:

```python
# src/hypermodern_python/console.py
import textwrap

import click

from . import __version__, wikipedia


@click.command()
@click.option(
    "--language",
    "-l",
    default="en",
    help="Language edition of Wikipedia",
    metavar="LANG",
    show_default=True,
)
@click.version_option(version=__version__)
def main(language: str) -> None:
    """The hypermodern Python project."""
    page = wikipedia.random_page(language=language)

    click.secho(page.title, fg="green")
    click.echo(textwrap.fill(page.extract))
```

Ignore import errors for `desert` and `marshmallow`:

```ini
# mypy.ini
[mypy-desert,marshmallow,nox.*,pytest,pytest_mock,_pytest.*]
ignore_missing_imports = True
```

We also need to handle validation errors:

```python
# src/hypermodern_python/wikipedia.py
    except (requests.RequestException, marshmallow.ValidationError) as error:
```

TODO test driven

Here is the final `wikipedia` module:

```python
# src/hypermodern_python/wikipedia.py
from dataclasses import dataclass
from typing import Final

import click
import desert
import marshmallow
import requests


API_URL: Final = "https://{language}.wikipedia.org/api/rest_v1/page/random/summary"


@dataclass
class Page:
    title: str
    extract: str


schema = desert.schema(Page, meta={"unknown": marshmallow.EXCLUDE})


def random_page(language: str = "en") -> Page:
    url = API_URL.format(language=language)

    try:
        with requests.get(url) as response:
            response.raise_for_status()
            data = response.json()
            return schema.load(data)
    except (requests.RequestException, marshmallow.ValidationError) as error:
        message = str(error)
        raise click.ClickException(message)
```

## Increasing type coverage with flake8-annotations

[flake8-annotations](https://github.com/python-discord/flake8-annotations) is a
flake8 plugin that detects the absence of type annotations for functions,
helping you keep track of unannotated code.

Install the plugin into the linting session:

```python
# noxfile.py
@nox.session(python=["3.8", "3.7"])
def lint(session):
    args = session.posargs or locations
    session.install(
        "flake8",
        "flake8-annotations",
        "flake8-bandit",
        "flake8-black",
        "flake8-bugbear",
        "flake8-import-order",
    )
    session.run("flake8", *args)
```

Enable the warnings generated by the plugin (`TYP` like *type*):

```ini
# .flake8
[flake8]
select = B,B9,BLK,C,E,F,I,S,TYP,W
```

## Adding type annotations to the test suite

If you run the lint session again, the plugin will spew out a multitude of
warnings about missing type annotations in the Nox sessions and the test suite.
It is possible to disable warnings for these locations using Flake8's
`per-file-ignores` option:

```ini
# .flake8
[flake8]
per-file-ignores =
    tests/*:S101,TYP
    noxfile.py:TYP
```

But don't functions without type annotations look *naked* to you.

Here are type annotations for the Nox sessions:

```python
# noxfile.py
from nox.sessions import Session

def black(session: Session) -> None:

def lint(session: Session) -> None:

def tests(session: Session) -> None:

def coverage(session: Session) -> None:
```

Annotating the test suite requires only a little research:

- The mock objects have the type `unittest.mock.Mock`.
- The mocker fixture has the type `pytest_mock.MockFixture`.
- The runner fixture has the type `click.testing.CliRunner`.

With this out of the way, annotating the test suite becomes pretty straightforward:

```python
# tests/conftest.py
from unittest.mock import Mock

from pytest_mock import MockFixture

def mock_sleep(mocker: MockFixture) -> Mock:
```

```python
# tests/test_console.py
from unittest.mock import Mock

from click.testing import CliRunner
from pytest_mock import MockFixture

def runner() -> CliRunner:

def mock_splines_reticulate(mocker: MockFixture) -> Mock:

def test_main_succeeds(runner: CliRunner, mock_splines_reticulate: Mock) -> None:

def test_main_prints_progress_message(runner: CliRunner, mock_sleep: Mock) -> None:
```

```python
# tests/test_splines.py
from unittest.mock import Mock

def test_reticulate_yields_count_times(mock_sleep: Mock) -> None:

def test_reticulate_sleeps(mock_sleep: Mock) -> None:
```

<center>[Continue to the next chapter](../hypermodern-python-05-documentation)</center>
