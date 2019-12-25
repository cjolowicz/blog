--- 
date: 2019-01-08T08:00:00+02:00
title: "Hypermodern Python 2: Testing"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - poetry
  - nox
  - pytest
  - coverage.py
  - pytest-mock
---

In this second installment of the Hypermodern Python series, I'm going to
discuss how to add unit tests to your project.[^1]

[^1]: The images in this chapter come from Émile-Antoine Bayard's Illustrations
    for From the Earth to the Moon (De la terre à la lune) by Jules Verne (1870)
    (source: [Internet
    Archive](https://archive.org/details/delaterrelalu00vern))

Here are the topics covered in this chapter:

- [Unit testing with pytest](#unit-testing-with-pytest)
- [Code coverage with coverage.py](#code-coverage-with-coveragepy)
- [Test automation with Nox](#test-automation-with-nox)
- [Mocking with pytest-mock](#mocking-with-pytest-mock)
- [More mocking with pytest-mock](#more-mocking-with-pytest-mock)

Here is a list of the articles in this series:

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing) (this article)
- *Chapter 3: Linting*
- *Chapter 4: Typing*
- *Chapter 5: Documentation*
- *Chapter 6: CI/CD*
- *Appendix: Docker*

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Each article in the guide corresponds to a set of commits in the GitHub
repository.

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/initial...chapter01)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter01.zip)

## Unit testing with pytest

It's never too early to add unit tests to a project. 

Unit tests, as the name says, verify the functionality of a *unit of code*, such
as a single function or class. While the
[unittest](https://docs.python.org/3/library/unittest.html) framework is part of
the Python standard library, [pytest](https://docs.pytest.org/en/latest/) has
become somewhat of a *de facto* standard.

Let's add this package as a development dependency, using Poetry's `--dev`
option:

```sh
poetry add --dev pytest
```

Organize tests in a [separate file
hierarchy](http://doc.pytest.org/en/latest/goodpractices.html#tests-outside-application-code)
next to `src`, named `tests`:

```sh
.
├── src
└── tests
    ├── __init__.py
    └── test_console.py

2 directories, 2 files
```

The file `__init__.py` is empty and serves to declare the test suite as a
package. While this is not strictly necessary, it allows your test suite to
mirror the source layout of the package under test, even when modules in
different parts of the source tree [have the same
name](https://github.com/pytest-dev/pytest/issues/3151). Furthermore, it gives
you the option to import modules from within your tests package.

The file `test_console.py` contains a test case for the `console` module,
checking if the program exits with a status code of zero.

```python
# tests/test_console.py
import click.testing

from hypermodern_python import console


def test_main_succeeds():
    runner = click.testing.CliRunner()
    result = runner.invoke(console.main)
    assert result.exit_code == 0
```

The `CliRunner` is needed to invoke the command-line interface from within a
test case. Since this is likely to be needed by most test cases in this module,
let's turn it into a *test fixture*. Test fixtures are declared as simple
functions with the `pytest.fixture` decorator. Test cases can use a test fixture
by including a function parameter with the same name as the test fixture.

```python
# tests/test_console.py
import click.testing
import pytest

from hypermodern_python import console


@pytest.fixture
def runner():
    return click.testing.CliRunner()


def test_main_succeeds(runner):
    result = runner.invoke(console.main)
    assert result.exit_code == 0
```

Invoke `pytest` to run the test suite:

```python
$ poetry run pytest
============================ test session starts =============================
platform linux -- Python 3.8.1, pytest-5.3.2, py-1.8.0, pluggy-0.13.1
rootdir: /hypermodern-python
collected 1 item

tests/test_console.py .                                                 [100%]

============================= 1 passed in 0.03s ==============================
```

## Code coverage with coverage.py

*Code coverage* is a measure of the degree to which the source code of your
program is executed while running its test suite. The code coverage of Python
programs can be determined using a tool called
[Coverage.py](https://coverage.readthedocs.io/). Install it together with the
[pytest-cov](https://pytest-cov.readthedocs.io/en/latest/) plugin, which
integrates Coverage.py with `pytest`:

```sh
poetry add --dev coverage[toml] pytest-cov
```

Update the `pyproject.toml` configuration file to inform the tool about your
source tree layout. The configuration also enables branch analysis and the
display of line numbers for missing coverage.

```toml
# pyproject.toml
[tool.coverage.paths]
source = ["src", "*/site-packages"]

[tool.coverage.run]
branch = true
source = ["hypermodern_python"]

[tool.coverage.report]
show_missing = true
```

To enable coverage reporting, invoke `pytest` with the `--cov` option:

```python
$ poetry run pytest -- --cov
============================= test session starts ==============================
platform linux -- Python 3.8.1, pytest-5.3.2, py-1.8.0, pluggy-0.13.1
rootdir: /hypermodern-python
plugins: cov-2.8.1
collected 1 item

tests/test_console.py .                                                 [100%]

--------------- coverage: platform linux, python 3.8.1-final-0 -----------------
Name                                 Stmts   Miss Branch BrPart  Cover   Missing
--------------------------------------------------------------------------------
src/hypermodern_python/__init__.py       1      0      0      0   100%
src/hypermodern_python/console.py        6      0      0      0   100%
--------------------------------------------------------------------------------
TOTAL                                    7      0      0      0   100%
============================== 1 passed in 0.09s ===============================
```

The reported code coverage is 100%. This number does not imply that your test
suite has meaningful test cases for all uses and misuses of your program.
However, aiming for 100% code coverage is a good practice, especially for a
fresh codebase. Later, we will see some tools that help you achieve this goal.

## Test automation with Nox

One of my personal favorites, [nox](https://nox.thea.codes/) is a successor to
the venerable [tox](https://tox.readthedocs.io/). At its core, the tool
automates testing in multiple Python environments. It makes it easy to run any
kind of job in an isolated environment, with only those dependencies installed
that the particular job needs.

Install Nox via [pip](https://pip.readthedocs.org/) or
[pipx](https://github.com/pipxproject/pipx):

```sh
pip install --user --upgrade nox
```

Unlike tox, Nox uses a standard Python file for configuration:

```python
# noxfile.py
import nox


@nox.session(python=["3.8", "3.7"])
def tests(session):
    session.run("poetry", "install", external=True)
    session.run("pytest", "--cov")
```

This file defines a session named `tests`, which installs the project
dependencies and runs the test suite. Poetry is not a part of the environment
created by Nox, so we specify `external` to avoid warnings about external
commands leaking into the isolated test environments.

Nox creates virtual environments for the listed Python versions (3.8 and 3.7),
and runs the session inside each environment:

```python
$ nox

nox > Running session tests-3.8
nox > Creating virtual environment (virtualenv) using python3.8 in .nox/tests-3-8
nox > poetry install
...
nox > pytest --cov
...
nox > Session tests-3.8 was successful.
nox > Running session tests-3.7
nox > Creating virtual environment (virtualenv) using python3.7 in .nox/tests-3-7
nox > poetry install
...
nox > pytest --cov
...
nox > Session tests-3.7 was successful.
nox > Ran multiple sessions:
nox > * tests-3.8: success
nox > * tests-3.7: success
```

Nox recreates the virtual environments from scratch on each invocation (a
sensible default). You can speed things up by passing the
[-\-reuse-existing-virtualenvs
(-r)](https://nox.thea.codes/en/stable/usage.html#re-using-virtualenvs) option:

```sh
nox -r
```

Sometimes, you need to pass additional options to `pytest`, for example to
select specific test cases. Change the session to allow overriding the options
passed to `pytest`, via the
[session.posargs](https://nox.thea.codes/en/stable/config.html#passing-arguments-into-sessions)
variable:

```python
# noxfile.py
import nox


@nox.session(python=["3.8", "3.7"])
def tests(session):
    args = session.posargs or ["--cov"]
    session.run("poetry", "install", external=True)
    session.run("pytest", *args)
```

Now you can run a specific test module inside the environments:

```sh
nox -- tests/test_console.py
```

## Mocking with pytest-mock

Unit tests should be [fast, isolated, and
repeatable](http://agileinaflash.blogspot.com/2009/02/first.html). The test for
`console.main` is neither of these: It performs an actual request against a live
API, which means that the test takes a full round-trip to the server to
complete, does not run in an isolated environment, and its outcome depends on
API behaviour, with the endpoint returning a random article. Even worse, the
test fails whenever the network is down.

The [unittest.mock](https://docs.python.org/3/library/unittest.mock.html)
standard library allows you to replace parts of your system under test with mock
objects. Use it via the [pytest-mock](https://github.com/pytest-dev/pytest-mock)
plugin, which integrates the library with `pytest`:

```sh
poetry add --dev pytest-mock
```

The plugin provides a `mocker` fixture, which can be used to replace the
`requests.get` function by a mock object. This mock object will be useful for
any test case involving the API, so let's create a test fixture for it. You can
place this fixture in a file named `conftest.py`, which will make it accessible
to all test modules without requiring an explicit import.

```python
# tests/conftest.py
import pytest


@pytest.fixture
def mock_requests_get(mocker):
    mock = mocker.patch("requests.get")
    mock.return_value.json.return_value = {
        "title": "Lorem Ipsum Dolor",
        "extract": "Lorem Ipsum Dolor",
    }
    return mock
```
    
The mock object needs to return something meaningful. `console.main` expects a
 of splines which it can print to the screen. You can configure the mock
object to return a specific value, effectively short-circuiting the call to the
real function. Here's how you do it:

```python
# tests/test_console.py
@pytest.fixture
def mock_splines_reticulate(mocker):
    mock = mocker.patch("hypermodern_python.splines.reticulate")
    mock.return_value = [1, 2, 3]
    return mock


def test_main_succeeds(runner, mock_splines_reticulate):
    result = runner.invoke(console.main)
    assert result.exit_code == 0
```

Cheating? Not so. Assuming that the function is fully covered by other test
cases, this allows you to focus on how `console.main` uses it for its own
purposes.

Use the mock in your test case by adding it as a function parameter:

```python
# tests/test_console.py
def test_main_succeeds(runner, mock_requests_get):
    ...
```
    
Now the test passes immediately and reproducibly, without invoking the actual
`requests.get` function.

You can also check that a mock object was called by the function under test:

```python
# tests/test_console.py
def test_main_invokes_requests_get(runner, mock_requests_get):
    runner.invoke(console.main)
    assert mock_requests_get.called
```

## More mocking with pytest-mock

While the code coverage is now reported as 100%, the tests do not cover the
`--count` option at all. Nor do they make any assertions about the output
produced by the program. Remember, code coverage only tells you that all lines
and branches in your code base were hit, not that the behavior of your program
was checked in any meaningful way.

Add another test case for the `console` module to check the program output when
passed the option `--count=1`:

```python
# tests/test_console.py
def test_main_prints_progress_message(runner, mock_sleep):
    result = runner.invoke(console.main, ["--count=1"])
    assert result.output == "Reticulating spline 1...\n"
```

Mocks help you test code units depending on bulky subsystems, but they are [not
the only
technique](https://blog.pragmatists.com/test-doubles-fakes-mocks-and-stubs-1a7491dfa3da)
to do so. For example, if your function requires a database connection, it may
be both easier and more effective to pass an in-memory database than a mock
object. Fake implementations are another good alternative to mock objects, which
can be too forgiving when faced with wrong usage. Large data objects can be
generated by test object factories, instead of being replaced by mock objects
(check out the excellent [factoryboy](https://factoryboy.readthedocs.io/)
package).

In the [next chapter](../hypermodern-python-03-linting), I'm going to discuss
how to add linting to your project.
