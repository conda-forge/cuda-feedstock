# CUDA metapackage

This metapackage corresponds to installing all packages in a CUDA release.
It is suitable for use by both developers aiming to build CUDA applications and end-users running CUDA.
More information for different classes of users is documented in the documents below:

- [Guide for End-Users Running CUDA Code](./doc/end_user_run_guide.md)
- [Guide for End-Users Compiling CUDA Code](./doc/end_user_compile_guide.md)
- [Guide for Maintainers of Recipes That Use CUDA](./doc/recipe_guide.md)
- [Guide for Maintainers of CUDA recipes](./doc/maintainer_guide.md)

## Versioning

On conda-forge, the version of installed CUDA packages can be controlled by
the meta package `cuda-version`. See the detailed documentation [here](https://github.com/conda-forge/cuda-version-feedstock/blob/main/recipe/README.md).

### CUDA Metapackage Versioning

The version of a CUDA Toolkit metapackage corresponds to the CUDA release
label. For example, the release label of CUDA 12.0 Update 1 is 12.0.1.  This
does not include the `cuda-version` metapackage which is versioned only by the
MAJOR.MINOR of a release label.

### Metapackage dependency versions

Installing a metapackage at a specific version should install all dependent
packages at the exact version from that CUDA release.

### Metapackage dependencies on cuda-version

Metapackages do not directly constrain to a specific `cuda-version` as their
version is more precise. Dependent packages will still install an appropriate
`cuda-version`.
