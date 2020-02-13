--- 
date: 2020-01-22T10:04:00+01:00
title: "Hypermodern Python Chapter 4: Typing"
description: "A guide to modern Python tooling with a focus on simplicity and minimalism."
tags:
  - python
  - nox
  - mypy
  - pytype
  - flake8
---

[Read this article on Medium](https://medium.com/@cjolowicz/hypermodern-python-4-typing-31bcf12314ff)

{{< figure src="/images/hypermodern-python-04/beard01.jpg" link="/images/hypermodern-python-04/beard01.jpg" >}}

In this fourth installment of the Hypermodern Python series, I'm going to
discuss how to add type annotations and type checking to your project.[^1]
Previously, we discussed [how to add linting, static analysis, and code
formatting](../hypermodern-python-03-linting). (If you start reading here, you
can also
[download](https://github.com/cjolowicz/hypermodern-python/archive/chapter03.zip)
the code for the previous chapter.)

[^1]: The images in this chapter are details from illustrations to the book *A
    Journey in Other Worlds: A Romance of the Future* by John Jacob Astor, 1894
    (source: [The Internet
    Archive](https://archive.org/details/journeyinotherwo00astouoft)). Dan Beard
    later went on to found the Boy Scouts of America, while John Jacob Astor
    built the Astoria Hotel in New York City---predecessor of the
    Waldorf-Astoria Hotel---and perished as the richest man on board the RMS
    Titanic.
    
Here are the topics covered in this chapter on Typing in Python:

- [Type annotations and type checkers](#type-annotations-and-type-checkers)
- [Static type checking with mypy](#static-type-checking-with-mypy)
- [Static type checking with pytype](#static-type-checking-with-pytype)
- [Adding type annotations to the package](#adding-type-annotations-to-the-package)
- [Data validation using Desert and Marshmallow](#data-validation-using-desert-and-marshmallow)
- [Runtime type checking with Typeguard](#runtime-type-checking-with-typeguard)
- [Increasing type coverage with flake8-annotations](#increasing-type-coverage-with-flake8annotations)
- [Adding type annotations to Nox sessions](#adding-type-annotations-to-nox-sessions)
- [Adding type annotations to the test suite](#adding-type-annotations-to-the-test-suite)

Here is a full list of the articles in this series:

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing) (this article)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation)
- [Chapter 6: CI/CD](../hypermodern-python-06-ci-cd)

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Each article in the guide corresponds to a set of commits in the GitHub
repository:

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/chapter03...chapter04)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter04.zip)

## Type annotations and type checkers

{{< figure src="/images/hypermodern-python-04/beard02.jpg" link="/images/hypermodern-python-04/beard02.jpg" >}}

[Type annotations](https://docs.python.org/3/library/typing.html), first
introduced in Python 3.5, are a way to annotate functions and variables with
types. Combined with tooling that understands them, they can make your programs
easier to understand, debug, and maintain. Here are two simple examples of type
annotations:

```python
# This is a variable holding an integer.
answer: int = 42


# This is a function which accepts and returns an integer.
def increment(number: int) -> int:
    return number + 1
```

The Python runtime does not enforce type annotations. Python is a dynamically
typed language: it only verifies the types of your program at runtime, and uses
*duck typing* to do so ("if it walks and quacks like a duck, it is a duck"). A
static type checker, by contrast, can use type annotations and type inference to
verify the type correctness of your program without executing it, helping you
discover many bugs that would otherwise go unnoticed.

The introduction of type annotations has paved the way for an entire generation
of static type checkers: [mypy](http://mypy-lang.org/) can be considered the
pioneer and *de facto* reference implementation of Python type checking; several
core Python developers are involved in its development. Google, Facebook, and
Microsoft have each announced their own static type checkers for Python:
[pytype](https://google.github.io/pytype/), [pyre](https://pyre-check.org/), and
[pyright](https://github.com/microsoft/pyright), respectively. JetBrain's Python
IDE [PyCharm](https://www.jetbrains.com/pycharm/) also ships with a static type
checker. In this chapter, we introduce mypy and pytype. We also take a look at
[Typeguard](https://github.com/agronholm/typeguard), a runtime type checker.

## Static type checking with mypy

{{< figure src="/images/hypermodern-python-04/beard09.jpg" link="/images/hypermodern-python-04/beard09.jpg" >}}

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
yourself, you should disable the error using the `mypy.ini` configuration file:

```ini
# mypy.ini
[mypy]

[mypy-nox.*,pytest]
ignore_missing_imports = True
```

Specifying the packages explicitly helps you keep track of dependencies that are
currently out of scope of the type checker. You may soon be able to cut down on
this list, as many projects are actively working on typing support.

## Static type checking with pytype

{{< figure src="/images/hypermodern-python-04/beard04.jpg" link="/images/hypermodern-python-04/beard04.jpg" >}}

Add [pytype](https://google.github.io/pytype/) as a development dependency, for
[Python 3.7 only](https://github.com/google/pytype/issues/440):


```sh
poetry add --dev --python=3.7 pytype
```

Add the following Nox session to run pytype:

```python
# noxfile.py
@nox.session(python="3.7")
def pytype(session):
    """Run the static type checker."""
    args = session.posargs or ["--disable=import-error", *locations]
    install_with_constraints(session, "pytype")
    session.run("pytype", *args)
```

In this session, we use the command-line option `--disable=import-error` because
pytype, like mypy, reports import errors for third-party packages without type
annotations.

Run the Nox session using the following command:

```sh
nox -rs pytype
```

Update `nox.options.session` to include static type checking with pytype by
default:

```python
nox.options.sessions = "lint", "mypy", "pytype", "tests"
```

## Adding type annotations to the package

{{< figure src="/images/hypermodern-python-04/beard05.jpg" link="/images/hypermodern-python-04/beard05.jpg" >}}

Let's add some type annotations to the package, starting with `console.main`.
Don't be distracted by the decorators applied to it: This is a simple function
accepting a `str`, and returning `None` by "falling off its end":

```python
# src/hypermodern_python/console.py
def main(language: str) -> None: ...
```

In the `wikipedia` module, the `API_URL` constant is a string:

```python
API_URL: str = "https://{language}.wikipedia.org/api/rest_v1/page/random/summary"
```

The `wikipedia.random_page` function accepts an optional parameter of type
`str`:

```python
# src/hypermodern_python/wikipedia.py
def random_page(language: str = "en"): ...
```

The `wikipedia.random_page` function returns the JSON object received from the
Wikipedia API. JSON objects are represented in Python using built-in types such
as `dict`, `list`, `str`, and `int`. Due to the recursive nature of JSON
objects, their type is still [hard to
express](https://github.com/python/typing/issues/182) in Python, and is usually
given as [Any](https://docs.python.org/3/library/typing.html#the-any-type):

```python
# src/hypermodern_python/wikipedia.py
from typing import Any


def random_page(language: str = "en") -> Any: ...
```

You can think of the enigmatic `Any` type as a box which can hold *any* type on
the inside, but behaves like *all* of the types on the outside. It is the most
permissive kind of type you can apply to a variable, parameter, or return type
in your program. Contrast this with `object`, which can also hold values of any
type, but only supports the minimal interface that is common to all of them.

## Data validation using Desert and Marshmallow

{{< figure src="/images/hypermodern-python-04/beard06.jpg" link="/images/hypermodern-python-04/beard06.jpg" >}}

Returning `Any` is unsatisfactory, because we know quite precisely [which JSON
structures we can expect](https://en.wikipedia.org/api/rest_v1/#/) from the
Wikipedia API. An API contract is not a type guarantee, but you can turn it into
one by validating the data you receive. This will also demonstrate some great
ways to use type annotations *at runtime*.

The first step is to define the target type for the validation. Currently, the
function should return a dictionary with several keys, of which we are only
interested in `title` and `extract`. But your program can do better than
operating on a dictionary: Using
[dataclasses](https://docs.python.org/3/library/dataclasses.html) from the
standard library, you can define a fully-featured data type in a concise and
straightforward manner. Let's define a `wikipedia.Page` type for our
application:

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
def random_page(language: str = "en") -> Page: ...
```

Data types have a beneficial influence on the structure of code bases. You can
see this by adapting `console.main` to use `wikipedia.Page` instead of the raw
dictionary. The body turns into a clear and concise three-liner:

```python
# src/hypermodern_python/console.py
def main(language: str) -> None:
    """The hypermodern Python project."""
    page = wikipedia.random_page(language=language)

    click.secho(page.title, fg="green")
    click.echo(textwrap.fill(page.extract))
```

Of course, we are still missing the actual code to create `wikipedia.Page`
objects. Let's start with a test case, performing a simple runtime type check:

```python
# tests/test_wikipedia.py
def test_random_page_returns_page(mock_requests_get):
    page = wikipedia.random_page()
    assert isinstance(page, wikipedia.Page)
```

How are we going to turn JSON into `wikipedia.Page` objects? Enter
[Marshmallow](https://marshmallow.readthedocs.io/): This library allows you to
define schemas to serialize, deserialize and validate data. Used by countless
applications, Marshmallow has also spawned an ecosystem of tools and libraries
built on top of it. One of these libraries,
[Desert](https://desert.readthedocs.io/), uses the type annotations of
dataclasses to generate serialization schemas for them. (Another great data
validation library using type annotations is
[typical](https://python-typical.org/).)

Add Desert and Marshmallow to your dependencies:

```sh
poetry add desert marshmallow
```

You need to add the libraries to `mypy.ini` to avoid import errors:

```ini
# mypy.ini
[mypy-desert,marshmallow,nox.*,pytest]
ignore_missing_imports = True
```

The basic usage of Desert is shown in the following example:

```python
# Generate a schema from a dataclass.
schema = desert.schema(Page)

# Use the schema to load data.
page = schema.load(data)
```

Our data type represents the Wikipedia page resource only partially. Marshmallow
errs on the side of safety and raises a validation error when encountering
unknown fields. However, you can tell it to ignore unknown fields via the `meta`
keyword. Add the schema as a module-level variable: 

```python
# src/hypermodern_python/wikipedia.py
import desert
import marshmallow


schema = desert.schema(Page, meta={"unknown": marshmallow.EXCLUDE})
```

Using the schema, we are ready to implement `wikipedia.random_page`:

```python
# src/hypermodern_python/wikipedia.py
def random_page(language: str = "en") -> Page:
    ...
    with requests.get(url) as response:
        response.raise_for_status()
        data = response.json()
        return schema.load(data)
```

Let's also handle validation errors gracefully by converting them to
`ClickException`, as we did in [chapter
2](../hypermodern-python-02-testing#example-cli-handling-exceptions-gracefully)
for request errors. For example, suppose that Wikipedia responds with a body of
"null" instead of an actual resource, due to a fictitious bug.

We can turn this scenario into a test case by configuring the `requests.get`
mock to produce `None` as the JSON object, and using
[pytest.raises](https://docs.pytest.org/en/stable/reference.html#pytest-raises)
to check for the correct exception:

```python
# tests/test_wikipedia.py
def test_random_page_handles_validation_errors(mock_requests_get: Mock) -> None:
    mock_requests_get.return_value.__enter__.return_value.json.return_value = None
    with pytest.raises(click.ClickException):
        wikipedia.random_page()
```

Getting this to pass is a simple matter of adding `marshmallow.ValidationError`
to the except clause:

```python
# src/hypermodern_python/wikipedia.py
except (requests.RequestException, marshmallow.ValidationError) as error:
```

This completes the implementation. Here is the `wikipedia` module with type
annotations and validation:

```python
# src/hypermodern_python/wikipedia.py
from dataclasses import dataclass

import click
import desert
import marshmallow
import requests


API_URL: str = "https://{language}.wikipedia.org/api/rest_v1/page/random/summary"


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

## Runtime type checking with Typeguard

{{< figure src="/images/hypermodern-python-04/beard10.jpg" link="/images/hypermodern-python-04/beard10.jpg" >}}

[Typeguard](https://github.com/agronholm/typeguard) is a runtime type checker
for Python: It checks that arguments match parameter types of annotated
functions as your program is being executed (and similarly for return values).
Runtime type checking can be a valuable tool when it is impossible or
impractical to strictly type an entire code path, for example when crossing
system boundaries or interfacing with other libraries. 

Here is an example of a type-related bug which can be caught by a runtime type
checker, but is not detected by mypy or pytype because the incorrectly typed
argument is loaded from JSON. (Do not add this test function to your test suite
permanently; it's for demonstration purposes only.)

```python
# tests/test_wikipedia.py
def test_trigger_typeguard(mock_requests_get):
    import json

    data = json.loads('{ "language": 1 }')
    wikipedia.random_page(language=data["language"])
```

Add Typeguard to your development dependencies:

```sh
poetry add --dev typeguard
```

Typeguard comes with a Pytest plugin which instruments your package for type
checking while you run the test suite. You can enable it by passing the
`--typeguard-packages` option with the name of your package. Add a Nox session
to run the test suite with runtime type checking:

```python
# noxfile.py
package = "hypermodern_python"


@nox.session(python=["3.8", "3.7"])
def typeguard(session):
    args = session.posargs or ["-m", "not e2e"]
    session.run("poetry", "install", "--no-dev", external=True)
    install_with_constraints(session, "pytest", "pytest-mock", "typeguard")
    session.run("pytest", f"--typeguard-packages={package}", *args)
```

Run the Nox session:

```sh
nox -rs typeguard
```

Typeguard catches the bug we introduced at the start of this section. You will
also notice a warning about missing type annotations for a Click object. This is
due to the fact that `console.main` is wrapped by a decorator, and its type
annotations only apply to the inner function, not the resulting object as seen
by the test suite.

## Increasing type coverage with flake8-annotations

{{< figure src="/images/hypermodern-python-04/beard07.jpg" link="/images/hypermodern-python-04/beard07.jpg" >}}

[flake8-annotations](https://github.com/python-discord/flake8-annotations) is a
Flake8 plugin that detects the absence of type annotations for functions,
helping you keep track of unannotated code.

Add the plugin to your development dependencies:

```sh
poetry add --dev flake8-annotations
```

Install the plugin into the linting session:

```python
# noxfile.py
@nox.session(python=["3.8", "3.7"])
def lint(session):
    args = session.posargs or locations
    install_with_constraints(
        session,
        "flake8",
        "flake8-annotations",
        "flake8-bandit",
        "flake8-black",
        "flake8-bugbear",
        "flake8-import-order",
    )
    session.run("flake8", *args)
```

Configure Flake8 to switch on the warnings generated by the plugin (`ANN` like
*annotations*):

```ini
# .flake8
[flake8]
select = ANN,B,B9,BLK,C,E,F,I,S,W
```

Run the lint session:

```sh
nox -rs lint
```

The plugin spews out a multitude of warnings about missing type annotations in
the Nox sessions and the test suite. It is possible to disable warnings for
these locations using Flake8's `per-file-ignores` option:

```ini
# .flake8
[flake8]
per-file-ignores =
    tests/*:S101,TYP
    noxfile.py:TYP
```

## Adding type annotations to Nox sessions

{{< figure src="/images/hypermodern-python-04/beard03.jpg" link="/images/hypermodern-python-04/beard03.jpg" >}}

<!--
Identifying the types associated with a third-party package still requires a
little research sometimes. If the project does not document its types, you may
need to take a look at their source code. For some packages, type annotations
are distributed separately, in the community-driven
[typeshed](https://github.com/python/typeshed) or in a package named `foo-stubs`
(where `foo` is the original package).
-->

In this section, I show you how to add type annotations to Nox sessions. If you
disabled type coverage warnings (`TYP`) for Nox sessions, re-enable them for the
purposes of this section.

The central type for Nox sessions is `nox.sessions.Session`, which is also the
first and only argument of your session functions. The return value of these
functions is `None`---the implicit return value of every Python function that
does not explicitly return anything. Annotate your session functions like this:

```python
# noxfile.py
from nox.sessions import Session


def black(session: Session) -> None: ...

def lint(session: Session) -> None: ...

def safety(session: Session) -> None: ...

def mypy(session: Session) -> None: ...

def pytype(session: Session) -> None: ...

def tests(session: Session) -> None: ...

def typeguard(session: Session) -> None: ...
```

Our helper function `install_with_constraints` is a wrapper for
`Session.install`. The positional arguments of this function are the
command-line arguments for [pip](https://pip.pypa.io/), so their type is `str`.
The keyword arguments are the same as for [Session.run](
https://nox.thea.codes/en/stable/config.html#nox.sessions.Session.run). Instead
of listing their types individually, we'll be pragmatic and type them as `Any`:

```python
# noxfile.py
def install_with_constraints(session: Session, *args: str, **kwargs: Any) -> None: ...
```

## Adding type annotations to the test suite

{{< figure src="/images/hypermodern-python-04/beard08.jpg" link="/images/hypermodern-python-04/beard08.jpg" >}}

In this section, I show you how to add type annotations to the test suite. If
you disabled type coverage warnings (`TYP`) for the test suite, re-enable them
for the purposes of this section.

Test functions in Pytest use parameters to declare fixtures they use. Typing
them is simpler than it may seem: You don't specify the type of the fixture
itself, but the type of the value that the fixture supplies to the test
function. For example, the `mock_requests_get` fixture returns a standard mock
object of type
[unittest.mock.Mock](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock).
(The actual type is
[unittest.mock.MagicMock](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.MagicMock),
but you can use the more general type to annotate your test functions.)

Let's start by annotating the test functions in the `wikipedia` test module:

```python
# tests/test_wikipedia.py
from unittest.mock import Mock


def test_random_page_uses_given_language(mock_requests_get: Mock) -> None: ...

def test_random_page_returns_page(mock_requests_get: Mock) -> None: ...

def test_random_page_handles_validation_errors(mock_requests_get: Mock) -> None: ...
```

Next, let's annotate `mock_requests_get` itself. The return type of this
function is the same `unittest.mock.Mock`. The function takes a single argument,
the `mocker` fixture from
[pytest-mock](https://github.com/pytest-dev/pytest-mock), which is of type
`pytest_mock.MockFixture`:

```python
# tests/conftest.py
from unittest.mock import Mock

from pytest_mock import MockFixture


def mock_requests_get(mocker: MockFixture) -> Mock: ...
```

Configure mypy to ignore the missing import for `pytest_mock`:

```ini
# mypy.ini
[mypy-desert,marshmallow,nox.*,pytest,pytest_mock]
ignore_missing_imports = True
```

Use the same type signature for the mock fixture in the `console` test module:

```python
# tests/test_console.py
from unittest.mock import Mock

from pytest_mock import MockFixture


def mock_wikipedia_random_page(mocker: MockFixture) -> Mock: ...
```

The test module for `console` also defines a simple fixture returning a
`click.testing.CliRunner`:

```python
# tests/test_console.py
from click.testing import CliRunner


def runner() -> CliRunner: ...
```

Typing the test functions for `console` is now straightforward:

```python
# tests/test_console.py
from unittest.mock import Mock

from click.testing import CliRunner
from pytest_mock import MockFixture


def test_main_succeeds(runner: CliRunner, mock_requests_get: Mock) -> None: ...

def test_main_succeeds_in_production_env(runner: CliRunner) -> None: ...

def test_main_prints_title(runner: CliRunner, mock_requests_get: Mock) -> None: ...

def test_main_invokes_requests_get(runner: CliRunner, mock_requests_get: Mock) -> None: ...

def test_main_uses_en_wikipedia_org(runner: CliRunner, mock_requests_get: Mock) -> None: ...

def test_main_uses_specified_language(runner: CliRunner, mock_wikipedia_random_page: Mock) -> None: ...

def test_main_fails_on_request_error(runner: CliRunner, mock_requests_get: Mock) -> None: ...

def test_main_prints_message_on_request_error(runner: CliRunner, mock_requests_get: Mock) -> None: ...
```

The missing piece to achieve full type coverage (as far as the Flake8
annotations plugin is concerned) is the `pytest_configure` hook. The function
takes a Pytest configuration object as its single parameter. Unfortunately, the
type of this object is not
([yet](https://github.com/pytest-dev/pytest/issues/3342)) part of Pytest's
public interface. You have the choice of using `Any` or reaching into Pytest's
internals to import `_pytest.config.Config`. Let's opt for the latter:

```python
# tests/conftest.py
from _pytest.config import Config


def pytest_configure(config: Config) -> None: ...
```

You also need to tell mypy to ignore missing imports for `_pytest.*`:

```ini
# mypy.ini
[mypy-desert,marshmallow,nox.*,pytest,pytest_mock,_pytest.*]
ignore_missing_imports = True
```

This concludes our foray into the Python type system. Type annotations make your
programs easier to understand, debug, and maintain. Static type checkers use
type annotations and type inference to verify the type correctness of your
program without executing it, helping you discover many bugs that would
otherwise go unnoticed. An increasing number of libraries leverage the power of
type annotations at runtime, for example to validate data. Happy typing!

## Thanks for reading!

The next chapter is about adding documentation for your project.

{{< figure src="/images/hypermodern-python-04/train.jpg" link="../hypermodern-python-05-documentation" class="centered" >}}
<span class="centered">[Continue to the next chapter](../hypermodern-python-05-documentation)</span>
