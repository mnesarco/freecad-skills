# Pivy / Coin3D scene graph best practices

Pivy is FreeCAD's Python binding for Coin3D. It is bundled with FreeCAD; do not
add it as a dependency. Use it only in GUI code, normally inside a custom
ViewProvider.

```python
from pivy import coin
```

`coin` is the Coin3D API: `SoSeparator`, `SoMaterial`, `SoCoordinate3`,
`SoIndexedFaceSet`, `SoTransform`, `SoCamera`, `SbVec3f`, `SbRotation`, and so
on.

## 1. Where Pivy belongs

- **GUI only.** Do not import `pivy` in `__init__.py`, `core/`, a document
  object proxy, or anything that `FreeCADCmd` imports.
- **ViewProvider only.** Build and update the scene graph in the ViewProvider,
  not in the document object's `execute()`.
- **Lazy import if unsure.** If a module may be imported in console mode, import
  `pivy` inside the method that needs it.
- **No `execute()` access.** FreeCAD 1.x may recompute documents in worker
  threads. Coin is not thread-safe, and `execute()` may run off the GUI thread.
- **No Qt from workers.** Same rule for widgets; marshal to the GUI thread with
  signals or `QTimer.singleShot`.

## 2. Minimal custom ViewProvider

```python
# freecad/MyAddon/line_view.py
from pathlib import Path

from pivy import coin

_ICON = str(Path(__file__).parent / "resources" / "icons" / "line.svg")


class LineViewProvider:
    def __init__(self, vobj):
        vobj.Proxy = self
        self.root = None
        self.coords = None
        self.lines = None

    def attach(self, vobj):
        self.root = coin.SoSeparator()
        self.material = coin.SoMaterial()
        self.material.diffuseColor = (0.2, 0.6, 1.0)
        self.coords = coin.SoCoordinate3()
        self.lines = coin.SoIndexedLineSet()

        self.root.addChild(self.material)
        self.root.addChild(self.coords)
        self.root.addChild(self.lines)
        vobj.addDisplayMode(self.root, "Default")

    def updateData(self, obj, prop):
        if prop != "Points" or self.root is None:
            return
        pts = [(p.x, p.y, p.z) for p in obj.Points]
        self.coords.point.setValues(0, len(pts), pts)
        self.lines.coordIndex.setValues(
            0, len(pts), list(range(len(pts)))
        )

    def getDisplayModes(self, vobj):
        return ["Default"]

    def getDefaultDisplayMode(self):
        return "Default"

    def getIcon(self):
        return _ICON

    def __getstate__(self):
        return None

    def __setstate__(self, state):
        pass
```

The document object needs a property named `Points` (for example
`App::PropertyVectorList`) for `updateData()` to receive that change.

Lifecycle:

- `__init__`: store the ViewObject, set `vobj.Proxy = self`. Do not build the
  scene graph yet; `attach` is the correct place.
- `attach(vobj)`: create the root and display modes. It is called when the
  ViewProvider is attached, after `__init__`; build the scene graph here.
- `updateData(obj, prop)`: called when a document property changes; update the
  scene graph here. It may run before `attach`; guard with `self.root is None`.
- `onChanged(vobj, prop)`: for ViewObject properties (color, transparency,
  display mode). Keep it cheap.
- `getDisplayModes` / `getDefaultDisplayMode` / `setDisplayMode`: expose modes
  such as `"Default"`, `"Wireframe"`, `"Points"`.
- `getIcon`: return an SVG path.
- `claimChildren`: return child document objects for the tree.
- `doubleClicked`: return `True` if handled.
- `__getstate__` / `__setstate__`: keep `None`/empty unless the ViewProvider has
  real serializable state.

### Display modes

A ViewProvider can expose several display modes, each with its own Coin root:

```python
class BoxViewProvider:
    def attach(self, vobj):
        self.solid_root = build_solid_root()
        self.wire_root = build_wire_root()
        vobj.addDisplayMode(self.solid_root, "Default")
        vobj.addDisplayMode(self.wire_root, "Wireframe")

    def updateData(self, obj, prop):
        if prop in {"Length", "Width", "Height"}:
            update_solid(self.solid_root, obj)
            update_wire(self.wire_root, obj)

    def getDisplayModes(self, vobj):
        return ["Default", "Wireframe"]

    def getDefaultDisplayMode(self):
        return "Default"

    def setDisplayMode(self, mode):
        return mode
```

Rules:

- Build each mode's root once in `attach()`; update it in `updateData()`.
- `getDefaultDisplayMode()` must return one of the names from
  `getDisplayModes()`.
- Register every mode with `vobj.addDisplayMode(root, name)`.
- If modes only toggle visibility of the same geometry, one mode with an
  internal `SoSwitch` is simpler; if they show different geometry, use separate
  mode roots.
- `setDisplayMode` may be called before `attach()`; guard and return the mode
  name.
- Visual properties (`DisplayMode`, `Visibility`, colors) live on `vobj`.

### Context menu, edit modes, and drag/drop

ViewProvider callbacks for interactive features:

```python
def setupContextMenu(self, vobj, menu):
    action = menu.addAction("Edit box")
    action.triggered.connect(self.open_editor)


def setEdit(self, vobj, mode):
    if mode != 0:
        return False
    self.panel = BoxTaskPanel(self.Object)
    Gui.Control.showDialog(self.panel)
    return True


def unsetEdit(self, vobj, mode):
    if Gui.Control.activeDialog():
        Gui.Control.closeDialog()
    if Gui.ActiveDocument:
        Gui.ActiveDocument.resetEdit()
    return True
```

- Add only the actions the object actually supports; translate labels.
- The context menu is temporary; do not store the `QMenu` or its actions beyond
  the callback.
- Task panels provide `form`, `accept()`, and `reject()`; close the dialog and
  call `Gui.ActiveDocument.resetEdit()` on exit.
- For drag/drop, implement `canDragObject(s)`, `canDropObject(s)`,
  `dragObject(vobj, obj)`, and `dropObject(vobj, obj)`. Document changes go in a
  transaction; update `Group`/children accordingly.
- `claimChildren()` must match the actual ownership/grouping; it only affects the
  tree, not the document graph.
- `onDelete(vobj, sub_elements)` returns a boolean; do not delete unrelated
  objects.
- Do not modify FreeCAD's core menus from these callbacks; use the provided
  menu.

### Serialization and validity

- The Coin scene graph is not saved. Rebuild modes/nodes in `attach()` after a
  document is loaded.
- ViewProvider `__getstate__`/`__setstate__` should carry only small runtime
  state; visual settings belong in ViewObject properties.
- A cached `view`, `camera`, `dragger`, or widget can be deleted by C++ while
  Python still holds a wrapper. Check `shiboken.isValid(obj)` (or `shiboken2`
  on Qt5) before using it, and reacquire the active view when the document/view
  changes.
- Remove observers/dragger callbacks when the ViewProvider is replaced or the
  document closes; do not rely on `__del__`.

## 3. Coordinates, units, and primitives

FreeCAD uses **millimetres** and a **Z-up** world. Coin primitives use their own
local axes and sizes:

| Node | Default geometry | Local axis |
|---|---|---|
| `SoCube` | side length 2, centered at origin | — |
| `SoSphere` | radius 1, centered | — |
| `SoCylinder` | radius 1, height 2, centered | Y |
| `SoCone` | bottom radius 1, height 2, centered | Y, apex +Y |

Do not assume a cylinder points along Z or that a cube has side 1. Use
`SoTransform` to place and orient primitives.

Convert a FreeCAD `Placement` to `SoTransform`:

```python
def transform_from_placement(placement):
    transform = coin.SoTransform()
    transform.translation = (
        placement.Base.x,
        placement.Base.y,
        placement.Base.z,
    )
    transform.rotation = placement.Rotation.Q  # (x, y, z, w)
    transform.scaleFactor = (1.0, 1.0, 1.0)
    return transform
```

- `SoTransform` applies translation, rotation, scale, scale orientation, and
  center to its children.
- A `SoSeparator` around each object keeps its transform/material from leaking
  to siblings.
- `SoTranslation` is enough for a simple offset; `SoTransform` for full
  placement.

## 4. Scene graph patterns

### Root separator

Use `SoSeparator` as the display-mode root:

```python
root = coin.SoSeparator()
vobj.addDisplayMode(root, "Default")
```

Use `SoGroup` only when you deliberately want traversal state to leak; normally
you do not. `SoSeparator` also enables render caching.

### Materials and state

```python
material = coin.SoMaterial()
material.diffuseColor = (0.8, 0.2, 0.2)
material.transparency = 0.25
root.addChild(material)
```

- State nodes (`SoMaterial`, `SoTransform`, `SoDrawStyle`, `SoShapeHints`)
  affect all following children in the same separator. Put them before the
  geometry and isolate each object with its own separator.
- `SoBaseColor` is a lighter way to set a base color; `SoMaterial` gives
  specular, emissive, and transparency control.
- For unlit/constant-color overlays, use `SoLightModel` with `BASE_COLOR`.

### Indexed geometry

For lines:

```python
coords = coin.SoCoordinate3()
coords.point.setValues(0, len(points), points)  # points as (x, y, z)

lines = coin.SoIndexedLineSet()
# -1 separates polylines; for one polyline, 0..n-1 is enough
lines.coordIndex.setValues(0, len(indices), indices)
```

For faces:

```python
faces = coin.SoIndexedFaceSet()
# Each face is a sequence of vertex indices terminated by -1.
faces.coordIndex.setValues(0, len(indices), indices)
```

- `SoIndexedFaceSet` / `SoIndexedLineSet` are efficient for large meshes.
- `SoVertexProperty` can hold coordinates, normals, and colors in one node and
  can be attached as `indexed_shape.vertexProperty`.
- Batch with `setValues(start, count, sequence)`. Avoid `set1Value()` in a
  Python loop for thousands of points.
- For a simple polyline, `SoLineSet` with `SoCoordinate3` is enough.
- For points, use `SoPointSet` (optionally with `SoDrawStyle` for size).

### Normals and lighting

For closed solids, set shape hints and normals so backface culling and lighting
behave:

```python
hints = coin.SoShapeHints()
hints.vertexOrdering = coin.SoShapeHints.COUNTERCLOCKWISE
hints.shapeType = coin.SoShapeHints.SOLID
root.addChild(hints)
```

- Provide `SoNormal` vectors for smooth shading; use `SoNormalBinding` to say
  whether normals are per-vertex, per-face, or indexed.
- For flat faceted geometry, per-face normals plus `SoShapeHints` are usually
  enough.
- Wrong winding order makes faces invisible or lit incorrectly.

### Visibility and display modes

Use `SoSwitch` instead of adding/removing children:

```python
switch = coin.SoSwitch()
switch.addChild(solid_root)
switch.addChild(wire_root)
switch.whichChild = 0   # first child
switch.whichChild = -1  # none
```

- `whichChild = 0` selects the first child; `-1` hides all children.
- Use one switch per conceptual mode, not one switch per node.
- Keep all display-mode roots referenced on `self` so Python wrappers survive
  and can be updated.

### Overlays

For geometry that must be visible on top of the model:

```python
overlay = coin.SoAnnotation()
overlay.addChild(material)
overlay.addChild(coords)
overlay.addChild(lines)
```

- `SoAnnotation` renders after the normal scene, ignoring the depth buffer.
- `SoPolygonOffset` is better when the overlay is coplanar with a surface but
  should still respect depth:

```python
offset = coin.SoPolygonOffset()
offset.factor = 1.0
offset.units = 1.0
```

- Use `SoPickStyle` with `UNPICKABLE` for gizmos/overlays that should not be
  selected.
- Use `SoDrawStyle` with `LINES` or `POINTS` for debug geometry.

### Labels and text

- `SoText2` is screen-aligned and cheap; `SoText3` is 3D geometry and more
  expensive.
- Use `SoFont` for size/family and `SoAnnotation` for labels that must stay
  visible.
- For many labels, consider a single `SoText2` per label only when necessary;
  text nodes are relatively expensive.

### Heavy primitives

- `SoSphere`, `SoCylinder`, and `SoCone` tessellation is controlled by
  `SoComplexity`; lower it for decorative gizmos.
- Use `SoLOD` for large scenes with multiple detail levels.
- For production geometry, prefer a `Part::FeaturePython` `Shape` (BRep) over
  approximating solids with Coin primitives. Coin is for visualization and
  gizmos, not the authoritative model.

## 5. Updating the scene graph efficiently

- Build the graph once in `attach()`. Update fields in `updateData()`.
- Update only the nodes affected by the changed property. Do not
  `removeAllChildren()` and rebuild on every change unless the whole graph
  genuinely changed.
- Batch array updates with `setValues()`.
- Reuse `SoCoordinate3` / `SoIndexedFaceSet` nodes and overwrite their fields.
- Use `SoSwitch` for visibility instead of add/remove.
- If a separator's content changes every frame, turn off render caching for that
  separator; otherwise leave the default. Do not disable caching globally.
- Avoid per-frame Python callbacks and per-vertex Python objects in hot paths.
- Do not call `QApplication.processEvents()` while updating the scene graph or
  recomputing the document. It causes re-entrancy and can invalidate nodes.
- If a build is expensive, compute the data in a worker, then update Coin on the
  GUI thread.

## 6. Threading

Coin and Qt are **not thread-safe**. The rules:

- `execute()` may run on the recompute worker thread when the object opts in
  to async recompute (`supportsAsyncRecompute`, see `document-objects.md`); it
  must not touch Coin, Qt, or the view.
- `updateData()`, `onChanged()`, `attach()`, and command/UI callbacks run on the
  GUI thread and may touch Coin.
- To send data from a worker to the GUI, emit a signal of a `QObject` that
  lives on the GUI thread, or use `QtCore.QTimer.singleShot(0, gui_object,
  callback)` with such a context object (without one, the timer belongs to the
  worker thread and never reaches the GUI). Do not call widget or Coin methods directly from the worker.
- `FreeCAD.Console.Print*` is safe from worker threads.
- Do not hold a lock while calling Coin; it may call back into Python.

## 7. Memory, references, and observers

- Pivy objects wrap C++ reference-counted nodes. `addChild()` keeps the C++
  node alive, but if you need to update a node later, **keep a Python reference**
  on `self` (as in the example).
- `removeAllChildren()` releases C++ references; clear your Python references
  too if you rebuild.
- Do not store scene-graph nodes on the document object proxy. They are not
  serialized, and the document object must be importable in console mode.
- Do not rely on `__del__` for cleanup. It may run late or not at all,
  especially during interpreter shutdown.
- Remove observers explicitly:

```python
import FreeCAD as App
import FreeCADGui as Gui

Gui.Selection.addObserver(self)
Gui.Selection.removeObserver(self)
App.addDocumentObserver(self)
App.removeDocumentObserver(self)
```

- Keep one observer instance and remove the same object. Do not create a new
  observer on every activation.
- If a ViewProvider owns an observer/dragger, clean it up when the view
  provider is replaced or the document closes.
- C++ may delete a view/camera/dragger/widget while Python still holds a
  wrapper. Check `shiboken.isValid(obj)` (or `shiboken2` on Qt5) before using a
  cached reference. Reacquire the active view when the document/view changes.
- The Coin scene graph is not serialized; rebuild it in `attach()` on load.

## 8. Camera manipulation

Prefer FreeCAD's view API for standard operations:

```python
view = Gui.activeDocument().activeView()
view.viewTop()
view.viewAxonometric()
view.fitAll()
view.zoomIn()
Gui.SendMsgToActiveView("ViewFit")
```

If you must manipulate the Coin camera directly:

```python
view = Gui.activeDocument().activeView()
cam = view.getCameraNode()  # SoCamera

# Read a copy before mutating.
pos = coin.SbVec3f(cam.position.getValue())
rot = coin.SbRotation(cam.orientation.getValue())

# Assign a new value; do not mutate the value returned by getValue() in place.
cam.position = coin.SbVec3f(pos[0], pos[1], pos[2])
cam.orientation = coin.SbRotation(rot)
```

Rules:

- Reacquire the camera when the active view/document changes; do not hold an
  `SoCamera` across view switches or document close.
- `getValue()` on `SoSF*` fields returns a reference to the field's internal
  value. Copy `SbRotation` / `SbVec3f` before mutating.
- `SbRotation` composition order is easy to get wrong. Test with known vectors
  and `multVec()`; do not guess.
- Do not fight FreeCAD's navigation. Save and restore camera state only for a
  dedicated mode, and leave navigation usable.
- Use `view.getViewer().getSoRenderManager()` for render-manager data such as
  the viewport region.
- Do not call camera operations from a worker thread.

## 9. Picking and draggers

### Picking

- FreeCAD's selection uses Coin picking. Custom scene nodes are pickable by
  default.
- Use `SoPickStyle` with `UNPICKABLE` for overlays, decorations, and gizmos that
  should not be selectable.
- For custom hit-testing, use Coin actions (`SoRayPickAction`) or FreeCAD's
  selection API on the GUI thread. Do not implement picking with raw screen
  coordinates unless necessary.
- Return meaningful `SoPickedPoint` details only if you implement custom
  selection behavior; otherwise let FreeCAD handle it.

### Draggers

- Coin draggers (`SoTranslate1Dragger`, `SoRotateSphericalDragger`, ...) are
  interactive gizmos.
- Create them in the ViewProvider; keep references so they are not garbage
  collected.
- On drag start, open a document transaction; on drag finish, commit; on
  cancel/error, abort.
- Do not let a dragger modify document data directly outside a transaction.
- Remove or hide dragger nodes when switching display modes or deactivating the
  tool.
- Keep dragger callbacks cheap and on the GUI thread.

## 10. Testing Coin code

- Test pure data production (point lists, index lists, transforms) outside
  FreeCAD with `pytest`.
- Test the scene graph in a GUI test: create a document, add the object, attach
  the ViewProvider, recompute, and assert the expected children/fields exist.
- Run GUI tests with `FreeCAD --run-test`; `freecadcmd` has no view.
- Check that a saved document with the object still loads after a code change.
- Check transparent/overlay ordering visually; it is hard to assert
  automatically.
- Check that the scene graph is not rebuilt every recompute (performance).

## 11. Common Pivy/Coin pitfalls

- `SoGroup` instead of `SoSeparator`: material/transform state leaks to other
  objects.
- Assuming Y-up or unit primitives: FreeCAD is Z-up and primitives are
  Y-axis-aligned/centered.
- Mutating the object returned by `getValue()` instead of copying and assigning.
- Touching Coin from `execute()` or a worker thread.
- Rebuilding the entire graph on every property change.
- Forgetting `vobj.addDisplayMode(root, "Default")` — nothing appears.
- Losing Python references to nodes you need to update.
- Not guarding `updateData()` before `attach()`.
- Using `SoIndexedFaceSet` without `-1` face terminators.
- Wrong winding order causing invisible/black faces.
- Forgetting `SoShapeHints`/normals for lit solids.
- Hardcoding colors that break in dark themes.
- Using `__del__` for observer cleanup.
- Calling `QApplication.processEvents()` inside recompute.
- Holding a stale `SoCamera` after a view/document switch.
- Assuming `SbRotation` multiplication order without testing.
- Using Coin for authoritative geometry instead of `Part`/BRep where the model
  must be saved and recomputed.
- Returning a display mode name from `getDefaultDisplayMode()` that is not in
  `getDisplayModes()`.
- Rebuilding every display mode on every property change.
- Leaving a task panel open or forgetting `Gui.ActiveDocument.resetEdit()`.
- Implementing drag/drop without a transaction or without updating the group.
- Storing context-menu `QAction`s after the menu is destroyed.
- Using a stale C++ view/camera/dragger wrapper after deletion.
- Calling `processEvents()` or touching Coin from a worker thread.
