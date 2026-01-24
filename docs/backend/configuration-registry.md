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


### Creating settings for your add-on

To create settings for your add-on, you need to:

1.  Define a schema interface for your settings
2.  Register the interface with the registry using GenericSetup
3.  Optionally create a control panel to edit the settings


#### Define a settings interface

Create an interface that defines your settings using `zope.schema` fields.

```python
# In your package's interfaces.py
from zope import schema
from zope.interface import Interface


class IMyPackageSettings(Interface):
    """Settings for my package."""

    enable_feature = schema.Bool(
        title="Enable feature",
        description="Turn this feature on or off",
        default=True,
    )

    max_items = schema.Int(
        title="Maximum items",
        description="The maximum number of items to display",
        default=10,
        min=1,
        max=100,
    )

    admin_email = schema.TextLine(
        title="Admin email",
        description="Email address for notifications",
        required=False,
    )
```


#### Register settings with GenericSetup

Create a `registry.xml` file in your package's `profiles/default` directory.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<registry>
    <records interface="my.package.interfaces.IMyPackageSettings"
             prefix="my.package">
        <!-- Set default values (optional) -->
        <value key="enable_feature">True</value>
        <value key="max_items">10</value>
        <value key="admin_email"></value>
    </records>
</registry>
```

The `prefix` attribute determines the dotted name prefix for the records.
With `prefix="my.package"`, the `enable_feature` field becomes `my.package.enable_feature` in the registry.


#### Access settings in your code

```python
from plone.registry.interfaces import IRegistry
from zope.component import getUtility
from my.package.interfaces import IMyPackageSettings


def get_settings():
    registry = getUtility(IRegistry)
    return registry.forInterface(IMyPackageSettings)


def is_feature_enabled():
    settings = get_settings()
    return settings.enable_feature
```


### Creating a control panel

To provide a user interface for editing your settings, create a control panel.

```python
# In your package's controlpanel.py
from plone.app.registry.browser.controlpanel import ControlPanelFormWrapper
from plone.app.registry.browser.controlpanel import RegistryEditForm
from plone.z3cform import layout
from my.package.interfaces import IMyPackageSettings


class MyPackageSettingsForm(RegistryEditForm):
    """Control panel form for my package settings."""

    schema = IMyPackageSettings
    label = "My Package Settings"
    description = "Configure settings for my package"


MyPackageSettingsView = layout.wrap_form(
    MyPackageSettingsForm,
    ControlPanelFormWrapper
)
```

Register the view and control panel in ZCML.

```xml
<configure xmlns="http://namespaces.zope.org/zope"
           xmlns:browser="http://namespaces.zope.org/browser">

    <browser:page
        name="my-package-settings"
        for="Products.CMFPlone.interfaces.IPloneSiteRoot"
        class=".controlpanel.MyPackageSettingsView"
        permission="cmf.ManagePortal"
        />

</configure>
```

Register the control panel configlet in `profiles/default/controlpanel.xml`.

```xml
<?xml version="1.0"?>
<object name="portal_controlpanel"
        xmlns:i18n="http://xml.zope.org/namespaces/i18n"
        i18n:domain="my.package">

    <configlet
        title="My Package Settings"
        action_id="my.package.settings"
        appId="my.package"
        category="Products"
        condition_expr=""
        url_expr="string:${portal_url}/@@my-package-settings"
        icon_expr=""
        visible="True"
        i18n:attributes="title">
            <permission>Manage portal</permission>
    </configlet>

</object>
```

```{seealso}
See the chapter {doc}`/backend/control-panels` for more information on creating control panels.
```


