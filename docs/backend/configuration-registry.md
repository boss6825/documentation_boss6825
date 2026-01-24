---
myst:
  html_meta:
    "description": "The Configuration Registry in Plone provides a central system for storing and managing site-wide settings using plone.registry and plone.app.registry."
    "property=og:description": "The Configuration Registry in Plone provides a central system for storing and managing site-wide settings using plone.registry and plone.app.registry."
    "property=og:title": "Configuration Registry"
    "keywords": "Plone 6, Configuration Registry, plone.registry, plone.app.registry, GenericSetup, settings, control panel"
---

(backend-configuration-registry-label)=

# Configuration Registry

The Configuration Registry in Plone is a central system for storing and managing site-wide settings and configuration options.
It allows developers and administrators to define, store, and retrieve configuration data in a structured way.

If you have the Manager role, you can directly view and change many variables that influence Plone through the {guilabel}`Configuration Registry` control panel in {guilabel}`Site Setup`.
Because there is a large number of settings, you can filter them or select a prefix to find the right one.
Add-on packages can create their own prefixes for their settings.

```{warning}
Setting variables directly in the Configuration Registry is not recommended as part of regular maintenance.
It is a useful tool for inspecting variables for expert users.
```

The Configuration Registry is built on two main packages:

[`plone.registry`](https://pypi.org/project/plone.registry/)
:   The core package that provides the low-level registry implementation for storing settings.

[`plone.app.registry`](https://pypi.org/project/plone.app.registry/)
:   The Plone integration layer that provides control panel support, GenericSetup integration, and user interface components.


(backend-configuration-registry-plone-registry-label)=

## `plone.registry`

The `plone.registry` package provides a debatably simple, persistent (ZODB-based) system for managing configuration settings.
It uses `zope.schema` fields to define the types and constraints for settings.

### Key concepts

Registry
:   The main storage container for all settings.
    In Plone, you access it as a utility: `from plone.registry import getUtility; registry = getUtility(IRegistry)`.

Records
:   Individual settings stored in the registry.
    Each record has a name (a dotted name string), a field (a `zope.schema` field), and a value.

Interfaces
:   You can register a `zope.interface` Interface with the registry.
    The registry will then contain one record for each field in the interface.


### Accessing registry values

You can access registry values programmatically in several ways.

#### Direct access by record name

```python
from plone.registry.interfaces import IRegistry
from zope.component import getUtility

registry = getUtility(IRegistry)

# Get a value directly
value = registry["my.package.setting"]

# Set a value
registry["my.package.setting"] = "new value"
```

#### Access through an interface

When you register an interface with the registry, you can use `registry.forInterface()` to get a proxy object that provides attribute access to the settings.

```python
from plone.registry.interfaces import IRegistry
from zope.component import getUtility
from my.package.interfaces import IMySettings

registry = getUtility(IRegistry)
settings = registry.forInterface(IMySettings)

# Access settings as attributes
value = settings.my_setting
settings.my_setting = "new value"
```


(backend-configuration-registry-plone-app-registry-label)=

## `plone.app.registry`

The `plone.app.registry` package integrates `plone.registry` with Plone.
It provides:

-   GenericSetup import and export handlers for registry settings
-   A control panel base class for creating settings forms
-   The {guilabel}`Configuration Registry` control panel for viewing and editing settings
-   Browser views for managing the registry