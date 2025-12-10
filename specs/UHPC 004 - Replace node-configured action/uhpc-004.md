| Index          | HPC049                                                           |                                                                                                                |             |
|:---------------|:-----------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------|:------------|
| Title          | Replace `node-configured` action                                 |                                                                                                                |             |
| **Type**       | **Author(s)**                                                    | **[Status](https://docs.google.com/document/d/1lStJjBGW7lyojgBhxGLUNnliUocYWjAZ1VEbbVduX54/edit?usp=sharing)** | **Created** |
| Implementation | [Jason Nucciarone](mailto:jason.nucciarone@canonical.com)        | Pending Review                                                                                                 | 2025-10-30  |
|                | **Reviewer(s)**                                                  | **Status**                                                                                                     | **Date**    |
|                | [Dominic Sloan-Murphy](mailto:dominic.sloanmurphy@canonical.com) | Accepted                                                                                                       | 2025-11-04  |
|                | [Julián Espina Del Ángel](mailto:julian.espina@canonical.com)    | Accepted                                                                                                       | 2025-11-04  |
|                | [James Beedy](mailto:james@vantagecompute.ai)                    | Pending Review                                                                                                 | 2025-12-10  |

# Abstract

This specification proposes replacing the existing `node-configured` action
on the slurmd charm with a better, more scalable action that provides
the same level of control to Charmed HPC's existing set of users.

# Rationale

The slurmd charm currently has an action named `node-configured`. The
action is used to move deployed compute nodes from their initial "down"
state to the "idle" state in Slurm after the slurmd application is
integrated with the slurmctld application, and provides "statefulness"
for new compute nodes. It enables a Charmed HPC cluster operator to
control when a newly deployed compute node is ready to have jobs
scheduled on it.

This stateful action is required because Slurm has no concept of provisioning; it
can only see the self-reported status of the slurmd service managed by the slurmd
charm on the compute node. While compute nodes could report their initial state as
“idle” by default, this behavior is problematic when scaling an existing partition
in a Charmed HPC cluster. For example, if an administrator were to scale their
Charmed HPC cluster under heavy load, they would run the command:

```shell
juju add-unit -n 100 slurmd
```

This would deploy 100 new compute nodes that will be enlisted
automatically in the "idle" state; therein lies the problem. While the
slurmd service sets the compute node's state to "idle" and advertises
itself as ready to receive jobs from the Slurm controller, the compute
node itself may not have finished provisioning. The SSSD and filesystem
client subordinate units may not have finished deploying, so a job
scheduled on the new compute node might not be able to drop down to the
correct user or access the necessary data on the shared filesystem.
Because the slurmd unit has no context of the SSSD and filesystem client
subordinate charms, the default state of the compute node must be set
to "down" so that jobs are not prematurely scheduled before all the
units have been deployed.

However, there are several issues with the `node-configured` action.

### **Scalability**

The `node-configured` action does not scale in large cluster
environments because Juju does not support application-level actions.
This means the `node-configured` action must be run on each deployed
compute node. The Charmed HPC cluster operator must therefore either:

1. Call `juju run slurmd/<unit-#> node-configured` on each new slurmd
   unit, or
2. Write a custom script that iteratively calls `node-configured` on
   each new slurmd unit that must be marked "idle".

Both approaches severely impact the operator experience as they require
operators to sink time into developing custom solutions ad-hoc to handle
something Charmed HPC should automatically handle by default.

### **State inconsistencies**

The Slurm charms have several conflicting actions for managing the state
of compute nodes. For example, the slurmctld charm has a `resume` action
that can set the state of multiple compute nodes to "idle". A Charmed
HPC cluster operator will likely use the `resume` action as it is more
convenient for activating the compute nodes; however, the state set by
this action will be overwritten by the `node-configured` action.

This is because `node-configured` modifies the stored state of each
slurmd unit, while the `resume` action only calls `scontrol` directly.
The state set by the `resume` action will be overwritten the next time an
event occurs in the slurmd unit, as the arguments passed to the
`--conf` flag are rebuilt from stored state in the slurmd unit. This
behavior is incorrect and frustrates Charmed HPC cluster operators.

Provided these issues above, it is clear that the `node-configured`
action must be removed from the slurmd charm and replaced with a better
implementation.

# Specification

The sections below propose new actions and configuration options that
will replace the `node-configured` action, and elaborate on how the new
actions will address the current issues caused by the Slurm charms.

## Assumptions

The proposed implementations below assume that Charmed Slurm uses the
unit name of a deployed slurmd charm rather than the machine hostname as
the enlisted node name in Slurm.

## Proposed actions

The actions proposed below will replace the "statefulness" feature of
`node-configured`. These actions provide the level of control necessary
to move new compute nodes from the "down" state to the "idle" state
before scheduling jobs.

### **`set-node-state`**

An action like `set-node-state` was originally proposed in a GitHub
Discussion with the Ubuntu High-Performance Computing community. 
The `set-node-state` action will provide the "statefulness" originally 
supplied by `node-configured`, but it will be more versatile. 
Rather than having `node-configured` which only updates the state of "new" 
nodes, `set-node-state` can be used to control the  state of large groups 
of nodes. For example, `set-node-state` can put nodes in the "drain" state
for scheduled maintenance:

```shell
juju run slurmctld/0 \
  set-node-state nodes='slurmd-[0-19]' \
  state=drain \
  reason='Removing for maintenance'
```

The `set-node-state` action will be callable from any unit in a slurmctld
application as the slurmctld application leader may failover to a
secondary controller. Operations may need to be performed on compute
nodes while the primary controller is recovered.

#### Persisting compute node state across slurmd service restarts

No additional mechanisms are required to persist compute nodes' states
between slurmd service restarts. Slurm will remember the current state
of a restarted node in Slurm's configured `StateSaveLocation`.

#### Persisting compute node state across machine restarts

Compute nodes that are rebooted using the `juju-reboot` command or with another cloud-provided mechanism will automatically be
brought back in the "down" state. There is no guarantee that compute
nodes were rebooted for a good reason, such as applying driver or
kernel updates, so compute nodes should be kept out of the allocatable
compute node pool by default until the operator can verify the node can
return to service.

If an operator does not want compute nodes to be brought up as "down"
after a machine reboot, an `_on_start` hook in the slurmd charm can
attempt to set the default state if the slurmd service is running:

```python
# Assume `SlurmdCharm` is fully defined.

def _on_start(self, _: ops.StartEvent) -> None:
    """Handle when machine is restarted."""
    if self.slurmd.is_active() and self.slurmctld.is_ready():
        scontrol(
            "update",
            f"nodename={self.unit.name}",
            f"state={self.config.get('default-state')}",
            f"reason={self.config.get('default-reason')}",
        )
```

The `_on_start` hook can return the compute node to service or set a
different default state and reason if the slurmd service has been
enabled by systemd and the slurmctld integration is ready.

The Charmed HPC cluster operator will still need to reset the state
they applied using the `set-node-state` action unless they updated the
proposed `default-state` and `default-reason` configurations to reflect
that state and reason.

## Proposed configuration options

The configuration options proposed below will modify the default
behavior of the slurmd charm, where a new compute node is enlisted into
Slurm in the "down" state with the reason "New node." These
configuration options provide the same behavior, but allow Charmed HPC
cluster operators to set other defaults.

#### `default-state`

The `default-state` configuration option will be used to set the default
state of a compute node when it is first deployed or its underlying
machine is rebooted. For example, to have new compute nodes enlisted in
the "idle" state when a new slurmd application is deployed:

```shell
juju deploy slurmd -n 100 --config default-state="idle"
```

To replicate the current behavior of the slurmd charm:

```shell
juju deploy slurmd -n 100 --config default-state="down"
```

The default value for `default-state` will be "down" as it prevents
operators from retroactively needing to prevent new compute nodes from
receiving jobs in a scaling partition. Charmed HPC power users can set
`default-state` to "idle" for brand-new deployments to save time.

#### `default-reason`

The `default-reason` configuration option will be used to set the
default reason when a node is first deployed or its underlying machine
is rebooted. For example, to have new compute nodes have "New node"
set as the reason why the node is enlisted as "down" by default:

```shell
juju deploy slurmd \
  --config default-state="down" \
  --config default-reason="New node."
```

`scontrol` does not let `Reason` be empty by default. However, an
operator may not have a default reason configured. The default value
for `default-reason` will be an empty string ("") that will evaluate
to "n/a" for "Not Available" when applying the node configuration
update:

```py
# Assume `SlurmdCharm` is fully defined.

def _on_start(self,_: ops.StartEvent) -> None:
    if self.slurmd.is_active() and self.slurmctld.is_ready():
        scontrol(
            "update",
            f"nodename={self.unit.name}",
            f"reason={self.config.get('default-reason') if ... else 'n/a'}",
        )
```

The `_on_start` hook will be run each time the machine is rebooted, so
the operator will need to modify the `default-reason` configuration
option after the initial cluster deployment:

```shell
# Assume there is an existing Charmed HPC deployment.

juju config slurmd default-reason="Machine rebooted"
```

# Further Information

## Relevant links

1. [Original Ubuntu HPC community discussion on GitHub Discussions](https://github.com/orgs/charmed-hpc/discussions/16)
2. [`StateSaveLocation` documentation](https://slurm.schedmd.com/slurm.conf.html#OPT_StateSaveLocation)
3. [`juju-reboot` documentation](https://documentation.ubuntu.com/juju/3.6/reference/hook-command/list-of-hook-commands/juju-reboot/)
4. [“down” state documentation](https://slurm.schedmd.com/slurm.conf.html#OPT_DOWN)
5. [“drain” state documentation](https://slurm.schedmd.com/slurm.conf.html#OPT_DRAIN)
6. [“idle” state documentation](https://slurm.schedmd.com/slurm.conf.html#OPT_UNKNOWN)
7. [`Reason` documentation](https://slurm.schedmd.com/slurm.conf.html#OPT_Reason)
8. [“fail” documentation](https://slurm.schedmd.com/slurm.conf.html#OPT_FAIL)
9. “[failing” documentation](https://slurm.schedmd.com/slurm.conf.html#OPT_FAILING)

# Spec History and Changelog

| Author(s) | Status | Date | Comment                                                                                                                                                            |
| :---- | :---- | :---- |:-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [Jason Nucciarone](mailto:jason.nucciarone@canonical.com) | Drafting | 2025-10-30 | Start initial draft.                                                                                                                                               |
| [Jason Nucciarone](mailto:jason.nucciarone@canonical.com) | Pending Review | 2025-11-04 | Submitted for initial review to [Dominic Sloan-Murphy](mailto:dominic.sloanmurphy@canonical.com) and [Julián Espina Del Ángel](mailto:julian.espina@canonical.com) |
| [Jason Nucciarone](mailto:jason.nucciarone@canonical.com) | Pending Review | 2025-11-04 | Remove proposed `mark-\*` actions, rename `set-state` to `set-node-state`                                                                                          |




