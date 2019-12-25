--- 
date: 2020-01-02T08:00:00+02:00
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
- [Mocking with pytest-mock](#mocking-with-pytestmock)
- [Fakes and Stubs](#fakes-and-stubs)
- [Meaningful tests](#meaningful-tests)
- [Refactoring the example application](#refactoring-the-example-application)
- [Example: Handling exceptions](#example-handling-exceptions)
- [Example: Adding an option to select the Wikipedia language edition](#example-adding-an-option-to-select-the-Wikipedia-language-edition)

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

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/chapter01...chapter02)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter02.zip)

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
let's turn it into a [test
fixture](https://docs.pytest.org/en/latest/fixture.html). Test fixtures are
declared as simple functions with the `pytest.fixture` decorator. Test cases can
use a test fixture by including a function parameter with the same name as the
test fixture.

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
[Coverage.py](https://coverage.readthedocs.io/). Install it with the
[pytest-cov](https://pytest-cov.readthedocs.io/en/latest/) plugin, which
integrates Coverage.py with `pytest`:

```sh
poetry add --dev coverage[toml] pytest-cov
```

You can configure Coverage.py using the `pyproject.toml` configuration file,
provided it was installed with the `toml` extra as shown above. Update this file
to inform the tool about your package name and source tree layout. The
configuration also enables branch analysis and the display of line numbers for
missing coverage:

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
$ poetry run pytest --cov
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
`console.main` is neither of these:

- It is not fast, because it takes a full round-trip to the Wikipedia API to
  complete.
- It does not run in an isolated environment, because it sends out an actual
  request over the network.
- It is not repeatable, because its outcome depends on the health, reachability,
  and behaviour of the API. In particular, the test fails whenever the network
  is down.

The [unittest.mock](https://docs.python.org/3/library/unittest.mock.html)
standard library allows you to replace parts of your system under test with mock
objects. Use it via the [pytest-mock](https://github.com/pytest-dev/pytest-mock)
plugin, which integrates the library with `pytest`:

```sh
poetry add --dev pytest-mock
```

The plugin provides a `mocker` fixture, which functions as a thin wrapper around
the standard mocking library. Use `mocker.patch` to replace the `requests.get`
function by a mock object. The mock object will be useful for any test case
involving the Wikipedia API, so let's create a test fixture for it:

```python
# tests/test_console.py
@pytest.fixture
def mock_requests_get(mocker):
    return mocker.patch("requests.get")
```

Add the fixture to the function parameters of the test case:

```python
def test_main_succeeds(runner, mock_requests_get):
    ...
```
    
<!--
Like before, the fixture is added as a function parameter to the test case using
it. This time, the fixture itself also depends on a fixture. This is expressed
in the same way, by adding `mocker` as a function parameter to
`mock_requests_get`.

-->

If you run Nox now, the test fails because click expects to be passed a string
for console output, and receives a mock object instead. Simply "knocking out"
`requests.get` is not quite enough. The mock object also needs to return
something meaningful, namely a response with a valid JSON object.

When a mock object is called, or when an attribute is accessed, it returns
another mock object. Sometimes this suffices to get you through a test case.
When it doesn't, you need to *configure* the mock object. To configure an
attribute, you simply set the attribute to the desired value. To configure the
return value for when the mock is called, you set `return_value` on the mock
object as if it was an attribute.

Let's look at the example again:

```python
with requests.get(API_URL) as response:
    response.raise_for_status()
    data = response.json()
```

The code uses the response as a [context
manager](https://docs.python.org/3/reference/datamodel.html#context-managers).
The `with` statement is syntactic sugar for the following slightly simplified
pseudocode:

```python
context = requests.get(API_URL)
response = context.__enter__()

try:
    response.raise_for_status()
    data = response.json()
finally:
    context.__exit__(...)
```

So what you have is essentially a chain of function calls:

```python
data = requests.get(API_URL).__enter__().json()
```

Rewrite the fixture, and mirror this call chain when you configure the mock:

```python
@pytest.fixture
def mock_requests_get(mocker):
    mock = mocker.patch("requests.get")
    mock.return_value.__enter__.return_value.json.return_value = {
        "title": "Lorem Ipsum",
        "extract": "Lorem ipsum dolor sit amet",
    }
    return mock
```
    
The `mock_requests_get` fixture demonstrates one drawback of mocking: Mocks can
be tightly coupled to implementation details of the system under test. In the
long run, you may be better off to implement an actual "fake" API and serve it
via HTTP for your tests.

## Fakes and Stubs

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

Implementing a fake API is out of the scope of this tutorial, but we will cover
one aspect of it: How to write a fixture which requires tear down code as well
as set up code. Suppose you have written the following fake API implementation:

```python
class FakeAPI:
    url = "http://localhost:5000/"

    @classmethod
    def create(cls):
        ...
    
    def shutdown(self):
        ...
```

The following will not work:

```python
@pytest.fixture
def fake_api():
    return FakeAPI.create()
```

The API needs to be shut down after use, to free up resources such as the TCP
port and the thread running the server. You can do this by writing the fixture
as a [generator](https://docs.python.org/3/tutorial/classes.html#generators):

```python
@pytest.fixture
def fake_api():
    api = FakeAPI.create()
    yield api
    api.shutdown()
```

Pytest takes care of running the generator, passing the yielded value to your
test function, and executing the shutdown code after it returns. If setting up
and tearing down the fixture is expensive, you may also consider extending the
[scope](https://docs.pytest.org/en/latest/fixture.html#scope-sharing-a-fixture-instance-across-tests-in-a-class-module-or-session)
of the fixture. By default, fixtures are created once per test function.
Instead, you could create the fake API server once per test session:

```python
@pytest.fixture(scope="session")
def fake_api():
    api = FakeAPI.create()
    yield api
    api.shutdown()
```

## Meaningful tests

While the code coverage is reported as 100%, the tests do not check the
functionality of the program at all. Remember, code coverage only tells you that
all lines and branches in your code base were hit, not that the behavior of your
program was checked in any meaningful way.

For example, you can check that the title returned by the API is printed to the
console:

```python
def test_main_prints_title(runner, mock_requests_get):
    result = runner.invoke(console.main)
    assert "Lorem Ipsum" in result.output
```

You can also check that `requests.get` is invoked to send a request to the API:

```python
# tests/test_console.py
def test_main_invokes_requests_get(runner, mock_requests_get):
    runner.invoke(console.main)
    assert mock_requests_get.called
```

## End-to-end testing

Testing against the live production server is bad practise for unit tests, but
there is nothing like the confidence you get from seeing your code work in a
real environment. Such tests are known as *end-to-end tests*, and while they are
usually too slow, brittle and unpredictable for the kind of automated testing
you would want to do on a CI server or in the midst of development, they do have
their place.

Let's reinstate the original test case, and use Pytest's
[markers](https://docs.pytest.org/en/latest/example/markers.html) to apply a
custom mark. This will allow you to select or skip them later, using Pytest's
`-m` option.

```python
# tests/test_console.py
@pytest.mark.e2e
def test_main_succeeds_in_production_env(runner):
    result = runner.invoke(console.main)
    assert result.exit_code == 0
```

Register the `e2e` marker using the `pytest_configure` hook, as shown below. The
hook is placed in a `conftest.py` file, at the top-level of your tests package.
This ensures that Pytest can discover the module and use it for the entire test
suite.

```python
# tests/conftest.py
def pytest_configure(config):
    config.addinivalue_line("markers", "e2e: mark as end-to-end test.")
```

Finally, exclude end-to-end tests from automated testing by passing `-m "not
e2e"` to Pytest:

```python
# noxfile.py
import nox


@nox.session(python=["3.8", "3.7"])
def tests(session):
    args = session.posargs or ["--cov", "-m", "not e2e"]
    session.run("poetry", "install", external=True)
    session.run("pytest", *args)
```

You can now run end-to-end tests by passing `-m e2e` to the Nox session, using a
double dash (`--`) to separate them from Nox's own options. For example, here's
how you would run end-to-end tests inside the testing environment for Python
3.8:

```sh
nox -rs tests-3.8 -- -m e2e
```

## Refactoring the example application

The great thing about a good test suite is that it allows you to refactor your
code without fear of breaking it. Let's move the API client code to a separate
module. Create a file `src/hypermodern-python/client.py` with the following
contents:

```python
# src/hypermodern-python/client.py
import requests


API_URL = "https://en.wikipedia.org/api/rest_v1/page/random/summary"


def get_random_fact():
    with requests.get(API_URL) as response:
        response.raise_for_status()
        return response.json()
```

Import `client` in the `console` module. The `console.main` function can now
simply invoke `client.get_random_fact` to retrieve the data for the random fact:

```python
# src/hypermodern-python/console.py
import textwrap

import click

from . import __version__, client


@click.command()
@click.version_option(version=__version__)
def main():
    """The hypermodern Python project."""
    data = client.get_random_fact()

    title = data["title"]
    extract = data["extract"]

    click.secho(title, fg="green")
    click.echo(textwrap.fill(extract))
```

Finally, invoke Nox to see that nothing broke:

```sh
$ nox -r
...
nox > Ran multiple sessions:
nox > * tests-3.8: success
nox > * tests-3.7: success
```

## Example: Handling exceptions

In this section, we add exception handling to the example application.

Generally, tests for a feature or bugfix should be written *before* implementing
it. This is also known as "writing a failing test". The reason for this is that
it provides confidence that the tests are actually testing something, and do not
simply pass because of a flaw in the tests themselves.

So let's start by adding a failing test. How do we get the API to raise an
error? The answer is again mocking. But this time, we need the `requests.get`
mock to raise an exception instead of returning a value. You can do so by
assigning an exception instance or class to the
[side_effect](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock.side_effect)
attribute of the mock, like so:

```python
# tests/test_console.py
import requests


@pytest.fixture
def mock_failing_requests_get(mocker):
    mock = mocker.patch("requests.get")
    mock.side_effect = requests.RequestException
    return mock
```

When faced with an error from requests, we expect our application to exit with a
non-zero status code, and print an error message to the console. You should
generally have a single assertion per test case, so we are going to add two test
cases.

The first test case asserts that the exit status is 1 on request errors:

```python
# tests/test_console.py
def test_main_fails_on_request_error(runner, mock_failing_requests_get):
    result = runner.invoke(console.main)
    assert result.exit_code == 1
```

The second test case asserts that the output contains the string "Error":

```python
def test_main_prints_message_on_request_error(runner, mock_failing_requests_get):
    result = runner.invoke(console.main)
    assert "Error" in result.output
```

(The first of these test cases is not actually a failing test. When the Python
interpreter is terminated by an unhandled exception, it also exits with a status
code of 1.)

Finally, let's get the test suite to pass by handling the exception. 

Exceptions from requests are subclasses of `requests.RequestException`, and you
can differentiate between them by type to print friendly, informative error
messages. For the purposes of this example, we will only deal with the base
class, and rely on the exception message to inform the user what happened.

You can exit a click application from anywhere in your program by raising a
`click.ClickException` with an error message. When click encounters this
exception, it will print the message to the standard error stream and exit the
program with a status code of 1.

```python
# src/hypermodern-python/client.py
import click
import requests


API_URL = "https://en.wikipedia.org/api/rest_v1/page/random/summary"


def get_random_fact():
    try:
        with requests.get(API_URL) as response:
            response.raise_for_status()
            return response.json()
    except requests.RequestException as error:
        message = str(error)
        raise click.ClickException(message)
```

## Example: Adding an option to select the Wikipedia language edition

In the final section, we will add an option to select the [language
edition](https://en.wikipedia.org/wiki/List_of_Wikipedias) of Wikipedia.
Wikipedia editions are identified by a code, which is used as a subdomain below
wikipedia.org. With some exceptions, this Wikipedia code is identical to the
two-letter or three-letter language code assigned to the language by [ISO
639-1](https://en.wikipedia.org/wiki/ISO_639-1) and [ISO
639-3](https://en.wikipedia.org/wiki/ISO_639-3). In a first step, we will pass
the language code as an optional argument to `client.get_random_fact`.

We start by adding a failing test to a new test module named `test_client.py`.
For this, we are going to need the `mock_requests_get` mock. Instead of copying
the fixture to the new test module, we can move it to `conftest.py`, where it
will be available to every test module in the test package.

```python
# tests/conftest.py
import pytest
import requests


@pytest.fixture
def mock_requests_get(mocker):
    mock = mocker.patch("requests.get")
    mock.return_value.__enter__.return_value.json.return_value = {
        "title": "Lorem Ipsum",
        "extract": "Lorem ipsum dolor sit amet",
    }
    return mock
```

When `get_random_fact` is passed an alternate language, it should send its API
request to the matching Wikipedia edition. Mock objects allow you to inspect the
arguments they were called with, using the
[call_args](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock.call_args)
attribute. This allows our test case to check that `requests.get` was called
with a URL containing the correct domain name:

```python
from hypermodern_python import client


def test_get_random_fact_uses_given_language(mock_requests_get):
    client.get_random_fact(language="de")
    args, _ = mock_requests_get.call_args
    assert "de.wikipedia.org" in args[0]
```

To get the test to pass, we turn `API_URL` into a format string, and interpolate
the specified language code into the URL, using
[str.format](https://docs.python.org/3/library/stdtypes.html#str.format).

```python
# src/hypermodern-python/client.py
import click
import requests


API_URL = "https://{language}.wikipedia.org/api/rest_v1/page/random/summary"


def get_random_fact(language="en"):
    url = API_URL.format(language=language)

    try:
        with requests.get(url) as response:
            response.raise_for_status()
            return response.json()
    except requests.RequestException as error:
        message = str(error)
        raise click.ClickException(message)
```

The second step is to make this functionality accessible from the command line,
using the `--language` option. Again, we start by adding a failing test. When
the option is specified, `console.main` is expected to invoke
`client.get_random_fact` with the specified language. We can check this using a
mock for `get_random_fact`:

```python
# tests/test_console.py
@pytest.fixture
def mock_get_random_fact(mocker):
    return mocker.patch("hypermodern_python.client.get_random_fact")
```

The test case uses the
[assert_called_with](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock.assert_called_with)
method on the mock to check that the language specified by the user is passed on
to the `client` module:

```python
# tests/test_console.py
def test_main_uses_specified_language(runner, mock_get_random_fact):
    result = runner.invoke(console.main, ["--language=pl"])
    mock_get_random_fact.assert_called_with(language="pl")
```

We are now ready to implement the new functionality using the
[click.option](https://click.palletsprojects.com/en/7.x/options/) decorator.
With no further ado, here is the final version of the `console` module:

```python
# src/hypermodern-python/console.py
import textwrap

import click

from . import __version__, client


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
def main(language):
    """The hypermodern Python project."""
    data = client.get_random_fact(language)

    title = data["title"]
    extract = data["extract"]

    click.secho(title, fg="green")
    click.echo(textwrap.fill(extract))
```

## Thank you for reading!

In the [next chapter](../hypermodern-python-03-linting), I'm going to discuss
how to add linting to your project.
