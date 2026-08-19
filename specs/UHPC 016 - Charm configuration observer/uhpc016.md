---
index: UHPC016
title: Charm configuration observer
---

# Charm configuration observer

## Abstract

This specification proposes implementing a charm configuration observer in
`charmed-hpc-libs` to provide a common interface for HPC charms using the
_conditions_ pattern to interact with application configuration data set 
using the `juju config` command.

The charm configuration observer implements the Observer design pattern
defined in specification UHPC010.

## Rationale

`ops` 2.23.0 introduced the `load_config` method on the `CharmBase` class. 
This `load_config` method provides a richer way to load charm application 
configuration data into a structured Python object such as a native Python 
dataclass or a data model constructed using the `pydantic` validation library. 

A benefit of using the `load_config` method instead of the `CharmBase` object's
`config` property is that `pydantic` data models can be used to provide richer
configuration value validation than what's provided by the `juju config` command.
`juju config` provides strict type validation - you can't provide a string
value for a configuration option that must be an integer - a `pydantic` dataclass/model
can be used to provide more granular validation on the provided value. For example,
while `juju config` can enforce that the value of a charm application's `port` 
configuration option is an integer, a `pydantic` dataclass/model can enforce that the
provided value is a specific or non-reserved port number:

```python
from typing import Annotated, Literal

import ops
from pydantic import Field
from pydantic.dataclasses import dataclass


@dataclass(frozen=True)
class ConfigData:
    """Charm configuration data."""

    port: Literal[22] | Annotated[int, Field(ge=1024, le=65535)]


# Assume charm is fully defined.
class OpenSSHCharm(ops.CharmBase):
    """SSH server charm."""

    def _on_config_changed(self, _: ops.ConfigChangedEvent) -> None:
        self.typed_config = self.load_config(ConfigData)
```

However, a "limitation" of `load_config` is that there is no standard way
to handle raised `ValueError`/`ValidationError` errors if a provided configuration
option value passes validation when set with `juju config`, but doesn't pass
validation when the configuration option value is parsed into a `pydantic` 
dataclass/model. Charm authors are therefore required to implement their
own error handling which can lead to edge-case bugs such as
[`canonical/slurm-charms#224`](https://github.com/canonical/slurm-charms/issues/224)
where configuration validation errors can cause charm execution loops
to return too early and skip critical events like `ops.InstallEvent`.

A configuration observer class, `ConfigObserver`, should be implemented in
`charmed-hpc-libs` to provide HPC charms with a standard way to access typed
configuration data easily with `load_config`, and handle both validation 
errors and event deferrals using the _conditions_ pattern.

## Specification

The sections below outline how the `ConfigObserver` class will be implemented
in `charmed-hpc-libs`, provide justifications for implementation decisions,
and examples on how `ConfigObserver` should be used in downstream HPC charms.

### Defining the `Observer` and `ConfigObserver` classes

The `Observer` class is conceptually similar to the `Interface` class
in `charmed-hpc-libs.interfaces` in that it provides the base set of attributes,
properties, and methods required for building a downstream Observer class.

```python
import ops

type _CharmType[T: ops.CharmBase] = T


class Observer(ops.Object):
    """Base observer for HPC observer implementations."""

    def __init__(self, charm: _CharmType) -> None:
        super().__init__(charm, f"{type(charm).__name__}")
        self._charm = charm

    @property
    def charm(self) -> _CharmType:
        """The charm object being observed."""
        return self._charm
```

This base `Observer` class can then be used in the annotation for hook functions
passed to the `refresh` decorator:

```python
import ops
from charmed_hpc_libs.ops import Observer, refresh


def check_charm(observer: Observer) -> ops.StatusBase:
    """Determine the current state of the charm."""

    if observer.charm.unit.is_leader():
        ...  # Do something.


refresh = refresh(hook=check_charm)
```

The benefit of having downstream `Observer` classes inherit from `Observer` is that:

1. It provides common base set of attributes, properties, and methods that generic
   `refresh` hooks and vendored charm modules can expect an Observer class to provide.
2. It encapsulates some peculiarities of the `ops` library such as requiring every
   class that inherits from `ops.Object` to have a unique identifier in the `ops.Framework`
   data structure.

`ConfigObserver` can then be a light wrapper around the `Observer` base class,
and provide a `load` method that acts as a standard interface for loading charm
application configuration data in HPC charms.

```python
class ConfigObserver[T](Observer):
    """Observe charm application configuration data set with ``juju config``."""

    def __init__(self, charm: _CharmType, config_cls: type[T]) -> None:
        super().__init__(charm)
        self._config_cls = config_cls

    def load(self) -> T:
        """Load charm application configuration data.

        Raises:
            StopCharm: Raised if the charm's application configuration fails validation.
        """
        try:
            return self._charm.load_config(self._config_cls)
        except ValidationError as e:
            failed_options = sorted({error["loc"][0] for error in e.errors() if error.get("loc")})
            message = (
                "configuration option(s) "
                + ", ".join(f"'{str(o).replace('_', '-')}'" for o in failed_options)
                + " failed validation"
            )
            _logger.exception(message)

            raise StopCharm(
                ops.BlockedStatus(f"{message.capitalize()}. See `juju debug-log` for details")
            ) from None
        except ValueError:
            # Handle if `_config_cls` is a native dataclass and not a `pydantic` object.
            message = "configuration option(s) failed validation"
            _logger.exception(message)

            raise StopCharm(
                ops.BlockedStatus(f"{message.capitalize()}. See `juju debug-log` for details")
            ) from None
```

The benefit of using the `load` method provided by `ConfigObserver` instead of calling
`load_config` directly is that the `load` method encapsulates the additional exception
handling required to surface validation errors in the blocked status message
format that Charmed HPC currently uses. 

In the implementation above, the `load` method intercepts two errors, `ValidationError` 
and `ValueError`, before raising a `StopCharm` exception. `ValidationError` is raised 
when a `pydantic` dataclass/model fails validation. `ValidationError` contains granular information 
on which configuration option(s) failed validation and why. `ValueError` is the parent class
of `ValidationError`, but does not provide as much information like which options failed 
validation. The `except` block for handling `ValueError` is for handling native Python 
dataclasses that implement their own validation mechanism in the `__post_init__` magic method:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class DataclassConfigData:
    """Native dataclass for charm application configuration data."""

    port: int

    def __post_init__(self) -> None:  # noqa D105
        if not (1 <= self.port <= 65535):
            raise ValueError(f"port must be in range, got {self.port}")
```

> [!NOTE]
>
> Charm configuration options modeled with Python's native dataclasses must provide 
> explicit validation steps and raise a `ValueError`, manually. `load_config` does not
> provide any additional validation utilities for native dataclasses.


### Is `ConfigObserver` an observer?

`ConfigObserver` qualifies as an Observer since the domain that it is observing is a
charm application's configuration data. A charm can either extend `ConfigObserver` to define 
an `_on_config_changed` event handler through inheritance, or the charm's Observers 
can access `ConfigObserver` through a attribute defined on the primary charm object.

#### Inheritance-based access to `ConfigObserver`

An Observer class can inherit from `ConfigObserver` if it needs to interface with
a charm application's configuration, or is defining the `_on_config_changed` event
handler. In the example below, a theoretical `LifecycleObserver` class inherits from
`ConfigObserver`, and uses the `load` method to load the charm's configuration data
in the `_on_config_changed` event handler:

> [!NOTE]
>
> `LifecycleObserver` is a theoretical Observer class whose observed domain is the
> core lifecycle events of a deployed charm like `InstallEvent`, `StartEvent`, 
> `ConfigChangedEvent`, `StopEvent`, and so on.

```python
# In operations/lifecycle.py

from typing import TYPE_CHECKING, Annotated, Literal

import ops
from charmed_hpc_libs.ops import ConfigObserver, refresh
from pydantic import Field
from pydantic.dataclasses import dataclass

if TYPE_CHECKING:
    from charm import MyCharm  # Assume `MyCharm` is predefined.


@dataclass(frozen=True)
class ConfigData:
    """Charm configuration data."""

    port: Literal[22] | Annotated[int, Field(ge=1024, le=65535)]


class LifecycleObserver(ConfigObserver):
    """Observe the core lifecycle events of a charm."""

    def __init__(self, charm: "MyCharm") -> None:
        super().__init__(charm, ConfigData)

        self.charm.framework.observe(charm.on.install, self._on_install)
        self.charm.framework.observe(charm.on.config_changed, self._on_config_changed)

    def _on_install(self, _: ops.InstallEvent) -> None: ...  # Install packages.

    @refresh(hook=None)
    def _on_config_changed(self, _: ops.ConfigChangedEvent) -> None:
        config = self.load()

        # Do stuff with `config.port`
```

In this implementation, `load` encapsulates the common error handling, message formatting, 
logging, and `StopCharm` emission pattern that is used in HPC charms. The failed 
`ConfigChangedEvent` is not deferred here since it can be assumed that the user will fix 
the incorrect configuration option with `juju config` and a new `ConfigChangedEvent` 
will be emitted.

#### Composition-based access to `ConfigObserver`

`ConfigObserver` can be accessed by another Observer by attaching `ConfigObserver` as an
attribute to the primary charm object. This mechanism is particularly useful when an integration
Observer needs to compare charm configuration data against integration data that the
integration Observer receives from a remote application. In the example below, `ConfigObserver` 
is attached to `MyCharm` through the `typed_config` attribute, and then the `SlurmdObserver` class
receives a reference to `MyCharm` with the `typed_config` attribute defined:

> [!NOTE]
>
> `SlurmdObserver` is a theoretical Observer class inspired by the `slurmd` integration
> endpoint on the `slurmctld` charm.

```python
# In charm.py

import ops
from charmed_hpc_libs.ops import ConfigObserver

# Assume `ConfigData` is predefined and has a `default_partition` attribute.
from config import ConfigData
from integrations import SlurmdObserver 


class MyCharm(ops.CharmBase):
    """My charm."""

    def __init__(self, framework: ops.Framework) -> None:
        super().__init__(framework)

        self.typed_config = ConfigObserver(self, ConfigData)
        self.slurmd_observer = SlurmdObserver(self)
```

Then, in the `SlurmdObserver` class, integration event handlers can use `ConfigObserver`
to load charm configuration data if the event handler needs to compare local configuration
data against received integration data:

```python
# In integrations/slurmd.py

from typing import TYPE_CHECKING

from charmed_hpc_libs.ops import Observer, StopCharm, refresh
from charmed_slurm_slurmd_interface import SlurmdReadyEvent, SlurmdRequirer

if TYPE_CHECKING:
    from charm import MyCharm


class SlurmdObserver(Observer):
    """Observe events associated with the ``slurmd`` integration."""

    def __init__(self, charm: "MyCharm") -> None:
        super().__init__(charm)
        
        self.slurmd = SlurmdRequirer(self.charm, "slurmd")
        self.charm.framework.observe(self.slurmd.on.slurmd_ready, self._on_slurmd_ready)

    @refresh(hook=None)
    def _on_slurmd_ready(self, event: SlurmdReadyEvent) -> None:
        """Handle when a new ``slurmd`` partition is ready."""
        try:
            config = self.typed_config.load()
        except StopCharm:
            event.defer()
            raise

        if config.default_partition == event.app.name:
            ... # Handle when the new partition is the default partition.
```

Unlike `LifecycleObserver`, `StopCharm` is caught and the `SlurmdReadyEvent` is explicitly
deferred before `StopCharm` is bubbled up to `refresh`. Having this finite control over
event deferrals is beneficial in Observers where certain events are "difficult" to repeat
such as re-emitting a  `RelationChangedEvent` for static integration data. However, like
the inheritance pattern, the benefits that the `load` method provides here are that:

1. It encapsulates the common error handling, message formatting, 
logging, and `StopCharm` emission pattern that is used in HPC charms.
2. A simple `try`/`except` block can be used to defer the failed event, and no
   additional handling is required around the raised `StopCharm` exception.

### Evaluated alternatives

#### Deferring events in `__init__`

The HPC charms currently load the typed charm configuration data in the `__init__` method
of charm objects. It was considered to use an `event` function like the one
implemented in `opentelemetry-collector` - which retrieves the value of either the 
`JUJU_HOOK_NAME` or `JUJU_ACTION_NAME` environment variable - and extend it to assemble the 
full `*Event` object and call `defer` if the `load_config` method failed. This approach was
initially favored since it would require the least amount of changes to the HPC charms,
however, this approach was determined to be unfeasible.

`ops`'s internal execution loop is what is responsible for constructing the `*Event` object,
and populates the created object with further information from the current hook context.
This execution loop also handles event deferrals. When `defer` is called on an `*Event` object,
it sets the attribute `deferred` to `True`. If `deferred` is `True`, the `ops` execution loop
snapshots the event and its data into a local SQLite database. This event snapshot is then
reloaded at the start of the next hook execution in the charm.

While `charmed-hpc-libs` _could_ reimplement this deferral mechanism, it would require
accessing non-public features of `ops`, and would require hijacking the `ops.main` entrypoint.

#### Set `typed_config` attribute in `_on_config_changed` event handler

Charms whose entire configuration operations are able to be encapsulated in a
single `_on_config_changed` event handler typically define a `typed_config` attribute
that holds the parsed configuration data. These charms rely on the `load_config` method's
built-in `errors` parameter. If `errors` is set to `blocked`, `load_config` sets a generic
blocked status message if the charm's configuration data fails validation.

This approach does not work for "complex" charms like the Slurm charms where a charm's
configuration data must be accessed in multiple event handlers. While `load_config`
could be called directly, it would introduce the burden of having to copy the same
error handling, message formatting, and exception logging everywhere it is invoked.
Reducing boilerplate was the original intention of calling `load_config` directly in the
`__init__` method.

Also, defining instance attributes like `typed_config` outside of an object's `__init__`
method is considered an anti-pattern since there is a possibility that the instance attribute
could be referenced before it is defined in another method.

#### Set a temporary attribute flag on the main charm object

Some charms set a private attribute after calling `load_config` to flag whether the configuration
successfully passed validation. This private attribute is then evaluated in a reconciliation
method like `_on_collect_app_status` to determine whether a "config invalid" status message 
should be set for the application.

While this approach wouldn't mandate handling a `StopCharm` exception, a similar "set the value
of a private attribute and evaluate it later in a reconciliation function" mechanism was 
originally used in the Slurm charms; however, it was refactored out. The private attribute
made it difficult to mentally track the logic of the Slurm charms, and it was difficult
to tell when and where the value of the private attribute was altered to modify the execution
flow of the reconciliation function.

## Further information

1. [Reference documentation for `ops.CharmBase.load_config`](https://canonical.com/juju/docs/ops/latest/_modules/ops/charm/#CharmBase.load_config)
2. [`pydantic` on GitHub](https://github.com/pydantic/pydantic)
3. [_conditions_ pattern in `charmed-hpc-libs`](https://github.com/canonical/charmed-hpc-libs/blob/main/src/charmed_hpc_libs/ops/conditions.py)
4. [`Interface` class in `charmed-hpc-libs.interfaces`](https://github.com/canonical/charmed-hpc-libs/blob/main/src/charmed_hpc_libs/interfaces/interface.py)
