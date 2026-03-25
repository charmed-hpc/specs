---
index: UHPC009
title: Distributing Slurm interfaces as Python packages
---

# Distributing Slurm interfaces as Python packages

## Abstract

This specification proposes distributing the Slurm integration interface implementations
as individual Python packages hosted in the `slurm-charms` monorepo. This specification
is a follow-on to specification [UHPC008](../UHPC%20008%20-%20%60charmed-hpc-libs%60%20for%20HPC%20charm%20development/uhpc008.md).

## Rationale

As part of the `hpc-libs` refinement work, it was decided to extricate the Slurm integration
interface implementations from the `interfaces` submodule. This decision was made because:

1. It doesn't make sense to distribute private Slurm interfaces like `slurmd` and `slurmdbd`
   to all charms that require one of the Slurm interfaces such as `slurm-oci-runtime`.
   For example, the Apptainer operator only needs the `slurm-oci-runtime` interface; it
   does not need all the other Slurm interfaces installed as well.
2. It's unnecessarily onerous to make required interface changes when the Slurm charms
   need to share additional data over integration databags. A contributor must first
   open a pull request against `hpc-libs`, and then pull that fork into the Slurm charms
   to test their interface changes. This process introduces a lot of overhead for minimal
   utility since the majority of these interfaces are only used by the Slurm charms and
   are not intended for public consumption.

During the same discussion, it was also determined that only certain Slurm interfaces should
be published. Interfaces such as:

- `sackd`
- `slurmd`
- `slurmdbd`
- `slurmrestd`

are private interfaces only used in the Slurm charms monorepo, whereas the `slurm-oci-runtime`
and `slurmctld` interfaces are publicly consumed by the Apptainer operator. A mechanism is required
to separate private, internal interface implementations from public interface implementations,
and automate publishing the public interface implementations to a package index like PyPI.

## Specification

The sections below outline how the Slurm integration interface implementations will be vendored
in the Slurm charms monorepo, and how public interface implementations will be automatically
published when there is a version update.

### Organizing public and private integration interface implementations

Interface implementations can be organized similar to the [standard Go project layout](https://github.com/golang-standards/project-layout)
where public interfaces are placed in the `pkg` directory, and private interfaces are placed
in the `internal` directory. Interface implementations under `pkg` would be published to a
public package index like PyPI while interface implementations under `internal` would not.
Each interface implementation would be its own individual `uv` workspace.

`pkg` will contain the `slurmctld` and `slurm-oci-runtime` interface implementations. `slurmctld`,
known as `common` in `hpc-libs` currently, is a public interface since it serves as a building block
for all Slurm interfaces such as `slurm-oci-runtime`. `slurm-oci-runtime` is a public interface
because it is used by charms such as the Apptainer operator to provide OCI runtime information to Slurm.

```shell
pkg/
├── slurmctld
└── slurm-oci-runtime
```

`internal` will contain the `sackd`, `slurmd`, `slurmdbd`, and `slurmrestd` interface implementations
since these interface implementations are only used within the Slurm charms monorepo:

```shell
internal/
├── sackd
├── slurmd
├── slurmdbd
└── slurmrestd
```

Support for the `internal` directory will need to be added to the _repository.py_ utility script.

### Interface implementations as `uv` workspaces

Having each interface implementation as its own `uv` workspace is beneficial because `uv`'s
tooling can be used to automate the building and publishing of each interface implementation. 
For example, to build and publish the `slurmctld` and `slurm-oci-runtime` implementations:

```shell
uv build --package pkg/slurmctld
uv build --package pkg/slurm-oci-runtime
uv publish --token ${PYPI_PUBLISH_TOKEN}
```

### Distributing public Slurm interface implementations

Public interface implementations under `pkg` will be automatically published to PyPI
when their version in `pyproject.toml` is bumped and the pull request introducing that
bump is merged into `main`. The following GitHub Actions workflow achieves this:

```yaml
name: Publish packages

on:
  push:
    branches:
      - main
    paths:
      - "pkg/*/pyproject.toml"

jobs:
  get-packages:
    name: Get changed packages
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.changed.outputs.packages }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Detect version bumps in pkg/
        id: changed
        run: |
          packages=$(
            git diff HEAD^ HEAD --name-only \
              | grep '^pkg/[^/]*/pyproject\.toml$' \
              | while read -r f; do
                  dir=$(dirname "$f")
                  old=$(git show HEAD^:"$f" 2>/dev/null | grep '^version' | head -1 | awk -F'"' '{print $2}')
                  new=$(git show HEAD:"$f"  | grep '^version' | head -1 | awk -F'"' '{print $2}')
                  if [ "$old" != "$new" ]; then
                    echo "$dir"
                  fi
                done \
              | jq -R . | jq -sc .
          )
          echo "packages=$packages" >> "$GITHUB_OUTPUT"

  publish:
    name: Publish ${{ matrix.package }}
    needs: get-packages
    if: needs.get-packages.outputs.packages != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package: ${{ fromJson(needs.get-packages.outputs.packages) }}
    permissions:
      id-token: write
    environment:
      name: pypi
      url: https://pypi.org/p/${{ matrix.package }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up uv
        uses: astral-sh/setup-uv@v5

      - name: Build ${{ matrix.package }}
        run: uv build --package ${{ matrix.package }}

      - name: Publish ${{ matrix.package }} to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
```

The workflow is triggered on every push to `main` that touches any `pyproject.toml`
file under `pkg/`. The `get-packages` job compares the `version` field in each
changed `pyproject.toml` against its value in the previous commit; only packages
whose version actually increased are forwarded to the `publish` job. Each qualifying
package is built with `uv build` and then uploaded to PyPI using
[`pypa/gh-action-pypi-publish`](https://github.com/pypa/gh-action-pypi-publish), which
uses [Trusted Publishers](https://docs.pypi.org/trusted-publishers/) (OIDC) so that no
long-lived PyPI API token needs to be stored as a repository secret.

### Common naming scheme for Charmed Slurm-related Python packages published to PyPI

A common naming scheme must be used for Charmed Slurm-related Python packages that are published to
PyPI to prevent collisions with similarly named packages and demonstrate that these specific
packages are used with Charmed Slurm. The naming scheme for packages published to PyPI will be:

```text
charmed-<application>-<interface-name>-interface
```

For example, the `slurm-oci-runtime` interface will be published under the name:

```text
charmed-slurm-oci-runtime-interface
```

> [!WARNING]
> Exceptions to the naming scheme will be made to avoid redundancy. For example,
> `slurm` is already in the package name `charmed-slurm-oci-runtime-interface`, so it
> would be redundant to include the full interface name `slurm-oci-runtime` exactly.

The package name will be set in the _pyproject.toml_ file:

```toml
[project]
name = "charmed-slurm-oci-runtime-interface"
version = "0.1.0"
description = "Integration interface implementation for the `slurm_oci_runtime` interface."
readme = "README.md"
authors = [{ name = "Canonical HPC", email = "hpc-ubuntu-group@canonical.com" }]
requires-python = ">=3.12"
dependencies = []
```

### Common naming scheme for internal packages

Naming collisions are not a concern for internal packages since these packages will not be
published to a public package index; however, internal packages should still use the same
naming scheme as public packages for charm developer-friendliness. For example, the 
`slurmd` interface implementation package should be named:

```text
charmed-slurm-slurmd-interface
```

### Organizing imports

All public classes and functions must be declared in `__all__` in the top-level *\_\_init\_\_.py* file
of an interface implementation package. For example, the top-level *\_\_init\_\_.py* file for
`charmed-slurm-oci-runtime-interface` would be:

```python
# __init__.py

__all__ = [
    "OCIRuntimeData",
    "OCIRuntimeDisconnectedEvent",
    "OCIRuntimeReadyEvent",
    "OCIRuntimeProvider",
    "OCIRuntimeRequirer",
]
```

The classes from the `charmed-slurm-oci-runtime-interface` would then be imported like so
in the `apptainer` and `slurmctld` charms:

```python
from charmed_slurm_oci_runtime_interface import OCIRuntimeData, ...
```

## Further information

1. [Standard Go project layout](https://github.com/golang-standards/project-layout)
2. [Apptainer operator's integrations on Charmhub](https://charmhub.io/apptainer/integrations)
3. [`uv` workspace documentation](https://docs.astral.sh/uv/concepts/projects/workspaces/)
