# FreeCAD API map for addon developers

A compact index of the FreeCAD APIs addons commonly need. Names are the raw
FreeCAD Python API; a private helper library may wrap them, but the behavior is
the same. Verify exact signatures against `freecad-stubs` or the running
FreeCAD version before relying on an uncommon method.

## Documents (`FreeCAD as App`)

| API | Use |
|---|---|
| `App.newDocument(name)` | Create a document. |
| `App.openDocument(path)` | Open an existing document. |
| `App.activeDocument()` | Current document or `None`. |
| `App.getDocument(name)` | Look up a document by name. |
| `App.listDocuments()` | All open documents. |
| `App.closeDocument(name)` | Close a document. |
| `App.setActiveDocument(name)` | Make a document active (useful in tests). |
| `doc.addObject(type, name=None, proxy=None, view_proxy=None, attach=False)` | Create an object; prefer a helper that also binds proxies. |
| `doc.removeObject(name)` | Remove an object. |
| `doc.getObject(name)` | Find an object by internal name. |
| `doc.getObjectsByLabel(label)` | Find objects by user-visible label. |
| `doc.Objects` | All top-level objects. |
| `doc.ActiveObject` | Current active object, if any. |
| `doc.recompute()` | Recompute dirty objects. |
| `doc.openTransaction(name)` | Start an undo transaction. |
| `doc.commitTransaction()` | Commit the transaction. |
| `doc.abortTransaction()` | Roll back the transaction. |
| `doc.save()` / `doc.saveAs(path)` | Save the document. |
| `doc.FileName` | File path, if saved. |
| `doc.Name` / `doc.Label` | Internal name / user-visible label. |
| `doc.Modified` | Dirty flag. |
| `App.Vector`, `App.Rotation`, `App.Placement`, `App.Matrix`, `App.BoundBox` | Core geometry types. |
| `App.Units.Quantity` | Quantity with a unit. |

Use a transaction context manager in application code rather than repeating
`openTransaction` / `commit` / `abort` by hand (see
`references/events-and-lifecycle.md`).

## Document objects

| API | Use |
|---|---|
| `obj.TypeId` | Concrete FreeCAD type, e.g. `Part::FeaturePython`. |
| `obj.Name` / `obj.Label` | Stable internal name / user-visible label. |
| `obj.Document` | Owning document. |
| `obj.Proxy` | Python data proxy. |
| `obj.ViewObject` | GUI proxy, or `None` in console mode. |
| `obj.addProperty(type, name, group, doc, attr)` | Add a dynamic property. |
| `obj.removeProperty(name)` | Remove a property (migration only). |
| `obj.PropertiesList` | Names of all properties. |
| `obj.getTypeIdOfProperty(name)` | Property type string. |
| `obj.getGroupOfProperty(name)` / `setGroupOfProperty(name, group)` | Property panel group. |
| `obj.getDocumentationOfProperty(name)` / `setDocumentationOfProperty(name, text)` | Property tooltip. |
| `obj.getEnumerationsOfProperty(name)` | Choices of an enumeration. |
| `obj.setEditorMode(name, mode)` | Editor mode: `0` default, `1` read-only, `2` hidden. |
| `obj.setPropertyStatus(name, flag)` / `getPropertyStatus(name)` | Runtime status flags such as `Immutable`, `NoRecompute`, `Output`, `Hidden`. |
| `obj.hasExtension(name)` / `addExtension(name)` | Dynamic extensions. |
| `obj.configLinkProperty(**mapping)` | Configure linkable properties for `App::LinkExtensionPython`. |
| `obj.positionBySupport()` | Recompute placement for an attachable feature. |
| `obj.adjustRelativeLinks(parent)` | Keep links correct when reparenting. |
| `obj.getParent()` / `getParentGeoFeatureGroup()` | Parent/group lookup. |
| `obj.isDerivedFrom(type_id)` | Check base type. |
| `obj.getSubObject(sub)` / `getSubObjects()` | Resolve sub-elements/children. |
| `obj.addObject(child)` / `removeObject(child)` | Group-like containers. |
| `obj.MustExecute` | Object needs recompute. |
| `obj.Removing` | Object is being removed. |
| `obj.touch()` / `purgeTouched()` / `enforceRecompute()` | Mark changed/unchanged/force recompute. |
| `obj.recompute()` | Recompute this object only. |
| `obj.setExpression(name, expr)` / `clearExpression(name)` | Bind/clear an expression. |
| `obj.getStatusString()` | Human-readable object status/error. |
| `obj.supportedProperties()` | Property types this object supports. |
| `obj.Shape` / `Mesh` / `Points` | Geometry containers, depending on type. |
| `obj.Placement` | Placement, where the type provides it. |
| `obj.getGlobalPlacementOf(target, root, sub)` | Link-aware global placement; prefer over deprecated `getGlobalPlacement`. |

Property flags and editor modes are covered in `references/document-objects.md`.

## Property types

Common types:

`App::PropertyBool`, `PropertyInteger`, `PropertyFloat`, `PropertyLength`,
`PropertyDistance`, `PropertyAngle`, `PropertyString`, `PropertyStringList`,
`PropertyFloatList`, `PropertyIntegerList`, `PropertyVector`,
`PropertyVectorList`, `PropertyPlacement`, `PropertyRotation`, `PropertyMatrix`,
`PropertyColor`, `PropertyEnumeration`, `PropertyLink`, `PropertyLinkList`,
`PropertyLinkSub`, `PropertyLinkSubList`, `PropertyLinkHidden`,
`PropertyXLink`, `PropertyXLinkList`, `PropertyFile`, `PropertyFileIncluded`,
`PropertyPath`, `PropertyMap`, `PropertyExpressionEngine`,
`PropertyPythonObject`.

Special notes:

- `PropertyLink`/`LinkList` support expressions and document links.
- `PropertyLinkHidden` is a link that is not shown in the editor; useful for
  internal references.
- `PropertyLinkSub` stores `(object, ["Face1", "Edge2"])` selections.
- `PropertyFileIncluded` stores a copy of a file inside the `.FCStd`; prefer it
  for data the document must carry.
- `PropertyFile`/`PropertyPath` store external paths; validate them before use.
- `PropertyMap` stores a dictionary; use it for small structured data, not large
  blobs.
- `PropertyPythonObject` is powerful but fragile; prefer native properties and
  keep serialization explicit.
- `PropertyEnumeration` choices must be set before assigning a value.
- `PropertyExpressionEngine` lets users bind expressions to properties; do not
  fight it.

## ViewProvider

| API | Use |
|---|---|
| `ViewObject.Proxy` | Python ViewProvider proxy; `0` means default. |
| `attach(vobj)` | Build display modes/scene graph. |
| `updateData(obj, prop)` | React to a document property change; GUI thread. |
| `onChanged(vobj, prop)` | React to a ViewObject property change. |
| `getIcon()` | SVG path for the tree. |
| `getDisplayModes()` / `getDefaultDisplayMode()` / `setDisplayMode(mode)` | Display mode list/default/switch. |
| `vobj.addDisplayMode(node, name)` | Register a Coin scene graph for a mode. |
| `vobj.listDisplayModes()` | List registered modes. |
| `doubleClicked(vobj)` | Double-click behavior; return `True` if handled. |
| `claimChildren()` | Objects nested under this one in the tree. |
| `setupContextMenu(vobj, menu)` | Add `QAction`s to the context menu. |
| `setEdit(vobj, mode)` / `unsetEdit(vobj, mode)` | Enter/leave an edit mode. |
| `onDelete(vobj, sub_elements)` | Confirm/handle deletion. |
| `canDragObject(obj)` / `canDragObjects()` | Drag support. |
| `canDropObject(obj)` / `canDropObjects()` | Drop support. |
| `dragObject(vobj, obj)` / `dropObject(vobj, obj)` | Perform drag/drop. |
| `vobj.Visibility` / `show()` / `hide()` | Visibility. |
| `vobj.DisplayMode` | Current display mode. |
| `vobj.ShapeColor`, `LineColor`, `Transparency`, `LineWidth`, `PointSize` | Visual properties. |
| `vobj.BoundingBox` | View bounding box. |
| `vobj.addProperty(...)` / `setEditorMode` / `setPropertyStatus` | Add/configure ViewObject-specific properties. |

`EditMode.Default`, `Transform`, `Cutting`, and `Color` are the standard edit
modes. Use `Gui.ActiveDocument.resetEdit()` when leaving a task panel.

## GUI, workbenches, commands

| API | Use |
|---|---|
| `Gui.addWorkbench(instance)` | Register a workbench. |
| `Gui.addCommand(name, instance)` | Register a command. |
| `Gui.runCommand(name, index)` | Invoke a command. |
| `Gui.addWorkbenchManipulator(obj)` / `removeWorkbenchManipulator(obj)` | Contribute to existing menus/toolbars. |
| `Gui.listWorkbenches()` / `activeWorkbench()` | Workbench lookup. |
| `Gui.getMainWindow()` | Parent for dialogs/docks. |
| `Gui.activeDocument()` / `Gui.ActiveDocument.ActiveView` | Active GUI document/view. |
| `Gui.SendMsgToActiveView("ViewFit")` | Standard view action. |
| `Gui.addIconPath(path)` / `addLanguagePath(path)` | Register resources. |
| `Gui.updateLocale()` | Apply the current language. |
| `Gui.supportedLocales()` / `getLocale()` / `setLocale()` | Locale information. |
| `Gui.addPreferencePage(path_or_class, group)` | Add a preferences page. |
| `Gui.PySideUic.loadUi(path)` / `Gui.UiLoader()` | Load Qt Designer forms. |
| `Gui.ExpressionBinding(widget)` | Bind an input to a document property/expression. |
| `Gui.Control.showDialog(panel)` / `closeDialog()` / `activeDialog()` | Task panels. |
| `Gui.ActiveDocument.resetEdit()` | Leave object edit mode after a task panel. |

Workbench methods: `Initialize`, `Activated`, `Deactivated`, `ContextMenu`,
`appendToolbar`, `appendMenu`, `appendContextMenu`, `appendCommandbar`,
`GetClassName`, plus `MenuText`, `ToolTip`, `Icon`.

Command resource keys: `MenuText`, `ToolTip`, `Pixmap`, `Accel`, `StatusTip`,
`WhatsThis`, `CmdType`, `Checkable`, `Exclusive`, `DropDownMenu`. Group commands
also provide `GetCommands()` and `GetDefaultCommand()`. Some command adapters
call `OnActionInit()`; keep it cheap and lazy.

Workbench manipulator methods: `modifyToolBars`, `modifyMenuBar`,
`modifyContextMenu(recipient)`. `modifyDockWindows` is currently a no-op.

### 3D view

| API | Use |
|---|---|
| `Gui.ActiveDocument.ActiveView` | Current 3D view. |
| `view.fitAll()` / `view.viewTop()` / `view.viewFront()` / `view.viewRight()` | Standard view operations. |
| `view.viewAxonometric()` / `view.viewIsometric()` | Axonometric/isometric view. |
| `view.zoomIn()` / `view.zoomOut()` | Zoom. |
| `view.getCameraNode()` | Coin `SoCamera`. |
| `view.getViewer().getSoRenderManager()` | Viewer/render manager. |
| `view.getActiveObject("pdbody")` | Active PartDesign Body. |
| `view.redraw()` | Request a redraw. |
| `Gui.SendMsgToActiveView("ViewFit")` | Send a standard view command. |

Reacquire the active view when the document or view changes; do not hold a stale
`View3DInventorPy` or `SoCamera` reference.

## Selection

| API | Use |
|---|---|
| `Gui.Selection.getSelection()` | Selected objects. |
| `Gui.Selection.getSelectionEx()` | Selected objects with sub-element names and pick points. |
| `Gui.Selection.addSelection(doc, obj, sub)` | Add to selection. |
| `Gui.Selection.removeSelection(doc, obj, sub)` | Remove from selection. |
| `Gui.Selection.clearSelection()` | Clear selection. |
| `Gui.Selection.getSelectionObject(doc, obj, sub)` | Build a selection record from names. |
| `Gui.Selection.getSelectionFromStack(doc, index)` | Read the selection history. |
| `Gui.Selection.addSelectionGate(filter)` / `removeSelectionGate()` | Restrict what the user can pick. |
| `Gui.Selection.setSelectionStyle(style)` | Highlight/preselection style. |
| `Gui.Selection.addObserver(observer)` / `removeObserver(observer)` | Observe selection changes. |
| `Gui.Selection.ResolveMode.NoResolve` | Avoid costly sub-object resolution in observers. |

Selection observer methods: `setPreselection`, `removePreselection`,
`addSelection`, `removeSelection`, `setSelection`, `clearSelection`,
`pickedListChanged`.

## Observers and events

| API | Use |
|---|---|
| `App.addDocumentObserver(observer)` / `removeDocumentObserver(observer)` | Document/object/transaction events. |
| `Gui.Selection.addObserver(observer)` / `removeObserver(observer)` | Selection events. |
| `view.addEventCallback(name, callback)` | 3D view string event. |
| `view.addEventCallbackPivy(coin_event, callback)` | 3D view Coin event. |
| `view.addDraggerCallback(dragger, name, callback)` | Dragger callback. |
| `view.removeEventCallback(...)` / `removeEventCallbackPivy(...)` / `removeDraggerCallback(...)` | Remove callbacks. |
| `App.GuiUp` | Console vs GUI mode. |
| `QTimer.singleShot(0, callback)` | Defer work to the next GUI turn. |
| `shiboken.isValid(obj)` | Check a C++ Qt object is still alive. |

Document observer slots include: `slotCreatedDocument`,
`slotDeletedDocument`, `slotRelabelDocument`, `slotActivateDocument`,
`slotRecomputedDocument`, `slotBeforeRecomputeDocument`, `slotUndoDocument`,
`slotRedoDocument`, `slotChangedDocument`, `slotBeforeChangeDocument`,
`slotStartSaveDocument`, `slotFinishSaveDocument`, `slotOpenTransaction`,
`slotCommitTransaction`, `slotAbortTransaction`, `slotBeforeCloseTransaction`,
`slotCloseTransaction`, `slotCreatedObject`, `slotDeletedObject`,
`slotChangedObject`, `slotBeforeChangeObject`, `slotRecomputedObject`,
`slotAppendDynamicProperty`, `slotRemoveDynamicProperty`,
`slotChangePropertyEditor`, `slotBeforeAddingDynamicExtension`,
`slotAddedDynamicExtension`.

Detailed lifecycle and cleanup rules: `references/events-and-lifecycle.md`.

## Preferences and parameters

| API | Use |
|---|---|
| `App.ParamGet("User parameter:BaseApp/Preferences/Mod/MyAddon")` | Preference group. |
| `group.GetBool/GetInt/GetFloat/GetString(key, default)` | Typed reads with defaults. |
| `group.SetBool/SetInt/SetFloat/SetString(key, value)` | Typed writes. |
| `group.RemBool/RemInt/RemFloat/RemString(key)` | Remove a value. |
| `group.GetGroups()` / `HasGroup(name)` / `RemGroup(name)` | Nested groups/presets. |
| `group.AttachManager(observer)` | Receive `slotParamChanged(group, type, name, value)`. |
| `group.Detach(observer)` | Stop receiving changes. |
| `group.Attach(observer)` | Register a direct parameter observer; it receives `onChange(group, type, name, value)`. |
| `Gui.addPreferencePage(path, group)` | Register a `.ui` page. |
| `Gui.addPreferencePage(PageClass, group)` | Register a programmatic page. |
| `Gui::PrefCheckBox`, `PrefComboBox`, `PrefSpinBox`, `PrefDoubleSpinBox`, `PrefLineEdit`, `PrefColorButton` | Standard preference widgets. |
| `Gui::InputField` / `App.Units.Quantity` | Unit-aware numeric input. |
| `App.Base.ProgressIndicator` | `start` / `next` / `stop` for long operations. |

A programmatic page class provides a `form` widget and `loadSettings()` /
`saveSettings()` methods. Loads read from `ParamGet`; saves validate first, then
write all values, then update the UI.

## Resources and translation

| API | Use |
|---|---|
| `importlib.resources.files(module)` | Traversable resource tree. |
| `importlib.resources.as_file(resource)` | Temporary filesystem path for a resource. |
| `importlib.resources.read_text(module, name)` / `read_binary` | Read asset contents. |
| `Gui.addIconPath(path)` / `Gui.addLanguagePath(path)` | Register directories. |
| `App.Qt.translate(context, text)` | Runtime translation. |
| `QT_TRANSLATE_NOOP(context, text)` | Mark a literal for extraction. |
| `App.Gui.updateLocale()` | Re-apply translations. |

Use `as_file()` inside a context manager; the path may be temporary.

## PartDesign and Link integration

| API | Use |
|---|---|
| `PartDesign::FeaturePython`, `PartDesign::FeatureAdditivePython`, `PartDesign::FeatureSubtractivePython` | PartDesign feature types. |
| `Part::AttachExtensionPython` | Makes a feature attachable; call `positionBySupport()`. |
| `BaseFeature` | Existing PartDesign base for additive/subtractive shape logic. |
| `AddSubShape` | Shape used by PartDesign patterns. |
| `Gui.ActiveDocument.ActiveView.getActiveObject("pdbody")` | Current PartDesign Body. |
| `body.addObject(feature)` | Insert the feature into the Body. |
| `App::LinkExtensionPython` | Behave like an `App::Link`. |
| `obj.configLinkProperty(**mapping)` | Map linked properties to the Link engine. |
| `PropertyLink`, `PropertyLinkHidden`, `PropertyLinkList` | Link targets. |
| `obj.setPropertyStatus("PlacementList", "-Immutable")` / `"Immutable"` | Allow/prevent per-element placement edits for link arrays. |
| `App::GroupExtensionPython` | Group-like container; `Group`, `addObject`, `removeObject`. |

PartDesign shape logic must respect `Placement`, `BaseFeature`, and
`AddSubShape`; do not assign a raw shape and ignore those. See
`references/document-objects.md`.

## Testing

| API | Use |
|---|---|
| `App.newDocument(name)` / `App.closeDocument(name)` | Test fixture document. |
| `App.setActiveDocument(name)` | Make the test document active. |
| `doc.recompute()` | Run `execute()` before asserting. |
| `unittest.TestCase` | Test class. |
| `FreeCAD.__unit_test__` | Register headless test modules. |
| `FreeCADGui.__unit_test__` | Register GUI/Qt test modules. |
| `freecadcmd --run-test ...` | Run headless tests. |
| `FreeCAD --run-test ...` | Run GUI tests. |
| `shape.BoundBox` / `shape.Volume` | Cheap geometry assertions with tolerances. |

Use `setUp`/`tearDown` to create and close documents; never leave a test document
open on failure. See `references/quality-checklist.md`.
