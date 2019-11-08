--- 
date: 2019-11-07T12:52:59+02:00
title: "The hypermodern Python project"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - poetry
  - pyenv
  - click
---

Welcome to the whirlwind tour of the Python ecosystem in late 2019! In this
first chapter, you are going to learn how to set up a Python project using pyenv
and Poetry. At the end of this chapter, you will have a simple command-line
application built using click.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-project-01-setup)
- [Chapter 2: Testing](../hypermodern-python-project-02-testing)
- [Chapter 3: Continuous Integration](../hypermodern-python-project-03-continuous-integration)
- [Chapter 4: Documentation](../hypermodern-python-project-04-documentation)

<!--
This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

## Introduction

After more than a decade of coexistence with Python 3, [the Python 2 sun will
set](https://www.python.org/doc/sunset-python-2/) on new year
2020. The Python landscape has changed significantly over this decade, with a
host of new tools and best practices improving the Python developer experience.
At the same time, their adoption has lagged behind, due to the constraints of
legacy support.

Time to show how to build a Python project for *hypermodernists*, from scratch.

This guide is aimed both at beginners who are keen to learn best practises from
the start, and seasoned Python developers whose workflows are affected by
boilerplate and workarounds required by the legacy toolbox. The focus is on
simplicity and minimalism.

> *You need a recent Linux, Unix, or Mac system with
> [bash](https://www.gnu.org/software/bash/), [curl](https://curl.haxx.se) and
> [git](https://www.git-scm.com) for this tutorial.*

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Introduction](#introduction)
- [Setting up a GitHub repository](#setting-up-a-github-repository)
- [Installing Python with pyenv](#installing-python-with-pyenv)
- [Setting up a Python project using Poetry](#setting-up-a-python-project-using-poetry)
- [Creating a package in src layout](#creating-a-package-in-src-layout)
- [Managing virtual environments with Poetry](#managing-virtual-environments-with-poetry)
- [Managing dependencies with Poetry](#managing-dependencies-with-poetry)
- [Command-line interfaces with click](#command-line-interfaces-with-click)

<!-- markdown-toc end -->

## Setting up a GitHub repository

In this guide, [GitHub](https://github.com) is used to host the public git
repository for your project. Other popular options are
[GitLab](https://gitlab.com/) and [BitBucket](https://bitbucket.org/). Create a
repository, and populate it with `README.md` and `LICENSE` files. For this
project, I will use the [MIT license](https://choosealicense.com/licenses/mit/),
a simple permissive license.

> *Throughout this guide, replace `hypermodern-python` with the name of your own
> repository. Choose a different name to avoid a name collision on PyPI.*

Clone the repository to your machine, and `cd` into it:

```sh
git clone git@github.com:<your-username>/hypermodern-python.git
cd hypermodern-python
```

As you follow the rest of this guide, create a series of [small, atomic
commits](https://deepsource.io/blog/git-best-practices/) documenting your steps.
Use `git status` to discover files generated by commands shown in the guide.

## Installing Python with pyenv

Let's continue by setting up the developer environment. First you need to get a
recent Python. Don't bother with package managers or official binaries. The tool
of choice is [pyenv](https://github.com/pyenv/pyenv), a Python version manager.
Install it like this:

```sh
curl https://pyenv.run | bash
```

Add the following lines to your `~/.bashrc`:

```sh
export PATH="~/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

Open a new shell, or source `~/.bashrc` in your current shell:

```sh
source ~/.bashrc
```

Install the Python build dependencies for your platform, using one of the
commands listed in the [official
instructions](https://github.com/pyenv/pyenv/wiki/Common-build-problems). For
example, on a recent [Ubuntu](https://ubuntu.com) this would be:

```sh
sudo apt update && sudo apt install -y make build-essential libssl-dev zlib1g-dev \
libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev \
libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python-openssl git
```

You're ready to install the latest Python releases. This may take a while:

```sh
pyenv install 3.8.0
pyenv install 3.7.5
```

Make your fresh Pythons available inside the repository:

```sh
pyenv local 3.8.0 3.7.5
```

Congratulations! You have access to the latest and greatest of Python:

```sh
$ python --version
Python 3.8.0

$ python3.7 --version
Python 3.7.5
```

Python 3.8.0 is the default version and can be invoked as `python`, but both
versions are accessible as `python3.7` and `python3.8`, respectively.

## Setting up a Python project using Poetry

[Poetry](https://poetry.eustace.io) is a tool to manage Python packaging and
dependencies. Its ease of use and support for modern workflows make it the ideal
successor to the venerable [setuptools](http://setuptools.readthedocs.io). It is
similar to `npm` and `yarn` in the JavaScript world, and to other modern package
and dependency managers.

With Poetry 1.0 [around the
corner](https://github.com/sdispater/poetry/projects/1), I would recommend you
install the preview version:

```sh
curl -sSL https://raw.githubusercontent.com/sdispater/poetry/master/get-poetry.py |
POETRY_PREVIEW=1 python
```

Open a new login shell or source `~/.poetry/env` in your current shell:

```sh
source ~/.poetry/env
```

Initialize your Python project:

```sh
poetry init -n  # --no-interaction
```

This command will create a `pyproject.toml` file, the new Python package
configuration file specified in [PEP
517](https://www.python.org/dev/peps/pep-0517/) and
[518](https://www.python.org/dev/peps/pep-0518/).

```toml
# pyproject.toml
[tool.poetry]
name = "hypermodern-python"
version = "0.1.0"
description = ""
authors = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python = "^3.8"

[tool.poetry.dev-dependencies]

[build-system]
requires = ["poetry>=0.12"]
build-backend = "poetry.masonry.api"
```

There you go: One declarative file in [TOML](https://github.com/toml-lang/toml)
syntax, containing the entire package configuration. Let's add some metadata to
the package:

```toml
# pyproject.toml
[tool.poetry]
...
description = "The hypermodern Python project"
license = "MIT"
readme = "README.md"
homepage = "https://github.com/<your-username>/hypermodern-python"
repository = "https://github.com/<your-username>/hypermodern-python"
keywords = ["hypermodern"]
```

Poetry added a dependency on Python 3.8, because this is the Python version you
ran it in. Support the previous release as well by changing this to Python 3.7:

```toml
[tool.poetry.dependencies]
python = "^3.7"
```

The caret (`^`) in front of the version number means "up to the next major
release". In other words, you are promising that your package won't break when
users upgrade to Python 3.8 or 3.9, but you're giving no guarantees for its use
with a future Python 4.0.

## Creating a package in src layout

Let's create an initial skeleton package. Organize your package in [src
layout](https://hynek.me/articles/testing-packaging/), like this:

```sh
.
├── pyproject.toml
└── src
    └── hypermodern_python
        └── __init__.py

2 directories, 2 files
```

The source file contains only a version declaration:

```python
# src/hypermodern_python/__init__.py
__version__ = "0.1.0"
```

Use [snake case](https://en.wikipedia.org/wiki/Snake_case) for the package name
`hypermodern_python`, as opposed to the [kebab
case](https://en.wiktionary.org/wiki/kebab_case) used for the repository name
`hypermodern-python`. In other words, name the package after your repository,
replacing hyphens by underscores.

> Replace `hypermodern-python` with the name of your own repository, to avoid a
> name collision on PyPI.

## Managing virtual environments with Poetry

A [virtual environment](https://docs.python.org/3/tutorial/venv.html) gives your
project an isolated runtime environment, consisting of a specific Python version
and an independent set of installed Python packages. This way, the dependencies
of your current project do not interfere with the system-wide Python
installation, or other projects you're working on.

Poetry manages virtual environments for your projects. To see this in action,
install the skeleton package using Poetry:

```sh
$ poetry install

Creating virtualenv hypermodern-python-rLESuZJY-py3.8 in …/pypoetry/virtualenvs
Updating dependencies
Resolving dependencies... (0.1s)

Writing lock file

Nothing to install or update

  - Installing hypermodern-python (0.1.0)
```

Poetry created a virtual environment dedicated to your project, and installed
your initial package into it. It also created a so-called *lock file*, named
`poetry.lock`. You will learn more about this file in the next section.

Let's run a Python session inside the new virtual environment:

```python
$ poetry run python

Python 3.8.0 (default, Oct 16 2019, 19:27:04)
[GCC 8.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import hypermodern_python
>>> hypermodern_python.__version__
'0.1.0'
>>>
```

## Managing dependencies with Poetry

Let's install the first dependency, the
[click](https://click.palletsprojects.com/) package. This Python package allows
you to create beautiful command-line interfaces in a composable way with as
little code as necessary.

```sh
$ poetry add click

Using version ^7.0 for click

Updating dependencies
Resolving dependencies... (0.1s)

Writing lock file


Package operations: 1 install, 0 updates, 0 removals

  - Installing click (7.0)
```

Several things are happening here:

- The package is downloaded and installed into the virtual environment.
- The installed version is registered in the lock file `poetry.lock`.
- A more general version constraint is added to `pyproject.toml`.

The dependency entry in `pyproject.toml` contains a [version
constraint](https://poetry.eustace.io/docs/versions/) for the installed package:
`^7.0`. This means that users of the package need to have at least the current
release, `7.0`. The constraint also allows newer releases of the package, as
long as they don't contain breaking changes.

Breaking changes are only allowed in major releases (incrementing the most
significant version digit). Packages with a version number smaller than 1.0.0
are a little special, in that breaking changes may occur in all releases except
patch releases (incrementing only the least significant digit). You can edit the
version constraint, for example if a package you depend on does not follow the
[Semantic Versioning](https://semver.org/) scheme.

By contrast, `poetry.lock` contains the exact version of `click` installed into
the virtual environment. Place this file under source control. It allows
everybody in your team to work with the same environment. It also helps you to
[keep production and development environments as similar as
possible](https://12factor.net/dev-prod-parity).

Upgrading the dependency to a new minor or patch release is now as easy as this:

```sh
poetry update click
```

To upgrade to a new major release, you need to update the version constraint
explicitly. Coming from the previous major release of `click`, you could use the
following command to upgrade to `7.0`:

```sh
poetry add click^7.0
```

## Command-line interfaces with click

Time to add some actual code to the package. As you may have guessed, we're
going to create a console application using `click`:

```python
# src/hypermodern_python/console.py
import click

from . import __version__


@click.command()
@click.version_option(version=__version__)
def main():
    """The hypermodern Python project."""
    click.echo("Hello, world!")
```

The `console` module defines a minimal command-line application, supporting
`--help` and `--version` options.

Register the script in `pyproject.toml`:

```toml
[tool.poetry.scripts]
hypermodern-python = "hypermodern_python.console:main"
```

Finally, install the package into the virtual environment:

```sh
poetry install
```

You can now run the script like this:

```sh
$ poetry run hypermodern-python -- --help

Usage: hypermodern-python [OPTIONS]

  The hypermodern Python project.

Options:
  --version  Show the version and exit.
  --help     Show this message and exit.
```

<center>[Continue to the next chapter](../hypermodern-python-project-02-testing)</center>