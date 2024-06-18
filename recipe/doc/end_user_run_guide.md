# Guide for End-Users Running CUDA Code

This guide is for people who wish to use conda environments to run CUDA code.

## Prerequisites

To run CUDA code, you must have an NVIDIA GPU on your machine and you must install the CUDA drivers.
Note that CUDA drivers _cannot_ be installed with conda and must be installed on your system using an appropriate installation method.
See the [CUDA documentation](https://docs.nvidia.com/cuda/) for instructions on how to install ([Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html), [Windows](https://docs.nvidia.com/cuda/cuda-installation-guide-microsoft-windows/index.html)). 

Note that the CUDA packages are designed to be installable in a *CPU-only* environment (i.e. no CUDA driver or GPU installed), see the FAQ at the end of this page.

## Installing CUDA

### Basic Installation

The easiest one-step solution to install the full CUDA Toolkit is to install the [`cuda` metapackage](https://github.com/conda-forge/cuda-feedstock/) with this command:

```
conda install -c conda-forge cuda cuda-version=12.5
```

Let's break down this command.
We are requesting the installation of two metapackages here, `cuda` and `cuda-version`.
The `cuda` metapackage pulls in all the components of the CUDA Toolkit (CTK) and is roughly equivalent to installing the CUDA Toolkit with traditional system package managers like `apt` or `yum` on Linux.
Similarly to such package managers, the separate components may also be installed independently.
The [`cuda-version` metapackage](https://github.com/conda-forge/cuda-version-feedstock/blob/main/recipe/README.md) is used to select the version of CUDA to install.
This metapackage is important because individual components of the CTK are typically versioned independently (the current versions may be found in the [release notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html).
The `cuda-version` metapackage provides a standard way to install the version of a specific CUDA component corresponding to a given version of the CTK.
This way, you never have to specify a particular version of any package; you just specify the `cuda-version` that you want, then list packages you want installed and conda will take care of finding the right versions for you.
The above command will install all components of CUDA from the latest minor release of CUDA 12.5.

_Warning: there are contents of OS CUDA Toolkit installs that are not available in the `conda-forge` packages, including_:
- Driver libraries, such as (but not limited to)
    - `libcuda`
    - `libnvidia-ml` library
- Older versions of CTK contained these
    - Documentation
    - Samples (use to be)
- GPUDirect Storage (GDS)
    - Missing file system components
- fabricmanager
- `libnvidia_nscq`
- IMEX
- Nsight Systems

### Installing Subsets of the CUDA Toolkit

Rather than installing all of CUDA at once, users may instead install just the packages that they need. 
For example, to install just `libcublas` and `libcusparse` one may run:
```
conda install -c conda-forge libcublas libcusparse cuda-version=<CUDA version>
```
The best way to get a current listing is to run:
```
conda install --dry-run -c conda-forge cuda cuda-version=<CUDA version>
```
For a complete listing of the packages that were originally created, see [this issue](https://github.com/conda-forge/staged-recipes/issues/21382).

### Metapackages

Existing conda documentation: https://docs.nvidia.com/cuda/cuda-installation-guide-linux/#conda-installation

For convenience, a number of additional metapackages are available:
- `cuda-runtime`: All CUDA runtime libraries needed to run a CUDA application
- `cuda-libraries`: All libraries required to run a CUDA application requiring libraries beyond the CUDA runtime (such as the CUDA math libraries) as well as packages needed to perform JIT compilation
- `cuda-visual-tools`: GUIs for visualizing and profiling such as Nsight Compute
- `cuda-command-line-tools`: Command line tools for analyzing and profiling such as cupti, cuda-gdb, and Compute Sanitizer
- `cuda-tools`: All tools for analyzing and profiling, both GUI (includes cuda-visual-tools) and CLI (includes cuda-command-line-tools)

### CUDA C++ Core Libraries (CCCL)

CCCL is a special case among CUDA packages.
Due to 1) being header-only, 2) fast-moving, and 3) independently-evolving, consumers may want a different (newer) version of CCCL than the one corresponding to their CTK version.
Instructions on how to install a suitable CCCL package from conda can be found [in the CCCL README](https://github.com/NVIDIA/cccl/?tab=readme-ov-file#conda). (See [this issue](https://github.com/conda-forge/cuda-cccl-impl-feedstock/issues/2) for more information on the history of these packages).

## `conda-forge` vs `nvidia` channel

Understanding the difference between the CUDA packages on the `conda-forge` and `nvidia` channels requires a bit of history because of how the relationship has evolved over time.
In particular, how these channels may or may not coexist will depend on the versions of CUDA that you need support for.

### Pre-CUDA 12:

Prior to CUDA 12, the only package available on `conda-forge` was the `cudatoolkit` package, a community-maintained, monolithic package containing the entire repackaged CTK.
During the CUDA 11 release cycle, NVIDIA began maintaining a set of CUDA Toolkit packages in the `nvidia` channel.
Unlike the monolithic `conda-forge` package, the `nvidia` channel distributed the CTK split into components such that each library was given its own package.
This package organization made it possible to install separate components independently and better aligned the conda packaging ecosystem with other package managers, such as those for Linux distributions.
However, this organization introduced a number of changes that were at times confusing -- such as the introduction of a `cuda-toolkit` (note the hyphen) metapackage that installs a partially overlapping set of components to the original `cudatoolkit` -- and at other times breaking, particularly in conda environments configured to pull packages from both `conda-forge` and the `nvidia` channel.
Therefore, in a CUDA 11 world the `conda-forge` and `nvidia` channels were difficult to use in the same environment without some care.

### CUDA 12.0-12.4

With the CUDA 12 release, NVIDIA contributed the new packaging structure to `conda-forge`, introducing the same set of packages that existed on the `nvidia` channel as a replacement for the old `cudatoolkit` package on `conda-forge`.
This was done starting with CUDA 12.0 to indicate the breaking nature of these changes compared to the prior CUDA 11.x packaging in `conda-forge`.
These packages became the standard mechanism for delivering CUDA conda packages.
Due to the scale of the reorganization, the CUDA 12.0, 12.1, and 12.2 releases also involved numerous additional fixes to the packaging structure to better integrate them in the Conda ecosystem.
Due to the number of such changes that were required and the focus on improving the quality of these installations, during this time period no corresponding updates were provided for packages on the `nvidia` channel.
While the `conda-forge` and `nvidia` channel package lists were the same (i.e. the same packages existed in both places with the same core contents like libraries and headers), the `nvidia` channel did not include many of the incremental fixes made on `conda-forge` to improve things like symlinks, static library handling, proper package constraints, etc.
As a result, `nvidia` and `conda-forge` CUDA packages remained incompatible from CUDA 12.0-12.4.

### CUDA 12.5+

With CUDA 12.5, the `nvidia` channel was fully aligned with `conda-forge`.
Packages on both channels are identical, ensuring safe coexistence of the two channels within the same conda environment.

Going forward, CUDA packages on the `conda-forge` and `nvidia` channels should be expected to remain compatible.

## FAQ

### What if I see an error saying `__cuda` is too old?

The `__cuda` virtual package is used by `conda` to represent the maximum CUDA version fully supported by the display driver. See the [conda docs on virtual packages](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-virtual.html) for more information.

To update the `__cuda` virtual package, you must install a newer driver:
- [Linux instructions](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#driver-installation)
- [Windows instructions](https://docs.nvidia.com/cuda/cuda-installation-guide-microsoft-windows/index.html#installing-cuda-development-tools)

If conda has incorrectly identified the CUDA driver, you can override by setting the `CONDA_OVERRIDE_CUDA` environment variable to a version number like `"12.5"` or `""` to indicate that no CUDA driver is detected.

### Can I install CUDA conda packages in a CPU-only environment (such as free-tier CIs)?

Yes! All of the CUDA packages can be installed in an environment without the presence of a physical GPU or CUDA driver.
The inter-package dependency is established properly so that this use case is covered.
If you want to test package installation assuming a certain driver version is installed, use the `CONDA_OVERRIDE_CUDA` environment variable mentioned above.
Even if the package requires CUDA to run, this allows the packaging and dependency resolution to be tested in a CPU-only environment.
