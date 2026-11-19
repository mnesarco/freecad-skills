# QML and Qt Quick interfaces

This reference covers Qt Quick in a FreeCAD 1.1+ addon: hosting QML from
Python, the Python/QML boundary, models, theming, task panels, dock panels,
overlays, tests, and packaging.

QML is not a replacement for every dialog. Use it when the interface needs
custom drawing, animation, large model views, floating controls over the 3D
view, or a task panel that QtWidgets makes awkward. Standard forms, simple
dialogs, and preference pages stay QtWidgets, where FreeCAD already provides
the widgets and layout conventions.

The patterns here are the ones used by FreeCAD-Nxt, a production addon that
replaces parts of FreeCAD's interface with Qt Quick entirely from Python.

## 1. Hosting options

| Surface | Host |
|---|---|
| Dock panel | `QDockWidget` containing a `QQuickWidget` |
| Task panel content | `QQuickWidget` as the task class's `form` |
| Toolbar / small strip | `QQuickWidget` added with `QToolBar.addWidget()` |
| Notice or floating control over the 3D view | `QQuickWidget` child of the view's GL widget |
| Standalone window | `QQuickView` (rare; usually tests only) |

`QQuickWidget` is the default. It is a real `QWidget`, so it can live in a
dock, a toolbar, a dialog, or FreeCAD's task view. `QQuickView` creates its own
top-level window and brings window-manager, stacking, and Wayland problems that
a child widget avoids.

A QML scene cannot contain a `QWidget`. If a surface needs FreeCAD's own
widgets, either keep the surface in QtWidgets or show the widget in a separate
native container. Do not try to wrap a widget into a QML item.

Give the view a size policy that matches its host: a dock or task panel fills
the available space (`SizeRootObjectToView`), while a toolbar strip or a
floating HUD follows the QML root's implicit size (`SizeViewToRootObject`).

## 2. Qt modules, availability, and the renderer

FreeCAD's `PySide` shim exposes `QtCore`, `QtGui`, and `QtWidgets`, but not the
Qt Quick modules. Keep the shim for the core modules and import Qt Quick from
`PySide6` directly, in a GUI-only helper module that is itself imported only
when the first QML surface is built:

```python
from PySide import QtCore, QtGui, QtWidgets
from PySide6 import QtQml, QtQuick, QtQuickWidgets
```

- Import Qt Quick lazily, at the point the QML surface is built. Never import
  it from `__init__.py`, and never build QML in console mode.
- Check availability and degrade to a QWidget message instead of crashing:

```python
def qtquick_available() -> bool:
    try:
        from PySide6 import QtQuickWidgets  # noqa: F401
    except ImportError:
        return False
    return True
```

- Do not declare a runtime dependency on PySide6. FreeCAD ships the binding
  and the Qt Quick modules it supports. Test with FreeCAD's own interpreter.
  A development-only `pyside6` dependency for headless tests is fine.
- Do not use modules the target FreeCAD's Qt build does not ship. Test the
  exact import in the target FreeCAD, not only in a development virtualenv.

### Renderer backend

Embedding a `QQuickWidget` over FreeCAD's Coin3D viewport can produce a blank,
black, or mismatched surface because Qt's RHI chooses a different graphics API
than Coin3D. If that happens, force OpenGL **before the first Qt Quick surface
exists**:

```python
import os


def use_shared_graphics_api() -> None:
    os.environ.setdefault("QSG_RHI_BACKEND", "opengl")
```

- It is process-wide and one-way: Qt reads it before the first Qt Quick
  window, so set it lazily when the surface is built, not at import.
- `setdefault` lets a user or FreeCAD override it.
- Do not set it for an addon that never creates a QML surface.
- Only apply it after reproducing the problem on the target platform; forcing
  OpenGL on a platform that works with its default RHI is unnecessary.

## 3. Resource layout and QML modules

Ship QML as plain files, like icons and translations, and never compile `.qrc`:

```text
freecad/MyAddon/
├─ resources/
│  └─ qml/
│     ├─ MyPanel.qml                  # root file loaded by Python
│     ├─ MyTask.qml
│     └─ MyAddon/
│        └─ Controls/
│           ├─ qmldir
│           ├─ TextButton.qml
│           └─ Chip.qml
```

Resolve the directory from the package, not from the current working
directory:

```python
from importlib.resources import files
from pathlib import Path


QML = Path(str(files("freecad.MyAddon") / "resources" / "qml"))
```

This works because addons are installed unpacked; QML needs real files on
disk. Add that directory as an import path. A module directory needs a
`qmldir`:

```text
module MyAddon.Controls
TextButton 1.0 TextButton.qml
Chip 1.0 Chip.qml
```

Then QML imports it by name:

```qml
import QtQuick
import MyAddon.Controls

Rectangle {
    Chip { text: "Hello" }
}
```

Rules:

- Root views stay at the top of `resources/qml/`; reusable components live in
  namespaced module directories under it.
- All QML files must be real files in a real directory. QML cannot import from
  inside a zip or wheel without compiled resources, and addons are normally
  installed unpacked. Do not use `as_file()` for an import path that outlives
  the context.
- Keep the module namespace addon-specific (`MyAddon.Controls`) so components
  cannot collide with another addon or with FreeCAD's own QML.
- No `.qrc`, no `pyside6-rcc`, no Qt-major-version-locked resources.
- Translate `.qml` files with `pyside6-lupdate`, like `.py` files.

## 4. Creating a view

Keep QML hosting in one small GUI-only module so every surface gets the same
engine setup, context handling, and error reporting:

```python
# freecad/MyAddon/ui/quick.py — GUI-only; imported lazily
from pathlib import Path
from typing import Any

from PySide import QtCore, QtGui, QtWidgets
from PySide6 import QtQuickWidgets

_QML = Path(__file__).parents[1] / "resources" / "qml"


class QmlLoadError(RuntimeError):
    """A QML file failed to load; the message lists QML's errors."""


def make_view(parent: QtWidgets.QWidget | None, source: str,
              context: dict[str, Any], *,
              floating: bool = False) -> Any:
    """A QQuickWidget showing a root QML file from resources/qml."""
    view = QtQuickWidgets.QQuickWidget(parent)
    modes = QtQuickWidgets.QQuickWidget.ResizeMode
    view.setResizeMode(modes.SizeViewToRootObject if floating
                       else modes.SizeRootObjectToView)
    if floating:
        view.setAttribute(QtCore.Qt.WidgetAttribute.WA_AlwaysStackOnTop)
        view.setClearColor(QtGui.QColor(0, 0, 0, 0))
    view.engine().addImportPath(str(_QML))
    root_context = view.rootContext()
    for name, value in context.items():
        root_context.setContextProperty(name, value)
    view.setSource(QtCore.QUrl.fromLocalFile(str(_QML / source)))
    if view.status() == QtQuickWidgets.QQuickWidget.Status.Error:
        detail = "\n".join(str(error.toString()) for error in view.errors())
        view.deleteLater()
        raise QmlLoadError(f"{source} failed to load:\n{detail}")
    return view
```

A minimal root view then looks like this:

```qml
import QtQuick

Rectangle {
    color: theme.background
    implicitWidth: 280
    implicitHeight: 400

    Text {
        anchors.centerIn: parent
        text: panel.count
    }
}
```

Why each part matters:

- **Set the import path before `setSource`.** QML resolves imports while
  loading.
- **Set context properties before `setSource`.** QML evaluates bindings
  immediately; a property added later arrives too late.
- **Check `status()` and print every `error`.** A QML load failure is otherwise
  a blank rectangle. A local file normally loads synchronously; if the status
  is still `Loading`, connect `statusChanged` and check when it settles. Show a
  QWidget fallback with the detail in a read-only text area.
- **Keep Python context objects alive.** `setContextProperty` does not take
  ownership. Parent the bridge and any per-view object to the view, or keep a
  strong reference on the owner. A shared theme/service must be owned by the
  addon, not by one of the views that read it. A collected object becomes
  `null` in QML and every binding that reads it produces a TypeError.
- **One engine per `QQuickWidget`.** Do not try to share an engine between
  widgets; sharing an engine couples object lifetimes and teardown. Share the
  Python context objects (a `Theme`, a service) instead.
- **Do not set a context property to `None` to hide or tear down a view.**
  Delete the `QQuickWidget`; that destroys its engine, object tree, and
  bindings in one step. Setting the property to null re-evaluates every
  binding against null while delegates still exist.
- **For a floating or translucent view**, the widget attributes and clear
  colour must agree. A floating child normally needs
  `WA_AlwaysStackOnTop` and a transparent clear colour; a fully see-through
  surface over another widget also needs `WA_TranslucentBackground`. Set the
  attributes and clear colour together; setting only some of them composites
  the scene over black on some drivers.
- **Set a clear colour** for opaque views too, normally the theme background,
  so the first frame does not flash the default colour.

QML load errors are not exceptions until you check. Report them to the Report
view, keep FreeCAD running, and leave an actionable widget in place.

## 5. The Python/QML boundary

QML must not import FreeCAD, touch documents, or own business logic. Every
surface gets one bridge `QObject` exposed as a context property; the bridge is
the only API QML can call.

```python
class PanelBridge(QtCore.QObject):
    changed = QtCore.Signal()

    def __init__(self, parent: QtCore.QObject | None = None) -> None:
        super().__init__(parent)
        self._count = 0

    @QtCore.Property(int, notify=changed)
    def count(self) -> int:
        return self._count

    @QtCore.Slot()
    def add(self) -> None:
        self._count += 1
        self.changed.emit()
```

QML:

```qml
import QtQuick

Text {
    text: panel.count

    TapHandler { onTapped: panel.add() }
}
```

Rules:

- Use camelCase for properties, slots, and signals; QML is JavaScript.
- Declare notify signals for every mutable property. Without one, QML caches
  the first value forever.
- Assign the new state, then emit. A `dataChanged` or notify signal emitted
  before the state is updated makes QML read the old value.
- Declare argument and result types (`@Slot(str, result=bool)`); avoid
  `QVariant` when a concrete type is known.
- Keep bridge methods short. A slot called from QML runs on the GUI thread and
  inside a Qt event handler. Do heavy work (recompute, file IO, document
  scans) on a queued zero-timer, and never let a Python exception escape the
  slot: log the traceback and leave the UI and document consistent.
- Open one transaction per user action. The bridge owns open/commit/abort;
  QML never calls transaction methods.
- Use parented `QTimer` instances for debouncing and deferred work. A static
  `QTimer.singleShot(0, self.method)` holds a bound method and can fire into a
  deleted object after a reload.
- Do not call `QApplication.processEvents()` to keep the UI responsive. Queue
  work; FreeCAD's progress indicator already re-enters Qt during a recompute.
- Parent the bridge to the view (`bridge.setParent(view)`) or keep it on the
  owning object that lives as long as the view.

### Never destroy QML from its own handler

A delegate handler must not synchronously cause the model that owns it to be
rebuilt. Destroying a QML object while its own signal handler is on the stack
calls `qFatal` and terminates FreeCAD, not merely a warning.

```qml
onClicked: panel.choose(row)
```

If `choose()` emits a signal that replaces the list, defer it:

```python
@QtCore.Slot(int)
def choose(self, row: int) -> None:
    QtCore.QTimer.singleShot(0, lambda: self._choose(row))
```

The same rule applies to anything that can nest an event loop: opening a task
dialog, `setEdit`, and recomputes that raise the progress indicator. Also
defer those. Qt's contract is that an exception must not propagate out of an
event handler; catch at the slot boundary.

## 6. Models

For a list or tree, use a flat `QAbstractListModel` with a `depth` role rather
than a Python `QAbstractItemModel` tree. A Python item model must hand out
stable C++ pointers for every index; when the document mutates underneath,
getting one wrong is a segfault. A flat model with presentation depth is
easier to test and makes filtering and lifting straightforward.

```python
class RowModel(QtCore.QAbstractListModel):
    NameRole = int(QtCore.Qt.ItemDataRole.UserRole) + 1

    def roleNames(self) -> dict[int, QtCore.QByteArray]:
        return {self.NameRole: QtCore.QByteArray(b"name")}

    def rowCount(self, parent=QtCore.QModelIndex()) -> int:
        return 0 if parent.isValid() else len(self._rows)

    def data(self, index, role=QtCore.Qt.ItemDataRole.DisplayRole):
        if not index.isValid():
            return None
        row = self._rows[index.row()]
        if role == self.NameRole:
            return row.name
        return None
```

Rules:

- Fill `roleNames()` and make every delegate's `required property` match a
  role name. A missing role shows as `undefined`, not as an error.
- Rebuild the snapshot off the document signal, not in the signal handler.
  Coalesce bursts and rebuild on the next event-loop turn.
- Diff the new row sequence against the old one with
  `difflib.SequenceMatcher` and apply `beginInsertRows` / `beginRemoveRows`.
  A blanket model reset loses scroll position, selection, and delegate
  identity and makes the view flash on every recompute.
- Emit targeted `dataChanged` for rows whose displayed fields changed. Emit a
  blanket change only when the data really did all change.
- A `Repeater` rebuilds every delegate when its model changes. Never drive an
  editable form's fields with a `Repeater` over a model that changes while the
  user types; drive it from a count or stable per-field objects.
- Do not put translated text in the model unless the model is refreshed on a
  language change. Prefer role values and translate in QML, or refresh the
  model from the i18n callback.

## 7. Theme and drawing

Do not hard-code colours, fonts, or pixel sizes that should scale. Expose a
`Theme` `QObject` as a context property with `QColor` and metric properties,
and notify on change:

```python
class Theme(QtCore.QObject):
    changed = QtCore.Signal()

    def __init__(self, reference: QtWidgets.QWidget) -> None:
        super().__init__()
        self._reference = reference
        self._palette = QtGui.QPalette(reference.palette())
        self._row_height = max(20, reference.fontMetrics().height())

    @QtCore.Property(QtGui.QColor, notify=changed)
    def background(self) -> QtGui.QColor:
        return self._palette.color(QtGui.QPalette.ColorRole.Window)

    @QtCore.Property(int, notify=changed)
    def rowHeight(self) -> int:  # noqa: N802 - QML API
        return self._row_height

    def refresh(self) -> None:
        self._palette = QtGui.QPalette(self._reference.palette())
        self._row_height = max(20, self._reference.fontMetrics().height())
        self.changed.emit()
```

- `QApplication.palette()` is not FreeCAD's stylesheet. FreeCAD applies a
  stylesheet that `QApplication.palette()` knows nothing about, so a light
  stylesheet on a dark desktop paints the panel in colours nothing around it
  shares.
- Read the palette from a reference widget of the same kind as the host, after
  it is polished: the stock tree view for a tree panel, the main window for a
  dialog. `QStyleSheetStyle` resolves rules into the widget's own palette.
- Re-read the palette on `StyleChange` / `PaletteChange`, deferred by a tick:
  the reference widget may not be polished yet when the QML view is created.
- Derive hover, border, selection, and severity colours from the palette; keep
  contrast readable in light and dark themes.
- Take sizes from the font metrics and the row height; support compact and
  roomy densities by scaling one base metric.
- Use an image provider for live FreeCAD icons:

```python
from PySide6.QtQuick import QQuickImageProvider


class IconProvider(QQuickImageProvider):
    def __init__(self) -> None:
        super().__init__(QQuickImageProvider.ImageType.Pixmap)

    def requestPixmap(self, image_id, size, requested_size):
        ...
```

Register it before `setSource` with
`view.engine().addImageProvider("myaddon", provider)`, and use
`image://myaddon/<id>` URLs. The engine takes ownership of the provider. Put a
revision in the URL when the image can change, so QML's image cache does not
serve an old icon. Convert through ARGB when desaturating or recolouring;
`Format_Grayscale8` has no alpha and turns transparent pixels black.

- Prefer drawing simple primitives over Qt Quick Controls. Where a Control is
  used, import `QtQuick.Controls.Basic` and restyle it from the theme. The
  default Controls style is the host's style, not yours, and can change the
  look and metrics of every button, field, and menu.
- Every state needs a non-colour cue: shape, outline, dashed versus solid,
  italic. Colour reinforces, it never carries meaning alone.

## 8. QML language and interaction rules

- Wrap user-facing text in `qsTr("...")`. The translation context is the QML
  file's base name. Never assemble a sentence from translated fragments; use
  placeholders.
- Keep the root item's `implicitWidth` / `implicitHeight` meaningful. A
  floating view is sized from the root; a fill view sizes the root.
- Declare delegate properties with `required property` so a missing role fails
  at load instead of rendering `undefined`.
- Avoid binding loops: do not measure a label against a sibling that is itself
  anchored to the label. Use `TextMetrics` for elided text, and read a reactive
  property alongside `itemAtIndex()` or the binding evaluates only once.
- A row-wide `MouseArea` that accepts presses takes an exclusive grab. Put it
  at `z: -1`, below the row's `TapHandler`s, or the handlers never fire.
- `TapHandler.tapped` fires again on a double tap. Use `singleTapped` and
  `doubleTapped` when the two gestures mean different things.
- A popup that closes on focus loss must outlive the click on one of its
  entries; defer the close by an event-loop turn.
- Under Wayland the application cannot read the global pointer or place
  top-level windows. Open popups and menus from a small anchor item at the
  click point. A `QQuickWidget` renders into an offscreen window, so a QML
  popup is positioned relative to that window and needs the view's offset in
  its top-level window; on other platforms the offset is zero. A menu that
  must reach past the view belongs in a native `QMenu` created by the bridge,
  not in a QML popup clipped to the view.
- Do not resize a `QQuickWidget` from `implicitHeightChanged` or another signal
  QML emits while laying out. The resize starts a frame inside a frame, both
  are dropped, and the view stays blank. Apply the new size on a zero-timer.
- Keyboard focus is per window. A press in a docked/floating QML view gives it
  the keyboard; in overlay mode the 3D view's window normally keeps it, so
  the host must `activateWindow()` and `setFocus()`. Declare items in reading
  order: QML's child order is the Tab order. Give every field and button a
  visible focus ring.
- When the keyboard leaves a view for good (Tab out, a click elsewhere, a
  shortcut), clear the scene's focus item so its ring does not stay lit. Do
  not do that when focus goes to the view's own popup or while the dialog is
  rebuilding.
- Animate opacity, not a colour through `"transparent"`; that is transparent
  black, not a fade.
- `Screen.width` is 0 until the view is shown. Guard geometry that reads it.
- Do not put `QWidget` or FreeCAD document logic in QML. QML is presentation;
  the bridge is the behaviour.

## 9. Task panels

A FreeCAD task class works with a `QQuickWidget` as its `form`:

```python
class MyTaskPanel:
    def __init__(self, obj) -> None:
        self.obj = obj
        self.doc = obj.Document
        self.bridge = TaskBridge(obj)
        self.form = make_view(
            None, "MyTask.qml", {"theme": theme, "task": self.bridge})
        self.bridge.setParent(self.form)
        self.doc.openTransaction("Edit feature")
        root = self.form.rootObject()
        root.implicitHeightChanged.connect(
            lambda: QtCore.QTimer.singleShot(0, self._fit_height))
        QtCore.QTimer.singleShot(0, self._fit_height)

    def _fit_height(self) -> None:
        root = self.form.rootObject()
        if root is not None:
            self.form.setFixedHeight(
                max(1, int(root.property("implicitHeight"))))

    def getStandardButtons(self) -> int:  # noqa: N802 - FreeCAD API
        buttons = QtWidgets.QDialogButtonBox.StandardButton
        return int((buttons.Ok | buttons.Cancel).value)

    def accept(self) -> bool:
        self.doc.recompute()
        self.doc.commitTransaction()
        Gui.Control.closeDialog()
        return True

    def reject(self) -> bool:
        self.doc.abortTransaction()
        self.doc.recompute()
        Gui.Control.closeDialog()
        return True
```

Then `Gui.Control.showDialog(MyTaskPanel(obj))`.

Rules:

- FreeCAD draws OK and Cancel; the QML form supplies the content only. Do not
  add another pair of buttons to the QML root.
- Use the same card layout and labels as the property inspector for the same
  feature, so editing in either place looks and works the same.
- Open the transaction when the panel opens, commit on accept, abort on
  reject. On reject, restore anything the preview changed. If construction
  fails after opening the transaction, abort it before re-raising. Wrap
  accept/reject so a failed recompute aborts the transaction instead of
  leaving it open.
- Remove selection observers, timers, and temporary view changes in a
  `finish()` path that both accept and reject call, even when an exception
  interrupts the edit.
- Fit the view to the root's `implicitHeight` on a zero-timer, never inside the
  signal that reported the change.
- The dialog owns the feature while it is open. If the panel edits FreeCAD's
  own task fields, go through those widgets; reading a property straight from
  the object gives the value before the dialog's edits.
- Keep one Tab sequence across the QML cards and the dialog's native buttons.
  A press in the view must activate its window and give it focus; when the
  field is done, return the keyboard to the 3D view.
- Translate the panel title and labels. `Gui.Control.closeDialog()` leaves the
  task view cleanly; if the panel also put a view provider into edit mode,
  call `Gui.ActiveDocument.resetEdit()`. `Gui.Control.activeDialog()` tells you
  whether another task is already open.

## 10. Dock panels, startup, and overlays

### Dock panel

```python
dock = QtWidgets.QDockWidget(translate("MyAddon", "My panel"), main)
dock.setObjectName("MyAddon_Panel")
dock.setWidget(make_view(dock, "MyPanel.qml", {
    "theme": theme, "panel": bridge,
}))
main.addDockWidget(QtCore.Qt.RightDockWidgetArea, dock)
```

- Give the dock a stable `objectName`; user layout persistence depends on it.
- Reuse an existing dock instead of creating a second one on every command.
- **Build QML only after the main window exists.** During `InitGui` the main
  window is not ready, so defer first creation:

```python
# init_gui.py
def _start() -> None:
    from .ui import panel
    panel.restore()


QtCore.QTimer.singleShot(0, _start)
```

- FreeCAD restores dock geometry only for docks that exist when its saved state
  is read. An addon dock created later must persist and restore its own
  placement, visibility, floating state, and size in the parameter store.
- Reads never raise: a missing or corrupt preference gives a working default,
  not a traceback at startup.

### Overlaying the 3D view

A dock is the default. Overlay only when the product needs it, and treat it as
advanced platform work:

- Put the `QQuickWidget` inside the active 3D view's GL widget as a child. Do
  not float a separate window over the view.
- Set `WA_TranslucentBackground`, `WA_AlwaysStackOnTop`, and a transparent
  clear colour together. This combination is fragile on some drivers; have a
  fallback.
- A translucent `QQuickWidget` does not repaint what is behind it when its
  content moves. Call `update()` on the view and its parent on scroll, drag,
  and any other movement, coalesced with a zero-timer.
- For hit testing, ask QML whether it draws at the point, and hand presses it
  does not take to the 3D view in the view's coordinates. When the pointer is
  over empty space, set `WA_TransparentForMouseEvents` so hover and the wheel
  reach the view natively. Do not build a mask; it goes stale on every scroll
  and relayout.
- Handle the wheel explicitly: over a drawn item it scrolls the panel,
  elsewhere it zooms the 3D view.
- The 3D view normally owns the keyboard. A press in the panel must activate
  its window and give the QML scene focus before an in-place edit can type.
- Reparenting a `QQuickWidget` to another top-level window puts it on a new
  QRhi while its scene graph still holds textures from the old one. Connect
  `ParentChange` and call `quickWindow().releaseResources()`.
- Wayland popups need the view's offset in its top-level window, as described
  in section 8.
- Re-read the 3D background colour when FreeCAD's view preferences change, and
  choose overlay ink against that background, not against the dock palette.

If any of this cannot be verified on the target platform, ship the dock mode
and document the limitation.

## 11. Tests

QML needs tests that run without a display and without FreeCAD. They catch
what `qmllint` cannot: binding loops, missing roles, and errors raised while a
delegate is actually created. The snippet assumes `QML`, `theme_stub`, and
`bridge_stub` are defined by the harness:

```python
import os

os.environ.setdefault("QT_QPA_PLATFORM", "offscreen")

from PySide6 import QtCore, QtGui, QtQuick  # noqa: E402
from PySide6.QtTest import QTest  # noqa: E402

app = QtGui.QGuiApplication.instance() or QtGui.QGuiApplication([])
problems = []


def handler(_mode, _context, message):
    text = str(message)
    print(text)
    bad = ("binding loop", "is not defined", "cannot read property",
           "unable to assign", "typeerror", "referenceerror",
           "cannot assign", "is not a function", "required property")
    if any(token in text.lower() for token in bad):
        problems.append(text)


QtCore.qInstallMessageHandler(handler)

view = QtQuick.QQuickView()
view.engine().addImportPath(str(QML))
view.rootContext().setContextProperty("theme", theme_stub)
view.rootContext().setContextProperty("panel", bridge_stub)
view.setResizeMode(QtQuick.QQuickView.ResizeMode.SizeRootObjectToView)
view.resize(320, 480)
view.setSource(QtCore.QUrl.fromLocalFile(str(QML / "MyPanel.qml")))
if view.status() == QtQuick.QQuickView.Status.Error:
    problems.extend(str(error.toString()) for error in view.errors())
view.show()
QTest.qWait(200)        # real time, not only event-loop turns
assert not problems
```

Rules:

- Use a real `QQuickView`, not a bare `QQmlComponent`. Items outside a window
  are never polished, so a `ListView` in a component realises no delegates and
  the harness passes while testing nothing.
- Create the application only if none exists; inside FreeCAD reuse its
  instance. Standalone tests use `QGuiApplication` with the offscreen platform.
- Call `view.show()` and let real time pass. Animations need wall-clock time;
  a `Behavior` that never advances leaves an item where it started, which
  looks exactly like a positioning bug.
- Build stubs from the real classes' declarations (`ast` over `@Property`,
  `@Slot`, signals, role names) so the harness cannot drift into testing an
  interface the panel does not have. Keep the context objects alive for the
  life of the view; dropping a context property while bindings still read it
  re-evaluates them against null.
- Test the behaviour a user notices: a row lands where it should, a control
  reaches the bridge, a popup opens at the click point. Do not assert on
  internal geometry that the next design change will move.
- Keep an audit test that needs no Qt:
  - every component used is present in a `qmldir` and imported;
  - every `bridge.*` / `theme.*` name exists in the Python class;
  - every `required property` on a delegate matches a model role;
  - every registered `qmldir` entry has a file, and every file is registered.
- Run `qmllint` if the target Qt ships it. It finds syntax and type issues, not
  runtime binding loops; keep both checks. It cannot see context properties
  (`theme`, `panel`), so do not treat unqualified-access warnings on them as
  failures.
- Run `pyside6-lupdate` over `*.qml` too, and test that `.qm` files match their
  `.ts` sources and placeholders survive.
- If `import PySide6` fails, run the test with FreeCAD's own interpreter before
  installing a second Qt into the environment.
- Manual only, on the target platform: look and feel, light and dark themes,
  dock placement/restore, overlay hit testing, Wayland popups, reload, and
  uninstall. Say what was not verified.

## 12. Packaging and release

- QML files are data: include them in the addon tree and in release zips.
  There is no manifest field for them; they load from
  `resources/qml/`.
- Load only QML files shipped with the addon. Do not use `Qt.createQmlObject`,
  `eval`, or a `Loader` source built from document or user input; QML is code
  execution, and Bandit does not scan it.
- Do not add `.qrc`, `.rcc`, a QML compiler step, or a runtime PySide6
  dependency. FreeCAD's Qt provides the modules; the addon ships source.
- Check the target FreeCAD's Qt build actually ships `QtQuickWidgets` and every
  QML module used. If not, show the QWidget fallback rather than failing
  startup.
- Create engines and load QML lazily. A QQuickWidget costs memory and starts a
  scene graph; startup should not pay for a panel the user never opens.
- On reload and uninstall, delete views, detach observers, stop timers, and
  restore any host settings the addon changed. A deleted C++ view with a live
  Python bridge or observer is a crash waiting for the next event.
- Test in a clean FreeCAD `Mod/` install: first open, reopen, workbench switch,
  language switch, theme switch, reload, and uninstall.

## 13. Review checklist

- [ ] QML is created only in GUI code, after the main window exists, and loaded
      lazily.
- [ ] Qt Quick modules come from `PySide6`; core Qt still comes from the
      `PySide` shim; imports are local to the GUI path.
- [ ] `QQuickWidget` import path and context properties are set before
      `setSource`; load errors are printed and shown in a fallback widget.
- [ ] Every context object has a live owner; none can be garbage-collected
      while bindings still read it.
- [ ] One bridge per surface; QML never touches FreeCAD documents or owns
      transactions.
- [ ] No model rebuild or view teardown from inside a QML handler; deferred
      with `Qt.callLater` or a zero-timer.
- [ ] No context property is nulled on teardown; the widget is deleted instead.
- [ ] Models are flat with role names; rebuilds are diffed; `dataChanged` is
      targeted and emitted after state changes.
- [ ] Theme colours and metrics come from a live palette-backed `Theme` object;
      no hard-coded colours or font sizes; light and dark both read.
- [ ] Controls, if any, use `QtQuick.Controls.Basic` and are restyled.
- [ ] User-facing text is `qsTr()`; language changes call
      `engine.retranslate()`; QML is included in `lupdate`.
- [ ] Task panels open a transaction, commit/abort correctly, fit height on a
      zero-timer, and keep one Tab sequence with FreeCAD's buttons.
- [ ] Floating/overlay views handle transparency, repaint-behind, hit testing,
      keyboard focus, Wayland offsets, and `releaseResources()` on reparent.
- [ ] Headless QML tests load a real `QQuickView`, catch binding loops and
      missing roles, and keep the reference audit passing.
- [ ] QML loads only files shipped with the addon; no `eval` or
      `Qt.createQmlObject` on untrusted input.
- [ ] Manual checks on the target platform are done and reported, including
      reload and uninstall.

## 14. Bad practices to reject

| Pattern | Why it breaks | Use instead |
|---|---|---|
| Building QML at import or during `InitGui` | Main window and GUI may not exist; startup cost even if unused | `QTimer.singleShot(0, ...)` from `init_gui`, lazy creation |
| `QApplication.palette()` for QML colours | Ignores FreeCAD's stylesheet; wrong colours in light/dark | Palette from a polished reference widget |
| Hard-coded hex colours, fonts, pixel sizes | Breaks themes, HiDPI, density settings | `Theme` properties and font metrics |
| QML calling FreeCAD or document APIs | Unstable, untestable, not reload-safe | One bridge `QObject` |
| `setSource(QUrl())` or null context properties as teardown | Delegates outlive the source; TypeErrors per binding | Delete the `QQuickWidget` |
| `beginResetModel()` on every document change | Loses selection/scroll, flashes, rebuilds delegates | Diff and insert/remove rows |
| Emitting `dataChanged` before assigning state | QML reads the old value | Assign, then emit |
| Rebuilding a model from a delegate's handler | `qFatal`: "This behavior is NOT supported!" | `Qt.callLater` / zero-timer |
| Resizing `QQuickWidget` inside `implicitHeightChanged` | Nested render frames; blank view | Zero-timer |
| `QQuickWidget` overlay without all transparency attributes | Black background | Attributes and clear colour together |
| QML popups placed with global coordinates on Wayland | Misplaced menus | Anchor item + view offset; native menu if needed |
| `QQmlComponent` as the QML test harness | No delegates, no layout, passes while broken | Offscreen `QQuickView` + `show()` + real time |
| Default `QtQuick.Controls` style | Host style leaks into the addon | `QtQuick.Controls.Basic` + theme |
| `QApplication.processEvents()` | Re-entrancy, invalidated objects | Queued timers, chunked work |
| Static `QTimer.singleShot` with a bound method | Fires into a deleted object after reload | Parented `QTimer` |
| Runtime PySide6 dependency | Conflicts with FreeCAD's Qt build | Use FreeCAD's binding; dev-only test dependency |
| `.qrc` / `pyside6-rcc` resources | Qt-major-version lock and build steps | Plain QML files under `resources/qml/` |
