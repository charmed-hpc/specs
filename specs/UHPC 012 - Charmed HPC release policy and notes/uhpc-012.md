---
index: UHPC012
title: Release policy for Charmed HPC
---

# Release policy for Charmed HPC

## Abstract

This spec details the release policy for Charmed HPC.

## Rationale

A consistent release policy is necessary to keep our community aware of upcoming major changes, bug fixes, and security updates, while ensuring that the community has some expected degree of stability.

Charmed HPC components:

<!-- Update this list as the Charmed HPC portfolio evolves -->

- Charmed Slurm (see [UHPC 003](../UHPC%20003%20-%20Release%20policy%20and%20notes%20for%20Charmed%20Slurm/uhpc-003.md))
- Additional workload manager charms
- HPC shared libraries (`charmed-hpc-libs`)
- Supporting infrastructure charms

Since Charmed HPC is a composition of multiple charms and artifacts spanning different workload managers and supporting services, there must be a well-defined release policy that developers and users can reference to know when they can expect new features, bug fixes, security updates, and compatibility guarantees across the set of charms.

## Specification

### Release cadence

Charmed HPC releases will follow a regular cadence aligned with the Ubuntu LTS release cycle. Each Charmed HPC release will target the current Ubuntu LTS release as its primary base.

Individual components within Charmed HPC (e.g. Charmed Slurm) may follow their own upstream-driven release cadences as defined in their respective specs. Since Charmed HPC is a set of charms rather than a single deployable artifact, a Charmed HPC release defines a tested, compatible set of charm versions that are verified to work together.

### Support life-cycle

Outside exceptional circumstances, each Charmed HPC release will receive bug and security fix support for 1 year.

Components that track upstream projects (e.g. Charmed Slurm tracking SchedMD's Slurm) will follow the support life-cycle defined in their respective release policies.

### Versioning scheme

Version format: `<year>.<patch version #>`. Example Charmed HPC release numbers:

- Initial release: "2026.0"
- Bugfix release: "2026.1"
- Security update: "2026.2"

* Major release - new features, new Ubuntu base version, updated component versions
* Minor release - bug and security fixes only
  * Note that running `juju refresh` is necessary to pull the latest security and bug fix updates

#### Release channels

Since Charmed HPC is a set of charms rather than a single charm, release channels apply to each constituent charm individually:

* Each major release has its own Charmhub track per charm, e.g. "2026"
* Each track has a corresponding GitHub branch, e.g. "2026"
* Each track on Charmhub provides three channels:
   * Edge, the development channel
   * Candidate, to test the new release before publishing
   * Stable
     * No breaking changes will be made to integrations, configuration options, or actions in a stable channel of a charm

### Documentation

Warnings/limitations that will be included in the published documentation alongside the release notes:

* Due to potential breaking changes between major releases, Charmed HPC cannot guarantee backwards compatibility with previous major versions
  * If a user requirement necessitates versions that are not from corresponding releases, they should open an issue on GitHub (or Support Discussion) and contact the team

#### Release Notes Template

````markdown
---
myst:
  html_meta:
    description: Release notes for Charmed HPC YYYY.Z, including highlights about new features, bug fixes, and other updates.
---

(ref-release-notes-charmed-hpc-YYYY.Z)=
# Charmed HPC YYYY.Z release notes

% NOTE: This is a template for Charmed HPC release notes. To use it, find
% and replace `YYYY` with the full release year (e.g. 2026), and `Z` with the
% patch version number (e.g. 0). Follow any instructions in comments, then
% delete the comment.

% Remove the comment prefix (%) from the appropriate sentence below, and delete
% the other.
% Charmed HPC YYYY.Z receives bug and security fix support for 1 year.
% Charmed HPC YYYY.Z is a patch release containing bug and security fixes only.

% Optionally add any other introductory notes here. Fill out the content of the
% sections below, and delete any sections that are not relevant to this release.

(ref-release-notes-charmed-hpc-YYYY.Z-requirements)=
## Requirements and compatibility

% List the requirements and compatibility information for this release, e.g.
% supported Ubuntu bases, Juju versions, and any other dependencies.

- Ubuntu base: <!-- e.g. Noble Numbat 24.04 LTS -->
- Juju version: <!-- e.g. 3.6+ -->

(ref-release-notes-charmed-hpc-YYYY.Z-highlights)=
## What's new

This section highlights new and improved features in this release.

### Feature short description placeholder (replace this text)

% Dedicate a separate ### section for each new and improved feature highlight
% description.

(ref-release-notes-charmed-hpc-YYYY.Z-bugfixes)=
## Bug fixes

The following bug fixes are included in this release.

% List of links to resolved bug fix issues from GitHub.

(ref-release-notes-charmed-hpc-YYYY.Z-incompatible)=
## Backwards-incompatible changes

% List any changes that are not backwards compatible below.

### Incompatible change short description placeholder (replace this text)

% Dedicate a separate ### section to describe each change that is not backwards
% compatible.

(ref-release-notes-charmed-hpc-YYYY.Z-deprecated)=
## Deprecated features

% List any features that have been deprecated or removed in this release.

### Deprecated feature short description placeholder (replace this text)

% Dedicate a separate ### section to describe each deprecated feature.

(ref-release-notes-charmed-hpc-YYYY.Z-components)=
## Component versions

% Update the table below with the component versions included in this release.

| Component      | Version |
|----------------|---------|
| Charmed Slurm  |         |

(ref-release-notes-charmed-hpc-YYYY.Z-channels)=
## Release channels

This release is available on Charmhub under the `YYYY` track:

- **Stable:** `juju deploy <charm> --channel YYYY/stable`
- **Candidate:** `juju deploy <charm> --channel YYYY/candidate`
- **Edge:** `juju deploy <charm> --channel YYYY/edge`

To upgrade an existing deployment, run `juju refresh` to pull the latest revision from the stable channel.
````

## Release notes sections

General release notes sections:

* Long-term support or not
  * Length of 'long-term'
* Requirements and compatibility
* What's new
* Backwards incompatible changes
* Deprecated features
* Component versions

## References

* [UHPC 003 - Release policy for Charmed Slurm](../UHPC%20003%20-%20Release%20policy%20and%20notes%20for%20Charmed%20Slurm/uhpc-003.md)
* [Canonical product release cycles](https://ubuntu.com/about/release-cycle#ubuntu)
