# Guide for End-Users Compiling CUDA Code

This guide is for people who wish to use conda environments to compile CUDA code.
Most of the sections of the guide for [running CUDA code](./end_user_run_guide.md) also apply here.
The main difference for users only compiling libraries is that for compilation neither the CUDA driver nor a GPU are required.
Therefore, for building CUDA code the conda packages alone are sufficient with no additional requirements on the user's system.
There are a few other important points that users compiling CUDA code in conda environments should be aware of.

## Package Naming Conventions

If you plan to install and build against CUDA packages, you will need to be aware of how libraries are split into packages.
Packages containing libraries (as opposed to compilers or header-only components) follow specific naming conventions.
Typically library components of the CUDA Toolkit (CTK) are split into three pieces: the base package, a `*-dev` package, and a `*-static` package.
Using [the cuBLAS library](https://github.com/conda-forge/libcublas-feedstock) as an example, we have three different packages:
The base `libcublas` package, which installs the `libcublas.so.X` symlink and the `libcublas.so.X.Y` shared library, is sufficient for use if you are simply installing other packages that require cuBLAS at runtime.
The `libcublas-dev` package installs additional files like cuBLAS headers, the `libcublas.so` symlink, and CMake files.
This package should be installed if you wish to compile your own code dynamically linking against cuBLAS within a conda environment.
The `libcublas-static` package installs the static `libcublas_static.a` library.
This library should be installed if you wish to compile your own code linking against a static cuBLAS within a conda environment.
Typically the `*-static` packages will require the `*-dev` packages to be installed in order to provide the necessary packaging (CMake, pkg-config) files to discover the library, but this is not currently enforced by the packages themselves.

## Development Metapackages

The above discussion of naming also applies to metapackages.
For instance, the `cuda-libraries` package contains all the runtime libraries, while `cuda-libraries-dev` also includes dependencies on the corresponding `*-dev` packages.
In addition, for the purposes of development there are a few additional key metapackages:
- `cuda-compiler`: All packages required to compile a minimal CUDA program (one that does not require e.g. extra math libraries like cuBLAS or cuSPARSE).
