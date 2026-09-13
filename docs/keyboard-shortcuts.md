# Keyboard shortcuts — argus-modular

**Updated:** June 2026

On **macOS**, chords use **⌘ (Command)**. On **Windows/Linux**, the same actions use **Ctrl**. ImGui maps these via `ImGuiMod_Ctrl` + `ConfigMacOSXBehaviors` on Apple (`keyboard_shortcuts.hpp`).

**Remapping:** **Argus → Preferences → UX** lists every chord action below. Overrides persist in `preferences.json` under `ux.bindings` and drive both menus and runtime shortcuts (`keyboard_bindings.*`).

---

## Argus

| Action | Mac | Win/Linux |
|--------|-----|-----------|
| Preferences… | ⌘, | Ctrl+, |

---

## File

| Action | Mac | Win/Linux |
|--------|-----|-----------|
| New document | ⌘⇧N | Ctrl+Shift+N |
| Open bundle… | ⌘O | Ctrl+O |
| Save bundle | ⌘S | Ctrl+S |
| Save bundle as… | ⌘⇧S | Ctrl+Shift+S |
| Reload from disk | ⌘R | Ctrl+R |
| Open recent ▸ | — *(hover submenu)* | — |
| Open template ▸ | — *(hover submenu)* | — |

New/open/reload prompt to save when the document is dirty. See [plan-app-preferences.md](plan-app-preferences.md).

---

## Mode

| Action | Mac | Win/Linux |
|--------|-----|-----------|
| Toggle enable / disable edit | ⌘E | Ctrl+E |
| Toggle edit (empty canvas) | ⌘+click empty space | Ctrl+click empty space |
| Edit toggle button | `[EDIT]` — top-right of menu bar | same |

---

## Side panel

| Action | Shortcut |
|--------|----------|
| Cycle tabs (Monitor → Node → Presets → hidden) | `Tab` |

Debug settings move to **Argus → Preferences** (side Debug tab removed — [plan-app-preferences.md](plan-app-preferences.md)).

---

## Canvas (enable edit only)

| Action | Mac | Win/Linux |
|--------|-----|-----------|
| Add node (floating picker) | ⌘N | Ctrl+N |
| Add node | Double-click empty canvas | same |
| Replace node type | Double-click node name | same |
| Select all nodes | ⌘A | Ctrl+A |
| Copy selection | ⌘C | Ctrl+C |
| Cut selection | ⌘X | Ctrl+X |
| Paste selection | ⌘V | Ctrl+V |
| Duplicate selection | ⌘D | Ctrl+D |
| Duplicate while dragging | Alt+drag node | same |
| Undo | ⌘Z | Ctrl+Z |
| Redo | ⌘⇧Z | Ctrl+Shift+Z |
| Delete selection | `Delete` or `Backspace` | same |
| Marquee multi-select | Drag on empty canvas | same |
| Toggle node in marquee | Shift+drag marquee | same |
| Toggle node in selection | ⌘+click or Shift+click node | Ctrl+click or Shift+click |
| Pan | Middle-mouse drag | same |
| Zoom | Scroll wheel | same |

---

## Floating node picker

| Action | Shortcut |
|--------|----------|
| Move highlight | Up / Down |
| Place / confirm | Enter |
| Close | Esc |

---

## Not implemented

| Action | Planned chord |
|--------|----------------|
| Paste replace | ⌘⇧V / Ctrl+Shift+V |
| Frame selection / fit all | `F` |
| Clear selection / cancel wire drag | `Esc` (canvas) |

See [roadmap.md](roadmap.md) deferred backlog.