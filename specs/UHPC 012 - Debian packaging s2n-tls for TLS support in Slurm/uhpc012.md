---
index: UHPC012
title: Debian packaging s2n-tls for TLS support in Slurm
---

# Debian packaging s2n-tls for TLS support in Slurm

## Abstract

This spec defines a detailed plan to package the [`s2n-tls`](https://github.com/aws/s2n-tls) library,
future improvements that we might add after the initial packaging efforts, and unanswered questions
about the process.

## Rationale

Enabling [TLS support in Slurm](https://slurm.schedmd.com/tls.html) currently requires building
the `s2n-tls` library from source. This is fine for manually deployed clusters, but for
Charmed HPC we require something that is not as heavy on deployment times.

Debian packaging `s2n-tls` would solve this problem, but packaging a library from scratch
is a whole ordeal. Therefore, we need to make an exhaustive plan of attack for this
problem.

## Specification

The following conventions mostly follow the recommendations established by the
Debian packaging team on [this document](https://dep-team.pages.debian.net/deps/dep14/).

### Preliminary work

No preliminary work exists on either Debian repositories or unofficial PPAs.
The slurm documentation itself recommends building the library by source, which
is not an optimal solution for our requirements.

### Repository structure

The repository must contain at least the following two branches:

- `upstream/latest`, the pristine upstream source tree.
- `pristine-tar`, branch containing deltas for the upstream tarballs.

The repository can also contain branches to package the library for every Ubuntu
release being supported, with the format `ubuntu/<version-number>-<release>`
(e.g. `ubuntu/24.04-noble`).
We might also want to include a separate `ubuntu/<version-number>-<release>-devel`
(e.g. `ubuntu/26.04-resolute-devel`)
branch if required, but for a first packaging pass, using a single branch per
release seems reasonable.

### Tooling

Since the upstream library uses git extensively to develop the library, it is
recommended to use `git-buildpackage` as the main tool to interact with the
repository. The Debian Wiki contains a useful [guide](https://wiki.debian.org/PackagingWithGit)
for this task.

This also means Quilt must be used to manage patches, since it will automatically
give us support for managing patches using `gbp pq`. Thus, the file
`debian/source/format` must contain the following:

```
3.0 (quilt)
```

### Upstream versioning

Upstream uses semantic versioning in the form `X.Y.Z`, but in an inconsistent way;
most releases use `vX.Y.Z` ([example](https://github.com/aws/s2n-tls/releases/tag/v1.7.2)),
but in some rare occasions they will use only `X.Y.Z`, without a "v" prefix
([example](https://github.com/aws/s2n-tls/releases/tag/1.7.1)).

If we want to support both formats, the `debian/watch` file might grow in complexity,
but it is feasible.

### Required packages

At the moment we only require a single package: `libs2n-1`. This package needs
to ship the shared library `libs2n.so.1` such that Slurm can dynamically import
it for its TLS support.

### Optional packages

We could also package a static version of the library if required (`libs2n-dev`),
to ship development headers for the library. This might or might not include the
Rust bindings offered by `s2n-tls`. At this point in time, we don't require those
for TLS support in Slurm, so it is intentionally excluded from the first packaging
pass.

### SONAME versioning

Unfortunately, the upstream library does _not_ update the shared library's SONAME
version on every new release, as shown on `CMakeLists.txt`:

```
# These Version numbers are for major updates only- we won't track minor/patch updates here.
set(VERSION_MAJOR 1)
set(VERSION_MINOR 0)
set(VERSION_PATCH 0)
```

Thus, for good measure we might need to patch this up on every version or try to
find a way to dynamically patch it using the release version.

### Build Rules

This section is just informational, since many details would need to be fleshed
out after starting the packaging work.

`s2n-tls` offers several options to customize the type of build required
([`CMakeLists.txt`](https://github.com/aws/s2n-tls/blob/65fd1a9f7bae1309a6c0815a136d70c30452a0f3/CMakeLists.txt)
should contain the exhaustive list of all build options).
The recommended set of options for this first pass are as follows:

- `BUILD_SHARED_LIBS=ON`: required by Slurm.
- `BUILD_TESTING=OFF`: no need to do integration testing for a first packaging pass.
- `UNSAFE_TREAT_WARNINGS_AS_ERRORS=OFF`: avoids erroring if compiling the library
  with an older version of g++ (might be common if supporting older Ubuntu releases).
- `S2N_INSTALL_S2NC_S2ND=OFF`: avoids bundling test binaries
- `S2N_INTERN_LIBCRYPTO=OFF`: do **not** bundle a libcrypto library, preferring
  the system's libcrypto.
- `S2N_ENFORCE_PROPER_LIBCRYPTO_FEATURE_PROBE=ON`: ensures the linked `libcrypto`
  has all the required features for the selected set of `s2n-tls` features.

A sample `debian/rules` file is provided, but it might have to be adjusted depending
on our future requirements:

```
#!/usr/bin/make -f

export DEB_BUILD_MAINT_OPTIONS = hardening=+all

%:
  dh $@ --buildsystem=cmake --builddirectory=_build

override_dh_auto_configure:
  dh_auto_configure -- \
		-DCMAKE_BUILD_TYPE=RelWithDebInfo \
		-DBUILD_SHARED_LIBS=ON \
		-DBUILD_TESTING=OFF \
		-DUNSAFE_TREAT_WARNINGS_AS_ERRORS=OFF \
		-DS2N_INSTALL_S2NC_S2ND=OFF \
		-DS2N_INTERN_LIBCRYPTO=OFF \
		-DS2N_ENFORCE_PROPER_LIBCRYPTO_FEATURE_PROBE=ON
```

Installing the required dynamic library only requires adding the
following file in `debian/libs2n-1.install`:

```
usr/lib/*/libs2n.so.1*
```

Then, its corresponding `debian/libs2n-1.symbols` file will be generated
after running `gbp buildpackage -us -uc` for the first time. This needs to be
committed such that we have a reference point to detect ABI breaks.

### Open questions

+ **SOVERSION**: Upstream hardcodes `SOVERSION=1` and
  `VERSION=1.0.0`, and they don't bump these between releases, relying on semver.
  We might need to contact the `s2n-tls` team to request proper ABI version stability.

+ **Build tests**: `s2n-tls` can run unit tests with the `BUILD_TESTING=ON`
  option. However, it also required specific dependencies that might be heavy for
  running them at build time. We need to investigate if Debian packages commonly
  enable these tests.

+ **Missing pkg-config file**: A `.pc` file for `pkg-config` compatibility is not
  strictly required at this point, since Slurm only needs to link against the
  shared library. However, it might be required if we plan on offering a `s2n-tls-dev`
  static library.

+ **Rust bindings**: The bindings for the Rust language are not required for a
  shared library build, so this first pass intentionally excludes them from the
  packaging efforts.

+ **Unstable API headers**: Upstream ships unstable headers under `api/unstable/`.
  Investigate what is the typical way to deal with this on Debian packages.
