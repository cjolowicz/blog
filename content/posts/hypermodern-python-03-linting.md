--- 
date: 2020-01-08T09:04:32+01:00
title: "Hypermodern Python Chapter 3: Linting"
description: "A guide to modern Python tooling with a focus on simplicity and minimalism."
draft: true
tags:
  - python
  - nox
  - flake8
  - black
  - bandit
---

[Read this article on Medium](https://medium.com/@cjolowicz/hypermodern-python-3-linting-xxxxxxxxxxxx)

{{< figure src="/images/hypermodern-python-03/cote01.jpg" link="/images/hypermodern-python-03/cote01.jpg" >}}

In this third installment of the Hypermodern Python series, I'm going to discuss
how to add linting, code formatting, and static analysis to your project.[^1]
Previously, we discussed [Automated Testing](../hypermodern-python-02-testing).

[^1]: The images in this chapter come from a series of futuristic pictures by
    Jean-Marc Côté and other artists issued in France in 1899, 1900, 1901 and
    1910 (source: [Wikimedia
    Commons](https://commons.wikimedia.org/wiki/Category:France_in_XXI_Century_(fiction))
    via [The Public Domain
    Review](https://publicdomainreview.org/collection/a-19th-century-vision-of-the-year-2000))

Here are the topics covered in this chapter on Linting in Python:

- [Linting with flake8](#linting-with-flake8)
- [Code formatting with Black](#code-formatting-with-black)
- [Checking imports with flake8-import-order](#checking-imports-with-flake8-import-order)
- [Finding more bugs with flake8-bugbear](#finding-more-bugs-with-flake8-bugbear)
- [Identifying security issues with bandit](#identifying-security-issues-with-bandit)
- [Pinning development dependencies with Poetry and Nox](#pinning-development-dependencies-with-poetry-and-nox)

Here is a full list of the articles in this series:

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting) (this article)
- *Chapter 4: Typing*
- *Chapter 5: Documentation*
- *Chapter 6: CI/CD*
- *Appendix: Docker*

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Each article in the guide corresponds to a set of commits in the GitHub
repository:

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/chapter02...chapter03)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter03.zip)

## Linting with flake8

{{< figure src="/images/hypermodern-python-03/cote02.jpg" link="/images/hypermodern-python-03/cote02.jpg" >}}

Linters analyze source code to flag programming errors, bugs, stylistic errors,
and suspicious constructs. The most common ones for Python are
[pylint](https://www.pylint.org), [flake8](http://flake8.pycqa.org), and
[pylama](https://github.com/klen/pylama). In this chapter, we use Flake8.

Add a Nox session to run Flake8 on your codebase:

```python
# noxfile.py
locations = "src", "tests", "noxfile.py"


@nox.session(python=["3.8", "3.7"])
def lint(session):
    args = session.posargs or locations
    session.install("flake8")
    session.run("flake8", *args)
```

The
[session.install](https://nox.thea.codes/en/stable/config.html#nox.sessions.Session.install)
method installs packages into the virtual environment via
[pip](https://pip.pypa.io/). By default, we run Flake8 on three locations: the
package source tree, the test suite, and `noxfile.py` itself. You can override
this by passing specific source files, separated from Nox's own options by `--`.

Flake8 glues together several tools. Each message is assigned an error code,
prefixed by one or more letters. These prefixes group the errors into so-called
violation classes:

- `F` are errors reported by [pyflakes](https://github.com/PyCQA/pyflakes), a
  tool which parses source files and finds invalid Python code.
- `W` and `E` are warnings and errors reported by
  [pycodestyle](https://github.com/pycqa/pycodestyle), which checks your Python
  code against some of the style conventions in [PEP
  8](http://www.python.org/dev/peps/pep-0008/).
- `C` are violations reported by [mccabe](https://github.com/PyCQA/mccabe),
  which checks the code complexity of your Python package against a configured
  limit.

Configure Flake8 using the `.flake8` configuration file, enabling all the
built-in violation classes and setting the complexity limit:

```ini
# .flake8
[flake8]
select = C,E,F,W
max-complexity = 10
```

By default, Nox runs all sessions defined in `noxfile.py`. Use the [\-\-session
(-s)](https://nox.thea.codes/en/stable/usage.html#specifying-one-or-more-sessions)
option to restrict it to a specific session:

```sh
nox -rs lint
```

There are many [awesome Flake8
extensions](https://github.com/DmytroLitvinov/awesome-flake8-extensions). Some
of these will be presented in later sections.

## Code formatting with Black

{{< figure src="/images/hypermodern-python-03/cote03.jpg" link="/images/hypermodern-python-03/cote03.jpg" >}}

The next addition to our toolbox is [Black](https://github.com/psf/black), the
uncompromising Python code formatter. One of its greatest features is its lack
of configurability. Blackened code looks the same regardless of the project
you're reading.

Adding Black as a Nox session is straightforward:

```python
# noxfile.py
@nox.session(python="3.8")
def black(session):
    args = session.posargs or locations
    session.install("black")
    session.run("black", *args)
```

With the Nox session in place, you can reformat your code like this:

```sh
$ nox -rs black

nox > Running session black
nox > Creating virtual environment (virtualenv) using python3.8 in .nox/black
nox > pip install black
nox > black src tests noxfile.py
All done! ✨ 🍰 ✨
5 files left unchanged.
nox > Session black was successful.
```

Invoking `nox` without arguments triggers all the sessions, including Black. It
would be better to only validate the coding style without modifying the
conflicting files. You can exclude Black from the sessions run by default, by
setting `nox.options.sessions`:

```python
# noxfile.py
nox.options.sessions = "lint", "tests"
```

Instead, check adherence to the Black code style inside the linter session. The
[flake8-black](https://github.com/peterjc/flake8-black) plugin generates
warnings if it detects that Black would reformat a source file:

{{< highlight python "hl_lines=5" >}}
# noxfile.py
@nox.session(python=["3.8", "3.7"])
def lint(session):
    args = session.posargs or locations
    session.install("flake8", "flake8-black")
    session.run("flake8", *args)
{{< /highlight >}}

Configure Flake8 to enable the `flake8-black` warnings, which are prefixed by
`BLK`. Also, some built-in warnings do not align well with Black. You need to
ignore warnings `E203` (*Whitespace before ':'*), and `W503` (*Line break before
binary operator*), and set the maximum line length to a more permissive value:

```ini
# .flake8
[flake8]
select = BLK,C,E,F,W
ignore = E203,W503
max-line-length = 88
```

## Checking imports with flake8-import-order

{{< figure src="/images/hypermodern-python-03/cote04.jpg" link="/images/hypermodern-python-03/cote04.jpg" >}}

The [flake8-import-order](https://github.com/PyCQA/flake8-import-order) plugin
checks that import statements are grouped and ordered in a consistent and [PEP
8](https://www.python.org/dev/peps/pep-0008/#imports)-compliant way. Imports
should be arranged in three groups, like this:

```python
# standard library
import time

# third-party packages
import click

# local packages
from hypermodern_python import wikipedia
```

Install the plugin in the linter session:

{{< highlight python "hl_lines=5" >}}
# noxfile.py
@nox.session(python=["3.8", "3.7"])
def lint(session):
    args = session.posargs or locations
    session.install("flake8", "flake8-black", "flake8-import-order")
    session.run("flake8", *args)
{{< /highlight >}}

Enable the warnings emitted by the plugin (`I` like *import*). 

```ini
# .flake8
[flake8]
select = BLK,C,E,F,I,W
```

Inform the plugin about package names which are considered local:

```ini
# .flake8
[flake8]
application-import-names = hypermodern_python,tests
```

Adopt the [Google
styleguide](https://google.github.io/styleguide/pyguide.html?showone=Imports_formatting#313-imports-formatting)
with respect to the grouping and ordering details:

```ini
# .flake8
[flake8]
import-order-style = google
```

## Finding more bugs with flake8-bugbear

{{< figure src="/images/hypermodern-python-03/cote05.jpg" link="/images/hypermodern-python-03/cote05.jpg" >}}

The [flake8-bugbear](https://github.com/PyCQA/flake8-bugbear) plugin helps you
find various bugs and design problems in your programs. Add the plugin to the
linter session in your `noxfile.py`:

```python
# noxfile.py
@nox.session(python=["3.8", "3.7"])
def lint(session):
    args = session.posargs or locations
    session.install("flake8", "flake8-black", "flake8-bugbear", "flake8-import-order")
    session.run("flake8", *args)
```

Enable the plugin warnings in Flake8's configuration file (`B` like *bugbear*):

```ini
# .flake8
[flake8]
select = B,B9,BLK,C,E,F,I,W
```

`B9` is required for Bugbear's more opinionated warnings, which are disabled by
default. In particular, `B950` checks the maximum line length like the built-in
`E501`, but with a tolerance margin of 10%. Ignore the built-in error `E501` and
set the maximum line length to a sane value:

```ini
# .flake8
[flake8]
ignore = E203,E501,W503
max-line-length = 80
```

## Identifying security issues with bandit

{{< figure src="/images/hypermodern-python-03/cote06.jpg" link="/images/hypermodern-python-03/cote06.jpg" >}}

[Bandit](https://github.com/PyCQA/bandit) is a tool designed to find common
security issues in Python code. Install it via the
[flake8-bandit](https://github.com/tylerwince/flake8-bandit) plugin:

```python
# noxfile.py
@nox.session(python=["3.8", "3.7"])
def lint(session):
    args = session.posargs or locations
    session.install(
        "flake8",
        "flake8-bandit",
        "flake8-black",
        "flake8-bugbear",
        "flake8-import-order",
    )
    session.run("flake8", *args)
```

Enable the plugin warnings in Flake8's configuration file (`S` like *security*):

```ini
# .flake8
[flake8]
select = B,B9,BLK,C,E,F,I,S,W
...
```

Bandit flags uses of `assert` to enforce interface constraints because
assertions are removed when compiling to optimized byte code. You should disable
this warning for your test suite, as `pytest` uses assertions to verify
expectations in tests:

```ini
# .flake8
[flake8]
per-file-ignores = tests/*:S101
...
```

## Managing development dependencies in Nox sessions

In the first Chapter, we saw that Poetry writes the exact version of each
package dependency to a file named `poetry.lock`. The same is done for
development dependencies like `pytest`. This is known as *pinning*, and it
allows you to build and test your package in a predictable and deterministic
way.

By contrast, this is how we have been installing packages into our Nox sessions
so far:
 
```python
session.install("flake8")
```

No version specified! Nox will install whatever pip considers the latest version
when the session is run. The checks may succeed when you run them on your local
machine, but suddenly break on another developer's machine or on a Continuous
Integration server, due to a change to Flake8 or one of its dependencies. These
things happen all the time, and the problem accumulates quickly as the
dependencies of your project grow.

We could pin Flake8 using something like the following:

```python
session.install("flake8==3.7.9")
```

This approach improves our situation, but it has some limitations:

- We're back to handling requirements manually, rather than using Poetry's rich
  support for dependency management.
- The check is still not deterministic, because dependencies of dependencies
  remain unpinned. Flake8 is a good example for this: At its core, it aggregates
  several more specialized tools, and these are still installed without any
  version constraint.

How about we declare Flake8 as a *development dependency* of our project, like
we did with Pytest in the previous chapter? Then we can benefit from Poetry as a
dependency manager, and record the versions of Flake8 and its dependencies in
its lock file. -- Well, there is a catch. Look how we installed development
dependencies in the Nox session for testing:

```python
session.run("poetry", "install", external=True)
```

This command installs a bunch of things our linting session does not need:

- the package under development
- the package dependencies
- unrelated development dependencies (e.g. Pytest)

A major difference between testing and linting is that you need to install your
package to be able to run the test suite, but you don't need to install your
package to run linters on it. Linters are *static analysis tools*, they don't
need to run your program.

Wouldn't it be great if we could install individual packages with
`session.install`, but somehow use Poetry's lock file to constrain their
versions? Fortunately, there is a pip feature that let's us do exactly this:
[constraints
files](https://pip.pypa.io/en/stable/user_guide/#constraints-files). If you have
used a `requirements.txt` file before, the format is exactly the same. And
Poetry has a command to export its lock file to requirements format. So we have
all the building blocks for a solution.

The function `install_with_constraints` below is a wrapper for
`session.install`. It generates a constraints file by running [poetry
export](https://python-poetry.org/docs/cli/#export), and passes that file to pip
using its `--constraint` option. The function uses the standard
[tempfile](https://docs.python.org/3/library/tempfile.html) module to create a
temporary file for the constraints.

```python
# noxfile.py
import tempfile


def install_with_constraints(session, *args, **kwargs):
    with tempfile.NamedTemporaryFile() as requirements:
        session.run(
            "poetry",
            "export",
            "--dev",
            "--format=requirements.txt",
            f"--output={requirements.name}",
            external=True,
        )
        session.install(f"--constraint={requirements.name}", *args, **kwargs)
```

Change the Nox sessions to call the `install_with_constraints` wrapper instead
of invoking `session.install` directly:

```python
@nox.session(python="3.8")
def black(session):
    args = session.posargs or locations
    install_with_constraints(session, "black")
    session.run("black", *args)


@nox.session(python=["3.8", "3.7"])
def lint(session):
    args = session.posargs or locations
    install_with_constraints(
        session,
        "flake8",
        "flake8-bandit",
        "flake8-black",
        "flake8-bugbear",
        "flake8-import-order",
    )
    session.run("flake8", *args)
```

You can now use Poetry to manage Black and Flake8 as development dependencies:

```sh
poetry add --dev \
    black \
    flake8 \
    flake8-bandit \
    flake8-black \
    flake8-bugbear \
    flake8-import-order
```

You should also adapt the testing session. The testing session only needs to
install packages required for running the test suite, and should not be
cluttered by anything else. Instead of simply invoking `poetry install`, pass
the `--no-dev` option. This excludes development dependencies, and installs only
the package itself and its dependencies. Then install the test requirements
explicitly using `install_with_constraints`. Here is the rewritten Nox session:

```python
@nox.session(python=["3.8", "3.7"])
def tests(session):
    args = session.posargs or ["--cov", "-m", "not e2e"]
    session.run("poetry", "install", "--no-dev", external=True)
    install_with_constraints(
        session, "coverage[toml]", "pytest", "pytest-cov", "pytest-mock"
    )
    session.run("pytest", *args)
```

## Thanks for reading!

The next chapter is about adding static type checking to your project. It will
be published on January 22, 2020.

{{< figure src="/images/hypermodern-python-03/train.jpg" class="centered" >}}
<!-- 
{{< figure src="/images/hypermodern-python-04/train.jpg" link="../hypermodern-python-04-typing" class="centered" >}}
<span class="centered">[Continue to the next chapter](../hypermodern-python-04-typing)</span>
-->
