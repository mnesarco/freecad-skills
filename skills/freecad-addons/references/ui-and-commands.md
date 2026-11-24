# Workbenches, commands, Qt, preferences, and logging

This reference covers the GUI side of a FreeCAD 1.1+ addon. It assumes the
modern layout from `references/layout-and-manifest.md`.

## 1. Workbench registration

A workbench is a class inheriting `FreeCADGui.Workbench`:

```python
from pathlib import Path

import FreeCADGui as Gui
from PySide.QtCore import QT_TRANSLATE_NOOP

_PKG = Path(__file__).parent
_ICON = str(_PKG / "resources" / "icons" / "logo.svg")


class MyWorkbench(Gui.Workbench):
    MenuText = QT_TRANSLATE_NOOP("Workbench", "My Addon")
    ToolTip = QT_TRANSLATE_NOOP("Workbench", "Tools for ...")
    Icon = _ICON

    def Initialize(self):
        # First activation only. Register commands, build toolbars/menus.
        from . import commands  # noqa: F401
        self.appendToolbar(
            QT_TRANSLATE_NOOP("Workbench", "My Addon"),
            ["MyAddon_DoThing", "Separator", "MyAddon_OtherThing"],
        )
        self.appendMenu(
            QT_TRANSLATE_NOOP("Workbench", "My Addon"),
            ["MyAddon_DoThing", "MyAddon_OtherThing"],
        )

    def Activated(self):
        # Every activation. Keep cheap.
        pass

    def Deactivated(self):
        pass

    def ContextMenu(self, recipient):
        # recipient is "Tree", "View", or "Sketch" (capitalized).
        if recipient == "View":
            self.appendContextMenu("MyAddon", ["MyAddon_DoThing"])

    def GetClassName(self):
        return "Gui::PythonWorkbench"


Gui.addWorkbench(MyWorkbench())
```

Rules:

- `MenuText`, `ToolTip`, and `Icon` are read before `Initialize()`. Set `Icon`
  as a class attribute, using an absolute filesystem path.
- `Initialize()` runs once, on first activation. Import command modules there
  so users who never use the workbench do not pay for them.
- `Activated()` / `Deactivated()` run on every switch. Do not do expensive setup
  there.
- `GetClassName()` must return `"Gui::PythonWorkbench"`.
- `appendToolbar(title, commands)` and `appendMenu(path, commands)` take lists of
  registered command names. `path` may be a string or a list of strings for a
  submenu.
- `appendCommandbar(title, commands)` adds a command bar, where each command is
  represented as a compact button. Use it for tool palettes, not for general
  menus.
- `"Separator"` is a special string; **do not translate it**.
- Toolbar and menu names are global. Prefix them (`MyAddon`, `MyAddon_Tools`)
  to avoid collisions with core and other addons.
- A toolbar name is also a preference key for visibility. Renaming a toolbar
  later silently discards the user's hidden/shown preference.

## 2. Command classes

A command is a plain class; FreeCAD looks up methods by name. It is not a
subclass of anything.

```python
from pathlib import Path

import FreeCAD as App
import FreeCADGui as Gui
from PySide.QtCore import QT_TRANSLATE_NOOP

_ICON = str(Path(__file__).parents[1] / "resources" / "icons" / "do-thing.svg")


class DoThing:
    def GetResources(self):
        return {
            "MenuText": QT_TRANSLATE_NOOP("MyAddon_DoThing", "Do thing"),
            "ToolTip": QT_TRANSLATE_NOOP("MyAddon_DoThing", "Does the thing"),
            "StatusTip": QT_TRANSLATE_NOOP("MyAddon_DoThing", "Does the thing"),
            "Pixmap": _ICON,
            "Accel": "Ctrl+Shift+D",
            "CmdType": "AlterDoc",
        }

    def IsActive(self):
        return App.activeDocument() is not None

    def Activated(self):
        doc = App.activeDocument()
        if doc is None:
            return
        doc.openTransaction("Do thing")
        try:
            # ... modify the document ...
            doc.commitTransaction()
        except Exception:
            doc.abortTransaction()
            raise


Gui.addCommand("MyAddon_DoThing", DoThing())
```

### `GetResources()` keys

| Key | Purpose |
|---|---|
| `MenuText` | Label in menus and default toolbar tooltip. Required in practice. |
| `ToolTip` | Longer hover text. |
| `Pixmap` | SVG path (absolute or registered icon filename). |
| `Accel` | Keyboard shortcut, e.g. `"Ctrl+Shift+D"`. |
| `StatusTip` | Status-bar text; falls back to `ToolTip`. |
| `WhatsThis` | "What's this?" help text. |
| `CmdType` | Declares what the command changes; see below. |

Some FreeCAD versions also honor `Checkable`, `Exclusive`, or `DropDownMenu` for
toolbar state. Treat those as optional; do not build core behavior on them.

### Toggle, grouped, and dropdown commands

Some FreeCAD command implementations support extra resource keys:

| Key | Use |
|---|---|
| `Checkable` | The command has a checked/unchecked state. |
| `Exclusive` | Only one command in a group may be checked. |
| `DropDownMenu` | The command renders as a dropdown/group. |

For grouped/dropdown commands, expose the sub-command names and the default
index through the command object's `GetCommands()` and `GetDefaultCommand()`
hooks. Keep the sub-command list stable and register every sub-command before
the group command.

A checkable command should accept the checked state in `Activated(self,
checked=False)` and use it to update its own state or the document. For a radio
group, only one command should be exclusive/checked.

Not every FreeCAD version honors every optional key. Test the target versions
and declare `freecadmin` accordingly. Do not build core behavior on an
undocumented key.

Some command adapters call an initialization hook such as `OnActionInit()`;
keep it cheap and lazy (no network, no heavy geometry). Do not rely on it for
document correctness.

### `CmdType`

`CmdType` tells FreeCAD whether the command may run while a task dialog is open:

- `AlterDoc` — modifies the document.
- `Alter3DView` — modifies only the 3D view.
- `AlterSelection` — modifies only the selection.
- `ForEdit` — stays available inside an edit/task dialog. Use only when the
  command genuinely must modify the document from inside an edit session.
- `NoTransaction` — currently has no effect on Python addon commands; manage
  transactions yourself.

If you omit `CmdType`, FreeCAD assumes `AlterDoc | Alter3DView |
AlterSelection`. A command with the default type is disabled as soon as a task
dialog (for example Sketcher edit mode) opens. Declare the real scope; do not add
`ForEdit` just to make a button clickable.

`CmdType` does not replace `IsActive()`. Both checks apply.

### `IsActive()`

- Return `False` to disable the toolbar/menu entry.
- Keep it fast: no disk I/O, no network, no full document graph walks.
- Typical checks: active document exists; selection count/type; current
  workbench/context.
- If no condition is needed, return `True` or omit the method.

### `Activated()`

- Do the work here. Wrap document changes in a transaction.
- For long work, show a progress dialog/task panel and keep the UI responsive
  without re-entrancy.
- Do not silently swallow exceptions. Log context to the Report view and show a
  user-facing message for actionable errors; re-raise unexpected errors.

### Transactions

```python
doc = App.activeDocument()
if doc is None:
    return

doc.openTransaction("My operation")
try:
    # modify document objects
    doc.commitTransaction()
except Exception:
    doc.abortTransaction()
    raise
```

- One user action should be one undo step.
- Do not open transactions in `execute()`; recompute already runs in a
  transaction.
- Do not call `doc.recompute()` inside the transaction if you can commit first;
  commit, then recompute.
- If a command changes multiple objects, keep them in one transaction.
- `NoTransaction` in `GetResources()` does not manage this for Python commands.

## 3. Command registration and naming

- Register once with `Gui.addCommand("MyAddon_DoThing", DoThing())`.
- The first argument is a flat, global name. Prefix it with the addon name.
- Do not reuse core prefixes (`Std_`, `Part_`, `Sketcher_`, ...).
- Importing the module that contains the `Gui.addCommand(...)` call is what
  registers the command. For an owned workbench, import command modules inside
  `Initialize()`. For a toolbar-only addon, import them in `init_gui.py` before
  registering the manipulator.
- Reference commands by name in `appendToolbar`, `appendMenu`,
  `appendContextMenu`, and manipulators.

A command registry is optional, but if the addon has many commands it should:

- apply the addon prefix automatically;
- reject duplicate names at registration time;
- keep one command instance (commands are stateless dispatchers, not per-click
  objects);
- expose the registered names for toolbars/menus;
- provide a `run(name)` helper that calls `Gui.runCommand(name, index)`.

Do not register the same command twice under different names. Do not keep
per-invocation state on the command instance; put state in the document, a
panel/model, or pass it explicitly.

### Selection validation

Commands that operate on the selection should validate it before doing work:

- Use `Gui.Selection.getSelection()` for objects.
- Use `Gui.Selection.getSelectionEx()` when sub-elements (`Face1`, `Edge2`) or
  pick points matter.
- Match `obj.TypeId` against expected types/patterns; return a clear message if
  the selection is wrong.
- Do not rely on selection order unless you explicitly sort/validate it.
- Preserve and restore the selection if the command temporarily changes it.
- Keep `IsActive()` cheap; do the detailed validation in `Activated()`.

See `references/events-and-lifecycle.md` for selection observers and
`references/freecad-api-map.md` for the full selection API.

### Progress

For operations longer than a moment, use `FreeCAD.Base.ProgressIndicator`:

```python
bar = App.Base.ProgressIndicator()
bar.start("Working...", len(items))
try:
    for item in items:
        ...
        bar.next()
finally:
    bar.stop()
```

- Always stop in `finally`.
- Do not call `QApplication.processEvents()` inside the transaction.
- If the operation is cancelable, check a cancel flag between chunks and abort
  the transaction cleanly.
- See `references/preferences-and-widgets.md` for a context-manager wrapper.

## 4. Workbench manipulators

Use a manipulator when the addon contributes commands to workbenches it does not
own, or defines no workbench at all. Manipulators are available in FreeCAD 1.0+.

```python
import FreeCADGui as Gui


class Manipulator:
    def modifyToolBars(self):
        return [
            {"append": "MyAddon_DoThing", "toolBar": "File"},
            {"insert": "MyAddon_OtherThing", "toolItem": "Std_Open"},
        ]

    def modifyMenuBar(self):
        return [
            {"append": "MyAddon_DoThing", "menuItem": "Std_DlgMacroRecord"},
            {"insert": "MyAddon_OtherThing", "menuItem": "Std_DlgParameter"},
        ]

    def modifyContextMenu(self, recipient):
        if recipient == "View":
            return [{"insert": "MyAddon_DoThing", "menuItem": "Std_ViewFitAll"}]
        return []


Gui.addWorkbenchManipulator(Manipulator())
```

### Return values

Each method returns a dictionary or a sequence of dictionaries. Each dictionary
describes one change, keyed by `append`, `insert`, or `remove`.

| Operation | Companion key | Effect |
|---|---|---|
| `append` | `toolBar` | Append command to the named toolbar. |
| `insert` | `toolItem` | Insert before the named command in its toolbar. |
| `remove` | none | Remove a toolbar or command by name. |
| `append` | `menuItem` | Append to the menu containing `menuItem`. |
| `insert` | `menuItem` | Insert before `menuItem`. |
| `insert` | `menuItem` + `after` | Insert after `menuItem`; `after` is a flag, its value is ignored. |

### Rules and pitfalls

- Manipulators run on **every workbench activation**, not once at startup. Keep
  them cheap.
- Register the manipulator while `init_gui.py` is imported, not later. Menus of
  the already-active workbench are not rebuilt when a manipulator is added
  afterwards, so a deferred registration only shows up after a workbench switch.
- Nothing is passed in; determine the current workbench yourself. Do not use
  `Gui.activeWorkbench().name()` during a core workbench's first activation
  (it can raise). Use identity lookup:

```python
def active_workbench_name() -> str:
    active = Gui.activeWorkbench()
    for name, handler in Gui.listWorkbenches().items():
        if handler is active:
            return name
    return ""
```

- Return `[]` for workbenches you do not intend to modify.
- Create a new toolbar by appending its name to `""` first, then append commands
  to it:

```python
return [
    {"append": "MyAddon", "toolBar": ""},
    {"append": "MyAddon_DoThing", "toolBar": "MyAddon"},
]
```

- Toolbar/menu names and core command names are global and version-coupled.
  Prefix your names, and test on every FreeCAD version you claim.
- A manipulator failure is caught by FreeCAD and printed to the Report view; the
  UI silently does nothing. Check the Report view when a toolbar is missing.
- `modifyDockWindows()` is currently a no-op. Add dock widgets directly with Qt.
- Keep a reference to the manipulator object if you need to remove it with
  `Gui.removeWorkbenchManipulator(obj)`.
- Do not hide, remove, or rewrite core UI or another addon's UI beyond the
  specific contribution the user installed the addon for. Return `[]` for
  workbenches the manipulator does not intentionally change.

## 5. Qt and PySide

### Imports

Use FreeCAD's Qt shim at runtime:

```python
from PySide import QtCore, QtGui, QtWidgets
```

Do **not** import `PySide2` or `PySide6` for these modules. The shim maps to the
Qt binding FreeCAD was built with and survives a Qt major-version bump.

The one exception is Qt Quick. FreeCAD's `PySide` shim does not expose
`QtQml`, `QtQuick`, or `QtQuickWidgets`; import those from `PySide6` directly,
lazily in GUI code. See `references/qml-and-qtquick.md`.

For type checking, import `PySide6` inside `TYPE_CHECKING`:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from PySide6 import QtCore, QtGui, QtWidgets
```

If the addon must support both Qt5 and Qt6 enum styles, centralize the
differences in one compatibility module and use it everywhere. Do not scatter
version checks through the code.

A compatibility module can also lazy-load Qt submodules and check C++ object
validity:

```python
# one place only
try:
    from shiboken6 import isValid
except ImportError:  # Qt5 build
    from shiboken2 import isValid
```

Use `isValid(widget)` before touching a cached widget/view that C++ may have
deleted. Do not keep a stale wrapper alive and call methods on it; that can crash
instead of raising.

If you wrap `PySide` in your own module, register lazy submodules in
`sys.modules` so `from myqt.QtCore import QObject` works, and keep that
complexity in one file. Do not reimplement the wrapper in every module.

### Dialogs and `.ui` files

```python
import FreeCADGui as Gui
from PySide import QtWidgets

mw = Gui.getMainWindow()
form = Gui.PySideUic.loadUi(str(_PKG / "resources" / "ui" / "dialog.ui"))
form.setParent(mw)
```

- Parent dialogs to `Gui.getMainWindow()` so they inherit the theme and do not
  get lost behind FreeCAD.
- Do not put `:/...` resource-path icons in `.ui` files. Set icons in Python
  after loading, using plain SVG paths.
- For simple dialogs, building widgets in Python is fine. For multi-input forms,
  a `.ui` file is easier to maintain.
- All GUI work happens on the main thread. Use signals to marshal from worker
  threads.

### Task panels

A task panel replaces the modal dialog and integrates with the Combo View:

```python
import FreeCAD as App
import FreeCADGui as Gui


class MyTaskPanel:
    def __init__(self):
        self.form = Gui.PySideUic.loadUi(str(_PKG / "resources" / "ui" / "panel.ui"))

    def accept(self):
        Gui.Control.closeDialog()
        if doc := App.ActiveDocument:
            doc.recompute()
        if Gui.ActiveDocument:
            Gui.ActiveDocument.resetEdit()
        return True

    def reject(self):
        Gui.Control.closeDialog()
        return True


Gui.Control.showDialog(MyTaskPanel())
```

Use task panels for iterative work where the user needs to see the 3D view.
`Gui.Control.activeDialog()` tells you whether a task panel is open;
`Gui.ActiveDocument.resetEdit()` leaves edit mode cleanly.

A `QQuickWidget` is a valid `form`; a QML task panel is built the same way,
with the QML content fitted to the dialog and the transaction handled in
Python. See `references/qml-and-qtquick.md` §9.

### Dock widgets

```python
from PySide import QtCore, QtWidgets
import FreeCADGui as Gui

mw = Gui.getMainWindow()
dock = QtWidgets.QDockWidget("My panel", mw)
dock.setObjectName("MyAddon_Dock")  # stable name for state restore
dock.setWidget(content_widget)
mw.addDockWidget(QtCore.Qt.RightDockWidgetArea, dock)
```

- Give the dock a stable object name; user layout persistence depends on it.
- Do not create duplicate docks on every command invocation; find and reuse.
- Parent to the main window.

### Threading

- All Qt widget and Coin3D scene-graph access must happen on the GUI thread.
- `execute()` may run in a worker thread during recompute. It must only touch
  document data, never Qt or Coin.
- Use `QThread` + signals, or `QtCore.QTimer.singleShot(0, callback)`, to
  marshal results back to the GUI thread.
- `FreeCAD.Console.Print*` is safe from worker threads.
- Do not call `QApplication.processEvents()` inside `execute()`, a transaction,
  or a long operation that mutates the document. It permits re-entrancy and can
  invalidate objects mid-operation. Use a modal progress dialog or chunk the
  work instead.
- Do not use `__del__` for critical cleanup; it is not guaranteed to run when
  you expect, especially during interpreter shutdown.
- A `QObject` with a queued signal is a reliable UI-thread bridge: emit from the
  worker, connect the slot with `Qt.QueuedConnection`, and do widget work only
  in the slot. A `QTimer.singleShot(0, callback)` is enough for a one-off.
- Keep non-modal dialogs referenced and call `deleteLater()` on close; otherwise
  they can be garbage-collected or leak. Connect `destroyed` to clear cached
  references.
- Use `QApplication.activeWindow()` / `Gui.getMainWindow()` to parent dialogs;
  do not create parentless modal dialogs.

### Themes, HiDPI, and style

- Do not hardcode colors or fonts. Use the application palette and FreeCAD's
  theme.
- Use SVG icons; they scale on HiDPI screens.
- Let layouts size widgets; avoid fixed pixel widths/heights unless required.
- Do not override `QApplication` styles globally.

## 6. Preferences

FreeCAD's native mechanism is a Qt Designer `.ui` page using `Gui::Pref*`
widgets, plus the parameter store.

### The `.ui` page

Use `Gui::PrefCheckBox`, `Gui::PrefComboBox`, `Gui::PrefSpinBox`,
`Gui::PrefDoubleSpinBox`, `Gui::PrefLineEdit`, `Gui::PrefColorButton`, etc.
Each widget needs:

- `prefPath`: usually `Mod/MyAddon`.
- `prefEntry`: the key name.

FreeCAD reads/writes the values automatically when the user opens/applies the
preferences dialog.

### Registration

```python
Gui.addPreferencePage(
    str(_PKG / "resources" / "ui" / "preferences.ui"),
    "MyAddon",
)
```

Call it once at startup. The sidebar icon is
`preferences-<modulename>.svg` (lowercase) on an icon path registered with
`Gui.addIconPath()`.

### Reading values

```python
import FreeCAD as App

params = App.ParamGet("User parameter:BaseApp/Preferences/Mod/MyAddon")
enabled = params.GetBool("EnableFoo", True)
count = params.GetInt("Count", 10)
ratio = params.GetFloat("Ratio", 0.5)
label = params.GetString("Label", "")
```

Always pass a default. Match the getter to the widget type:

| Widget | Getter / setter |
|---|---|
| `Gui::PrefCheckBox` | `GetBool` / `SetBool` |
| `Gui::PrefSpinBox` | `GetInt` / `SetInt` |
| `Gui::PrefDoubleSpinBox` | `GetFloat` / `SetFloat` |
| `Gui::PrefComboBox` | `GetInt` / `SetInt` (stores index) |
| `Gui::PrefLineEdit` | `GetString` / `SetString` |

- Use the parameter store, not `QSettings`, for addon settings.
- Keep internal/hidden preferences in a separate group, e.g.
  `Preferences/Mod/MyAddon/Internal`.
- Do not store large data blobs in preferences; use files under
  `App.getUserAppDataDir()` and validate paths.
- If a preference changes behavior live, subscribe to parameter changes or
  re-read on use. Do not cache forever without invalidation.

### Programmatic pages and live updates

For dynamic pages, validation, or presets, `Gui.addPreferencePage` also accepts
a class with a `form` widget and `loadSettings()` / `saveSettings()` methods:

```python
class MyPreferencesPage:
    def __init__(self, _parent=None):
        self.form = build_form()

    def loadSettings(self):
        self.form.setValues(read_all_settings())

    def saveSettings(self):
        validate_all_settings()
        write_all_settings()


Gui.addPreferencePage(MyPreferencesPage, "MyAddon")
```

- Validate everything before writing; do not partially save a page.
- Load values from `App.ParamGet` on open, not from stale module globals.
- Use `App.ParamGet(group).AttachManager(observer)` with
  `slotParamChanged(group, value_type, name, value)` for live updates;
  `Detach(observer)` on teardown.
- Presets live under nested groups (`.../presets/<Name>`); do not delete or
  rename `Default`.
- Use a deferred translation wrapper for class-level labels/options so they
  translate when displayed, not at import.

Full patterns: `references/preferences-and-widgets.md`.

## 7. Logging and user feedback

FreeCAD console levels:

```python
import FreeCAD as App

App.Console.PrintMessage("Operation completed\n")
App.Console.PrintLog("Debug detail\n")
App.Console.PrintWarning("Using fallback value\n")
App.Console.PrintError("Could not parse file\n")
```

Rules:

- Console output is developer-facing (Report view). It is not translated.
- Always include a trailing newline.
- `PrintLog` is hidden unless the user enables the Log category; use it for
  diagnostics.
- `PrintWarning` / `PrintError` do not raise; they only print.
- Do not use `print()` in production code.
- Prefix diagnostic logs with a stable tag and context, e.g.
  `[MyAddon] ...`, so users can filter the Report view.
- Do not log secrets, tokens, or full user paths unnecessarily.
- Do not use console output for important user decisions. Use `QMessageBox` or
  the status bar:

```python
QtWidgets.QMessageBox.warning(
    Gui.getMainWindow(),
    translate("MyAddon", "Selection required"),
    translate("MyAddon", "Please select a face before running this command."),
)

Gui.getMainWindow().statusBar().showMessage(
    translate("MyAddon", "Recomputing..."), 3000
)
```

- Translate user-facing messages; do not translate logs.
- Do not swallow exceptions. Log with context, show an actionable message, then
  re-raise unexpected errors so the Report view keeps the traceback.

## 8. Common GUI mistakes

- Heavy imports in `init_gui.py` (numpy, OCC, big UI modules) — defer them.
- Registering a workbench but building toolbars in `__init__` instead of
  `Initialize()`.
- Setting `Icon` inside `Initialize()` — too late; FreeCAD already read it.
- Using a bare `QDialog` with no parent — it can disappear behind FreeCAD.
- Calling `QApplication.processEvents()` during recompute — re-entrancy.
- Touching widgets or Coin nodes from a worker thread.
- Using `PySide6` directly for `QtCore`/`QtGui`/`QtWidgets` — breaks on future
  Qt builds. It is correct only for `QtQml`/`QtQuick`/`QtQuickWidgets`, which
  the shim does not expose.
- Building QML at import time or before the main window exists.
- Putting FreeCAD logic in QML instead of a bridge object.
- Nulling QML context properties or resetting a model from inside a QML
  handler instead of deleting the view / deferring the change.
- Hardcoding colors/fonts — breaks themes and dark mode.
- Adding commands to core menus globally at import — use a manipulator and scope
  it to the intended workbench.
- Translating `"Separator"` or technical identifiers.
- Relying on undocumented `GetResources` keys.
- Forgetting `CmdType` and wondering why a button is disabled in Sketcher.
- Registering selection/document observers and never removing them.
- Keeping a stale widget/view/camera wrapper after C++ deleted it; use
  `shiboken.isValid`.
- Leaving a task panel open or forgetting `Gui.ActiveDocument.resetEdit()`.
- Calling `QApplication.processEvents()` to "keep the UI alive".
- Implementing `dumps`/`loads` or `GetCommands` without registering the
  sub-commands.
- Using a modal dialog when a task panel is the FreeCAD-native interaction.
