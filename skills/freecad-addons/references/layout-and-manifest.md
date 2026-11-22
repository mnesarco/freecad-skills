# Layout, manifest, packaging, and release

This reference covers the skeleton of a modern FreeCAD 1.1+ addon: where code
and resources live, how FreeCAD loads them, what `package.xml` must contain, and
how to release without breaking users.

## 1. Modern namespaced layout

```text
MyAddon/
├─ freecad/                       # namespace directory — NO __init__.py
│  └─ MyAddon/                    # the addon package — stable import name
│     ├─ __init__.py              # console-safe initializer
│     ├─ init_gui.py              # GUI entry point (optional only for console-only addons)
│     ├─ commands/                # command classes and registration
│     ├─ core/                    # pure logic; keep FreeCAD imports minimal
│     ├─ preferences.py
│     ├─ resources/
│     │  ├─ icons/*.svg
│     │  ├─ ui/*.ui
│     │  ├─ qml/*.qml               # QML root views and module directories
│     │  └─ translations/*.ts, *.qm
│     └─ _vendor/                 # only when vendoring is truly unavoidable
├─ package.xml
├─ pyproject.toml
├─ README.md
├─ CHANGELOG.md
├─ LICENSE-Code
├─ LICENSE-Assets
└─ .gitignore
```

Rules:

- `freecad/` is a **namespace directory**. It must not contain `__init__.py`.
  The FPA Addon-Reports checker reports this as a HIGH issue (`L002`).
- The repository root must not contain `__init__.py` (`L001`).
- Do not use `Init.py` or `InitGui.py` in a new addon. They are legacy layout.
- The addon package name (`MyAddon`) is permanent: it is part of every import
  and of the proxy module paths stored in `.FCStd` files. Choose it once.
- Keep code out of the repository root. Root-level Python files are legacy
  layout and collide with other addons.
- Resources belong inside the addon package. Root-level `Resources/` is a
  legacy pattern; it works, but modern addons keep resources under
  `freecad/<ModName>/resources/` so `__file__` paths are simple and packaging is
  self-contained.

## 2. Entry point lifecycle

### `freecad/<ModName>/__init__.py`

- Imported when the addon package is imported. It may be imported in console
  mode (`FreeCADCmd`), so it must not import `FreeCADGui`, `PySide`, or `pivy`,
  and must not create windows, dialogs, or dock widgets.
- Keep it small: version metadata, pure-Python setup, optional importers or
  handlers that are safe without a GUI.
- If the addon must do work at console startup (import/export handlers), put it
  here and verify with `FreeCADCmd`. Do not reintroduce a top-level `Init.py`.
- `__version__` is optional. If present, keep it in sync with `pyproject.toml`
  and `package.xml` via the build script; do not maintain three copies by hand.

### `freecad/<ModName>/init_gui.py`

- Loaded by FreeCAD at startup, for every user, even if they never activate the
  addon's workbench if any.
- Module-level code should only define the workbench class, register it, and
  register icon/language paths. No heavy imports, no geometry, no network.
- Do not create QML views here. The main window is not ready during `InitGui`;
  defer Qt Quick setup with `QTimer.singleShot(0, ...)` and import the QML
  helper module only then.
- `Initialize()` runs the first time the user activates the workbench. Import
  command modules there and build toolbars/menus there.
- `Activated()` / `Deactivated()` run every time the user switches into/out of
  the workbench. Keep them cheap.
- `GetClassName()` must return the literal string `"Gui::PythonWorkbench"`.
- `Icon` must be an absolute path set as a class attribute, before
  `FreeCADGui.addWorkbench()`.

```python
class MyWorkbench(Gui.Workbench):
    MenuText = "My Addon"
    ToolTip = "Tools for ..."
    Icon = _ICON

    def Initialize(self):
        from . import commands  # noqa: F401
        self.appendToolbar("MyAddon", ["MyAddon_DoThing"])
        self.appendMenu("MyAddon", ["MyAddon_DoThing"])

    def GetClassName(self):
        return "Gui::PythonWorkbench"
```

### Non-workbench addons

An addon that only adds commands to existing menus/toolbars still has an
`init_gui.py`; it registers commands and a `WorkbenchManipulator` instead of a
workbench. Its manifest uses `<content><other/></content>` on FreeCAD 1.1+.
See `references/ui-and-commands.md`.

## 3. Import boundaries

- Use relative imports inside the package (`from .core import ...`).
- Never manipulate `sys.path`. The namespace package exists to avoid that.
- `core/` should contain pure logic where practical: parsers, calculations,
  validation, path handling. This code can be tested outside FreeCAD.
- Import heavy dependencies (`numpy`, `scipy`, `Part`, `pivy`, big UI modules)
  inside the function that needs them, not at module scope.
- `TYPE_CHECKING` imports are for type checkers only; they must not create a
  runtime dependency:

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    import FreeCAD as App
```

- Do not import `FreeCADGui` from `__init__.py`, `core/`, or any module that
  console mode may import.

## 4. `package.xml`

The manifest is read by the Addon Manager and by the FPA checker. It must be at
the repository root, be well-formed XML, use `/` in paths even on Windows, and
declare only known tags.

### Complete workbench template

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<package format="1" xmlns="https://wiki.freecad.org/Package_Metadata">
  <name>MyAddon</name>
  <description>What the addon does, in one or two sentences.</description>
  <version>0.1.0</version>
  <date>2026-11-15</date>

  <maintainer email="you@example.org">Your Name</maintainer>
  <author email="you@example.org">Your Name</author>

  <license file="LICENSE-Code">LGPL-3.0-or-later</license>
  <license file="LICENSE-Assets">CC-BY-SA-4.0</license>

  <icon>freecad/MyAddon/resources/icons/logo.svg</icon>
  <freecadmin>1.1.0</freecadmin>
  <pythonmin>3.11</pythonmin>

  <url type="repository" branch="main">https://github.com/you/MyAddon</url>
  <url type="readme">https://raw.githubusercontent.com/you/MyAddon/main/README.md</url>
  <url type="bugtracker">https://github.com/you/MyAddon/issues</url>
  <url type="documentation">https://github.com/you/MyAddon/wiki</url>

  <content>
    <workbench>
      <classname>MyWorkbench</classname>
    </workbench>
  </content>

  <tag>myaddon</tag>
</package>
```

### Field rules

| Element | Rule |
|---|---|
| `package format` | Must be `format="1"` with the `https://wiki.freecad.org/Package_Metadata` namespace. |
| `name` | Canonical addon name. Used by `depend type="addon"`. Must be filename-safe. |
| `description` | Plain text, concise. The Addon Manager truncates long text. |
| `version` | SemVer (`1.2.3`, `1.2.3-beta`) or CalVer (`2026.11.15`). Never mix styles. Bump on every tracked-branch release. |
| `date` | `YYYY-MM-DD`, the release date. |
| `maintainer` | At least one, with a working email. Orphaned addons use `no-one@freecad.org`. |
| `author` | Optional but recommended; use a real contact where possible. |
| `license` | Exact SPDX identifier, one element per license. `file` is relative to `package.xml` and must exist. Every license used in code/assets needs an element. Keep the manifest, license files, and any per-file SPDX headers identical. `LGPL2` is normalized to `LGPL-2.0`; write `LGPL-2.1-or-later`. |
| `icon` | SVG, relative to `package.xml`; keep it under 16 kB. Missing icon or a bitmap icon is a LOW issue in Addon-Reports. |
| `freecadmin` | Minimum FreeCAD version as `MAJOR.MINOR.PATCH`. Set it honestly. |
| `freecadmax` | Optional exact maximum. To cover a whole series, use e.g. `1.1.99`. |
| `pythonmin` | Minimum Python as `MAJOR.MINOR`. Must match `requires-python`. |
| `url type="repository"` | Required; `branch` must match the branch the manifest lives on. Mismatch is a HIGH issue. |
| `url type="readme"` | Highly recommended; use a raw Markdown URL, not the GitHub HTML page. |
| `url type="bugtracker"` | Recommended. |
| `url type="documentation"` | Recommended. |
| `content` | Required by the schema. Use `<workbench>` for a real workbench; `<other/>` for toolbar/menu-only addons on 1.1+; `<macro>` for macros; `<preferencepack>` for themes/packs. |
| `classname` | Required for workbench content. Must exactly match the Python workbench class name in `init_gui.py`. |
| `subdirectory` | Omit in a modern layout. It exists for legacy packages and multi-content packages. |
| `tag` | Searchable keywords; useful for discovery. |
| `depend` | Declare `type="python"`, `type="addon"`, or `type="internal"`. Do not rely on `automatic`. |
| `conflict` / `replace` | Use only when genuinely needed; be explicit about versions. |

### Dependency rules

```xml
<depend type="python">numpy</depend>
<depend type="addon">Curves</depend>
<depend type="addon" version_gte="0.6">Curves</depend>
<depend type="internal">part</depend>
<depend type="python" optional="true">matplotlib</depend>
```

- Python dependencies must be functionally necessary and on the Addon Index
  allow-list. No version specifiers for Python deps in `package.xml`; the Addon
  Manager does not install versions for them. Current list:
  `FreeCAD/Addons` → `Data/Python/<python-version>/Allowed-Packages` (legacy:
  `FreeCAD/FreeCAD-addons` → `ALLOWED_PYTHON_PACKAGES.txt`). Request additions
  with the `Package - Addition` issue template; do not work around the list.
- Do not install packages at runtime. The Addon Manager installs allow-listed
  dependencies; a separate user-installed program may manage its own environment.
- `type="addon"` names the canonical `<name>` of the other addon, not its repo.
- `type="internal"` uses the lower-case internal workbench name (`part`,
  `sketch`, `mesh`, `import`, `draft`, ...).
- `optional="true"` only when the addon genuinely works without it.
- Avoid vendoring. The Addon Index strongly discourages it and treats it as
  grounds for rejection. Never vendor an allow-listed package or one with
  compiled extensions. If technically unavoidable, isolate it under
  `freecad/<ModName>/_vendor/`, preserve the upstream `LICENSE`, add a
  `_vendor/README.md` with source/version/modifications, never modify it without
  recording why, and import it through your namespace
  (`from freecad.MyAddon._vendor.somepkg import thing`).

### License files and SPDX headers

- Ship the full license text in the files referenced by `<license file="...">`.
  Most licenses (LGPL, GPL, MIT, BSD, Apache, CC-BY-SA) require it.
- Recommended defaults: `LGPL-2.1-or-later` for code, `CC-BY-SA-4.0` for
  non-code assets, in separate `LICENSE-Code` and `LICENSE-Assets` files.
- Add an SPDX header to source files if the project uses them, and keep it
  identical to the manifest identifier:
  `# SPDX-License-Identifier: LGPL-2.1-or-later`.
- Keep `pyproject.toml` and README license statements consistent with
  `package.xml`.
- Vendored code keeps its upstream license and attribution.

## 5. `pyproject.toml`

Use standard Python metadata for development tooling. FreeCAD does not install
the addon from PyPI; this file is for `uv`, linters, type checkers, and build
scripts.

```toml
[project]
name = "MyAddon"
version = "0.1.0"
description = "What the addon does"
readme = "README.md"
requires-python = ">=3.11"
license = "LGPL-3.0-or-later"

maintainers = [
    { name = "Your Name", email = "you@example.org" },
]
authors = [
    { name = "Your Name", email = "you@example.org" },
]
keywords = ["freecad", "addon", "myaddon"]

[project.urls]
repository = "https://github.com/you/MyAddon"
documentation = "https://github.com/you/MyAddon/wiki"
readme = "https://raw.githubusercontent.com/you/MyAddon/main/README.md"

[dependency-groups]
dev = [
    "freecad-stubs>=1.1",
    "PySide6",
    "ruff",
    "bandit[toml]",
    "codespell",
    "pytest",
]

[tool.ruff]
line-length = 100
indent-width = 4

[tool.ruff.lint]
select = ["ALL"]
ignore = [
    "D100", "D102", "D103", "D104", "D105",
    "D200", "D212", "D400", "D401", "D415",
    "ANN002", "ANN003", "ANN401",
    "FBT001", "FBT002", "FBT003",
    "N802", "N806", "N813",
    "SIM108", "SIM117", "TID252",
]
# Tune the ignore list to the project; do not disable whole rule families.

[tool.bandit]
exclude_dirs = [".venv", "freecad/MyAddon/_vendor"]

[tool.codespell]
skip = "*/translations/*,*.svg,uv.lock"
```

Rules:

- Do **not** list FreeCAD, `FreeCADGui`, `Part`, or `pivy` as dependencies; they
  are provided by the FreeCAD runtime.
- `PySide6` is a dev dependency for Qt tooling (`pyside6-lupdate`,
  `pyside6-lrelease`, Qt Designer) and type checking. Runtime code imports
  `PySide`, not `PySide6`; the one exception is `QtQml`/`QtQuick`/
  `QtQuickWidgets`, imported lazily from `PySide6` because the shim does not
  expose them. Do not add `PySide6` as a runtime dependency.
- `freecad-stubs` is for editors/type checkers only. Do not import it at runtime.
  Configure the type checker to ignore missing imports for FreeCAD modules that
  are not present in the development environment.
- Keep `requires-python` equal to `pythonmin`.
- `uv.lock` may be committed for reproducible development; exclude it from the
  addon release zip.
- A single version source of truth (`[project] version`) is strongly preferred.
  A build script can sync `package.xml`, `__version__`, and the release date.

### Optional `[tool.freecad]` conventions

These keys are not read by FreeCAD itself. They are useful if the project has a
build script (for example an `fcbuild.py`-style wrapper) that generates
`package.xml`, translations, and the release zip:

```toml
[tool.freecad]
branch = "main"
icon = "freecad/MyAddon/resources/icons/logo.svg"
freecad-min = "1.1"
freecad-max = "1.1.99"
lupdate-files = []
lupdate-langs = ["en", "de", "es", "fr"]
addon-dependencies = ["Curves >=0.6"]
internal-dependencies = ["part"]
```

If there is no build script, omit the section; it has no runtime effect.

## 6. Versioning, branches, and releases

Recommended structure:

- `main` (or `master`) is the stable branch tracked by the Addon Index.
- `develop` is the day-to-day branch; merge to `main` only for a release.
- Bump `version` and `date` on `main` for every release.
- Tag the release (`v0.1.0`) and write release notes in `CHANGELOG.md`.
- Keep the `branch` attribute in `package.xml` equal to the branch that contains
  that file. If you copy the manifest to another branch, update the branch.
- Do not point the index at a development branch; Addon-Reports warns about it.

### Build a zip with standard tools

`git archive` respects `.gitattributes` and includes only tracked files:

```bash
mkdir -p dist
git archive --format=zip --prefix=MyAddon/ HEAD \
  -o "dist/MyAddon-$(git describe --tags --always).zip"
```

Add export rules so development files stay out of the archive:

```gitattributes
/.github        export-ignore
/tests          export-ignore
/dist           export-ignore
/uv.lock        export-ignore
/pyproject.toml export-ignore
/fcbuild.py     export-ignore
```

The zip root must contain `package.xml` and `freecad/`, plus `README.md` and the
license files. Do not ship `.git`, `.venv`, `dist/`, `__pycache__`, `*.pyc`,
test data, CI configuration, or development scripts.

If the project uses an `fcbuild.py`-style script, it may also sync versions,
regenerate `.ts`/`.qm`, and update `package.xml`. That is optional; the same
result can be achieved with standard tools.

### Release smoke test

1. Extract the zip into a clean FreeCAD user `Mod/` directory.
2. Start FreeCAD in GUI mode; check the workbench icon and menu.
3. Start `FreeCADCmd`; check the addon package imports without a GUI.
4. Activate the workbench, run each command, and undo each change.
5. Open Preferences and change a value; restart and verify persistence.
6. Switch languages and check translated UI strings.
7. Open a document created with the previous version and verify custom objects
   still load and recompute.
8. If the addon uses an external service, decline consent and confirm no request
   is made; accept and confirm the disclosed data matches what is sent. For AI
   code execution, approve a sample run and verify one undo reverts it.
9. Test on the newest FreeCAD version you claim, in a clean user directory.

## 7. README and repository hygiene

`README.md` should contain:

- one-paragraph purpose;
- installation via Addon Manager;
- the main commands/workflow;
- requirements and compatibility (`freecadmin`, Python);
- screenshots or a short GIF (stored under resources/media, not root clutter);
- license summary and links;
- known limitations and which features do/do not work;
- maintainer/contact and a link to `SECURITY.md`;
- `## Privacy and external services` when any network, telemetry, AI service, or
  listener exists: service, endpoint, data sent, trigger, frequency, retention,
  consent, and opt-out.

`CHANGELOG.md` should list user-visible changes per version.

`.gitignore` at minimum:

```gitignore
__pycache__/
*.py[cod]
.venv/
dist/
build/
*.egg-info/
.pytest_cache/
.ruff_cache/
```

Do not commit dependency trees, virtual environments, compiled binaries, large
media, or generated resources. Addon-Reports flags committed dependency trees
and binary artifacts.

## 8. Legacy migration

Map legacy files to modern ones:

| Legacy | Modern |
|---|---|
| `Init.py` | `freecad/<Mod>/__init__.py` |
| `InitGui.py` | `freecad/<Mod>/init_gui.py` |
| `manifest.ini` | `package.xml` |
| `setup.py` | `pyproject.toml` |
| root `__init__.py` | remove; use package `__init__.py` |
| `freecad/__init__.py` | remove; `freecad/` is a namespace |
| `sys.path.append(...)` | relative imports |
| `.qrc` / `.rcc` | plain SVG + `addIconPath()`; plain QML under `resources/qml/` |
| `PySide2` / `PySide6` runtime imports | `from PySide import ...`; `PySide6` only for `QtQml`/`QtQuick`/`QtQuickWidgets`, lazily |
| vendored library | declared dependency or isolated `_vendor/` |

If the addon has a legacy `manifest.ini`, map it rather than copying it:

| `manifest.ini` | `package.xml` |
|---|---|
| `[general] name` / `title` | `<name>` |
| `[general] description` | `<description>` |
| `[general] version` | `<version>` |
| `[general] iconPath` | `<icon>` |
| `[general] license` / `licensePath` | `<license file="...">SPDX</license>` |
| `[general] authors` | `<maintainer>` and `<author>` |
| `[general] categories` | `<tag>` |
| `[dependencies] pylibs` | `<depend type="python">` |
| `[dependencies] workbenches` | `<depend type="addon">` or `<depend type="internal">` |
| `[git] repository` | `<url type="repository" branch="...">` |
| `[install] notesPath` | `<url type="readme">` |
| `[install] mode` / `checkUpdates` | no direct equivalent; use the branch/release workflow |

`manifest.ini` is legacy metadata. Do not keep it in a new addon, and remove it
after a successful migration; do not maintain both files in parallel.

Migration order:

1. Create `freecad/<Mod>/` and move modules inside, converting imports to
   relative imports.
2. Rename `Init.py` to `freecad/<Mod>/__init__.py`; strip GUI code.
3. Rename `InitGui.py` to `freecad/<Mod>/init_gui.py`; keep only registration
   and workbench definition at module scope.
4. Remove root `__init__.py`, `freecad/__init__.py`, `sys.path` manipulation,
   `manifest.ini`, and `setup.py`.
5. Add `package.xml`, `pyproject.toml`, `README.md`, licenses, and `.gitignore`.
6. Replace `.qrc`/`.rcc` with plain SVG and `addIconPath()`.
7. Test in `FreeCADCmd`, then in GUI, then with an old `.FCStd`.

**Migration risk — saved documents.** FreeCAD locates a FeaturePython proxy by
its saved module and class name. If the old path was `MyModule.Box` and the new
path is `freecad.myaddon.box.Box`, old documents cannot bind the proxy. Options:

- keep the old module path as a compatibility shim that re-exports the class;
- provide a migration script that rewrites the proxy path in saved documents;
- accept a major-version break and document it clearly.

A top-level shim only works if the old module is still importable. In the modern
layout the repository root is not added to `sys.path`, so a root-level shim may
not help. Test by opening a real old document — not by assuming.

## 9. Layout checklist

- [ ] `freecad/` exists and has no `__init__.py`.
- [ ] Addon package has `__init__.py`; GUI addon has `init_gui.py`.
- [ ] No root `__init__.py`, `Init.py`, `InitGui.py`, `manifest.ini`, or
      `setup.py`.
- [ ] No `sys.path` manipulation.
- [ ] `package.xml` is at the root and schema-valid.
- [ ] `package.xml` branch matches the branch.
- [ ] `version`/`date` are current.
- [ ] `classname` matches the Python workbench class.
- [ ] Icon is SVG and under 16 kB.
- [ ] License files exist and match SPDX identifiers.
- [ ] `pyproject.toml` has dev tooling and no FreeCAD/PySide runtime deps.
- [ ] Release zip contains `package.xml` + `freecad/` only.
