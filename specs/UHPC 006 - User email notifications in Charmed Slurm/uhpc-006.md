| Index          | UHPC006                                                          |                   |             |
|:---------------|:-----------------------------------------------------------------|:------------------|:------------|
| Title          | User email notifications in Charmed Slurm                        |                   |             |
| **Type**       | **Author(s)**                                                    | **Status**        | **Created** |
| Implementation | [Dominic Sloan-Murphy](mailto:dominic.sloanmurphy@canonical.com) | Drafting          | 2026-02-23  |
|                | **Reviewer(s)**                                                  | **Status**        | **Date**    |
|                | []()                                                             | Pending Review    |             |


# Abstract

This spec details the implementation of email notification support in Charmed HPC. The Slurm-Mail drop-in replacement for native Slurm email is used to provide richer job statistics and to simplify configuration through Slurm-Mail's built-in SMTP support. A new `smtp` interface is defined for the `slurmctld` charm, allowing integration with the `smtp-integrator` charm to grant access to an external mail server. New handlers are implemented in the `slurmctld` charm for when the `smtp` relation is established, reacting to data being made available in the `smtp` databag, and performing appropriate cleanup when the relation is broken.

# Rationale

User email notifications of job statuses, such as when a job starts or finishes, is a common feature of HPC clusters and is not currently supported by Charmed Slurm. Users are currently required to log in to an interactive session and manually poll the status of their jobs using the `squeue` command. Given that jobs can be queued for days on busy HPC clusters, this is an inefficient use of time and a frustrating user experience.

The Slurm workload manager has built-in mail support, configured through the `MailProg` configuration value set in `slurm.conf`. By default, this points to `/bin/mail` and the Slurm controller calls whichever program with arguments suitable for the default `mail` program. The built-in mail support is limited in the detail it provides per job and can be expanded through the popular [Slurm-Mail](https://github.com/neilmunday/slurm-mail) plugin. Additionally, the `mailutils` package containing `/bin/mail` is in the Universe repository of the Ubuntu archive, bringing into question the level of support which could be offered under Charmed Slurm, while Slurm-Mail uses its own internal SMTP library to send mail.

User mailing is a notoriously difficult feature to configure on HPC clusters but highly desired by users. Providing a simple method of installing and configuring mail support sets us apart from the pack and is valuable to both cluster administrators and users.

# Specification

The `slurmctld` charm is expanded to include mail support through the Slurm-Mail plugin using the [smtp-integrator](https://charmhub.io/smtp-integrator) charm maintained by Canonical IS DevOps. Only integration with an existing mail server is considered \- set up of an SMTP server is out of scope.

## Installation and configuration of Slurm-Mail

As a prerequisite, Slurm-Mail requires Slurm to be using an accounting database. That is, `slurmdbd` must have been deployed and integrated with `slurmctld`. Deploying Slurm-Mail without an accounting database results in mail failures due to errors attempting to gather job data with the `sacct` command.

Various installation and distribution methods for Slurm-Mail were considered. Details follow.

### From deb (chosen)

This method was chosen as the upstream Slurm-Mail project provides deb package generation scripts that can be used as a basis, and as the method integrates well with the PPA Charmed HPC uses for distributing Slurm itself.

The Slurm-Mail project includes a [`build-tools`](https://github.com/neilmunday/slurm-mail/tree/8337cf2/build-tools) directory containing scripts that dynamically generate deb and rpm packages targeting various Linux distributions, including the Ubuntu family. These scripts focus on producing packages for use by the upstream release process and, as such, cannot be used directly to produce a deb package suitable for Charmed HPC. In particular, the scripts employ constants specific to the maintainer: name, email address, as so on.

To address this, a [Slurm-Mail fork containing static Debian build files](https://github.com/charmed-hpc/slurm-mail/tree/9902100/debian) has been created in the Charmed HPC org. The resulting deb package has been added to the [PPA used to distribute Slurm for Charmed HPC](https://launchpad.net/~ubuntu-hpc/+archive/ubuntu/slurm-wlm-25.11) to allow for installation from within the `slurmctld` charm using the existing repository configuration.

### From snap (rejected)

This method was rejected as Slurm-Mail is tightly coupled with the Slurm installation on the host machine, preventing strict confinement, and a classic snap was determined to introduce unnecessary maintenance requirements over basing a package on the upstream deb package generation scripts.

Packaging Slurm-Mail as a strictly confined snap was attempted with the following approaches:

* using the `system-files` interface to expose just the host `sacct` and `scontrol` binaries, including linked libraries. This was unsuccessful due to to libc.so mismatches between the host and snap environments, resulting in the binaries failing to launch.
* bundling the `slurm-client` package from the Ubuntu archive with the snap. This was unsuccessful due to Slurm version mismatches - `slurm-client` is version 23.11 in the Ubuntu archive for Noble and is unable to communicate with Slurm version 25.11, currently served by the Slurm charms.
* bundling multiple `slurm-client` versions and using a wrapper script to detect the host version and switch to an appropriate version. This was unsuccessful due to the presence of Slurm plugins on the host. For example, should the host `slurm.conf` contain:

```
PluginDir=/usr/lib/x86_64-linux-gnu/slurm-wlm
```

This directory would need to be made available to the snap to allow the bundled `sacct` and `scontrol` binaries to communicate with the host successfully. This is not feasible as the `PluginDir` could be modified by the cluster administrator to point to any system path and host paths cannot be made available to a snap dynamically at run time. Additionally, the directory could contain custom plugins essential to the operation of the cluster, hence it is not feasible to bundle the necessary plugins with the snap.

A [`slurm-mail-snap` repository](https://github.com/charmed-hpc/slurm-mail-snap/) has been created with the outputs of this investigation.

### From source (rejected)

This method was rejected as using a package to distribute Slurm-Mail was determined to be more manageable.

The Slurm-Mail utility can be installed from source following [these steps](https://github.com/neilmunday/slurm-mail?tab=readme-ov-file#from-source-as-root):

```shell
git clone https://github.com/neilmunday/slurm-mail
cd slurm-mail
python3 setup.py install
cp etc/logrotate.d/slurm-mail /etc/logrotate.d/
cp etc/cron.d/slurm-mail /etc/cron.d/
install -d -m 700 -o slurm -g slurm /var/log/slurm-mail
```

On an Ubuntu 24.04 machine, this results in slurm-utils being installed under `/usr/local`, rather than `/usr` as expected by the packaged configuration files. Edits should be made to:

* The `/etc/cron.d/slurm-mail` file, replacing the path `/usr/bin/slurm-send-mail` with `/usr/local/bin/slurm-send-mail`.
* The /`etc/slurm/slurm.conf` file, replacing the existing `MailProg` entry with `MailProg=/usr/local/bin/slurm-spool-mail`.

The Slurm-Mail spool directory must be created manually and write permissions granted to the `slurm` user:

```shell
mkdir /var/spool/slurm-mail/
chown slurm: /var/spool/slurm-mail/
chmod 750 /var/spool/slurm-mail/
```

## Integration with `smtp-integrator`

The slurmctld charm uses the library at [https://charmhub.io/smtp-integrator/libraries/smtp](https://charmhub.io/smtp-integrator/libraries/smtp) and integrates with smtp-integrator configured against an SMTP server as in [https://charmhub.io/smtp-integrator/configurations](https://charmhub.io/smtp-integrator/configurations).

A new `mail.py` module has been added which includes functions for installing, configuring, and uninstalling Slurm-Mail. This module is strictly scoped to Slurm-Mail - all charming concerns (unit status changes, logging, error handling) are handled by the main `charm.py` code, which imports the `mail` module and calls its functions from the appropriate hooks. This architecture follows that of the GPU driver detection and installation implemented in the `slurmd` charm.

The `mail` module defines three functions:

* `install`: installs the Slurm-Mail deb package using the `apt` module from `hpc_libs.machine`
* `uninstall`: uninstalls the Slurm-Mail deb package also using `apt`
* `configure`: modifies the `/etc/slurm-mail/slurm.conf` file provided by the deb package to update Slurm-Mail configuration options:
  * `smtpServer`
  * `smtpPort`
  * `smtpUseTls`
  * `smtpUserName`
  * `smtpPassword`
  * `emailFromName`

with values passed to the function via `**kwargs`. The `slurm-mail.conf` file uses standard "ini" syntax so `configparser` from the Python default library is used to perform these modifications.

The slurmctld charm observes events:

* `relation-created`: installs Slurm-Mail package via `mail.install`
* `smtp_data_available`, a custom event provided by the `smtp-integrator` library: configures Slurm-Mail by retrieving the SMTP server details, credentials and other configuration from the application relation databag then passing them to `mail.configure`
* `relation-broken`: uninstalls Slurm-Mail via `mail.uninstall` and removes `MailProg` entry from `slurm.conf`.

### `relation-broken` HA workaround

In a `slurmctld` HA setup, scaling down the application causes a `relation-broken` event to be observed by the departing unit. If the departing unit is the `slurmctld` leader, this would result in the `MailProg` entry being erroneously removed from `slurm.conf` while the `smtp` integration was still in place for the application.

To address this, the charm now also observes the `relation-departed` event for the `smtp` integration, in which it uses `StoredState` to set a flag if the current unit is the departing unit. The `relation-broken` hook is then skipped if this flag is `True`. The use of `StoredState` is necessary as it is not possible to determine from the `relation-broken` hook whether the current unit is departing. Further discussion on this issue can be found at https://bugs.launchpad.net/juju/+bug/1979811

# Future work

The Slurm-Mail addon supports customization of its default templates to provide tailored emails from a site. Investigations into providing this functionality via the Slurm charms would be a logical focus for future work.
