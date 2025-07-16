---
title: release policy and notes for Charmed Slurm

---

# Abstract

This spec details the release policy and release notes format for Charmed Slurm releases. Charmed Slurm contains the Slurm workload manager and related infrastructure components such as observability and scaling.

# Rationale

For a given product, a consistent release policy and release notes format is necessary to update users and developers
on the updates and changes to the product. Different products, both Canonical and upstream, have a variety of release
policies and cadances, and requires a plan of action to best align Charmed HPC's releases.

# Specification

## Release Cadence

Current release cadence will be based around feature implementation and upstream Slurm releases, which happen twice a year. Exact timing will vary, and all major releases will be announced.


### Cadence considerations

* Don't tie release to one specific upstream, or Ubuntu
* Not currently in a position to commit to LTS releases
* Potentially a yearly release - 'Charmed HPC 2025.1'
* Focus on how the charms are released
  * When is 'stable' reached for a given charm
  * Core features early in cycle to have an idea by mid-cycle for what's needed to finish cycle strong


## Upstream dependencies and release cadences

* Slurm - major releases are twice a year
* Ubuntu - minor release twice a year, LTS release every other year

## Release Types

* Major release - new version of Slurm, breaking changes, new base OS
* Minor release - new features, but no breaking changes
* Patch release - bug and security fixes
  * Note that running `juju refresh` is necessary to pull latest security and bug fix updates


### Release channels

* Edge, the development channel
* Candidate, test the new release before publishing (to catch early bugs)
* Stable

* Each release has its own track
  * Corresponding GitHub branch and each of the three channels
  * Corrensponding commit hash is included in the charm channel release
* No breaking changes should be made to integrations, configuration options, or actions in a stable track of a charm
  * To introduce breaking changes, a new track should be created

* 'stable' vs 'edge' for slurm charms
  * how to handle when one charm updates but not others?

## Documentation

* User is expected to use the charm version that comes with a specific release
  * Cannot guarantee 'Franken'-charm collections
  * If a user requirement necessitates versions that are not from the same release, they should open an issue on Github
(or Support Discussion) and contact the team

## Support Timeframe

Each major release will be supported for 18 months. For major commercial offerings, there will also be a 10 LTS (long-term support) release.

## Process for updates

* Docementation for migrating to a new version
* Dedicate a team member to doing upgrade/refresh tests
* Ubuntu version updates will likely be more costly than service version

### Security Patches

* Pushed to edge, release after standard testing process

## Supported Artifacts

<!-- List of supported artifacts with source code links and issue tracker refs
 -->

### Release Structure

<!-- Table of corresponding pieces and commit/channel/tag/etc for each:

* E.g. stable release of the Slurm charms are published to 25.04/stable on Charmhub
  * ^ Mirror this bit in the release notes. E.g. how do you actually pull Charmed Slurm 25.04
* PPA to get supported Slurm packages.
* Any Terraform plans for reference deployment (maybe, homies could just pull from the correct channel) -->

## Examples

### Anbox Cloud

[Anbox Release Page](https://documentation.ubuntu.com/anbox-cloud/en/latest/reference/release-notes/release-notes/)

* From home page, Roadmap has its own left-side TOC below contribute
* After selection: Roadmap appears only within references in left-side TOC
  * Preferred style, need to figure out sphinx set up
* Has recent releases and upcoming release roadmap
* Has defined release cycle and defined support policy

### Kubernetes

[Kubernetes Release Page](https://ubuntu.com/kubernetes/docs/release-notes)

* Release notes within refs section
  * Has a 'what's new' section
* No Roadmap

### MaaS

[Maas Release Page](https://maas.io/docs/reference-release-notes-maas-3-5)

* Each major release has a tab in Refs

# Release notes sections

General Release Notes sections:
* Long-term support or not
  * Length of 'long-term'
* Requirements and compatibility
* What's new
* Backwards incompatible changes
* Deprecated features


# References
* [Release Notes Format](https://docs.google.com/document/d/1L-FxU2Si7Mt6TqnTk_CZYPezAajsBKiGDNP65Pf688s/edit?tab=t.0#heading=h.g4gdk7o1d9xn)
* [Documentation Release Notes landing page Format](https://docs.google.com/document/d/187hrGJd-l9WkUarEqw7FLOu_X6k849xaiDp_T9HHDHI/edit?tab=t.0#heading=h.y7atuj5xt6qt)
* [Canonical product release cycles](https://ubuntu.com/about/release-cycle#ubuntu)
* [Kubeflow publish action for charms](https://github.com/canonical/charmed-kubeflow-workflows/blob/main/.github/workflows/_publish.yaml)
* [Other Kubeflow actions](https://github.com/canonical/charmed-kubeflow-workflows/tree/main/.github/workflows)

# Spec History and Changelog
| Date    | Status  | Author(s)           | Comment     |
|:--------|:--------|:--------------------|:------------|
| 2025-07-09 | Braindump | [Ashley Cliff](mailto:ashley.cliff@canonical.com) | Initial braindump |
| 2025-07-15 | Braindump | [Ashley Cliff](mailto:ashley.cliff@canonical.com) | Initial braindump |
