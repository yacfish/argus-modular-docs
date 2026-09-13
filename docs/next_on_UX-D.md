# UX-D — Menu bar edit control + canvas/panel polish

**Branch:** `ux-d`  
**Prerequisite:** UX-C on `main` (merged [#18](https://github.com/yacfish/argus-modular/pull/18)).

---

## Goal

Remove canvas chrome clutter; keep a single prominent **edit toggle** on the menu bar. Flush graph layout, readable side panel, and live marquee feedback.

---

## Scope

### Add

- **`[EDIT]`** button — top-right of the menu bar (same row as File / Mode)
- Same red-on / gray-off styling as the old canvas toolbar button
- Clears canvas selection when turning edit off; syncs editor when turning on
- Side panel background — slightly darker than locked (run-mode) canvas `rgb(42,45,51)` → panel `rgb(36,39,44)` at 86% brightness
- Side panel inner padding — **10px** left/right, **5px** top/bottom (content **280px** wide in a **300px** column)
- Live marquee selection — nodes select/unselect as the box moves (not only on mouse-up)
- Marquee drag border colors — base selection `rgb(255,160,100)`; while dragging marquee `rgb(255,200,100)`

### Remove

- Canvas top row: `[EDIT]`, `[FPS]`, shortcut hint text
- Bottom status bar (tried and dropped — bundle name in title, selection visible on canvas, FPS in Debug tab)
- Debug tab section titles and helper copy (`Diagnostics`, `Debug tags`, stderr toggle blurb)

### Move / rename

- **FPS logging** toggle → **Debug** side tab only (stderr when enabled); label **`fps`**, rate hint in tooltip
- Tab label **`DEBUG`** → **`Debug`**

### Polish

- Graph area: zero window padding / child border (flush to menu bar and panel edge)
- Canvas background +50% brighter for contrast with menu chrome (`rgb(42,45,51)` run mode)
- Canvas black border fix — style/padding before root `Begin`, full-viewport background fill, `glClear`, edge-to-edge child paint
- Side panel layout fixes (see **Layout notes** below)
- **Number scrub** — post-release cursor restore + ImGui mouse sync so fields stay hovered (re-scrub without nudging)
- **Wire curves** — control offset scales with port proximity (see **Wire drawing** below)

### Unchanged

- Mode menu, `⌘E`, Cmd-click empty canvas for edit toggle
- Right panel tabs, floating picker
- Shift+marquee toggle semantics (preview matches release; base selection captured at drag start)

---

## Layout notes (ImGui)

Documented so we do not regress panel width or padding.

| Issue | Cause | Fix |
|-------|--------|-----|
| Black ring around canvas | Default 8px `WindowPadding` applied after `Begin`; child bg inset | Push padding 0 + canvas-colored bg **before** root `Begin`; `BackgroundDrawList` + `glClear`; paint canvas child full rect |
| Panel only ~290px wide | `SameLine()` default **8px** `ItemSpacing` between canvas and panel | `SameLine(0, 0)`; canvas child width `ImVec2(-kRightPanelWidth, 0)` to reserve 300px |
| Right padding invisible | ImGui clears `WindowPadding` on borderless children unless flagged | `ImGuiChildFlags_AlwaysUseWindowPadding` on `edit_right_inner` |
| Symmetric 5px padding “lost” | Same as above — padding push ignored without flag | Use `AlwaysUseWindowPadding`; separate **X=10** / **Y=5** constants |

**Files:** `src/app.cpp`, `src/edit_mode_panel.cpp`, `src/graph_canvas.cpp`, `src/editor_log.cpp`

---

## Number scrub — post-release hover

After mouse-up from a canvas/panel number scrub, GLFW relative capture leaves ImGui with a stale mouse position until the pointer moves. Without a sync step the field is not hovered and cannot be clicked again immediately.

| Piece | Role |
|-------|------|
| `end_scrub_session()` | Immediate `restore_scrub_anchor()` on release; queue GLFW warp for next frame |
| `inject_scrub_mouse_events()` | `AddMousePosEvent(anchor)` after `ImGui_ImplGlfw_NewFrame()`, before `ImGui::NewFrame()` |
| `lock_interaction_for_frame()` | 2-frame post-scrub: `TeleportMousePos` + pin `HoveredId` on released field |
| `float_field()` | Hover/cursor via `IsMouseHoveringRect` after restore |

**Frame order (`app.cpp`):** `glfwPollEvents` → `flush_pending_restore` → `ImGui_ImplGlfw_NewFrame` → `inject_scrub_mouse_events` → `ImGui::NewFrame` → `lock_interaction_for_frame`

**Files:** `src/param_digit_drag.cpp`, `src/param_digit_drag.hpp`, `src/app.cpp`

---

## Wire drawing

Connections are cubic Bézier cords in screen space (`ImDrawList::AddBezierCubic`).

| Function | Role |
|----------|------|
| `link_endpoints()` | Output pin `p0`, input pin `p1` (pan/zoom + canvas origin) |
| `wire_control_offset()` | Tangent length from port proximity |
| `wire_bezier_control_points()` | `c1 = (p0.x, p0.y + off)`, `c2 = (p1.x, p1.y − off)` |
| `GraphCanvas::draw()` | Render committed connections |
| `hit_test_link()` | Same geometry, 12px tolerance (`dist_point_bezier`) |
| `link_center_graph()` | Midpoint at `t = 0.5` for paste/placement |

**Control offset** — fixed ±50px tangents loop when nodes are very close. Offset now ramps with the tighter axis span:

```
proximity = min(|Δx|, |Δy|)   // between p0 and p1

proximity ≥ 45  →  off = 50   (unchanged from legacy)
proximity = 0   →  off = 5
else            →  off = 5 + 45 × (proximity / 45)
```

Used for draw, hit-test, placement midpoint, and in-progress drag wire (replaces the old 40px drag constant).

**Files:** `src/graph_canvas.cpp`

---

## Marquee selection

- **`apply_marquee_selection()`** runs each frame while marquee is active, **before** nodes draw (same-frame highlight)
- **Shift+drag:** toggles nodes in box relative to selection at marquee start (`marquee_base_selection_`)
- **Border colors:** `kSelectionBorderColor` when selected; `kMarqueeSelectionBorderColor` while `marquee_active_`

---

## Not in UX-D

Shell rework, docking, transport strip, presets polish — all dropped or deferred.

---

## Test plan

- [x] No toolbar or hint above canvas
- [x] `[EDIT]` on menu bar top-right toggles edit
- [x] FPS logging only in Debug tab (`fps` + tooltip)
- [x] Canvas flush to menu bar and window edges (no black border)
- [x] Side panel 300px column, darker bg, 10/5px inner padding on all tabs
- [x] Marquee selects nodes during drag; gold/lighter-gold borders
- [x] Number scrub: re-scrub immediately after release (no mouse nudge)
- [x] Wire curves: no loop when stacked nodes &lt; 45px apart; unchanged at ≥ 45px
- [x] Mode menu / `⌘E` / shortcuts unchanged (manual)
- [x] Build + smoke tests pass

---

## Changelog (`ux-d`, after canvas border fix `0630d4f`)

| Commit | Summary |
|--------|---------|
| `c901f0b` | Side panel background + inner child |
| `a8c552d` | Debug tab simplified; padding experiments |
| `489a9c4` / `340d851` | `AlwaysUseWindowPadding`; full 300px panel (`SameLine(0,0)`, negative canvas width) |
| `d08fea6` | Panel padding 10px horizontal, 5px vertical |
| `f85267c` | Live marquee selection during drag |
| `807e20f` | Selection borders: gold base, lighter gold while marquee active |

(Marquee border color iterations `fc1eec6`…`6a1f3ae` superseded by `807e20f`.)

| wire/scrub polish | Number scrub post-release hover sync; wire offset 50→5 as `min(|Δx|,|Δy|)` goes 45→0 |