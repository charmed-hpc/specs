---
index: UHPC007
title: Prometheus alerts in Charmed Slurm
---

# Prometheus alerts in Charmed Slurm

## Abstract

This specification outlines how Prometheus alerts are implemented within the Slurm charms, and how
node performance metrics will be scraped from `sackd` and `slurmd` units with `node_exporter`.

## Rationale

Prometheus alerts are necessary for alerting Charmed HPC cluster administrators when there are
problems with their Slurm deployment so that cluster administrators are not required
to be continuously logged into their cluster to determine its health. These alerts provide crucial 
information such as if compute nodes have failed or if an abnormal amount of jobs are failing. 
Additional alerts for if a login node is oversubscribed or if a compute node has ghost processes 
can also be provided.

However, an effective categorization of alert rules is required to prevent alert fatigue, and additional
work is required on `sackd` and `slurmd` charms to expose crucial machine metrics such as CPU utilization.

## Specification

### Alert rules

Defining specific Prometheus alert rules is out-of-scope for this specification. The list of included
alerts will be dynamic and change as Charmed HPC is further developed, so it would be unmaintainable
to track all alerts in a single specification. Instead, the list of available alert rules should be
documented in the reference section of Charmed HPC's documentation, and that reference section should
be the "source-of-truth" for all alerts provided by Charmed Slurm.

The heuristic for determining whether an alert rule should be included within Charmed HPC will be:

1. Is it useful to all Charmed HPC cluster administrators? If an alert is only useful to one particular
   person and not an entire administration team, the alert shouldn't be included.
2. Is it actionable? Alerts should point out specific problems that must be addressed or the cluster 
   administrators should be aware of for incoming user support requests. The cluster administration
   team should not receive alerts like when a specific user's job has been scheduled or completed, but
   they should be aware if their cluster's resource are fully utilized.
3. Does the alert avoid false positives? An alert should not be fired every time the slurmd
   service is restarted on a compute node, but an alert should be fired if the slurmd service has been
   down for more than 5 minutes.

### Common alert name prefixes

Grafana's [Alert list](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/visualizations/alert-list/)
visualization can filter displayed alerts by a common prefix. This filtering functionality can
be used to display specific groups of alerts such as alerts for individual compute nodes
or alerts for the cluster scheduler.

The list below provides the common alert name prefixes for the Prometheus alerts provided 
by the Slurm charms:

- `SlurmNodes*`: Alerts related to all nodes in the cluster.

- `SlurmPartitions*`: Alerts related to the partitions in the cluster.
 
- `SlurmJobs*`: Alerts related to all the jobs in the cluster.
 
- `SlurmScheduler*`: Alerts related to the health of the Slurm scheduler and corresponding services.
 
- `SlurmComputeNode*`: Alerts specific to individual compute nodes.
 
- `SlurmLoginNode*`: Alerts specific to individual login nodes.

> [!WARNING]
> `SlurmComputeNode` is used instead of `SlurmNode` to prevent filtering clashes between
> `SlurmNode` and `SlurmNodes` in Grafana.

### Alert rule groups

The sections below outline the different alert rule groups that will be provided by Charmed Slurm and
which Slurm charm will own the alert rule group under the `prometheus_alert_rules` directory:

#### Alert rule groups provided by the `slurmctld` charm

##### _nodes.rules_

The _node.rules_ alert group file will contain alerts related to the state of 
compute nodes in the Slurm cluster. Example alerts include:

```yaml
groups:
- name: SlurmNodes
  rules:
    - alert: SlurmNodesDown
    # ...
    - alert: SlurmNodesNotResponding
    # ...
```

##### _partitions.rules_

The _partitions.rules_ alert group file will contain alerts related to the state of
partitions in the Slurm cluster. Example alerts include:

```yaml
groups:
- name: SlurmPartitions
  rules:
    - alert: SlurmPartitionMaxCPUUsage
      # ...
    - alert: SlurmPartitionMaxMemoryUsage
      # ...
```

##### _jobs.rules_

The _jobs.rules_ alert group file will contain alerts related to the state of all jobs 
in the Slurm cluster. These alerts should not be used to provide job-specific updates 
such as when a job starts or completes, but instead when there's noticeable issues such 
as a high-rate of job failures:

```yaml
groups:
- name: SlurmJobs
  rules:
    - alert: SlurmJobsHighRateOfFailure
      # ...
    - alert: SlurmJobsLongQueueTime
      # ...
```

##### _scheduler.rules_

The _scheduler.rules_ alert group file will contain alerts related to the health of the scheduler 
and `slurmctld` service. Example alerts include:

```yaml
groups:
- name: SlurmScheduler
  rules:
    - alert: SlurmSchedulerTooManyFailedDbdMessages
      # ...
    - alert: SlurmSchedulerServiceFailed
      # ...
```

#### Alert rule groups provided by the `slurmd` charm

##### _compute-node.rules_

The _compute-node.rules_ alert group file will contain alerts related to individual compute nodes rather
than provide broad alerts about all nodes in the Slurm cluster. Instead of firing alerts for when
there's down nodes detected in the cluster, these alerts will fire if there's issues such as
ghost processes running on the compute node when none of the node's resources have been allocated.
For example:

```yaml
groups:
- name: SlurmComputeNode
  rules:
    - alert: SlurmComputeNodeHighCPUUsageWhileIdle
      # ...
    - alert: SlurmComputeNodeComputeServiceFailed 
      # ...
```

#### Alert rule groups provided by the `sackd` charm

##### _login-node.rules_

The _login-node.rules_ alert group file will contain alerts related to individual login nodes.
These alerts will monitor for situations such as too much memory or CPU usage by users.
Example alerts include:

```yaml
groups:
- name: SlurmLoginNode
  rules:
    - alert: SlurmLoginNodeHighCPUUsage
    # ...
    - alert: SlurmLoginNodeHighMemoryUsage
    # ...
```

### Collecting metrics from `sackd` and `slurmd` units

`node_exporter` must be installed on the `sackd` and `slurmd` units to collect machine-level
metrics such as CPU usage and Slurm service health. While the OpenTelemetry Collector charm
does install the `node_exporter` snap package, the charm cannot be deployed as a subordinate with 
`sackd` and `slurmd` units because of scalability issues. 

OpenTelemetry Collector uses the peer integration `otelcol_replica` for replication. Peer integrations
scale by O(n^n) so unit settling time will increase exponentially for each additional OpenTelemetry
Collector unit. This performance degradation isn't an issue for `slurmctld` applications since 
three controller units are adequate for scheduler high-availability, but this performance 
degradation is an issue for `sackd` and `slurmd` applications where the total amount of units
is unbounded.

The `sackd` and `slurmd` charms must manage their own instance of `node_exporter` because of these
scalability issues with the OpenTelemetry Collector charm.

#### Scraping `node_exporter` metrics from `sackd` and `slurmd` units

Charmed HPC integrates with the Canonical Observability Stack (COS) through an OpenTelemetry
Collector application integrated with the `slurmctld` application. Rather than deploy additional
OpenTelemetry Collector units to the `sackd` and `slurmd` applications, instead the `sackd` and
`slurmd` applications can provide a `metrics-endpoint` integration over the `prometheus_scrape`
interface to the OpenTelemetry Collector application.

The `prometheus_scrape` charm library can be used to set the necessary integration data in the
`sackd` and `slurmd` charms to expose the `node_exporter` metrics endpoint to OpenTelemetry Collector.

#### Associating `node_exporter` metrics with Slurm exporter metrics

Slurm's built-in metrics exporter surfaces the name of node that the metrics were collected from.
For example, Slurm's metrics exporter will report the amount of free memory on node `compute-0`.
However, the `node_exporter` by default surfaces the fully-qualified domain name of the node,
not the name that the node is registered under in Slurm. This naming mismatch makes it difficult
to write advanced Prometheus queries for detecting potential issues such as ghost processes running
on compute nodes when they have no allocated jobs.

The `prometheus_scrape` charm library can supply a custom scrape configuration that adds the node
name to metrics collected:

```python
# Assume `SlurmdCharm` is fully defined

import socket

from cosl import JujuTopology


class SlurmCharm(ops.CharmBase):
    def __init__(self, ...) -> None:
        topology = JujuTopology.from_charm(self)
        self.metrics_endpoint = MetricsEndpointProvider(
            self, 
            "metrics-endpoint",
            jobs=[
                {
                    # This job name is overwritten with "otelcol" when remote-writing
                    "job_name": f"juju_{topology.identifier}_node-exporter",
                    "scrape_interval": "60s",
                    "static_configs": [
                        {
                           "targets": [
                               f"0.0.0.0:{9011}"  # Port number associated with `node_exporter`
                           ],
                           "labels": {
                               "instance": socket.getfqdn(),
                               "juju_charm": topology.charm_name,
                               "juju_model": topology.model,
                               "juju_model_uuid": topology.model_uuid,
                               "juju_application": topology.application,
                               "node": self.unit.name.replace("/", "-")
                           },
                        }
                    ],
                }
            ]
        )
```

#### Installing `node_exporter` on `sackd` and `slurmd` units

The `node_exporter` snap will be installed by default on all `sackd` and `slurmd` units.
The utility of the metrics gathered from these units outweigh the total resource consumption
of the exporter service running on the machine.

## Further information

1. [`node_exporter` GitHub repository](https://github.com/prometheus/node_exporter)
2. [Writing great charmed alert rules](https://charmhub.io/alertmanager-k8s/docs/writing-great-alerts)
3. [Grafana "Alert list" visualization](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/visualizations/alert-list/)
4. [OpenTelemetry Collector on Charmhub](https://charmhub.io/opentelemetry-collector)
5. [`node-exporter` snap on the Snap Store](https://snapcraft.io/node-exporter)
6. [`otelcol_replica` in the OpenTelemetry Collector charm's _charmcraft.yaml_ file](https://github.com/canonical/opentelemetry-collector-operator/blob/d8293400c2669144a03d198424beeda4b97e5b7d/charmcraft.yaml#L147-L149)
7. ["Integrate with COS" how-to for Charmed HPC's documentation](https://documentation.ubuntu.com/charmed-hpc/latest/howto/integrate/integrate-with-cos/)
8. [`metrics-endpoint` integration for OpenTelemetry Collector](https://charmhub.io/opentelemetry-collector/integrations#metrics-endpoint)
9. [`prometheus_scrape` integration interface](https://charmhub.io/integrations/prometheus_scrape)
10. [`prometheus_scrape` charm library](https://charmhub.io/prometheus-k8s/libraries/prometheus_scrape)
11. [Prometheus scrape configuration documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)