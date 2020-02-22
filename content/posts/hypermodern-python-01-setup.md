--- 
date: 2020-01-01T08:00:00+02:00
title: "Hypermodern Python"
description: "A guide to modern Python tooling with a focus on simplicity and minimalism."
tags:
  - python
  - poetry
  - pyenv
  - click
  - requests
---

[Read this article on Medium](https://medium.com/@cjolowicz/hypermodern-python-d44485d9d769)

{{< figure src="/images/hypermodern-python-01/opera_crop01.jpg" link="/images/hypermodern-python-01/opera_crop01.jpg" >}}

New Year 2020 [marks the end](https://www.python.org/doc/sunset-python-2/) of
more than a decade of coexistence of Python 2 and
3. The Python landscape has changed considerably over this period: a host of new
tools and best practices now improve the Python developer experience. Their
adoption, however, lags behind due to the constraints of legacy support.

This article series is a guide to modern Python tooling with a focus on
simplicity and minimalism.[^1] It walks you through the creation of a complete
and up-to-date Python project structure, with unit tests, static analysis,
type-checking, documentation, and continuous integration and delivery.

[^1]: The title of this guide is inspired by the book *Die hypermoderne
    Schachpartie* (The hypermodern chess game), written by [Savielly
    Tartakower](https://en.wikipedia.org/wiki/Savielly_Tartakower) in 1924. It
    surveys the revolution that had taken place in chess theory in the decade
    after the First World War. The images in this chapter are details from the
    hand-colored print *Le Sortie de l'opéra en l'an 2000* (Leaving the opera in
    the year 2000) by Albert Robida, ca 1902 (source: [Library of
    Congress](http://www.loc.gov/pictures/item/2007676247/)).

This guide is aimed at beginners who are keen to learn best practises from the
start, and seasoned Python developers whose workflows are affected by
boilerplate and workarounds required by the legacy toolbox.

### Requirements

You need a recent Linux, Unix, or Mac system with
[bash](https://www.gnu.org/software/bash/), [curl](https://curl.haxx.se) and
[git](https://www.git-scm.com) for this tutorial.

On Windows 10, enable the [Windows Subsystem for
Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10) (WSL) and
install the Ubuntu 18.04 LTS distribution. Open Ubuntu from the Start Menu, and
install additional packages using the following command:

```sh
sudo apt update && sudo apt install -y make build-essential libssl-dev zlib1g-dev \
libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev \
libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python-openssl git
```

## Overview

{{< figure src="/images/hypermodern-python-01/opera_crop04.jpg" link="/images/hypermodern-python-01/opera_crop04.jpg" link="/images/hypermodern-python-01/opera_crop04.jpg" >}}

In this first chapter, we set up a Python project using
[pyenv](https://github.com/pyenv/pyenv) and
[Poetry](https://python-poetry.org/). Our example project is a simple
command-line application, which uses the Wikipedia API to display random facts
on the console.

Here are the topics covered in this chapter:

- [Setting up a GitHub repository](#setting-up-a-github-repository)
- [Installing Python with pyenv](#installing-python-with-pyenv)
- [Setting up a Python project using Poetry](#setting-up-a-python-project-using-poetry)
- [Creating a package in src layout](#creating-a-package-in-src-layout)
- [Managing virtual environments with Poetry](#managing-virtual-environments-with-poetry)
- [Managing dependencies with Poetry](#managing-dependencies-with-poetry)
- [Command-line interfaces with click](#command-line-interfaces-with-click)
- [Example: Consuming a REST API with requests](#example-consuming-a-rest-api-with-requests)

Here is a list of the articles in this series:

- [Chapter 1: Setup](../hypermodern-python-01-setup) (this article)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation)
- [Chapter 6: CI/CD](../hypermodern-python-06-ci-cd)

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Each article in the guide corresponds to a set of commits in the GitHub
repository.

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/initial...chapter01)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter01.zip)

## Setting up a GitHub repository

{{< figure src="/images/hypermodern-python-01/opera_crop02.jpg" link="/images/hypermodern-python-01/opera_crop02.jpg" >}}

For the purposes of this guide, [GitHub](https://github.com) is used to host the
public git repository for your project. Other popular options are
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

{{< figure src="/images/hypermodern-python-01/opera_crop03.jpg" link="/images/hypermodern-python-01/opera_crop03.jpg" >}}

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
pyenv install 3.8.1
pyenv install 3.7.6
```

Make your fresh Pythons available inside the repository:

```sh
pyenv local 3.8.1 3.7.6
```

Congratulations! You have access to the latest and greatest of Python:

```sh
$ python --version
Python 3.8.1

$ python3.7 --version
Python 3.7.6
```

Python 3.8.1 is the default version and can be invoked as `python`, but both
versions are accessible as `python3.7` and `python3.8`, respectively.

## Setting up a Python project using Poetry

{{< figure src="/images/hypermodern-python-01/opera_crop05.jpg" link="/images/hypermodern-python-01/opera_crop05.jpg" >}}

[Poetry](https://python-poetry.org/) is a tool to manage Python packaging and
dependencies. Its ease of use and support for modern workflows make it the ideal
successor to the venerable [setuptools](http://setuptools.readthedocs.io). It is
similar to `npm` and `yarn` in the JavaScript world, and to other modern package
and dependency managers. For alternatives to Poetry, have a look at
[flit](https://github.com/takluyver/flit),
[pipenv](https://pipenv.kennethreitz.org/en/latest/),
[pyflow](https://github.com/David-OConnor/pyflow), and
[dephell](https://dephell.org/).

Install Poetry:

```sh
curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
```

Open a new login shell or source `~/.poetry/env` in your current shell:

```sh
source ~/.poetry/env
```

Initialize your Python project:

```sh
poetry init --no-interaction
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

{{< figure src="/images/hypermodern-python-01/opera_crop06.jpg" link="/images/hypermodern-python-01/opera_crop06.jpg" >}}

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

{{< figure src="/images/hypermodern-python-01/opera_crop07.jpg" link="/images/hypermodern-python-01/opera_crop07.jpg" >}}

A [virtual environment](https://docs.python.org/3/tutorial/venv.html) gives your
project an isolated runtime environment, consisting of a specific Python version
and an independent set of installed Python packages. This way, the dependencies
of your current project do not interfere with the system-wide Python
installation, or other projects you're working on.

Poetry manages virtual environments for your projects. To see it in action,
install the skeleton package using [poetry
install](https://python-poetry.org/docs/cli/#install):

```sh
$ poetry install

Creating virtualenv hypermodern-python-rLESuZJY-py3.8 in …/pypoetry/virtualenvs
Updating dependencies
Resolving dependencies... (0.1s)

Writing lock file

Nothing to install or update

  - Installing hypermodern-python (0.1.0)
```

Poetry has now created a virtual environment dedicated to your project, and
installed your initial package into it. It has also created a so-called *lock
file*, named `poetry.lock`. You will learn more about this file in the next
section.

Let's run a Python session inside the new virtual environment, using [poetry
run](https://python-poetry.org/docs/cli/#run):

```python
$ poetry run python

Python 3.8.1 (default, Dec 29 2019, 16:28:23)
[GCC 8.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import hypermodern_python
>>> hypermodern_python.__version__
'0.1.0'
>>>
```

## Managing dependencies with Poetry

{{< figure src="/images/hypermodern-python-01/opera_crop08.jpg" link="/images/hypermodern-python-01/opera_crop08.jpg" >}}

Let's install the first dependency, the
[click](https://click.palletsprojects.com/) package. This Python package allows
you to create beautiful command-line interfaces in a composable way with as
little code as necessary. You can install dependencies using [poetry
add](https://python-poetry.org/docs/cli/#add):

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
constraint](https://python-poetry.org/docs/versions/) for the installed package:
`^7.0`. This means that users of the package need to have at least the current
release, `7.0`. The constraint also allows newer releases of the package, as
long as the version number does not indicate breaking changes. (After 1.0.0,
[Semantic Versioning](https://semver.org/) limits breaking changes to major
releases.)

By contrast, `poetry.lock` contains the exact version of `click` installed into
the virtual environment. Place this file under source control. It allows
everybody in your team to work with the same environment. It also helps you
[keep production and development environments as similar as
possible](https://12factor.net/dev-prod-parity).

Upgrading the dependency to a new minor or patch release is now as easy as
invoking [poetry update](https://python-poetry.org/docs/cli/#update) with the
package name:

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

{{< figure src="/images/hypermodern-python-01/opera_crop09.jpg" link="/images/hypermodern-python-01/opera_crop09.jpg" >}}

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
$ poetry run hypermodern-python

Hello, world!
```

You can also pass options to your script:

```sh
$ poetry run hypermodern-python --help

Usage: hypermodern-python [OPTIONS]

  The hypermodern Python project.

Options:
  --version  Show the version and exit.
  --help     Show this message and exit.
```

## Example: Consuming a REST API with requests

{{< figure src="/images/hypermodern-python-01/opera_crop10.jpg" link="/images/hypermodern-python-01/opera_crop10.jpg" >}}

Let's build an example application which prints random facts to the console. The
data is retrieved from the [Wikipedia
API](https://www.mediawiki.org/wiki/REST_API).

Install the [requests](https://requests.readthedocs.io/) package, the *de facto*
standard for making HTTP requests in Python:

```sh
poetry add requests
```

Next, replace the file `src/hypermodern-python/console.py` with the source code
shown below.

```python
# src/hypermodern_python/console.py
import textwrap

import click
import requests

from . import __version__


API_URL = "https://en.wikipedia.org/api/rest_v1/page/random/summary"


@click.command()
@click.version_option(version=__version__)
def main():
    """The hypermodern Python project."""
    with requests.get(API_URL) as response:
        response.raise_for_status()
        data = response.json()

    title = data["title"]
    extract = data["extract"]

    click.secho(title, fg="green")
    click.echo(textwrap.fill(extract))
```

Let's have a look at the imports at the top of the module first. 

```python
import textwrap

import click
import requests

from . import __version__
```

The [textwrap](https://docs.python.org/3/library/textwrap.html) module from the
standard library allows you to wrap lines when printing text to the console. We
also import the newly installed `requests` package. Blank lines serve to group
imports as recommended in [PEP
8](https://www.python.org/dev/peps/pep-0008/#imports) (standard library--third
party packages--local imports).

```python
API_URL = "https://en.wikipedia.org/api/rest_v1/page/random/summary"
```

The `API_URL` constant points to the [REST API](https://restfulapi.net/) of the
English Wikipedia, or more specifically, its `/page/random/summary` endpoint,
which returns the summary of a random Wikipedia article.

```python
with requests.get(API_URL) as response:
    response.raise_for_status()
    data = response.json()
```

In the body of the `main` function, the `requests.get` invocation sends an [HTTP
GET](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) request to the
Wikipedia API. The `with` statement ensures that the HTTP connection is closed
at the end of the block. Before looking at the response body, we check the HTTP
status code and raise an exception if it signals an error. The response body
contains the resource data in [JSON](https://www.json.org/) format, which can be
accessed using the `response.json()` method.

```python
title = data["title"]
extract = data["extract"]
```

We are only interested in the `title` and `extract` attributes, containing the
title of the Wikipedia page and a short plain text extract, respectively.

```python
click.secho(title, fg="green")
click.echo(textwrap.fill(extract))
```

Finally, we print the title and extract to the console, using the `click.echo`
and `click.secho` functions. The latter function allows you to specify the
foreground color using the `fg` keyword attribute. The `textwrap.fill` function
wraps the text in `extract` so that every line is at most 70 characters long.

Let's try it out!

```sh
$ poetry run hypermodern-python

Jägersbleeker Teich
The Jägersbleeker Teich in the Harz Mountains of central Germany is a
storage pond near the town of Clausthal-Zellerfeld in the county of
Goslar in Lower Saxony. It is one of the Upper Harz Ponds that were
created for the mining industry.
```

Feel free to play around with this a little. Here are some things you might want
to try:

- Display a friendly error message when the API is not reachable.
- Add an option to select the Wikipedia edition for another language.
- If you feel adventurous: auto-detect the user's preferred language edition,
  using [locale](https://docs.python.org/3.8/library/locale.html).

## Thanks for reading!

The next chapter is about adding unit tests to your project.

{{< figure src="/images/hypermodern-python-01/train.png" link="../hypermodern-python-02-testing" class="centered" >}}
<span class="centered">[Continue to the next chapter](../hypermodern-python-02-testing)</span>
