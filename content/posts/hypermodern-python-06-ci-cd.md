--- 
date: 2020-02-05T06:02:59+02:00
title: "Hypermodern Python Chapter 6: CI/CD"
description: "A guide to modern Python tooling with a focus on simplicity and minimalism."
draft: true
tags:
  - Python
  - GitHub Actions
  - Codecov
  - PyPI
  - TestPyPI
  - Read the Docs
  - Poetry
  - Nox
---

<!-- [Read this article on Medium](https://medium.com/@cjolowicz/hypermodern-python-6-ci-cd-xxxxxxxxxxxx) -->

{{< figure
    src="/images/hypermodern-python-06/nasa01.jpg"
    link="/images/hypermodern-python-06/nasa01.jpg"
>}}

In this sixth and last installment of the Hypermodern Python series,
I'm going to discuss
how to add continuous integration and delivery to your project
using GitHub Actions, Poetry, and Nox.[^1]
In the [previous chapter](../hypermodern-python-05-documentation), we discussed
how to add documentation.
(If you start reading here,
you can also 
[download the code](https://github.com/cjolowicz/hypermodern-python/archive/chapter05.zip) 
for the previous chapter.)

[^1]: The images in this chapter are
      artistic renderings of space colonies from the 1970s,
      made during a series of space colony summer studies
      held by the Princeton physicist Gerard O'Neill,
      with the help of NASA Ames Research Center and Stanford University
      (source:
      [NASA Ames Research Center](https://settlement.arc.nasa.gov/70sArt/art.html)
      via
      [The Public Domain Review](https://publicdomainreview.org/collection/space-colony-art-from-the-1970s)). 

Here are the topics covered in this chapter on
Continuous Integration and Delivery:

- [Continuous integration using GitHub Actions](#continuous-integration-using-github-actions)
- [Coverage reporting with Codecov](#coverage-reporting-with-codecov)
- [Uploading your package to PyPI](#uploading-your-package-to-pypi)
- [Documenting releases with Release Drafter](#documenting-releases-with-release-drafter)
- [Single-sourcing the package version](#single-sourcing-the-package-version)
- [Uploading your package to TestPyPI](#uploading-your-package-to-testpypi)
- [Hosting documentation at Read the Docs](#hosting-documentation-at-read-the-docs)
- [Conclusion](#conclusion)

Here is a full list of the articles in this series:

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation)
- [Chapter 6: CI/CD](../hypermodern-python-06-ci-cd) (this article)

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Each article in the guide corresponds to a set of commits in the GitHub repository:

- [View changes](https://github.com/cjolowicz/hypermodern-python/compare/chapter05...chapter06)
- [Download code](https://github.com/cjolowicz/hypermodern-python/archive/chapter06.zip)

## Continuous integration using GitHub Actions

{{< figure
    src="/images/hypermodern-python-06/nasa02.jpg"
    link="/images/hypermodern-python-06/nasa02.jpg"
>}}

*Continuous integration* (CI) helps you
automate the integration of code changes into your project.
When changes are pushed to the project repository,
the CI server verifies their correctness,
triggering tools such as unit tests, linters, or type checkers.

[Pull Requests](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests)
are an important building block in this workflow.
They let you propose a set of changes to the repository,
for example a specific bugfix or new feature.
When the pull request gets accepted,
its changes are merged into the target branch,
typically master.
GitHub displays Pull Requests with a green tick if they pass CI,
and with a red x if the CI pipeline failed.
In this way,
continuous integration functions as a gate commits need to pass
to enter the master branch.

You have a plethora of options when it comes to continuous integration.
Traditionally, many open-source projects have employed
[Travis CI](https://travis-ci.com).
Another popular choice are Microsoft's
[Azure Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/).
In this guide, we use GitHub's own offering,
[GitHub Actions](https://github.com/features/actions).

Configure GitHub Actions by
adding the following [YAML](https://yaml.org) file
to the `.github/workflows` directory:

```yaml
# .github/workflows/tests.yml
name: Tests
on: push
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-python@v1
      with:
        python-version: 3.8
        architecture: x64
    - run: pip install nox==2019.11.9
    - run: pip install poetry==1.0.3
    - run: nox
```

This file defines a so-called *workflow*.
A workflow is an automated process
executing a series of steps,
grouped into one or many jobs.
Workflows are triggered by events,
for example when someone pushes a commit to a repository,
or when someone creates a pull request, an issue, or a release.

The workflow above
is triggered on every push to your GitHub repository, and
executes the test suite using Nox.
It
is aptly named "Tests",
and consists of a single job
running on the latest supported [Ubuntu image](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/virtual-environments-for-github-hosted-runners#supported-runners-and-hardware-resources).
The job executes five steps,
using either official GitHub Actions or
invoking shell commands:

1. Check out your repository using [actions/checkout](https://github.com/actions/checkout).
2. Install Python 3.8 using [actions/setup-python](https://github.com/actions/setup-python).
3. Install Nox with pip.
4. Install Poetry with pip.
5. Run your test suite with Nox.

You can learn more about the workflow language and its supported keywords in the
[official reference](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions).

Every tool used in the CI process is pinned to a specific version.
The reasoning behind this is that
a CI process should be predictable and deterministic.
There is a common fallacy that
you should always use the latest version of a specific tool,
and therefore that tool should not be pinned.

By all means, use the latest versions of your tools, 
but be explicit about it.
This gives your project a higher level of auditability,
and prevents things from magically breaking and un-breaking in CI.
Upgrading tools in a dedicated pull request
also lets you investigate the impact of an upgrade,
rather than breaking your entire CI
when a new version becomes available.

The workflow currently uses Python 3.8 only,
but you should really
test your project on all Python versions it supports.
You can achieve this using a *build matrix*.
A build matrix lets you define variables,
such as for the operating system or for the Python version,
and specify multiple values for them.
Jobs can reference these variables,
and are instantiated for every combination of values.

Let's define a build matrix for the Python versions
supported by the project (Python 3.7 and 3.8).
Use the [strategy](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#jobsjob_idstrategy) and [matrix](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#jobsjob_idstrategymatrix) keywords
to define a build matrix with a `python-version` variable,
and reference the variable using the syntax `${{ matrix.python-version }}`:

```yaml {hl_lines=["7-10",15]}
# .github/workflows/tests.yml
name: Tests
on: push
jobs:
  tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.7', '3.8']
    name: Python ${{ matrix.python-version }}
    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-python@v1
      with:
        python-version: ${{ matrix.python-version }}
        architecture: x64
    - run: pip install nox==2019.11.9
    - run: pip install poetry==1.0.3
    - run: nox
```

If you commit and push now,
you can watch the workflow execute
under the *Actions* tab in your GitHub repository,
with a separate job for each Python version.

You should also add a GitHub Actions badge to your repository page.
The badge indicates whether the tests are passing or failing on the master branch,
and links to the GitHub Actions dashboard for your project.
It looks like this:

> [![tests](https://github.com/cjolowicz/hypermodern-python/workflows/tests/badge.svg)](https://github.com/cjolowicz/hypermodern-python/actions?workflow=tests)

Add the line below to the top of your `README.md` to display the badge:

```markdown
[![tests](https://github.com/<your-username>/hypermodern-python/workflows/tests/badge.svg)](https://github.com/<your-username>/hypermodern-python/actions?workflow=tests)
```

## Coverage reporting with Codecov

{{< figure
    src="/images/hypermodern-python-06/nasa03.jpg"
    link="/images/hypermodern-python-06/nasa03.jpg"
>}}

[Earlier](../hypermodern-python-02-testing#code-coverage-with-coveragepy),
we configured the test suite to fail
if code coverage drops below 100%.
A coverage reporting service offers some additional benefits,
by making code coverage more visible:

- Pull requests get automated comments
  with a quick rundown of how the changes affect coverage.
- The site visualizes code coverage
  for every commit and file in your repository,
  using graphs and code listings.
- You can display code coverage on your repository page,
  using a badge.

In this section,
we use [Codecov](https://codecov.io/) for coverage reporting;
another common option is [Coveralls](https://coveralls.io/).
Sign up at Codecov,
install their GitHub app,
and add your repository to Codecov.
The sign up process will guide you through these steps.

Add the official [codecov CLI](https://github.com/codecov/codecov-python)
to your development dependencies:

```sh
poetry add --dev codecov
```

Add the Nox session shown below.
The session exports the coverage data to
[cobertura](https://cobertura.github.io/cobertura/) XML format,
which is the format expected by Codecov.
It then uses `codecov` to upload the coverage data.

```python
# noxfile.py
@nox.session(python="3.8")
def coverage(session: Session) -> None:
    """Upload coverage data."""
    install_with_constraints(session, "coverage[toml]", "codecov")
    session.run("coverage", "xml", "--fail-under=0")
    session.run("codecov", *session.posargs)
```

Note that you need to disable the coverage minimum
using the command-line option `--fail-under=0`.
Otherwise, you would only get coverage reports
when coverage is at 100%,
defeating their very purpose.

Next, grant GitHub Actions access to upload to Codecov:

1. Go to your repository settings on Codecov,
   and copy the *Repository Upload Token*.
2. Go to your repository settings on GitHub,
   and add a secret named `CODECOV_TOKEN` with the token you just copied.

Add the following GitHub workflow to upload coverage data:

```yaml
# .github/workflows/coverage.yml
name: Coverage
on: push
jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-python@v1
      with:
        python-version: '3.8'
        architecture: x64
    - run: pip install nox==2019.11.9
    - run: pip install poetry==1.0.3
    - run: nox --sessions tests-3.8 coverage
      env:
        CODECOV_TOKEN: ${{secrets.CODECOV_TOKEN}}
```

In contrast to the Tests workflow,
the Coverage workflow only runs on Python 3.8.
It invokes Nox to execute the test suite and upload the coverage data,
providing the Codecov token as an environment variable.

Add the Codecov badge to your `README.md`:

```markdown
[![Codecov](https://codecov.io/gh/<your-username>/hypermodern-python/branch/master/graph/badge.svg)](https://codecov.io/gh/<your-username>/hypermodern-python)
```

The badge looks like this:

> [![Codecov](https://codecov.io/gh/cjolowicz/hypermodern-python/branch/master/graph/badge.svg)](https://codecov.io/gh/cjolowicz/hypermodern-python)

## Uploading your package to PyPI

{{< figure
    src="/images/hypermodern-python-06/nasa04.jpg"
    link="/images/hypermodern-python-06/nasa04.jpg"
>}}

[PyPI](https://pypi.org/) is the official Python package registry,
also known by its affectionate nickname
"[the Cheese Shop](https://en.wikipedia.org/wiki/Cheese_Shop_sketch)".
Uploading your package to PyPI allows others to
install it with [pip](https://pip.readthedocs.org/), like so:

```sh
pip install hypermodern-python
```

Before you can upload your Python package,
you need to generate *distribution packages*.
These are compressed archives which
an end-user can download and install on their system.
They come in two flavours:
source (or *sdist*) archives, and 
binary packages
in the [wheel](https://www.python.org/dev/peps/pep-0427/) format.
Poetry supports generating both with the
[poetry build](https://poetry.eustace.io/docs/cli/#build) command:

```sh
$ poetry build

Building hypermodern-python (0.1.0)
 - Building sdist
 - Built hypermodern-python-0.1.0.tar.gz

 - Building wheel
 - Built hypermodern_python-0.1.0-py3-none-any.whl
```

Poetry also supports uploading your package to PyPI,
with the
[poetry publish](https://poetry.eustace.io/docs/cli/#publish) command.
Here's what it would look like
if you uploaded your package now:

```sh
$ poetry publish

Publishing hypermodern-python (0.1.0) to PyPI
Username: <your-username>
Password:
 - Uploading hypermodern-python-0.1.0.tar.gz 100%
 - Uploading hypermodern_python-0.1.0-py3-none-any.whl 100%
```

In the remainder of this section,
we are going to automate the PyPI release process.
Automation helps you ensure
your Python package passes all checks
before it is published,
and keeps the build and upload process itself reliable.

Sign up at [PyPI](https://pypi.org/),
and enable two-factor authentication
for an additional layer of security.
Next, grant GitHub Actions permission to upload to PyPI:

1. Go to the Account Settings on PyPI,
   generate an API token,
   and copy it.
2. Go to the repository settings on GitHub,
   and add a secret named `PYPI_TOKEN`
   with the token you just copied.

The following GitHub workflow
uploads your package to PyPI
when you release it:

```yaml
# .github/workflows/release.yml
name: Release
on:
  release:
    types: [published]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-python@v1
      with:
        python-version: '3.8'
        architecture: x64
    - run: pip install nox==2019.11.9
    - run: pip install poetry==1.0.3
    - run: nox
    - run: poetry build
    - run: poetry publish --username=__token__ --password=${{ secrets.PYPI_TOKEN }}
```

The release workflow is triggered
when you publish a
[GitHub Release](https://help.github.com/en/github/administering-a-repository/about-releases).
GitHub Releases are based on
[Git tags](https://git-scm.com/book/en/v2/Git-Basics-Tagging),
which mark a specific point in your repository's history.
Here's how you can create and publish a release for your project:

1. Go to the Releases tab of your main repository page.
2. Click **Draft a new release**.
3. Enter the Git tag. By convention, these tags have the form `v<version>`.
4. Enter the title and description for your release.
5. Click **Publish release**.

The release workflow should now start building your package,
and upload the resulting artifacts to PyPI.
At the start of the workflow,
the test suite is run once more to
ensure that the package passes all checks.

Add a badge to `README.md` which links to your PyPI project page and displays
the latest release:

```markdown
[![PyPI](https://img.shields.io/pypi/v/hypermodern-python.svg)](https://pypi.org/project/hypermodern-python/)
```

The badge looks like this: 
[![PyPI](https://img.shields.io/pypi/v/hypermodern-python.svg)](https://pypi.org/project/hypermodern-python/)

## Documenting releases with Release Drafter

{{< figure
    src="/images/hypermodern-python-06/nasa05.jpg"
    link="/images/hypermodern-python-06/nasa05.jpg"
>}}

The [Release Drafter](https://github.com/release-drafter/release-drafter) action
drafts your next release notes
as pull requests are merged into master.
It does this by creating and maintaining a draft release.
When a pull request gets accepted,
the release description is updated to
include its title, author, and a link to the pull request itself.

When you're ready to make a release,
you simply need to add the tag name, and click *Publish*.
You're also free to edit the release description further,
for example to include a more general description of the release.

Add the following workflow to use Release Drafter:

```yaml
# .github/workflows/release-drafter.yml
name: Release Drafter
on:
  push:
    branches:
      - master
jobs:
  draft_release:
    runs-on: ubuntu-latest
    steps:
      - uses: release-drafter/release-drafter@v5.6.1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Another useful feature of Release Drafter is
to group pull requests based on labels you apply to them.
This would allow you, for example,
to have separate headings for features, bugfixes, and documentation
in the release notes,
using GitHub's *enhancement*, *bug*, and *documentation* labels.

The configuration file below goes somewhat further,
and is very loosely based on the
[Angular Commit Message Convention](https://github.com/angular/angular/blob/master/CONTRIBUTING.md#commit) and
[gitmoji](https://gitmoji.carloscuesta.me/).
You will need to add the remaining labels to your repository manually,
using *New Label* on the Issues tab of the repository.

```yaml
# .github/release-drafter.yml
categories:
  - title: ':boom: Breaking Changes'
    label: 'breaking'
  - title: ':package: Build System'
    label: 'build'
  - title: ':construction_worker: Continuous Integration'
    label: 'ci'
  - title: ':books: Documentation'
    label: 'documentation'
  - title: ':rocket: Features'
    label: 'enhancement'
  - title: ':beetle: Fixes'
    label: 'bug'
  - title: ':racehorse: Performance'
    label: 'performance'
  - title: ':hammer: Refactoring'
    label: 'refactoring'
  - title: ':fire: Removals and Deprecations'
    label: 'removal'
  - title: ':lipstick: Style'
    label: 'style'
  - title: ':rotating_light: Testing'
    label: 'testing'
template: |
  ## What’s Changed

  $CHANGES
```

## Single-sourcing the package version

{{< figure
    src="/images/hypermodern-python-06/nasa06.jpg"
    link="/images/hypermodern-python-06/nasa06.jpg"
>}}

Before creating the next release of your package,
you need to bump the version of your package.
Use [poetry version](https://poetry.eustace.io/docs/cli/#version)
to update the version declared in `pyproject.toml`:

```sh
poetry version <version>
```

You can pass the new version explicitly,
or a rule such as `major`, `minor`, or `patch`.
In a nutshell, increment the major version
if the release contains breaking changes,
the patch level if the release contains only bugfixes,
and the minor version in all other cases.
This assumes that your project has a stable and public API,
and a version number greater than or equal to `1.0.0`.
For more details,
refer to the [Semantic Versioning](https://semver.org/) standard.

As it stands, you also need to
update the version declared inside your package's `__init__.py`.
We can streamline this process even further,
by determining the version automatically
using the installed package metadata.
This is possible in Python 3.8 using the standard
[importlib.metadata](https://docs.python.org/3/library/importlib.metadata.html)
library,
and in Python 3.7 and earlier using its backport
[importlib_metadata](https://importlib-metadata.readthedocs.io/en/latest/).

Add the backport to your dependencies,
for older Python versions only:

```sh
poetry add --python="<3.8" importlib_metadata
```

Determining the package version should really
be as simple as the following:

```python
# src/hypermodern_python/__init__.py
"""The hypermodern Python project."""
from importlib.metadata import version


__version__ = version(__name__)
```

However, the actual implementation is slightly more complicated,
due to the fact
that the import path depends on the Python version,
and the possibility that the package has not been installed
(although this should rarely happen,
[thanks to src layout](https://blog.ionelmc.ro/2014/05/25/python-packaging/)).
For the same reasons, we need to disable
type checking and coverage for certain lines.

```python
# src/hypermodern_python/__init__.py
"""The hypermodern Python project."""
try:
    from importlib.metadata import version, PackageNotFoundError  # type: ignore
except ImportError:  # pragma: no cover
    from importlib_metadata import version, PackageNotFoundError  # type: ignore


try:
    __version__ = version(__name__)
except PackageNotFoundError:  # pragma: no cover
    __version__ = "unknown"
```

With this in place,
`pyproject.toml` has become
the single source of truth for your package version.

## Uploading your package to TestPyPI

{{< figure
    src="/images/hypermodern-python-06/nasa07.jpg"
    link="/images/hypermodern-python-06/nasa07.jpg"
>}}

[TestPyPI](https://test.pypi.org)
is a separate instance of the Python Package Index
that allows you to try distribution tools and processes
without affecting the real index.

The following GitHub workflow
builds and uploads your package to TestPyPI
from the master branch of your repository:

```yaml
# .github/workflows/test-pypi.yml
name: TestPyPI
on:
  push:
    branches:
      - master
jobs:
  test_pypi:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-python@v1
      with:
        python-version: '3.8'
        architecture: x64
    - run: pip install poetry==1.0.3
    - run: >-
        poetry version patch &&
        version=$(poetry version | awk '{print $2}') &&
        poetry version $version.dev.$(date +%s)
    - run: poetry build
    - uses: pypa/gh-action-pypi-publish@v1.0.0a0
      with:
        user: __token__
        password: ${{ secrets.TEST_PYPI_TOKEN }}
        repository_url: https://test.pypi.org/legacy/
```

TestPyPI does not allow you to overwrite an existing package version.
The workflow therefore bumps the version and
appends a suffix of the form `.dev.<timestamp>`,
indicating a
[developmental release](https://www.python.org/dev/peps/pep-0440/#developmental-releases).
The package is then built
and uploaded using the
[PyPI publish GitHub Action](https://github.com/marketplace/actions/pypi-publish)
of the [Python Packaging Authority](https://www.pypa.io/).

## Hosting documentation at Read the Docs

{{< figure
    src="/images/hypermodern-python-06/nasa08.jpg"
    link="/images/hypermodern-python-06/nasa08.jpg"
>}}

[Read the Docs](https://readthedocs.org/) hosts documentation for countless
open-source Python projects. The hosting service also takes care of rebuilding
the documentation when you update your project. Users can browse documentation
for every published version, as well as the latest development version.

Create the `.readthedocs.yml` configuration file:

```yaml
# .readthedocs.yml
version: 2
sphinx:
  configuration: docs/conf.py
formats: all
python:
  version: 3.7
  install:
    - requirements: docs/requirements.txt
    - path: .
```

The `install` section in the configuration file tells Read the Docs how to
install your package and its documentation dependencies. Read the Docs does not
support installation of dependencies using Poetry directly. Luckily, pip can
install any package with a standard `pyproject.toml` file, and will use Poetry
behind the scenes.

On the other hand,
the development dependencies on Sphinx and its extensions
are not declared in the package.
(And even if they were,
dependencies declared in the package are not pinned,
so we might run into problems due to
slightly different build environments.)
Uploading the documentation from a GitHub workflow would solve this nicely,
but this is [currently not supported](https://github.com/readthedocs/readthedocs.org/issues/1083).
So let's be pragmatic and
duplicate the documentation dependencies
using a separate requirements file:

```python
# docs/requirements.txt
sphinx==2.3.1
sphinx-autodoc-typehints==1.10.3
```

Sign up at Read the Docs, and import your GitHub repository, using the button
*Import a Project*. Read the Docs automatically starts building your
documentation. When the build has completed, your documentation will have a
public URL like this:

> https://hypermodern-python.readthedocs.io/

You can display the documentation link on PyPI by including it in your package
configuration file:

```toml
# pyproject.toml
[tool.poetry]
documentation = "https://hypermodern-python.readthedocs.io"
```

Let's also add the link to the GitHub repository page, by adding a Read the Docs
badge to `README.md`:

```markdown
[![Read the Docs](https://readthedocs.org/projects/hypermodern-python/badge/)](https://hypermodern-python.readthedocs.io/)
```

The badge looks like this: [![Read the
Docs](https://readthedocs.org/projects/hypermodern-python/badge/)](https://hypermodern-python.readthedocs.io/)

## Conclusion

{{< figure
    src="/images/hypermodern-python-06/nasa09.jpg"
    link="/images/hypermodern-python-06/nasa09.jpg"
>}}

Thank you for reading this far. This chapter concludes the Hypermodern Python
series.

I would like to acknowledge these wonderful people
for providing their feedback to early drafts:
[Alex Grönholm](https://github.com/agronholm),
[Bruno Oliveira](https://github.com/nicoddemus),
[Holger Krekel](https://github.com/hpk42),
[Hynek Schlawack](http://hynek.me/),
[Ian Stapleton Cordasco](https://ian.stapletoncordas.co/),
Oliver Gondring,
[Phil Jones](https://pgjones.dev/),
[Pierre-Jean Vincent](https://github.com/Iracinder),
[Ronny Pfannschmidt](https://github.com/RonnyPfannschmidt),
[Terrence Reilly](https://github.com/terrencepreilly),
[Thea Flowers](https://thea.codes/).
Mistakes are mine.

The idea to use
[retrofuturist](https://www.reddit.com/r/RetroFuturism/)
imagery as a visual theme for this blog
came from Marianna Jolowicz,
whom I also wish to thank for supporting my spending endless hours on it,
and for turning my fantasy language into actual English.

This series of articles is dedicated to my father
who introduced me to programming in the 1980s,
and who is an avid chess book collector.
His copy of *Die hypermoderne Schachpartie* (The hypermodern chess game) --
written by [Savielly Tartakower](https://en.wikipedia.org/wiki/Savielly_Tartakower) in 1925
to describe recent advances in chess theory --
inspired the title of this guide.

{{< figure src="/images/hypermodern-python-06/boat.jpg" class="centered" >}}
