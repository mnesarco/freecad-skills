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

## 2. Addon Index admission

Passing the checker is not sufficient: the official Index is curated and also
enforces governance, GDPR/consent, licensing, codebase, and AI-agent
requirements. The full admission checklist, implementation patterns, evidence,
and common rejection causes are in `references/addon-index-qualities.md`.

Admission-critical rules:

- an active maintainer, timely issue/PR/security handling, and a repository at
  least 30 days old;
- in-FreeCAD consent before any external data transfer, per service and data
  kind, re-asked when either changes; no default service and no ambient
  credentials before consent; secure local storage;
- exact SPDX identifiers consistent across manifest, license files, and per-file
  headers, with full license texts;
- a valid manifest with the correct content type, matching workbench
  `classname`, and matching repository branch; latest FreeCAD compatibility or an
  honest `freecadmin`/`freecadmax`;
- functionally necessary dependencies on the allow-list; no runtime installs;
  vendoring only when the allow-list path is impossible;
- commands/menus confined to the addon's own workbench or a scoped manipulator;
  no expensive work, network, or global side effects at import/startup;
- a substantially complete release branch and addressed Bandit findings;
- AI addons: exact approved code execution, auto-approval off by default with an
  in-UI warning, bridge approval prompt intact, loopback/authenticated/verified
  listeners, private agent files, and one undo transaction per approved run.

Curation (`curated: true`) is a stronger, human-reviewed subset. It is not a
substitute for admission requirements.

## 3. Security checklist

### Code execution and AI addons

- Do not use `eval()` or `exec()` on untrusted input.
- If the addon intentionally runs user/model-generated code, show the complete
  code inside FreeCAD when the addon runs the agent, require explicit approval,
  and run only the exact code shown. For a file, run the contents that were
  displayed, not a fresh read. File reads/writes at paths the model chooses also
  need approval; fixed document operations with model-supplied values do not.
- Wrap each approved run in one undo transaction.
- Automatic approval is opt-in: it may only be a setting, off by default, with a
  warning in the interface itself. An addon that launches or configures an agent
  must not enable auto-approval for code or file tools. A bridge must keep the
  agent's approval prompt intact and warn inside FreeCAD when it starts.
- Before any remote AI request, including one that only lists models, ask for
  consent inside FreeCAD. Name the service and every kind of data sent,
  including automatic data. Consent is per service and is asked again when the
  service or data kinds change. No default service; do not use ambient API keys
  or logins before consent.
- Network listeners start only if the user enabled them, require authentication
  on every request with no off switch, bind loopback by default, reject browser
  requests (HTTP `Origin` header, or mismatched `Host`), and close on
  out-of-protocol input. Remote access needs an explicit user-chosen address,
  an in-FreeCAD warning and acceptance per address, and verified TLS (pinned
  certificate or TLS-PSK; unchecked self-signed certificates do not count).
- Agent tokens, pairing secrets, configuration, instructions, working
  directories, and screenshots are private to the current user and never in
  shared temporary directories. Credentials go in secure storage.
- Full requirements and implementation patterns:
  `references/addon-index-qualities.md` §5.

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

### QML tests

Run QML tests offscreen, without FreeCAD, against a real `QQuickView`:

```bash
QT_QPA_PLATFORM=offscreen python tests/test_panel_qml.py
```

- Install a Qt message handler and fail on `binding loop`, `is not defined`,
  `Cannot read property`, `TypeError`, `ReferenceError`, and `required
  property`.
- Load every root view; check `view.status()` and `view.errors()`. Show the
  view and let real time pass (animations use wall-clock time).
- Stub the bridge and theme from the real classes' properties, slots, and role
  names; keep the stubs in sync so a deleted API fails the test.
- Test delegate creation, control-to-bridge calls, and the geometry the user
  notices.
- Add a no-Qt audit: every QML component is registered in a `qmldir` and
  imported; every `bridge.*` / `theme.*` name exists; every delegate
  `required property` matches a model role.
- Run `qmllint` if available, and include `.qml` files in `lupdate`. Keep both
  checks: lint does not find runtime binding loops.

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
# QML addons: run the offscreen QML tests and the reference audit too
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
- QML dock/floating panel opens on first use, restores placement, and survives
  a theme and language switch.
- QML overlay hit testing, keyboard focus, popups on Wayland, and repaint
  behind the panel work on the target platform.
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
- [ ] License files exist; exact SPDX identifiers match manifest, files, and
      per-file headers.
- [ ] Latest FreeCAD tested, or `freecadmin`/`freecadmax` states the honest
      range.
- [ ] All Python dependencies are necessary and allow-listed; no runtime
      package installs; vendoring justified, isolated, and licensed.
- [ ] README and `SECURITY.md` updated; privacy/external-services disclosure
      matches the code.
- [ ] In-FreeCAD consent is per service/data, revocable, and re-asked on change;
      no default service or ambient credentials.
- [ ] Credentials in secure storage; private files use restricted permissions
      (`0600` on POSIX); no secrets in logs/repo/shared temp.
- [ ] AI addons: exact approved code, auto-approval off by default with in-UI
      warning, bridge prompt intact, listeners loopback/authenticated/verified
      TLS, one undo per run.
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

### Governance, compliance, and licensing

- [ ] Active maintainer with working contact; repository at least 30 days old.
- [ ] `SECURITY.md` with a private reporting route, supported versions, and
      prompt security-fix/advisory practice, including previously released
      versions.
- [ ] External services, data sent, trigger, frequency, retention, and opt-out
      are documented in the README.
- [ ] In-FreeCAD consent before any external data transfer, per service and data
      kind; re-asked when either changes; revocable; no default service; no
      ambient credentials before consent.
- [ ] Credentials in the OS keyring where available; other private data in a
      restricted file (`0600` on POSIX) under the user's app-data directory;
      nothing in shared temp, the addon directory, logs, or the repo.
- [ ] Exact SPDX identifiers agree across `package.xml`, license files, and
      per-file headers; full license texts shipped; every license declared.
- [ ] Python dependencies are necessary, typed, and allow-listed; no runtime
      package installs; vendoring is justified and isolated with its upstream
      license.
- [ ] Latest FreeCAD tested, or `freecadmin`/`freecadmax` honestly restricts
      the supported range and the README says so.

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
- [ ] Runtime imports use `PySide`, not `PySide2`/`PySide6`, except
      `QtQml`/`QtQuick`/`QtQuickWidgets`, which come lazily from `PySide6`.
- [ ] QML is created only in GUI code after the main window exists; load
      errors show a fallback, not a blank panel.
- [ ] One bridge per QML surface; QML never changes documents or owns
      transactions; context objects stay alive.
- [ ] No model rebuild or view teardown from a QML handler; no context
      property is nulled; the view is deleted instead.
- [ ] QML models are flat with role names, diffed rebuilds, targeted
      `dataChanged` after state changes.
- [ ] QML colours/metrics come from a live palette-backed theme; Controls, if
      used, are `QtQuick.Controls.Basic` and restyled.
- [ ] `qsTr()` wraps QML text; `engine.retranslate()` runs on language change.
- [ ] QML tests load real `QQuickView`s offscreen and the reference audit
      passes.
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

### AI, code execution, and listeners

- [ ] The complete code is shown before it runs (inside FreeCAD when the addon
      runs the agent), and only the exact shown code runs; file contents are not
      re-read after approval.
- [ ] Model-chosen file reads/writes require approval; fixed document operations
      with model-supplied values are allowed without it.
- [ ] Each approved run is one undo transaction; errors abort it.
- [ ] Auto-approval is only a setting, off by default, and warns in the UI; the
      addon does not preset it when launching/configuring an agent.
- [ ] A bridge keeps the agent's approval prompt, adds no auto-approve config or
      skip-asking tool descriptions, and warns inside FreeCAD when started.
- [ ] Every remote AI request, even model listing, is preceded by in-FreeCAD
      consent naming the service and all data (including automatic data); per
      service; re-asked on change; no default service or ambient credentials.
- [ ] Listeners start only when enabled, authenticate every request with no off
      switch, bind loopback, reject browser/out-of-protocol input, and use
      explicit non-wildcard remote addresses with verified TLS.
- [ ] Agent tokens/config/instructions/workdirs/screenshots are private
      (restricted permissions, app-data, no shared temp); credentials use
      secure storage.
- [ ] Name/icon/description do not imply FreeCAD or AI-vendor endorsement.

### Quality and security

- [ ] Tests cover non-trivial logic and migrations.
- [ ] No `eval`/`exec` on untrusted input; AI code execution has consent and a
      transaction.
- [ ] QML loads only files shipped with the addon; no JavaScript `eval` or
      `Qt.createQmlObject` on untrusted input.
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
| `.qrc` / `.rcc` / `pyside6-rcc` | Tied to the Qt major version | Plain SVG + `Gui.addIconPath()`; plain QML under `resources/qml/` |
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
| Runtime `PySide2` / `PySide6` imports | Breaks across FreeCAD Qt builds | `from PySide import ...`; `PySide6` only for `QtQml`/`QtQuick`/`QtQuickWidgets`, lazily |
| QML built at import or in `InitGui` | Main window missing; startup cost | Deferred `QTimer.singleShot(0, ...)` creation |
| QApplication palette used for QML | Ignores FreeCAD's stylesheet | Palette from a polished reference widget |
| Hardcoded colors/fonts/pixel sizes | Breaks dark mode, themes, HiDPI | Palette, layouts, SVG icons; QML theme object |
| Nulling QML context properties to tear down | Bindings re-evaluate against null | Delete the `QQuickWidget` |
| Model reset or rebuild from a delegate handler | `qFatal`; FreeCAD exits | `Qt.callLater` / zero-timer |
| Building QML in a bare `QQmlComponent` test | No delegates, false pass | Offscreen `QQuickView` with `show()` |
| Duplicated version strings | Drift between manifest, package, and UI | One source of truth + build sync |
| Vendored helper library | Index policy discourages it; stale/security risk | Declared dependency or isolated `_vendor/` with a strong reason |
| `exec`/`eval` of user code without approval | Arbitrary code execution | Show code, require consent, run in one transaction |
| Consent only in the README, or a default cloud service | Data leaves without permission; Index/GDPR failure | In-FreeCAD consent per service/data; no default service |
| Previewing code, then re-reading the file to execute it | Runs code the user never saw | Execute the exact text that was displayed |
| Auto-approve on by default, or an agent config with approvals disabled | Model code runs unreviewed | Off by default; in-UI warning; keep the agent prompt |
| Listener on `0.0.0.0`, optional auth, or unchecked TLS | Anyone reachable can run code; traffic unverified | Loopback; mandatory auth; verified TLS for explicit remote addresses |
| Tokens or agent files in `/tmp` or the addon directory | Secrets are world-readable or survive reinstall | App-data with restricted permissions; keyring for credentials |
| `LGPL2` or another non-SPDX license string | Normalized to a more restrictive license; manifest/header drift | Exact SPDX ID in manifest, files, and headers |
| Shipping a release branch full of alpha scaffolding | Index rejects incomplete addons | Finish features or keep them on a development branch |

## 9. Severity guide

When reporting review findings:

- **HIGH** — install/startup broken; data loss; security issue; saved documents
  fail to load; manifest invalid or branch mismatch; root/freecad `__init__.py`;
  unconsented data transfer or a default remote service; unauthenticated or
  wildcard listener; auto-approval of model code on by default; secrets in
  shared temp; a QML model/view rebuilt from its own handler (FreeCAD exits); a
  live QML context object that can be collected.
- **MEDIUM** — command doesn't work; undo broken; heavy startup cost; UI blocks
  or crashes; broken translations; missing migration; Coin accessed from
  `execute()`; QML built at import, blank QML on a load error, wrong theme
  palette, or no `engine.retranslate()` on a language change.
- **LOW** — icon size/format; missing docs; style/consistency; performance
  polish; QML control not restyled to the theme.

A review is not a rewrite request. Fix the smallest complete set of issues that
makes the addon correct, safe, and maintainable.
