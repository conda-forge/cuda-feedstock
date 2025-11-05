# Guide for Maintainers of CUDA recipes

This guide is intended for maintainers of the CUDA packages themselves.

## Rationale for split packages

In addition to the standardized dev/static division of libraries, some packages are also divided into multiple pieces for more specialized reasons.

### nvcc split

While the `nvcc` compiler natively supports cross-compilation, a target-specific headers are still needed.
To support this, the `nvcc` compiler is split into a couple of feedstocks, [`cuda-nvcc`](https://github.com/conda-forge/cuda-nvcc-feedstock/) and [`cuda-nvcc-impl`](https://github.com/conda-forge/cuda-nvcc-impl-feedstock/).
These packages split the files such that we can have the compiler package be dependent exclusively on the platform for which it is compiled while the `cuda-nvcc-impl` package is dependent only on the cross-compilation target and includes the required headers (and other files) such that compilation will succeed.
This way, the two packages may be updated or changed in parallel and will interoperate properly in cross-compilation environments.

## CCCL packaging

As discussed in the [runtime guide](./doc/end_user_run_guide.md), CCCL is a special case for a number of reasons.

### Packages to install

CCCL packaging is split into two primary feedstocks:
- The [cuda-cccl](https://github.com/conda-forge/cuda-cccl-feedstock) provides the version of CCCL associated with a given CUDA Toolkit (CTK) version.
- The [cccl](https://github.com/conda-forge/cccl-feedstock) provides the latest version of CCCL independent of CTK version.

The `cccl` package installs headers directly into `${PREFIX]/include`.
The [cuda-cccl recipe](https://github.com/conda-forge/cuda-cccl-feedstock/blame/main/recipe/meta.yaml) is actually a split recipe that produces multiple packages:
- `cuda-cccl_{{target}}`: As discussed in [the recipe guide](./doc/recipe_guide.md), to support cross-compilation these CCCL headers are split into target-specific packages in the `${PREFIX}/targets` directories on Linux. These packages are used as dependencies by other CTK packages like `cuda-nvcc` and `cuda-cudart` to indicate a dependency that also presumes that the nvcc compiler will know to search inside this `targets`-based layout. These files also do not install any headers into the `${PREFIX}/include` directory. These files should never be used directly by end-users.
- `cuda-cccl`: This is the user-facing package mentioned above that provides the version of CCCL associated with a given CTK version. It is actually nothing more than a metapackage that brings in corresponding versions of `cccl` and `cuda-cccl_{{target}}` as dependencies to ensure consistent versioning.

This combination of packages preserves user flexibility to select CCCL by its own version or by CTK version.

### Include directory ordering

The need to support two different CCCL packages introduces significant additional complexity when considering when and how include directories are added to compilation commands.
A major source of complexity is how compilers handle includes in general, particularly when mixing system includes and user includes:

- Compilers parse include directories in the order they are specified on the command line from left to right, so later includes will override earlier ones.
- `-I` includes take precedence over `-isystem` includes, BUT
- If the same directory is added as both `-I` and `-isystem`, `-isystem` always takes precedence. This is because:
- `-isystem` includes suppress warnings from headers included from those directories, so it is often desirable to include third-party headers as `-isystem` to avoid spamming users with warnings from code they do not control.

`nvcc` automatically configures some include directories via the `nvcc.profile` file.
The top-level `include/` directory is added automatically by `nvcc` as a `-I` include directory.
In practice it is very common to compile CUDA code using CMake since it has the most complete CUDA support of any build system.
CMake's FindCUDAToolkit module creates targets that always mark CTK include directories as `-isystem` includes (primarily so that warnings will be suppressed, as discussed above).
That further complicates the story because if the CTK CCCL installation is in the root `include/` directory then the `-isystem` include of `include/` will take precedence over the `nvcc` default of `-I`.
Other non-CMake build systems may have similar behavior.

These include order rules make it impossible to specify what version of CCCL to use if multiple copies are installed on the same system and the system CCCL is installed directly into `include/` and potentially included as `-isystem`.
To resolve this problem, in CUDA 13 the CTK layout was modified to place CCCL headers into a dedicated subdirectory `include/cccl`, and this directory was added to `nvcc.profile` as a `-isystem` include.
This change made it possible for users to override CTK-installed CCCL headers with a different version in another directory.
Most of the discussion here is thus largely for CUDA>=13 since multiple separate CCCL installations simply cannot be supported in CUDA<=12.

In normal system installations of CUDA (e.g. via deb or RPM), the contents of `${PREFIX]/include` and `${PREFIX}/targets/${target}/include` are identical.
However, this separation exists precisely so that cross-compilation scenarios like conda's may be supported.
The conda-forge CUDA packages take advantage of this by not installing CCCL headers into `${PREFIX}/include` at all, but by [patching `nvcc.profile` to move all of its searches into the target-specific directory](https://github.com/conda-forge/cuda-nvcc-impl-feedstock/blob/main/recipe/nvcc.profile.patch).
Ultimately, this means that the CCCL headers provided by the CTK in `${PREFIX}/targets/${target}/include` are included as `-isystem` from the targets directory by `nvcc` automatically, even though the `${PREFIX}/include` directory is included as `-I`.
Since the `cccl` package installs headers directly into `${PREFIX}/include`, these headers will take precedence over those provided by the CTK during compilation.
Therefore, if a user installs the `cccl` package alongside parts of the CTK that install one of the `cuda-cccl_{{target}}` packages, the user-installed CCCL will be used during compilation instead of the CTK-installed CCCL.

All of the above discussion regarding supporting dual CCCL installations applies to all platforms.
Note that on Windows since cross-compilation is not supported there the targets directory is not generally used by most CTK packages.
However, the CCCL do use it so that this dual installation mechanism still works properly.
