# Guide for Maintainers of Recipes That Use CUDA

This guide is intended for maintainers of other recipes that depend on CUDA.
It assumes familiarity with the user guides for both [running CUDA code](./end_user_run_guide.md) and [compiling CUDA code](./end_user_compile_guide.md) with `conda-forge` packages.

## Best Practices

Recipe maintainers are encouraged not to use the metapackages for specifying dependencies, but to instead specify only the minimal subset of CUDA components required for usage of their libraries.
The `*-dev` variants of the packages all have [`run_exports`](https://docs.conda.io/projects/conda-build/en/latest/resources/define-metadata.html#export-runtime-requirements) set up such that if your package requires a certain CUDA library at run time, specifying the corresponding `dev` package as a host requirement will result in your package exporting the corresponding non-dev package as a runtime dependency.

## CUDA Enhanced Compatibility

Since CUDA 11, [CUDA has offered a number of increased compatibility guarantees](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#cuda-compatibility-developer-s-guide).
The most up to date documentation for these may be found in [the CUDA best practices documentation](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#cuda-toolkit-versioning).
Of particular interest from a conda packaging perspective is the binary backwards compatibility made by the CUDA Toolkit (CTK): packages built with newer versions of the CTK will also run with older minor versions (within the same major family) of the CTK installed, assuming that your package has properly guarded any usage of APIs introduced in newer versions with suitable checks.
This guide assumes that you already know how to write and compile code that supports such behavior.
If your library is properly configured as such, you will need to do a bit of extra work to ensure that your conda package supports this as well:

- You must define [`ignore_run_exports_from`](https://docs.conda.io/projects/conda-build/en/latest/resources/define-metadata.html#export-runtime-requirements) for any CUDA library `-dev` packages that your package depends on. Otherwise, the `run_exports` would require the runtime libraries to have versions equal to or greater than the versions used to build the package.
- You must `ignore_run_exports` of the `cuda-version` package.
- You must add explicit runtime dependencies (or `run_constrained` for soft/optional dependencies) that specify the desired version range for the dependencies whose `run_exports` have been ignored.

As an example, consider that you have built a package that requires `libcublas`:


```yaml
requirements:
  build:
    - {{ compiler('cuda') }}
  host:
    - libcublas-dev
    - cuda-version={{ cuda_compiler_version }}
```

Because of run-exports in the `libcublas-dev` package, your library will have a `run` requirement on `libcublas` corresponding to `{{ cuda_compiler_version }}` or newer.
To make this compatible with all CUDA minor versions from the same CUDA major version family, you must add the following:
```yaml
build:
  # Ignore run exports from CUDA libraries in the host section
  ignore_run_exports_from:
    - libcublas-dev
  # Ignore cuda-version directly. Do not ignore from {{ compiler('cuda') }}
  # because this package has other exports
  ignore_run_exports:
    - cuda-version

requirements:
  build:
    - {{ compiler('cuda') }}
  host:
    - libcublas-dev
    - cuda-version={{ cuda_compiler_version }}
  run:
    # Since we've ignored the run export from libcublas-dev, we pin
    # libcublas manually. Having a pin_compatible on cuda-version with
    # max_pin='x' and min_pin='x' will constrain libcublas to the same
    # major version as what we pinned in host.
    - {{ pin_compatible('cuda-version', max_pin='x', min_pin='x') }}
    - libcublas
```

This strategy applies to many other CUDA libraries as well.

For packages that need to support CUDA major versions earlier than CUDA 12, you will need to use selectors and/or Jinja tricks to separate out the requirements. [cupy-feedstock](https://github.com/conda-forge/cupy-feedstock) offers a good example.


## Cross-compilation

The CUDA recipes are designed to support cross-compilation.
As such, a number of CUDA components on `conda-forge` are split into `noarch: generic` component packages that are named according to the supported architecture, rather than being architecture-specific packages.
The canonical example is [the cuda-nvcc package](https://github.com/conda-forge/cuda-nvcc-feedstock/blob/main/recipe/meta.yaml) that contains the CUDA `nvcc` compiler.
This package is split into the `cuda-nvcc` package -- which is architecture specific and must be installed on the appropriate target platform (e.g.
x86-64 Linux) -- and the `cuda-nvcc_${TARGET_PLATFORM}` packages -- each of which is architecture-independent and may be installed on any target, but are only suitable for use in compiling code for the specified target platform.
This approach allows using host machines with a single platform to compile code for multiple platforms.

## Building for ARM Tegra devices

> [!IMPORTANT]
> Support for Tegra builds on conda-forge is only available for CUDA 12.9 and later.

The [arm-variant](https://github.com/conda-forge/arm-variant-feedstock) package allows
end-users to select which variant of ARM packages are installed into their environment.
However, since there are no Tegra devices available for compilation, these packages must be
cross-compiled from x86. Recipes that wish to build for both SBSA ARM and Tegra ARM devices
should have something like the following in their recipe:

```yaml
requirements:
  build:
    - {{ compiler('cuda') }}
    - arm-variant * {{ arm_variant_type }}  # [linux and aarch64]
```

> [!NOTE]
> The `arm-variant` package will cause overlinking warnings for itself from `conda-build`.
> This is expected and may be safely ignored.

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
device-code compiled for Tegra and non-Tegra devices are not interchangeable, so one build
cannot service all ARM variants. For developer convenience, the CUDA compiler package
activation script for `cuda-nvcc >=12.8` automatically sets the environment variable
`CUDAARCHS` to all supported CUDA architectures for the target platform when `CONDA_BUILD` is
detected and when `CUDAARCHS` is not already set. This environment variable is a standard CMake
environment variable.

## Directory structure

### Linux

The `conda-forge` CUDA packages aim to satisfy two sets of constraints.
On one hand, the packages aim to retain as similar a structure as possible to the CUDA packages that may be installed via system package manager (e.g. `apt` and `yum`) while supporting cross-compilation.
On the other hand, the packages aim to provide a seamless experience at both build time and run time within conda environments.
To satisfy the first requirement, all files in CUDA conda packages are installed into the `$PREFIX/targets` directory.
This includes libraries, headers, and packaging files, along with other miscellaneous files that may be present in any given package.
To satisfy the second requirement, we symlink a number of these files into standard sysroot locations so that they can be found by standard tooling (e.g.
CMake, compilers, ld, etc).
Specifically, we apply the following conventions:
- Shared libraries are symlinked into `$PREFIX/lib`. This includes the bare name (`libcublas.so`), the SONAME, and the full name.
- Pkgconfig files are installed directly into `$PREFIX/lib/pkgconfig`. These are not symlinked from `$PREFIX/targets`, but are directly moved to this location. The reason is that pkgconfig files contain relative paths to libraries/headers/etc and the paths cannot be accurate relative to both the `targets` directory and the `lib/pkgconfig` directory. Since the latter is what `pkgconfig` will use, we choose to install the files into `lib/pkgconfig` and reroot the paths accordingly.
- Static libraries and header files are not symlinked into the sysroot directories. Instead, conda installations of `nvcc` know how to search for these packages in the correct directories.
