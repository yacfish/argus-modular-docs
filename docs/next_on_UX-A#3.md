# UX-A #3 — Node tab full inspector + in-canvas node chrome

**Branch:** `ux-a`  
**Order:** After #1 (shell) and #2 (palette). Before #4 (live preview pixels + global monitor).

---

## Two surfaces (do not conflate)

| Surface | Role |
|---------|------|
| **Node tab (inspector)** | Full authoring: every param, visibility, widgets, preview slots, routing |
| **Canvas node box** | Operate + glance: filtered controls and previews embedded in node chrome |

The right-panel **Parameters** section exists today. The **canvas controls band** (between name and outputs) does **not** — that is the purple rectangle in the layout sketch.

---

## Node chrome layout (top → bottom)

```
┌─────────────────────────┐
│  ● inputs               │
├─────────────────────────┤
│      node name          │  ← double-click (edit mode) → type picker (replace)
├─────────────────────────┤
│  canvas controls        │  ← #3 (this work)
├─────────────────────────┤
│  in-node preview        │  ← #4 pixels; #3 reserves band + placeholder
├─────────────────────────┤
│  outputs            ●   │
└─────────────────────────┘
```

**controls-above-preview** is the fixed order.

---

## #3 work items

### 1. In-canvas parameters (controls band)

**Today:** Canvas draws pins + name only (`node_height` = inputs + name + outputs).

**Target:**
- Controls section between name and preview band.
- Show params that pass `should_show_parameter(profile, ui.params.*)` — same rules as run `ControlPanel`.
- Widgets: toggle, slider/drag, path+Open, text (from `ui.params.widget`).
- `set_parameter` on change; graph keeps ticking.
- `node_width` / `node_height` include controls (+ preview band when configured).
- ImGui widgets in screen space over the node; clicks on widgets must not start node drag.
- **Disable edit:** canvas controls are primary operate surface (manifest `control_profile` + node `profile_override`).
- **Enable edit:** controls stay live; Node tab still shows the full list for authoring.

**Node tab vs canvas:**

| Node tab | Canvas |
|----------|--------|
| Every schema param | Visible subset only |
| Includes `visible: false` for config | Only `visible: true` (or profile default) |
| Visibility / widget / routing config | Operate only |

**Status:** Done (in-canvas controls + preview placeholder band).

### 2. Floating node picker (add + replace)

Replaces the side-panel Palette tab.

| Action | Mode | When |
|--------|------|------|
| **`⌘N`** | **Add** | Edit mode; picker at placement hint |
| **Double-click empty canvas** | **Add** | Edit mode; picker at click position |
| **Double-click canvas name** | **Replace** | Edit mode; same picker, hot-swap selected node |
| **`⌘⇧N`** | New document | Global (moved from `⌘N`) |

Picker UX: search (optional), scroll / ↑↓ to cycle catalog names (`ocv_camera`, `gl_window`, …), Enter or double-click entry to confirm, Esc dismiss.

**Replace semantics:**
- Prune connections that cannot be maintained (port index / type); one-line status e.g. `Dropped 2 connections`.
- Hot-swap under topology lock; update `type` + canonical `name`.
- Param transfer only when `manifest.allow_param_transfer_on_node_replace: true` (merge matching param keys; rest from new-type defaults).

**Status:** Done.

### 3. Preview slot fields (Node tab)

Per `ui.previews[]` slot — **authoring + JSON persistence** (#4 wires live feeds):

| Field | Values |
|-------|--------|
| **source** | `port` \| `inner` |
| **port** / **inner** | Output port name or inner key |
| **display** | `in_node` \| `global_monitor` \| `both` |
| **kind** | Signal type (`cv_mat`, `gl_texture`, …) |
| **viewer** | Presentation (`auto`, `image`, `text`, `sparkline`) |
| **max_hz** | Throttle (unchanged) |

Legacy slots with only `port` infer `source: port`, `display: in_node`, `viewer: auto`.

**Status:** Done (UI + bundle persistence). Inner feeds and live viewers are #4.

### 5. Global monitor shell

#3 = preview slot persistence only. Live viewer + **GM** assignment shipped in #4 (**Monitor** tab).

### 6. Node tab parameter polish (deferred)

- JSON/multiline editors for object params — defer until a node needs them
- `operation` on `OpenCVProcessor` — **removed** from live params at core; fixed at node add/replace via palette name (`ocv_blur`, …)

---

## #3 vs #4 split

| Piece | #3 | #4 |
|-------|----|----|
| Controls band on canvas | ✓ | — |
| Preview band layout + placeholder | ✓ shell | ✓ live frames |
| Floating picker add/replace | ✓ | — |
| Inspector: all params | partial | — |
| Preview: source, routing, viewer fields | ✓ | — |
| Global monitor panel | ✓ shell | ✓ video/data |
| **GM** on canvas | — | ✓ |
| SignalBridge → preview textures | — | ✓ 4a+ |

---

## Status

**#3 complete** (merged to `main`). **#4 complete** — merged [#16](https://github.com/yacfish/argus-modular/pull/16) — see **[next_on_UX-A#4.md](next_on_UX-A#4.md)**.

Deferred from #3: JSON param editors, inspector polish.

Multi-select batch edit is **UX-B**, not #3.