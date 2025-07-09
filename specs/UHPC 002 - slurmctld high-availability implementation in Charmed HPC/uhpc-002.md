---
title: slurmctld high-availability implementation in Charmed HPC

---

# Abstract

This spec details the implementation of high availability (HA) in the `slurmctld` charmed operator used for the Slurm controller in a Charmed HPC cluster deployment. Support for the `filesystem-client` charm has been added to allow for storage of Slurm controller checkpoint data, also known as the StateSaveLocation data, in a shared file system mounted by all controller instances, a prerequisite for the native `slurmctld` active-passive HA setup. Slurm configuration data is also made available via this file system. Related charms: `sackd, slurmd, slurmrestd,` and `slurmdbd` have been updated to support multiple controller addresses. In addition to the final implementation. details of the other approaches considered, including proofs of concept developed, are provided.

# Rationale

From the [slurmctld documentation](https://slurm.schedmd.com/slurmctld.html):

*“**slurmctld** is the central management daemon of Slurm. It monitors all other Slurm daemons and resources, accepts work (jobs), and allocates resources to those jobs. Given the critical functionality of **slurmctld**, there may be a backup server to assume these functions in the event that the primary server fails.”*

At present, the `slurmctld` charm in Charmed HPC does not support backup instances, allowing only a single `slurmctld` controller unit to be operational at any one time. This creates a single point of failure for a cluster deployment. To address this, the charms must be extended with support for `slurmctld` HA.

# Specification

Multiple `slurmctld` instances are supported natively using the [HA functionality within Slurm](https://slurm.schedmd.com/quickstart_admin.html#HA). The architecture is active-passive: a primary is configured with one or more backup controllers on standby. Scaling up the number of controllers does not allow for more requests to be served. A backup takes over if the primary controller fails, that is, if the controller does not respond to a request within 120 seconds (configurable via the [SlurmctldTimeout parameter](https://slurm.schedmd.com/slurm.conf.html)).

Failover order is determined by the ordering of [SlurmctldHost](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmctldHost) entries in the `slurm.conf` configuration file. For example, a `slurm.conf` with entries:

```
SlurmctldHost=juju-9cf56a-13
SlurmctldHost=juju-9cf56a-16
SlurmctldHost=juju-9cf56a-14
SlurmctldHost=juju-9cf56a-15
```

results in the following when the `scontrol ping` command is used to query controller state:

```shell
$ scontrol ping
Slurmctld(primary) at juju-9cf56a-13 is UP
Slurmctld(backup1) at juju-9cf56a-16 is UP
Slurmctld(backup2) at juju-9cf56a-14 is UP
Slurmctld(backup3) at juju-9cf56a-15 is UP
```

In this setup, the unit with hostname `juju-9cf56a-13` runs as the initial controller serving requests. On primary failure, the first backup controller, `juju-9cf56a-16`, takes over. For subsequent failures, the remaining backups take over in order.

## Performant sharing of StateSaveLocation data

An HA setup has the following prerequisites:

**The contents of Slurm’s StateSaveLocation directory must be available to all slurmctld instances** (primary and all backups). From the [Slurm Quick Start Administrator Guide](https://slurm.schedmd.com/quickstart_admin.html#Config):

*“The **StateSaveLocation** is used to store information about the current state of the cluster, including information about queued, running and recently completed jobs. The directory used should be on a low-latency local disk to prevent file system delays from affecting Slurm performance. If using a backup host, the StateSaveLocation should reside on a file system shared by the two hosts. We do not recommend using NFS to make the directory accessible to both hosts, but do recommend a shared mount that is accessible to the two controllers and allows low-latency reads and writes to the disk. If a controller comes up without access to the state information, queued and running jobs will be cancelled.”*

**The solution used to share StateSaveLocation must be fast** (low-latency, sufficient bandwidth) as I/O to this directory governs the responsiveness of the controller and scalability of the entire cluster. From the [Slurm User Group 2023 Field Notes](https://slurm.schedmd.com/SLUG23/Field-Notes-7.pdf):

*“Your maximum system throughput, and overall Slurm controller*  
*responsiveness under heavy load, will be governed by latency*  
*reading/writing from StateSaveLocation.*

*“In high-throughput (\~200k+ jobs/day) environments, you may be much*  
*better off with a local NVMe drive in a single controller.*

*“Especially if the alternative is an NFS mount shared with users that*  
*gets hammered frequently.*

*“You’re more likely to see performance issues related to this than an*  
*outage from the controller dying.”*

**StateSaveLocation data must either exist on shared storage independent from any instance or be replicated on storage local to each instance**. Suggestions from the [slurm-users mailing list](https://groups.google.com/g/slurm-users/c/XkLznfHw_sw/m/I30UpmzSBgAJ) include [DRBD](https://linbit.com/drbd/) and rsync: “you can use the controller local disk for StateSaveLocation and place a cron job (on the same node or somewhere else) to take that data out via e.g. rsync”. The capacity of StateSaveLocation varies depending on cluster size and utilization. For a point of comparison, [one mailing list message](https://groups.google.com/g/slurm-users/c/aEgBpfiK8_0/m/Srapnf2-BgAJ) states “We have just over 400 nodes and the StateSaveLocation directory has \~600MB of data”. 

### Implemented approach: filesystem-client charm integration

To allow for flexibility in the shared file system, support for the [`filesystem-client` charm](https://github.com/charmed-hpc/filesystem-charms) has been implemented. This enables the user to integrate with the file system of their choice, e.g. their own CephFS deployment, a cloud-specific managed file system, or another that meets their performance and capacity requirements.

Using the local disk for StateSaveLocation data, without the `filesystem-client`, continues to be supported via the new `use-network-state` configuration option for the `slurmctld` charm. The default is `False`, which results in the `slurmctld` charm functioning as it does now: StateSaveLocation data is stored locally at `/var/lib/slurm/checkpoint` and the application cannot scale to support HA.

The `use-network-state` option must be set to `True` at `slurmctld` deployment time for HA to function. When set to `True`, the charm blocks until `/var/lib/slurm/checkpoint` is mounted (specifically until [`Path.is_mount`](https://docs.python.org/3/library/pathlib.html#pathlib.Path.is_mount) returns `True`). The charm leader then generates the JWT key, the auth key, and all configuration files while peers defer until these files appear in the shared storage.

## Charm interface and logic changes

Changes to the Slurm charmed operators are necessary to support an HA configuration as the `slurmctld` operator has, until now, been developed under the assumption that only a single unit would be in operation at any one time.

Updates to the interfaces for `slurmd, sackd, slurmrestd,` and `slurmdbd` are required to account for multiple `slurmctld` instances now observing `relation-*` events.

The `slurmctld` charm must coordinate configuration files such as slurm.conf, gres.conf, acct\_gather.conf, and all others necessary for `slurmctld` operation to ensure the files are available to the primary and all backup units. Similarly, the JWT and cluster authentication keys must be shared securely with all units for failover to be possible.

### Implemented approach: use of shared file system for configuration

The JWT key is already stored in `/var/lib/slurm/checkpoint` in the latest release of `slurmctld`, hence is already available to all units via the shared file system used for the StateSaveLocation. This approach has been expanded upon \- the auth key and configuration files (`slurm.conf`, `gres.conf`, `acct_gather.conf`, etc.) are now added to this location by relocating `/etc/slurm/` to the shared storage. The original `/etc/slurm/` is replaced with a symlink to `/var/lib/slurm/checkpoint/etc-slurm` so all `slurmctld` instances have a coherent view of all necessary files.

To avoid data loss, any existing `/etc/slurm/` is backed up to `/etc/slurm_[timestamp]` on the unit. To prevent peers from reading partially written configuration files, the latest version of `slurmutils` now makes atomic updates.

### `slurmctld` charm changes

The `Install` hook no longer generates keys or start services; it only performs package install operations. Key generation and service starts have been moved to the `Start` hook both to allow for the `filesystem-client` mount to be in place and for a more logical separation of concerns. The setting of the `cluster_name` has been moved to the peer `relation-creation` event to simplify `Start`.

Also added to the peer `relation-creation` event is every controller unit writing its hostname into its unit databag. This allows for a mapping between the unit name, e.g. `slurmctld/1`, and its hostname, e.g. `juju-9cf56a-3`. This mapping is necessary as Slurm requires all `SlurmctldHost` entries to use the controller hostname. Attempts to use the Juju-provided ingress IP address result in the `slurmctld` service failing to start.

A `get_controllers()` method has been added to gather the hostnames of all controller units. The order this function returns the controllers is the same as is written to `SlurmctldHost` lines in `slurm.conf`. This specifies the failover order for high availability: the first `SlurmctldHost` entry is the primary controller, and the remaining entries are backup controllers, used in the order they appear.

Controllers are sorted by join order, with the most recently added unit being the lowest priority backup. This order is maintained by retrieving the list of existing controllers from `slurm.conf` and comparing with the set of controllers in the peer relation. Controllers in the file but not the peer relation have departed, while controllers in the peer relation but not the file are newly added. This ensures that scaling the application does not switch which units are the primary/backups.

An `all_units_observed()` method has been added which returns `True` if the unit has observed all other units in the peer relation and `False` otherwise, i.e. `len(self._relation.units) == self.model.app.planned_units()-1`. This guards uses of `get_controllers()` and the new `update_controllers()` to prevent partial updates of controller hostnames to the `slurm.conf` and `sackd`/`slurmd` relation data.

There is risk of a partial update when a new unit is elected leader as it joins an existing `slurmctld` application (such as when all other units are inaccessible). The `all_units_observed()` check prevents the new leader from attempting a controller update before it has seen the `relation-joined` events for all other controllers. This should allow for recoverability of a severely degraded cluster where all controller units have failed through simply `juju add-unit slurmctld`. However, this scenario was not in scope for this HA implementation, as it relates to disaster recoverability rather than maintaining availability, and has not been extensively tested.

A `get_controller_status()` method has been added which performs an `scontrol ping --json` and returns the current unit's status, e.g. `primary - DOWN`, `backup1 - UP`. This is called in `_check_status()` to update the unit `ActiveStatus` and give insight into controller state when the user runs `juju status`.

The `auth_key` and `jwt_key` are no longer part of the charm stored state. Each access now goes through the `hpclibs` interface which retrieves the key value from the file on disk. This ensures a consistent key is returned across all `slurmctld` units.

### Other charm changes

The `sackd` and `slurmd` interfaces in `slurmctld` have been updated to support multiple controller hostnames and their main charm code now constructs a comma-separated list of hostnames with port ":6817" appended. Updates to controller data for these charms are guarded with `if not self._charm.all_units_observed():` checks to prevent partial updates.

New `if not self.framework.model.unit.is_leader():` guards have been placed around method calls in the interfaces for `sackd`, `slurmd`, `slurmrestd`, and `slurmdbd` where previously a single controller unit was assumed.

The `if not self._charm.slurm_installed:` guards in these interfaces have been removed as keys are now generated in the `Start` hook rather than `Install`. The guards have been replaced with checks for the existence of `auth_key` and `jwt_rsa`.

## Failover tests

TODO: Details of Jubilant test scenarios to go here.

## Alternative approaches considered

Before implementing the above, other options for the StateSaveLocation and Slurm configuration data were explored but found to be lacking in comparison. Details are provided below, including information on proofs of concepts developed to prototype HA candidates.

### Considered sharing of StateSaveLocation data

#### Juju shared storage

Juju manages the storage for the `slurmctld` application. All units share the same storage.

**Advantages**

* Trivial implementation. Juju abstracts the complexities of the clouds and minimal changes needed to the `slurmctld` code and charmcraft.yaml.  
    
* No additional software or services required. HA can be achieved using only native Juju and Slurm functionality.  
    
* Minimal operational complexity. User simply requests storage as usual at application deployment.  
    
* The intended method for making use of storage within a charm.

**Disadvantages**

* Planned but not currently supported by Juju.  
    
* Performance implications unclear until supported by Juju.

#### Primary instance NFS export

The StateSaveLocation data is stored on the primary instance local storage and exported to all others via NFS.

**Advantages**

* Trivial implementation. NFS is a standard protocol, the management of which is already implemented in a number of charms.  
    
* Minimal operational complexity. Transparent to the user; they deploy the charm as usual.

**Disadvantages**

* Provides only minimal additional availability. Would provide resilience against `slurmctld` service failures but not against node failures; backup instances would be unable to take over if the primary instance lost network connectivity. For example, from a software crash in the network stack, a failed network adapter in the underlying host, and so on.

#### Cloud native shared storage

![](https://i.imgur.com/lMILAt4.png)
*Example use case for Microsoft Azure shared disks. [Source](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-shared#persistent-reservation-flow).*

Cloud-specific functionality such as [Azure shared disks](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-shared), a feature allowing users to attach a disk to multiple VMs simultaneously, is used to store StateSaveLocation data. Either Terraform plans or calls to the relevant APIs from the charm code are used to provision and attach the storage.

**Advantages**

* Seamlessly provides a shared storage solution.

**Disadvantages**

* Vendor lock-in. A solution for one cloud will not be portable to another, losing the benefits of Juju and charms.

#### DRBD replication

![](https://i.imgur.com/i3gGsia.png)
*Example of DRBD data replication. [Source](https://linbit.com/blog/shared-nothing-high-availability/).*

StateSaveLocation data is stored on a dedicated block device per `slurmctld` instance. The primary instance writes to its local block device and this is mirrored to all backup instances using [Distributed Replicated Block Device (DRBD)](https://linbit.com/drbd/).

**Advantages**

* StateSaveLocation can be on high-speed storage local to each VM.  
    
* Juju could be used to manage storage provisioning.  
    
* DRBD is an established offering for HA setups, with mature solutions to common replication issues (split brain, rebalancing, self-healing, etc.).

**Disadvantages**

* It is recommended that DRDB be used only with three or more instances to allow for quorum to resolve a split brain. Recovery of a split brain is complex when using only two instances.  
  * The typical `slurmctld` HA setup would use only two instances.


* Only the primary instance can mount the block device. Secondary/backup instances cannot mount the device so cannot read the replicated files until the instance is promoted to primary. The `slurmctld` HA functionality expects all service instances to be able to read the “heartbeat” file located in the StateSaveLocation to determine when a failover has occurred. This would not be possible with DRBD.  
    
* Complex setup and management.  
  * Further investigation is needed to determine the feasibility of dynamically adding and removing nodes.  
  * Any solution would likely require additional management software such as [pacemaker](https://clusterlabs.org/projects/pacemaker/) to detect HA events and perform fail over from primary to backup. This would need careful configuration to ensure DRBD and `slurmctld` agree which backup is promoted.


* Cannot replicate a directory on an existing file system. Requires a dedicated block device.  
  * An image file could be mounted as a loopback device, however this is inadvisable given the management complexity and expected low performance.  
      
* Juju storage provision is currently [bugged for Microsoft Azure](https://github.com/juju/juju/issues/19423), the primary target for Charmed HPC, making it challenging to request a dedicated block device.

#### Syncthing replication

StateSaveLocation data is stored in its default location on storage local to the primary `slurmctld` instance. [Syncthing](https://syncthing.net/) is used to monitor the primary directory for changes and replicate files to all backup instances in real time.

**Advantages**

* Works at the directory level. Does not require provisioning of additional block devices or other storage.  
    
* StateSaveLocation can be on high-speed storage local to each VM.

**Disadvantages**

* Syncthing is intended more for consumer devices (Windows, macOS, Android, etc.) over the internet. It is uncertain how it would perform running long term in a cluster deployment.  
    
* Heavyweight. Uses a web-based GUI for configuration.  
    
* Authentication of devices is performance via secret “device IDs”. Unclear how these could be managed within a charm.

#### Rsync over SSH replication

The [rsync](https://rsync.samba.org/) tool is used to synchronize SaveStateLocation data between primary and backups over SSH. Either all backups periodically run rsync to pull data from the primary via a cron job/systemd timer, or the primary pushes the data to all backups using rsync+inotify or [lrsync](https://github.com/clsync/lrsync) for live syncing.

**Advantages**

* SSH and rsync are already installed by default on all units with lrsync being available from the Ubuntu archive.  
    
* Lightweight. Employs tools established for use in long-running Linux servers.  
    
* StateSaveLocation can be on high-speed storage local to each VM.

**Disadvantages**

* Requires management of SSH credentials to enable transfer of data.  
    
* Requires a method of detecting HA events to determine which host(s) to configure rsync to push to/pull from.  
  * The [SlurmctldPrimaryOffProg](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmctldPrimaryOffProg) and [SlurmctldPrimaryOnProg](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmctldPrimaryOnProg) slurm.conf options allow specification of a program that is executed when a slurmctld primary becomes a backup or a slurmctld backup becomes a primary, respectively. These options could be used to launch a script that performs the necessary rsync configuration.

* Not an established setup for HA. Challenging to recover from undesirable states (split brain, partial copies of files, etc.).

#### Rsync over NFS replication

![](https://i.imgur.com/NwC3BjM.png)

As in "Rsync over SSH replication" but NFS shares are used to make the StateSaveLocation data available, rather than SSH. All instances export their StateSaveLocation directory over NFS. All backup instances mount the primary’s instance. The backup instances periodically run rsync to maintain a local duplicate of the data.

If the primary fails, the first backup instance takes over and starts using its local data copy. Remaining backups switch their NFS share to point to the backup which has taken over.

**Advantages**

* Rsync is already installed by default on all units with NFS utilities being available from the Ubuntu archive.  
    
* Lightweight. Employs tools established for use in long-running Linux servers.  
    
* StateSaveLocation can be on high-speed storage local to each VM.  
    
* Authentication is handled by the cluster identity stack. NFS export access is limited only to IP addresses of slurmctld instances.

**Disadvantages**

* Requires a method of detecting HA events to determine which host   
  * The [SlurmctldPrimaryOffProg](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmctldPrimaryOffProg) and [SlurmctldPrimaryOnProg](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmctldPrimaryOnProg) slurm.conf options allow specification of a program that is executed when a slurmctld primary becomes a backup or a slurmctld backup becomes a primary, respectively. These options could be used to launch a script that performs the necessary rsync configuration.

* Not an established setup for HA. Challenging to recover from undesirable states (split brain, partial copies of files, resyncing updated data to a recovered primary, etc.).

A proof of concept using this approach was developed where AutoFS is used to provide the NFS mount of the active instance to the non-active instances.

A systemd timer is used to periodically perform the rsync from an NFS mount to the local storage on each backup. The inotify subsystem was considered for real-time synchronization but it does not function with NFS mounts \- it cannot detect file changes made by other clients.

On a relation-changed event, triggered by either an update of `slurm.conf` data or a failover event:

* Non-leader units update their slurm.conf from the peer relation data.  
* The output of command `scontrol ping --json` is parsed to determine the activate instance (the command requires an up-to-date slurm.conf to run).  
* Non-active instances:  
  * Stop any current rsync operation and timer.  
  * Forcibly unmount any current NFS mount.  
  * Mount and begin rsync-ing against the active instance.  
    

A shell script is provided to [SlurmctldPrimaryOnProg](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmctldPrimaryOnProg) which executes command:

`sudo juju-exec JUJU_DISPATCH_PATH=hooks/slurmctld_failover {self._charm.charm_dir}/dispatch`

to trigger a custom “slurmctld\_failover” event in the charm whenever a `slurmctld` instance is promoted to active.

The `slurm` user is granted sudo privileges to execute this command.

The “slurmctld\_failover” event on the new active instance/promoted backup:

* Stops the current rsync operation and timer.  
* Forcibly unmounts the NFS mount for the previous active.  
* Removes the NFS mount for the previous active from AutoFS.  
* Writes a “failover” value into the unit databag to trigger a relation-changed event on all other units, resulting in all non-active instances mounting and rsync-ing against the new active.

##### **Proof of concept \- lessons learned**

This approach proved too brittle to continue refining into production due to risks concerning the:

**Reliability of NFS**. Mount points would frequently become stuck in testing, leading to stalled rsync processes and stale StateSaveLocation data on each backup. This is not easily detectable or addressable. AutoFS is not equipped to handle stuck mounts and a manual `unmount –force` command is required. Adjusting NFS mount options (`ro,soft,retrans=1,retry=0`) provides some limited mitigation but has its own implications for reliability.

**Barriers to recoverability from undesirable states.** When StateSaveLocation data becomes out of sync between instances, e.g. a split brain, options are limited for recovery, typically requiring SSH-ing to each instance to manually inspect the state and fix.

Data can easily become out of sync from a stuck NFS mount, a failed/partially successful rsync attempt, etc. or as part of normal operation, such as when a primary instance is recovered after some time and its local copy is stale.

There also exists the possibility of backups syncing from the wrong host. This is not expected to occur in typical operation, however the unmount/remount operation on failover contains significant scope for error and could result if the event is not handled successfully. This risk is exacerbated by a failover realistically only occurring during a disaster event on the units.

**Sudo access granted to the `slurm` user.** The `juju-exec` command used to trigger the failover event requires privileged access. This command is launched by the `slurmctld` instance, which is run under the `slurm` user, hence sudo access was necessary. Granting this access, even limited only to the exact `juju-exec` command, raises security concerns.

**Complexity of tracking active/non-active status.** The charm code required significant extension to handle differing behavior for the active instance and all non-active instances. This was further complicated by the differing behavior necessary for leader and non-leader units, with consideration needed for situations such as a backup/non-active instance being the leader unit and the implications for charm actions, etc. This would be less of a concern with a shared file system, rather than a replication, approach.

#### Csync2 Replication

The [csync2 utility](https://github.com/LINBIT/csync2) is used to keep StateSaveLocation data in sync across all instances. This utility maintains an internal database of files on each host in a cluster and pushes changes whenever it is executed. File conflicts can be resolved automatically by configuring an auto-resolution policy, such as `younger`, where the most recently modified file wins the conflict.

**Advantages**

* csync2 is targeted at this use case with the documentation stating: “It can be used to keep files on multiple hosts in a cluster in sync. \[...\] It is expedient for HA-clusters, HPC-clusters, COWs and server farms.”  
    
* Updates are performed securely using TLS.  
    
* Is available in the Ubuntu archive.  
    
* Low complexity both in configuration and operation.  
    
* Development is supported by LINBIT, the group behind DRBD.  
    
* Addresses the issues identified with the rsync proof of concept:  
  * It does not use NFS so does not have the issue of stuck mounts or other challenges.  
  * No charm action is needed on failover so the slurm user does not require sudo access.  
  * There is no need to track which controllers are active or non-active.

**Disadvantages**

* The maintenance status of the project is unclear with the most recent commit being 5 years ago at time of writing.

A proof of concept using this approach was developed wherein the charm installs csync2 from archive, generates authentication keys and certificates, and writes out the following configuration file:

```
# /etc/csync2.cfg
group charmedhpc {
host <leader-unit-hostname>;
key /etc/csync2.key;
include /var/lib/slurm/checkpoint;
auto younger;
}
```

As the `slurmctld` application is scaled, unit hostnames are added/removed from this configuration file to form a space-separated list in an arbitrary order. Whenever a new unit is added, the charm marks the `/var/lib/slurm/checkpoint` StateSaveLocation directory as “dirty” in the csync2 database (via command `csync2 -mrv /var/lib/slurm/checkpoint`) to allow for an initial sync to the joining unit.

As in the rsync proof of concept, a systemd timer is used to periodically run csync2 and synchronize files across all units.

##### **Proof of concept \- lessons learned**

This approach also proved too brittle to develop further due to risks concerning the:

**Ability to recover from a partially successful sync.** A node failure interrupting a sync would result in a backup controller attempting to take over with incoherent StateSaveLocation data. This would lead to complex failures and a challenging road to recovery.

**Complexity of stale data on a failed primary.** Returning a failed primary to service should ideally require only a restart of the unit once it is in working order. The unit should then rejoin the Juju application and take over from the active backup seamlessly. However, using a csync2, a sync of StateSaveLocation must be completed before the recovered primary can take over, otherwise stale data from before the unit went down would be used.

This could be mitigated by delaying the service start until StateSaveLocation data is synchronized across all hosts, however, as this data is constantly being modified as cluster users manipulate jobs, it is unclear when the recovered service would have the opportunity to start.

In essence, the issue with csync2 in this use case is it guarantees “eventual coherence” whereas the `slurmctld` service expects that its view of StateSaveLocation data is always coherent across instances.

#### MicroCeph replication

[MicroCeph](https://canonical-microceph.readthedocs-hosted.com/en/latest/) is used by the `slurmctld` charm to set up a CephFS file system with units as members of the cluster. StateSaveLocation data is made available to all units via this shared file system.

**Advantages**

* Transparent to the user. They can deploy as usual and can scale `slurmctld` without any additional configuration.  
    
* Addresses the issues identified with the csync2 proof of concept. CephFS gives a coherent view of files once the file system has been mounted.  
    
* StateSaveLocation can take advantage of Ceph features like CephFS snapshots.

**Disadvantages**

* High implementation and management complexity. The charm must manage a file system in addition to the `slurmctld` service, including the ability to manage and troubleshoot Ceph services.  
    
* Larger system requirements than `slurmctld`. Requires a unit specced to host all Ceph services in addition to the Slurm controller.  
    
* Does not support a cluster size of two as cannot achieve quorum with this count. A minimum of three `slurmctld` instances would be required for HA, which would preclude the typical configuration of a primary controller and a single backup.  
    
* Discouraged to add single units with `juju add-unit` as even cluster sizes create challenges, such as difficulty resolving split brains by majority vote.   
    
* Cluster sizes conflict with `slurmctld` view of controller configuration. For example, `slurmctld` configured with three hosts reports it has one primary and two backups, so expects to be tolerant of two failures. This is not the case; a cluster of size three can tolerate only a single failure.  
    
* Requires a separate block device for storage. Loopback files can be used but may not meet performance requirements.

An set of steps for setting up a test MicroCeph cluster of three `slurmctld` units follows:

```shell
# Deploy fork of slurm-charms that allows for multiple slurmctld units: https://github.com/dsloanm/slurm-charms/tree/slurm-ha-rsync
juju add-model charmed-hpc
just repo build slurmctld
# Do not use default VM size. 1GB RAM is not enough and VM will begin swapping after microceph is installed.
juju deploy ./_build/slurmctld.charm --constraints="virt-type=virtual-machine cores=2 mem=4G" -n 3
# Charms are deployed to units: juju-9d0634-[0-2]

# Prepare the cluster
juju ssh slurmctld/0
sudo snap install microceph
sudo microceph cluster bootstrap
# Add 1GB storage
sudo microceph disk add loop,1G,1

# Generate join tokens for the remaining two nodes
# Note that the name used when running "microceph cluster add <name>" must match the hostname of the node where "microceph cluster join" is being run
# See: https://canonical-microceph.readthedocs-hosted.com/en/squid-stable/reference/release-notes/#important-changes
sudo microceph cluster add juju-9d0634-1
sudo microceph cluster add juju-9d0634-2

# Join remaining two nodes to the cluster and add 1GB storage
# juju-9d0634-1
logout
juju ssh slurmctld/1
sudo snap install microceph
sudo microceph cluster join <juju-9d0634-1 token>
sudo microceph disk add loop,1G,1
# juju-9d0634-2
logout
juju ssh slurmctld/2
sudo snap install microceph
sudo microceph cluster join <juju-9d0634-2 token>
sudo microceph disk add loop,1G,1
logout

# Create a new CephFS file system with two pools: one for data and one for metadata
juju ssh slurmctld/0
sudo ceph osd pool create cephfs_meta
sudo ceph osd pool create cephfs_data
sudo ceph fs new newFs cephfs_meta cephfs_data
# size = number of replicas, min_size = minimum number of replicas that must be available
sudo ceph osd pool set cephfs_data size 3
sudo ceph osd pool set cephfs_meta size 3
sudo ceph osd pool set cephfs_data min_size 1
sudo ceph osd pool set cephfs_meta min_size 1

# Mount the new CephFS storage
sudo apt install ceph-common
sudo ln -s /var/snap/microceph/current/conf/ceph.keyring /etc/ceph/ceph.keyring
sudo ln -s /var/snap/microceph/current/conf/ceph.conf /etc/ceph/ceph.conf
sudo mkdir /mnt/mycephfs
sudo mount -t ceph :/ /mnt/mycephfs/ -o name=admin,fs=newFs
```

These steps were adapted into a proof of concept implementation wherein the charm leader performs cluster bootstrapping, as above, and provides join keys to new units via the peer relation. The CephFS storage is mounted via AutoFS, as in the rsync proof of concept.

##### **Proof of concept \- lessons learned**

This proof of concept was not pursued further due to the complexity of managing a clustered file system within the controller charm (we could not reasonably replicate the dedicated [MicroCeph charm](https://charmhub.io/microceph)) and the requirements around maintaining cluster quorum. It also locked the user to the Ceph file system.

#### Gluster replication

![](https://i.imgur.com/TA19seG.png)

A Gluster [replicated volume](https://docs.gluster.org/en/main/Administrator-Guide/Setting-Up-Volumes/#creating-replicated-volumes) is used to create copies of StateSaveLocation data across all instances, with each instance contributing its own brick (export directory).

**Advantages**

* Works at the directory level. Does not require provisioning of additional block devices or other storage.  
    
* Low configuration complexity. All instances can mount the storage simultaneously.  
    
* StateSaveLocation can be on high-speed storage local to each VM.

**Disadvantages**

* The future of Gluster is not immediately clear. The latest product offering [reached end of life on 31 December 2024](https://access.redhat.com/support/policy/updates/rhs) with no further offerings announced at time of writing. Discussions [on the project GitHub](https://github.com/gluster/glusterfs/discussions/4324) suggest Gluster may no longer be supported.  
    
* Scalability and performance is reportedly poor.  
  * Scalability is less of a concern as an HA setup would use only 2-3 `slurmctld` instances.


* Options for recovery from split brain or self-healing are unclear.

An example of this approach, creating a replicated volume on two hosts then expanding to a third:

```shell
# Two servers: juju-d7b2ca-27 and juju-d7b2ca-28

# On both, install gluster
sudo apt install glusterfs-server
sudo systemctl enable --now glusterd

# On juju-d7b2ca-27, add juju-d7b2ca-28
sudo gluster peer probe juju-d7b2ca-28

# On both, create a new brick directory
sudo mkdir -p /gluster/brick1

# On juju-d7b2ca-27, create a new replicated volume over both servers
# "force" needed because we've used the root filesystem as a brick for this example
sudo gluster volume create myvol replica 2 transport tcp juju-d7b2ca-27:/gluster/brick1 juju-d7b2ca-28:/gluster/brick1 force

# On both, mount the filesystem
sudo mkdir /mnt/glusterfs
## On juju-d7b2ca-27:
sudo mount -t glusterfs juju-d7b2ca-27:/myvol /mnt/glusterfs
## On juju-d7b2ca-28:
sudo mount -t glusterfs juju-d7b2ca-28:/myvol /mnt/glusterfs

# Create a file on on juju-d7b2ca-27 and it will be replicated to juju-d7b2ca-28:
sudo nano /mnt/glusterfs/hello.txt
ls /mnt/glusterfs/

# Expand to a third server: juju-d7b2ca-29
# On juju-d7b2ca-29
sudo apt install glusterfs-server
sudo systemctl enable --now glusterd

# On either juju-d7b2ca-27 or juju-d7b2ca-28, add juju-d7b2ca-29
sudo gluster peer probe juju-d7b2ca-29

# On juju-d7b2ca-29, expand the volume (increasing replica count)
sudo mkdir -p /gluster/brick1
sudo gluster volume add-brick myvol replica 3 juju-d7b2ca-29:/gluster/brick1 force
sudo mount -t glusterfs juju-d7b2ca-29:/myvol /mnt/glusterfs
ls /mnt/glusterfs
```

#### Modification of slurmctld code to use a database

The `slurmctld` service source code is modified to store the StateSaveLocation data in a database rather than on the file system.

**Advantages**

* Eliminates the need for a file system-based solution. All sharing/replication is handled by native database functionality.  
    
* Databases have strong support in the charm ecosystem.  
    
* Likely to be more performant than the file system, given the I/O access pattern.  
    
* Potential to leverage the existing `slurmdbd` service, which currently provides an interface to the Slurm accounting database.

**Disadvantages**

* High implementation complexity. Would require significant effort.  
  * Writes to StateSaveLocation are in a number of locations within the Slurm source code. A buffer is created with the relevant data and is written out to disk. This buffering could potentially be converted to database calls.  
  * The corresponding read function reads into a buffer, which could similarly be converted to database queries.


* Would require approval from SchedMD to be accepted into the Slurm codebase.

#### Shared storage via filesystem-client (chosen approach)

![](https://i.imgur.com/D8ZusWr.png)

An independent storage system is set up, e.g. using storage services provided by the underlying cloud or the user’s own MicroCeph deployment. Each `slurmctld` unit mounts this storage, via the [filesystem-client](https://charmhub.io/filesystem-client) subordinate charm, and uses it to store StateSaveLocation data.

A challenge with this approach is, given the shared storage is provided via subordinate charm, it is not available during the install hook of the `slurmctld` charm. The charm must block until the subordinate finishes its setup before the `slurmctld` service can start and StateSaveLocation data can be written.

**Advantages**

* Low implementation complexity. All that is needed is successful integration of the filesystem-client.  
    
* There is no minimum unit count. The typical case of a primary controller and a single backup is supported. The complexities of clustering and maintaining quorum are delegated to the shared storage implementation.  
    
* The user is not locked into a specific file system and can choose one that meets the requirements for their specific cluster deployment.  
    
* An independent shared file system is the approach slurmctld expects for HA.

**Disadvantages**

* Moderate operational complexity. The user is required to integrate a separate subordinate charm.

The proof of concept for this approach was carried forward and became the final implementation.

### Considered charm interface and logic changes

The use of the peer relation to store configuration data, as in `slurmrestd`, was implemented and used in the proofs of concept for rsync, csync2, and MicroCeph, but was deprecated in favour of the shared file system due to challenges around:

* the number of configuration files needing stored (`slurmrestd` requires only `slurm.conf` but each controller instance would require copies of every conf file: `slurm.conf, gres.conf, acct_gather.conf, oci.conf`, etc.).  
    
* ensuring data in the relation remains in sync with the configuration files on disk  
    
* timing issues ensuring all peers have written the latest configuration data before the leader issues an `scontrol reconfigure`

Only the aspect of each unit writing its own hostname into its databag and some interface changes remain in the final implementation. Further details on this considered approach are given below.

#### slurmctld

![](https://i.imgur.com/JcyZ0Z8.png)

*Illustration of flow for data in the slurmctld peer relation and local unit databags.*

As an initial change, the install hook is modified such that non-leader units are no longer immediately set to `BlockedStatus("slurmctld high-availability not supported")` on deploy.

#### SlurmctldHost definition and ordering

Assembly of the slurm.conf file is modified to allow for multiple [SlurmctldHost](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmctldHost) definitions. The order of these definitions determines HA failover order: “the first entry will run as the primary and all other entries as backups. If the first specified host fails, the daemon will execute on the second host. If both the first and second specified host fails, the daemon will execute on the third host”.

SlurmctldHost requires a machine hostname as its value. To ensure this is available to the application, during the relation-creation hook:

* each unit writes its hostname into its unit peer relation databag with key “hostname”  
* the charm leader additionally writes its hostname into the application peer relation databag with key “controllers”

At each subsequent relation-joined hook, the charm leader retrieves the hostname of the joining unit from its unit databag and appends it to the (comma-separated) list of controllers in the application databag. This establishes a consistent ordering of primary and backup `slurmctld` hosts within the application, with the leader (at time of deployment) as the primary and backups in order of unit joining.

During configuration, the charm leader assembles the `slurm.conf` file by querying the peer relation for the list of “controllers” and writes out `SlurmctldHost=<hostname>` entries in the order specified.

#### Duplicating configuration data across peers

All units in the application must have their own identical copies of Slurm configuration files: `slurm.conf, gres.conf,` and all others. The [“configless”](https://slurm.schedmd.com/configless_slurm.html) approach employed by `slurmd, sackd`, and other Slurm charms cannot be employed as `slurmctld` is the source of configuration data in this setup. Similarly, the cluster authentication key must be shared with all peers.

The solution follows that of the `slurmrestd` charm, which similarly requires its own copy of `slurm.conf`. The application peer relation is now used to store all configuration data, ensuring persistence if any `slurmctld` unit fails.

The `slurmctld` leader writes out configuration data to disk using methods: `_on_write_slurm_conf, _update_gres_conf,` and `_refresh_gres_conf`. Each is modified to also push a copy of the data into the application peer relation.

On this push, a `relation-changed` event is triggered and non-leader instances retrieve configuration data and the cluster authentication key from the peer relation. The data is written to the appropriate paths on disk and relevant services are reloaded or restarted.

#### Other charm relations

The `slurmd` and `sackd` charms integrate with `slurmctld` and have been adjusted to handle multiple instances. The `config_server` attribute of the `slurmd` and `sackd` managers now takes a comma-separated list of `slurmctld` hostnames, rather than a single hostname.

The application relation data between `slurmctld` and `slurmd`/`sackd` has been extended to include the ordered list of `slurmctld` hostnames from the `slurmctld` peer relation data, rather than just the leader `slurmctld` hostname.

As the list of hostnames can now change dynamically (through `juju add-unit` and `juju remove-unit`), the `slurmctld` hostname(s) in the relation data must now be updated whenever a new `slurmctld` instance joins or leaves the relation. This is accomplished by defining a custom `SlurmctldAvailableEvent` event that is emitted following a `slurmctld` peer relation-joined event. The handler for this event then updates the `slurmd` and `sackd` relations with the latest list of controllers.

# References

The Slurm documentation on HA is available at:

* [https://slurm.schedmd.com/quickstart\_admin.html\#HA](https://slurm.schedmd.com/quickstart_admin.html#HA)  
* [https://slurm.schedmd.com/slurm.conf.html\#OPT\_SlurmctldHost](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmctldHost)  
* [https://slurm.schedmd.com/quickstart\_admin.html\#Config](https://slurm.schedmd.com/quickstart_admin.html#Config)

# Spec History and Changelog

| Date       | Status    | Author(s)                                                        | Comment                                                |
|:---------- |:--------- |:---------------------------------------------------------------- |:------------------------------------------------------ |
| 2025-04-01 | Braindump | [Dominic Sloan-Murphy](mailto:dominic.sloanmurphy@canonical.com) | Initial braindump                                      |
| 2025-04-30 | Braindump | [Dominic Sloan-Murphy](mailto:dominic.sloanmurphy@canonical.com) | Updates to potential approaches                        |
| 2025-06-25 | Drafting  | [Dominic Sloan-Murphy](mailto:dominic.sloanmurphy@canonical.com) | Revised all sections in preparation for public release |