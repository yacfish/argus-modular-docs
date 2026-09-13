# UX-B — Multi-select, group move, zoom

**Branch:** `ux-b` (merged [#17](https://github.com/yacfish/argus-modular/pull/17))  
**Prerequisite:** UX-A #1–#4 on `main` (merged #16).  
**Design reference:** [plan-ui-ux.md](plan-ui-ux.md) §3.1, §3.3, §6

---

## Goal

Canvas selection and navigation for daily editing: select many nodes, move them together, zoom the graph. **Duplicate (`⌘D`) is UX-C** — in-place copy-paste without touching the shared clipboard.

---

## Scope

| Item | Detail |
|------|--------|
| **Marquee** | Drag on empty canvas (enable edit) → box-select nodes |
| **Additive select** | Shift-click add (UX-B); ⌘/Ctrl-click toggle membership |
| **Group move** | Drag any selected node → all selected nodes move |
| **Group delete** | `Delete` removes full selection (nodes + selected wires) |
| **Select all** | `⌘A` / `Ctrl+A` — all nodes in current graph |
| **Zoom** | Scroll wheel on canvas; optional fit/frame (`F`) later |
| **Align / distribute** | Deferred — inspector actions after multi-select works |

## Not in UX-B

| Item | Track |
|------|-------|
| `⌘C` / `⌘X` / `⌘V` clipboard | **UX-C** |
| `⌘D` duplicate in place | **UX-C** |
| Undo / redo | **UX-C** |

---

## Test plan

- [x] Marquee selects nodes inside rect; Shift adds to selection
- [x] Group drag preserves relative positions
- [x] Delete clears multi-selection
- [x] Zoom does not break pin hit-testing or chrome widgets
- [x] Single-select and wire select still work; no regression on GM / controls

**Follow-up (UX-C):** Shift-click should **toggle (XOR)** — deselect when already selected — not add-only. See [next_on_UX-C.md](next_on_UX-C.md).