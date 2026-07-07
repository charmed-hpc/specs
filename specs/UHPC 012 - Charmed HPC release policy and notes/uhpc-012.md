---
index: UHPC012
title: Release policy for Charmed HPC
---

# Release policy for Charmed HPC

## Abstract

This spec details the release policy for Charmed HPC.

## Rationale

A consistent release policy is necessary to keep our community aware of upcoming major changes, bug fixes, and security updates, while ensuring that the community has some expected degree of stability.

Since Charmed HPC is a composition of multiple charms and artifacts, there must be a well-defined release policy that developers and users can reference to know when they can expect new features, bug fixes, security updates, and compatibility guarantees across the set of charms.

## Specification

## Artifacts

Charmed HPC artifacts:

<!-- Update this list as the Charmed HPC portfolio evolves -->

- Charmed Slurm (see [UHPC 003](../UHPC%20003%20-%20Release%20policy%20and%20notes%20for%20Charmed%20Slurm/uhpc-003.md))
- filesystem-charms:
  - cephfs-server-proxy
  - filesystem-client
  - lustre-server-proxy
  - nfs-server-proxy
  - test-mount-client
- lustre-server
- sssd-operator

### Versioning scheme

Version format: `<major release>.<patch version #>`. Example Charmed HPC release numbers:

- Initial release: "1.0"
- Bugfix release: "1.1"
- Security update: "1.2"

* Major release - new features, new Ubuntu base version, updated artifact versions
* Minor release - bug and security fixes only
  * Note that running `juju refresh` is necessary to pull the latest security and bug fix updates

## Release cadence

<!-- Charmed HPC releases will follow a regular cadence aligned with the Ubuntu LTS release cycle. Each Charmed HPC release will target the current Ubuntu LTS release as its primary base. -->

Individual components within Charmed HPC (e.g. Charmed Slurm) may follow their own upstream-driven release cadences as defined in their respective specs. Since Charmed HPC is a set of charms rather than a single deployable artifact, a Charmed HPC release defines a tested, compatible set of charm versions that are verified to work together.

### Release channels and branches

Since Charmed HPC is a set of charms rather than a single charm, release channels apply to each constituent charm individually.

* Each track has a corresponding GitHub branch
* Each track on Charmhub provides four channels:
   * Edge, the development channel
   * Beta
   * Candidate, to test the new release before publishing
   * Stable
     * No breaking changes will be made to integrations, configuration options, or actions in a stable channel of a charm

### Release cycle and feature freezes

A Charmed HPC release progresses through three phases between freezes:

* **Soft freeze** — all new feature work stops. Each charm is published to its **Beta** channel, where beta-level testing is performed.
* **Hard freeze** — each charm is promoted from Beta to **Candidate**, where candidate-level testing is performed.
* **Release day** — all charms are promoted from Candidate to **Stable**.

Given the variety of charms, the Candidate/Stable for a given charm may be the same as for the prior release. Freeze dates are set during cycle planning.

```mermaid
%%{init: {'themeVariables': {'critBorderColor': '#ff0000', 'critBkgColor': '#ff0000'}}}%%
gantt
  title Charmed HPC Release steps
  dateFormat YYYY-MM
  todayMarker off
  section Slurm
        Slurm 26.05 released by SchedMD                         :milestone, a1, 2026-05, 0d
        Charmed Slurm 26.05 release                             :milestone, 2026-06, 0d
        Slurm 26.11 released by SchedMD                         :milestone, a2, 2026-11, 0d
        Charmed Slurm 26.11 release                             :milestone, 2026-12, 0d
  section Charmed HPC X dev
        filesystem-charms dev                                   :f1, 2026-05, until v1
        filesystem-charms Stable X                              :milestone, 2026-10, 0d
        sssd-operator dev                                       :s1, 2026-03, 2026-06
        sssd-operator Stable X                                  :milestone, 2026-10, 0d
  section X Freeze
        'Soft freeze'                                           :crit, milestone, v1, 2026-08, 0d
        Beta                                                    :p1, after v1, until v2
        'Hard freeze'                                           :crit, milestone, v2, 2026-09, 0d
        Candidate                                               :c1, after v2, until r1
        Release X.0                                             :crit, milestone, r1, 2026-10, 0d
  section Charmed HPC Y dev
        filesystem-charms dev                                   :f2, 2026-11, until v3
        filesystem-charms Stable Y                              :milestone, 2027-05, 0d
        sssd-operator dev                                       :s3, 2026-10, 2027-01
        sssd-operator Stable Y                                  :milestone, 2027-05, 0d
  section Y Freeze
        'Soft freeze'                                           :crit, milestone, v3, 2027-03, 0d
        Beta                                                    :p2, after v3, until v4
        'Hard freeze'                                           :crit, milestone, v4, 2027-04, 0d
        Candidate                                               :c2, after v4, until r2
        Release Y.0                                             :crit, milestone, r2, 2027-05, 0d
  section Ubuntu
        Resolute Raccoon 26.04 LTS                              :2026-04, 2027-06
        Noble Numbat 24.04 LTS                                  :2026-02, 2026-04
```




## Support life-cycle

Outside exceptional circumstances, each Charmed HPC release will receive bug and security fix support for [TBD].

<!-- Components that track upstream projects (e.g. Charmed Slurm tracking SchedMD's Slurm) will follow the support life-cycle defined in their respective release policies. -->





## Documentation

Warnings/limitations that will be included in the published documentation alongside the release notes:

* Due to potential breaking changes between major releases, Charmed HPC cannot guarantee backwards compatibility with previous major versions
  * If a user requirement necessitates artifact versions that are not from a single release, they should open an issue on GitHub (or Discourse) and work with the team

#### Release Notes Template

See the [Release Notes Template](release-notes-template.md) for the template used when drafting a new release's notes.

## Release notes sections

General release notes sections for Charmed HPC:

* Release summary
* Artifacts and versions included in the release
* What's new (features and improvements)
* Requirements and compatibility (Ubuntu base, Juju version, Charmhub tracks)
* Backwards incompatible changes
* Deprecated features
* Known issues
* Upgrade notes (including `juju refresh` instructions)
* Support lifecycle

## References

* [UHPC 003 - Release policy for Charmed Slurm](../UHPC%20003%20-%20Release%20policy%20and%20notes%20for%20Charmed%20Slurm/uhpc-003.md)
* [Canonical product release cycles](https://ubuntu.com/about/release-cycle#ubuntu)
