# Graph canvas file split

**Updated**: June 2026  
**Status**: **GC-4 shipped** — refactor complete; no behavior change.  
**Related:** [architecture.md](architecture.md) §5 (canvas editor)

**Problem:** `src/graph_canvas.cpp` was **~3114 lines** (one class, ~60 methods, ~50 member fields). `node_canvas_chrome.cpp` (~1174 lines) already extracted in-node chrome; the rest was monolithic.

**Goal:** Same functionality and public `GraphCanvas` API; every new/edited file **under ~1k lines**; `draw()` orchestration **under ~100 lines**. **Achieved** (GC-1–GC-4).

---

## 1. Current anatomy (post-split)

| File | ~Lines | Role |
|------|--------|------|
| `graph_canvas.cpp` | 26 | Façade: commit flags, pending picker requests |
| `graph_canvas_draw.cpp` | ~596 | Wires, nodes pass, overlays, marquee |
| `graph_canvas_input.cpp` | ~568 | Pointer, shortcuts, keyboard/focus |
| `graph_canvas_geometry.cpp` | ~332 | Pin positions, beziers, `link_endpoints` |
| `graph_canvas_hit_test.cpp` | ~210 | Node/wire bands; delegates pin helpers |
| `graph_canvas_topology.cpp` | ~340 | Sync, palette add, delete, undo restore |
| `graph_canvas_selection.cpp` | ~259 | Selection, marquee, z-order |
| `graph_canvas_clipboard.cpp` | ~135 | Copy/cut/paste/duplicate, undo/redo |
| `graph_canvas_view.cpp` | ~243 | Pan/zoom, placement hints |
| `graph_canvas_sticky.cpp` | ~255 | Sticky Alt-drag state machine |

**Testable modules** (Strategy B — GC-4):

| Module | Helpers |
|--------|---------|
| `graph_editor_context.hpp` | `GraphEditorContext` — document state bundle |
| `graph_topology` | `node_json_by_id`, `add_connection`, `add_palette_entry`, `delete_selection`, … |
| `graph_hit_test` | `pin_at`, `hover_pin` + `CanvasView` |
| `graph_clipboard` | `paste_fragment` |

---

## 2. Constraints

### Shared mutable state

Document fields live in `GraphEditorContext doc_` on `GraphCanvas`. View/input-only state (`pan_`, `zoom_`, `link_start_`, marquee, sticky) remains on `GraphCanvas`.

**Safe:** one `GraphCanvas` object; method bodies across `.cpp` files.  
**Done:** `GraphEditorContext` for testable document slice without duplicating `graph_json_`.

### Do not

- Duplicate `graph_json_` across parallel controller types
- Move canvas pointer/focus handling to `edit_mode_panel`
- Merge editor JSON into `graph_session` in this refactor

### Boundaries (unchanged)

| Module | Owns |
|--------|------|
| `node_canvas_chrome.*` | Chrome metrics, in-node controls/previews |
| `graph_clipboard.*` | Fragment serialize/parse |
| `editor_undo_stack.*` | JSON checkpoints |
| `graph_session.*` | Engine topology + bundle I/O |

---

## 3. Strategy A — Multi-`.cpp`, single class (GC-1–GC-3) ✅

Same `graph_canvas.hpp` and member layout. Valid C++: one class, methods defined across multiple translation units. **Zero public API change.**

`draw()` is ~80 lines of sequencing (`draw_wires` → `draw_nodes_pass` → pointer/drag handlers).

---

## 4. Strategy B — `GraphEditorContext` (GC-4) ✅

```cpp
struct GraphEditorContext {
    nlohmann::json graph_json;
    EditorUndoStack undo_stack;
    // selection, placement, committed/dirty flags, status
};
```

`GraphCanvas` holds `GraphEditorContext doc_` plus view/input-only state. Free functions:

- `graph_topology::add_connection`, `add_palette_entry`, `delete_selection`, …
- `graph_hit_test::pin_at`, `hover_pin`
- `graph_clipboard::paste_fragment`

---

## 5. Phased delivery

| Phase | Work | Status |
|-------|------|--------|
| **GC-1** | Extract `draw()` subroutines | ✅ Shipped (PR #22) |
| **GC-2** | Multi-`.cpp` — geometry, hit_test, selection | ✅ Shipped (PR #23) |
| **GC-3** | topology + clipboard + input + sticky files | ✅ Shipped (PR #24) |
| **GC-4** | `GraphEditorContext` + headless smoke tests | ✅ Shipped |

**Call sites unchanged:** `edit_mode_panel`, `floating_node_picker`, `node_panel` use `GraphCanvas` only.

---

## 6. Verification

| Check | Covers |
|-------|--------|
| Manual smoke | Wire, delete, palette add, duplicate, undo/redo, paste, pan/zoom, marquee |
| `argus_test_graph_clipboard_smoke` | Clipboard serialization |
| `argus_test_graph_topology_smoke` | JSON topology helpers (headless) |
| `argus_test_graph_editor_helpers_smoke` | `pin_at`, `hover_pin`, `paste_fragment`, `add_palette_entry`, `delete_selection` |
| `argus_test_bypass_ui_smoke` | Control visibility / chrome integration |

---

## 7. Acceptance criteria

1. All manual smoke scenarios identical before/after each phase. ✅
2. No `GraphCanvas` public API removals. ✅
3. `src/graph_canvas.cpp` **< 700 lines**; no new source file **> 1000 lines**. ✅
4. `cmake --build build --target argus_ui` and smoke tests green. ✅

---

## 8. Non-goals

- imgui-node-editor migration
- Reactive `topology_changed` resync (see [plan-live-topology.md](plan-live-topology.md))
- Deleting sticky Alt-drag behavior (only isolated to `graph_canvas_sticky.cpp`)
- Performance profiling / draw-list optimization

**Optional later:** extract `hit_test_node` / `hit_test_link` helpers; more headless coverage for replace-node and undo restore.