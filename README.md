# FreeCAD agent skills

OpenCode skills for FreeCAD addon development. Each skill is a directory with a
`SKILL.md` entry point plus reference files the agent loads on demand.

## Skills

### `freecad-addons` — FreeCAD 1.1+ addon development

Use when creating, reviewing, refactoring, packaging, or debugging a FreeCAD
addon: workbenches, commands, preferences, translations, FeaturePython document
objects, extensions, observers, and Pivy/Coin3D view providers. Covers the
modern namespaced layout, `package.xml` metadata, testing, security, and release
quality. Not for FreeCAD core C++ work or single-file macros.

| Reference | Contents |
|---|---|
| `layout-and-manifest.md` | Directory layout, entry points, `package.xml`, `pyproject.toml`, release |
| `ui-and-commands.md` | Workbench, commands, menus/toolbars, Qt dialogs, logging |
| `preferences-and-widgets.md` | Preference pages, presets, validation, input widgets, progress |
| `resources-i18n.md` | Icons, SVG, translations, Qt tooling |
| `document-objects.md` | FeaturePython objects, properties, extensions, ViewProviders, migrations |
| `pivy-coin.md` | Coin3D scene graphs, display modes, camera, context menus, drag/drop |
| `events-and-lifecycle.md` | Document/object/selection observers, view callbacks, cleanup |
| `freecad-api-map.md` | Key FreeCAD API methods by area |
| `quality-checklist.md` | Tests, security, CI, Addon-Reports checks, review checklist |

## Install

### Opencode or similar

```bash
npx skills add mnesarco/freecad-skills --skill freecad-addons
```

### Direct clone

Clone and copy into your OpenCode skills directory:

```bash
git clone https://github.com/mnesarco/freecad-skills.git
cp -r freecad-skills/skills/freecad-addons ~/.config/opencode/skills/
```

Restart OpenCode; the skill is picked up from its `SKILL.md` frontmatter.

## Layout

```text
skills/
└─ freecad-addons/
   ├─ SKILL.md
   └─ references/*.md
```

