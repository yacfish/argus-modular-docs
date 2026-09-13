# CodeBox — UX & authoring spec (B4)

**Status:** **Shipped** on `main` (PR [#35](https://github.com/yacfish/argus-modular/pull/35)–[#38](https://github.com/yacfish/argus-modular/pull/38)) — keep in sync with [codebox-lua-api.md](codebox-lua-api.md)
**Track:** Boundaries B4 — **done**; post-B4 nice-to-haves in §16  
**Related:** [plan-boundaries.md](plan-boundaries.md) (B1–B3 done), [codebox-lua-api.md](codebox-lua-api.md) (Lua API — **first iteration**, B4), [plan-parameter-widgets.md](plan-parameter-widgets.md) (deferred `ui.params` types)

This is the **first** CodeBox authoring iteration. It replaces the interim B4 checklist (open/reveal only). ArgusCore has a fixed-port Lua **prototype** for headless tests; B4 implements the contract in this doc — not a migration from a prior user-facing API.

---

## 1. Mental model

`CodeBox` is a **boundary node** like `SubgraphNode`, but implementation is a **Lua file**, not a canvas graph.

| | SubgraphNode | CodeBox |
|---|---|---|
| Implementation | `graphs/…json` | `scripts/…lua` |
| “Go inside” | Canvas tab + title-bar **↓** | External editor (no canvas tab) |
| Creation choice | Embedded \| Load from disk | **Same** |
| Edit surface | In-app canvas | External editor only |
| Ports (v1) | User-authored Inlet/Outlet inside subgraph | **Declared in Lua** — inferred at load / reload |

**Lua choice:** avoid image processing in script code. Image may be an **input** (trigger / pacing); **no image outlet** in v1. Dict outlets (e.g. `outlet(0, { bypass = true })`) feed downstream **`ctl`** inlets.

Primary job: close the loop **edit file → reload → inferred ports → wire → run** — not merely open/reveal the path.

---

## 2. Creation panel (palette add)

Same moment as `SubgraphNode` — small panel at drop point:

```
Choose how to create this script:

  [ Embedded ]
  [ Load from disk… ]

  Esc to cancel
```

### 2.1 Embedded

- Bundle **must have a save path** (no temp files, no RAM script text).
- Allocate system-owned path: `scripts/embedded_0.lua`, `embedded_1.lua`, … — user **cannot** rename.
- **Write rich stub immediately** (declares `inlets` / `outlets` + handler examples — see [codebox-lua-api.md](codebox-lua-api.md)).
- Node params:

```json
{ "script_mode": "embedded", "script": "scripts/embedded_0.lua" }
```

If bundle is untitled / has no root: **Embedded** is disabled with message *“Save bundle first to create embedded script.”* Disk mode still works (absolute path to an existing library file).

### 2.2 Load from disk

- Native open dialog: `*.lua` (single file — not a bundle picker).
- Binds an external library file; **never copied** into the project bundle on save.

```json
{ "script_mode": "file_ref", "script_ref": "/absolute/path/to/lib/overlay.lua" }
```

Tooltip: *Bind an external script (reusable across projects; not copied into this bundle).*

---

## 3. Param model

| Param | Embedded | File ref |
|-------|----------|----------|
| `script_mode` | `"embedded"` | `"file_ref"` |
| `script` | `scripts/embedded_N.lua` | — |
| `script_ref` | — | path to external `.lua` (absolute or resolvable) |

**Mandatory on every CodeBox (core, not script):** `ctl` inlet + `enable` param.

**Identity / debugging:** filename stem + graph node id / name — no user-facing `script_id`.

**Dropped:** `script_source`, `script_pending_file`, user-editable embedded paths, fixed C++ port list.

**Post-B4:** unified `{ "scope", "path" }` **AssetRef** for `script` (replaces `script_mode` + dual keys) — [plan-asset-ref.md](plan-asset-ref.md). B4 implements this table as-is.

---

## 4. Authoring loop

```
Add CodeBox → choose mode → script loads → ports appear on node
    → wire in0… / out0… / ctl (last) → Open in editor → edit → save
    → hot reload → re-infer ports → inner previews (numbers / dict / text)
```

- Ports may **change** after reload; graph may need rewiring (see §15).
- **No in-app Lua editor** (v1).
- **No undo of script text** — undo restores graph binding only.

---

## 5. Node tab — Script section

### 5.1 Shared (both modes)

| Element | Behavior |
|---------|----------|
| Status | `● Loaded` / `● Error` / `○ Missing` |
| Last error | Parse error, port validation, or handler runtime error |
| **Open in editor** | Platform default app for `.lua` |
| **Reveal in Finder** | Reveal file; other platforms open containing folder |
| **Port map** | **Dynamic** — inferred ports after load (§5.6); **`ctl` last** |
| **Inspector UI** | Params, buttons, preview rows from script `ui` table (§5.7) |

### 5.2 Embedded mode

| Element | Behavior |
|---------|----------|
| Path | **Read-only** label: `scripts/embedded_0.lua` |
| Create script | N/A — stub written at create |

### 5.3 File ref mode

| Element | Behavior |
|---------|----------|
| Header | **External script** |
| Path | **Editable** `script_ref` — text + **Choose script…** |
| Re-bind | Same node; reload + re-infer ports |

### 5.4 Control inlet (`ctl`)

- Standard cold inlet: each key/value in the control dict updates params (`enable`, `script` / `script_ref` when sent, etc.).
- Script may read via `argus.param("key")`.
- Path change via ctl → reload script + update watch path.

### 5.5 Other params

- `enable` + ctl profile in Node tab.
- No editable `script` path for **embedded** mode.

### 5.6 Port map (dynamic)

After successful load, show inferred surface. **Canvas + map order:** warm `in0…`, warm `out0…`, then **`ctl` last** (cold control is never first).

```
Ports (from script)
  in0      in   image        → inlet_0()
  in1      in   dict         → inlet_1(dict)
  out0     out  dict         ← outlet(0, …)
  ctl      in   control      (mandatory, always last)
```

- Load / parse failure: keep **last-good** ports; show error in Node tab (§15).
- Full API: [codebox-lua-api.md](codebox-lua-api.md).

### 5.7 Script-declared inspector UI (params, buttons, inner previews)

Declared in Lua at load (same pass as `inlets`/`outlets`). Core registers metadata; app builds Node tab + in-node chrome from introspection (reuse existing `ui.params` / `ui.previews` — same model as `OpenCVMoviePlayer` inner `fps`).

```lua
ui = {
  params = {
    threshold = { type = "number", default = 0.5, min = 0, max = 1, in_node = true },
    label     = { type = "string", default = "" },
    asset     = { type = "path" },
    verbose   = { type = "bool", default = false },
  },
  buttons = {
    reset = "on_reset",
  },
  previews = {
    stats = { type = "dict" },
    rate  = { type = "number" },   -- kind: numbers (like movie player fps)
    note  = { type = "text" },
  },
}
```

| Script `type` | Preview `kind` (app) | `viewer` | Example |
|---------------|------------------------|----------|---------|
| `"number"` | `numbers` | `text` / `sparkline` | measured rate, counter |
| `"dict"` | `dict` (or `default` until dict viewer ships) | `text` | structured debug table |
| `"text"` | `default` | `text` | status string |

**Not in v1:** `cv_mat` / image previews from CodeBox (no image outlet).

| Surface | Read / emit | React | Inspector |
|---------|-------------|-------|-----------|
| Params | `argus.param("threshold")` | `on_param_change(key, val)` — inspector, `ctl`, and defaults on load | Node tab; `in_node` default; visibility toggles in graph `ui` |
| Buttons | — | `function on_reset() … end` (name from `ui.buttons`) | Node tab + in-node on click |
| Previews | app `sample_inner_preview(key)` | `publish_preview(key, value)` from script | `source: inner` slots — Monitor + in-node (same as `inner: fps` on movie player) |

**Param flow:** `ui.params` keys are registered in the node param bag. User edits (or `ctl` updates) call `on_param_change` when defined; otherwise scripts poll with `argus.param`. Only keys in `ui.params` invoke the handler — not `enable` or path params.

**Preview flow:** script calls `publish_preview`; core stores typed values; app samples via `CodeBox::sample_inner_preview(key)` (like `OpenCVMoviePlayer::sample_inner_preview("fps")` → `ScalarF32`). App infers preview `kind` from script declaration when seeding rows.

**B4 slice:** port inference first; `ui` + typed previews in same pass if feasible, else **B4+**.

---

## 6. Canvas chrome

- **Do not** reuse subgraph **↓**.
- Title-bar **script-open** (`</>` or equivalent).
- **Error badge** when load or handler errors.
- **Error tint:** pale pink node background when script file is missing, parse fails, or reload leaves the node on last-good broken state (§15) — visible on the graph without opening Node tab.
- **In-node controls:** `enable` plus script `ui.params` with `in_node: true` (default). Hide core path keys (`script`, `script_ref`, `script_mode`, `script_id`) — same rule as Node tab Parameters table.
- Port order on node: **`in0…` → `out0…` → `ctl` last**. Tooltips: `image` / `dict` / `control`.

---

## 7. Lua runtime contract (v1)

**Authoritative detail:** [codebox-lua-api.md](codebox-lua-api.md).

Summary:

```lua
inlets  = { argus.image, argus.dict }
outlets = { argus.dict }
ui      = { params = { … }, buttons = { … }, previews = { … } }

function on_param_change(key, val) … end
function on_reset() … end
function inlet_0() … end
function inlet_1(dict) … end
outlet(0, { bypass = true })
publish_preview("rate", 12.5)
print("debug")                     -- gated by CodeBox debug pref
```

**Teaching (v1):**

1. Rich stub at create (ports + `ui` + handlers).
2. `docs/codebox-lua-api.md`.
3. Dynamic port map (§5.6) + inspector rows from `ui` (§5.7).

---

## 8. Hot reload & feedback

- **Core:** `watch.scripts` + `script_reloaded`; reload must **re-parse port tables** and reconfigure node.
- **Reload failure** (missing file, syntax error, invalid port declaration): keep **last-good** ports + last-loaded script behavior until the user fixes the file; Node tab shows error; canvas **pale pink** tint (§6).
- **Port declaration change** on successful reload: remove graph connections that are incompatible (**wrong type** or **port index no longer exists**). Log disconnect details when **CodeBox debug** is enabled (Preferences).
- **App:** status chip + last error from load and runtime.
- **Previews:** script `ui.previews` → inner slots (`numbers`, `dict`, `text`) — not image in v1.

---

## 9. Lifecycle: delete, undo, fork

### 9.1 Delete node

- Topology only; **do not** delete `scripts/embedded_N.lua` immediately.

### 9.2 Undo / redo

- Restores binding (`script` / `script_ref`); reload + re-infer ports.
- Tracks path/mode changes (UI + ctl); not line-level script edits.

### 9.3 Duplicate / paste

- **Embedded:** copy file → new `embedded_M.lua`.
- **File ref:** may share `script_ref`.

---

## 10. Unused embedded script cleanup

Optional app preference (default **off**):

> **Remove unused embedded scripts when closing a bundle**

Help text:

> Deletes `scripts/embedded_*.lua` not referenced by any embedded `CodeBox` in **main.json or any `graphs/embedded_*.json`**.

Runs on bundle close or app quit (not on every Save). Order: dirty-save prompt → persist decision → cleanup → unload.

Recursive reference scan: all `CodeBox` with `script_mode: embedded` in main + embedded subgraph JSON; skip `bundle_ref` subgraphs. **Never delete** `file_ref` targets.

| Close outcome | Graphs scanned |
|---------------|----------------|
| **Saved** | Editor main + all reachable embedded inners |
| **Don’t save** | On-disk `main.json` + `graphs/embedded_*.json` |
| **Not dirty** | Current editor graphs |

Manual **Remove unused embedded scripts…** (confirm + count) always available. Preference: `DocumentsPrefs.auto_delete_unused_embedded_scripts`.

---

## 11. Intentional asymmetry vs SubgraphNode

| | Subgraph embedded | CodeBox embedded |
|---|---|---|
| First disk write | On **Save** | **On create** |
| Port surface | Inlet/Outlet nodes inside graph | **`inlets` / `outlets` in Lua** |
| User controls asset path | `embedded_N` system-owned | `embedded_N.lua` system-owned |

### 11.1 Path editability (locked from review)

External disk refs must be **rebindable on the same node** (Node tab + picker **and** `ctl` dict). Otherwise users must delete/recreate the node and cannot swap scripts/bundles from the graph.

| Mode | Path param | User edits path in UI? | Via `ctl` dict? |
|------|------------|------------------------|-----------------|
| Subgraph embedded | `subgraph_file` / `embedded_N` | **No** — system-owned | No |
| Subgraph disk ref | `bundle_ref` | **Yes** — picker + text | **Yes** — reload inner graph |
| CodeBox embedded | `script` / `embedded_N.lua` | **No** — system-owned | No |
| CodeBox file ref | `script_ref` | **Yes** — picker + text | **Yes** — reload + re-infer ports |

**B4 (slices 13–14):** disk `SubgraphNode` `bundle_ref` and CodeBox `script_ref` are rebindable in the Node tab (picker + text → `set_node_parameter` / ctl reload). Embedded path keys (`script`, `subgraph_file`) stay system-owned.

---

## 12. Core vs app (B4)

| Layer | Work |
|-------|------|
| **Core (blocks app)** | Parse `inlets`/`outlets`/`ui`; register warm ports (`ctl` last); `inlet_N` / `outlet` / `publish_preview` / `on_param_change` / `argus.param`; gated `print`; `sample_inner_preview`; reject image outlets; reload reconfig + last-good; `script_ref` |
| **App** | Shared creation panel; Node tab (port map + `ui` inspector); script-open; pale-pink error tint; `script_reloaded` + CodeBox debug log; fork; GC pref; `bundle_ref` + `script_ref` rebind |

---

## 13. Out of scope (v1)

- In-app Lua IDE
- Image **outlet** or pixel read API
- Fixed four-port C++ prototype (replaced in B4 core — not part of this API)
- Python CodeBox
- Temp script files / RAM `script_source`
- In-app API cheat sheet (use `codebox-lua-api.md`)
- Optional `on_ctl(dict)` hook

---

## 14. Implementation slices

**Core (argus-core) — do first**

1. Port + `ui` declaration parser (`inlets`/`outlets`/`ui.params`/`ui.buttons`/`ui.previews`). **Done** — core [#44](https://github.com/yacfish/argus-core/pull/44).
2. Dynamic port registration (`ctl` last) + hot-reload reconfig + last-good on failure.
3. `inlet_N` / `outlet` / `publish_preview` / `on_param_change` / `argus.param` / gated `print`.
4. `sample_inner_preview(key)` — `numbers` / `dict` / `text` kinds.
5. `script_ref` + path reload (incl. ctl); wire prune on port change.
6. Default stub template; replace fixed-port prototype; update demo scripts/tests.
7. `argus_test_codebox_ports` (+ inner preview tests).

**App (argus-modular)** — **Done** (`main`, PR #35–#38). Core pin `3987251…` (`invoke_ui_button`).

8. **Single** creation panel (Subgraph + CodeBox) + `script_mode`. **Done**
9. Node tab: Script section, dynamic port map, inspector rows from `ui` introspection. **Done**
10. Title-bar script-open; error badge; **pale pink** error tint; in-node controls filtered (§6). **Done**
11. `script_reloaded` status; CodeBox debug pref (wire prune + `print` logs). **Done**
12. Fork embedded on duplicate/paste (bundle path required for embedded create). **Done**
13. Rebindable `script_ref` + `bundle_ref` (picker + ctl). **Done**
14. Orphan embedded-script GC preference (main + `graphs/embedded_*.json`). **Done**
15. Keep [codebox-lua-api.md](codebox-lua-api.md) in sync with core. **Done**
16. Smoke tests (`argus_test_boundary_authoring_smoke`). **Done**

---

## 15. Resolved decisions

| # | Topic | Decision |
|---|-------|----------|
| 1 | Port change on successful reload | **Disconnect** incompatible wires (wrong type or port gone). Warn in log when **CodeBox debug** pref is on — not a modal. |
| 2 | Reload failure (missing file, bad code) | **Keep last-good** ports/runtime until user fixes file. Node tab error + **pale pink** CodeBox background on canvas (§6). |
| 3 | Image inlet handler | **`inlet_0()`** trigger only — no frame metadata arg in v1. |
| 4 | Creation panel | **Single panel** for SubgraphNode and CodeBox (type-specific title/buttons). |
| 5 | Convert embedded ↔ disk ref | **Nice to have** (not required for B4 ship). If implemented, support **both** CodeBox (`embedded` ↔ `file_ref`) and SubgraphNode (`embedded` ↔ `bundle_ref`). |

---

## 16. Nice to have (post-B4 or if time)

- Embedded ↔ disk ref conversion UI (CodeBox + SubgraphNode) — §15 row 5.
- Port map wired/unwired hints.
- `inlet_0(meta)` with `frame_number` — **partial:** `argus.inlet_image("in0")` + `argus.cpu_time()` on `main`; see [codebox-lua-api.md](codebox-lua-api.md).
- Richer `ui.params` types (`vec2`, `enum`, `int`/`float`, colors) + shared widget registry — [plan-parameter-widgets.md](plan-parameter-widgets.md).

---

## Changelog

| Date | Summary |
|------|---------|
| 2026-06 | Initial draft (open/reveal checklist) |
| 2026-06 | `script_id` dropped; editable `script_ref`; `bundle_ref` rebind |
| 2026-06 | API in `codebox-lua-api.md` only |
| 2026-06 | First-iteration API: script-declared ports, `inlet_N` / `outlet`, dynamic port map |
| 2026-06 | Remove incorrect “legacy API” framing — B4 is first user-facing CodeBox ship |
| 2026-06 | Doc pass: align plan-boundaries, architecture, roadmap, AGENTS, README |
| 2026-06 | `ctl` last in port order; `ui` table for params/buttons/typed inner previews |
| 2026-06 | §15 decisions: last-good + pink error tint; prune bad wires; single creation panel |
| 2026-06 | Completeness pass: §12/§14/ui previews; status → reviewed |
| 2026-06 | Rename sandbox emit API `out` → `outlet` (pairs with `inlet_N`) |
| 2026-06 | `on_param_change(key, val)` + `publish_preview(key, value)` for ui params/previews |
| 2026-06 | §14: defer `ARGUS_CORE_REF` bump until core slices 1–7 land; mark slice 1 done (core #44) |
| 2026-06 | **B4 shipped** on `main` (app PR #35–#38); core pin `3987251` |