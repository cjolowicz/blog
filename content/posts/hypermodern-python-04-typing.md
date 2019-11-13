--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 4: Typing"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - mytype
  - pytype
---

In this fourth installment of the Hypermodern Python series, I'm going to
discuss how to add static type checking to your project.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation)
- [Chapter 6: CI/CD](../hypermodern-python-05-ci-cd)

<!--
This post has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Static typing and Python](#static-typing-and-python)
- [Static type checking with pytype](#static-type-checking-with-pytype)
- [Adding type annotations to the package](#adding-type-annotations-to-the-package)
- [Increasing type coverage with flake8-annotations](#increasing-type-coverage-with-flake8-annotations)
- [Adding type annotations to the test suite](#adding-type-annotations-to-the-test-suite)
- [Static type checking with mypy](#static-type-checking-with-mypy)
- [Static type checking with pyre](#static-type-checking-with-pyre)

<!-- markdown-toc end -->

## Static typing and Python

With the advent of [type
annotations](https://docs.python.org/3/library/typing.html), several static type
checkers for Python have come into existence:

- Jukka Lehtosalo's [mypy](http://mypy-lang.org/) (2012)
- JetBrain's [PyCharm](https://www.jetbrains.com/pycharm/) (2015)
- Google's [pytype](https://google.github.io/pytype/) (2016)
- Facebook's [pyre](https://pyre-check.org/) (2018)
- Microsoft's [pyright](https://github.com/microsoft/pyright) (2019)

Pyright is written in JavaScript and may be interesting for you if you use the
Microsoft Visual Code editor, but I won't cover it here.

## Static type checking with pytype

Google's [pytype](https://google.github.io/pytype/) is able to [infer types for
unannotated
code](https://docs.google.com/presentation/d/1GYqLeLkknjYaYX2JrMzxX8LGw_rlO-6kTk-VNPVG9gY/edit?usp=sharing),
giving you some of the benefits of type checking at minimal cost. Pytype's type
inference makes it a nice tool to start with. Add the following session to
`noxfile.py` to add run pytype:

```python
# noxfile.py
@nox.session(python="3.7")
def pytype(session):
    """Run the static type checker."""
    args = session.posargs or locations
    session.install("pytype")
    session.run("pytype", *args)
```

This session runs in Python 3.7 only, because Python 3.8 is [not yet
supported](https://github.com/google/pytype/issues/440) in pytype.

Some third-party packages are distributed without type annotations, resulting in
import errors reported by pytype. You could add a `# pytype:
disable=import-error` comment to the `import` statements of offending packages,
but this can get pretty repetitive. Instead, create the `pytype.cfg`
configuration file and disable import errors globally:

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

Update `nox.options.session` to include static type checking in the default Nox
sessions:

```python
nox.options.sessions = "lint", "pytype", "tests"
```

## Adding type annotations to the package

While pytype uses type inference to validate Python code even in the absence of
type annotations, it can clearly perform a much better job if you add [type
annotations](https://docs.python.org/3/library/typing.html) to your codebase.

The `splines.reticulate` function accepts an optional `int`, and yields values
of type `int`. The local variable `spline` is also an `int`.

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

The `console.main` function looks scary with its many decorators. But at its
core, it is a simple function accepting an `int` and returning `None`.

```python
# src/hypermodern_python/console.py
def main(count: int) -> None:
    ...
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

## Static type checking with mypy

Jukka Lehtosalo's [mypy](http://mypy-lang.org/) may be considered the pioneer of
Python type checking. Its core team is employed by Dropbox and includes or has
included several Python core developers, such as a certain Guido van Rossum. By
contrast with pytype, mypy enables gradual adoption by checking only annotated
code.

Add the following session to `noxfile.py` to run mypy:

```python
@nox.session(python=["3.8", "3.7"])
def mypy(session: Session) -> None:
    args = session.posargs or locations
    session.install("mypy")
    session.run("mypy", *args)
```

Include mypy in the default Nox sessions:

```python
nox.options.sessions = "lint", "mypy", "pytype", "tests"
```

Like pytype, mypy generates warnings every time it does not find types for a
third-party package. You can ignore these errors globally using the `mypy.ini`
configuration file:

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

## Static type checking with pyre

Facebook's [pyre](https://pyre-check.org/) supports incremental checking with
[Watchman](https://facebook.github.io/watchman/) and, like pytype, type
inference and stub generation.

<center>[Continue to the next chapter](../hypermodern-python-05-documentation)</center>
