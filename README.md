<!---
MIT License

CMake build script for the Accelerate LAPACKE project
Copyright (c) 2025 Tim Kaune

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
--->

# Accelerate LAPACKE #

Unfortunately, Apple's Accelerate framework doesn't provide the LAPACKE
C-interface library. Since MacOS 13.3 Ventura, the [BLAS/LAPACK
interface][accelerate-docs] provided by Accelerate is recent enough to support
the LAPACKE library.

This project uses [Accelerate LAPACK][accelerate-lapack] to build the LAPACKE
library against the Accelerate NEWLAPACK interface. More info on BLAS/LAPACK in
Accelerate can be found there, too.

[accelerate-docs]: https://developer.apple.com/documentation/accelerate/blas-library
[accelerate-lapack]: https://github.com/lepus2589/accelerate-lapack

- [How to compile](#how-to-compile)
  - [Prerequisites](#prerequisites)
  - [Workflow with CMake](#workflow-with-cmake)
    - [CMake v4 compatibility](#cmake-v4-compatibility)
  - [Using LAPACKE in another project](#using-lapacke-in-another-project)

## How to compile ##

It is recommended to use the Apple System C Compiler `/usr/bin/cc`. You can also
use a more recent Clang compiler provided by Homebrew or MacPorts. If you have
other compilers installed on your system, make sure CMake finds the correct one.
Otherwise, help CMake by setting the environment variable `$ export
CC=/usr/bin/cc` in your terminal window.

It is also recommended to use at least CMake v3.25 with presets, but the CMake
script also works down to CMake v3.12, if you set the required variables on the
command line.

### Prerequisites ###

Obviously, your operating system must be Mac OS X >=v13.3 Ventura with XCode
installed. Additionally, you'll need the following software (easily obtainable
via Homebrew or MacPorts):

- CMake
- Fortran compiler (e.&nbsp;g. `gfortran` contained in the `gcc` package) to
  make it through the Reference LAPACK configure script

### Workflow with CMake ###

First, you must [select a MacOS SDK matching your MacOS system][sdk-selection].

Then use the `accelerate-lapacke32-install` preset (or the
`accelerate-lapacke64-install` preset for the ILP64 interface) with CMake called
by `xcrun`, as described in the SDK selection:

```shell
$ cmake --workflow --preset accelerate-lapacke32-install
```

This will configure and install LAPACKE in your `~/.local` directory by
default. If you prefer a different install location (e.&nbsp;g. `/opt/custom`),
you can change it using a `CMakeUserPresets.json` file, for which a template
file is provided:

```shell
$ cmake --workflow --preset user-accelerate-lapacke32-install
```

I wouldn't recommend installing to `/usr/local` (used by Homebrew on Intel Macs)
or `/opt/local` (used by MacPorts).

If your install location is write protected, use the `user-accelerate-lapacke32`
and `user-accelerate-lapacke64` workflow presets to configure and build the
project only and then install with:

```shell
$ sudo cmake --build --preset user-accelerate-lapacke32-install
```

and

```shell
$ sudo cmake --build --preset user-accelerate-lapacke64-install
```

Analyzing the resulting `.dylib` with `otool`, you can see:

```shell
$ otool -L <install prefix>/lib/liblapacke.dylib
<install prefix>/lib/liblapacke.dylib:
    @rpath/liblapacke.3.dylib (compatibility version 3.0.0, current version 3.12.0)
    /System/Library/Frameworks/Accelerate.framework/Versions/A/Accelerate (compatibility version 1.0.0, current version 4.0.0)
    /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1351.0.0)
```

Only the Accelerate framework and the System library are linked into the
`.dylib`. No `libgfortran` or other libraries are needed.

[sdk-selection]: https://github.com/lepus2589/accelerate-lapack?tab=readme-ov-file#maxos-sdk-selection

### Using LAPACKE in another project ###

You can use your self-compiled LAPACKE library in other projects by importing
the CMake package in the other project's `CMakeLists.txt` file:

```cmake
find_package(LAPACKE CONFIG)
```

or

```cmake
find_package(LAPACKE64 CONFIG)
```

and providing the above install location via the `CMAKE_PREFIX_PATH` variable
from the command line:

```shell
$ cmake -S . -B ./build -D "CMAKE_PREFIX_PATH=~/.local"
```

This makes the imported `lapacke`/`lapacke64` shared library target available in
the other project's `CMakeLists.txt`.
