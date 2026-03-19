---
index: UHPC008
title: `charmed-hpc-libs` for HPC charm development
---

# `charmed-hpc-libs` for HPC charm development

## Abstract

This specification formally defines the `hpc-libs` library used by all the HPC-owned charms,
and it proposes renaming the library to `charmed-hpc-libs` to better reflect the scope
of how this library is used in HPC-owned code bases such as `slurm-charms`.

## Rationale

The `hpc-libs` library has been used for the last two years to host and distribute common HPC charm
building blocks like the "conditions" framework or interface definitions. However, there has never
been a formal definition for the scope of this library, which has caused `hpc-libs` to become a catch-all
for HPC charm building blocks. The lack of a formal definition for the purpose of `hpc-libs` has caused
several issues such as:

1. It's difficult to determine what should go in `hpc-libs` versus what should be provided by
   local workspaces like `slurm-ops` or live directly within a charm.
2. Making updates to interfaces is unnecessarily onerous. The Slurm interfaces have to be updated in
   `hpc-libs` first before required changes can be used downstream in the Slurm charms. Contributors
   have to build charms against their own personal forks of `hpc-libs` while they wait for the changes
   they need to be approved and merged.
3. The lack of a consistent versioning scheme makes it difficult to react to security issues. An issue
   in a downstream charm cannot be easily patched because `hpc-libs` does not have a stable API.
4. It's difficult to contribute to `hpc-libs`. Without a clear, defined purpose, contributors don't
   know what should actually be contributed to this library versus other code bases.
5. Discoverability is poor. Users often duplicate functionality already provided by `hpc-libs`
   because `hpc-libs` does not have documentation for how it should be used in HPC charms, and it's
   not obvious that this library provides common building blocks for HPC charms.

Because of the issues presented above:

1. `hpc-libs` should be renamed to `charmed-hpc-libs` to make it more clear that this library provides
   common building blocks for HPC charms.
2. The `hpc-libs`/`charmed-hpc-libs` API should be formally defined to make it more clear what each
   subpackage provides to HPC charm developers.

## Specification

### `charmed-hpc-libs` modules

The sections below describe the modules that will be provided by `charmed-hpc-libs`
and the purpose they serve for HPC charm development.

#### `charmed-hpc-libs.errors`

`charmed-hpc-libs.errors` provides common errors that can be raised within HPC charms.
Example exceptions include:

- `SnapError`: Raised if an error occurs when interfacing with `snap`.
- `SystemdError`: Raised if an error occurs when interfacing with `systemd` or `systemctl`.

This module also provides the `Error` base exception so that custom errors follow the same
template such as exposing the `message` property.

`charmed-hpc-libs.errors` will mirror `hpc-libs.errors` directly.

#### `charmed-hpc-libs.interfaces`

`charmed-hpc-libs.interfaces` provides common building blocks for HPC charm integration interfaces.
Example building blocks include:

- The `Interface` base class.
- Macros for interfacing with Juju secrets.

This module should not contain specific integration interface implementations because
hosting integration interface implementations was discovered to be challenging in
`hpc-libs.interfaces`. For example, having the Slurm integration interfaces hosted within
`hpc-libs.interfaces` has made it unnecessarily difficult to introduce required
changes to the Slurm integration interfaces. A contributor must first open a pull request
against `hpc-libs.interfaces`, and then use their personal fork until the necessary
specific integration interface changes are merged.

`uv` workspaces can be used instead to publish integration interface implementations
directly from the charm repository the interface implementation is associated with:

```shell
# In the `slurm-charms` monorepo.
uv init --lib pkgs/charmed-slurm-interfaces
# Make changes to libraries under `charmed-slurm-interfaces`.
uv build --package charmed-slurm-interfaces
uv publish --token ${PYPI_PUBLISHING_TOKEN}
```

`charmed-hpc-libs.interfaces` will contain the existing building blocks that exist in
`hpc-libs.interfaces`, and the Slurm integration interface implementations will be
relocated to the Slurm charms monorepo. The conditions `block_unless` and `wait_unless`
will be moved to `charmed-hpc-libs.ops`.

#### `charmed-hpc-libs.ops`

`charmed-hpc-libs.ops` will contain common operations logic shared by all HPC charms.

The `charmed-hpc-libs.ops` will be a combination of the existing `hpc-libs.machine`
and `hpc-libs.utils` modules. Both these modules contain HPC charm building blocks
such as the `refresh` decorator and `SystemdManager` and `SnapManager` classes.

`hpc-libs.machine.apt` will be deprecated in favor of `charmlibs-apt`, and `charmed-hpc-libs.ops`
will provide any required macros around `apt` that the HPC charms require.

### Versioning scheme

`charmed-hpc-libs` will follow the semantic versioning scheme `major.minor.patch`.
Following a semantic versioning scheme will enable HPC charms to easily constrain the version
of `charmed-hpc-libs`, and it will fix the current versioning issues with `hpc-libs` where each
charm must pin itself against a specific commit version. Without proper dependency versioning
automation, it is unnecessarily onerous on HPC charm contributors to keep the version of `hpc-libs`
up-to-date in their charms.

A new major version will be released when there is a breaking change in `charmed-hpc-libs` such
as non-backwards compatible changes to the `Interface` base class.

A new minor version will be released when a non-breaking feature is introduced to `charmed-hpc-libs`
that does not break the expected API of downstream HPC charms.

A new patch version will be released when new changes are merged to address confirmed bugs or modify
non-public behavior of the library.

### Publishing

`charmed-hpc-libs` will be published to PyPI automatically using GitHub Actions workflows.
Two workflows are required to automate publishing: one to build and test the package, and
another to publish the package to PyPI when a new GitHub release is created.

A new GitHub release should be created automatically when a new version tag (e.g. `v1.0.0`)
is pushed to the upstream repository. The workflow below handles this by using the
[`softprops/action-gh-release`](https://github.com/softprops/action-gh-release) action.
The `softprops/action-gh-release` action is widely used across Canonical-owned projects such
as `snapcraft` and `rockcraft`:

```yaml
# .github/workflows/release.yaml
name: Release

on:
  push:
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"

jobs:
  release:
    name: Create GitHub release
    runs-on: ubuntu-latest
    permissions:
      contents: write   # required to create a GitHub release
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Create GitHub release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
```

Once the GitHub release is published by the workflow above, a separate publish workflow
is triggered to build and upload the package to PyPI:

```yaml
# .github/workflows/publish.yaml
name: Publish to PyPI

on:
  release:
    types: [published]

jobs:
  build:
    name: Build package
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install uv
        run: sudo snap install astral-uv --classic

      - name: Build package
        run: uv build

      - name: Upload dist artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  publish:
    name: Publish package
    needs: build
    runs-on: ubuntu-latest
    environment: release        # matches the Trusted Publishing environment on PyPI
    permissions:
      id-token: write           # required for OIDC Trusted Publishing
    steps:
      - name: Download dist artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Install uv
        run: sudo snap install astral-uv --classic

      - name: Publish to PyPI
        run: uv publish
```

OIDC Trusted Publishing is the recommended authentication method because it removes the
need to manage long-lived PyPI API tokens as GitHub secrets. To configure it, go to the
PyPI project settings, navigate to *Publishing*, and add a new Trusted Publisher for the
GitHub repository, specifying the workflow filename _publish.yaml_ and the environment
name `release`.

The overall publishing flow for a new `charmed-hpc-libs` release is:

1. Update the version in `pyproject.toml` following the [versioning scheme](#versioning-scheme) above.
2. Push a version tag (e.g. `git tag v1.0.0 && git push origin v1.0.0`).
3. The _release.yaml_ workflow automatically creates a GitHub release with generated notes.
4. The _publish.yaml_ workflow triggers on the new release and publishes the package to PyPI.

## Further information

1. [`hpc-libs` on GitHub](https://github.com/charmed-hpc/hpc-libs)
2. [`uv` workspace documentation](https://docs.astral.sh/uv/concepts/projects/workspaces/#searching-across-multiple-indexes)
3. [`uv` publish documentation](https://docs.astral.sh/uv/guides/package/#updating-your-version)
4. [`charmlibs-apt` on PyPI](https://pypi.org/project/charmlibs-apt/)
