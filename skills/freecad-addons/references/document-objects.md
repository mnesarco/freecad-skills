# FeaturePython document objects and migrations

This reference covers parametric objects that live in `.FCStd` documents. The
same object may be loaded years later, on another FreeCAD version, with the
addon updated. Treat the class path and property schema as a public API.

## 1. The minimal object

A document object has two halves:

1. A FreeCAD document object (`Part::FeaturePython`,
   `App::FeaturePython`, ...).
2. A Python proxy attached with `obj.Proxy = self`.

```python
class Box:
    def __init__(self, obj):
        obj.Proxy = self
        obj.addProperty(
            "App::PropertyLength", "Length", "Box", "Length of the box"
        ).Length = 10.0
        obj.addProperty(
            "App::PropertyLength", "Width", "Box", "Width of the box"
        ).Width = 10.0
        obj.addProperty(
            "App::PropertyLength", "Height", "Box", "Height of the box"
        ).Height = 10.0

    def execute(self, obj):
        import Part  # lazy: keep the proxy module import-light

        obj.Shape = Part.makeBox(
            float(obj.Length),
            float(obj.Width),
            float(obj.Height),
        )

    def __getstate__(self):
        return None

    def __setstate__(self, state):
        return None
```

Create it from a command:

```python
doc = App.activeDocument()
obj = doc.addObject("Part::FeaturePython", "Box")
Box(obj)
obj.ViewObject.Proxy = 0  # use FreeCAD's default Part ViewProvider
doc.recompute()
```

`Part::FeaturePython` gives the object a `Shape` property. `App::FeaturePython`
is the minimal container when you do not need BRep geometry. The object type
must be a Python-enabled type (it ends with `Python`); other examples are
`PartDesign::FeatureAdditivePython`, `PartDesign::FeatureSubtractivePython`,
`Mesh::FeaturePython`, and `Points::FeaturePython`.

## 2. Where the proxy lives

- Put each proxy in its own module under `freecad/<Mod>/`.
- The module path and class name are saved in the document. They must remain
  importable forever, or old documents lose their parametric behavior.
- Do not put the proxy in `init_gui.py`, a command module that is imported only
  on activation, or a module that imports Qt/Coin. FreeCAD must be able to
  import the proxy in console mode when opening a document.
- Use a dedicated module, not a class defined inside a function or command.

Good:

```text
freecad/MyAddon/box.py          -> class Box
freecad/MyAddon/box_view.py     -> class BoxViewProvider
```

Bad:

```python
# in a command's Activated()
class Box: ...
```

The class in the second example has no stable import path.

## 3. Properties

`obj.addProperty(type, name, group, description)` returns the object, so you can
set the initial value by chaining.

Common types:

| Type | Use |
|---|---|
| `App::PropertyBool` | checkbox |
| `App::PropertyInteger` | integer |
| `App::PropertyFloat` | unitless number |
| `App::PropertyLength` | length in document units |
| `App::PropertyDistance` | signed length |
| `App::PropertyAngle` | angle in degrees |
| `App::PropertyString` | text |
| `App::PropertyEnumeration` | fixed choices |
| `App::PropertyColor` | color |
| `App::PropertyVector` | 3D vector |
| `App::PropertyPlacement` | position + rotation |
| `App::PropertyLink` | reference to one object |
| `App::PropertyLinkList` | reference to several objects |
| `App::PropertyVectorList` | list of points |

### Property rules

- Property names and types are part of the saved schema. Choose them carefully
  and do not change them casually.
- Property descriptions are user-visible; wrap them with
  `QT_TRANSLATE_NOOP("App::Property", "...")`.
- Do not translate property names.
- Use property groups to keep the property panel readable.
- Hide internal properties with `obj.setEditorMode("SchemaVersion", 2)`.
- `App::PropertyEnumeration` needs its choices assigned before a value:

```python
obj.addProperty("App::PropertyEnumeration", "Mode", "Box", "Operation mode")
obj.Mode = ["Add", "Subtract", "Intersect"]
obj.Mode = "Add"
```

- Put persistent data in properties, not in the proxy. Proxy attributes are not
  a reliable database: they may not be saved, and they are not visible to users
  or other tools.
- Use `App::PropertyLink` / `PropertyLinkList` for object references, not raw
  object names. FreeCAD handles document links and relative links.
- Do not store Python objects, Qt objects, Coin nodes, or open file handles in
  properties.
- A descriptor layer may map a Python `snake_case` attribute to a FreeCAD
  `CamelCase` property name. Keep the stored name explicit and stable; renaming
  the Python attribute must not rename the saved property.

### Property modes and status

Properties can carry flags that control editing, persistence, and recompute:

| Flag | Effect |
|---|---|
| `ReadOnly` | Read-only in the property editor. |
| `Transient` | Not saved to the document; use for caches. |
| `Hidden` | Hidden in the property editor. |
| `Output` | Output-only; does not mark the parent container as changed. |
| `NoRecompute` | Changing it does not trigger a recompute. |
| `NoPersist` | Not persisted; use for runtime-only data. |

Pass the flags as the `attr` argument of `addProperty`, or use the runtime APIs:

```python
obj.addProperty("App::PropertyInteger", "SchemaVersion", "Internal", "", 4)  # Hidden
obj.setEditorMode("SchemaVersion", 2)  # 0 default, 1 read-only, 2 hidden
```

`setPropertyStatus`/`getPropertyStatus` control runtime status such as
`Immutable`, `NoModify`, `PartialTrigger`, `NoRecompute`, `Single`, `Ordered`,
`EvalOnRestore`, `CopyOnChange`, and `UserEdit`. Use them sparingly; they change
how FreeCAD treats the property.

To update an immutable property intentionally, temporarily remove the flag and
restore it in `finally`:

```python
def set_immutable(obj, name, value):
    was_immutable = "Immutable" in obj.getPropertyStatus(name)
    if was_immutable:
        obj.setPropertyStatus(name, "-Immutable")
    try:
        setattr(obj, name, value)
    finally:
        if was_immutable:
            obj.setPropertyStatus(name, "Immutable")
```

Do not hide or make properties transient just to avoid migrations. Users expect
their data to persist.

Property metadata can be inspected or changed at runtime:

```python
obj.getTypeIdOfProperty("Length")
obj.getGroupOfProperty("Length")
obj.setGroupOfProperty("Length", "Dimensions")
obj.getDocumentationOfProperty("Length")
obj.setDocumentationOfProperty("Length", "Length of the box")
obj.getEditorMode("Length")
obj.setEditorMode("Length", 1)  # read-only
obj.getPropertyStatus("Length")
obj.setPropertyStatus("Length", "NoRecompute")
obj.getEnumerationsOfProperty("Mode")
```

Use these for diagnostics or migrations; do not rename/retype properties at
runtime. Group/description changes are safe UI improvements; type changes are
schema changes.

Property-level change observers (a descriptor-layer convenience) still run
inside `onChanged`; use them for validation/invariants, not for rebuilding the
3D view. The ViewProvider's `updateData()` is the correct place for visual
updates.

### Extended property types

| Type | Use |
|---|---|
| `App::PropertyQuantity` | Generic quantity with a unit. |
| `App::PropertyFile` | External file path. |
| `App::PropertyFileIncluded` | File copied into the `.FCStd`; prefer for document data. |
| `App::PropertyPath` | Filesystem path. |
| `App::PropertyMap` | Small dictionary. |
| `App::PropertyStringList` | List of strings. |
| `App::PropertyVectorList` | List of points. |
| `App::PropertyLinkHidden` | Link not shown in the editor. |
| `App::PropertyLinkSub` | Object plus sub-element names. |
| `App::PropertyLinkSubList` | Several object/sub-element pairs. |
| `App::PropertyXLink` | Cross-document link. |
| `App::PropertyExpressionEngine` | Expression bindings. |

Rules:

- Use `PropertyFileIncluded` when the document must carry the data; use
  `PropertyFile`/`PropertyPath` only for external references and validate the
  path before reading.
- `PropertyMap` is for small structured data, not large blobs.
- `PropertyPythonObject` is powerful but fragile; prefer native properties or a
  JSON string with explicit validation.
- `PropertyEnumeration` choices must be assigned before a value. Dynamic option
  providers are useful, but refresh the choices when their dependencies change;
  do not leave a property pointing at a removed option.
- Properties support expressions automatically where appropriate; do not fight
  the expression engine or overwrite expression-bound values without a reason.

### Link properties and extensions

To behave like an `App::Link`, add `App::LinkExtensionPython` and mark the
properties that participate in the link:

```python
class MyLinkFeature:
    def __init__(self, obj):
        obj.Proxy = self
        obj.addExtension("App::LinkExtensionPython")

        obj.addProperty("App::PropertyLink", "SourceObject", "Link", "")
        obj.addProperty("App::PropertyPlacement", "Placement", "Link", "")

        obj.configLinkProperty(
            LinkedObject="SourceObject",
            Placement="Placement",
        )
```

- `configLinkProperty(**mapping)` maps the Link engine's property names to your
  property names.
- Mark linked properties with the Link extension; do not reimplement Link
  placement/array behavior manually.
- For link arrays, `PlacementList` is normally immutable until `ShowElement` is
  enabled. Toggle it with `setPropertyStatus("PlacementList", "-Immutable")` /
  `"Immutable"` as appropriate.
- `App::GroupExtensionPython` gives group behavior: a `Group` property plus
  `addObject`/`removeObject`; the ViewProvider should return the group contents
  from `claimChildren()`.
- `App::SuppressibleExtensionPython`, `App::GeoFeatureGroupExtensionPython`, and
  `App::OriginGroupExtensionPython` add their respective behaviors. Add only the
  extensions the object actually implements.
- ViewProvider extensions (`Gui::ViewProvider...`,
  `PartGui::ViewProviderAttachExtensionPython`) are separate from document-object
  extensions. Add both when the object needs both.

### Attachable and PartDesign features

`Part::AttachExtensionPython` makes a feature attachable:

```python
obj.addExtension("Part::AttachExtensionPython")
obj.positionBySupport()
```

PartDesign additive/subtractive features use `PartDesign::FeatureAdditivePython`
or `PartDesign::FeatureSubtractivePython`, plus the Attach extension. Their shape
logic must respect `Placement`, `BaseFeature`, and `AddSubShape`:

```python
def set_part_design_shape(obj, shape):
    if not obj.TypeId.startswith("PartDesign::"):
        raise ValueError("Not a PartDesign feature")

    if not obj.hasExtension("Part::AttachExtensionPython"):
        obj.addExtension("Part::AttachExtensionPython")
        obj.positionBySupport()

    shape.Placement = obj.Placement

    if base := getattr(obj, "BaseFeature", None):
        full_shape = (
            base.Shape.cut(shape)
            if "Subtractive" in obj.TypeId
            else base.Shape.fuse(shape)
        )
        full_shape.transformShape(obj.Placement.inverse().toMatrix(), True)
        obj.Shape = full_shape
    else:
        obj.Shape = shape

    if hasattr(obj, "AddSubShape"):
        shape.transformShape(obj.Placement.inverse().toMatrix(), True)
        obj.AddSubShape = shape
```

To create a PartDesign feature, find the active Body:

```python
view = Gui.ActiveDocument.ActiveView if Gui.ActiveDocument else None
body = view.getActiveObject("pdbody") if view else None
if body:
    feature = body.addObject(obj)
```

Do not copy this logic into every feature; keep one helper and test it against
additive, subtractive, and pattern cases.

## 4. `execute()`

`execute()` is called whenever the object is recomputed. It must:

- derive outputs (typically `obj.Shape`) from the current property values;
- be idempotent: same inputs, same output;
- not modify other document objects;
- not open transactions;
- not call `doc.recompute()`;
- not touch Qt, Coin, dialogs, or the 3D view;
- not perform network or subprocess work.

```python
def execute(self, obj):
    obj.Shape = Part.makeBox(
        float(obj.Length),
        float(obj.Width),
        float(obj.Height),
    )
```

Async recompute ([FreeCAD#21292](https://github.com/FreeCAD/FreeCAD/pull/21292), in development builds since
April 2026, enabled by default through the `EnableAsyncRecompute` preference)
can run a recompute request on a worker thread, but only when the request
target is worker-safe. `App::FeaturePython` objects are worker-unsafe by
default and stay on the GUI thread. A Python proxy opts in with:

```python
def supportsAsyncRecompute(self, obj):
    return True  # only if execute() never touches Qt, Coin or FreeCADGui
```

Opt in only when `execute()` is pure: no GUI, no view objects, no timers, no
event loop. The GIL is held for the whole recompute, so Python code stays
serialized, but GUI access from the worker fails. Anything that needs the GUI
belongs in the ViewProvider (see `references/pivy-coin.md`).

If a computation is expensive, cache by input values, not by wall-clock time.
Do not leave stale derived state when an input changes.

## 5. Object lifecycle and creation

Raw FeaturePython proxies use these callbacks:

| Callback | Use |
|---|---|
| `execute(obj)` | Recompute derived outputs. |
| `onBeforeChange(obj, prop)` | Capture old value/validate before a change. |
| `onChanged(obj, prop)` | React to a property change; keep it cheap. |
| `onDocumentRestored(obj)` | Migrate schema/state after load. |
| `unsetupObject(obj)` | Release resources when the object is removed. |
| `dumps()` / `loads(state)` | Serialize/restore proxy state in versions that use this pair. |
| `__getstate__()` / `__setstate__(state)` | Alternative serialization pair; use one convention consistently. |
| `getViewProviderName(obj)` | Override the ViewProvider type when needed. |

Some addon frameworks add conveniences such as `attach`, `on_create`, or
`on_start`; raw FreeCAD may call a different subset. Keep the standard callbacks
correct and do not rely on a convenience callback for correctness.

### Create an object safely

```python
def create_box(name="Box", label=None, doc=None):
    doc = doc or App.activeDocument() or App.newDocument()
    obj = doc.addObject("Part::FeaturePython", name)
    Box(obj)
    obj.Label = label or name

    if App.GuiUp and obj.ViewObject:
        obj.ViewObject.Proxy = 0  # or a custom ViewProvider

    obj.touch()
    doc.recompute()
    return obj
```

- Prefer an explicit document, then the active document, then a new document.
- Set both a unique internal `Name` and a user-visible `Label`.
- Bind the ViewProvider only when `App.GuiUp` and `ViewObject` exist.
- Recompute after creation so the initial shape is available.
- If the object needs an initial execution that FreeCAD does not trigger on
  creation, call the same code path used by `execute()` explicitly.

### State checks

- `obj.MustExecute` — the object is dirty and needs recompute.
- `obj.Removing` — the object is being removed; do not start new work.
- `obj.Document` — owning document.
- `obj.ViewObject` — GUI proxy, or `None` in console mode.
- `obj.PropertiesList` — current property names (use for migration checks).

### Rebind recovery

If a saved object loses its proxy (for example after a failed import), a recovery
helper can recreate the proxy and call `attach`/initialization. Use this only for
recovery or migration; normal creation should go through the object constructor.
Keep the class module path stable so FreeCAD can bind it automatically.

## 6. `onChanged()`

`onChanged(obj, prop)` is called on property changes. Use it for cheap
validation and invariants, not for heavy work.

```python
def onChanged(self, obj, prop):
    if not hasattr(obj, "Length"):
        return  # still being initialized
    if prop == "Length" and obj.Length < 0:
        obj.Length = 0
```

Rules:

- It is called during initialization, before all properties exist. Guard with
  `hasattr`.
- If `obj.Removing` is true, do not start new work.
- Keep it fast; it runs often.
- Do not trigger a full recompute from it unless necessary; setting properties
  marks the object for recompute.
- Do not use it to rebuild the Coin scene graph; that belongs in the
  ViewProvider's `updateData()`.

## 7. ViewProviders

The ViewProvider is the visual half of a document object: icon, display modes,
tree children, double-click behavior, and custom Coin geometry.

For plain `Part::FeaturePython` objects, use FreeCAD's default:

```python
obj.ViewObject.Proxy = 0
```

Write a custom ViewProvider only when you need:

- a custom tree icon;
- custom Coin/Coin3D rendering;
- tree children (`claimChildren`);
- custom double-click/edit behavior;
- custom display modes.

Put it in a separate module so console-mode imports of the document object do
not pull in GUI code:

```python
# freecad/MyAddon/box_view.py
from pathlib import Path


class BoxViewProvider:
    def __init__(self, vobj):
        vobj.Proxy = self

    def attach(self, vobj):
        self.Object = vobj.Object
        # build display modes / scene graph here

    def getIcon(self):
        return str(Path(__file__).parent / "resources" / "icons" / "box.svg")

    def claimChildren(self):
        return []

    def doubleClicked(self, vobj):
        return False

    def __getstate__(self):
        return None

    def __setstate__(self, state):
        pass
```

Attach it only in GUI mode:

```python
if App.GuiUp and obj.ViewObject:
    BoxViewProvider(obj.ViewObject)
```

- `attach(vobj)` is where the scene graph is created. `updateData(obj, prop)` is
  where it is updated. See `references/pivy-coin.md`.
- `getIcon()` returns an SVG path.
- `claimChildren()` returns document objects that should appear nested under
  this object in the tree. Do not include the object itself.
- `doubleClicked()` returns `True` if it handled the event.
- Keep `__getstate__`/`__setstate__` returning `None`/doing nothing unless the
  ViewProvider has real serializable state.
- Visual properties (color, transparency, line width) belong on
  `obj.ViewObject`, not on the document object, when they are purely visual.

ViewProvider API surface:

| Method | Purpose |
|---|---|
| `attach(vobj)` | Build display modes and the scene graph. |
| `updateData(obj, prop)` | React to a document property change (GUI thread). |
| `onChanged(vobj, prop)` | React to a ViewObject property change. |
| `getIcon()` | SVG path for the tree. |
| `getDisplayModes()` | List of display mode names. |
| `getDefaultDisplayMode()` | Mode used initially. |
| `setDisplayMode(mode)` | Apply a mode. |
| `setupContextMenu(vobj, menu)` | Add `QAction`s to the context menu. |
| `doubleClicked(vobj)` | Handle double-click; return `True` if handled. |
| `claimChildren()` | Objects nested under this object in the tree. |
| `setEdit(vobj, mode)` / `unsetEdit(vobj, mode)` | Enter/leave an edit mode. |
| `onDelete(vobj, sub_elements)` | Handle/confirm deletion. |
| `canDragObject(obj)` / `canDragObjects()` | Drag support. |
| `canDropObject(obj)` / `canDropObjects()` | Drop support. |
| `dragObject(vobj, obj)` / `dropObject(vobj, obj)` | Perform drag/drop. |

- Display modes are registered with `vobj.addDisplayMode(root_node, name)` in
  `attach()`. Build each mode's Coin root once; update nodes in `updateData()`.
- A context menu action should call a proxy method and be translated. Do not
  modify FreeCAD's core menus from a context-menu handler.
- Edit modes are for interactive tools (for example a task panel that edits the
  object). Return `True`/`False` consistently and close the task dialog.
- Drag/drop should be enabled only for objects that genuinely own/group others;
  perform the document change in a transaction.
- Visual properties (`ShapeColor`, `LineColor`, `Transparency`, `LineWidth`,
  `PointSize`, `DisplayMode`, `Visibility`) belong on `vobj`.

Detailed Coin and ViewProvider patterns: `references/pivy-coin.md`.

## 8. Serialization

FreeCAD saves:

- the object type (`Part::FeaturePython`, ...);
- the proxy module and class name;
- every property name/type/value;
- whatever `__getstate__` returned, passed back to `__setstate__` on load.

For most objects:

```python
def __getstate__(self):
    return None

def __setstate__(self, state):
    return None
```

Older addons may implement `dumps()` / `loads()`. Keep them if existing objects
rely on them; do not add both `dumps`/`loads` and `__getstate__`/`__setstate__`
to the same class. Do not put persistent data in proxy state when a property can
hold it.

Additional rules:

- Keep the serialized state small and versioned. A dictionary with a schema
  version is easier to migrate than a pickled object.
- Do not serialize Qt widgets, Coin nodes, file handles, or other runtime
  objects.
- The ViewProvider scene graph is rebuilt in `attach()` after load; it is not
  part of the document state.
- Test save/reopen with a real `.FCStd`, not only in-memory creation.
- If the proxy state changes shape between versions, migrate it in
  `onDocumentRestored()` before using it.
- A wrapper layer may expose `on_serialize`/`on_deserialize` hooks around the
  same state dictionary. Keep the state small and versioned; do not rely on
  wrapper-specific state being available in raw FreeCAD.

## 9. Migrations

`onDocumentRestored(obj)` runs after a saved document is loaded. It runs every
time the document is opened, so migrations must be idempotent.

### Add a property

```python
def onDocumentRestored(self, obj):
    if not hasattr(obj, "Depth"):
        obj.addProperty(
            "App::PropertyLength", "Depth", "Box", "Depth of the box"
        ).Depth = 5.0
```

`__init__` does not run for objects that already exist in an old document. This
is the most common migration bug.

### Rename a property

```python
def onDocumentRestored(self, obj):
    if hasattr(obj, "Length") and not hasattr(obj, "BoxLength"):
        old_value = obj.Length
        obj.addProperty(
            "App::PropertyLength", "BoxLength", "Box", "Length of the box"
        )
        obj.BoxLength = old_value
        obj.removeProperty("Length")
```

Log the old value before discarding it; never silently drop user data.

### Change a property type

```python
def onDocumentRestored(self, obj):
    if obj.getTypeIdOfProperty("Length") == "App::PropertyFloat":
        old_value = float(obj.Length)
        obj.removeProperty("Length")
        obj.addProperty(
            "App::PropertyLength", "Length", "Box", "Length of the box"
        )
        obj.Length = old_value
```

Type changes are intrusive. Test with a real old document.

### Schema version

For objects that will evolve repeatedly, add a hidden integer:

```python
SCHEMA_VERSION = 3


def __init__(self, obj):
    obj.Proxy = self
    obj.addProperty(
        "App::PropertyInteger", "SchemaVersion", "Internal", ""
    ).SchemaVersion = SCHEMA_VERSION
    obj.setEditorMode("SchemaVersion", 2)
    # ... other properties ...


def onDocumentRestored(self, obj):
    if not hasattr(obj, "SchemaVersion"):
        obj.addProperty(
            "App::PropertyInteger", "SchemaVersion", "Internal", ""
        ).SchemaVersion = 0
        obj.setEditorMode("SchemaVersion", 2)

    while obj.SchemaVersion < SCHEMA_VERSION:
        self._migrate_one_step(obj)
        obj.SchemaVersion += 1
```

Each `_migrate_one_step` handles one version transition. Do not skip versions;
users may upgrade from any older release.

### Explicit migration hooks

For larger objects, separate the migration phases instead of putting everything
in `onDocumentRestored`. These `on_migrate_*` methods are addon-defined hooks;
FreeCAD only calls `onDocumentRestored`.

```python
CURRENT_VERSION = 3


def onDocumentRestored(self, obj):
    # If the proxy class itself changed, recreate/rebind it first.
    version = getattr(obj, "SchemaVersion", 1)
    if version < CURRENT_VERSION:
        self.on_migrate_upgrade(obj, version, CURRENT_VERSION)
    elif version > CURRENT_VERSION:
        self.on_migrate_downgrade(obj, version, CURRENT_VERSION)


def on_migrate_upgrade(self, obj, from_version, to_version):
    # Apply one version step at a time; update SchemaVersion after each step.
    ...


def on_migrate_downgrade(self, obj, from_version, to_version):
    # Best effort; log what cannot be preserved.
    ...


def on_migrate_error(self, obj, error):
    App.Console.PrintError(f"Migration failed: {error}\n")


def on_migrate_complete(self, obj):
    App.Console.PrintLog("Migration complete\n")
```

- Run one version step at a time.
- Update the stored version only after a successful step.
- Log the old/new version and any discarded data.
- On failure, leave the object loadable and report the error; do not silently
  corrupt it.
- Test migration from every supported old version, not just the previous one.

### Rename or move the proxy class

The saved document remembers `module.ClassName`. Renaming or moving it breaks
old documents unless you:

1. **Keep the old module path as an alias** (preferred):

```python
# old location: freecad/myaddon/box.py
from freecad.myaddon.shapes.box import Box  # noqa: F401
```

2. **Or migrate at load time** by recreating the proxy when binding fails. This
   is more involved and only worth it when aliasing is impossible.

A top-level legacy module may not be importable in the modern layout, because
the repository root is not on `sys.path`. Test opening a real old `.FCStd`
before and after the move; do not assume an alias works.

## 10. Transactions and document safety

Commands that modify document objects must wrap the work in a transaction.
Prefer a context manager so a forgotten branch cannot leave the document
mid-transaction:

```python
from contextlib import contextmanager


class TransactionAbort(Exception):
    pass


@contextmanager
def transaction(name, doc=None):
    doc = doc or App.activeDocument()
    if doc is None:
        raise ValueError("No active document")

    doc.openTransaction(name)
    try:
        yield
    except TransactionAbort:
        doc.abortTransaction()
    except Exception:
        doc.abortTransaction()
        raise
    else:
        doc.commitTransaction()
```

```python
doc = App.activeDocument()
with transaction("Create box", doc):
    obj = doc.addObject("Part::FeaturePython", "Box")
    Box(obj)
doc.recompute()
```

- Keep one user action in one transaction.
- Commit before recomputing when practical.
- Do not leave a half-created object after an error.
- Do not modify the document from `execute()`.
- If a command changes many objects, validate inputs before opening the
  transaction when possible.
- If the operation is long, show progress; do not call `processEvents()` inside
  the transaction.
- To cancel intentionally, raise `TransactionAbort` inside the `with` block;
  do not return a half-finished state.

See `references/events-and-lifecycle.md` for transaction observers and
`references/preferences-and-widgets.md` for the progress indicator pattern.

## 11. Testing document objects

### Pure logic

Keep geometry-independent logic (parameter validation, path handling, schema
migration decisions, parsers) in modules that do not import FreeCAD at module
scope. Test with `pytest` outside FreeCAD.

### In-FreeCAD tests

Use `unittest` and register modules with `FreeCAD.__unit_test__` for console
tests, or `FreeCADGui.__unit_test__` for tests that need a Qt event loop:

```python
import unittest

import FreeCAD


class TestBox(unittest.TestCase):
    def setUp(self):
        self.doc = FreeCAD.newDocument(self.__class__.__name__)
        FreeCAD.setActiveDocument(self.doc.Name)

    def tearDown(self):
        FreeCAD.closeDocument(self.doc.Name)

    def test_volume(self):
        obj = self.doc.addObject("Part::FeaturePython", "Box")
        Box(obj)
        obj.Length = 10.0
        obj.Width = 20.0
        obj.Height = 30.0
        self.doc.recompute()
        self.assertAlmostEqual(obj.Shape.Volume, 6000.0, delta=1.0)
```

Run headless:

```bash
freecadcmd --run-test freecad.MyAddon.tests.test_box
```

Test geometry with tolerances, not exact equality. Bounding box and volume are
cheap sanity checks.

### Migration tests

Keep a small fixture of `.FCStd` files created by older addon versions. Open each
with the current code and check:

- the proxy binds;
- properties exist with the expected types;
- migrated values are preserved;
- `execute()` produces a valid shape;
- no data is silently dropped.

## 12. Common document-object mistakes

- Adding a property only in `__init__` and expecting old documents to get it.
- Renaming/moving a proxy class and breaking old files.
- Storing persistent data in the proxy instead of a property.
- Touching Qt or Coin from `execute()`.
- Calling `doc.recompute()` inside `execute()`.
- Modifying other document objects from `execute()`.
- Opening transactions in `execute()`.
- Forgetting `doc.recompute()` after creating an object.
- Setting `obj.Label` when you need a unique `obj.Name` (or vice versa).
- Adding a ViewProvider in console mode, where `obj.ViewObject` is `None`.
- Not guarding `onChanged()` with `hasattr` during initialization.
- Removing a property without a migration.
- Using `Part.show()` for a custom parametric object instead of
  `Part::FeaturePython`.
- Leaving `ViewObject.Proxy` unset for custom visuals.
- Not testing an old `.FCStd` after a schema change.
- Using `Transient`/`NoPersist` for data the user expects to keep.
- Reusing a property name with a different type instead of migrating.
- Overusing `PropertyPythonObject` for structured data.
- Adding `App::LinkExtensionPython` but forgetting `configLinkProperty`.
- Adding `Part::AttachExtensionPython` but forgetting `positionBySupport()`.
- Ignoring `BaseFeature`/`AddSubShape` in a PartDesign feature.
- Changing an `Immutable` property without restoring the status flag.
- Serializing Qt/Coin objects or large runtime state in the proxy.
- Mixing `dumps`/`loads` and `__getstate__`/`__setstate__` in one class.
- Running migrations only from the immediately previous version.
- Leaving a test document open when an assertion fails.
