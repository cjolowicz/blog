--- 
date: 2019-11-07T12:52:59+02:00
title: "The hypermodern Python project, pt. 3"
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

In this third installment of the Hypermodern Python series, I'm going to
discuss how to add static type checking to your project.

For your reference, below is a list of the articles in this series.

- Chapter 1: Basics
- Chapter 2: Documentation
- Chapter 3: Type Checking

<!--
This post has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Reticulating splines](#reticulating-splines)
- [Mocking with pytest-mock](#mocking-with-pytest-mock)
- [Static type checking with pytype](#static-type-checking-with-pytype)
- [Increasing type coverage with flake8-annotations](#increasing-type-coverage-with-flake8-annotations)
- [Linting import order with flake8-import-order](#linting-import-order-with-flake8-import-order)
- [Linting with flake8-bugbear](#linting-with-flake8-bugbear)
- [Finding security issues with bandit](#finding-security-issues-with-bandit)
- [Conclusion](#conclusion)

<!-- markdown-toc end -->

## Reticulating splines

The Hypermodern Python package doesn't do anything useful yet. Let's change this
and add a module for [reticulating
splines](https://www.quora.com/What-does-reticulating-splines-mean):

```python
# src/hypermodern_python/splines.py
import time


def reticulate(count = -1):
    spline = 0
    while count < 0 or spline < count:
        time.sleep(1)
        spline += 1
        yield spline
```

This `splines` module defines a single function `splines.reticulate`, which
yields an infinite number of splines, or the number of splines specified by the
`count` parameter. Each spline is reticulated in a time-consuming process.

Let's make this exciting functionality accessible from the command-line
interface:

```python
# src/hypermodern_python/console.py
import click
 
from . import __version__, splines
 
 
@click.command()
@click.version_option(version=__version__)
def main():
    """The hypermodern Python project."""
    for spline in splines.reticulate():
        click.echo(f"Reticulating spline {spline}...")
```

For users with less than an infinite amount of time on their hands, add an
option to control the number of splines to reticulate:

```python
# src/hypermodern_python/console.py
...
@click.command()
@click.option("-n", "--count", default=-1, help="Number of splines to reticulate")
@click.version_option(version=__version__)
def main(count):
    """The hypermodern Python project."""
    for spline in splines.reticulate(count):
        click.echo(f"Reticulating spline {spline}...")
```

Add a test case for the `splines` module:

```python
# tests/test_splines.py
from hypermodern_python import splines


def test_reticulate_yields_count_times():
    iterator = splines.reticulate(2)
    assert sum(1 for _ in iterator) == 2
```

Add a test for the new feature in the `console` module:

```python
# tests/test_console.py
def test_main_prints_progress_message(runner, mock_sleep):
    result = runner.invoke(console.main, ["--count=1"])
    assert result.output == "Reticulating spline 1...\n"
```

## Mocking with pytest-mock

Reticulating is a complicated and time-consuming process. As a consequence, the
unit tests are getting rather slow.

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

## Static type checking with pytype

With the advent of [type
annotations](https://docs.python.org/3/library/typing.html), several static type
checkers for Python have come into existence: [mypy](http://mypy-lang.org/),
Google's [pytype](https://google.github.io/pytype/), and Microsoft's
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

With this in place, the static checker should be happy with the current state of
affairs:

```sh
nox -rs pytype
```

While pytype uses type inference to validate Python code even in the absence of
type annotations, it can clearly perform a much better job if you add type
information to your code base.

Add type annotations to the `splines` module:

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

Add type annotations to the `console` module:

```python
# src/hypermodern_python/console.py
def main(count: int) -> None:
```

## Increasing type coverage with flake8-annotations

[flake8-annotations](https://github.com/python-discord/flake8-annotations) is a
flake8 plugin that detects the absence of type annotations for functions.

Install the plugin into the linting session:

```python
# noxfile.py
...
def lint(session):
    ...
    session.install("flake8", "flake8-black", "flake8-annotations")
    ...
```

Enable the warnings generated by the plugin, which are prefixed by `TYP`:

```ini
# .flake8
select = BLK,C,E,F,TYP,W
```

If you run `nox -rs lint` again, the plugin will spew out a multitude of
warnings about functions lacking type information in `noxfile.py` and the test
suite.

You can disable warnings for these locations as follows:

Alternatively, if you prefer to annotate the Nox sessions and test suite, this
is how you would do it:

```python
# noxfile.py
def black(session: nox.sessions.Session) -> None:
def lint(session: nox.sessions.Session) -> None:
def tests(session: nox.sessions.Session) -> None:
def coverage(session: nox.sessions.Session) -> None:

# tests/conftest.py
import unittest.mock
import pytest_mock
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

## Linting import order with flake8-import-order

The [flake8-import-order](https://github.com/PyCQA/flake8-import-order) plugin
checks whether the order of import statements is consistent and [PEP
8](https://www.python.org/dev/peps/pep-0008/#imports)-compliant. Install the
plugin in the linter session:

```python
# noxfile.py
...
def lint(session):
    ...
    session.install("flake8", "flake8-black", "flake8-annotations", "flake8-import-order")
    ...
```

Enable the warnings emitted by the plugin, which are prefixed by `I`. Also
inform the plugin about the local package name, which affects sorting order:

```ini
# .flake8
[flake8]
select = BLK,C,E,F,I,TYP,W
...
application-import-names = hypermodern_python,tests
```

## Linting with flake8-bugbear

The [flake8-bugbear](https://pypi.org/project/flake8-bugbear/) plugin helps you
finding bugs and design problems in your program. Add the plugin to the linter
session in your `noxfile.py`:

```python
# noxfile.py
...
def lint(session):
    ...
    session.install(
        "flake8",
        "flake8-annotations",
        "flake8-black",
        "flake8-bugbear",
        "flake8-import-order",
    )
    ...
```

Enable Bugbear's warnings in the `.flake8` configuration file. These warnings are
prefixed with `B`:

```ini
# .flake8
[flake8]
select = B,B9,BLK,C,E,F,I,TYP,W
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

## Finding security issues with bandit

[bandit](https://github.com/PyCQA/bandit): Find common security issues in Python code

https://github.com/tylerwince/flake8-bandit: Automated security testing using
bandit and flake8

## Conclusion

<!--
{{< figure src="http://www.vintagecomputer.net/ctc/3300/CTC_DataPoint-3300_pic3.jpg" caption="Fun fact: Consoles have supported dark mode since 1969, exactly half a century before iOS 13." alt="DataPoint 3300 (1969)" link="https://www.youtube.com/watch?v=dEGlKpIBujc" width="80%" class="centered" >}}
-->
