# Observers, events, and lifecycle

FreeCAD addons should react to documents, objects, selection, transactions, and
the 3D view through observers and callbacks. Polling loops and global state
checks are slower, miss events, and leak references.

This reference covers the raw FreeCAD observer APIs and the lifecycle rules that
make them safe.

## 1. GUI vs console

- `App.GuiUp` is `False` in `FreeCADCmd`; GUI observers and widgets must not be
  created then.
- `init_gui.py` only runs in GUI mode, so it is the natural place to install
  GUI-only observers.
- If a shared module must install GUI observers, guard with `App.GuiUp` and
  defer with `QTimer.singleShot(0, ...)`; do not assume a GUI-up event exists in
  every FreeCAD version.
- Remove `FreeCADGui`, `PySide`, and `pivy` imports from console-safe modules.

## 2. Document observers

Register one observer object and keep a reference:

```python
import FreeCAD as App


class DocumentObserver:
    def slotCreatedDocument(self, doc):
        App.Console.PrintLog(f"Document created: {doc.Name}\n")

    def slotDeletedDocument(self, doc):
        # Drop cached references to this document.
        pass

    def slotChangedObject(self, obj, prop):
        # A document object property changed.
        pass

    def slotRecomputedObject(self, obj):
        # The object finished recomputing; invalidate caches.
        pass


observer = DocumentObserver()
App.addDocumentObserver(observer)

# On teardown:
# App.removeDocumentObserver(observer)
```

### Document observer slots

| Slot | When it fires |
|---|---|
| `slotCreatedDocument(doc)` | A document was created. |
| `slotDeletedDocument(doc)` | A document is being deleted/closed. |
| `slotRelabelDocument(doc)` | Document label changed. |
| `slotActivateDocument(doc)` | Document became active. |
| `slotRecomputedDocument(doc)` | Document recompute finished. |
| `slotBeforeRecomputeDocument(doc)` | Recomputed is about to start. |
| `slotUndoDocument(doc)` / `slotRedoDocument(doc)` | Undo/redo applied. |
| `slotChangedDocument(doc, prop)` | Document property changed. |
| `slotBeforeChangeDocument(doc, prop)` | Document property is about to change. |
| `slotStartSaveDocument(doc, path)` / `slotFinishSaveDocument(doc, path)` | Save start/finish. |
| `slotOpenTransaction(doc, name)` | Transaction opened. |
| `slotCommitTransaction(doc)` | Transaction committed. |
| `slotAbortTransaction(doc)` | Transaction aborted. |
| `slotBeforeCloseTransaction(doc)` / `slotCloseTransaction(doc)` | Transaction closing. |
| `slotCreatedObject(obj)` | Object added to a document. |
| `slotDeletedObject(obj)` | Object removed. |
| `slotChangedObject(obj, prop)` | Object property changed. |
| `slotBeforeChangeObject(obj, prop)` | Object property is about to change. |
| `slotRecomputedObject(obj)` | Object recompute finished. |
| `slotAppendDynamicProperty(obj, prop)` / `slotRemoveDynamicProperty(obj, prop)` | Dynamic property added/removed. |
| `slotChangePropertyEditor(obj, prop)` | Property editor mode changed. |
| `slotBeforeAddingDynamicExtension(obj, ext)` / `slotAddedDynamicExtension(obj, ext)` | Dynamic extension added. |

Rules:

- Keep slot handlers cheap. They run during user actions and recomputes.
- Document/object observer callbacks can be triggered during recompute; do not
  assume the GUI thread. Defer widget/Coin work with `QTimer.singleShot(0, ...)`.
- Do not mutate the document from a `slotChanged*` handler unless you guard
  against re-entrancy; prefer deferring with `QTimer.singleShot(0, ...)`.
- `slotDeletedDocument` / `slotDeletedObject` must release cached references and
  cancel pending work for that document/object.
- Do not keep a strong reference to a document or object in a long-lived global
  observer; store names or weak references and resolve on demand.
- `slotBeforeRecomputeDocument` is not a place to start heavy work; it runs in
  the recompute path.

## 3. Selection observers

```python
import FreeCADGui as Gui


class SelectionObserver:
    def addSelection(self, doc, obj, sub, pnt):
        # User selected an object or sub-element.
        pass

    def removeSelection(self, doc, obj, sub):
        pass

    def clearSelection(self, doc):
        pass

    def setSelection(self, doc):
        pass

    def setPreselection(self, doc, obj, sub):
        # Fires very often while moving the mouse. Keep it trivial.
        pass

    def removePreselection(self, doc, obj, sub):
        pass

    def pickedListChanged(self):
        pass


observer = SelectionObserver()
Gui.Selection.addObserver(observer)  # optionally: Gui.Selection.ResolveMode.NoResolve

# On teardown:
# Gui.Selection.removeObserver(observer)
```

Selection rules:

- `setPreselection` / `removePreselection` fire on hover and can be very
  frequent. Do not run expensive work there.
- Selection callbacks are GUI-thread activities. Keep them fast; defer heavy
  work with `QTimer.singleShot(0, ...)`.
- Check that `doc` matches the document your tool is operating on before
  updating state.
- Selection callbacks receive names (`doc`, `obj`, `sub`), not live objects.
  Resolve them with `App.getDocument(doc).getObject(obj)` and
  `getSubObject(sub)`, and handle already-deleted objects.
- Use `Gui.Selection.getSelectionEx()` when you need sub-element names and pick
  points; `getSelection()` only returns objects.
- Resolve sub-elements lazily. `getSelectionEx()` objects and `SubElementNames`
  can be expensive to resolve.
- Preserve and restore selection when a command temporarily changes it.
- Remove the observer when the owning widget or tool is destroyed (see below).

### Auto-remove on widget destruction

If the observer belongs to a dialog/panel, remove it when the widget dies:

```python
widget.destroyed.connect(lambda *_: Gui.Selection.removeObserver(observer))
```

This avoids a callback into a deleted Qt widget.

### Typed, ordered selection

A robust command validates the selection before doing work:

```python
def selected_types(*patterns):
    sel = Gui.Selection.getSelectionEx()
    # Match obj.Object.TypeId against patterns in the requested order.
    ...
```

Use the pattern list from most specific to least specific, and return a
boolean plus the objects in the requested order. Do not rely on selection order
unless you explicitly sort/validate it.

## 4. Transaction events

A document observer can watch transactions:

```python
class TransactionObserver:
    def slotOpenTransaction(self, doc, name):
        pass

    def slotCommitTransaction(self, doc):
        pass

    def slotAbortTransaction(self, doc):
        pass
```

Use these for UI state, logging, grouping, or cache invalidation. Do not use
`slotBeforeCloseTransaction` to perform more document edits; it is called while
the transaction is closing.

For application code, prefer a transaction context manager:

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

A command can intentionally abort with an exception or a controller object, but
the document must never be left mid-transaction.

## 5. Object-level callbacks

For a FeaturePython proxy, prefer object-level callbacks over a global observer:

| Callback | Use |
|---|---|
| `execute(obj)` | Recompute derived outputs. |
| `onBeforeChange(obj, prop)` | Capture old values / validate before change. |
| `onChanged(obj, prop)` | React to a property change; keep cheap. |
| `onDocumentRestored(obj)` | Migrate schema/state after load. |
| `unsetupObject(obj)` | Release resources when removed. |
| `dumps()` / `loads(state)` or `__getstate__` / `__setstate__` | Serialize proxy state; prefer properties. |
| `getViewProviderName(obj)` | Override the ViewProvider type if needed. |

Rules:

- Do not call `doc.recompute()` from `onChanged`; setting a property already
  marks the object dirty.
- Do not open transactions in `execute()`.
- `onChanged` runs during initialization before all properties exist; guard with
  `hasattr`.
- ViewProvider `updateData()` is for visual updates; `execute()` must not touch
  Qt/Coin.
- If a callback needs to do heavy work, schedule it once with
  `QTimer.singleShot(0, ...)`, and collapse duplicate requests.

## 6. View and dragger callbacks

`View3DInventorPy` supports 3D view event callbacks:

```python
view = Gui.activeDocument().activeView()

view.addEventCallback("SoMouseButtonEvent", on_mouse)
view.addEventCallbackPivy(coin.SoMouseButtonEvent.getClassTypeId(), on_mouse)
view.addDraggerCallback(dragger, "addFinishCallback", on_drag_finish)
```

- `addEventCallback` takes a string event name.
- `addEventCallbackPivy` takes a Coin event type.
- `addDraggerCallback` attaches a callback to a Coin dragger.
- Remove the matching callback when the tool/view is torn down:
  `removeEventCallback`, `removeEventCallbackPivy`,
  `removeDraggerCallback`.
- The active view can change or be deleted. Reacquire the view and check C++
  validity before using a stored reference.
- View callbacks run on the GUI thread; still keep them short.
- Do not start a transaction on every mouse move. Start on drag start, commit on
  finish, abort on cancel.

## 7. Subscription lifecycle

Whatever observer style you use, follow these rules:

1. **Keep the observer instance.** FreeCAD removes observers by identity; a
   temporary object cannot be removed later.
2. **Remove the exact instance.** `removeObserver(a)` does not remove `b`, even
   if they have the same methods.
3. **Tie lifetime to an owner.** A widget, document, workbench, or tool should
   own its observer and remove it on destruction. Use the Qt `destroyed` signal
   or an explicit `close()`/`uninstall()`.
4. **Do not rely on `__del__`.** It can run late, run during interpreter
   shutdown, or not run at all. Use explicit teardown.
5. **Weak references for listeners.** If you build your own event bus, store
   weak references to bound methods or explicit subscription handles, and
   support unsubscribe.
6. **One-shot initialization.** If initialization must happen once, guard with a
   boolean and make it idempotent. `App.GuiUp` plus a deferred check is safer
   than assuming startup ordering.
7. **Clear caches on delete.** `slotDeletedDocument` / `slotDeletedObject` are
   the correct places to drop cached paths, names, and weak refs.
8. **No global leaks.** Do not store `Document` or `DocumentObject` instances in
   module-level variables; store names and resolve them, or store weak refs.

## 8. Debounce and coalesce

High-frequency events (selection hover, property changes, document changes) can
arrive in bursts. Collapse them:

```python
_pending = False


def on_selection_changed(*_):
    global _pending
    if _pending:
        return
    _pending = True
    QtCore.QTimer.singleShot(0, _update)


def _update():
    global _pending
    _pending = False
    # Do the expensive work once per event-loop turn.
```

This is preferable to `QApplication.processEvents()` and preferable to a polling
timer. If you truly need polling, make it cheap, fixed-rate, and centralized;
do not create one timer per widget.

## 9. C++ object validity

Qt/Coin objects can be deleted by C++ while Python still holds a wrapper. Before
using a cached widget, view, camera, dragger, or document object:

```python
from shiboken6 import isValid  # shiboken2 on Qt5 builds

if not isValid(widget):
    return
```

Centralize this in one compatibility helper. Do not call methods on stale
wrappers; it can crash the process rather than raise a Python exception.

## 10. Testing observers

- Create a document in `setUp`; close it in `tearDown`.
- Register an observer, trigger a property change/recompute/selection, and assert
  the callback ran.
- Remove the observer in `tearDown` even if the test failed.
- Use `QSignalSpy` or a small recording observer for Qt signals.
- Do not leave global observers registered between tests; they can affect later
  tests and keep documents alive.
- Run GUI observer tests with `FreeCAD --run-test`; console tests with
  `freecadcmd --run-test`.

```python
class TestObserver(unittest.TestCase):
    def setUp(self):
        self.doc = App.newDocument("ObserverTest")
        self.observer = RecordingObserver()
        App.addDocumentObserver(self.observer)

    def tearDown(self):
        App.removeDocumentObserver(self.observer)
        App.closeDocument(self.doc.Name)
```

## 11. Common observer mistakes

- Registering a new observer on every command invocation, then never removing
  the old ones.
- Removing a different instance than the one registered.
- Doing heavy work in `setPreselection` or `slotChangedObject`.
- Mutating the document from a change callback, causing re-entrancy.
- Keeping strong references to deleted documents/objects.
- Using `__del__` for cleanup.
- Assuming the active view/document never changes.
- Using selection order without validating it.
- Forgetting `App.GuiUp` and crashing in console mode.
- Calling `QApplication.processEvents()` inside a callback.
- Polling the whole document instead of listening to the relevant event.
- Not clearing caches on document close or object deletion.
- Leaving test observers registered across tests.
