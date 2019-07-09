Below you'll find a series of tutorials-by-example of increasing complexity, utilising more of Rez's functionality as we go, solving more and more specific problems.

<br>

## Basics

Let's start with the basics.

<br>

### Shortest Possible Example

Create and use a new package from scratch in under 40 seconds.

```powershell
mkdir mypackage                           # Name of your Git project
cd mypackage                              # Rez definition
"name = `"mypackage`""  | Add-Content package.py  # Rez package name
"version = `"1.0`""     | Add-Content package.py  # Rez package version
"build_command = False" | Add-Content package.py  # Called on building package
rez build --install                       # Build package
rez env mypackage                         # Use package
> $                                       # A new environment with your package
```

- The `>` symbol means you are in a Rez "context".
- Type `exit` to exit the context.

<br>

### Environment Variables

Most packages will modify their environment in some way.

**package.py**

```python
name = "mypackage"
version = "1.1"
build_command = False

def commands():
    global env  # Global variable available to `commands()`
    env["MYVARIABLE"] = "Yes"
```

This package will assign `"Yes"` to MYVARIABLE.

- `env` A global Python variable representing the environment
- `env["MYVARIABLE"]` - An environment variable
- `env.MYVARIABLE` - This is also OK

```powershell
$ rez build --install
$ rez env mypackage
> $ echo %MYVARIABLE%
Yes
```

<br>

### Environment Paths

A package can also modify paths, like `PATH` and `PYTHONPATH`, without removing what was there before.

**package.py**

```python
name = "mypackage"
version = "1.2"
build_command = False

def commands():
    global env
    env["PYTHONPATH"].prepend("{root}")
    env["PYTHONPATH"].prepend("{root}/python")
```

This package will assign `"{root}"` to `PYTHONPATH`.

- `{root}` expands to the absolute path to the installed package
- `env["PYTHONPATH"].prepend()` - Prepend a value to this variable
- `env["PYTHONPATH"].append()` - Append a value to this variable

```powershell
$ rez build --install
$ rez env mypackage
> $ echo %PYTHONPATH%
\\server\packages\mypackage\1.2;\\server\packages\int\mypackage\1.2\python
```

<br>

### Requirements

Most packages will depend on another package.

```powershell
cd mypackage
cd ..
mkdir mypackage2
touch mypackage2/package.py
```

**mypackage2/package.py**

```python
name = "mypackage2"
version = "1.0"
build_command = False
requires = ["python-3", "mypackage-1.2"]
```

This package now requires `python-3` and `mypackage-1.2`.

```powershell
$ rez build --install
$ rez env mypackage2
resolved by manima@toy, on Thu Jun 27 11:12:18 2019, using Rez v2.32.1

requested packages:
mypackage2
~platform==windows           (implicit)
~arch==AMD64                 (implicit)
~os==windows-10.0.18362.SP0  (implicit)

resolved packages:
arch-AMD64        C:\Users\manima\packages\arch\AMD64                                (local)
mypackage-1.3     C:\Users\manima\packages\mypackage\1.3                             (local)
mypackage2-1.0    C:\Users\manima\packages\mypackage2\1.0                            (local)
platform-windows  C:\Users\manima\packages\platform\windows                          (local)
python-3.7.3      C:\Users\manima\packages\python\3.7.3\platform-windows\arch-AMD64  (local)
> $
```

<br>

### Payload

Most packages will have additional files, such as Python modules. This is where `build_command` comes in.

```powershell
$ cd mypackage
$ touch install.py                           # Additional script for build
$ mkdir python                               # Payload directory
$ cd python                                  # 
$ echo print("Hello World!") >> mymodule.py  # Python payload shipped alongside package
```

**package.py**

```python
name = "mypackage"
version = "1.3"
build_command = "python {root}/install.py"      # Run this command on `rez build`
requires = ["python-3"]

def commands():
    global env
    env["PYTHONPATH"].prepend("{root}/python")  # Add payload to environment
```

**install.py**

```python
# This script is called on `rez build`
import os
import shutil

print("Running install.py...")
root = os.path.dirname(__file__)
build_dir = os.environ["REZ_BUILD_PATH"]
install_dir = os.environ["REZ_BUILD_INSTALL_PATH"]

print("Copying payload to %s.." % build_dir)
shutil.copytree(
    os.path.join(root, "python"),
    os.path.join(build_dir, "python"),
    ignore=shutil.ignore_patterns("*.pyc", "__pycache__")
)

if int(os.getenv("REZ_BUILD_INSTALL")):
    # This part is called with `rez build --install`
    print("Installing payload to %s..." % install_dir)
    shutil.copytree(
        os.path.join(build_dir, "python"),
        os.path.join(install_dir, "python"),
    )
```

Now let's build it.

```powershell
$ rez build --install
$ rez env mypackage
> $ python -m mymodule
Hello World!
```