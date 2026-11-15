# Quality, security, testing, and review checklist

This reference maps the rules in the skill to the FreeCAD addon ecosystem's
automated checks and gives a release/review workflow.

## 1. What the FPA Addon-Reports checker looks at

The FreeCAD Addon-Reports project is a static analyzer for indexed addons. It
clones each addon, runs checks, and produces a ranked report. If the project or
its output is available, use it before release.

Typical commands (from the Addon-Reports project):

```bash
uv run analyze            # clone and analyze all indexed addons
uv run analyze --skip-clone
```

### Layout checks

| Rule | Severity | Meaning |
|---|---|---|
| `L001` | HIGH | Root `__init__.py`. Legacy exec-based layout; remove it. |
| `L002` | HIGH | `freecad/__init__.py`. Invalid; `freecad/` is a namespace directory. |
| — | INFO | `Init.py` / `InitGui.py` detected: legacy layout. |
| — | INFO | `freecad/` directory detected: extension-based layout. |
| — | INFO | No init scripts found. |

### Package/manifest checks

| Check | Severity | Meaning |
|---|---|---|
| RelaxNG schema | HIGH | `package.xml` must validate. Unknown/misordered tags fail. |
| Repository URL | HIGH | `<url type="repository">` is required. |
| Repository branch | HIGH | `branch` attribute is required on the repository URL. |
| Branch mismatch | HIGH | Declared branch must match the checked-out git branch. |
| License file | HIGH | Every `<license file="...">` must exist. |
| Version | HIGH | `<version>` is required. |
| Description | HIGH | `<description>` is required. |
| Maintainer/author | INFO | Missing people data is flagged; the curation policy treats a missing maintainer as a hard failure. |
| Icon | LOW | Missing icon; icon over 16 kB; non-SVG icon. |

### Other analyzer checks

- **Bandit** security scan.
- **Dependencies** extracted from Python sources (pipreqs).
- **GitHub stats**: stars, forks, subscribers, open issues.
- **Matomo stats**: Addon Manager download counts.
- **Custom rules** (for example, an exception for the Addon Manager itself).

Scoring starts at 100; HIGH loses 3, MEDIUM loses 1, LOW loses 0.1, clamped at 0.
Addons are sorted by score, then 30-day downloads, then name.

## 2. Addon Index qualities

Admission to the official index requires more than passing the static checks:

- an active maintainer;
- timely issue/PR/security handling;
- a repository and codebase at least 30 days old;
- GDPR compliance and clear disclosure of external connections;
- consistent licensing (manifest, LICENSE files, SPDX headers);
- Python 3 code that uses FreeCAD's Qt wrappers;
- a valid manifest with the correct content type, workbench classname, and
  repository branch;
- commands/toolbars/menus confined to the addon's own workbench (or intentional
  manipulator contributions);
- no expensive work, network access, or global side effects at import;
- compatibility with the latest FreeCAD, or an honest `freecadmin`/`freecadmax`;
- only necessary dependencies, on the allow-list;
- vendoring only when technically necessary (strongly discouraged);
- a substantially complete addon, not alpha-level scaffolding;
- Bandit issues addressed.

Curation (`curated: true`) is a stronger, human-reviewed subset. It is not a
substitute for admission requirements.

## 3. Security checklist

### Code execution

- Do not use `eval()` or `exec()` on untrusted input.
- If the addon intentionally runs user/model-generated code, show the complete
  code, require explicit approval, and run it in one undo transaction. Do not
  auto-approve; do not hide the code.
- Do not add an "auto-approve" setting that is on by default.
- If the addon is an AI/MCP bridge, keep the agent's approval prompt intact,
  bind to loopback by default, require authentication, and warn inside FreeCAD
  before starting a listener. Follow the Addon Index AI requirements.

### Input and files

- Use `defusedxml` for untrusted XML/SVG instead of the stdlib parser.
- Validate paths read from documents, preferences, or user input.
- Prevent path traversal and zip-slip when extracting archives.
- Do not unpickle untrusted data.
- Do not write secrets/tokens into the addon directory or world-readable temp
  files.
- Use the OS keyring for credentials where available.
- Do not commit secrets, API keys, or user data.

### Processes and network

- Use `subprocess` with an argument list; no `shell=True` with user data.
- Add `# nosec` only with a specific justification.
- Do not make network calls at import or startup. Do it on explicit user action.
- Document what data is sent, to whom, how often, and why. Ask for consent where
  required (GDPR).
- Use TLS and verify certificates; do not disable verification to "make it work".
- Do not silently install Python packages into FreeCAD's environment.

### Logging and errors

- Do not log secrets, tokens, or full user paths unnecessarily.
- Do not silently swallow exceptions.
- Show actionable user errors in a dialog/status bar; keep tracebacks in the
  Report view.
- Avoid `print()` in production.

### Bandit

Run Bandit over the addon sources:

```bash
uv run bandit -r freecad/
```

Fix real issues. Suppress a finding only with a narrow `# nosec` and a comment
explaining why it is safe. Do not exclude the whole addon.

## 4. Testing and CI

### Pure Python tests

Keep parsers, calculations, validation, path handling, and migration decisions
in modules that do not import FreeCAD at module scope. Test them with `pytest`:

```bash
uv run pytest
```

### In-FreeCAD tests

Use `unittest` and register modules with `FreeCAD.__unit_test__` for console
tests, or `FreeCADGui.__unit_test__` for GUI/Qt tests:

```python
# freecad/MyAddon/__init__.py (console tests only)
import FreeCAD

FreeCAD.__unit_test__ += ["freecad.MyAddon.tests.test_geometry"]
```

```python
# freecad/MyAddon/init_gui.py (GUI tests only)
import FreeCADGui

FreeCADGui.__unit_test__ += ["freecad.MyAddon.tests.test_dialog"]
```

Run:

```bash
freecadcmd --run-test 0
freecadcmd --run-test freecad.MyAddon.tests.test_geometry
FreeCAD --run-test freecad.MyAddon.tests.test_dialog
```

Test geometry with tolerances:

```python
self.assertAlmostEqual(shape.BoundBox.XLength, 900.0, delta=0.1)
self.assertAlmostEqual(shape.Volume, 75_600_000.0, delta=1.0)
```

### Migration tests

Keep a fixture of `.FCStd` files from older versions. Open each with the current
code and check proxy binding, property types/values, and recompute.

### API and lifecycle tests

Add focused tests for the parts that fail silently:

- Property creation: type, group, default, enumeration options, property modes.
- Extension setup: Attach/Link/Group configuration and `configLinkProperty`.
- Command registration: prefix, duplicate detection, checkable/grouped state.
- Transactions: commit on success, abort on exception, intentional cancel.
- Observers: callback fires, same instance removes, no callback after teardown.
- Selection validation: wrong type/order rejected; sub-elements resolved.
- Preferences: defaults, validation before save, presets, `Default` protection.
- Display modes: every name registered; default is in the list.
- Task panel: open/accept/reject closes the dialog and resets edit mode.
- Serialization: save/reopen retains the expected state.

### CI steps

A minimal CI pipeline:

```bash
uv sync
uv run ruff check .
uv run ruff format --check .
uv run bandit -r freecad/
uv run codespell
uv run pytest
# optional: validate package.xml, build zip, run freecadcmd tests
```

If the addon declares a range of FreeCAD versions, test the oldest and newest
supported versions. Do not claim compatibility you have not tested.

### What to test manually

- Fresh install from the zip/branch in a clean FreeCAD user directory.
- GUI startup and console startup.
- Workbench icon, menu, toolbar, command availability.
- Undo/redo for every document-changing command.
- Preferences persistence across restart; live-update preferences take effect.
- Language switch and translation restart.
- Opening an old document with custom objects.
- Display modes switch correctly; context menu actions work.
- Edit/task panel opens, accepts, rejects, and resets edit mode.
- Drag/drop updates the group and is undoable.
- Observers stop firing after their panel/tool is closed.
- Progress indicator stops on success and on error.
- Addon removal/reinstall.
- Large document/selection performance.

## 5. Anti-slop and completeness

The ecosystem explicitly rejects low-effort or AI-generated slop. Before
calling an addon done:

- Remove placeholder `pass`, `TODO`, `FIXME`, `...`, and "not implemented"
  paths that are user-visible.
- Every command in a menu/toolbar actually works.
- README explains installation, usage, requirements, and limitations.
- A real maintainer and contact are present.
- License files and SPDX headers are consistent.
- Tests cover non-trivial logic.
- A release tag exists for the tracked version.
- No committed `.venv`, dependency trees, `dist/`, `__pycache__`, or binary
  artifacts.
- No generated "AI marker" phrases in source/docs.
- No alpha-level scaffolding in the stable branch.

## 6. Release checklist

### Before tagging

- [ ] Version bumped in `pyproject.toml` and `package.xml`; date updated.
- [ ] Branch in `package.xml` matches the release branch.
- [ ] `package.xml` schema-valid; repository/readme URLs correct.
- [ ] `classname` matches the workbench class.
- [ ] Icon is SVG and under 16 kB.
- [ ] License files exist; SPDX identifiers match.
- [ ] `README.md` and `CHANGELOG.md` updated.
- [ ] `ruff`, `bandit`, `codespell`, and tests pass.
- [ ] Addon-Reports/curation issues reviewed.
- [ ] Translations updated and `.qm` compiled for supported locales.
- [ ] Old `.FCStd` migration tested.
- [ ] Clean install smoke test in GUI and `FreeCADCmd`.

### Package

- [ ] Zip root contains `package.xml` and `freecad/`.
- [ ] No `.git`, `.venv`, `dist/`, `__pycache__`, tests, CI, or dev scripts.
- [ ] `README.md` and license files included.
- [ ] Zip extracted into a clean `Mod/` directory works.

### After release

- [ ] Tag pushed (`vX.Y.Z`).
- [ ] Release notes published.
- [ ] Addon Index branch/version updated if needed.
- [ ] Post-release issue watch and crash reports.

## 7. Review checklist

Use this when reviewing an addon. Report only violations, with file/line and the
rule. Rank by impact.

### Structure and metadata

- [ ] Modern `freecad/<Mod>/` layout; no root `__init__.py`.
- [ ] No `freecad/__init__.py`.
- [ ] No `Init.py` / `InitGui.py` in a new addon.
- [ ] No `manifest.ini`, `setup.py`, or `sys.path` manipulation.
- [ ] `package.xml` schema-valid, branch matches, version/date current.
- [ ] Maintainer, author, licenses, icon present and valid.
- [ ] `pyproject.toml` has dev tooling; no FreeCAD/PySide runtime deps.
- [ ] README/CHANGELOG useful; no placeholder text.

### Startup and imports

- [ ] `__init__.py` is console-safe; no `FreeCADGui`/Qt/`pivy`.
- [ ] `init_gui.py` has minimal module-level work.
- [ ] Heavy imports are deferred to `Initialize()`/`Activated()`.
- [ ] No network, subprocess, thread, dialog, or global UI work at import.
- [ ] No cleanup left to `__del__`.

### Commands and UI

- [ ] Command names prefixed; no core collisions.
- [ ] `GetResources()` has correct labels/tooltips/icons.
- [ ] Toggle/grouped/dropdown commands declare `Checkable`/`Exclusive`/
      `DropDownMenu` correctly and register sub-commands.
- [ ] `CmdType` declared honestly.
- [ ] `IsActive()` is fast and correct.
- [ ] Selection is validated by type/order; sub-elements resolved only when
      needed; selection restored when changed.
- [ ] Document changes are in transactions; errors abort.
- [ ] Long operations use a progress indicator stopped in `finally`.
- [ ] No `processEvents()` in recompute/transactions.
- [ ] Widgets are parented to the main window.
- [ ] No hardcoded colors/fonts; SVG icons.
- [ ] Runtime imports use `PySide`, not `PySide2`/`PySide6`.
- [ ] Workbench/manipulator changes are scoped and prefixed.
- [ ] Preferences use `Gui::Pref*` or a programmatic page plus `ParamGet`, with
      defaults; presets and validation are correct; live-update managers detach.
- [ ] Logs are developer-facing; user messages are translated dialogs/status.
- [ ] Task panels close cleanly and call `Gui.ActiveDocument.resetEdit()`.

### Document objects and Coin

- [ ] Proxy class has a stable module path; no inline class.
- [ ] `execute()` is idempotent, side-effect-free, GUI-free.
- [ ] Properties are the schema; migrations exist for changes.
- [ ] Property modes (`Transient`, `Hidden`, `Output`, `NoRecompute`,
      `NoPersist`) and editor/status flags are deliberate.
- [ ] `PropertyFileIncluded` vs external file/path properties are used
      correctly.
- [ ] Attach/Link/Group extensions are declared and configured
      (`positionBySupport`, `configLinkProperty`).
- [ ] PartDesign features respect `Placement`, `BaseFeature`, `AddSubShape`.
- [ ] `onDocumentRestored()` is idempotent and tested on old files.
- [ ] One serialization convention (`dumps`/`loads` or
      `__getstate__`/`__setstate__`), with small versioned state.
- [ ] ViewProvider is separate and GUI-only.
- [ ] Coin scene graph built in `attach()`, updated in `updateData()`.
- [ ] Display modes are all registered; default mode is valid.
- [ ] Context-menu actions are translated and scoped; edit modes close cleanly.
- [ ] Drag/drop updates the group/children inside a transaction.
- [ ] `SoSeparator` roots; state isolated; units Z-up/mm.
- [ ] Batched updates; no full rebuild per property.
- [ ] No Coin/Qt from `execute()` or worker threads.
- [ ] Observers/draggers cleaned up explicitly; same instance removed.
- [ ] Cached view/camera/dragger/widget checked with `shiboken.isValid`.
- [ ] Camera values copied before mutation; camera reacquired on view change.

### Quality and security

- [ ] Tests cover non-trivial logic and migrations.
- [ ] No `eval`/`exec` on untrusted input; AI code execution has consent and a
      transaction.
- [ ] XML/SVG parsing is hardened where input is untrusted.
- [ ] subprocess uses argument lists; no shell injection.
- [ ] Network access is explicit, documented, consented.
- [ ] No secrets/credentials in the repo.
- [ ] No committed dependency trees/binaries.
- [ ] Release zip clean; clean-install smoke test done.

## 8. Patterns to reject when reviewing older addons

Older addons in this ecosystem may still contain these. Treat them as debt, not
as precedent:

| Pattern | Why it is rejected | Modern replacement |
|---|---|---|
| `Init.py` / `InitGui.py` at the repo root | Legacy layout; global namespace pollution | `freecad/<Mod>/__init__.py` + `init_gui.py` |
| Root `__init__.py` or `freecad/__init__.py` | Breaks the namespace and the FPA layout checks | Remove both |
| `manifest.ini` | Legacy metadata, superseded by `package.xml`; do not add it to a modern addon | `package.xml` |
| `setup.py` | Legacy packaging | `pyproject.toml` + standard tools |
| `sys.path.append(...)` | Namespace collisions, install-order bugs | Relative imports in `freecad/<Mod>/` |
| `.qrc` / `.rcc` / `pyside6-rcc` | Tied to the Qt major version | Plain SVG + `Gui.addIconPath()` |
| Heavy imports or heavyweight setup at module scope | Slows every FreeCAD startup | Defer to `Initialize()` / `Activated()`; keep `addCommand` registration lightweight |
| `QApplication.processEvents()` in recompute/transaction | Re-entrancy, invalidated objects, crashes | Task panel / chunked work / progress dialog |
| `__del__` for observers or critical cleanup | Not guaranteed to run; late shutdown crashes | Explicit `removeObserver` / lifecycle methods |
| Polling documents/selection on a timer | Slow, misses events, burns CPU | Document/selection observers; debounce only high-frequency events |
| Registering a new observer per command | Leaks callbacks and keeps old widgets alive | One observer per owner, removed on teardown |
| Using a stale C++ widget/view/camera wrapper | Crashes instead of a Python exception | `shiboken.isValid()` and reacquire |
| Leaving a task panel open / not resetting edit mode | Stuck edit state, disabled commands | `Gui.Control.closeDialog()` + `Gui.ActiveDocument.resetEdit()` |
| Drag/drop without a transaction | Non-undoable or partial document changes | Transaction around `dropObject`/group changes |
| Saving preferences before validating | Partial/invalid configuration | Validate all, then write all |
| Display mode not registered/default mismatch | Blank view or wrong mode | `addDisplayMode` for every mode; default in the list |
| Group/dropdown command without sub-commands | Dead toolbar/menu entry | Register sub-commands and `GetCommands`/`GetDefaultCommand` |
| Runtime `PySide2` / `PySide6` imports | Breaks across FreeCAD Qt builds | `from PySide import ...` |
| Hardcoded colors/fonts/pixel sizes | Breaks dark mode, themes, HiDPI | Palette, layouts, SVG icons |
| Duplicated version strings | Drift between manifest, package, and UI | One source of truth + build sync |
| Vendored helper library | Index policy discourages it; stale/security risk | Declared dependency or isolated `vendor/` with a strong reason |
| `exec`/`eval` of user code without approval | Arbitrary code execution | Show code, require consent, run in one transaction |
| Alpha scaffolding in the release branch | Index rejects incomplete addons | Finish features before release |

## 9. Severity guide

When reporting review findings:

- **HIGH** — install/startup broken; data loss; security issue; saved documents
  fail to load; manifest invalid or branch mismatch; root/freecad `__init__.py`.
- **MEDIUM** — command doesn't work; undo broken; heavy startup cost; UI blocks
  or crashes; broken translations; missing migration; Coin accessed from
  `execute()`.
- **LOW** — icon size/format; missing docs; style/consistency; performance
  polish.

A review is not a rewrite request. Fix the smallest complete set of issues that
makes the addon correct, safe, and maintainable.
