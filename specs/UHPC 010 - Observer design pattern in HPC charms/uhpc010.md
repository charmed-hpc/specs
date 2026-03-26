---
index: UHPC010
title: Observer design pattern in HPC charms
---

# Observer design pattern in HPC charms

## Abstract

This specification proposes implementing the observer pattern in the HPC charms to:

1. Reduce the complexity of the primary charm object in _charm.py_.
2. Improve the logical organization of HPC charms. One individual file can map to 
   one domain.
3. Streamline the process for making independent changes to how an HPC charm handles 
   events from a specific integration without having to modify the entire charm.
4. Simplify adding and removing integrations from the HPC charms.
5. Define common typing patterns for HPC charms so that expected public attributes 
   and methods are enforced by the type checker.

## Rationale

As part of implementing the changes proposed by specification UHPC008, it was determined that
the _charm.py_ file in some of the HPC charms are becoming unwieldy as the number of
event handlers increases. This unwieldiness makes it unnecessarily onerous to roll out updates
or maintain complex charms such as `slurmctld` which integrates with many other charmed 
applications. The visual complexity of the _charm.py_ file also makes it unnecessarily onerous
to determine how a change to one event handler or helper method may affect the behavior
of the entire charm.

The observer design pattern can be used to address these challenges.
Observers can logically organize event handlers and reduce the visual complexity of the
_charm.py_ file. Forward references can define expected APIs for charm object. This way
a type checker like `pyright` can be used to protect against changes to public attributes
or methods that accidentally break observers.

## Specification

The sections below outline how the observer design pattern with forward references will be
implemented in the HPC charms, using the `slurmctld` charm as an example.

### Scope of observers

Observer classes will be one-to-one; one observer will observe one domain. 

A domain is something in a charm that emits events that can be observed. 
Examples of domains include integrations or a particular subset of actions. An observer 
named `SlurmctldObserver` would observe events related to the `slurmctld` integration. 
The `slurmctld` integration would is the domain in this example. Observers for actions
can observe logical groups of actions such as key rotation or compute node management,
however, these observers will have a more flexible scope compared to integration observers
that will be decided by maintainers during implementation.

### Passing forward references to observers

Forward references will be used to declare the expected public API of the primary charm 
class when it is passed by reference to an observer. For example, a forward reference 
can be passed to an observer in the `slurmctld` charm to declare the expected API
of the `SlurmctldCharm` class:

```python
# In observer/sackd.py

from typing import TYPE_CHECKING

import ops

if TYPE_CHECKING:
   from charm import SlurmctldCharm


class SackdObserver(ops.Object):
    """Observe `sackd` integration events."""
   
    def __init__(self, charm: "SlurmctldCharm") -> None:
        self._charm = charm
        self._charm.slurmctld.do_something()
```

In the example above, if the `SlurmctldCharm` does not have a public `slurmctld` attribute, then
a type checker like `pyright` will emit a `reportAttributeAccessIssue` error. This mechanism
can be used to guarantee that a primary charm object will have the public attributes and
methods required by an observer, and preemptively catch errors if the public attributes of a
charm are modified.

> [!NOTE]
> The `if TYPE_CHECKING: ...` block is required to prevent circular import errors during runtime.
> This block allows forward references to be used properly when type checking with a static type
> checking tool like `pyright`.

### Defining observers

An observer will accept only one argument; the charm object that it is observing: 

```python
# In observer/sackd.py

__all__ = ["SACKD_INTEGRATION_NAME", "SackdObserver"]

from typing import TYPE_CHECKING

import ops

if TYPE_CHECKING:
   from charm import SlurmctldCharm

SACKD_INTEGRATION_NAME = "login-node"


class SackdObserver(ops.Object):
    """Observe `sackd` integration events."""

    def __init__(self, charm: "SlurmctldCharm") -> None:
        self._charm = charm
        self.sackd = SackdRequirer(charm, SACKD_INTEGRATION_NAME)

    def _on_sackd_connected(self, event: SackdConnectedEvent) -> None:
        self.sackd.do_something()
```

The name of the integration being observed will be declared as a constant in the observer module.
This constant should be imported by other modules if access to the integration name is required.

#### Observer class naming scheme

Observer class names will use a common postfix to indicate that the class is an observer.
The naming scheme is:

```python
<Domain>Observer
```

where `<Domain>` is the name of the domain that the observer is observing. For example,
the observer class names for the `sackd`, `slurmctld`, `slurmd`, `slurmdbd`, and `slurmrestd`
integrations will be as follows:

1. `SackdObserver`
2. `SlurmctldObserver`
3. `SlurmdObserver`
4. `SlurmdbdObserver`
5. `SlurmrestdObserver`

#### Evaluated alternatives
                                                                                
Enforcing the single argument passed to `__init__` using a generic `Observer` class that inherits
from `typing.Protocol`/`abc.ABC` was considered. This approach was deemed not viable because 
all the major static type checker implementations such as `pyright` and `mypy` do not enforce 
type signatures for the `__init__` method since it is usually overridden by child classes in 
an incompatible way.

A possible workaround was to declare an alternative constructor for the `Observer` class and then
call the alternative constructor from the parent `__init__` method:

```python
from abc import abstractmethod
from typing import Protocol

import ops


class Observer(Protocol):
    def __init__(self, charm: ops.CharmBase) -> None:
        self.observe(charm)

    @abstractmethod
    def observe(self, charm: ops.CharmBase) -> None:
        raise NotImplementedError


class SackdObserver(Observer):
    def observe(self, charm: ops.CharmBase) -> None:
        self._charm = charm
```

While the static type checker would flag if `observe` was overridden incorrectly, other issues
would be raised such as instance attributes being declared outside the `__init__` method. Also,
using an alternative constructor with a pluggable architecture to enforce type signatures is
clever, but it would make creating observers unnecessary complex. It's common practice in the
charming ecosystem to map event handlers to events in the `__init__` method, so the HPC charms
should not deviate from this convention.

It's easier and simpler to "enforce convention by code review" where this specification is 
cited when reviewing a proposed observer class, and evaluating if the proposed observer is
compliant with this specification.

### Organizing observers

Observers will be placed under the `src/observers/` directory in a charm repository. For example,
the `slurmctld` charm's integration observers will be organized as follows:

```shell
observers/
├── __init__.py
├── ha.py
├── sackd.py
├── slurmdbd.py
├── slurmd.py
├── slurm-oci-runtime.py
└── slurmrestd.py
```

The observer classes will be exposed using `__all__` in the *\_\_init\_\_.py* files:

```python
# In __init__.py

__all__ = [
   "SackdObserver",
   "SlurmdObserver",
   "SlurmdbdObserver",
   ...
]
```

This way the observer classes can be unified under one import statement in the _charm.py_ file:

```python
# In charm.py

from observers import ...
```

#### Evaluated alternatives

Using a flat `src/` structure with observer class files identified by the common postfix 
*\*_observer.py* was considered, but was decided against since having a *\*_observer.py*
file for each observer class would unnecessarily bloat the `src/` directory. It would
also make it more difficult to easily discover other charm functionality such as configuration
or state management since more files would have to be sifted through.

An `observers/` directory makes it easier to quickly identify how many observers a charm has.

## Further information

1. ["Forward reference" in PEP 484](https://peps.python.org/pep-0484/#forward-references)
2. [`pyright` diagnostic settings](https://microsoft.github.io/pyright/#/configuration?id=diagnostic-settings-defaults)
3. [ISD014 on Charmhub](https://discourse.charmhub.io/t/specification-isd014-managing-charm-complexity/11619?u=nuccitheboss)