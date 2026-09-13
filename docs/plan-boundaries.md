# Boundaries — SubgraphNode & CodeBox

**Track:** modular boundary nodes (not a UX polish phase)  
**Status:** **Done** — core **done** (C1–C6); app **B1–B5 done** on `main` (B4 [#38](https://github.com/yacfish/argus-modular/pull/38); B5 `c6453e1…` parent **↑** exposure)
**Core pin:** `ARGUS_CORE_REF` ≥ `398725154303881d84dd06efe7a5fcc0cb3188ff` (`invoke_ui_button`)  
**Prerequisites:** UX-D shipped ([#19](https://github.com/yacfish/argus-modular/pull/19)); [plan-graph-canvas-split.md](plan-graph-canvas-split.md) shipped (GC-1–GC-4).

**Layer tags:** `[app]` = argus-modular UI/session only · `[core]` = argus-core only · `[core&app]` = both repos / coordinated PRs

---

## Goal [core&app]

Make **SubgraphNode** and **CodeBox** first-class authoring targets — not just palette entries with a path field. Both are **boundary nodes**: implementation in bundle files (`graphs/…` vs `scripts/…`) with **embedded vs load-from-disk** creation. Subgraph authoring uses **canvas tabs**; CodeBox uses **external editor** + script-declared ports (see [plan-codebox-ux.md](plan-codebox-ux.md)).

**Core:** node types, bundle `graphs/` + `scripts/` I/O, `SubgraphNode::inner_graph()` sync, Inlet type inference, CodeBox port inference from Lua, script file watch, `visible_in_parent`.  
**App:** shared creation panel (single panel, type-specific copy), canvas tab bar (subgraph), script-open chrome (CodeBox), disk-reference rules, save/dirty glue.

---

## Why together [core&app]

| Concern | SubgraphNode | CodeBox |
|---------|--------------|---------|
| Bundle file param | Embedded: `graphs/embedded_N.json`; disk: `bundle_ref` | Embedded: `scripts/embedded_N.lua`; disk: `script_ref` (`.lua`) |
| Inspector (Node tab) | Embedded path read-only; **Open original** (disk); parent **↑** | Script section; dynamic port map; open/reveal; editable `script_ref` |
| “Go inside” | Canvas tab + title-bar **↓** | Title-bar script-open → external editor |
| Boundary ports | Inlet/Outlet inside embedded subgraph tabs | `inlets` / `outlets` in Lua — inferred at load (B4) |
| Bypass | Hidden v1 | Hidden v1 |

Shared work: bundle-relative path editing, validation, create-if-missing stubs, Node-tab actions, and consistent canvas chrome (title-bar action buttons per boundary type).

---

## Architecture — shared base, subgraph first [core&app]

Subgraph and CodeBox share one **boundary authoring module** in the app layer. Core node types stay separate (`SubgraphNode`, `CodeBox`); the shared piece is UI + session glue for bundle-relative files. Expect **core PRs** alongside app work (bundle file APIs, inner-graph load/export, Inlet type inference, script reload, `visible_in_parent`).

### Proposed shape [app]

```
BoundaryAuthoring (shared)
├── bundle-relative path handling (embedded paths in bundle; disk = bundle / file picker)
├── exists check + create stub (embedded)
├── creation option panel (embedded | load from disk) — **single panel**, Subgraph + CodeBox
├── Node-tab actions (Open / Reveal / Open original as applicable)
└── canvas chrome hooks (title-bar ↓ subgraph · script-open CodeBox)

SubgraphBoundary (specialization)
├── canvas tab bar (main + embedded_N + disk bundle names)
├── tab lifecycle (× close hides tab; subgraph keeps running; ↓ re-opens)
├── embedded vs disk-reference edit rules
├── Inlet/Outlet inner-tab-only authoring
└── visible_in_parent ↑ affordance in node panel tables

ScriptBoundary (specialization)
├── `script_mode` embedded | file_ref
├── external editor + Reveal
├── `ui` table → inspector params / buttons / inner previews
├── dynamic port map + reload status (see plan-codebox-ux)
└── core file-watch + port re-inference (no canvas tabs)
```

### Core touchpoints [core]

| Area | Likely core work |
|------|------------------|
| Bundle I/O | Read/write `graphs/*.json`, `scripts/*.lua` on save/load; embedded graph storage in current bundle |
| SubgraphNode | Load inner graph JSON into `inner_graph()`; reconfigure on inner edit; disk-ref bundle attach |
| Inlet/Outlet | Boundary node types + port relay to parent `SubgraphNode` |
| Inlet type inference | First wire (inside or outside subgraph) sets Inlet type; block incompatible second wire; indeterminate when disconnected |
| Parent exposure | `visible_in_parent` on params/viewers; surfaced on parent `SubgraphNode` |
| CodeBox | Port inference from Lua; file watch + hot reload; `script_ref` / embedded paths |

**Sequencing:** implement **Subgraph first (B1 → B5)**. Extract shared `BoundaryAuthoring` helpers while wiring embedded subgraphs. CodeBox (B4) plugs into the same base — not a second path-picker implementation.

**Inheritance vs composition:** prefer a small shared helper module + type dispatch (`is_boundary_node(type)`) unless tab/session state forces a class hierarchy. One parent module owns common methods; children add tab navigation, disk-reference rules, or script open behavior.

**Not in shared base:** canvas tab bar, disk-reference edit lock (subgraph inner tab), `visible_in_parent` — Subgraph-only. Script-open + port map — CodeBox-only. **Shared:** creation panel shell, embedded vs disk choice pattern, external path rebind (`bundle_ref` / `script_ref`).

---

## UX / affordances (decided) [app]

### Global (all graphs)

| Action | Behavior |
|--------|----------|
| **Double-click node name** | **Replace type** (hotswap) — unchanged from UX-D. Does **not** open a subgraph tab. |

### Subgraph creation [app]

1. User adds a **SubgraphNode** (palette / picker).
2. A **small option panel** opens immediately.
3. User chooses:
   - **Embedded** — new inline subgraph in the **current** bundle.
   - **Load from disk** — **Open bundle** dialog; binds an external bundle reference.

### Canvas tabs [app]

When at least one subgraph tab exists (embedded or opened via **↓** on a disk ref), the graph canvas shows a **tab bar at the top**:

| Tab | Rules |
|-----|--------|
| **main** | Always first tab; the root graph of the current document. **No** close button. |
| **embedded_N** | One tab per embedded subgraph (`embedded_0`, `embedded_1`, …). Auto-created and **auto-selected** on first embed. |
| **Disk bundle name** | Tab label = external bundle name; created only when user presses **↓** on a disk-loaded `SubgraphNode`. |

| Action | Behavior |
|--------|----------|
| **Select tab** | Switches active graph session on the same canvas editor. Stashes pan/zoom/selection/undo for the outgoing tab; always reloads the incoming tab’s stashed editor (even when re-selecting an already-active tab after a fix). |
| **Close tab (×)** | Small **circular ×** (6px radius) on embedded and disk tabs only — **not** on **main**. Closing hides the tab; the subgraph **keeps running** in the background. |
| **Re-enter closed subgraph** | Title-bar **↓** on the `SubgraphNode` — re-opens / selects its tab. **↓** also **selects that node exclusively** (clears other selection). |
| **Tab order** | **main** always first. Subgraph tabs append **rightmost** on first open; revisiting a tab does **not** reorder it. |
| **When tabs open** | **Embedded** creation (creation panel commit) and **↓** drill only. **Not** opened by duplicate, paste, or undo restore — user drills in when ready. |

### Title-bar drill button [app]

Small **circular button**, **arrow down (↓)**, **right-aligned in the node title bar** (name row in `node_canvas_chrome`).

| Subgraph kind | **↓** behavior |
|---------------|----------------|
| **Embedded** | Opens or selects the `embedded_N` tab (empty graph on first embed). |
| **Load from disk** | Creates the disk tab (bundle name) if not open; selects it. **Grayed** for nested `SubgraphNode`s inside a disk-reference tab. |

### Embedded subgraph workflow [core&app]

1. User picks **Embedded** in the creation panel.
2. Canvas tab bar appears: **main**, **embedded_0** (auto-selected).
3. **embedded_0** graph is **empty**; user adds Inlet/Outlet and wires internal nodes between them.
4. User can create more embedded subgraphs → **embedded_1**, … each with its own tab.
5. Inlets/Outlets are addable **only** inside an embedded (or editable) subgraph tab — not from **main** picker/palette.

### RAM-only embedded until save [app]

Before the first explicit **Save** / **Save as**, embedded subgraphs live **in memory only**:

| Param | Role |
|-------|------|
| `subgraph_json` | Inline inner graph JSON (authoring source of truth while unsaved) |
| `subgraph_pending_file` | Reserved bundle path (`graphs/embedded_N.json`) for tab label + eventual disk write — **no file created yet** |
| `subgraph_file` | Absent until save; set by `materialize_embedded_subgraphs_for_save()` |

Create, duplicate, and paste **fork** embedded subgraphs in RAM (`fork_embedded_subgraph_params` copies live inner JSON + allocates next `embedded_N` pending path). **No disk write** on those operations.

On explicit save, `materialize_embedded_subgraphs_for_save()` hot-swaps each RAM-only `SubgraphNode` to `subgraph_file`, writes inner JSON into the bundle, and strips `subgraph_json` / `subgraph_pending_file`. Open tabs are preserved via `sync_main_editor_from_session()` after save (main editor snapshot + per-tab stash, not a full tab reset).

Undo checkpoints on the **main** tab call `sync_ram_embedded_params_in_editor()` first so delete → undo restores embedded inner content. `reconcile_tabs_after_topology_restore()` refreshes subgraph tab editors and reloads the active tab after undo/redo.

### Inlet type inference [core]

- **First connection** to an Inlet (wire from inside *or* outside the subgraph) **sets** the Inlet’s type.
- A **second** connection attempt to an incompatible port is **blocked**.
- When **no** inside or outside connection remains, the Inlet returns to **indeterminate**.

### Load from disk workflow [core&app]

1. User picks **Load from disk** → **Open bundle** dialog.
2. Selected bundle’s **main graph** is loaded and bound to the `SubgraphNode`, but **no canvas tab is created yet**.
3. User presses **↓** on the node → tab appears labeled with the **bundle name**; user is taken inside.
4. **Inside a disk-reference tab:**
   - Parameters and buttons are **interactive** (live tweak while running).
   - **Edit mode cannot be enabled** — no add/wire/delete topology changes.
   - Nested `SubgraphNode`s **execute** but their **↓** is **grayed out** (not drillable).
5. To **modify** the external bundle: open it as its own document (**⌘O**) or **Open original** in the `SubgraphNode` Node tab — then enter subgraphs the same way as embedded (full edit).

### Parent exposure [core&app]

Inside an embedded subgraph, the user can expose inner node params and viewers to the parent graph:

| Item | Behavior |
|------|----------|
| **`visible_in_parent` flag** | Per parameter and per viewer in that node’s Node-tab table. |
| **Affordance** | Tiny **circular ↑** button, right-aligned in the param/viewer name row — minimal chrome, no extra columns. |
| **Parent graph** | Exposed controls/viewers appear on the parent `SubgraphNode` (or its Node tab) for control/monitor from **main**. |

### CodeBox [app]

Authoring UX: **[plan-codebox-ux.md](plan-codebox-ux.md)** · Lua API: **[codebox-lua-api.md](codebox-lua-api.md)**. Summary: creation panel (embedded \| file ref), script-open chrome, dynamic port map, hot-reload status, pale-pink error tint, optional orphan `scripts/embedded_*.lua` GC.

---

## Scope [core&app]

### 1. Shared — boundary file params (B1) [core&app]

| Item | Layer | Notes |
|------|-------|--------|
| Subgraph creation panel | app | **Embedded** vs **Load from disk** on new `SubgraphNode` |
| Embedded path / storage | core&app | Inline `graphs/…json` in current bundle; sensible default on embed |
| Disk bundle bind | core&app | Open-bundle dialog; load external main graph; defer tab until **↓** |
| Bundle-relative path (CodeBox) | app | B1: `script` path widget (partial). **B4:** `script_mode`, `embedded_N`, `script_ref` — [plan-codebox-ux.md](plan-codebox-ux.md) |
| Exists / create stub | core&app | B1: optional `scripts/main.lua` stub button. **B4:** `scripts/embedded_N.lua` + port-declaration stub at embed |
| Dirty + save | core&app | Embedded graph edits mark current bundle dirty; explicit save materializes RAM embeds → `graphs/*.json` |
| **Open original** | app | Node tab button on disk-ref `SubgraphNode` → opens external bundle as document |
| RAM embedded params | app | `subgraph_json` + `subgraph_pending_file` until save; `sync_ram_embedded_params_in_editor` before main-tab undo |

**Status:** **Done** (`feature/boundaries-b2`).

### 2. SubgraphNode — tabs & navigation (B2) [core&app]

| Item | Layer | Notes |
|------|-------|--------|
| Canvas tab bar | app | **main** + `embedded_N` + disk bundle name tabs at top of graph canvas |
| Tab close (×) | app | Circular × on non-main tabs; close hides tab only — subgraph keeps running |
| Title-bar **↓** | app | Re-open / select tab; grayed for nested subgraphs inside disk-reference tabs |
| Multi-embedded | app | User can create many embedded subgraphs; each gets `embedded_N` tab |
| Disk-reference edit lock | app | No edit mode inside disk tab; params/buttons still live |
| Inner graph session | core&app | One active graph per tab; core syncs `SubgraphNode::inner_graph()` per embedded path |
| Tab registry key | app | One tab per `SubgraphNode` **graph id** (not `embedded_N` path) |
| Tab title (v1) | app | Filename stem: `embedded_N`, or disk bundle folder name — same chrome for all kinds until B2 styling |
| Duplicate / paste fork | app | Embedded: fork inner JSON in RAM + new `subgraph_pending_file` — **no tab**, **no disk**; disk ref: same `bundle_ref`, tab on **↓** only |
| Per-tab editor isolation | app | Stash pan/zoom/selection/undo per tab; `commit_tab_state` on switch; inner edits via `commit_subgraph_editor` |
| Save tab preservation | app | `refresh_after_save` → `sync_main_editor_from_session` updates main stash without resetting open tabs |
| Display label (deferred) | app | User-chosen tab label decoupled from on-disk `embedded_N`; bundle filenames unchanged on relabel |

**Status:** **Done** (`feature/boundaries-b2`) — tab bar, drill, RAM embeds, isolation, undo/save glue.

### 3. Subgraph inlets/outlets (B3) [core&app]

| Item | Layer | Notes |
|------|-------|--------|
| Inlet/Outlet authoring | core&app | Add only inside embedded (editable) subgraph tabs — hidden at **main** picker/palette |
| Port relay | core | Parent `SubgraphNode` ports reflect inner Inlet/Outlet |
| Inlet type inference | core | First wire sets type; incompatible second wire blocked; indeterminate when disconnected |

**Status:** **Done** (`feature/boundaries-b3`) — inner-tab-only Inlet/Outlet palette; boundary index allocation; parent port refresh on commit; multi-tab + main-graph save/sync fixes (no main↔inner id position bleed).

### 4. CodeBox — script authoring (B4) [core&app]

**UX spec:** [plan-codebox-ux.md](plan-codebox-ux.md) · **Lua API:** [codebox-lua-api.md](codebox-lua-api.md)

First CodeBox authoring iteration — scripts declare `inlets` / `outlets`; ports inferred at load (no image outlet v1). Replaces core fixed-port prototype — see [plan-codebox-ux.md](plan-codebox-ux.md).

| Item | Layer | Notes |
|------|-------|--------|
| Port inference | **core** | `inlets`/`outlets`; `inlet_N`; `outlet(i, dict)`; **`ctl` last**; mandatory `enable` |
| Script `ui` table | **core&app** | `on_param_change`; `publish_preview`; buttons; inner previews (`numbers`/`dict`/`text`); `sample_inner_preview` |
| Creation panel | app | **Single panel** with Subgraph; CodeBox embedded \| `.lua` file ref |
| Embedded scripts | core&app | `scripts/embedded_N.lua` at create; stub declares ports + `ui` |
| File ref + rebind | core&app | Editable `script_ref`; picker + ctl reload |
| Node tab | app | Dynamic port map + inspector from `ui` — plan-codebox-ux §5 |
| Open script / Reveal | app | External editor + Finder/folder |
| Hot reload + status | core&app | Last-good on failure; pale-pink tint; prune bad wires; `script_reloaded` |
| Disk subgraph rebind | app | Editable `bundle_ref` — plan-codebox-ux §11.1 |
| Orphan embedded GC | app | Optional on bundle close/quit; main + `graphs/embedded_*.json` |

**Status:** **Done** (`main`, PR [#35](https://github.com/yacfish/argus-modular/pull/35)–[#38](https://github.com/yacfish/argus-modular/pull/38)) — creation panel, port inference, script chrome, `ui` previews/buttons, rebind, embedded-script GC, fork on duplicate/paste.

**Out of scope for CodeBox:** integrated Lua IDE; image outlet / pixel API from Lua.

### 5. Parent exposure (B5) [core&app]

| Item | Layer | Notes |
|------|-------|--------|
| `visible_in_parent` flag | core | Persist per param/viewer on inner nodes |
| **↑** affordance | app | Tiny circular up-arrow in Node-tab param/viewer name row |
| Parent surfacing | core&app | Exposed controls/viewers available on parent `SubgraphNode` from **main** |

**Status:** **Done** on `main` (`c6453e1…`) — core C5 (`parent_exposure_entries`, `set_exposed_parameter`, `sample_exposed_preview`); app **↑** in inner-tab Node panel; exposed params on parent `SubgraphNode` canvas chrome + Node tab; exposed previews listed on parent Node tab (read-only).

---

## Implementation order [core&app]

1. **B1 — Shared boundary + creation panel** `[core&app]` — embedded vs disk choice, bundle I/O, CodeBox path stub  
2. **B2 — Canvas tabs** `[core&app]` — tab bar, embedded workflow, **↓** / **×**, disk-reference tab + edit lock + gray nested drill  
3. **B3 — Inlets/outlets + type inference** `[core&app]` — inner-tab-only add, first-wire-wins typing  
4. **B4 — CodeBox workflow** `[core&app]` — [plan-codebox-ux.md](plan-codebox-ux.md): port inference, creation panel, script authoring loop
5. **B5 — Parent exposure** `[core&app]` — `visible_in_parent`, **↑** in node panel, parent surfacing  
6. **AssetRef (AR1)** — after B4–B5; before GPU — [plan-asset-ref.md](plan-asset-ref.md) (unified `bundle` / `external` path params; B4 ships legacy `script_mode` + `script` / `script_ref`)

Refine slices after B1 spike (`GraphSession` likely needs a **tab registry** mapping `SubgraphNode` → graph session + tab label; core may need disk-ref attach API).

**Branching:** expect paired PRs — `argus-core` + `argus-modular` — or core-first when app depends on new APIs.

---

## Open questions [core&app]

| Topic | Layer | Question |
|-------|-------|----------|
| Embedded tab naming | app | **Decided (B2 v1):** tab registry key = `SubgraphNode` graph id; tab **title** = bundle-relative filename stem (`embedded_N` from `graphs/embedded_N.json`, or disk bundle folder name). **Deferred:** user **display label** independent of on-disk `embedded_N` (rename label in UI without moving bundle file). Duplicate **labels** allowed for disk-import tabs (same bundle name); duplicate **node display names** allowed; on-disk `embedded_N` paths stay unique (`max+1` on fork). |
| Tab keyboard shortcuts | app | Shortcut to switch tabs? Esc → **main**? |
| Executor scope | core&app | Whole document always runs, including closed tabs? (assumed yes) |
| Undo scope | app | **Partial (B2):** per-tab undo stacks for inner edits; main-tab topology undo (delete/cut/paste/duplicate) snapshots main graph with RAM embeds synced in. Undo restore calls `reconcile_tabs_after_topology_restore`. Global cross-tab undo TBD. |
| Preview / monitor | core&app | Same rules inside subgraph tabs as **main**? Exposed viewers on parent? |
| Untitled bundles | core&app | **Decided (B2):** inner graphs in `subgraph_json` (RAM); `subgraph_pending_file` reserves `graphs/embedded_N.json` for label + save materialization — no disk until explicit save |
| Inner-graph sync | core | Full reconfigure vs incremental `inner_graph()` on tab edit? |
| Parent exposure UI | app | **Decided (B5 v1):** exposed **params** on parent canvas chrome **and** Node tab; exposed **previews** on parent Node tab only (read-only table). Canvas image viewers for exposed previews deferred. |
| PR granularity | core&app | One branch per B-slice vs stacked PRs? Core pin bump cadence? |

---

## Already done (do not re-plan) [core&app]

| Item | Layer |
|------|-------|
| `SubgraphNode` + `CodeBox` in catalog (P0 palette) with default params — UX-A #2 | core |
| Path widget for `subgraph_file` / `script` in Node tab and in-node controls — UX-A #3 | app |
| Bypass hidden on both — [plan-bypass-eligibility.md](plan-bypass-eligibility.md) | app (+ core no-op) |
| Clipboard subgraph fragments (`argus-subgraph-v1`) — UX-C (parent graph only today) | app |
| Floating node picker, CPU preview viewers — UX-A #3–#4 | app |
| Double-click name → hotswap — UX-D | app |
| Creation panel (embedded / load from disk) — B1 | app |
| `boundary_authoring` module (RAM embeds, save materialize, tab labels) — B1 | app |
| Canvas tab bar + **↓** / **×** + per-tab editor isolation — B2 | app |
| RAM-only embedded until explicit save — B2 | app |
| Disk-reference drill lock (gray nested **↓**) — B2 partial | app |
| Inlet/Outlet inner-tab-only palette + boundary index — B3 | app |
| Parent `SubgraphNode` port relay on inner commit — B3 | core&app |
| Multi-tab / main-graph embedded save + position sync — B3 | app |
| CodeBox B4 — creation panel, port inference, script chrome, `ui` previews/buttons, rebind, GC — B4 | app |
| Embedded CodeBox fork on duplicate/paste; bundle path required for embedded create — B4 | app |
| Parent exposure — `visible_in_parent`, **↑** affordance, parent param surfacing — B5 | core&app |

Boundaries track **B1–B5** is **done** on `main`.

---

## Deferred (after boundaries — see GPU pipeline) [app]

| Item | Track |
|------|-------|
| GPU path + `gl_viewer` + GLSL shader lib palette | [plan-gpu-pipeline.md](plan-gpu-pipeline.md) GP1–GP3 |
| Advanced data viewers (inner feeds, monitor parity) | Backlog after GPU pipeline |

---

## Out of scope

- Live topology (LT-UI) — [plan-live-topology.md](plan-live-topology.md)
- GPU shader path — [plan-gpu-pipeline.md](plan-gpu-pipeline.md)
- Timeline / automation — [roadmap.md](roadmap.md) #15
- `⌘⇧V` paste replace (deferred from UX-C)
- Docking, shell rework
- Integrated script IDE
- Breadcrumb / navigation stack (superseded by canvas tabs)
- Editing disk-reference bundles in-place (must **Open original** or **⌘O**)

---

## Test plan [core&app]

- [x] Add `SubgraphNode` → creation panel → **Embedded** → tab bar shows **main** + **embedded_0** (selected, empty)
- [x] Add Inlet/Outlet inside **embedded_0** — not available from **main** picker (B3)
- [x] First Inlet wire sets type; incompatible second wire rejected; indeterminate when all wires removed (core `argus_test_subgraph_boundaries` / `argus_test_inlet_outlet_types`)
- [x] Close **embedded_0** tab (×) → tab hidden, subgraph still runs; **↓** on node re-opens tab
- [x] Second embedded subgraph → **embedded_1** tab; multiple tabs coexist
- [x] Tab order: new tabs append right; revisiting does not reorder
- [x] Duplicate / paste embedded subgraph → new node in RAM — **no** auto tab
- [x] Delete embedded subgraph → undo restores node **and** inner `subgraph_json`
- [x] Switch tabs → pan/zoom/selection isolated; return to **main** without position bleed
- [x] Save → materializes `graphs/embedded_N.json`; open tabs preserved
- [x] Double-click subgraph **name** → hotswap only — does **not** open tab
- [x] **↓** drill → selects that `SubgraphNode` exclusively
- [ ] **Load from disk** → bundle bound, no tab until **↓** → tab named for bundle
- [ ] Inside disk tab: params live; edit mode cannot enable; nested **↓** grayed
- [ ] **Open original** / **⌘O** opens external bundle for full edit
- [ ] **↑** on inner param/viewer → exposed on parent `SubgraphNode` from **main**
- [x] Add `CodeBox` → creation panel → embedded stub `scripts/embedded_N.lua` → ports inferred → external edit → hot reload (`feature/boundaries-b4-chrome`)
- [x] CodeBox file ref: editable `script_ref` + ctl reload; disk subgraph: editable `bundle_ref` (`feature/boundaries-b4-rebind-gc`)
- [x] Orphan embedded-script GC preference + manual File action (`feature/boundaries-b4-rebind-gc`)
- [x] Reload failure → last-good + pale pink tint; port change → prune incompatible wires (debug pref logs prune)
- [x] Script `ui` previews (`numbers`/`dict`/`text`) visible in Node tab + in-node (like movie player `fps`) (`feature/boundaries-b4-script-ui-canvas`)
- [x] Script `ui.buttons` in-node click invokes Lua handler (`feature/boundaries-b4-script-ui-canvas`)
- [x] CodeBox canvas chrome: script-open, error badge/tint; in-node controls = `enable` + script `ui.params` (`in_node`)
- [x] Duplicate/paste embedded CodeBox → new `scripts/embedded_N.lua` with copied stub (`feature/boundaries-b4-script-ui-canvas`)
- [x] Embedded CodeBox create blocked until bundle has save path; file ref works on untitled doc
- [ ] Save bundle persists embedded `graphs/…json` and parent bindings
- [ ] Clipboard paste works at current tab’s graph level (no cross-tab paste v1)

---

## Changelog

| Commit | Summary |
|--------|---------|
| — | Plan doc opened; scope: Subgraph + CodeBox |
| — | Renamed from UX-E → **Boundaries** track |
| — | Architecture: shared boundary module, subgraph-first sequencing |
| — | UX: drill = title-bar ↓; double-click = hotswap |
| — | Layer tags `[core]` / `[app]` / `[core&app]` on all sections |
| — | Subgraph UX law: canvas tabs, embedded/disk, inlet inference, parent exposure; breadcrumb removed |
| 2026-06 | Core C1–C6 merged; `ARGUS_CORE_REF` bumped; app B1–B5 ready to start |
| 2026-06 | **B1:** creation panel, `boundary_authoring`, embedded vs disk params, CodeBox path stub |
| 2026-06 | **B2:** canvas tab bar, RAM-only embeds until save, per-tab isolation, tab-order/drill/undo/save fixes (`feature/boundaries-b2`) |
| 2026-06 | **B2+:** disk-ref tab live nodes/previews, global monitor cross-tab, bundle tab labels (`feature/boundaries-b2-plus` / stacked on B3 branch) |
| 2026-06 | **B3:** Inlet/Outlet inner-tab authoring, parent outlet/inlet relay, multi-tab save, main↔inner position sync (`feature/boundaries-b3`) |
| 2026-06 | **B4 docs:** [plan-codebox-ux.md](plan-codebox-ux.md), [codebox-lua-api.md](codebox-lua-api.md); align boundaries/architecture/roadmap |
| 2026-06 | **B4:** CodeBox creation panel, script chrome, hot reload, rebind, embedded-script GC (PR #35–#37) |
| 2026-06 | **B4:** script `ui` previews + in-node buttons; fork embedded on duplicate; dirty/layout fixes (PR #38) |
| 2026-06 | **B5:** parent **↑** exposure — `visible_in_parent`, inner-tab affordance, parent `SubgraphNode` param surfacing (`c6453e1…`) |