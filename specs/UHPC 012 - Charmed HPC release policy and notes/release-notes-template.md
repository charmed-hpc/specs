# Charmed HPC Release Notes Template

This document provides the template for Charmed HPC release notes. Copy this template when drafting a new release's notes.

````
# Charmed HPC <major release>.<patch version #> Release Notes

Release date: YYYY-MM-DD

## Summary

Brief overview of this release, including the primary focus (e.g., new Ubuntu base,
new artifact versions, bug fixes, security updates).

## Artifacts in this release

| Artifact | Track | Revision | Notes |
|----------|-------|----------|-------|
| Charmed Slurm | <track> | <revision> | See [Charmed Slurm release notes] |
| cephfs-server-proxy | <track> | <revision> | |
| filesystem-client | <track> | <revision> | |
| lustre-server-proxy | <track> | <revision> | |
| nfs-server-proxy | <track> | <revision> | |
| test-mount-client | <track> | <revision> | |
| lustre-server | <track> | <revision> | |
| sssd-operator | <track> | <revision> | |

## Underlying dependencies

| Dependency | Version | Notes |
|------------|---------|-------|
| charmed-hpc-libs | <version> | |

## What's new

### New features

- Feature description and the artifact(s) it affects.

### Improvements

- Improvement description.

## Requirements and compatibility

### Supported Ubuntu bases

- Ubuntu <version> LTS (Noble Numbat / etc.)

### Juju version

- Minimum Juju version: <version>

## Backwards incompatible changes

- Change description and required user action, if any.

## Deprecated features

- Feature or option that is deprecated, with recommended alternative.

## Known issues

- Issue description and any available workaround.

## Upgrade notes

### Upgrading from <previous major release>.x

Instructions or considerations for upgrading from the previous major release.

### Refreshing charms

```bash
juju refresh <charm-name> --channel <track>/stable
```

## Support lifecycle

| Release | Release date | End of support |
|---------|--------------|----------------|
| <major release>.0 | YYYY-MM-DD | YYYY-MM-DD |

Bug and security fix support is provided for [TBD] months after release.

## References

- [Charmed HPC release policy](link to this spec)
- [Charmed Slurm release notes](link)
````
