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

## Building for ARM Tegra devices

The [arm-variant](https://github.com/conda-forge/arm-variant-feedstock) package allows
end-users to select which variant of ARM packages are installed into their environment.
However, since there are no Tegra devices available for compilation, these packages must be
cross-compiled from x86. Recipes that wish to build for both SBSA ARM and Tegra ARM devices
should have something like the following in their recipe:

```yaml
requirements:
    build:
        - {{ compiler('cuda') }}
        - arm-variant * {{ arm_variant_type }}
    host:
        - arm-variant * {{ arm_variant_type }}
```

where the `recipe/conda_build_config.yaml` contains something like:

```yaml
arm_variant_type:  # [linux and aarch64]
  - sbsa           # [linux and aarch64]
  - tegra          # [linux and aarch64]
```

where the `conda-forge.yml` contains something like:

```yaml
build_platform:
  linux_aarch64: linux_64
provider:
  linux_aarch64: default
```

The compute capabilities for GPUs on Tegra and non-Tegra devices are mutually exclusive, and
code compiled for Tegra and non-Tegra devices are not interchangeable, so one build cannot
service all ARM variants. For developer convenience, the CUDA compiler package activation
script automatically sets the environment variables `CUDAARCHS` and `TORCH_CUDA_ARCH_LIST`
to all supported CUDA architectures for the target platform. These environment variables are
consumed by CMake and Pytorch respectively.
