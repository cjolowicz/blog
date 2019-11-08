--- 
date: 2019-11-07T12:52:59+02:00
title: "The hypermodern Python project, pt. 3"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - GitHub Actions
  - Codecov
  - PyPI
---

# Chapter 3: Continuous Integration

In this third installment of the Hypermodern Python series, I'm going to discuss
how to add continuous integration to your project.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-project-01-setup)
- [Chapter 2: Testing](../hypermodern-python-project-02-testing)
- [Chapter 3: Continuous Integration](../hypermodern-python-project-03-continuous-integration)
- [Chapter 4: Documentation](../hypermodern-python-project-04-documentation)

<!--
This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Continuous integration using GitHub Actions](#continuous-integration-using-github-actions)
- [Coverage reporting with Codecov](#coverage-reporting-with-codecov)
- [Uploading your package to PyPI](#uploading-your-package-to-pypi)
- [A typical release process](#a-typical-release-process)

<!-- markdown-toc end -->

## Continuous integration using GitHub Actions

Continuous integration (CI) refers to the practice of automating the integration
of code changes into a software project. The CI process can trigger tools such
as unit tests, linters, or typecheckers to verify the correctness of the
changes.

You have a plethora of options when it comes to continuous integration.
Traditionally, many open-source projects have employed [Travis
CI](https://travis-ci.com). Another popular choice are Microsoft's [Azure
Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/). In
this guide, you are going to use GitHub's brand new offering, [GitHub
Actions](https://github.com/features/actions).

Configure GitHub Actions by adding the following [YAML](https://yaml.org) file
to the `.github/workflows` directory:

```yaml
# .github/workflows/main.yml
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
```

This file defines a so-called *workflow*, which is triggered on every push to
your GitHub repository, and runs on the latest supported Ubuntu image. The
workflow consists of four steps:

1. Use [actions/checkout](https://github.com/actions/checkout) to fetch and check out your code.
2. Use [excitedleigh/setup-nox](https://github.com/excitedleigh/setup-nox) to activate Python and install Nox.
3. Use [dschep/install-poetry-action](https://github.com/dschep/install-poetry-action) to install Poetry.
4. Invoke `nox` to run your test suite.

You can add a GitHub Actions badge to your repository page:

> [![tests](https://github.com/cjolowicz/hypermodern-python/workflows/tests/badge.svg)](https://github.com/cjolowicz/hypermodern-python/actions?workflow=tests)

Add the line below to the top of your `README.md` to get the badge:

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
...

@nox.session(python="3.8")
def coverage(session):
    """Upload coverage data."""
    session.install("coverage", "codecov")
    session.run("coverage", "xml")
    session.run("codecov", *session.posargs)
```

Next, grant GitHub Actions access to upload to Codecov. On Codecov, copy the
*Repository Upload Token* from the settings page of your repository. On GitHub,
go to the settings page of your repository, and add a secret named
`CODECOV_TOKEN` with the token you copied.

Invoke the session from the GitHub Actions workflow, providing the
`CODECOV_TOKEN` secret as an environment variable:

```yaml
# .github/workflows/main.yml
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

Sign up at PyPI, and generate an API token on the Account Settings page. This
token permits GitHub Actions to upload packages to your PyPI account. Go to your
repository settings on GitHub, and add the token as a secret named `PYPI_TOKEN`.

Add the following lines to the bottom of your GitHub workflow:

```yaml
# .github/workflows/main.yml
...
    - run: poetry build
    - if: github.event_name == 'push' && startsWith(github.event.ref, 'refs/tags')
      run: |
        poetry publish --username=__token__ --password=${{ secrets.PYPI_TOKEN }}
```

You can now trigger package uploads by pushing a release tag to GitHub. In
addition to this, the package is built on every push to GitHub, for testing
purposes.

Add a badge to your `README.md` which links to your PyPI project page and always
displays the latest release:

```markdown
[![PyPI](https://img.shields.io/pypi/v/hypermodern-python.svg)](https://pypi.org/project/hypermodern-python/)
```

The badge looks like this: 
[![PyPI](https://img.shields.io/pypi/v/hypermodern-python.svg)](https://pypi.org/project/hypermodern-python/)

## A typical release process

Let's walk through the entire process of creating a release.

1. **Bump the version**

    Skip this step if you are releasing the initial version of your package.
    Otherwise, use Poetry to update the version declared in `pyproject.toml`:

    ```sh
    poetry version <version>  # for example: "major", "minor", "patch", "0.2.1"
    ```

    Don't forget to also update the version in your package's `__init__.py`.

2. **Update the changelog**

    The second step is to document the release. For example, add a
    `CHANGELOG.md` file to your repository, using the format specified at [Keep
    a Changelog](https://keepachangelog.com/). Another option is to use [GitHub
    Releases](https://help.github.com/en/github/administering-a-repository/creating-releases).

3. **Push a release tag**

    The third step is to create and push an annotated Git tag. By convention,
    these tags have the form `v<version>`.

    ```sh
    git tag --message="hypermodern-python 1.0.0" v1.0.0
    git push --follow-tags
    ```

<center>[Continue to the next chapter](../hypermodern-python-project-04-documentation)</center>