--- 
date: 2021-01-01T08:00:00+02:00
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

This article series is a guide to modern Python tooling
with a focus on simplicity and minimalism.[^1]
It walks you through the creation of a complete Python project structure, with
unit tests,
static analysis,
type-checking,
documentation, and
continuous integration and delivery.

[^1]: The title of this guide is inspired by the book
    *Die hypermoderne Schachpartie* (The hypermodern chess game),
    written by [Savielly Tartakower] in 1924.
    It surveys the revolution that had taken place in chess theory in the decade after the First World War.
    The images in this chapter are details from the hand-colored print
    *Le Sortie de l'opéra en l'an 2000*
    (Leaving the opera in the year 2000)
    by Albert Robida, ca 1902
    (source: [Library of Congress][Albert Robida]).

[Savielly Tartakower]: https://en.wikipedia.org/wiki/Savielly_Tartakower
[Albert Robida]: http://www.loc.gov/pictures/item/2007676247/

The guide is aimed at intermediate developers
and presumes basic knowledge of the Python programming language.
You may also find it an interesting read if you are already a seasoned Python developer,
but want to see how to implement best practices with a new generation of Python tools.
If you are a complete beginner, I would recommend to read an introductory book first,
for example [Automate the Boring Stuff with Python],
or any of the learning resources listed in [The Hitchhiker's Guide to Python].

[Automate the Boring Stuff with Python]: https://automatetheboringstuff.com/
[The Hitchhiker's Guide to Python]: https://docs.python-guide.org/intro/learning/

### Requirements

You need a recent Linux, Unix, or Mac system with
[bash], [curl], and [git] for this tutorial,
or Windows 10 with [Git for Windows].

[bash]: https://www.gnu.org/software/bash/
[curl]: https://curl.haxx.se
[git]: https://www.git-scm.com
[Git for Windows]: https://gitforwindows.org/

## Overview

{{< figure src="/images/hypermodern-python-01/opera_crop04.jpg" link="/images/hypermodern-python-01/opera_crop04.jpg" link="/images/hypermodern-python-01/opera_crop04.jpg" >}}

In this first chapter,
we set up a Python project using [pyenv] and [Poetry].
Our example project is a simple command-line application
that uses the Wikipedia API to display random facts on the console.

[pyenv]: https://github.com/pyenv/pyenv
[Poetry]: https://python-poetry.org/

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

This guide has a companion repository: [cjolowicz/hypermodern-python].
Each article in the guide corresponds to a set of commits in the GitHub repository.

[cjolowicz/hypermodern-python]: https://github.com/cjolowicz/hypermodern-python

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/initial...chapter01)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter01.zip)

## Setting up a GitHub repository

{{< figure src="/images/hypermodern-python-01/opera_crop02.jpg" link="/images/hypermodern-python-01/opera_crop02.jpg" >}}

For the purposes of this guide,
[GitHub] is used to host the public git repository for your project.
Other popular options are [GitLab] and [BitBucket].
Create a repository, and populate it with `README.md` and `LICENSE` files.
For this project, I will use the [MIT license],
a simple permissive license.

[GitHub]: https://github.com
[GitLab]: https://gitlab.com/
[BitBucket]: https://bitbucket.org/
[MIT license]: https://choosealicense.com/licenses/mit/

> *Throughout this guide, replace `hypermodern-python` with the name of your own repository.
> Choose a different name to avoid a name collision on PyPI.*

Clone the repository to your machine, and `cd` into it:

```sh
git clone git@github.com:<your-username>/hypermodern-python.git
cd hypermodern-python
```

As you follow the rest of this guide,
create a series of [small, atomic commits][git-best-practises]
documenting your steps.
Use `git status` to discover files generated by commands shown in the guide.

[git-best-practices]: https://deepsource.io/blog/git-best-practices/

Let's continue by setting up the developer environment.

## Installing Python with pyenv

{{< figure src="/images/hypermodern-python-01/opera_crop03.jpg" link="/images/hypermodern-python-01/opera_crop03.jpg" >}}

When maintaining Python projects,
you need to be able to check them against different versions of Python.
These should be the latest releases of every major Python version
for which your project declares support via its package metadata.
Bonus points if you also check against the development version of the upcoming Python version (3.10).

If you are working natively on Windows,
it is recommended to install the official binary for each desired Python version from [python.org].
In this guide, we will be using the current release of Python (3.9) and its predecessor (3.8).
During installation, do not check the *Add Python to PATH* box,
unless you wish to use that particular Python version as your system default.
During development, use the [Python launcher for Windows] to run the desired Python version:

```sh
> py -3.9 --version
Python 3.9.0

> py -3.8 --version
Python 3.8.6
```

[python.org]: https://www.python.org/downloads/
[Python launcher for Windows]: https://docs.python.org/3/using/windows.html#python-launcher-for-windows

If you are working on Linux, Unix, or Mac, or using the [Windows Subsystem for Linux],
you can install multiple Python versions with [pyenv], a Python version manager.
Install pyenv like this:

[Windows Subsystem for Linux]: https://docs.microsoft.com/en-us/windows/wsl/install-win10
[pyenv]: https://github.com/pyenv/pyenv

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

Install the Python build dependencies for your platform,
using one of the commands listed in the [official instructions](https://github.com/pyenv/pyenv/wiki/Common-build-problems).
For example, on a recent [Ubuntu] this would be:

```sh
sudo apt update && sudo apt install -y build-essential libssl-dev zlib1g-dev \
libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev \
libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python-openssl git
```

You're ready to install the latest Python releases. This may take a while:

```sh
pyenv install 3.9.0
pyenv install 3.8.6
```

Make your fresh Pythons available inside the repository:

```sh
pyenv local 3.9.0 3.8.6
```

Congratulations! You have access to the latest and greatest of Python:

```sh
$ python --version
Python 3.9.0

$ python3.8 --version
Python 3.8.6
```

Python 3.9.0 is the default version and can be invoked as `python`, but both
versions are accessible as `python3.8` and `python3.9`, respectively.

<!--
If you are working on Linux,
you may be able to use the package manager of your Linux distribution.
For example, on [Fedora] you can install multiple Python versions from binary packages with ``dnf``.
On a [Debian]-based Linux distribution such as [Ubuntu],
you can install multiple Python versions from the [deadsnakes PPA].
You can then invoke a specific Python version from the console using its versioned name, such as ``python3.9``.

[Fedora]: https://developer.fedoraproject.org/tech/languages/python/multiple-pythons.html
[Debian]: https://www.debian.org/
[Ubuntu]: https://ubuntu.com/
[deadsnakes PPA]: https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa

This section introduces [pyenv], a Python version manager,
a flexible and portable alternative that works on Linux, Unix, and Mac systems,
as well as Windows Subsystem for Linux.
Unlike the installation methods mentioned above,
pyenv downloads and builds Python from source.
While most of this happens behind the scenes,
it does mean that installation takes quite a bit longer.
Also, you will need to install the Python build requirements on your system.
On the plus side, pyenv covers a wide variety of Python implementations, interpreter versions, and target platforms,
and makes it easy to switch between Python versions.
-->

## Setting up a Python project using Poetry

{{< figure src="/images/hypermodern-python-01/opera_crop05.jpg" link="/images/hypermodern-python-01/opera_crop05.jpg" >}}

[Poetry] is a Python packaging and dependencies manager,
similar in spirit to JavaScript's `npm` and Rust's `cargo`.
Common alternatives are [setuptools], [pipenv], and [flit].
<!-- somewhat less common ones [hatch], [pyflow], and [dephell]. -->

[Poetry]: https://python-poetry.org/ 
[setuptools]: http://setuptools.readthedocs.io
[pipenv]: https://pipenv.pypa.io/en/latest/
[flit]: https://github.com/takluyver/flit
[hatch]: https://github.com/ofek/hatch
[pyflow]: https://github.com/David-OConnor/pyflow
[dephell]: https://dephell.org/

Install [Poetry] by downloading and running [get-poetry.py]:

[get-poetry.py]: https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py

```sh
python get-poetry.py
```

Open a new shell, or configure your current shell using the following command:

```sh
source ~/.poetry/env
```

Inside your repository, initialize a new Python project:

```sh
poetry init --no-interaction
```

This command will create a `pyproject.toml` file,
the Python package configuration file specified in [PEP 517] and [518][PEP 518].

[PEP 517]: https://www.python.org/dev/peps/pep-0517/
[PEP 518]: https://www.python.org/dev/peps/pep-0518/

```toml
# pyproject.toml
[tool.poetry]
name = "hypermodern-python"
version = "0.1.0"
description = ""
authors = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python = "^3.9"

[tool.poetry.dev-dependencies]

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

[TOML]: https://github.com/toml-lang/toml

There you go: One declarative file in [TOML] format,
containing the entire package configuration.
Let's add some metadata to the package:

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

Poetry added a dependency on Python 3.9,
because this is the Python version you ran it in.
Support the previous release as well by changing this to Python 3.8:

```toml
[tool.poetry.dependencies]
python = "^3.8"
```

The caret (`^`) in front of the version number means "up to the next major release".
In other words, you are promising that your package won't break when users upgrade to Python 3.9 or 3.10,
but giving no guarantees for its use with a future [Python 4.0].

[Python 4.0]: https://twitter.com/gvanrossum/status/1306082472443084801?s=20

## Creating a package in src layout

{{< figure src="/images/hypermodern-python-01/opera_crop06.jpg" link="/images/hypermodern-python-01/opera_crop06.jpg" >}}

Let's create an initial skeleton package.
Organize your package in [src layout], like this:

[src layout]: https://hynek.me/articles/testing-packaging/

```sh
.
├── pyproject.toml
└── src
    └── hypermodern_python
        └── __init__.py

2 directories, 2 files
```

The `hypermodern_python` directory contains a single empty file named `__init__.py`,
which declares it as an importable [Python package].

[Python package]: https://docs.python.org/3/tutorial/modules.html#packages

Use underscores for the package name `hypermodern_python`,
instead of any hyphens in the repository name `hypermodern-python`,
to ensure that the package name is a valid Python identifier.
These naming conventions are also known as [snake case] and [kebab case].

[snake case]: https://en.wikipedia.org/wiki/Snake_case
[kebab case]: https://en.wiktionary.org/wiki/Kebab_case

> Replace `hypermodern-python` with the name of your own repository,
> to avoid a name collision on PyPI.

## Managing virtual environments with Poetry

{{< figure src="/images/hypermodern-python-01/opera_crop07.jpg" link="/images/hypermodern-python-01/opera_crop07.jpg" >}}

A [virtual environment] gives your project an isolated runtime environment,
consisting of a specific Python version and an independent set of installed Python packages.
This way, the dependencies of your current project do not interfere with the system-wide Python installation,
or other projects you're working on.

[virtual environment]: https://docs.python.org/3/tutorial/venv.html

Poetry manages virtual environments for your projects.
To see it in action,
install the skeleton package using the command [poetry install]:

[poetry install]: https://python-poetry.org/docs/cli/#install

```sh
$ poetry install

Creating virtualenv hypermodern-python-PsI7ns-N-py3.9 in /root/.cache/pypoetry/virtualenvs
Updating dependencies
Resolving dependencies... (0.1s)

Writing lock file

Installing the current project: hypermodern-python (0.1.0)
```

Poetry created a virtual environment dedicated to your project,
and installed your initial package into it.
It also created a so-called *lock file*, named `poetry.lock`.
You will learn more about this file in the next section.

Let's run a Python session inside the new virtual environment,
using the command [poetry run]:

[poetry run]: https://python-poetry.org/docs/cli/#run

```python
$ poetry run python

Python 3.9.0 (default, Oct 13 2020, 20:14:06)
[GCC 8.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import hypermodern_python
>>>
```

There isn't much more we can do at the moment,
because our package does not contain any Python code yet.

## Managing dependencies with Poetry

{{< figure src="/images/hypermodern-python-01/opera_crop08.jpg" link="/images/hypermodern-python-01/opera_crop08.jpg" >}}

[click]: https://click.palletsprojects.com/

Let's install the first dependency, the [click] package.
This Python package allows you to create beautiful command-line interfaces
in a composable way with as little code as necessary.
You can install dependencies using the command [poetry add]:

[poetry add]: https://python-poetry.org/docs/cli/#add

```sh
$ poetry add click

Using version ^7.1.2 for click

Updating dependencies
Resolving dependencies... (0.1s)

Writing lock file

Package operations: 1 install, 0 updates, 0 removals

  - Installing click (7.1.2)
```

Several things are happening here:

- The package is downloaded and installed into the virtual environment.
- The installed version is registered in the lock file `poetry.lock`.
- A more general version constraint is added to `pyproject.toml`.

[version constraint]: https://python-poetry.org/docs/versions/

The dependency entry in `pyproject.toml` contains a [version constraint] for the installed package: `^7.1.2`.
This means that users of the package need to have at least the current release, `7.1.2`.
The constraint also allows newer releases of the package,
as long as the version number does not indicate breaking changes.
([Semantic Versioning] limits breaking changes to major releases, once the version reaches `1.0.0`.)

[Semantic Versioning]: https://semver.org/

By contrast, `poetry.lock` contains the exact version of `click` installed into the virtual environment.
Place this file under source control.
It allows everybody in your team to work with the same environment.
It also helps you [keep production and development environments as similar as possible][dev-prod-parity].

[dev-prod-parity]: https://12factor.net/dev-prod-parity

Upgrading the dependency to a new minor or patch release is now as easy as
invoking [poetry update] with the package name:

[poetry update]: https://python-poetry.org/docs/cli/#update

```sh
poetry update click
```

To upgrade to a new major release, use the following command instead:

```sh
poetry add click@latest
```

This will also update the version constraint.

## Command-line interfaces with click

{{< figure src="/images/hypermodern-python-01/opera_crop09.jpg" link="/images/hypermodern-python-01/opera_crop09.jpg" >}}

Time to add some actual code to the package.
As you may have guessed, we're going to create a console application using `click`.
Create a file named `__main__.py` next to `__init__.py` with the following contents:

```python
# src/hypermodern_python/__main__.py
import click

@click.command()
@click.version_option()
def main():
    """The hypermodern Python project."""
    click.echo("Hello, world!")
```

This module defines a minimal command-line application, supporting `--help` and `--version` options.

Register the script in `pyproject.toml`:

```toml
[tool.poetry.scripts]
hypermodern-python = "hypermodern_python.__main__:main"
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

Prefixing the command by `poetry run` is only required during development,
before the application has been installed into its final location.

You can also pass options to your script.
Let's use the `--help` option to print a usage message:

```sh
$ poetry run hypermodern-python --help

Usage: hypermodern-python [OPTIONS]

  The hypermodern Python project.

Options:
  --version  Show the version and exit.
  --help     Show this message and exit.
```

As you can see, 
`click` used the documentation string at the start of the `main` function
as a general description of the command-line interface.

The `--version` option uses the package metadata to determine the version of our console application:

```sh
$ poetry run hypermodern-python --version

hypermodern-python, version 0.1.0
```

Using the name `__main__.py` is not a mere convention.
If you invoke python with the option `-m` followed by the name of your package,
it will execute this module.
To leverage this, add two lines to the bottom of the module, as shown below:

```python
# src/hypermodern_python/__main__.py
import click

@click.command()
@click.version_option()
def main():
    """The hypermodern Python project."""
    click.echo("Hello, world!")
    
if __name__ == "__main__":
    main(prog_name="hypermodern-python")
```

Here we simply invoke `main`, passing the program name explicitly.
(It would be displayed as `__main__` otherwise.)
Let's try it:

```sh
$ poetry run python -m hypermodern_python

Hello, world!
```

Don't worry if you don't see why people would want to do this.
It may be more to type, but [sometimes][xkcd1987]
it's nice to be able to specify exactly which Python you want to run an application with.
Besides, this technique does not require an entry-point script at all.

[xkcd1987]: https://xkcd.com/1987/

## Example: Consuming a REST API with requests

{{< figure src="/images/hypermodern-python-01/opera_crop10.jpg" link="/images/hypermodern-python-01/opera_crop10.jpg" >}}

Let's build an example application that prints random facts to the console.
The data is retrieved from the [Wikipedia API].

[Wikipedia API]: https://www.mediawiki.org/wiki/REST_API

Install [httpx], an HTTP client library:

[httpx]: https://www.python-httpx.org/

```sh
poetry add httpx
```

If you are familiar with [requests], httpx uses a very similar API.

[requests]: https://requests.readthedocs.io/

Next, replace the file `src/hypermodern-python/__main__.py` with the source code shown below.

```python
# src/hypermodern_python/__main__.py
import textwrap

import click
import httpx

API_URL = "https://en.wikipedia.org/api/rest_v1/page/random/summary"

@click.command()
@click.version_option()
def main():
    """The hypermodern Python project."""
    response = httpx.get(API_URL)
    response.raise_for_status()
    data = response.json()

    title = data["title"]
    extract = data["extract"]

    click.secho(title, fg="green")
    click.echo(textwrap.fill(extract))

if __name__ == "__main__":
    main(prog_name="hypermodern-python")
```

Let's have a look at the imports at the top of the module first. 

```python
import textwrap

import click
import httpx
```

[textwrap]: https://docs.python.org/3/library/textwrap.html
[PEP 8]: https://www.python.org/dev/peps/pep-0008/#imports

The [textwrap] module from the standard library allows you to wrap lines when printing text to the console.
We also import the newly installed `httpx` package.
Blank lines serve to group imports as recommended in [PEP 8] (standard library--third party packages--local imports).

```python
API_URL = "https://en.wikipedia.org/api/rest_v1/page/random/summary"
```

[REST API]: https://restfulapi.net/

The `API_URL` constant points to the [REST API] of the English Wikipedia,
or more specifically, its `/page/random/summary` endpoint,
which returns the summary of a random Wikipedia article.

```python
response = httpx.get(API_URL)
response.raise_for_status()
data = response.json()
```

[HTTP GET]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods
[JSON]: https://www.json.org/

In the body of the `main` function,
the `httpx.get` invocation sends an [HTTP GET] request to the Wikipedia API.
Before looking at the response body, we check the HTTP status code and raise an exception if it signals an error.
The response body contains the resource data in [JSON] format,
which can be accessed using the `response.json()` method.

```python
title = data["title"]
extract = data["extract"]
```

We are only interested in the `title` and `extract` attributes,
containing the title of the Wikipedia page and a short plain text extract, respectively.

```python
click.secho(title, fg="green")
click.echo(textwrap.fill(extract))
```

Finally, we print the title and extract to the console, using the `click.echo` and `click.secho` functions.
The latter function allows you to specify the foreground color using the `fg` keyword attribute.
The `textwrap.fill` function wraps the text in `extract` so that every line is at most 70 characters long.

Let's try it out!

```sh
$ poetry run hypermodern-python

Jägersbleeker Teich
The Jägersbleeker Teich in the Harz Mountains of central Germany is a
storage pond near the town of Clausthal-Zellerfeld in the county of
Goslar in Lower Saxony. It is one of the Upper Harz Ponds that were
created for the mining industry.
```

Feel free to play around with this a little.
Here are some things you might want to try:

- Display a friendly error message when the API is not reachable.
- Add an option to select the Wikipedia edition for another language.
- If you feel adventurous: auto-detect the user's preferred language edition, using [locale].

[locale]: https://docs.python.org/3.8/library/locale.html

## Thanks for reading!

The next chapter is about adding unit tests to your project.

{{< figure src="/images/hypermodern-python-01/train.png" link="../hypermodern-python-02-testing" class="centered" >}}
<span class="centered">[Continue to the next chapter]</span>

[Continue to the next chapter]: ../hypermodern-python-02-testing
