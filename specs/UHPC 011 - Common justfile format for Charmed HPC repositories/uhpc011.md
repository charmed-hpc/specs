---
index: UHPC011
title: Common justfile format for Charmed HPC repositories
---

# Common justfile format for Charmed HPC repositories

## Abstract

This spec defines a common `justfile` format for all Charmed HPC repositories, allowing for a
consistent contributor experience and facilitating the sharing of GitHub workflows. Only the set of
common recipes is defined; how the recipes are implemented is left to the individual repository to
determine.

## Rationale

Repositories in the Charmed HPC organization make use of the [`just`](https://github.com/casey/just)
command runner for operations such as code formatting, linting, and running test suites. At present,
there is no standard for the recipes defined in the `justfile`, causing inconsistencies across
repositories. For example, the command for running the unit test suite may be `just unit` in one
repository and `just repo unit` in another.

This results in a disjointed experience for contributors and creates difficulties for CI/CD. GitHub
workflows must be customized per repository to account for differences in `justfile`.

A common `justfile` format is proposed here to address these issues.

## Specification

Every Charmed HPC `justfile` must run with `just` version 1.48.1. Use of unstable features is
permitted.

Use of [groups](https://just.systems/man/en/groups.html) is prohibited. All recipes must be
ungrouped.

### Prerequisites

The single common prerequisite that must be installed on the host before a Charmed HPC `justfile`
can be used is `just` itself. The installation method is not prescribed. It is expected the typical
methods will be through the distribution package manager, through the Snap Store, or through an
external GitHub action within a workflow.

Repositories may delegate to tooling, such as `repository.py`, [`uv`](https://github.com/astral-sh/uv)
and [`OpenTofu`](https://github.com/opentofu/opentofu), in the implementation of recipes. In which
case, it is recommended that the Just [`require()`](https://just.systems/man/en/functions.html#executables)
function be used in the `justfile`, for example `require("uv")`, to ensure these tools are present.
As with `just`, the installation method for these tools is not prescribed.

### Required recipes

All Charmed HPC repositories must provide a `justfile` which implements the recipes defined in this
section. The implementation may be a "no-op", if not appropriate for a repository, but the recipe
must still be present.

Recipes:

```
# Performed if no recipes are specified. Must have this implementation
[private]
default:
    @just help

# Show available recipes
help:

# Prepare the local environment
setup:

# Clean project directory
clean:

# Apply static checks
# Implementation expected to be composed of optional/recommended recipes: fmt, lint, and typecheck.
check:

# Run specified target test suites, or all test suites if none specified
test *targets:
```

### Recommended recipes

Repositories may implement the following recipes as required. If implemented, they must use the
recipe names given below:

```
# Apply formatting standards
fmt:

# Check against style standards
lint:

# Perform type checking
typecheck:

# Run all test suites
# Implementation expected to be composed of optional/recommended recipes: unit, integration, and
# other test types.
test-all:

# Run unit tests for specified artifacts, or all artifacts if none specified
unit *args:

# Run integration tests for specified artifacts, or all artifacts if none specified
integration *args:

# Build specified artifacts, or all artifacts if none specified
build *args:

# Regenerate uv.lock
lock:

# Create a uv development environment
env:

# Upgrade uv.lock with the latest dependencies
upgrade:
```

### Suggested implementation of `test` recipe

Given the relative complexity of the required `test` recipe, a suggested implementation is provided
here. However, this is not enforced and the recipe can be implemented as desired so long as the
specified semantics are followed.

Note, this implementation requires an environment where `bash` is available.

```
test *targets:
    #!/usr/bin/env bash
    if [ "{{targets}}" = "" ]; then
        just test-all
        exit 0
    fi

    for target in {{targets}}; do
        if just --show $target > /dev/null 2>&1; then
            echo "Running $target tests..."
            just $target
        else
            echo "$target tests not found, skipping."
            exit 1
        fi
    done

test-all: unit functional integration

unit:
  @echo "Running unit tests"

functional:
  @echo "Running functional tests"

integration:
  @echo "Running integration tests"
```

### Example justfile - slurm-charms repository

```
# Copyright 2026 Canonical Ltd.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

uv := require("uv")

export PY_COLORS := "1"
export PYTHONBREAKPOINT := "pdb.set_trace"

uv_run := "uv run --frozen --extra dev"

[private]
default:
    @just help

# Prepare the local environment
setup: env

# Clean project directory
clean:
    {{uv_run}} repository.py clean

# Apply static checks
check: fmt lint typecheck

# Run tests for specified targets, or all tests if none specified
test *targets:
    #!/usr/bin/env bash
    if [ "{{targets}}" = "" ]; then
        just test-all
        exit 0
    fi

    for target in {{targets}}; do
        if just --show $target > /dev/null 2>&1; then
            echo "Running $target tests..."
            just $target
        else
            echo "$target tests not found, skipping."
            exit 1
        fi
    done

# Run all test suites
test-all: unit integration

# Run unit tests
unit *args: lock
    {{uv_run}} repository.py unit {{args}}

# Run integration tests
integration *args: lock
    {{uv_run}} repository.py integration {{args}}

# Regenerate uv.lock
lock:
    uv lock

# Create a uv development environment
env: lock
    uv sync --extra dev

# Upgrade uv.lock with the latest dependencies
upgrade:
    uv lock --upgrade

# Apply formatting standards
fmt: lock
    {{uv_run}} repository.py fmt

# Check files against style standards
lint: lock
    {{uv_run}} repository.py lint

# Perform type checking
typecheck:
    {{uv_run}} repository.py typecheck

# Run action on monorepo. For a full list of actions, run `just repo`
repo *args: lock
    {{uv_run}} repository.py {{args}}

# Show available recipes
help:
    @just --list --unsorted
```

### Example justfile - charmed-hpc-terraform repository

```
# Copyright 2026 Canonical Ltd.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.\
# Necessary to use `||` logical operator.

set unstable := true

project_dir := justfile_directory()
modules_dir := project_dir / "modules"
default_module_list := shell("ls -d -- $1/*", modules_dir)
uv := require("tofu")

[private]
default:
    @just help

# Prepare the local environment
setup: init

# Clean project directory
clean:
    find . -name .terraform -type d | xargs rm -rf
    find . -name .terraform.lock.hcl -type f | xargs rm -rf
    find . -name "terraform.tfstate*" -type f | xargs rm -rf

# Apply static checks
check: fmt validate

# Run tests for specified targets, or all tests if none specified
test *targets:
    #!/usr/bin/env bash
    if [ "{{targets}}" = "" ]; then
        just test-all
        exit 0
    fi

    for target in {{targets}}; do
        if just --show $target > /dev/null 2>&1; then
            echo "Running $target tests..."
            just $target
        else
            echo "$target tests not found, skipping."
            exit 1
        fi
    done

# Run all test suites
test-all: integration

# Run integration tests
integration *args:
    tofu apply -auto-approve

# Apply formatting standards to project
fmt:
    just --fmt --unstable
    tofu fmt -recursive

# Initialize Terraform modules
init *modules:
    #!/usr/bin/env bash
    set -euxo pipefail
    modules=({{ prepend(modules_dir, modules) || default_module_list }})
    for module in ${modules}; do
        tofu -chdir=${module} init
    done

# Validate Terraform modules
validate *modules: (init modules)
    #!/usr/bin/env bash
    set -euxo pipefail
    modules=({{ prepend(modules_dir, modules) || default_module_list }})
    for module in ${modules}; do
        tofu -chdir=${module} fmt -check
        tofu -chdir=${module} validate
    done

# Show available recipes
help:
    @just --list --unsorted
```

## Evaluated alternatives

For the required `setup` recipe, `init` and `env` were considered as alternative names because of
their existing usage in current `justfiles`, but `setup` was chosen instead because the terms `env`
and `init` are tightly integrated with `uv` and `terraform` toolchains, respectively, whereas
`setup` is a more toolchain-agnostic recipe name.

## Further work

To be determined:
* A method of enforcing `just` version requirements for the `justfile`. [Issue \#2290](https://github.com/casey/just/issues/2290) is open regarding this.
