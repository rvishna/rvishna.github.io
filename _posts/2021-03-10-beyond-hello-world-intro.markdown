---
layout: post
title: "Beyond Hello World — 1: A Python-Based Command-Line Greeter"
date: 2021-03-10 23:09:00 -0600
categories: python
---
This is the first of a series of articles in which I will provide a step-by-step for the elements involved in building and deploying a command-line application in Python. We will keep the scope of the application very small in order to cast sufficient attention to the complexities of building a production-ready application for the average end-user. Specifically, we will build an application that accepts the name of the user as input and greets the user.

This series is intended for Python beginners, but may also be useful to intermediate/advanced developers with limited experience in deploying production-ready applications. Some of the things I will cover in this series of articles include:

- Working with virtual environments
- Working with dependencies
- Exception handling
- Parsing command-line arguments
- Logging
- Automated linting and testing
- Deploying a standalone executable
- Branching strategies with Git
- CI/CD
- Semantic versioning
- Publishing packages to a registry

Before we begin, all of the code referenced in this article is available at <https://github.com/rvishna/pygreeter>.

## Initial Setup

I will assume that you are working on a `bash` or `bash`-like shell. On Windows, you can get a `bash` shell using [Git Bash](https://git-scm.com/download/win) or [Windows Subsystem for Linux (WSL)](https://docs.microsoft.com/en-us/windows/wsl/install-win10). I also assume that you have Python installed and the output of `python --version` is `3.7` or newer.

### Virtual Environments

One of the first steps when developing a Python application or library is to setup a virtual environment. A virtual environment is simply an isolated environment that keeps the dependencies of your projects separate. For example, if one of your projects has a dependency on `django 2.2.19`, and another project has a dependency on `django 3.1.7`, the easiest way to keep these projects separate and install both of these dependencies is to create two virtual environments, one for each project. Even if you are _currently_ only working on a single project and don't think this applies to you, it is still a good idea to create a virtual environment before starting your project.

Most Python packages use semantic versioning, which I will cover in greater detail in a future post. In semantic versioning, a version is represented as `MAJOR.MINOR.PATCH`, e.g., `1.2.14`. Hence, version `2.1.0` is newer than `2.0.6`, which is newer than `1.9.12`. There is no strict upper limit to the three components that comprise a version. A `MAJOR` version of 0 typically indicates that the package is in a development stage and is not production ready. When you specify a Python package as a dependency for your project, you may specify either an exact version (called pinning), a minimum version, or a range of versions that are compatible with the way you use that package.

There are several tools in the Python ecosystem that allow you to create virtual environments and manage your dependencies. We will use [poetry](https://python-poetry.org/) for this application.

### Directory Structure

Once you have `poetry` [installed](https://python-poetry.org/docs/), run the following:
```
$ poetry new pygreeter
```

If you see the following error,
```
bash: poetry: command not found
```
it is likely that `poetry` hasn't been added to your `PATH`. By default, installing `poetry` modifies your `PATH` by updating `$HOME/.profile`, but you may need to open a new shell to have the change take effect.

If you see a message that Python 2.7 will no longer be supported, you may need to set your default Python version to 3, as described [here](https://linuxconfig.org/how-to-change-from-default-to-alternative-python-version-on-debian-linux).

If you see the following error:
```
ModuleNotFoundError: No module named 'distutils.util'
```
you may need to install the `python3-distutils` package:
```
sudo apt-get install python3-distutils
```

Poetry creates the following directory structure:
```
$ tree pygreeter
pygreeter
├── README.rst
├── pygreeter
│   └── __init__.py
├── pyproject.toml
└── tests
    ├── __init__.py
    └── test_pygreeter.py

2 directories, 5 files
```

Notice that there is a parent directory `pygreeter`, which contains another directory called `pygreeter`. In Python, a directory that contains a file named `__init__.py` is considered a package. The top-level package directory may contain several levels of nested child directories that contain `__init__.py` files. These are referred to as modules and they can be referenced and imported as, e.g., `import pygreeter.logging` or `from pygreeter.logging import get_logger`. In this example, the directory named `pygreeter` containing the `__init__.py` is treated as a package and can be imported from any environment which adds the parent directory named `pygreeter` to the Python path. There is no reason for these two directories to have the same name, but this is a common practice in the Python community.

One reason to choose such a directory structure is to ensure that configuration files, documentation and tests are kept separate from the Python source files which contain the program logic. When using a version control system like `git`, the root of the repository is usually the parent directory `pygreeter`, which includes the configuration files and tests required to build and publish the package. In a [Monorepo](https://en.wikipedia.org/wiki/Monorepo) structure, the repository root may be at a level above this directory.

The `pyproject.toml` file contains some basic information about the project in the `tool.poetry` section, including the name, version, description and list of authors. You should update this section if you decide to publish your application to a registry.

Keep in mind that running `poetry new pygreeter` creates a virtual environment in a cache directory, which defaults to one of the following:

- macOS: `~/Library/Caches/pypoetry`
- Windows: `C:\Users\<username>\AppData\Local\pypoetry\Cache`
- Unix: `~/.cache/pypoetry`

You can get more information about this virtual environment by navigating to the `pygreeter` directory and running `poetry env info`. To clean up after yourself and remove this virtual environment, run
```
poetry delete pygreeter-w8vL6YN4-py3.7
```
replacing `pygreeter-w8vL6YN4-py3.7` with the name of your virtual environment.

### Dependencies

Sometimes, I overlook the versatility of the Python standard library, which includes modules for text processing, functional programming, reading/writing  file formats including CSV, JSON, XML, etc., and a basic mathematical module. There are several use cases for which you can simply use the Python standard library instead of installing a package. You do not need `numpy` to generate a random number. Small to medium size arrays can simply be represented as native Python lists, and you can use the `math` module, list comprehension and functional programming paradigms including `map`, `filter` and `reduce` in-place of most `numpy` functions.

> If it is possible to avoid a dependency without sacrificing complexity and performance, you should always consider doing so.

However, most medium to large Python codebases require multiple dependencies. There are two types of dependencies - development and production. Development dependencies are packages that you install locally in your development environment and may include tools like `black`, `pylint`, `pytest`, `mypy`, etc. These are examples of tools that improve your productivity as a developer through formatting, linting, type checking, testing, etc. They are not required for executing the main program logic. Production dependencies are packages that are required to execute the program logic of your application or library.

For this simple application, we do not require any production dependencies. We will install some development dependencies using `poetry`. First, navigate to the newly created `pygreeter` directory, open the `pyproject.toml` file, and under the `tool.poetry.dev-dependencies` section, add the following lines:

```toml
[tool.poetry.dev-dependencies]
black = "*"
isort = "*"
mypy = "*"
pylint = "*"
pytest = "*"
```

Now, run `poetry install`. This resolves your dependencies and installs them. Specifying the wildcard `*` for all of the development dependencies means that the latest versions of those packages will be used. This is not a problem for development dependencies, but when specifying production dependencies, it is generally a good idea to pin the version by using `==x.y.z` in place of `*`.

## Implementation

Before we implement the actual program logic, there are a few things to think about:

1. How does the user install the application?
2. Does the user run the application by double-clicking on an icon, or by typing in its name on the command-line?
3. Does the user provide input interactively? Using a command-line argument? From a file? A database?
4. Will the user be required to install Python?
5. What platforms would the application support? Windows/Linux/MacOS? Android/iOS?
6. How do users report bugs or request new features for the application?
7. How do users receive updates to the application?

Although the example I present here is quite simple, these questions recur at the beginning of every software product's lifecycle. There is often no one right answer to these questions. It is also easy to fall into the trap of answering questions like question 5 above with "Let's support every platform". Keep in mind that each of these answers have a big impact on the effort and complexity involved in developing and _maintaining_ the application. 

There are also questions that occur when starting a software project that I haven't included here. Remember that we have already stated what technology we are going to use (Python) and the scope and requirements of the software product (greeting the user). Often, it is these questions that take a significant amount of time to answer and have the most impact on the outcome of the project, including its successful adoption or impact.

### A First Attempt

We have already seen that the `pygreeter` directory containing the `__init__.py` file is considered a package, or a top-level module. Poetry adds the following line to this file when it creates it:

```python
__version__ = 0.1.0
```

Start the Python interpreter from the parent `pygreeter` directory and run the following:

```
>>> import pygreeter
>>> print(pygreeter.__version__)
```

This should print `0.1.0`, which is the current version of the `pygreeter` package.

If we simply added the following statement to `__init__.py`:

```python
__version__ = 0.1.0

print("Hello, User")
```

Now, importing the `pygreeter` package results in the following:

```
>>> import pygreeter
Hello, User
```

This is not exactly the behavior we want (setting aside the fact that we may not want the user to have to install the Python interpreter to run our application). It also doesn't accept the name of the user as input and simply prints a generic "Hello, User" to greet everyone. However, here is an important lesson in developing a complex software application (which this is not): 

> Often, an initial prototype with very limited functionality that takes very little time to build can be a valuable tool to demonstrate what the finished application might be capable of.

An initial prototype also serves as a pivot from which you could branch out your efforts to focus on something else. For instance, if the user requires the application to be deployed on an Android platform, now may be a good time to flesh out the rest of the deployment toolchain to see if an Android deployment is viable before spending any further time on development.

### Module as a Script and the Execution Pattern

A common idiom to have code that does not run when the module is imported but only when it is executed as a script is to check whether the name of the current scope is `__main__`. The Python interpreter sets the name of the scope to `__main__` whenever a module is executed using `python -m`. An alternative is to create a file named `__main__.py` alongside the `__init__.py` file in the `pygreeter` directory and place all top-level execution code in this file. We will use this approach and create the `__main__.py` file with the following contents:

```python
import sys
from typing import List


def main(argv: List[str]) -> int:
    print(f"Hello, {argv[0]}")
    return 0


if __name__ == "__main__":
    exit_code = main(sys.argv[1:])
    sys.exit(exit_code)
```

There are a few differences between this file and the first attempt:

1. The execution logic is delegated to a function named `main`, which accepts a list of strings as a positional argument and returns an integer indicating whether the function was successful. This function is called from the conditional block that checks whether the name of the current scope is `__main__`. This is commonly referred to as the execution pattern or [command pattern](https://en.wikipedia.org/wiki/Command_pattern). 
2. The application now accepts a command-line argument, which is presumably the name of the user and greets the user instead of printing a generic "Hello, User" message.
3. The argument and return value of the `main` function are annotated using type annotations. Python is a type-agnostic language and the Python interpreter doesn't require these annotations to execute the program. But it is a good practice to include type annotations wherever possible. It makes debugging easier, allows your IDE to come up with better suggestions for autocompletion, and while it is not a substitute for good documentation, it communicates the intended use of your functions.

> The execution pattern is quite useful when writing tests that can call the `main` function and pass an arbitrary list of strings instead of, say, monkey patching `sys.argv`. This is also the reason why the `__main__.py` file includes a conditional block that checks whether the name of the current scope is `__main__`. That way, the test module can do: `from pygreeter.__main__ import main`. More on this when we cover automated testing.

The application can now be run from the top-level `pygreeter` directory (which contains the `pyproject.toml` file) as:
```
$ python -m pygreeter Jane
Hello, Jane
```

However, there are a few issues with this code:

1. Running `python -m pygreeter` without arguments raises an `IndexError` exception.
2. Running `python -m pygreeter Jane Doe` prints `Hello, Jane` whereas the user's expectation may be to be greeted as `Hello, Jane Doe`.
3. Perhaps an unassuming user sees that the behavior of the application is to accept an arbitrary string as input and print the message `Hello, <string>`. She then runs `python -m pygreeter Jane&Co` with every hope that the application might greet her and her friends, who are being refered to colloquially as `Jane&Co`. What would she see as output? `Hello, Jane`? `Hello, Jane&Co`? What about running `python -m pygreeter $100`?
4. Mike, who owns a billboard company has decided to use `pygreeter` to create billboards that greet people with the name of their choice. He receives orders with the name to be greeted via email, runs `pygreeter`, and sends the output to a special printer that prints out a large billboard with the greeting. Business starts booming. Mike's job, however, has become quite monotonous - he simply copy-pastes the name sent by users via email and runs `pygreeter`. That is, until a North Korean hacker finds out about his business and decides to wreak havoc by sending Mike the name `Gotcha&rm -rf /`.

Depending on your background, the above list may evoke different reactions from you. Some of you may be thinking that these are completely overblown hypotheticals that should not be paid any further attention. Who in their right mind is going to run `pygreeter` without arguments when it is *obvious* that it takes in your name as the argument? And if someone wants to be greeted "Hello, Jane Doe", why don't they just use double quotes and run `python -m pygreeter "Jane Doe"`? In the next section, we will see how the first two issues can be addressed. The third and fourth issues can not be directly addressed as the `&` symbol is interpreted by `bash`-like shells as an instruction to run the command preceeding it in the background and the `$` symbol is used to evaluate environment variables. However, that doesn't mean there is nothing we can do about it.

## Building Blocks of a Robust Application

As a graduate student, I worked on a fairly large codebase using Fortran 90 and MPI to run massively parallel simulations of the flow from the exhaust of a jet engine to predict how loud it would sound to an observer on the ground. The simulations had to be submitted to a high-performance cluster and would take 2-3 days to simulate what corresponded to a real-time of a few seconds of flow. Back then, I was a total novice and my code hardly included any of the elements we are talking about, including logging, testing and exception handling. Therefore, it was an arduous experiment in trial and error, with each failed attempt causing more frustration and an impatient advisor always lurking in the shadows. 

My story is not unique - I know from anecdotal evidence that most graduate students write code this way. You could argue that my code wasn't intended for an end-user. Or that I wasn't going to test the limits of the code by giving it inputs outside the expected range. But at the end of the day, the time and effort required to implement a few basic principles including exception handling, logging and unit testing would have paid big dividends even for my graduate student codebase.

### Exception Handling

> An exception is a condition that is outside the set of expected operating conditions for a program. 

For instance, if `pygreeter` expects users to provide their name as a command-line argument, it is an exception when zero command-line arguments are provided. Since we did not "handle" this exception, it was "raised" by the Python interpreter and our users got to see an ugly traceback when they accidentally omitted the command-line argument. Once you get into the habit of raising and handling exceptions, it becomes much easier to write functions without worrying about all the inputs a user of the function could possibly conceive of. Raising exceptions is a good way to place bounds on what a function can and cannot handle.

Let us modify the `__main__.py` file to handle this exception:
```python
import sys
from typing import List


def main(argv: List[str]) -> int:
    try:
        print(f"Hello, {argv[0]}")
    except IndexError:
        print("Please provide your name to be greeted.")
        return 1
    return 0


if __name__ == "__main__":
    exit_code = main(sys.argv[1:])
    sys.exit(exit_code)
```

Notice that we not only catch the exception and print an error message, but also return a non-zero exit status to indicate that the program execution was unsuccessful.

For this simple application, there are no custom exceptions and no functions that raise an exception. In general, it is a good practice to define a base exception class for all custom exceptions that will be raised by your package and define custom exception classes that inherit from the base exception class. To demonstrate this, let's create a file named `exceptions.py` alongside `__main__.py` with the following contents:

```python
class PyGreeterError(Exception):
    """Base class for exceptions raised by pygreeter."""


class MissingNameError(PyGreeterError):
    """Raised when pygreeter is called without a name."""

    def __str__(self):
        return "A name is required for greeting."
```

Now, we can raise and catch our custom exception in `__main__.py`:

```python
import sys
from typing import List

from pygreeter.exceptions import MissingNameError, PyGreeterError


def main(argv: List[str]) -> int:
    try:
        print(f"Hello, {argv[0]}")
    except IndexError as exc:
        raise MissingNameError from exc
    return 0


if __name__ == "__main__":
    try:
        exit_code = main(sys.argv[1:])
    except PyGreeterError as exc:
        print(f"An error occurred when running pygreeter: {str(exc)}")
        exit_code = 1
    except Exception as exc:
        print(f"An unexpected error occurred when running pygreeter: {str(exc)}")
        exit_code = 2
    finally:
        sys.exit(exit_code)
```

Deriving our custom exceptions from a common base exception class `PyGreeterError` and catching it this way ensures that all custom exceptions raised by `pygreeter` are handled gracefully. The application also catches any other exceptions that occur and reports a slightly different error message. This is good practice for applications, but if you are developing a Python library, you should not catch the generic `Exception` class. Catching `PyGreeterError` _before_ the generic `Exception` class ensures custom exceptions raised by `pygreeter` are handled first. This is handy if you choose to include additional information in your exception class, such as the filename and line number where the exception occured, or take additional actions for these exceptions, such as sending an email to someone with the traceback of the exception.

There's still the issue of passing multiple names to the application. There are a couple of ways we could address this issue. One is to simply reject all but the first argument, which is something this code already does. The other is to allow the user to enter more than one name and use the input verbatim. This is also straightforward to implement. But how do we communicate the correct usage of the application in either of these instances to the user? Many unix commands including `sed`, `grep`, etc. print their correct usage when you pass the argument `-h` or `--help` on the command-line. Perhaps we could implement this feature for `pygreeter`? Assuming we can, how do we handle the following invocations of `pygreeter`:

```
$ python -m pygreeter Mary -h
$ python -m pygreeter --help Jack
```

For an application with fairly limited scope, we have already introduced significant complexity. Luckily, parsing command-line arguments is such a frequent requirement for Python-based command-line tools that the standard library includes a module to help you do just that. This is the topic of the next article.