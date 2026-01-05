| Index          | UHPC005                                                          |                   |             |
|:---------------|:-----------------------------------------------------------------|:------------------|:------------|
| Title          | Use unit name as node name                                       |                   |             |
| **Type**       | **Author(s)**                                                    | **Status** 		     | **Created** |
| Implementation | [Jason Nucciarone](mailto:jason.nucciarone@canonical.com)        | Drafting          | 2025-12-17  |
|                | **Reviewer(s)**                                                  | **Status**        | **Date**    |
|                | [Dominic Sloan-Murphy](mailto:dominic.sloanmurphy@canonical.com) | Pending Review    | 2026-01-05  |
|                | [Julián Espina Del Ángel](mailto:julian.espina@canonical.com)    | Pending Review    | 2026-01-05  |
|                | [James Beedy](mailto:james@vantagecompute.ai)                    | Pending Review    | 2026-01-05  |
|                | [Ashley Cliff](mailto:ashley.cliff@canonical.com)                | Pending Review			 | 2026-01-05  |

# Abstract

This specification proposes modifying the existing slurmd charm to use assigned
unit names instead of machine hostnames as the node name for registered compute
nodes in Charmed HPC.

# Rationale

The slurmd charm currently uses units' underlying machine hostnames as the node
name for compute nodes in Charmed HPC. While technically valid, using units'
machine hostname as their node name in Slurm presents several usability challenges.

### **Associating machine hostnames with slurmd units**

It's difficult to associate a machine hostname with a specific unit quickly.
Units are associated with a machine ID number in the output of `juju status`.
However, the Charmed HPC cluster administrator now has to view the output 
of the `"Machine"` section in the `juju status` output to identify the hostname
of the machine that the unit is running on:

```shell
$ juju status slurmd
Model    Controller              Cloud/Region         Version  SLA          Timestamp
testing  charmed-hpc-controller  localhost/localhost  3.6.12   unsupported  23:02:37Z

App     Version        Status  Scale  Charm   Channel  Rev  Exposed  Message
slurmd  25.11.0-1ppa2  active      1  slurmd             0  no       

Unit       Workload  Agent  Machine  Public address  Ports     Message
slurmd/0*  active    idle   0        10.136.168.96   6818/tcp  

Machine  State    Address        Inst id        Base          AZ                       Message
0        started  10.136.168.96  juju-cf9d96-0  ubuntu@24.04  integration-test-runner  Running
```

While this workflow of visually parsing the output of `juju status` works for 
small Charmed HPC cluster deployments, it does not scale as the number of slurmd units
and machines grows. Eventually, cluster administrators are forced to rely on 
custom, ad-hoc scripts to quickly get the machine hostnames of units before
performing operations with `scontrol` or `sacctmgr`.

Also, a compute node name such as `slurmd-0` is easier for an administrator to recall
from memory than the name `juju-xjh620-127`.

### **Machine hostname assignment is non-deterministic**

There's no guarantee that a slurmd unit will be assigned a specific machine hostname.
This non-deterministic behavior makes it difficult to use Slurm's nodename syntax
to perform bulk operations on compute nodes. For example, ad-hoc administrative scripts 
that use Slurm charm actions such as `set-node-state` must be tailored to each
Charmed HPC cluster deployment rather than using a predictable node name range such
as `slurmd-[0-59]`.

There's also no guaranteed order in which the machines will be assigned to slurmd units.
While the unit `slurmd/0` could be assigned machine `juju-xyz123-1`, unit `slurmd/1`
might be assigned machine `juju-xyz123-3` because machine `juju-xyz-2` was assigned
to a different application. The node name range then becomes the less user-friendly
`juju-xyz123-[1,3]` instead of a continuous range such as `slurmd-[0-1]`.

Given these issues above, using compute nodes' unit name rather than their hostname
in Slurm will improve the overall user experience for Charmed HPC.

# Specification

The sections below outline the proposed implementation for using a unit name rather
than a machine hostname for registered compute nodes, and identify workarounds for
using unit names in cross-model and cross-controller integrations.

## Setting the node name to the unit name

The `-N` flag can be used with slurmd to set a custom node name when starting the
slurmd service in dynamic mode:

```shell
$ slurmd -N "slurmd-0" --conf-server ... --conf ...
```

The `-N` flag is commonly used when emulating a larger system on a single node and
can be used with Charmed HPC since its Slurm packages are built with the 
`--enable-multiple-slurmd` option enabled. This option will be added as a modifiable 
property on `SlurmdManager` that will be set in _/etc/default/slurmd_.

### Considerations

Unit names, like `slurmd/0` for example, cannot be used as the node name exactly because 
the `/` character interferes with the cgroup hierarchy created by systemd for slurmd. 
`/` must be substituted with a different, valid character such as `-`:

```python3
# Assume `SlurmdCharm` is fully defined.

def _on_slurmctld_ready(self, event: SlurmctldReadyEvent) -> None:
    self.slurmd.name = self.unit.name.replace("/", "-")
```

## Handling node name collisions

Application names must be unique when deployed in the same model, but applications
can share the same name if they are deployed in separate models or with
different controllers. This can cause challenges when using the unit names as the
compute nodes' node name as Slurm requires node names to be unique. The integration 
ID number can be used to circumvent this issue. 

The slurmctld charm uses "Include" configuration files to store partition 
information. This mechanism can be updated to include the integration ID number
in the include file name suffix:

```python
# Assume `SlurmdCharm` is fully defined.

def _on_slurmctld_ready(self, event: SlurmdReadyEvent) -> None:
    data = self.slurmd.get_compute_data(event.relation.id)
    name = data.partition.partition_name
    include = f"slurm.conf.{event.relation.id}.{name}"
```

Now, by including the integration ID number in the include file name, the slurmctld
charm can check if a new partition collides with an existing partition:

```python
# Assume `SlurmdCharm` is fully defined.

def _on_slurmctld_ready(self, event: SlurmdReadyEvent) -> None:
    data = self.slurmd.get_compute_data(event.relation.id)
    name = data.partition.partition_name
    includes = self.slurmctld.config.includes
    
    for include in includes:
        integration_id, partition = include.rsplit(".")[-2:]
        if partition == name and integration_id != event.relation.id:
            raise StopCharm(
                f"Cannot register new partition '{name}'. Conflicts with an existing partition."
            )
```

The `SlurmdReadyEvent` will not be deferred if there is a partition name collision as
there is no additional data to wait for from the conflicting slurmd application.

### Considerations

Handling partition naming conflicts is considered out-of-scope for this specification
as its focus is on switching compute node names from machine hostnames to unit names;
however, it may be possible to supply a custom partition and node name scheme so that
compute nodes being moved from one controller to another do not have to be destroyed
and redeployed.

The configuration option `partition_name` could be provided that enables a
deployed `slurmd` application to update its name and registered node names. The
slurmd charm could handle updating the partition name and node names in the 
`_on_config_changed` event handler:

```python
# Assume `SlurmdCharm` is fully defined.

def _on_config_changed(self, event: ops.ConfigChangedEvent) -> None:
    if self.slurmctld.is_ready() and (name := self.config.get("partition_name")):
        partition = Partition(partition_name=name)
        self.slurmctld.set_compute_data(ComputeData(partition=partition))

        self.slurmd.name = f"{name}-{self.unit.name.split('/')[-1]}"
        scontrol("delete", f"nodename={self.unit.name.replace('/', '-')}")
        self.slurmd.service.restart()
```

A separate specification is required to fully design the implementation for supporting
custom partition and node names. For example, the `_on_slurmctld_ready` event handler would
also need to be updated to handle when a Charmed HPC cluster administrator has provided a
custom name for the partition.

# Further Information

1. [`slurmd` manpage](https://manpages.ubuntu.com/manpages/resolute/en/man8/slurmd.8.html)
2. [Issue #83 on `charmed-hpc/slurm-charms`](https://github.com/charmed-hpc/slurm-charms/issues/83)
3. [`slurm.conf` manpage](https://slurm.schedmd.com/slurm.conf.html#SECTION_DESCRIPTION)

# Spec History and Changelog

| Author(s)                                                 | Status   | Date       | Comment              |
|:----------------------------------------------------------|:---------|:-----------|:---------------------|
| [Jason Nucciarone](mailto:jason.nucciarone@canonical.com) | Drafting | 2025-12-17 | Start initial draft. |