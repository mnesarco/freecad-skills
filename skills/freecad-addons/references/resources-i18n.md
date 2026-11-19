# Resources, icons, and translations

This reference covers how a FreeCAD 1.1+ addon ships and loads icons, other
assets, and translations. The short version: **ship plain files, register their
directories, and never compile Qt resources.**

## 1. Resource layout

Keep resources inside the addon package:

```text
freecad/MyAddon/
├─ resources/
│  ├─ icons/
│  │  ├─ logo.svg
│  │  ├─ do-thing.svg
│  │  └─ preferences-myaddon.svg
│  ├─ ui/
│  │  └─ preferences.ui
│  ├─ qml/
│  │  ├─ MyPanel.qml
│  │  └─ MyAddon/
│  │     └─ Controls/
│  │        ├─ qmldir
│  │        └─ Chip.qml
│  ├─ translations/
│  │  ├─ MyAddon.ts
│  │  ├─ MyAddon_de.ts
│  │  ├─ MyAddon_de.qm
│  │  └─ ...
│  └─ media/
│     └─ screenshot.png
├─ init_gui.py
└─ ...
```

- `resources/` is data, not a Python package. An `__init__.py` is optional and
  not needed for resource loading.
- Do not use a root-level `Resources/` directory in a new addon. It works, but
  it splits the package and makes `__file__`-relative paths awkward.
- `media/` is for screenshots and documentation assets; keep icons separate.

## 2. Icons and other assets

### Use SVG

- Icons must be SVG. They scale on HiDPI displays and adapt to theme changes.
- Do not use PNG/JPEG/WebP for icons. Bitmaps blur, and WebP depends on an
  optional Qt plugin that is not guaranteed to be present.
- The Addon Manager manifest icon must be SVG and should be under 16 kB.
- A 48×48 or 64×64 `viewBox` with a transparent background is a good default.
- Keep the source SVG editable; run an optimizer such as `scour` offline and
  commit the optimized output if you want smaller files. Do not hand-edit
  optimized SVGs.

### Register the icon path

In `init_gui.py`:

```python
from pathlib import Path

import FreeCADGui as Gui

_ICON_DIR = Path(__file__).parent / "resources" / "icons"
Gui.addIconPath(str(_ICON_DIR))
```

Then commands can use a bare filename:

```python
def GetResources(self):
    return {"Pixmap": "do-thing.svg", "MenuText": "Do thing"}
```

Absolute paths are also valid and avoid name collisions:

```python
_ICON = str(_ICON_DIR / "do-thing.svg")
```

Use absolute paths for the workbench `Icon` class attribute; it is read before
the workbench is registered.

### Other asset paths

Use `Path(__file__)` as the base for simple, unpacked addons:

```python
from pathlib import Path

_PKG = Path(__file__).parent
_UI = _PKG / "resources" / "ui" / "preferences.ui"
_DATA = _PKG / "resources" / "data" / "table.csv"
```

- Addon files are unpacked on disk by the Addon Manager; `Path(__file__)` works.
- Do not depend on the current working directory.
- Do not write into the addon directory at runtime. Use
  `App.getUserAppDataDir()` for user data and `App.getUserConfigDir()` for
  configuration.
- Validate paths read from documents, preferences, or user input before opening
  them.

### `importlib.resources` for packaged assets

For code that may be packaged differently, or to avoid assumptions about
`__file__`, use `importlib.resources`:

```python
from importlib import resources

pkg = resources.files("freecad.MyAddon")
icon = pkg / "resources" / "icons" / "logo.svg"
data = pkg / "resources" / "data" / "table.csv"

text = data.read_text(encoding="utf-8")
raw = data.read_bytes()

# Only if a real filesystem path is required:
with resources.as_file(icon) as path:
    use_path(path)
```

Rules:

- `as_file()` yields a temporary path; use it inside the context. Do not store
  it after the context exits, and do not register a temporary path with
  `Gui.addIconPath()` for later use.
- For long-lived icon/translation directories, use the real package directory
  (`Path(__file__).parent`) or copy assets to a persistent user directory.
- `resources.files()` accepts a module object or a dotted module name.
- Prefer `read_text`/`read_binary` over manual open/close.
- This pattern is robust if the addon is ever packaged as a zip/wheel; FreeCAD
  addons are normally unpacked, but the API is still cleaner for data files.

### Install resources once

Register icons/translations from a GUI-only entry point, once:

```python
import FreeCAD as App
import FreeCADGui as Gui
from pathlib import Path

_PKG = Path(__file__).parent
_installed = False


def install_resources():
    global _installed
    if _installed or not App.GuiUp:
        return

    icons = _PKG / "resources" / "icons"
    translations = _PKG / "resources" / "translations"
    if icons.is_dir():
        Gui.addIconPath(str(icons))
    if translations.is_dir():
        Gui.addLanguagePath(str(translations))
    Gui.updateLocale()
    _installed = True
```

- Do not install the same path repeatedly.
- Log installation to `App.Console.PrintLog` for debugging.
- Skip missing directories instead of registering invalid paths.
- Call this from `init_gui.py` or a one-shot GUI-up initialization; never from
  `__init__.py` in console mode.

### Do not ship compiled Qt resources

Do **not** use `.qrc` files, `pyside6-rcc`, `pyside-rcc`, or compiled `.rcc`
files. Compiled Qt resources are tied to the Qt major version that produced
them. A resource built with Qt5 tools cannot be loaded by a Qt6 runtime, and
FreeCAD builds differ across platforms and releases.

Consequences:

- Ship the plain SVG files and load them from the filesystem.
- In `.ui` files, do not use `:/icons/foo.svg` paths. Set icons in Python after
  `FreeCADGui.PySideUic.loadUi()`.
- Do not rely on `QResource` registration for addon assets.
- QML files are also plain files. Load them from `resources/qml/` with
  `QUrl.fromLocalFile` and an engine import path; do not compile them into a
  resource bundle or a Python module.

### Manifest icon

`package.xml`:

```xml
<icon>freecad/MyAddon/resources/icons/logo.svg</icon>
```

The path is relative to `package.xml`. The Addon Manager uses this icon for the
addon listing. For a workbench, the workbench `Icon` can be the same file.

### Preferences sidebar icon

FreeCAD looks for `preferences-<modulename>.svg` (all lowercase) on a registered
icon path. For a preferences group named `MyAddon`, ship
`preferences-myaddon.svg` and register its directory with `Gui.addIconPath()`.

## 3. Translations

### Runtime lookup and build-time extraction

Two functions cover most cases:

```python
import FreeCAD as App
from PySide.QtCore import QT_TRANSLATE_NOOP

translate = App.Qt.translate
```

- `translate(context, "literal")` performs the runtime lookup. It also marks
  the literal for `lupdate`, but only if the argument is a string literal.
- `QT_TRANSLATE_NOOP(context, "literal")` does nothing at runtime; it only marks
  the string for extraction. Use it for strings that are evaluated before the
  translator is installed, such as class attributes, module constants, and
  command resource dictionaries.

`lupdate` is a text scanner, not a Python interpreter:

```python
# Extracted and translated:
label = translate("MyAddon", "Do thing")

# NOT extracted (lupdate sees the variable name):
label = translate("MyAddon", some_variable)

# Extracted now, translated later by FreeCAD:
MENU_TEXT = QT_TRANSLATE_NOOP("MyAddon", "Do thing")
```

### QML text

Use `qsTr("literal")` in `.qml` files. The translation context is the QML
file's base name, so renaming a file changes its context and loses existing
translations.

- `pyside6-lupdate` scans `.qml` files like `.py` files; include them in the
  update/build commands and commit the resulting `.ts` / `.qm`.
- FreeCAD installs its translator application-wide, so `qsTr` works once
  `Gui.addLanguagePath()` and `Gui.updateLocale()` have run.
- FreeCAD can change language while it runs. On `QEvent.LanguageChange`, call
  `view.engine().retranslate()` on every live QML view (or route it through one
  i18n helper). Do not cache translated strings in Python models without
  refreshing them too.
- Do not build a sentence from translated fragments; translate the whole
  sentence with `%1` placeholders.

Details: `references/qml-and-qtquick.md` §7–8.

### Deferred translation

Class-level labels, descriptions, and option dictionaries are evaluated at import
time, often before translations are installed. A small deferred wrapper stores
the context and source literal and translates when converted to a string:

```python
class DeferredTranslation:
    def __init__(self, context, source):
        self.context = context
        self.source = source

    def __str__(self):
        return App.Qt.translate(self.context, self.source)

    def __repr__(self):
        return str(self)


LABEL = DeferredTranslation("MyAddon", "Enable feature")
```

Use this for preference labels/options and other constants. The literal still
has to be extractable by `lupdate`; if the wrapper is not recognized, also mark
the literal with `QT_TRANSLATE_NOOP` or configure `-tr-function-alias`.

Do not defer strings that must be translated at the exact moment they are read
and can change without a language change; use `translate()` directly there.

### Contexts

Use consistent, stable contexts:

| String kind | Context |
|---|---|
| General addon UI, dialogs, messages | Addon or module name, e.g. `"MyAddon"` |
| Command `MenuText` / `ToolTip` | Command name, e.g. `"MyAddon_DoThing"` |
| Workbench menu/toolbar names | `"Workbench"` |
| Property descriptions | `"App::Property"` |
| `.ui` file strings | Qt Designer sets the context; use the form's class/object context consistently |

Capitalization matters: `"MyAddon"` and `"myaddon"` are different contexts.

### What to translate

Translate user-visible UI:

- menu items, toolbar and workbench labels, tooltips;
- dialog labels, buttons, messages;
- property descriptions;
- object `Label` values.

Do **not** translate:

- console/log output (`PrintMessage`, `PrintLog`, ...);
- `print()` output;
- property names, class names, command IDs, internal identifiers;
- file-format keys, XML/JSON tags, protocol names;
- `"Separator"` in toolbar/menu lists.

Do not translate a document object's `Name`; translate its `Label` instead.

### Files and directories

Recommended:

```text
freecad/MyAddon/resources/translations/
├─ MyAddon.ts              # locale-agnostic template
├─ MyAddon_de.ts
├─ MyAddon_de.qm
├─ MyAddon_es.ts
├─ MyAddon_es.qm
└─ ...
```

- Name files `<AddonName>_<locale>.ts` and `<AddonName>_<locale>.qm`.
- Commit both `.ts` and `.qm`. The `.qm` is what FreeCAD loads at runtime; the
  `.ts` is what translators update.
- A root-level `Resources/translations/` is also accepted by FreeCAD, but keep
  one convention per project. The modern package-level location is simpler.
- Locale names may use hyphens (`pt-BR`); Qt tools may rewrite them to
  underscores in the `.ts` language field. Keep the filename convention
  consistent and test the locale.

### Register translations

In `init_gui.py`:

```python
from pathlib import Path
import FreeCADGui as Gui

Gui.addLanguagePath(str(Path(__file__).parent / "resources" / "translations"))
Gui.updateLocale()
```

- Call `addLanguagePath()` once at startup.
- Call `updateLocale()` after registering paths so the current language is
  applied.
- Do not load translations from `__init__.py` in console mode; there is no GUI
  language state to update.

### Generating `.ts` and `.qm`

Install PySide6 as a dev dependency so the Qt tools are available in the `uv`
environment:

```bash
uv add --dev PySide6
```

Qt 6 `lupdate` handles `.py`, `.ui`, and `.qml` files in one pass:

```bash
uv run pyside6-lupdate \
  freecad/MyAddon/**/*.py \
  freecad/MyAddon/resources/ui/*.ui \
  freecad/MyAddon/resources/qml/**/*.qml \
  -ts freecad/MyAddon/resources/translations/MyAddon.ts
```

Compile each locale:

```bash
uv run pyside6-lrelease \
  freecad/MyAddon/resources/translations/MyAddon_de.ts \
  -qm freecad/MyAddon/resources/translations/MyAddon_de.qm
```

If the code uses a custom translate alias (for example `tr` or a project-specific
wrapper), tell `lupdate`:

```bash
uv run pyside6-lupdate ... \
  -tr-function-alias translate+=tr \
  -ts freecad/MyAddon/resources/translations/MyAddon.ts
```

Qt 5 required separate `pylupdate` / `lconvert` steps. Do not use that workflow
for new addons; target Qt 6 / FreeCAD 1.1+.

### Translating a new locale

```bash
uv run pyside6-lupdate \
  freecad/MyAddon/**/*.py \
  freecad/MyAddon/resources/ui/*.ui \
  -source-language en \
  -target-language de \
  -ts freecad/MyAddon/resources/translations/MyAddon_de.ts
```

Then translate the `.ts` in Qt Linguist
(`uv run pyside6-linguist`) or a text editor, and compile with `lrelease`.

### Crowdin and contributors

FreeCAD hosts a Crowdin project for addons:
<https://crowdin.com/project/freecad-addons>. If the addon is listed there:

- upload the locale-agnostic `<AddonName>.ts`;
- pull translated locale files back into `resources/translations/`;
- compile `.qm` files as part of the release cycle;
- avoid accepting direct `.ts` pull requests that Crowdin will overwrite,
  unless the project has agreed otherwise.

For a small addon, local `.ts` files and pull requests are fine. Pick one
workflow and document it in `resources/translations/README.md`.

### Testing translations

1. Start FreeCAD in the target language.
2. Ensure `Gui.addLanguagePath()` points at the translations directory.
3. Evaluate a known string:

```python
FreeCAD.Qt.translate("MyAddon", "Do thing")
```

4. Check that it returns the translated text, not the English original.
5. Check command tooltips, workbench labels, preferences, and dialog text.

A typo in either the context or the source string silently falls back to the
original. Inspect the `.ts` file if a translation does not appear.

### Translation pitfalls

- Passing a variable to `translate()` translates at runtime but does not extract
  the string for translators. Use a literal in at least one place.
- `QT_TRANSLATE_NOOP` does not translate; it only marks. FreeCAD translates
  command resource strings later.
- `.ui` strings are extracted automatically; no `QT_TRANSLATE_NOOP` needed
  there.
- Some strings (workbench labels, menu names) may require a FreeCAD restart to
  change language. Test the restart path.
- Do not translate console output or internal identifiers.
- Do not ship only `.ts` files; runtime needs `.qm`.
- Do not compile resources into a `.rcc`; ship the `.qm` files as plain files.
- If the addon uses a custom translation wrapper (a project-specific `tr` or
  another function name), tell `lupdate` with `-tr-function-alias` or mark
  literals with `QT_TRANSLATE_NOOP`, otherwise the strings never reach the `.ts`
  file.
- A deferred translation wrapper must translate in `__str__` and keep a stable
  hash/comparison if it is used as a dict key.

## 4. Resource and i18n checklist

- [ ] Icons are SVG, under 16 kB for the manifest icon.
- [ ] `Gui.addIconPath()` registers the icon directory.
- [ ] No `.qrc`, `.rcc`, `pyside6-rcc`, or `:/...` paths in `.ui` files.
- [ ] QML is shipped as plain files under `resources/qml/`, not compiled.
- [ ] `qsTr()` wraps QML text; `.qml` files are included in `lupdate`.
- [ ] Live QML views call `engine().retranslate()` on a language change.
- [ ] Resources are under `freecad/<Mod>/resources/`.
- [ ] Runtime paths use `Path(__file__)`, not the current working directory.
- [ ] User data goes to `App.getUserAppDataDir()`, not the addon directory.
- [ ] `Gui.addLanguagePath()` and `Gui.updateLocale()` are called once in
      `init_gui.py`.
- [ ] `.ts` and `.qm` files are committed for every supported locale.
- [ ] Command/workbench/property strings use the right translation context.
- [ ] Console logs are not translated; user messages are.
- [ ] A translated locale was tested, including a restart.
