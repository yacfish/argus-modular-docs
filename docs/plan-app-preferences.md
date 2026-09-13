# App preferences & Argus menu

**Updated**: June 2026  
**Status**: **Shipped** (PREF-1–4 on `main`). **Asset resolver** (`search_paths`) wired in AR1 — [plan-asset-ref.md](plan-asset-ref.md).  
**Related:** [architecture.md](architecture.md) §2, §8 · [keyboard-shortcuts.md](keyboard-shortcuts.md)

**Goal:** App-level settings in user `preferences.json`; **Argus** menu + extended **File** menu; full **Preferences** modal (all sections including UX remap); shipped **templates** tree; **search paths** for shared assets. Load at boot, save on **Apply**.

**Not in this track:** bundle/preset format changes, autosave, cloud sync, user template folders, grid/snap canvas prefs.

---

## 1. Menus

### Argus (leftmost)

| Item | Notes |
|------|-------|
| **About Argus v_X.X** | Version (`ARGUS_UI_VERSION`), core pin, links — modal, no persist |
| **Preferences…** | Full settings modal — **⌘,** / **Ctrl+,** |

### File (extended)

| Item | Shortcut |
|------|----------|
| New document | ⌘⇧N |
| Open bundle… | ⌘O |
| **Open recent ▸** | Hover submenu — `recent_bundles[]` from prefs (§1.1) |
| **Open template ▸** | Hover submenu — shipped `templates/` tree (§1.2) |
| **Reload from disk** | ⌘R — re-read saved `bundle_path_`; dirty prompt |
| Save / Save as… | ⌘S / ⌘⇧S |

### 1.1 Open recent ▸ (hover submenu)

- **File → Open recent ▸** opens a **nested menu to the right** on hover (ImGui `BeginMenu`).
- Items: **valid** paths from `recent_bundles[]` (most recent first), **excluding the currently open bundle**; label = bundle filename (e.g. `my-show.bundle`).
- **Validity:** directory exists and contains `manifest.json` (same rule as Open bundle). Invalid/missing paths are **excluded** from the menu and pruned on prefs load.
- Empty list (after filtering): submenu present but disabled (“No recent documents”).
- Picking an item opens that bundle (dirty prompt if needed). No modal.

### 1.2 Open template ▸ (hover submenu)

- **File → Open template ▸** opens a **nested menu to the right** on hover.
- Second level: **category** folders under `templates/`.
- Third level: **graph name** = bundle directory stem (e.g. `video-pipeline` from `video-pipeline.bundle`).
- No `catalog.json`, no descriptions, no extra metadata — folder layout is the index.

### Templates (shipped — not a preference)

```
templates/
  <category>/
    <graph_name>.bundle/
      manifest.json
      graphs/...
```

- Repo: `templates/<category>/<graph_name>.bundle/` — normal bundle dirs only.
- Build: stage whole `templates/` tree next to `argus_ui` (like `media/`).
- Runtime: scan `templates/` — category = immediate subdir; bundle = child dir named `*.bundle` **and** containing `manifest.json` (invalid entries skipped).
- **Pick template** → load staged bundle in memory as **Untitled\*** (no path; first save prompts Save As). Staged originals stay read-only.
- No user template path in prefs.

### Side panel

Remove **Debug** tab. Tabs: **Monitor · Node · Presets** (`Tab` cycles Monitor → Node → Presets → hidden). Debug only in Preferences.

---

## 2. Preferences modal

- Centered **80%** of window; dimmed modal backdrop; blocks app input.
- Sidebar: **System · Documents · UI · Monitor · Debug · UX**
- **Cancel** / **Apply** — disabled until draft ≠ applied; **Esc** = Cancel.
- **Apply-only** (no live preview except optional UI colour preview if cheap).

### Persistence

| OS | Path |
|----|------|
| macOS | `~/Library/Application Support/ArgusModular/preferences.json` |
| Linux | `$XDG_CONFIG_HOME/argus-modular/preferences.json` |

```json
{
  "version": 1,
  "system": { },
  "documents": { },
  "ui": { },
  "monitor": { },
  "debug": { },
  "ux": { }
}
```

---

## 3. Sections (complete delivery)

### System

Startup (empty vs reopen last bundle; default **reopen_last** — uses `last_bundle_path` at boot unless `--bundle` is passed), executor tick cap, VSync, GPU backend preference (stored; applied when GP1 exists), **Reset to defaults** (confirm → resets all sections in draft). Label **Restart graph** / **Restart app** where needed.

### Documents

`last_bundle_path`, `recent_bundles[]` (valid bundles only; pruned on load; stored cap = **`max_recent_bundles + 1`** so the menu can still show `max_recent_bundles` after excluding the open doc), **`max_recent_bundles`** (user setting, default 10, clamped 1–50), default open/save directory, **`search_paths[]`** (ordered absolute folders). Preferences **Documents** section exposes `max_recent_bundles`.

**Resolver** (bundle path first, then each search path): graphs, `scripts/*.lua`, media file refs. Wire into `GraphSession` load/save and file pickers.

### UI

Theme preset, canvas background, side panel width, window geometry restore, wire/selection colours (move from `graph_canvas_detail` constants).

### Monitor

GM default behaviour (follow selection vs sticky), preview viewer mode (`image` / future `gl_viewer`), max preview resolution, refresh policy (hovered vs always). JSON keys stable for GPU work.

### Debug

`fps` logging, all `editor_log::Tag` toggles, param change log enable/clear, sticky trace verbose. Hydrate at boot; no side-tab mirror.

### UX

Full **shortcut remapping** for every action in [keyboard-shortcuts.md](keyboard-shortcuts.md): semantic action id → chord, conflict detection, menu labels driven from binding table. Behaviour toggles: default edit mode on open, sticky-duplicate reset policy, marquee modifiers (as listed in plan backlog). No cheat-sheet-only mode.

---

## 4. PR stack

Merge order bottom → top. Each PR is reviewable; stack ships as one feature.

```
PREF-1 ──► PREF-2 ──► PREF-3 ──► PREF-4
```

| PR | Title | Scope | Exit |
|----|-------|-------|------|
| **PREF-1** | App preferences & asset resolver | `app_preferences.{hpp,cpp}` (struct, load/save, migrate, apply/hydrate); platform path; `ARGUS_UI_VERSION`; `resolve_asset_path(bundle, relative, search_paths)` + unit smoke; boot load in `App` | `argus_test_app_preferences_smoke`, `argus_test_asset_resolver_smoke` · **shipped** |
| **PREF-2** | Menus, file ops, templates | **Argus** menu + About modal; **File**: Reload, **Open recent ▸**, **Open template ▸** (hover submenus); `templates/<category>/<name>.bundle/` + `cmake/stage_templates.cmake` + directory scan; update recents/last path on open/save; Reload implementation | Manual: recent/template/reload · `argus_test_template_catalog_smoke` · **shipped** |
| **PREF-3** | Preferences modal & sections | `preferences_panel.{hpp,cpp}` — modal chrome, sidebar, Cancel/Apply dirty; wire **System, Documents, UI, Monitor, Debug**; remove Debug side tab; startup reopen last bundle; System reset | All non-UX prefs persist + Apply |
| **PREF-4** | UX bindings & integration | `keyboard_bindings.{hpp,cpp}` — action registry, rebind UI in Preferences **UX**, conflict checks; rewire `app.cpp`, `graph_canvas`, menus to binding table; sync [keyboard-shortcuts.md](keyboard-shortcuts.md); `argus_test_keyboard_bindings_smoke` | Full remap works; docs match |

**PREF-3** and **PREF-4** can be parallelized after **PREF-2** if two reviewers; **PREF-4** must follow binding hooks from **PREF-3** menu integration or touch same files last.

### Files (expected)

| Area | New / touched |
|------|----------------|
| Prefs | `app_preferences.*`, `preferences_panel.*`, `about_modal.*` |
| Assets | `asset_resolver.*` or `graph_session` resolver |
| Templates | `templates/`, `cmake/stage_templates.cmake` |
| UX | `keyboard_bindings.*`, `keyboard_shortcuts.hpp` |
| Shell | `app.cpp`, `app.hpp`, `edit_mode_panel.*`, `editor_log.*` |
| Build | `CMakeLists.txt`, `project(VERSION)` |

---

## 5. Verification

| Check | PR |
|-------|-----|
| `argus_test_app_preferences_smoke` | PREF-1 |
| `argus_test_asset_resolver_smoke` | PREF-1 |
| `argus_test_keyboard_bindings_smoke` | PREF-4 |
| `argus_ui` builds | all |
| Manual: Preferences Apply/Cancel/Reset | PREF-3 |
| Manual: Open recent, template, reload, search path resolve | PREF-2+3 |
| Manual: Remap ⌘S, verify menu + action | PREF-4 |

---

## 6. Non-goals

- Prefs inside bundle / presets
- User-curated template directories
- Autosave `.autosave/`
- Cloud / multi-user settings
- Per-project preference profiles

---

*Sign off → execute PREF-1.*