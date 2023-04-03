---
nav_order: 4
---

# Python and Bazel

## This tutorial covers

- Background of Bazel and Bazel's Python support
- Covering the use of rules_python with Bazel and Python
- Creating a basic Python project for the tutorial
- Implementing a `MODULES.bazel` and `BUILD.bazel` files 
- Using Gazelle to generate more and `BUILD.bazel` updates
- Utilizing different Bazel commands
- An overview of Gazelle and Python dependency management
- Understanding the contents of the `WORKSPACE` and `BUILD.bazel` files
- Creating new internal dependencies and using Gazelle to update build files
- Adding a new external Python dependency
- Running and implementing Python unit tests with Bazel
- Learning about other rules in rules_python

## About Bazel and Go

This tutorial is going to cover how Bazel supports the programming language Python.
[Bazel](https://bazel.build) is an open-source build and test application that supports 
the software development lifecycle. Bazel strives to allow a developer to have hermetic and 
deterministic builds, testing, packaging, and deployment.  This build tool works with 
multiple languages and cross compilation for different operating systems and hardware architecture.

[Python](https://python.org) is an open-source programming language maintained by the Python Software Foundation.
Now that we have some background about Bazel
now let's cover some Bazel concepts.

One of the concepts within Bazel is a rule.  A Bazel rule defines a series of actions and outputs. Toolchains are another
concept/part of Bazel. The toolchain framework is a way for rule authors to decouple their rule logic from 
the platform-based selection of tools. So we need a rule, or usually a set of rules
(a "ruleset") that provides a toolchain to support a programming language.  
The Bazel open-source community maintains
[rules_python](https://github.com/bazelbuild/rules_python).  This ruleset provides the following support:

- Building python libraries, binaries, and tests
- Python dependency management
- Support for "third_party" packages using pip
- Support for Wheel dists
- Support for different versions of Python in same project
- Download Python in order to keep the build hermetic 
- Remote execution
- Code coverage testing

The Bazel open-source community also provides another tool called [Gazelle](https://github.com/bazelbuild/bazel-gazelle).
Gazelle addresses the creation and maintenance of [BUILD](https://bazel.build/concepts/build-files) files.  
Every Bazel project has BUILD (`BUILD.bazel`) files that define the various rules that are used within a project.
When you add more code or dependencies to a project you need to update your build files.  When you add a new folder
you need to add another `BUILD.bazel` file. If you have ever worked with Bazel you know how much time you spend maintaining
these files, if you maintain the files by hand.  Gazelle was created to reduce the previously mentioned pain points.

> Gazelle is a build file generator for Bazel projects. It can create new `BUILD.bazel` files for a project that follows language conventions, and it can update existing build files to include new sources, dependencies, and options. Gazelle natively supports Go and protobuf, and it may be extended to support new languages and custom rule sets.
>
>  -- <cite>https://github.com/bazelbuild/bazel-gazelle#gazelle-build-file-generator</cite> 

Initially Gazelle was created to support Go, and now supports many other languages. These 
languages include but are not limited to Haskell, Java, JavaScript/TypeScript, Python, R, Starlark, and Go.

Part of learning Bazel is understanding the configuration language that Bazel uses.
The language is called [Starlark](https://github.com/bazelbuild/starlark).

> Starlark (formerly known as Skylark) is a language intended for use as a configuration language. It was designed for the Bazel build system, but may be useful for other projects as well. This repository is where Starlark features are proposed, discussed, and specified. It contains information about the language, including the specification. There are multiple implementations of Starlark.
>
> Starlark is a dialect of Python. Like Python, it is a dynamically typed language with high-level data types, first-class functions with lexical scope, and garbage collection.
> - <cite>https://github.com/bazelbuild/starlark#overview</cite>

Before we start creating a simple Python project, we need to install bazel if you do not have it already
installed.

## Dependencies for the tutorial

We use the bazelisk for this tutorial:

- bazelisk: <https://github.com/bazelbuild/bazelisk#installation>

Now, how about we write some code? We will create a
simple Python program and then add Bazel to the project.  We have
structured the tutorial in this manner since at times, you migrate
to using Bazel with an existing project, and at other times you
start a new project with Bazel.

## The project

We are going to create a small example project first using Python.  

The project is going to consist of a Python program that starts a Flask
servers which serves a random number.

## Generate the project framework

First, create a git repository to store your work.  For this project, we are using
<https://github.com/bazelbuild/rules_python/tree/main/examples/build_file_generation>, and replace any references
to that repository with your own. You can refer to the above repository for 
the final source code base.


Run the following comment to create the Python project and files:

```bash
$ mkdir  random_number_generator
$ touch BUILD.bazel MODULE.bazel WORKSPACE __init__.py __main__.py \
gazelle_python.yaml random_number_generator/__init__.py \
random_number_generator/generate_random_number.py \
requirements.in requirements_lock.txt requirements_windows.txt
```

In your favorite editor edit the `__init__.py` file and add:

```python
from flask import Flask, jsonify
from random_number_generator import generate_random_number

app = Flask(__name__)

@app.route('/random-number', methods=['GET'])
def get_random_number():
    return jsonify({'number': generate_random_number.generate_random_number()})

"""Start the python web server"""
def main():
    app.run()
```

Edit the `__main__.py` file and add:

```python

from __init__ import main

if __name__ == '__main__':
    main()
```

In the `requirements.in` file add the following lines:

```
flask
importlib-metadata
```

Next add the following Python code to the `random_number_generator/generate_random_number.py`
file.

```
import random

"""Generate a random number"""
def generate_random_number():
    return random.randint(1, 10)
```

We should now have a file layout like the following:

```bash
$ tree
.
├── BUILD.bazel
├── MODULE.bazel
├── WORKSPACE
├── __init__.py
├── __main__.py
├── gazelle_python.yaml
├── random_number_generator
│   ├── __init__.py
│   └── generate_random_number.py
├── requirements.in
├── requirements_lock.txt
└── requirements_windows.txt

1 directory, 11 files
```

Next add a .gitignore file by running the following command.

```bash
$ tee -a .gitignore << EOF
/bazel-$(basename $(pwd))
/bazel-bin
/bazel-out
/bazel-testlogs
EOF
```

Bazel creates various directories in the project root and this file will allow git 
to ignore those directories.

This is a good time to push your files into a remote git repository like GitHub. Now
we cover `rules_python` and Gazelle.

If we where not using Bazel we would next setup the Python envinronment using a tool like `virtualenv` and
install the dependies using `pip` or `easy_install`.  But when using Bazel all of that tooling and
project configuration is handled for us. In order to understand more about about how Bazel manages dependencies
and provides Python version configuration lets cover Bazel Rules.

<!-- TODO bazelrc file -->

## Bazel Rules

As we mentioned previously, Bazel provides rules_python and Gazelle. You can find more
about them here:

- <https://github.com/bazelbuild/rules_python>
- <https://github.com/bazelbuild/bazel-gazelle>

At a high level, we use Starlark to define that Bazel will use rules from rules_python
to create the Go support within a project. We use Gazelle to manage our `BUILD.bazel` files,
and `MODULES.bazel` file, and other Bazel-specific files.

If you are not familiar with `BUILD.bazel` files or `MODULES.bazel` files look at:
https://bazel.build/concepts/build-files.

We are providing instuctions on how to setup a Pyton project using bzlmod. Older projects
may use a WORKSPACE file for dependency management.

Next, let's create our `MODULES.bazel` file so that Bazel knows it is using rules_python and Gazelle.

## Create a `MODULES.bazel` file

Bazel files, including the `MODULES.bazel` and other `BUILD.bazel` files, include [Starlark](https://bazel.build/rules/language) definitions.

Use your favorite editor and edit `MODULES.bazel` in the root directory of your project. 

```python
# Declares certain properties of the Bazel module represented by the current Bazel repo.
# These properties are either essential metadata of the module (such as the name and version),
# or affect behavior of the current module and its dependents.
module(
    # update the name to match your project folder
    name = "build_file_generation",
    version = "0.0.0",
    compatibility_level = 1,
)

# The following stanza defines the dependency rules_python and gazlle.
# Update the version number with the latest release that are found here
# https://github.com/bazelbuild/rules_python/releases
bazel_dep(name = "rules_python", version = "0.0.0")
bazel_dep(name = "rules_python_gazelle_plugin", version = "0.20.0")

# The following stanza defines the dependency rules_python.
# Update the version based on the version found here: 
# https://github.com/bazelbuild/bazel-gazelle/releases
bazel_dep(name = "gazelle", version = "0.30.0", repo_name = "bazel_gazelle")

# The following stanze Returns a proxy object representing a module extension;
# its methods can be invoked to create module extension tags.
python = use_extension("@rules_python//python:extensions.bzl", "python")

# Using the module extension we register a hermetic Python interpreter rather than relying on
# a system-installed interpreter.
# This toolchain will allow bazel to download a specific  python version, and use that version
# for compilation.
python.toolchain(
    name = "python3_10",
    python_version = "3.10",
)

# Import the python toolchain generated by the given module extension into the scope of the current module.
use_repo(python, "python3_10_toolchains")

# Register an already-defined toolchain so that Bazel can use it during toolchain resolution.
register_toolchains(
    "@python3_10_toolchains//:all",
)

# Use the pip extension
pip = use_extension("@rules_python//python:extensions.bzl", "pip")

# Use the extension to call the `pip_repository` rule that invokes `pip`, with `incremental` set.
# Accepts a locked/compiled requirements file and installs the dependencies listed within.
# Those dependencies become available in a generated `requirements.bzl` file.
# You can instead check this `requirements.bzl` file into your repo.
# Because this project has different requirements for windows vs other
# operating systems, we have requirements for each.
pip.parse(
    name = "pip",
    # Generate user friendly alias labels for each dependency that we have.
    incompatible_generate_aliases = True,
    requirements_lock = "//:requirements_lock.txt",
    requirements_windows = "//:requirements_windows.txt",
)

# Imports the pip toolchain generated by the given module extension into the scope of the current module.
use_repo(pip, "pip")
```

The above `MODULES.bazel` file contains specific version numbers for rules_python and Gazelle.  Refer to the 
python_rules site to use the latest versions.  Also, update the `python_register_toolchains(version = "3.10")`
to the version you would like to use of Python.

Next, we need a `BUILD.bazel` file in the root project directory.

## Create the initial `BUILD.bazel` file

Open your editor and create a file named `BUILD.bazel`. Write the following contents to the `BUILD.bazel`
file:

```python
# Load various rules so that we can have bazel download
# various rulesets and dependencies.
# The `load` statement imports the symbol for the rule, in the defined
# ruleset. When the symbol is loaded you can use the rule.
load("@bazel_gazelle//:def.bzl", "gazelle")
load("@pip//:requirements.bzl", "all_whl_requirements")
load("@rules_python//python:defs.bzl", "py_binary", "py_library", "py_test")
load("@rules_python//python:pip.bzl", "compile_pip_requirements")
load("@rules_python_gazelle_plugin//:def.bzl", "GAZELLE_PYTHON_RUNTIME_DEPS")
load("@rules_python_gazelle_plugin//manifest:defs.bzl", "gazelle_python_manifest")
load("@rules_python_gazelle_plugin//modules_mapping:def.bzl", "modules_mapping")

# This stanza calls a rule that generates targets for managing pip dependencies
# with pip-compile.
compile_pip_requirements(
    name = "requirements",
    extra_args = ["--allow-unsafe"],
    requirements_in = "requirements.in",
    requirements_txt = "requirements_lock.txt",
    requirements_windows = "requirements_windows.txt",
)

# This repository rule fetches the metadata for python packages we
# depend on. That data is required for the gazelle_python_manifest
# rule to update our manifest file.
modules_mapping(
    name = "modules_map",
    exclude_patterns = [
        "^_|(\\._)+",  # This is the default.
        "(\\.tests)+",  # Add a custom one to get rid of the psutil tests.
    ],
    wheels = all_whl_requirements,
)

# Gazelle python extension needs a manifest file mapping from
# an import to the installed package that provides it.
# This macro produces two targets:
# - //:gazelle_python_manifest.update can be used with `bazel run`
#   to recalculate the manifest
# - //:gazelle_python_manifest.test is a test target ensuring that
#   the manifest doesn't need to be updated
gazelle_python_manifest(
    name = "gazelle_python_manifest",
    modules_mapping = ":modules_map",
    pip_repository_name = "pip",
    requirements = "//:requirements_lock.txt",
    # NOTE: we can use this flag in order to make our setup compatible with
    # bzlmod.
    use_pip_repository_aliases = True,
)

# Our gazelle target points to the python gazelle binary.
# This is the simple case where we only need one language supported.
# If you also had proto, go, or other gazelle-supported languages,
# you would also need a gazelle_binary rule.
# See https://github.com/bazelbuild/bazel-gazelle/blob/master/extend.rst#example
gazelle(
    name = "gazelle",
    data = GAZELLE_PYTHON_RUNTIME_DEPS,
    gazelle = "@rules_python_gazelle_plugin//python:gazelle_binary",
)
```

We now have done the initial creation of the `WORKSPACE` and `BUILD.bazel` files. 
These files contain the basic framework that allows Bazel and Gazelle to intially run.
Next, we will use Bazel to run the Gazelle target.

## Run the Gazelle commands

Now run the following command to update the python pip requirements and 
use gazelle to modify the `BUILD.bazel` files.

```bash
$ bazelisk run //:requirements.update
$ bazelisk run //:gazelle_python_manifest.update
$ bazelisk run //:gazelle update
```

We previously mentioned we use Bazel to run Gazelle, and 
Gazelle manages the `BUILD.bazel` files for us.  We are using bazelisk to 
manage and run Bazel, but we will typically say "run bazel" 
instead of "run bazelisk".  

Let's break down what each of the above commands did.

First running `bazelisk run //:requirements.update` uses pip to update the requirments.in file, and configures pip.
Next running `bazelisk run //:gazelle_python_manifest.update` updates the `gazelle_python.yaml`
to include information for gazelle about the pip dependencies.

Lastly running `bazelisk run //:gazelle update` causes gazelle to run and update the BUILD.bazel file in the root directory
and create a BUILD.bazel file in the random_number_generator folder.  We will cover more
about the BUILD.bazel files in later sections.

### Basic Bazel commands

Bazel has various [commands](https://bazel.build/run/build#available-commands) that 
are defined.

The main ones that developers typically run are [build](https://bazel.build/run/build#bazel-build),
[test](https://bazel.build/docs/user-manual#running-tests) and [run](https://bazel.build/docs/user-manual#running-executables).

The build and test commands are pretty self-explanatory.  The build command builds the source code
for your project, and the test command runs any tests that are defined. The run command
executes a rule, for instance, executes a go binary.

In the project, you can run

```bash
$ bazelisk build //...
```

This will build the Python code for our example project. 


We can run the binary that Bazel creates with the following command:

```bash
$ bazelisk run $(bazelsk query "filter('_bin$', //...:)")
```

We are using the query argument to find the target that ends with `_bin`.  Gazelle added that target
to your build file.

So the commands build, run, and test are pretty easy to get your head around, but the third part of the
command was a bit confusing for me when I first learned Bazel.  The "//..." or "//:something"  
is called a target.

You can refer to the documentation [here](https://bazel.build/run/build#bazel-build).  The text "//..."
and "//:test_python_example_bin" are all the targets in a given directory or the name of a 
specific target.  Some commands like build, and test can run multiple targets, 
while a command like run can only execute one target.

The below table provides a great guide for targets:

<table>
<tbody><tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>bar:wiz</code></td>
  <td>Just the single target <code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>bar:wiz</code>.</td>
</tr>
<tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>bar</code></td>
  <td>Equivalent to <code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>bar:bar</code>.</td>
</tr>
<tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>bar:all</code></td>
  <td>All rule targets in the package <code translate="no" dir="ltr">foo/<wbr>bar</code>.</td>
</tr>
<tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>.<wbr>.<wbr>.<wbr></code></td>
  <td>All rule targets in all packages beneath the directory <code translate="no" dir="ltr">foo</code>.</td>
</tr>
<tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>.<wbr>.<wbr>.<wbr>:all</code></td>
  <td>All rule targets in all packages beneath the directory <code translate="no" dir="ltr">foo</code>.</td>
</tr>
<tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>.<wbr>.<wbr>.<wbr>:&#42;</code></td>
  <td>All targets (rules and files) in all packages beneath the directory <code translate="no" dir="ltr">foo</code>.</td>
</tr>
<tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>foo/<wbr>.<wbr>.<wbr>.<wbr>:all-targets</code></td>
  <td>All targets (rules and files) in all packages beneath the directory <code translate="no" dir="ltr">foo</code>.</td>
</tr>
<tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>.<wbr>.<wbr>.<wbr></code></td>
  <td>All targets in packages in the workspace. This does not include targets
  from <a href="/docs/external">external repositories</a>.</td>
</tr>
<tr>
  <td><code translate="no" dir="ltr">/<wbr>/<wbr>:all</code></td>
  <td>All targets in the top-level package, if there is a `BUILD.bazel` file at the
  root of the workspace.</td>
</tr>
</tbody></table>

> <cite>https://bazel.build/run/build#specifying-build-targets</cite>

If we look in the `BUILD.bazel` file in the root directory will find a python_library rule
named test_python_example, and this is a target we can build.

```bash
$ bazelisk build :test_python_example
```

This "python_library" target is named by Gazelle automatically depending on the name of your project, so
the name may differ.

Or we can build all of the targets under the random_number_generator directory:

```bash
$ bazelisk build //random_number_generator/...
```

<!-- FIXME -->

### Where Gazelle defines the dependencies

One of the features of Gazelle is to "vendor" Go projects.  Within this example, we are 
using Go vendoring at the base, but Bazel must also have the external dependencies defined.

The Gazelle update-repos command takes the go.mod file and creates the StarkLark code that
defines the external vendoring that Bazel uses. External dependencies are defined in one 
of two locations; in the `WORKSPACE` file or in an external file that is referenced in
the `WORKSPACE` file. The list of external dependencies can grow very long, so we recommend that
it is defined as a reference in the `WORKSPACE` file.

Each of the following lines within the `WORKSPACE` file defines the location of the `deps.bzl` file:

```python
# load Bazel and Gazelle rules
load("@io_bazel_rules_go//go:deps.bzl", "go_register_toolchains", "go_rules_dependencies")
load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")

############################################################
# Define your own dependencies here using go_repository.
# Else, dependencies declared by rules_go/gazelle will be used.
# The first declaration of an external repository "wins".
############################################################

load("//:deps.bzl", "go_dependencies")
```

One challenge you can run into is that you need to manually override a dependency, and  you can
do this by adding the code "http_archive". Below we have an example of overriding the "buildtools" dependency.

```python
http_archive(
    name = "com_github_bazelbuild_buildtools",
    sha256 = "a02ba93b96a8151b5d8d3466580f6c1f7e77212c4eb181cba53eb2cae7752a23",
    strip_prefix = "buildtools-3.5.0",
    urls = [
        "https://github.com/bazelbuild/buildtools/archive/3.5.0.tar.gz",
    ],
)
```

This example is from the cockroach database operator project. You can see
the full definition [here](https://github.com/cockroachdb/cockroach-operator/blob/0ef4d1e1b4c94a8edf1393b0fa72d9de8bc21477/WORKSPACE#L20).

Now let's cover what is inside of the `BUILD.bazel` files. As we mentioned, Bazel rules are in essence, 
Starlark libraries.

### The `BUILD.bazel` files

The rules_go has several "Core rules" defined.  These include:

- go_binary
- go_library
- go_test
- go_source
- go_path

See [here](https://github.com/bazelbuild/rules_go/blob/master/docs/go/core/rules.md) for more details.
And these Starlark rules are used inside of the `BUILD.bazel` files, and are often updated automatically by Gazelle.

After we ran Gazelle, the `BUILD.bazel` file was updated to include two new Starlark definitions:

```python
go_library(
    name = "basic-gazelle_lib",
    srcs = ["main.go"],
    importpath = "github.com/bazelbuild/rules_go/tree/master/examples/basic-gazelle",
    visibility = ["//visibility:private"],
    deps = ["//cmd"],
)

go_binary(
    name = "basic-gazelle",
    embed = [":basic-gazelle_lib"],
    visibility = ["//visibility:public"],
)
```

Both the go_library and go_binary rules are defined for Bazel. The go_library rule defines the build of a Go library from a set of source files that are all part of the same package. The go_binary rule defines the build of an executable from a set of source files, which must all be in the main package.  The go_rules project includes a great documentation [section](https://github.com/bazelbuild/rules_go/blob/master/docs/go/core/rules.md#introduction) if you want more details.

More `BUILD.bazel` files were also created. Here is the `BUILD.bazel` file that was created in 
the cmd folder.

```python
load("@io_bazel_rules_go//go:def.bzl", "go_library")

go_library(
    name = "cmd",
    srcs = [
        "roll.go",
        "root.go",
    ],
    importpath = "github.com/bazelbuild/rules_go/tree/master/examples/basic-gazelle/cmd",
    visibility = ["//visibility:public"],
    deps = [
        "@com_github_spf13_cobra//:cobra",
    ],
)
```

The first line load the Starlark definition from the go_rules library. You can then
use "go_library" which is used directly after.  This go_library definition also mentions
an external dependency using cobra.

### How these files work together

The `WORKSPACE`, `deps.bzl`, and `BUILD.bazel` files create a dependency graph that Bazel uses.
This blog [post](https://blog.bazel.build/2015/06/17/visualize-your-build.html) covers
visualizing the dependency graph.  Take a peak if you want to learn a bit about
"bazel query" command.

Next, we cover more definitions in the `WORKSPACE` file.  We can start with the following
code:

```
http_archive(
    name = "io_bazel_rules_go",
```

We are not including the full call for brevity. This http_archive definition tells
Bazel to download and use a specific version of rules_go. If you look at the `BUILD.bazel` file in the
root directory you can see the "load" command for rules_go, which exports go_library.

```python
load("@io_bazel_rules_go//go:def.bzl", "go_binary", "go_library")
```

The go_library definition is then used later in the file.

```
go_library(
    name = "basic-gazelle_lib",
```

So the `WORKSPACE` file includes the definition of which rules_go we are using and then the `BUILD.bazel`
files load those rules and use one of the definitions in the rules. 


The same kind of dependency graph is used for external dependencies. The `WORKSPACE` file includes the
definition for Gazelle (http_archive) and an import for the `deps.bzl` file. The `deps.bzl` file
includes load definitions for the Gazelle "go_repository" rule.   The go_repository rules define various
external go dependencies that are then vendored.  One of those dependencies is cobra, which is used
as a dependency by all of the go files inside of the cmd directory. Inside of the `BUILD.bazel` file, located in the cmd 
directory, the "deps" are a parameter passed in the go_library rule.

```python
    deps = ["@com_github_spf13_cobra//:go_default_library"],
```

So Bazel now has the capability to:

- Build a dependency graph for the project
- Various rules are defined that impact the dependency tree
- go_rules, and Gazelle define various rules
- The Bazel dependency tree includes go_library rules
- External dependencies are defined in go_repository rules
- deps are passed into go_library rules

All of these definitions create a dependency graph that allows Bazel to run:

```bash
$ bazelisk build //...
```

When the command is executed, Bazel will download and cache all dependencies, including but not limited to:

- The defined GoLang compiler and libraries
- The defined rules sets
- build the Go binary

Downloading and caching the above components is part of Bazel providing hermetic and
deterministic builds.  All of the downloaded components are checked against an SHA that 
verifies the checksum of the downloaded file(s) checksum.

Next, we will make some code changes and introduce an internal code
dependency.

## Using the files under pkg

Now, we want to modify and use the files under the pkg directory.

Edit roll.go under the cmd folder and add an import to roll_dice.

You will now have:

```go
import (
    "fmt"

    "github.com/bazelbuild/rules_go/tree/master/examples/basic-gazelle/pkg/roll"
    "github.com/spf13/cobra"
)
```

Then call `roll.Roll()` after the `fmt.Println` statement. This will give you:

```go
   Run: func(cmd *cobra.Command, args []string) {
       fmt.Println("roll called")
       fmt.Println(roll.Roll())
   },
```

You have edited the following files.

```
├── cmd
│   ├── roll.go
└── pkg
    └── roll
        └── roll_dice.go
```

We now need to update the `BUILD.bazel` files, and the easiest way to do this is to run Gazelle.

Execute the following command:

```bash
$ bazelisk run //:gazelle
```

We can now use bazel to run the binary again:

```bash
$ bazelisk run //:basic-gazelle roll
```

The above commands build the Go binary and executes it.  The following
is an example of the output from the run command.

```
INFO: Analyzed target //:basic-gazelle (1 packages loaded, 6 targets configured).
INFO: Found 1 target...
Target //:basic-gazelle up-to-date:
  bazel-bin/basic-gazelle\_/basic-gazelle
INFO: Elapsed time: 0.316s, Critical Path: 0.16s
INFO: 3 processes: 1 internal, 2 linux-sandbox.
INFO: Build completed successfully, 3 total actions
INFO: Build completed successfully, 3 total actions
roll called
roll dice
```

Running the Gazelle target modified the `BUILD.bazel` file under the cmd directory.  Here is the diff.

```diff
diff --git a/cmd/BUILD.bazel b/cmd/BUILD.bazel
index ac66183..9033b86 100644
--- a/cmd/BUILD.bazel
+++ b/cmd/BUILD.bazel
@@ -9,5 +9,8 @@ go_library(
     ],
     importpath = "github.com/bazelbuild/rules_go/tree/master/examples/basic-gazelle/cmd",
     visibility = ["//visibility:public"],
-    deps = ["@com_github_spf13_cobra//:cobra"],
+    deps = [
+        "//pkg/roll",
+        "@com_github_spf13_cobra//:cobra",
+    ],
 )
```

The line was added inside of the deps stanza that points to the package where roll.go resides.

Next update the `BUILD.bazel` files using gazelle:

```bash
$ bazelisk run //:gazelle
```

Now we have `BUILD.bazel` updated. Here is the diff:

```diff
diff --git a/cmd/BUILD.bazel b/cmd/BUILD.bazel
index ac66183..891b0e1 100644
--- a/cmd/BUILD.bazel
+++ b/cmd/BUILD.bazel
@@ -9,5 +9,9 @@ go_library(
     ],
     importpath = "github.com/bazelbuild/rules_go/tree/master/examples/basic-gazelle/cmd",
     visibility = ["//visibility:public"],
-    deps = ["@com_github_spf13_cobra//:cobra"],
+    deps = [
+        "//pkg/roll",
+        "@com_github_spf13_cobra//:cobra",
+    ],
 )
```


The project is now modified so that the files under the pkg folder are now used.  This is the 
principle of using internal dependencies.  Next, we will add a Go project dependency
hosted out of GitHub; an "external dependency".

## Adding an external dependency

We are going to add klog as an external dependency, which is located here: https://github.com/kubernetes/klog.

To initialize klog we add the `klog.InitFlags(nil) line to the main.go file:

```go
func main() {
    klog.InitFlags(nil)
    cmd.Execute()
}
```
The add the import:

```go
   "k8s.io/klog/v2"
```

Edit pkg/roll_dice.go file to add the call to klog, and add the required import statement.
Here is an example of using klog in the roll_dice.go file.

```go
    klog.Info("rolling the dice")
```

Also replace the fmt.Println statement in cmd/roll.go:

```go
        Run: func(cmd *cobra.Command, args []string) {
               klog.Info("calling roll")
               fmt.Printf("Number rolled: %s\n", roll.Roll())
        }
```

Once that code change is done, we need to run go mod to update the project's 
dependencies. We can use Bazel to run the Go binary instead of having
to installing the Go SDK ourselves.  The Go rules have already downloaded
the Go SDK, so use the following command.

```bash
$ bazelisk run @go_sdk//:bin/go -- mod tidy
```

Keeping go.mod updated allows us to either use Go directly or Bazel to build
and run the code.

We now need to update the Bazel import, and the easiest way to do this is to run Gazelle.

```bash
$ bazelisk run //:gazelle-update-repos
$ bazelisk run //:gazelle
```

The first Bazel command updated the `deps.bzl` file. The second command
updates the `BUILD.bazel` file in pkg/roll.  Below is the diff of the 
updates.

```diff
diff --git a/examples/basic-gazelle/pkg/roll/BUILD.bazel b/examples/basic-gazelle/pkg/roll/BUILD.bazel
index bd37d646..0ced314d 100644
--- a/examples/basic-gazelle/pkg/roll/BUILD.bazel
+++ b/examples/basic-gazelle/pkg/roll/BUILD.bazel
@@ -5,6 +5,7 @@ go_library(
     srcs = ["roll_dice.go"],
     importpath = "github.com/bazelbuild/rules_go/examples/basic-gazelle/pkg/roll",
     visibility = ["//visibility:public"],
+    deps = ["@io_k8s_klog_v2//:klog"],
 )
```

You can see the deps is now updated and points to the external repo `"@io_k8s_klog_v2//:klog"`
The "@" references an external code base that Bazel will download so that the Go SDK can build
the code.

This GitHub repo is defined in `deps.bzl` file in the following go_repository stanza.

```python
     go_repository(
         name = "io_k8s_klog_v2",
         importpath = "k8s.io/klog/v2",
         sum = "h1:atnLQ121W371wYYFawwYx1aEY2eUfs4l3J72wtgAwV4=",
         version = "v2.80.1",
     )
```

We can now run our Go binary and see the changes.

```bash
$  bazel run //:basic-gazelle roll
INFO: Analyzed target //:basic-gazelle (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //:basic-gazelle up-to-date:
  bazel-bin/basic-gazelle_/basic-gazelle
INFO: Elapsed time: 0.119s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Build completed successfully, 1 total action
I1129 14:45:14.253052   22596 roll.go:36] calling roll
I1129 14:45:14.253087   22596 roll_dice.go:26] rolling the dice
Number rolled: 43
```
One of the things that you may notice is that you do not have to run "bazel build" and then "bazel run".
Bazel will determine that the code is not built, and will run the "build" phase for you automatically.

To recap what we have done.  We have modified our code to use the babble Go code on 
GitHub.  We then use Bazel to run `go mod`, which updates go.mod file. Next we ran the targets gazelle-update-repos and gazelle with Bazel. The first Bazel alias updated the `deps.bzl` file with the external dependency and the Gazelle target 
updated the deps section in pkg/roll/BUILD.bazel.  Bazel can then download the external dependency
and use that dependency when our example Go program is compiled.

How about we add a Go unit test so we can run "bazel test"?

## Go tests

As we mentioned, Bazel supports running code tests, as defined in Bazel rules. One of the rules from go_rules
is go_test.  Now let's add a test.

Create a new file in the pkg/roll directory called roll_dice_test.go.
Include the following code:

```go
package roll

import (
        "testing"
)

func TestGenerateNumber(t *testing.T) {
        result := generateNumber()

        if result == "" {
                t.Error("got an empty string")
        }
}
```

We have a unit test now, but Bazel does not know about it.  Again we need 
Bazel to have the target in its dependency graph, and to do that, we need
to update the `BUILD.bazel` file.  The easiest way to do that is with Gazelle.

Simply run:

```bash
$ bazelisk run //:gazelle
```

This now updates the `BUILD.bazel` file in the pkg/roll directory with the following lines:

```python
go_test(
    name = "roll_test",
    srcs = ["roll_dice_test.go"],
    embed = [":roll"],
)
```

We now have a [go_test](https://github.com/bazelbuild/rules_go/blob/master/docs/go/core/rules.md#go_test) 
rule, which is part of the rules_go ruleset. Now we can run:

```bash
$ bazelisk test //...

The above command should print out results similar to

```bash
$ bazelisk test //...
INFO: Analyzed 6 targets (0 packages loaded, 0 targets configured).
INFO: Found 5 targets and 1 test target...
INFO: Elapsed time: 0.125s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
//pkg/roll:roll_test                                            (cached) PASSED in 0.0s

Executed 0 out of 1 test: 1 test passes.
INFO: Build completed successfully, 1 total action
```

You may also notice that the command printed out a target named `//pkg/roll:wroll`.
We can also run just the specific target:

```bash
$ bazelisk test //pkg/roll:roll
```

Let's now see what happens when a test fails since debugging unit tests are often part of the
development process. In the roll_dice_test.go file, change the "if" statement as shown below.

```go
    if result == "" {
```

Now if we run

```
$ bazelisk test //pkg/roll:roll_test
```

We get an output like

```bash
$ bazelisk test //pkg/roll:roll_test
INFO: Analyzed target //pkg/roll:roll_test (0 packages loaded, 0 targets configured).
INFO: Found 1 test target...
FAIL: //pkg/roll:roll_test (see /home/clove/.cache/bazel/_bazel_clove/f408d421e706f9a6112d2f3205e6556c/execroot/__main__/bazel-out/k8-fastbuild/testlogs/pkg/roll/roll_test/test.log)
Target //pkg/roll:roll_test up-to-date:
  bazel-bin/pkg/roll/roll_test_/roll_test
INFO: Elapsed time: 0.336s, Critical Path: 0.18s
INFO: 6 processes: 1 internal, 5 linux-sandbox.
INFO: Build completed, 1 test FAILED, 6 total actions
//pkg/roll:roll_test                                                     FAILED in 0.0s
  /home/clove/.cache/bazel/_bazel_clove/f408d421e706f9a6112d2f3205e6556c/execroot/__main__/bazel-out/k8-fastbuild/testlogs/pkg/roll/roll_test/test.log

INFO: Build completed, 1 test FAILED, 6 total action
```

The line that displays the path to the test.log file will differ between systems, but it provides output from the unit test.
If we cat the file we see the results:

```bash
$ cat /home/clove/.cache/bazel/_bazel_clove/f408d421e706f9a6112d2f3205e6556c/execroot/__main__/bazel-out/k8-fastbuild/testlogs/pkg/roll/roll_test/test.log
exec ${PAGER:-/usr/bin/less} "$0" || exit 1
Executing tests from //pkg/roll:roll_test
-----------------------------------------------------------------------------
--- FAIL: TestGenerateNumber (0.00s)
    roll_dice_test.go:25: got an empty string
FAIL
```

Adding the "test_ouput" argument to the Bazel test command will output the test results to the console.

```bash
$ bazelisk test --test_output=errors //...
INFO: Analyzed 5 targets (0 packages loaded, 0 targets configured).
INFO: Found 4 targets and 1 test target...
FAIL: //pkg/roll:roll_test (see /home/clove/.cache/bazel/_bazel_clove/f408d421e706f9a6112d2f3205e6556c/execroot/__main__/bazel-out/k8-fastbuild/testlogs/pkg/roll/roll_test/test.log)
INFO: From Testing //pkg/roll:roll_test:
==================== Test output for //pkg/roll:roll_test:
--- FAIL: TestGenerateNumber (0.00s)
    roll_dice_test.go:25: got an empty string
FAIL
================================================================================
INFO: Elapsed time: 0.191s, Critical Path: 0.03s
INFO: 3 processes: 1 internal, 2 linux-sandbox.
INFO: Build completed, 1 test FAILED, 3 total actions
//pkg/roll:roll_test                                                     FAILED in 0.0s
  /home/clove/.cache/bazel/_bazel_clove/f408d421e706f9a6112d2f3205e6556c/execroot/__main__/bazel-out/k8-fastbuild/testlogs/pkg/roll/roll_test/test.log

INFO: Build completed, 1 test FAILED, 3 total actions
```

If you like you can change the "if" statement back so that the unit test passes.

So now we know how to include a new unit test, update `BUILD.bazel` rules with Gazelle, and then run the test.

## Other rules in rules_go

The rules_go [documentation](https://github.com/bazelbuild/rules_go#documentation) provides a great reference to the different
rules provided in the ruleset.

We have covered three of the top rules: `go_binary`, `go_library`, and `go_test`.  We also covered rules that
Gazelle uses to manage dependencies called `go_repository`.

Other rules in the go_rules ruleset include:

- Proto rules that generate Go packages from .proto files. These packages can be imported like regular Go libraries.
- The Go toolchain is a set of rules used to customize the behavior of the core Go rules.  The Go toolchain allows for the configuration
of the Go distribution utilized. The toolchain declares Bazel toolchains for each target platform that Go supports. The context rules are all for writing custom rules
that are compatible with rules_go.
- Also, go_rules includes a rule for using go mock and the rule go_embed_data.
The rule go_embed_data generates a .go file that contains data from a file or a list of files. 
- The nogo rule support using nogo during testing. The code analysis tool nogo screens code preventing bugs and code anti-patterns, and can also run vet.

Other capabilities of go_rules include:

- creating pure go binaries
- building go static binaries
- basic race condition detection

And lastly, you probably know that Go supports cross-compilation, and this is really nice when we are developing with containers.  Within rules_go they 
have included go_cross_binary, which allows you to define the creation of a binary for a specific operating system and CPU architecture. This
can allow us to develop on a Mac and run the binary on that Mac, while also building a binary for Linux.  We then would use a set of Bazel
rules that support the building of containers, and Bazel can put the Linux binary in the container.

## Summary

- Bazel supports the building and testing of the Go programming language using the rules_go ruleset.
- Initially, you need to create a basic `WORKSPACE` and `BUILD.bazel` file in the root directory of your project.
- You can use Gazelle to create and maintain various Bazel files.
- Gazelle can update various Bazel files when you add a new go file or go tests.
- Bazel supports many commands, and we covered the build, run and test commands.
- Bazel uses a dependency graph that is based on `WORKSPACE`, `BUILD.bazel`, and other Bazel files.
- The ruleset rules_go provides various rules like go_binary, go_library and go_test.  They are used
to build binaries, libraries, and support unit testing.
- Gazelle can update `BUILD.bazel` and `deps.bzl` files with either internal or external Go dependencies.
- The go_test rule is used to define Go unit tests.
- go_rules definers various other rules.  These rules include managing protocol buffers, grpc, cross-compilation, and controlling various
aspects of how the Go SDK is downloaded and configured.
