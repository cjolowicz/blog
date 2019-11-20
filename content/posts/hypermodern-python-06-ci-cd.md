--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 6: CI/CD"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - GitHub Actions
  - Codecov
  - PyPI
  - readthedocs
---

In this sixth and last installment of the Hypermodern Python series, I'm going
to discuss how to add continuous integration and delivery to your project.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 4: Typing](../hypermodern-python-04-typing)
- [Chapter 5: Documentation](../hypermodern-python-05-documentation)
- [Chapter 6: CI/CD](../hypermodern-python-06-ci-cd) (this article)
- [Appendix: Docker](../hypermodern-python-07-deployment)

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Continuous integration using GitHub Actions](#continuous-integration-using-github-actions)
- [Coverage reporting with Codecov](#coverage-reporting-with-codecov)
- [Uploading your package to PyPI](#uploading-your-package-to-pypi)
- [A typical release process](#a-typical-release-process)

<!-- markdown-toc end -->

This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python).
Here is the link for the changes contained in this chapter:

▶ **[View code](https://github.com/cjolowicz/hypermodern-python/compare/chapter05...chapter06)**

## Continuous integration using GitHub Actions

*Continuous integration* (CI) helps you automate the integration of code changes
into your project. The CI server verifies the correctness of changes, triggering
tools such as unit tests, linters, or type checkers. GitHub displays Pull
Requests with a green tick if they pass CI, and with a red x if the CI pipeline
failed.

You have a plethora of options when it comes to continuous integration.
Traditionally, many open-source projects have employed [Travis
CI](https://travis-ci.com). Another popular choice are Microsoft's [Azure
Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/). In
this guide, you are going to use GitHub's own offering, [GitHub
Actions](https://github.com/features/actions).

Configure GitHub Actions by adding the following [YAML](https://yaml.org) file
to the `.github/workflows` directory:

```yaml
# .github/workflows/tests.yml
name: tests
on: push
jobs:
  test:
    name: nox
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@1.0.0
    - uses: excitedleigh/setup-nox@0.1.0
    - uses: dschep/install-poetry-action@v1.2
      with:
        preview: true
    - run: nox
    - run: poetry build
```

This file defines a so-called *workflow*, which is triggered on every push to
your GitHub repository, and runs on the latest supported Ubuntu image. The
workflow consists of five steps, either using preexisting GitHub Actions or
invoking shell commands:

1. Fetch and check out your code, using [actions/checkout](https://github.com/actions/checkout).
2. Activate Python and install Nox using [excitedleigh/setup-nox](https://github.com/excitedleigh/setup-nox).
3. Install Poetry using [dschep/install-poetry-action](https://github.com/dschep/install-poetry-action).
4. Run your test suite by invoking `nox`.
5. Build the package by invoking [poetry
   build](https://poetry.eustace.io/docs/cli/#build).

You should also add a GitHub Actions badge to your repository page. The badge
indicates whether the tests are passing or failing on the master branch, and
links to the GitHub Actions dashboard for your project. It looks like this:

> [![tests](https://github.com/cjolowicz/hypermodern-python/workflows/tests/badge.svg)](https://github.com/cjolowicz/hypermodern-python/actions?workflow=tests)

Add the line below to the top of your `README.md` to display the badge:

```markdown
[![tests](https://github.com/<your-username>/hypermodern-python/workflows/tests/badge.svg)](https://github.com/<your-username>/hypermodern-python/actions?workflow=tests)
```

## Coverage reporting with Codecov

Let's also display code coverage right on your GitHub repository page. We will
use [Codecov](https://codecov.io/) for this; another common option is
[Coveralls](https://coveralls.io/). Sign up at Codecov, install their GitHub
app, and add your repository to Codecov. The sign up process will guide you
through these steps.

Add the Nox session shown below. This session exports the coverage data to
[cobertura](https://cobertura.github.io/cobertura/) XML format, which is the
format expected by Codecov. It then uses the official
[codecov CLI](https://github.com/codecov/codecov-python) to upload the coverage
data.

```python
# noxfile.py
@nox.session(python="3.8")
def coverage(session: Session) -> None:
    """Upload coverage data."""
    session.install("coverage", "codecov")
    session.run("coverage", "xml")
    session.run("codecov", *session.posargs)
```

Next, grant GitHub Actions access to upload to Codecov:

1. On Codecov, copy the *Repository Upload Token* from the settings page of your
   repository.
2. On GitHub, go to the settings page of your repository, and add a secret named
   `CODECOV_TOKEN` with the token you copied.

Invoke the session from the GitHub Actions workflow, providing the
`CODECOV_TOKEN` secret as an environment variable:

```yaml
# .github/workflows/tests.yml
name: tests
on: push
jobs:
  test:
    name: nox
    runs-on: ubuntu-latest
    steps:
...
    - run: nox
    - run: nox -e coverage
      env:
        CODECOV_TOKEN: ${{secrets.CODECOV_TOKEN}}
    - run: poetry build
```

Finally, add the Codecov badge to your `README.md`:

```markdown
[![Codecov](https://codecov.io/gh/<your-username>/hypermodern-python/branch/master/graph/badge.svg)](https://codecov.io/gh/<your-username>/hypermodern-python)
```

The badge looks like this:

> [![Codecov](https://codecov.io/gh/cjolowicz/hypermodern-python/branch/master/graph/badge.svg)](https://codecov.io/gh/cjolowicz/hypermodern-python)

## Uploading your package to PyPI

[PyPI](https://pypi.org/) is the official Python package registry, also known by
its affectionate nickname "[the Cheese
Shop](https://en.wikipedia.org/wiki/Cheese_Shop_sketch)". Uploading your package
to PyPI allows others to install it with [pip](https://pip.readthedocs.org/),
like so:

```sh
pip install hypermodern-python
```

Poetry supports uploading your package to PyPI with the command [poetry
publish](https://poetry.eustace.io/docs/cli/#publish).

Sign up at PyPI, if you do not have an account yet. Next, grant GitHub Actions
permission to upload to PyPI:

1. On PyPI, go to the Account Settings page. Generate an API token, and copy it.
2. On GitHub, go to the repository settings. Add a secret named `PYPI_TOKEN`
   with the token you copied.

Add the following lines to the bottom of your GitHub workflow:

```yaml
# .github/workflows/tests.yml
...
    - if: github.event_name == 'push' && startsWith(github.event.ref, 'refs/tags')
      run: |
        poetry publish --username=__token__ --password=${{ secrets.PYPI_TOKEN }}
```

You can now trigger a PyPI release by creating and pushing an annotated Git tag.
By convention, these tags have the form `v<version>`:

```sh
git tag --message="hypermodern-python 0.1.0" v0.1.0
git push --follow-tags
```

When releasing the next version of your package, you will need to bump the
version of your package. Use [poetry
version](https://poetry.eustace.io/docs/cli/#version) to update the version
declared in `pyproject.toml`:

```sh
poetry version minor  # or: major, patch, 0.2.0, etc.
```

Don't forget to also update the version in your package's `__init__.py`:

```python
# src/hypermodern_python/__init__.py
__version__ = "0.2.0"  # or: 0.1.1, 1.0.0
```

You should also document your release. For example, add a `CHANGELOG.md` file to
your repository, using the format specified at [Keep a
Changelog](https://keepachangelog.com/). Another option is to use [GitHub
Releases](https://help.github.com/en/github/administering-a-repository/creating-releases).

Add a badge to `README.md` which links to your PyPI project page and displays
the latest release:

```markdown
[![PyPI](https://img.shields.io/pypi/v/hypermodern-python.svg)](https://pypi.org/project/hypermodern-python/)
```

The badge looks like this: 
[![PyPI](https://img.shields.io/pypi/v/hypermodern-python.svg)](https://pypi.org/project/hypermodern-python/)

## Hosting documentation at Read the Docs

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
    - method: pip
      path: .
```

The `install` section in the configuration file tells Read the Docs how to
install your package and its documentation dependencies. Read the Docs does not
support installation of dependencies using Poetry directly. Luckily, pip can
install any package with a standard `pyproject.toml` file, and will use Poetry
behind the scenes.

While this means that you could install the documentation dependencies using
`pip install .[docs]`, this installation method does not honor the exact pinned
versions from `poetry.lock`, only the more generic version constraints in
`pyproject.toml`. This is why you should specify the dependencies using pip's
`requirements.txt` format. Export your pinned requirements to
`docs/requirements.txt` using the following command:

```sh
poetry export -f requirements.txt -E docs > docs/requirements.txt
```

This file needs to be placed under source control to ensure that it is available
when Read the Docs installs your dependencies. Update the file whenever you
upgrade or otherwise change your documentation dependencies.

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

Thank you for reading this far. This chapter concludes the Hypermodern Python
series.

This series of articles is dedicated to my father who introduced me to
programming in the 1980s, and who is an avid chess book collector. I saw a copy
of *Die hypermoderne Schachpartie* (The hypermodern chess game) in his
bookshelf, written by [Savielly
Tartakower](https://en.wikipedia.org/wiki/Savielly_Tartakower) in 1925 to
modernize chess theory. That inspired the title of this guide.
