---
title: release policy and notes for Charmed Slurm

---

# Abstract

This spec details the release policy for Charmed Slurm releases. Charmed Slurm contains the Slurm workload manager and related infrastructure components.

# Rationale

For a given product, a consistent release policy is necessary to update users and developers
on the updates and changes to the product. Different products, both Canonical and upstream, have a variety of release policies and cadances, and requires a plan of action to best align Charmed Slurms's releases.

# Specification

## Release Cadence

Current release cadence will be based around feature implementation and upstream Slurm releases, which happen twice a year. Exact timing will vary, and all major releases will be announced.

### Upstream dependencies release cadences

* Slurm - major releases are twice a year
* Ubuntu - minor release twice a year, LTS release every other year

## Release Types

* Major release - new version of Slurm, breaking changes, new base OS
* Minor release - new features, but no breaking changes
* Patch release - bug and security fixes
  * Note that running `juju refresh` is necessary to pull latest security and bug fix updates


### Release channels and branches

* For each major release, each Charmed Slurm charm will have its own Charmhub track using the upstream Slurm version in `YY.MM` notation, e.g. `25.11`
  * With a corresponding GitHub branch

* Edge, the development channel
* Candidate/beta, test the new release before publishing (to catch early bugs)
* Stable
  * No breaking changes should be made to integrations, configuration options, or actions once the stable channel has been established

## Documentation

* User is expected to use the charm version that comes with a specific release
  * Cannot guarantee cross-compatibility of charm versions from different releases
  * If a user requirement necessitates versions that are not from the same release, they should open an issue on Github or contact the team

## Support Timeframe

Each major release will be supported for [TBD]. For major commercial offerings, there will also be a [TBD] LTS (long-term support) release.

## Process for updates

* Docementation for migrating to a new version
* Dedicate a team member to doing upgrade/refresh tests

### Security Patches

* Pushed to edge, release after standard testing process

## Release notes template

```markdown
# Charmed Slurm `YY.MM` release notes

Release date: <date>

## Charm versions

| Charm | Track | Revision |
|-------|-------|----------|
| slurmctld | <track> | <revision> |
| slurmd | <track> | <revision> |
| slurmdbd | <track> | <revision> |
| slurmrestd | <track> | <revision> |
| slurm-configurator | <track> | <revision> |

## What's new

## Requirements and compatibility

## Deprecations and breaking changes

## Fixed issues

## Known issues
```



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
| 2026-06-26 | Draft | [Ashley Cliff](mailto:ashley.cliff@canonical.com) | Added release notes template and updated version notation |
