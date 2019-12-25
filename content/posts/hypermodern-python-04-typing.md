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
discuss how to add static type checking to your project.[^1] Previously, we
discussed [how to add linting, static analysis, and code
formatting](../hypermodern-python-03-linting).

[^1]: The images in this chapter are details from illustrations to the book *A
    Journey in Other Worlds: A Romance of the Future* by John Jacob Astor, 1894
    (source: [The Internet
    Archive](https://archive.org/details/journeyinotherwo00astouoft)).

Here are the topics covered in this chapter on Typing in Python:

- [Static type checkers and type annotations](#static-type-checkers-and-type-annotations)
- [Static type checking with mypy](#static-type-checking-with-mypy)
- [Static type checking with pytype](#static-type-checking-with-pytype)
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

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

<!-- markdown-toc end -->

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Here is the link for the changes contained in this chapter:

▶ **[View code](https://github.com/cjolowicz/hypermodern-python/compare/chapter03...chapter04)**

## Static type checkers and type annotations

Static type checkers analyze your source code for type errors (like treating a
string as an integer) without executing it. While Python is a dynamically typed
language--it verifies the types of your program at runtime--, Python 3.5 and 3.6
have introduced a way to [annotate functions and variables with
types](https://docs.python.org/3/library/typing.html). Type annotations make
your programs easier to understand, debug, and maintain.

Type annotations are entirely optional: They do not interfere when running your
program, and type checkers do not require your code to be fully annotated. Some
type checkers are able to infer types for unannotated code, but all support the
coexistence of dynamically and statically typed Python.

The introduction of type annotations has paved the way for an entire generation
of static type checkers: [mypy](http://mypy-lang.org/) can be considered the
pioneer and *de facto* reference implementation of Python type checking; several
core Python developers are involved in its development. Google, Facebook, and
Microsoft have each announced their own static type checkers for Python:
[pytype](https://google.github.io/pytype/), [pyre](https://pyre-check.org/), and
[pyright](https://github.com/microsoft/pyright), respectively. JetBrain's Python
IDE [PyCharm](https://www.jetbrains.com/pycharm/) also ships with a static type
checker.

Let's add type annotations to the `console.main` function. This is a simple
function which accepts an `int` and returns `None` by falling off its end:

```python
# src/hypermodern_python/console.py
def main(count: int) -> None:
    ...
```

The `splines.reticulate` function is a more complex example: It accepts an
optional parameter of type `int` and yields values of type `int`, in other
words, it returns a generator. You can express this using the `typing.Iterator`
type, as shown below. The function body contains a local variable named
`spline`, which is also an `int`.

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

## Static type checking with mypy

Add the following session to `noxfile.py` to type-check your source code with
[mypy](http://mypy-lang.org/):

```python
@nox.session(python=["3.8", "3.7"])
def mypy(session: Session) -> None:
    args = session.posargs or locations
    session.install("mypy")
    session.run("mypy", *args)
```

Include mypy in the default Nox sessions:

```python
nox.options.sessions = "lint", "mypy", "tests"
```

mypy generates warnings if it does not find types for a third-party package. You
can ignore these errors globally using the `mypy.ini` configuration file:

```ini
# mypy.ini
[mypy]
ignore_missing_imports = True
```

Even better, you can disable the warning for specific packages only, as shown
below. This helps you keep track which of your dependencies are currently out of
scope of the type checker:

```ini
# mypy.ini
[mypy]

[mypy-nox.*,pytest,pytest_mock]
ignore_missing_imports = True
```

## Static type checking with pytype

Google's [pytype](https://google.github.io/pytype/) supports type inference,
giving you some of the benefits of static typing even before you start adding
type annotations to your code. Add the following session to `noxfile.py` to run
pytype:

```python
# noxfile.py
@nox.session(python="3.7")
def pytype(session):
    """Run the static type checker."""
    args = session.posargs or locations
    session.install("pytype")
    session.run("pytype", *args)
```

The session runs in Python 3.7 only, because Python 3.8 is [not yet
supported](https://github.com/google/pytype/issues/440) in pytype.

Like mypy, pytype reports import errors for third-party packages without type
annotations. You could add a `# pytype: disable=import-error` comment to imports
of offending packages, but this can get pretty repetitive. Instead, create the
`pytype.cfg` configuration file and disable import errors globally:

```ini
# pytype.cfg
[pytype]
disable = import-error
```

You need to specify the configuration file explicitly when invoking pytype:

```python
# noxfile.py
@nox.session(python="3.7")
def pytype(session):
    """Run the static type checker."""
    args = session.posargs or locations
    session.install("pytype")
    session.run("pytype", "--config=pytype.cfg", *args)
```

With this in place, the static checker should be happy with the current state of
affairs:

```sh
nox -rs pytype
```

Update `nox.options.session` to include static type checking with pytype in the
default Nox sessions:

```python
nox.options.sessions = "lint", "mypy", "pytype", "tests"
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
