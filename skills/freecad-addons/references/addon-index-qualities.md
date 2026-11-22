# Addon Index admission qualities

The FreeCAD Addon Index is curated. Passing Addon-Reports, or having a valid
`package.xml`, is necessary but not sufficient: an addon can be rejected or
removed for governance, privacy, licensing, codebase, or AI-agent reasons.

Official requirements:
<https://freecad.github.io/Addon-Academy/Topics/Addon-Index/Index/Qualities.html>

Use this reference when preparing a release for the Index and when reviewing an
addon against the Index requirements. It paraphrases the requirements and adds
the FreeCAD-specific implementation and evidence expected by reviewers. When the
official page and this file disagree, the official page wins.

The Index requirements are a floor, not a score. “Curation” is a stricter,
human-reviewed subset; it does not replace any requirement below.

## 1. Governance

An indexed addon must stay maintained:

- At least one active maintainer, or a documented, active search for one.
- Issues, pull requests, and security advisories are answered in a timely
  manner. Abandoned addons are removed.
- The repository and codebase are at least 30 days old. This cannot be
  shortened by a good implementation; plan the submission after the project has
  existed for a month.

Evidence to provide:

- A real maintainer and working contact in `package.xml` and `README.md`.
- A `SECURITY.md` with a private reporting route, supported versions, and the
  expected response.
- Issue/PR templates or a short contribution note; respond to reports instead of
  letting them rot.
- If the addon is orphaned, say so and look for a maintainer. Do not present an
  unmaintained addon as active.

## 2. Compliance and privacy

The addon must comply with GDPR and be honest about every external connection.
Reviewers look for the behavior, not just a README paragraph.

### Data handling rules

- Do not send user data to third parties unless the user expressly permits it.
- Send the minimum data needed for the feature. Prefer no data.
- Tell the user, in FreeCAD, which service is contacted, what data is sent,
  when, how often, and why. A README notice is not consent.
- Ask again when the service changes, or when the kinds of data sent change.
- Let the user revoke consent and stop the feature.
- Do not enable a remote service by default. Do not reuse API keys, tokens, or
  logins found on the system until the user has consented to that service.
- Network access is consented and disclosed; no network at import or startup,
  and no silent background requests.

### Consent UX

Consent is asked inside FreeCAD, before the first request reaches the service:

```python
import FreeCAD as App
import FreeCADGui as Gui
from PySide import QtWidgets

translate = App.Qt.translate

answer = QtWidgets.QMessageBox.warning(
    Gui.getMainWindow(),
    translate("MyAddon", "External service"),
    translate(
        "MyAddon",
        "This sends your prompt, the active document name, and error output "
        "to https://api.example.com. It is sent when you run Generate. "
        "Continue?",
    ),
    QtWidgets.QMessageBox.Yes | QtWidgets.QMessageBox.No,
    QtWidgets.QMessageBox.No,
)
```

Implementation expectations:

- Show the service name/endpoint and **every** kind of data, including data sent
  automatically: prompts, object names and dimensions, file paths, file
  contents, screenshots, error output, and so on.
- Consent is per service. Store the accepted scope in `App.ParamGet`, keyed by
  service and by a data-scope version/hash, so a later data change triggers a
  new prompt instead of inheriting the old permission.
- Provide a preference to review and revoke consent. Revoking stops requests;
  it does not only hide a checkbox.
- Wrap every request in a check that refuses to send without current consent.
  Do not rely on the UI path alone.
- Document the same facts in `README.md` (for example a `## Privacy and external
  services` table: service, endpoint, data sent, trigger, frequency, consent and
  opt-out, retention). A `PRIVACY.md` is optional; the in-app consent is not.

Consent state is small and non-secret, so the parameter store is enough. Key it
by service and bump the scope when the service or data changes:

```python
CONSENT_SCOPE = 2  # bump when the service or the data sent changes
consent = App.ParamGet(
    "User parameter:BaseApp/Preferences/Mod/MyAddon/Consent"
)


def ensure_consent(service_id: str) -> bool:
    key = f"{service_id}:scope{CONSENT_SCOPE}"
    if consent.GetBool(key, False):
        return True
    if not ask_consent_dialog(service_id):  # the dialog above
        return False
    consent.SetBool(key, True)
    return True
```

### Local data and secure storage

- Credentials, tokens, and pairing secrets go in the user's secure storage:
  the OS keyring where the platform/FreeCAD environment exposes one. Do not
  declare a package that is not on the allow-list just to get a keyring.
- If secure storage is unavailable, use a private file under
  `App.getUserAppDataDir()` with restrictive permissions (`0600` on POSIX), and
  never a shared temporary directory, the addon source directory, or a
  world-readable path.
- Do not write secrets to logs, reports, crash dialogs, or the document.
- Apply the same privacy to agent files: configuration, instructions, working
  directories, screenshots, and pairing secrets are readable only by the
  current user.

### Security response

- Accept security reports privately; do not require a public issue.
- Fix reports promptly, including in previously released versions. Publish a
  patched release.
- Tell users about the issue when appropriate, with enough detail and local
  remediation steps to protect themselves (upgrade, disable a feature, rotate a
  token, remove a file).
- Keep a supported-versions table in `SECURITY.md`; do not silently drop old
  releases without saying so.

## 3. Licensing

- Choose one or more fitting licenses and include the full license text when
  the license requires it. The Addon Academy recommends
  `LGPL-2.1-or-later` for code and `CC-BY-SA-4.0` for non-code assets when
  there is no other preference.
- Use exact SPDX identifiers. A non-standard string is normalized to the most
  restrictive match: `LGPL2` becomes `LGPL-2.0`, which is probably not what the
  author intended. Write `LGPL-2.1-or-later`.
- Declare every license used in the package, one `<license>` element per
  license, each with a `file` that exists:

```xml
<license file="LICENSE-Code">LGPL-2.1-or-later</license>
<license file="LICENSE-Assets">CC-BY-SA-4.0</license>
```

- Keep the license consistent everywhere:
  - `package.xml` `<license>` tags;
  - the referenced `LICENSE-*` files;
  - per-file SPDX headers, if used;
  - `pyproject.toml` and README license statements.
- Include the upstream license and attribution for vendored code or copied
  assets. “MIT” in a README is not enough; ship the text.
- Do not use `UNLICENSED` or `SEE LICENSE IN <file>` to avoid shipping required
  text. Choose a fitting license; the Index expects addons to be clear about
  their licensing.

Review check: every source file that carries an SPDX header must name a license
declared in the manifest, and every manifest license must have a file.

## 4. Codebase

### Language and Qt

- Python 3 only; declare `pythonmin` honestly.
- Use FreeCAD's Qt wrappers (`from PySide import QtCore, QtGui, QtWidgets`) at
  runtime. `PySide2`/`PySide6` runtime imports are rejected practice; the only
  exception is `QtQml`/`QtQuick`/`QtQuickWidgets`, imported lazily from
  `PySide6` because the shim does not expose them.

### Manifest

The manifest must be valid and accurate:

- `<content>` matches what the package actually provides:
  `<workbench>` for a real workbench, `<other/>` for a toolbar/menu extension,
  `<macro>` for macros, `<preferencepack>` for preference packs/themes.
- For a workbench, `<classname>` exactly matches the Python workbench class
  registered in `init_gui.py` (or legacy `InitGui.py`).
- `<icon>` exists and is SVG.
- `<url type="repository" branch="...">` names the branch (or tag) where that
  `package.xml` actually lives. A mismatch is a hard failure.
- `<freecadmin>` and `<freecadmax>` are honest. Use `1.1.99` to cover a whole
  `1.1.x` series if needed.
- Required fields are present: `name`, `version`, `date`, `description`,
  `maintainer`, `license`, `icon`, `content`.
- Use explicit `type="python"`, `type="addon"`, or `type="internal"` on
  `<depend>`; do not rely on `automatic`.

### UI confinement

- Keep commands, toolbars, menus, docks, and other side effects inside the
  addon's own workbench.
- When contributing to other workbenches, use a scoped `WorkbenchManipulator`
  and return `[]` for workbenches you do not intend to change.
- Do not modify, hide, reorder, or rewrite FreeCAD's core interface or another
  addon's interface. Do not register global shortcuts or stylesheet overrides.
- Prefix all global names (`MyAddon_DoThing`, toolbar/menu names).

### Startup and compatibility

- No expensive work, network access, subprocesses, threads, dialogs, geometry, or
  global side effects at import or startup. Defer to `Initialize()`, `Activated()`,
  or the user's command.
- Be compatible with the latest FreeCAD. If the addon is intentionally for older
  versions, declare that range with `freecadmin`/`freecadmax` and say so in the
  README.
- Test the versions you claim. Do not claim compatibility you have not tested.

### Dependencies and vendoring

- A Python dependency must be functionally necessary, declared as
  `type="python"`, and on the Addon Manager allow-list:
  - FreeCAD 1.1+:
    `FreeCAD/Addons` → `Data/Python/<python-version>/Allowed-Packages`
  - legacy: `FreeCAD/FreeCAD-addons` → `ALLOWED_PYTHON_PACKAGES.txt`
- Request an addition with the `Package - Addition` issue template in
  `FreeCAD/Addons`; do not work around the list.
- Do not install Python packages at runtime, into FreeCAD's environment or any
  other. The Addon Manager installs allow-listed packages. A separate program
  the user installs (for example an MCP server with `pip`/`uv`) may manage its
  own environment.
- Do not put version specifiers for Python dependencies in `package.xml`;
  versions are pinned centrally by the allow-list constraints.
- Vendoring is strongly discouraged and is grounds for rejection. Vendor only
  when the package cannot reasonably be allow-listed, the code is pure Python
  (or your own fork), the license permits redistribution, and the copy is small
  or stable. Never vendor an allow-listed package or a package with compiled
  extensions.
- If vendoring is technically unavoidable, isolate it under
  `freecad/<ModName>/_vendor/`, preserve the upstream `LICENSE`, add a
  `_vendor/README.md` with source/version/modifications, and import it through
  your namespace:
  `from freecad.MyAddon._vendor.somepkg import thing`. Never import the
  vendored copy by its upstream top-level name.

### Completeness

- Pre-alpha and alpha code is not suitable for the Index. The tracked release
  branch must be substantially complete.
- Every command in a menu/toolbar works.
- The README states installation, usage, requirements, compatibility, known
  limitations, and which features work and which do not.
- Remove user-visible placeholders: `pass`-only commands, `TODO`/`FIXME` paths,
  “not implemented” buttons, dead menu entries.
- Address the public Bandit report for the addon:
  <https://freecad.github.io/Addon-Reports/>. Fix real findings; suppress only a
  narrowly scoped false positive with a justification.

## 5. AI agents, model-generated code, and bridges

These requirements apply when an addon lets an AI model generate code or
commands that run inside FreeCAD, whether the addon runs the agent itself or
exposes FreeCAD to an outside agent (for example an MCP server, a *bridge*).

The package allow-list still applies. The addon may not install Python packages
itself; a separate user-installed MCP server may manage its own environment.

### Approval: show the code, then wait

- The user sees the complete code before it runs.
- For an addon that runs the agent itself, the code review and approval happen
  inside FreeCAD, not in an external agent UI.
- Only the code the user approved runs, exactly as shown. For code read from a
  file, execute the contents that were displayed, not a fresh read of the file.
- Tools that read or write files at paths the model chooses also need approval.
  Fixed operations on the document with values supplied by the model do not.
- Documents and data the model reads can contain third-party instructions.
  The approval review is the user's main protection; do not undermine it with
  vague summaries.
- Wrap each approved run in a single undo transaction, so one undo reverts the
  whole run:

```python
import FreeCAD as App

approved_text = show_code_and_wait_for_consent(code)  # exact displayed text
if approved_text is None:
    return

doc = App.activeDocument()
doc.openTransaction("AI generated change")
try:
    # The approval is the security boundary; a narrow namespace is only hygiene.
    exec(compile(approved_text, "<approved>", "exec"), {"__name__": "__main__"})
    doc.commitTransaction()
except Exception:
    doc.abortTransaction()
    raise
```

Bandit will flag `exec`. That is expected for an addon whose feature is approved
code execution: keep the finding visible, add a narrow `# nosec B102` with the
justification that the shown code was explicitly approved, and do not hide the
rest of the addon from Bandit.

### Automatic approval

- Automatic approval is opt-in: it may only be a setting, off by default.
- The warning must be in the interface itself, not only a tooltip or the README:
  the model's code will run with the user's full permissions.
- An addon that launches or configures an agent must not set it to approve code
  execution or file tools automatically. Setting it to deny them is fine.
- Do not add an “auto-approve” default, a hidden config, or a first-run prompt
  that preselects it.

### Bridges

- The agent's own approval prompt is the approval step. The bridge must not
  weaken it:
  - no agent configuration that approves tools automatically;
  - no tool descriptions telling the agent to skip asking;
  - no tool that runs code from a file unless FreeCAD shows the file's contents
    first.
- When started, the bridge warns inside FreeCAD that connected agents can run
  code with the user's full permissions.

### Remote AI services

- Before any request reaches a remote AI service — even a request that only
  lists models — the addon asks for consent inside FreeCAD.
- The request names the service and every kind of data sent, including data sent
  automatically.
- Consent is per service and is asked again when the service or the kinds of
  data change.
- No remote AI service is selected by default.
- Do not use API keys or logins found on the system (environment variables,
  another program's saved login) until the user has consented to that service.
  An addon built around one vendor's tool may then use the user's existing login
  for that vendor.

### Network listeners and MCP servers

Listeners are the highest-risk surface. Apply all of these:

- Start the listener with FreeCAD only if the user has enabled that. Listening
  is not a default.
- Require authentication on every request after the handshake, with no setting
  to turn it off.
- HTTP listeners reject browser requests: reject any request with an `Origin`
  header, or a `Host` header that does not match the configured address. Other
  listeners close the connection on any input outside their protocol.
- Bind to loopback (`127.0.0.1`/`::1`) by default. Prefer remote access through
  an SSH tunnel, or start the MCP server on the remote machine over SSH, rather
  than exposing FreeCAD directly.
- Offer another bind address only if all of the following hold:
  - the setting is off by default;
  - before the listener starts on each new address, FreeCAD warns that anyone
    who can reach and authenticate to it can run code with the user's full
    permissions, and the user accepts;
  - the address is a specific one the user chose, never a wildcard such as
    `0.0.0.0` or `::`;
  - traffic is always encrypted with Python's `ssl` module or another
    maintained library, not the addon's own cryptography;
  - the client verifies the server, for example with a pinned certificate or
    TLS-PSK. An unchecked self-signed certificate does not count; do not set
    `CERT_NONE`, `check_hostname=False`, or an “ignore certificate” option;
  - clients the addon does not control (an agent's MCP client, a chat front
    end) can also verify the server; explain where the setting lives and how to
    set it up, and never suggest turning verification off.

### Agent files and credentials

- Tokens, pairing secrets, agent configuration, instructions, working
  directories, and screenshots are readable only by the current user.
- Keep them out of shared temporary directories. Use the user's app-data area
  with restrictive permissions.
- Credentials go in secure storage where the system provides it (see
  Compliance).

### Branding

- The addon's name, icon, and description must not suggest it is an official
  product of the AI vendor or of FreeCAD. Descriptive names such as “for Claude
  Code” or “Gemini connector” are fine.
- Do not use vendor logos in a way that implies endorsement.

### Existing addons

Addons already in the Index when these rules were introduced have 60 days from
notification to comply. New addons must comply from the start.

## 6. Best practices that affect admission

- Use the modern `freecad/<ModName>/` layout. Legacy `Init.py`/`InitGui.py` is
  tolerated, but discouraged; legacy addons must not manipulate `sys.path` or
  rely on top-level module names that can collide with other addons.
- Publish from a stable release branch. Develop on another branch; merge only
  for a release, and bump `<version>` and `<date>` every time the tracked branch
  changes.
- Keep the release branch free of alpha scaffolding and generated clutter.
- For model-generated code, one approved run is one undo transaction.
- Keep documentation and metadata consistent: version, date, branch, license,
  content type, and classname cannot drift.

## 7. Definition of done

Use this as the final gate before submitting to the Index or approving a release.

| Area | Pass condition |
|---|---|
| Governance | Active maintainer/contact; issue and security reports answered; repo ≥ 30 days old. |
| Privacy | No unconsented data transfer; in-FreeCAD consent names service and all data; per-service; re-asked on change; revocable. |
| Disclosure | README/PRIVACY documents service, data, trigger, frequency, retention, opt-out. |
| Local data | Credentials in keyring where available; other private data restricted under app-data (`0600` on POSIX); no secrets in logs/repo/shared temp. |
| Security response | `SECURITY.md`; private reports; patched supported/previous releases; user advisory with remediation. |
| Licensing | Exact SPDX IDs; manifest, files, headers agree; full texts shipped; every license declared. |
| Manifest | Schema-valid; correct `<content>`; workbench `<classname>` matches; repository `<branch>` matches; icon exists. |
| Compatibility | Latest FreeCAD tested, or honest `freecadmin`/`freecadmax` and documented range. |
| Dependencies | Necessary, typed, allow-listed; no runtime installs; no version specifiers in `package.xml`. |
| Vendoring | Only if allow-list is impossible; `_vendor/` with license/readme; no allow-listed or compiled packages. |
| UI scope | Own workbench or scoped manipulator; no core/other-addon modifications; prefixed names. |
| Startup | No expensive work, network, subprocess, thread, dialog, or global UI side effect at import/startup. |
| Completeness | Release branch substantially complete; working commands; README states what works and does not. |
| Bandit | Public report reviewed; real findings fixed; only narrow justified suppressions. |
| AI approval | Complete code shown; exact shown code runs; file reads/writes at model-chosen paths approved; one undo per run. |
| AI auto-approval | Setting only, off by default, warned in the UI; launcher/bridge never enables it; bridge keeps agent prompt. |
| AI remote | In-FreeCAD consent before any request; no default service; no ambient credentials before consent; per-service and data re-consent. |
| AI listeners | Opt-in start; authentication always; loopback default; explicit address only; verified TLS for remote; third-party clients can verify. |
| AI files | Agent secrets/config/screenshots private; no shared temp; credentials in secure storage. |
| Branding | No implied FreeCAD/vendor endorsement. |

## 8. Common reasons addons fail review

| Shortcut | Why it fails | Do instead |
|---|---|---|
| Consent only in the README | Users never see it before data leaves; no re-consent | In-FreeCAD dialog before the first request; document the same facts |
| A default cloud service or endpoint | Sends data without permission | No default; explicit service selection and consent |
| Reusing ambient API keys/logins | Uses credentials the user did not consent to share with that service | Ask first; use the vendor login only after consent for that vendor |
| “Auto-approve” default, or agent config with approvals disabled | Model code runs without review | Off by default; warn in the UI; bridge leaves the agent prompt intact |
| Executing a fresh file read after showing a preview | Runs code the user did not see | Execute the exact displayed text |
| One transaction per model command instead of per approved run, or none | Partial AI changes cannot be undone in one step | One transaction per approved run |
| Listener on `0.0.0.0`, auth optional, self-signed certificate | Anyone on the network can run code, or traffic is unverified | Loopback, mandatory auth, verified TLS for explicit remote addresses |
| Tokens or agent config in `/tmp` or the addon directory | Other users can read secrets; reinstall leaks them | Private app-data directory with restrictive permissions; keyring for credentials |
| Code execution without an undo transaction | User cannot revert a bad run | Wrap the run in one transaction and abort on error |
| Commands added to core menus at import | Interferes with FreeCAD and other addons | Own workbench or scoped manipulator |
| Network or heavy work at import | Slows every startup and may leak data | Defer to an explicit command |
| Vendoring an allow-listed package | Duplicates versions, bypasses review | Declare `<depend type="python">` |
| `LGPL2`, “GPL”, or another non-SPDX string | Normalized to a more restrictive license; manifest/header mismatch | Exact SPDX ID everywhere |
| Missing/incorrect repository branch | Addon Manager fetches the wrong branch; hard failure | Match `branch` to the branch containing `package.xml` |
| Alpha scaffolding on the tracked branch | Index rejects incomplete addons | Finish it or keep it on a development branch |
| Untested “latest FreeCAD” claim | Users hit immediate breakage | Test latest, or declare an honest max |

## 9. Review procedure

1. Read `package.xml`, `README.md`, `SECURITY.md`, `PRIVACY.md` (if present),
   and the license files.
2. Check the manifest against the branch, content type, classname, version/date,
   and SPDX requirements.
3. Trace every network call, every credential read, and every listener. Confirm
   in-FreeCAD consent and secure storage.
4. For AI addons, trace the approval path, the exact code executed, the undo
   transaction, auto-approval defaults, and bridge configuration.
5. Check UI scope and startup work.
6. Run the automated gates (ruff, Bandit, tests) and inspect the public
   Addon-Reports entry.
7. Report only violations, with file/line and the requirement broken. Rank by
   impact; do not ask for unrelated rewrites.
