--- 
date: 2019-11-07T12:52:59+02:00
title: "Hypermodern Python 3: Linting"
description: "Coding in Python like Savielly Tartakower."
draft: true
tags:
  - python
  - nox
  - black
  - flake8
---

In this third installment of the Hypermodern Python series, I'm going to discuss
how to add linting to your project.

For your reference, below is a list of the articles in this series.

- [Chapter 1: Setup](../hypermodern-python-01-setup)
- [Chapter 2: Testing](../hypermodern-python-02-testing)
- [Chapter 3: Linting](../hypermodern-python-03-linting)
- [Chapter 3: Continuous Integration](../hypermodern-python-03-continuous-integration)
- [Chapter 4: Documentation](../hypermodern-python-04-documentation)
- [Chapter 5: Typing](../hypermodern-python-05-typing)

<!--
This guide has a companion repository:
[cjolowicz/hypermodern-python](https://github.com/cjolowicz/hypermodern-python)
-->

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**In this chapter:**

- [Linting with flake8](#linting-with-flake8)
- [Code formatting with Black](#code-formatting-with-black)
- [Checking the order of import statements with flake8-import-order](#checking-the-order-of-import-statements-with-flake8-import-order)
- [Finding bugs and design problems with flake8-bugbear](#finding-bugs-and-design-problems-with-flake8-bugbear)
- [Finding security issues with bandit](#finding-security-issues-with-bandit)

<!-- markdown-toc end -->

## Linting with flake8

Linters analyze source code to flag programming errors, bugs, stylistic errors,
and suspicious constructs. The most common ones for Python are
[pylint](https://www.pylint.org) and [flake8](http://flake8.pycqa.org). In this
chapter, you will use Flake8.

Add a Nox session to run Flake8 on your codebase:

```python
# noxfile.py
import nox


locations = "src", "tests", "noxfile.py"


@nox.session(python=["3.8", "3.7"])
def lint(session):
    """Lint using flake8."""
    session.install("flake8")
    session.run("flake8", *locations)

...
```

Flake8 assigns each of its messages an error code, prefixed by one or more
letters. These prefixes group the errors into so-called violation classes:

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

By default, Nox runs all sessions defined in `noxfile.py`, but you can restrict
it to the lint session using the `--session (-s)` option:

```sh
nox -rs lint
```

There are many [awesome Flake8
extensions](https://github.com/DmytroLitvinov/awesome-flake8-extensions). Some
of these will be presented in later sections.

## Code formatting with Black

The next addition to our toolbox is [Black](https://github.com/psf/black), the
uncompromising Python code formatter. One of its greatest features is its lack
of configurability. Blackened code looks the same regardless of the project
you're reading.

Adding Black is straightforward:

```python
# noxfile.py
...

@nox.session(python="3.8")
def black(session):
    """Run black code formatter."""
    session.install("black")
    session.run("black", *locations)
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

Invoking `nox` without arguments now also triggers the code formatter. It would
be better to simply check the code style without performing any actual
formatting. You can exclude Black from the sessions run by default, by setting
`nox.options.sessions`:

```python
# noxfile.py
import nox


nox.options.sessions = "lint", "tests"
...
```

Instead, check adherence to the Black code style inside the linter session. The
[flake8-black](https://pypi.org/project/flake8-black/) plugin generates warnings
if it detects that Black would reformat a source file:

```python
# noxfile.py
...
def lint(session):
    ...
    session.install("flake8", "flake8-black")
    ...
```

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
...
```

## Checking the order of import statements with flake8-import-order

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

## Finding bugs and design problems with flake8-bugbear

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

<center>[Continue to the next chapter](../hypermodern-python-03-continuous-integration)</center>
