# Preferences, input widgets, progress, and Qt lifecycle

This reference covers settings and user-facing UI beyond the basic command/workbench
material in `references/ui-and-commands.md`.

## 1. Parameter store basics

FreeCAD preferences live in the parameter store, not in `QSettings` or a random
JSON file:

```python
import FreeCAD as App

GROUP = "User parameter:BaseApp/Preferences/Mod/MyAddon"
params = App.ParamGet(GROUP)

enabled = params.GetBool("EnableFoo", True)
count = params.GetInt("Count", 10)
ratio = params.GetFloat("Ratio", 0.5)
label = params.GetString("Label", "")
```

Rules:

- Always pass a default.
- Match the getter/setter to the stored type.
- Remove values with `RemBool` / `RemInt` / `RemFloat` / `RemString` when the
  user resets a setting.
- Use nested groups for logical sections or presets:
  `User parameter:BaseApp/Preferences/Mod/MyAddon/Editor`.
- Do not store large data blobs in preferences. Use files under
  `App.getUserAppDataDir()` and validate paths.
- Do not cache preferences forever. Read on use, or subscribe to changes.

## 2. Two valid preference-page styles

### A. Qt Designer `.ui` page

Use FreeCAD `Gui::Pref*` widgets (`Gui::PrefCheckBox`, `Gui::PrefComboBox`,
`Gui::PrefSpinBox`, `Gui::PrefDoubleSpinBox`, `Gui::PrefLineEdit`,
`Gui::PrefColorButton`, ...). Each widget needs:

- `prefPath`: usually `Mod/MyAddon`.
- `prefEntry`: the key name.

Register once:

```python
import FreeCADGui as Gui

Gui.addPreferencePage(
    str(Path(__file__).parent / "resources" / "ui" / "preferences.ui"),
    "MyAddon",
)
```

FreeCAD reads and writes the values automatically when the page is opened and
applied. This is the standard, lowest-code option for a static form.

### B. Programmatic preference page

For dynamic pages, complex validation, presets, or generated widgets, register
a Python class with a `form` widget and `loadSettings` / `saveSettings`:

```python
class MyAddonPreferencesPage:
    def __init__(self, _parent=None):
        self.form = build_my_form()  # a QWidget

    def loadSettings(self):
        # Read from App.ParamGet and populate widgets.
        pass

    def saveSettings(self):
        # Validate first; if invalid, show a message and do not partially save.
        # Then write all values to App.ParamGet.
        pass


Gui.addPreferencePage(MyAddonPreferencesPage, "MyAddon")
```

Rules:

- `loadSettings()` is called when the page opens.
- `saveSettings()` is called when the user applies/accepts. Validate everything
  before writing; a half-saved page is worse than a rejected one.
- Keep the page class constructible without a running GUI only if you also guard
  `App.GuiUp`; normally it is registered from `init_gui.py`.
- The `form` widget must be parented/owned by FreeCAD's preferences dialog.

## 3. Declarative preferences (optional pattern)

If the addon has many settings, a small descriptor layer keeps reads, writes,
defaults, UI grouping, and validation in one place. The general shape:

```python
@dataclass
class Preference:
    group: str
    name: str
    default: object = None
    value_type: type | None = None
    options: dict[object, object] | None = None
    label: object | None = None
    description: object | None = None
    ui_group: object | None = None
    ui_page: object | None = None
    ui_section: object | None = None
    validators: list | None = None

    def read(self):
        group = App.ParamGet(f"User parameter:BaseApp/Preferences/{self.group}")
        if self.value_type is bool:
            return group.GetBool(self.name, self.default)
        if self.value_type is int:
            return group.GetInt(self.name, self.default)
        if self.value_type is float:
            return group.GetFloat(self.name, self.default)
        return group.GetString(self.name, self.default)

    def write(self, value):
        group = App.ParamGet(f"User parameter:BaseApp/Preferences/{self.group}")
        if self.value_type is bool:
            group.SetBool(self.name, bool(value))
        elif self.value_type is int:
            group.SetInt(self.name, int(value))
        elif self.value_type is float:
            group.SetFloat(self.name, float(value))
        else:
            group.SetString(self.name, str(value))
```

Use this to generate the preference page widgets and to read values with a
single call. Keep it internal; do not expose a second configuration format.

### Presets

Store presets in a nested group:

```text
User parameter:BaseApp/Preferences/Mod/MyAddon/presets/<PresetName>
```

- `Default` is the normal group, not a preset group.
- Saving a preset copies all declared values into the preset group.
- Renaming = save under the new name, then remove the old group.
- Deleting removes the preset group.
- Do not delete or rename `Default`.
- Validate before saving a preset.
- Update the preset selector after save/rename/delete.

Common validators: `required`, `min`, `max`, `min_length`, `max_length`, and
`regex`. Validate all fields before writing anything; show an inline error near
the invalid field and abort the save.

### Deferred translation

Class-level labels and option dicts are evaluated before the translator may be
ready. A deferred translation wrapper stores `(context, source)` and translates
on `__str__`:

```python
class DeferredTranslation:
    def __init__(self, context, source):
        self.context = context
        self.source = source

    def __str__(self):
        return App.Qt.translate(self.context, self.source)
```

Use it for preference labels/descriptions/options and other constants. At
runtime, `App.Qt.translate(context, "literal")` is still the normal path.
`QT_TRANSLATE_NOOP` only marks a literal for extraction; it does not translate.

## 4. Live preference updates

`App.ParamGet(...).AttachManager(observer)` calls
`observer.slotParamChanged(group, value_type, name, value)` when a parameter in
that group changes:

```python
class PreferenceWatcher:
    def slotParamChanged(self, group, value_type, name, value):
        if name == "EnableFoo":
            update_ui(value)


watcher = PreferenceWatcher()
params = App.ParamGet("User parameter:BaseApp/Preferences/Mod/MyAddon")
params.AttachManager(watcher)

# On teardown:
# params.Detach(watcher)
```

Rules:

- Keep the watcher object alive; keep a reference to the parameter group too if
  needed.
- Detach it when the owning widget/tool is destroyed.
- Do not write preferences from inside `slotParamChanged` without a re-entrancy
  guard; it can loop.
- Apply live changes idempotently; the callback may fire for values you did not
  expect.
- If a preference only matters at next startup, say so in the UI description
  instead of pretending it is live.

## 5. Unit-aware and expression-aware inputs

FreeCAD properties can carry units and expressions. Prefer a unit-aware input
over a plain float where the value has a unit:

- Use `Gui::InputField` for a quantity input.
- Use `App.Units.Quantity` when setting/reading a quantity.
- Use `Gui.ExpressionBinding(widget).bind(obj, property_name)` to connect an
  input to a document property and its expression engine. `setAutoApply()`
  controls when the value is applied.

Rules:

- Do not strip units by converting everything to `float`; store `PropertyLength`,
  `PropertyAngle`, `PropertyQuantity`, etc.
- Validate that the quantity has the expected unit before writing.
- For a property-bound widget, recompute the object after applying (or let the
  expression binding do it), and refresh dependent views. If the property feeds
  a `VarSet` or expression consumer, recompute that consumer too.
- Do not create a second binding loop: one binding per property/widget pair.

### Language changes at runtime

Some panels receive `QEvent.LanguageChange`; re-apply labels, tooltips, section
titles, and combo-box item texts there. FreeCAD does not automatically rebuild
custom pages on a language switch.

- Connect `changeEvent` (or a language-change signal) and call a single
  `apply_translations()` method.
- Re-read deferred translation wrappers with `str(...)`, not at construction.
- If a page cannot update live, say that a restart is required in its
  description instead of leaving stale text.
- Do not translate internal keys or preserve display text as the stored value.

## 6. Widget and form conventions

- Use layouts (`QVBoxLayout`, `QHBoxLayout`, `QGridLayout`, `QFormLayout`,
  `QSplitter`, `QScrollArea`, `QTabWidget`); never absolute positioning.
- Give important widgets an `objectName`; saved UI state and tests rely on it.
- Use a consistent value protocol for reusable inputs: `value()` and
  `setValue(value)`.
- Put labels beside/before inputs; set tooltips for non-obvious settings.
- Translate labels, tooltips, and messages; do not translate internal keys.
- Use a combo box's item data for the stored value, not the display text:
  `addItem(label, value)`, `currentData()`.
- Use a read-only table for output; use a model/view for large editable data.
- Use `QScrollArea` for long preference pages; `QTabWidget`/`QSplitter` for
  complex tools.
- Use `QDialogButtonBox` with standard buttons instead of hand-rolled OK/Cancel.
- Use `QFileDialog` for paths; validate the result and use a sensible start
  directory.
- Use `QMessageBox` for user-facing warnings/errors; use the Report view for
  diagnostics.
- Use `QIcon.fromTheme()` with an SVG fallback; do not hardcode colors or fonts.
- Set a stable `objectName` on docks/dialogs and reuse existing instances instead
  of creating duplicates.
- For panel state, persist explicit values (by `objectName`) in the parameter
  store; do not serialize whole widget trees or rely on transient widget state.

## 7. Custom widgets and painting

Prefer standard Qt widgets. When a standard widget cannot express the result:

- Subclass `QWidget` and implement `paintEvent`. Create `QPainter(self)`, call
  `save()` before changing state, and `restore()`/`end()` in `finally`.
- Use `QSvgRenderer` for SVG assets; scale with the widget's `viewBoxF()` and
  aspect ratio. Use `QPixmap` only for raster images and enable
  `SmoothPixmapTransform`.
- Use `QPlainTextEdit` for logs and large text; `QTextEdit`/`QLabel` for rich
  text/HTML; avoid enormous HTML documents.
- Use `QTableWidget` for small read-only tables; use a model/view
  (`QAbstractTableModel` + `QTableView`) for large or editable data.
- Use `QScrollArea` for long content, but it holds one widget: put a container
  with a layout inside it.
- Use `QSplitter` for resizable panes and `QTabWidget` for grouped pages.
- Do not paint outside `paintEvent`; do not keep a `QPainter` beyond it.
- Avoid per-frame allocations in `paintEvent`; cache pens/fonts/pixmaps when
  possible.

## 8. Dialog and task-panel lifecycle

- Parent dialogs to `Gui.getMainWindow()` or the active window.
- Keep a Python reference to non-modal dialogs; otherwise they can be garbage
  collected.
- Use `deleteLater()` and clear references on close.
- For a task panel, the FreeCAD contract is a `form` widget plus `accept()` and
  `reject()`:

```python
class MyTaskPanel:
    def __init__(self):
        self.form = build_my_form()

    def accept(self):
        Gui.Control.closeDialog()
        if doc := App.ActiveDocument:
            doc.recompute()
        Gui.ActiveDocument.resetEdit()
        return True

    def reject(self):
        Gui.Control.closeDialog()
        return True
```

- `Gui.Control.activeDialog()` tells you whether a task dialog is open.
- `Gui.Control.closeDialog()` closes the active task panel.
- Do not leave the document in edit mode after a task panel closes;
  `Gui.ActiveDocument.resetEdit()` is the cleanup call.
- A task panel that modifies the document must participate in the command's
  transaction or create its own.

### Task panels that recompute (live preview)

FreeCAD pumps Qt events while it recomputes: progress reporting
(`Gui::SequencerBar::setProgress`) calls `processEvents()` from inside OCC
operations. So the panel's own timers, queued slots, selection callbacks and
dialog buttons can run *in the middle of* the panel's `doc.recompute()`. If one
of them runs geometry (a boolean such as `Shape.cut()`, a shape query that
computes, or a nested recompute), it can wait forever on OCC's
`OSD_ThreadPool`, which the paused outer boolean still holds: FreeCAD freezes
with every thread idle. Headless tests do not reproduce it, because nothing
pumps events there.

Rules:

- Wrap every recompute the panel starts in a gate. While it is held, handlers
  and timers queue their work instead of running it.
- Never drop queued work: run it, in order, when the outermost hold ends, so the
  panel catches up with the last value the user typed.
- Run a deferred OK/Cancel/Apply last, after the whole queue (including work
  queued while a queued item recomputed). Otherwise a click made before OK is
  lost when OK closes the panel.
- While the gate is held, `accept()` / `reject()` return `False` (the dialog
  stays open); the deferred button closes it with `Gui.Control.closeDialog()`
  when it succeeds.
- Re-check that the panel is still open after each recompute before touching
  its widgets.

The gate below is the code MaxProFrame runs in three task panels, with its
long docstrings shortened: [`recompute_gate.py`](https://github.com/sclMAX/MaxProFrame/blob/main/freecad/maxproframe/recompute_gate.py),
tested by `tests/test_recompute_gate*.py` and `tests/test_*_recompute_gate.py`
in the same repository on FreeCAD 1.1 and a 27.1 development build.

```python
import contextlib
from functools import partial

import FreeCADGui as Gui


class RecomputeGate:
    """Held while a panel recomputes; what the panel is asked to do meanwhile
    waits for the hold to end. ``active`` tells whether the panel still runs:
    queued work is dropped once it closed."""

    def __init__(self, active):
        self._active = active
        self._depth = 0
        self._waiting = []
        self._button = None
        # Whether the queue is being run: a hold ending meanwhile leaves what
        # it queued to that run.
        self._releasing = False

    @property
    def held(self):
        return self._depth > 0

    @contextlib.contextmanager
    def hold(self):
        """Hold the gate for the block (a recompute of the panel and its
        follow-up); then run what waited."""
        self._depth += 1
        try:
            yield
        finally:
            self._depth -= 1
        if not self._depth:
            self._release()

    def wait(self, work):
        """Whether ``work`` waits for the recompute to end; an equal work
        queued already runs once."""
        if not self.held:
            return False
        if work not in self._waiting:
            self._waiting.append(work)
        return True

    def defer(self, button):
        """Whether a dialog button waits; the last one pressed runs after the
        queued work and closes the dialog when it returns True."""
        if not self.held:
            return False
        self._button = button
        return True

    def clear(self):
        """Drop what waits: the panel closed."""
        self._waiting = []
        self._button = None

    def _release(self):
        """Run what waited, in order, then the button, only from the outermost
        release. A work that raises does not stop the others nor the button:
        the first error is raised once they ran."""
        if self._releasing:
            return
        self._releasing = True
        error = None
        try:
            while self._waiting or self._button is not None:
                if self._waiting:
                    work = self._waiting.pop(0)
                else:
                    work, self._button = partial(self._press, self._button), None
                if not self._active():
                    continue
                try:
                    work()
                except Exception as exception:
                    error = error or exception
        finally:
            self._releasing = False
        if error is not None:
            raise error

    @staticmethod
    def _press(button):
        """Run the deferred ``button``; close the dialog when it accepts
        (FreeCAD took False for it meanwhile)."""
        if button() and Gui.Control.activeDialog():
            Gui.Control.closeDialog()
```

How a panel uses it (a sketch; see `create_extend_cut_tool.py` in the same
repository for a complete panel):

```python
class PreviewPanel:
    def __init__(self):
        self._closed = False
        self._gate = RecomputeGate(lambda: not self._closed)

    def update_preview(self):  # slot of the panel's preview timer
        if self._gate.wait(self.update_preview):
            return
        with self._gate.hold():
            ...  # set the preview objects from the fields
            self.doc.recompute()

    def accept(self):
        if self._gate.defer(self.accept):
            return False
        ...  # commit; the panel is closing
        self._closed = True
        self._gate.clear()
        return True
```

#### Previews through `PreviewExtension` (FreeCAD 27+)

Since [FreeCAD#29641](https://github.com/FreeCAD/FreeCAD/pull/29641) (development
builds from August 2026, not FreeCAD 1.1) a Python feature can add
`Part::PreviewExtensionPython` and publish its preview from
`recomputePreview()`, without recomputing the document. When an addon targets
those versions, prefer it for live previews: the panel no longer needs a
`doc.recompute()` per keystroke, which removes most of the window described
above.

It does not make the gate obsolete:

- Panels that preview by recomputing real document objects (the only option on
  FreeCAD 1.1) still need it.
- `PreviewUpdateScheduler` runs `updatePreview()` from a queued call, with no
  check for a running recompute. Queued calls are what the progress bar's
  `processEvents()` delivers, so a preview could start inside a recompute that
  reports progress. Keep `recomputePreview()` free of booleans (show the tool
  or the affected volume, as the framework's documentation suggests), so it
  never waits on OCC's thread pool.

## 9. Progress and long operations

Use FreeCAD's progress indicator rather than a custom busy loop:

```python
from FreeCAD import Base

bar = Base.ProgressIndicator()
bar.start("Working...", steps)
try:
    for item in items:
        # ... work ...
        bar.next()
finally:
    bar.stop()
```

A context manager wrapper is safer:

```python
@contextmanager
def progress(message, steps=0):
    bar = Base.ProgressIndicator()
    bar.start(message, steps)
    try:
        yield bar
    finally:
        bar.stop()
```

Rules:

- Always stop the indicator in `finally`.
- Do not call `QApplication.processEvents()` inside the operation. It permits
  re-entrancy and can invalidate objects mid-operation.
- For long tasks, chunk the work and let the event loop breathe between chunks
  only through a deliberate mechanism (modal progress, task panel, or scheduled
  callbacks), not `processEvents()`.
- Report errors after stopping the indicator; do not leave it running.
- If the operation is cancelable, check a cancel flag between chunks and abort
  the transaction cleanly.

## 10. Selection input widgets

A reusable object-picker follows the selection observer contract:

- A button toggles "pick mode".
- `setPreselection` updates a preview only while active.
- `addSelection` stores `(doc, obj, sub, point)` and optionally deactivates pick
  mode.
- `removeSelection` clears the value if it matches.
- `clearSelection`/`setSelection` handle global selection changes.
- Remove the observer when the widget is destroyed.
- Resolve the object/sub-element lazily; do not store raw document object
  wrappers indefinitely.
- For multi-selection, use a set/list of selection records and keep the display
  in sync.

Do not install one global observer per widget. Either share a dispatcher or
remove observers aggressively.

## 11. Qt compatibility and threading

### Binding

Runtime code uses FreeCAD's Qt shim:

```python
from PySide import QtCore, QtGui, QtWidgets
```

Do not import `PySide2`/`PySide6` at runtime. If the addon must support both Qt5
and Qt6, put enum/version differences in one compatibility module. For type
checking, import `PySide6` under `TYPE_CHECKING`.

A lazy compatibility module can defer loading Qt submodules until used; if you
do this, register the proxy modules in `sys.modules` so `from myqt.QtCore import
QObject` works. Keep this complexity in one file, not across the addon.

### UI thread bridge

- All widgets and Coin scene graphs live on the GUI thread.
- `execute()` runs on the GUI thread unless the object opts in to async
  recompute (see `document-objects.md`); then it must not touch GUI/Coin.
- To run a callback on the GUI thread from another thread, emit a signal of a
  `QObject` that lives on the GUI thread (queued connection), or use
  `QTimer.singleShot(0, gui_object, callback)` with such a context object. A
  context-free `QTimer.singleShot(0, callback)` uses the calling thread's event
  loop, so from a plain worker thread it never reaches the GUI thread. Do not
  call widget methods from a worker.
- `FreeCAD.Console.Print*` is safe from workers.
- Check `shiboken.isValid(obj)` (or `shiboken2` on Qt5) before using a cached
  Qt/Coin wrapper; C++ may have deleted it.
- Do not use `__del__` for cleanup; use explicit teardown or `destroyed`
  signals.
- Do not call `QApplication.processEvents()` to "make it responsive"; use a
  task panel, progress indicator, or chunked work.

## 12. Testing preferences and widgets

- Test parameter read/write with defaults and removal; don't pollute the user's
  real parameter store in tests. Use a test-specific group and clean it up.
- Test validation before saving; assert that invalid input does not partially
  write.
- Test preset save/rename/delete; assert `Default` cannot be removed.
- Test a programmatic page's `loadSettings`/`saveSettings` directly without
  showing the dialog where possible.
- Test selection widgets with a fake observer or the real `Gui.Selection` in a
  GUI test; remove observers in `tearDown`.
- Run GUI tests with `FreeCAD --run-test`; headless tests with
  `freecadcmd --run-test`.
- Do not leave task panels or dialogs open between tests.

## 13. Common preferences/UI mistakes

- Using `QSettings` instead of `App.ParamGet`.
- Reading a preference without a default.
- Writing a preference from a change callback without a re-entrancy guard.
- Saving a form partially before validation.
- Renaming/deleting the `Default` preset.
- Forgetting to detach a parameter manager or selection observer.
- Using display text as the stored combo-box value.
- Building forms with fixed positions instead of layouts.
- Hardcoding colors/fonts and breaking dark mode/HiDPI.
- Creating a new dialog/dock every invocation instead of reusing it.
- Not parenting dialogs to the main window.
- Using `processEvents()` inside a long operation.
- Touching widgets from a worker thread.
- Holding a stale C++ widget/view reference after deletion.
- Forgetting `Gui.ActiveDocument.resetEdit()` after a task panel.
- Leaving a progress indicator running after an exception.
