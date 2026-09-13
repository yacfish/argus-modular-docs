# UX-C — Clipboard, duplicate, undo, selection polish

**Branch:** `ux-c` (merged [#18](https://github.com/yacfish/argus-modular/pull/18))  
**Prerequisite:** UX-B on `main` (merged [#17](https://github.com/yacfish/argus-modular/pull/17); smoke-tested).  
**Design reference:** [plan-ui-ux.md](plan-ui-ux.md) §3.1–§3.2

---

## Goal

Daily editing workflows on the canvas: copy/cut/paste subgraph fragments, duplicate in place, undo/redo, and a small selection tweak carried over from UX-B smoke.

---

## Scope

### Clipboard (`⌘C` / `⌘X` / `⌘V`)

| Item | Detail |
|------|--------|
| **Copy** | Serialize current node selection + **internal** connections (both endpoints inside selection) to clipboard JSON |
| **Cut** | Copy, then remove selection (nodes + internal wires; prune external connections at cut boundary) |
| **Paste** | Parse clipboard; remap node ids; offset positions; append to graph; select pasted nodes |
| **Context** | Enable edit only; canvas focused; no text-field capture (`WantTextInput`) |

Clipboard format: editor subgraph fragment (same schema as `graph_json` nodes/connections subset). Stable enough for cross-session paste within the app; not a public file format v1.

### Duplicate (`⌘D` / Alt+drag)

| Item | Detail |
|------|--------|
| **Behaviour** | Same as copy+paste in one step with a fixed offset (default 40×40) |
| **Alt+drag** | Option/Alt+click-drag on selection duplicates in place at zero offset, selects clones, and drags them until mouse up; internal wires copy with the selection |
| **Sticky offset** | After Alt+drag, `⌘D` repeats using that drag’s offset until any other click or non-`⌘D` keyboard input; then revert to default |
| **Clipboard** | Does **not** read or write the shared clipboard buffer |

### Undo / redo (`⌘Z` / `⌘⇧Z`)

| Item | Detail |
|------|--------|
| **Stack** | Editor JSON snapshots or inverse commands for topology + node metadata |
| **Commands** | Add/delete nodes; move (incl. group move); connect/disconnect; paste/cut/duplicate; param changes committed from canvas/node panel |
| **Coalesce** | Group drag → one undo step; marquee-only selection changes optional skip |

Start with canvas topology edits; expand to panel param commits if stack plumbing is shared.

### Selection polish (from UX-B smoke)

| Item | Detail |
|------|--------|
| **Shift-click** | **Toggle (XOR)** membership — add if absent, **remove if already selected** |
| **Status** | Shipped — Shift and ⌘/Ctrl both toggle; marquee + Shift still additive |

### Canvas polish (shipped with #18)

| Item | Detail |
|------|--------|
| **Z-order** | Per-node draw order; only topmost widget band accepts input; params still drawn on occluded nodes (disabled) |
| **Chrome drag** | Widget-band hit test no longer blocks chrome drag when no control is hovered |
| **GM in edit** | Global monitor buttons on preview slots work while edit mode is on |
| **Edit off** | Turning off edit clears node and connection selection |
| **Debug logging** | Tagged stderr logs + DEBUG side panel (off by default); global toolbar DEBUG toggle removed |

---

## Not in UX-C

| Item | Track |
|------|-------|
| `⌘⇧V` paste replace | Future — see plan-ui-ux §3.2 |
| Align / distribute selection | After clipboard stable |
| `F` frame selection / fit all | UX-B optional; still deferred |
| Multi-select wires | Later |

---

## Implementation notes

1. **Id remap on paste** — allocate new `id`s; rewrite `connections` endpoints; preserve `type`, `params`, `ui`, `position` (+ offset).
2. **Cut boundary** — drop connections that reference removed nodes; do not delete peer nodes outside selection.
3. **Undo + live graph** — paste/cut/delete must go through the same commit path as today’s canvas edits (`apply_changes` / topology patch when LT-UI lands).
4. **Shift XOR** — small change in `graph_canvas` click handler: route Shift-click through `toggle_node_in_selection` (same as mod-click), not `add_node_to_selection`.

---

## Test plan

- [x] `⌘C` copies selection; paste in empty doc area creates clone with new ids
- [x] Internal wires copy; external wires dropped on cut; paste does not auto-wire outward
- [x] `⌘D` duplicates with offset without altering clipboard contents
- [x] Alt+drag duplicates selection and drags clones until mouse up
- [x] `⌘D` after Alt+drag uses last drag offset; resets on other click/key
- [x] Overlapping nodes draw chrome/widgets in stable z-order (selected/dragging on top)
- [x] `⌘Z` / `⌘⇧Z` undo/redo move, delete, paste, wire add/remove
- [x] Shift-click removes node from selection when already selected (XOR)
- [x] ⌘/Ctrl-click toggle unchanged; marquee + Shift still additive
- [x] Shortcuts disabled when edit off or ImGui wants keyboard focus
- [x] No regression on UX-B group move, zoom, GM, in-node controls

**Automated:** `argus_test_graph_clipboard_smoke` — fragment build/parse round-trip (passes on `main`).