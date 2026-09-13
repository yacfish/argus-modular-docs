# AssetRef — unified bundle / external paths

**Track:** param model + path resolution  
**Status:** **Shipped** on `main` — CP1–CP5 done (merged [#41](https://github.com/yacfish/argus-modular/pull/41), June 2026)  
**Core pin:** `5c495a235f4331f20ec208027c54c33a52f1fc14` (codebox_globals_setup)  
**Prerequisites:** [plan-boundaries.md](plan-boundaries.md) B4 on `main`  
**Related:** [plan-app-preferences.md](plan-app-preferences.md) (`search_paths` wiring)

**Layer tags:** `[app]` = argus-modular only · `[core]` = argus-core only · `[core&app]` = both repos

---

## Goal

Replace the duplicated **mode flag + two path keys** pattern with one shared **AssetRef** value type for "where does this file live?" — bundle-relative vs external absolute — while keeping **resource-specific param names** (`script`, `subgraph`, `path`, …).

This makes every path param uniform in shape, eliminates mode flags, and gives GPU/media nodes a single path model to adopt later.

---

## Problem today

| Node | Mode | In-bundle | External |
|------|------|-----------|----------|
| SubgraphNode | `subgraph_mode` | `subgraph_file` (+ RAM `subgraph_json` / `subgraph_pending_file`) | `bundle_ref` |
| CodeBox (B4) | `script_mode` | `script` | `script_ref` |
| Media / generic | — | bare relative `path` | — |

Three different shapes for the same concept. The app duplicates: `classify_path_param()`, separate pickers, separate ctl/rebind rules.

---

## AssetRef shape `[app]`

**JSON param value:**

```json
{ "scope": "bundle",   "path": "scripts/embedded_0.lua" }
{ "scope": "external", "path": "/Users/lib/overlay.lua" }
```

**Optional string form** (serialization alias for debugging / concise UI):

```
bundle:scripts/embedded_0.lua
external:/Users/lib/overlay.lua
```

| `scope` | Meaning | Resolve against | UI |
|---------|---------|-----------------|-----|
| `bundle` | Path relative to current bundle root | `GraphSession` bundle root | Read-only when system-allocated (`embedded_N`); editable for user media |
| `external` | Absolute filesystem path | As-is | Editable + native picker; not copied into bundle on save |

**Param rename:** `subgraph_file` + `bundle_ref` → `subgraph` (single AssetRef param). `script` + `script_ref` → `script` (single AssetRef param). The `subgraph_json` sidecar (RAM-only before save) is **kept** — AssetRef only replaces the path storage, not the inline graph model.

**Mode flags eliminated:** `subgraph_mode` and `script_mode` are dropped on new saves. The scope is now part of the AssetRef value.

---

## Why app-only (no core changes)

The core stores SubgraphNode and CodeBox params as opaque JSON via `serialized_fixed_params()` / node JSON — not through `ParameterBag`. Neither `argus::SubgraphNode` nor `argus::CodeBox` cares about the internal shape of `subgraph_file` vs `bundle_ref` vs `script` vs `script_ref`. They just read the values they need:

- `SubgraphNode::bundle_ref()` — reads the `bundle_ref` param string
- `CodeBox::resolve_bound_script_path()` — reads `script` or `script_ref`
- `Graph::resolve_path_relative_to_load_base()` — resolves relative paths against bundle root

**AR1 migrates the app layer only.** The core continues to receive strings from the app — those strings just come from `AssetRef::resolve()` instead of ad-hoc string lookups. If a future core version wants first-class AssetRef support, that's a separate incremental change (no pin bump needed).

---

## What changes in each repo

### argus-modular (`[app]`)

| File | Change |
|------|--------|
| `src/asset_ref.hpp` | **New** — `AssetScope` enum, `AssetRef` struct, JSON serialization, resolve, legacy conversion |
| `src/asset_ref.cpp` | **New** — implementation |
| `src/boundary_authoring.hpp` | Add AssetRef-aware classifiers; keep legacy for backward compat |
| `src/boundary_authoring.cpp` | Update `classify_path_param`, `is_subgraph_bundle_reference`, `is_codebox_file_reference`, `is_embedded_subgraph_params`, `is_embedded_codebox_params` to detect new format |
| `src/boundary_authoring.cpp` | Update `make_embedded_codebox_params`, `make_file_ref_codebox_params`, `make_embedded_subgraph_params`, `make_bundle_ref_subgraph_params` to output AssetRef |
| `src/boundary_authoring.cpp` | Update `fork_embedded_subgraph_params`, `fork_embedded_codebox_params`, `materialize_embedded_subgraphs_for_save` |
| `src/boundary_authoring.cpp` | Update `resolve_codebox_script_path` |
| `src/node_panel.cpp` | Add AssetRef widget rendering (scope selector + path + picker) |
| `src/node_canvas_chrome.cpp` | Add AssetRef widget rendering on canvas |
| `src/asset_resolver.cpp` | Wire `resolve_asset_path` with preferences `search_paths` |
| `src/control_visibility.cpp` | Add `asset_ref` widget mapping |
| `cmake/argus_core_debug_perf.cmake` | Debug UI builds: compile argus-core at `-O2` (UI stays `-O0`) |
| `src/signal_bridge.*` / `src/preview_view.*` / `src/graph_session.*` | Zero-copy preview ingest; generation dedup (see Post-CP5 fixes) |
| `CMakeLists.txt` | Add `asset_ref.cpp`, test targets, `argus_apply_core_debug_perf()` |

### argus-core (`[core]`)

No changes required for AR1. The existing API surface (`SubgraphNode::bundle_ref()`, `CodeBox::resolve_bound_script_path()`, `Graph::resolve_path_relative_to_load_base()`) handles strings — AssetRef resolves to strings before passing to core.

**Future** (`AR1+` / optional): core could gain a native `AssetRef` type for bundle I/O, but that's not needed for this track.

---

## Migration strategy

AR1 uses a **dual-write / backward-read** strategy so existing bundles open without migration:

1. **Writer mode (new saves):** `materialize_embedded_subgraphs_for_save()` and all creation functions write the new AssetRef format (`{"scope": "bundle", "path": "..."}`). `subgraph_mode` / `script_mode` are omitted.
2. **Reader mode (load):** When reading node params, detect legacy format by checking for `subgraph_mode`, `bundle_ref`, `subgraph_file`, `script_mode`, `script_ref`. Convert to AssetRef-equivalent at read time via `AssetRef::from_legacy()`.
3. **Dual-format tolerance:** All classifiers check **both** legacy and new formats throughout the migration. This lets us open old bundles, edit, and re-save — the re-save produces new format.

This means no migration script, no version bump, no breaking changes.

---

## Checkpoints

### CP1 — AssetRef type (items 1–2 of implementation)

| Deliverable | Files |
|-------------|-------|
| `AssetScope` enum + `AssetRef` struct | `src/asset_ref.hpp`, `src/asset_ref.cpp` |
| JSON serialization (`to_json`, `from_json`) | same |
| String form (`to_string`, `from_string`: `bundle:...` / `external:...`) | same |
| `resolve(const GraphSession&) -> std::string` | same |
| `from_legacy_params(json) -> AssetRef` — converts subgraph_mode/script_mode/bundle_ref/subgraph_file/script/script_ref | same |
| Updated classifiers in `boundary_authoring` | `boundary_authoring.cpp` |
| Added to build system | `CMakeLists.txt` |

**Acceptance:**
- [x] `AssetRef` parses `{"scope":"bundle","path":"x"}` and `{"scope":"external","path":"/y"}`
- [x] Rejects missing scope, empty path, invalid scope
- [x] `from_legacy_params({subgraph_mode:"embedded", subgraph_file:"graphs/e0.json"})` → `{scope:bundle, path:"graphs/e0.json"}`
- [x] `from_legacy_params({subgraph_mode:"bundle_ref", bundle_ref:"/a/b.bundle"})` → `{scope:external, path:"/a/b.bundle"}`
- [x] `from_legacy_params({script_mode:"embedded", script:"scripts/e0.lua"})` → `{scope:bundle, path:"scripts/e0.lua"}`
- [x] `from_legacy_params({script_mode:"file_ref", script_ref:"/x.lua"})` → `{scope:external, path:"/x.lua"}`
- [x] `resolve()` returns bundle-root-joined path for `bundle` scope, absolute path for `external`
- [x] `classify_path_param()` and friends detect new AssetRef format

### CP2 — Boundary creation + fork migration (items 3–4)

| Deliverable | Files |
|-------------|-------|
| `make_embedded_codebox_params` → produces `script: {scope:bundle, path:"scripts/e_N.lua"}` | `boundary_authoring.cpp` |
| `make_file_ref_codebox_params` → produces `script: {scope:external, path:"..."}` | same |
| `make_embedded_subgraph_params` → produces `subgraph: {scope:bundle, path:"graphs/e_N.json"}` | same |
| `make_bundle_ref_subgraph_params` → produces `subgraph: {scope:external, path:"..."}` | same |
| `fork_embedded_*` → forks with AssetRef | same |
| `materialize_embedded_subgraphs_for_save` → writes AssetRef | same |

**Acceptance:**
- [x] Creating embedded CodeBox writes `script` as AssetRef, no `script_mode`
- [x] Creating file-ref CodeBox writes `script` as external AssetRef, no `script_mode`
- [x] Creating embedded subgraph writes `subgraph` as AssetRef, no `subgraph_mode`
- [x] Creating bundle-ref subgraph writes `subgraph` as external AssetRef, no `subgraph_mode`
- [x] Forking duplicates with new `embedded_N` paths in AssetRef format
- [x] Save materialization writes AssetRef format to bundle + node params

### CP3 — Resolution + backward compat (items 5, 7–8)

| Deliverable | Files |
|-------------|-------|
| `resolve_codebox_script_path` → handles AssetRef | `boundary_authoring.cpp` |
| Legacy param detection at load → implicit AssetRef conversion | `boundary_authoring.cpp` (classifiers + read helpers) |
| Dual-format tolerance in all classifiers | `boundary_authoring.cpp` |
| `resolve_asset_path` wired with preferences `search_paths` | `asset_resolver.cpp`, `app_preferences.cpp` |

**Acceptance:**
- [x] Legacy bundles open without error — all classifiers return correct results
- [x] Editing a legacy bundle and re-saving produces AssetRef format
- [x] `resolve_codebox_script_path` works with both legacy and AssetRef params
- [x] Embedded script path resolves correctly against bundle root
- [x] External script path resolves as absolute path
- [x] Search paths from preferences are consulted for bundle-scope resolution

### CP4 — AssetRef UI widget (item 6)

| Deliverable | Files |
|-------------|-------|
| AssetRef widget in node panel | `node_panel.cpp` |
| AssetRef widget on canvas chrome | `node_canvas_chrome.cpp` |
| `asset_ref` widget mapping in control_visibility | `control_visibility.cpp` |

**Widget layout:**
- Row shows the param name + a scope badge (📦 `bundle` / 🔗 `external`)
- Path is editable text + native picker button
- Embedded/system-allocated paths have read-only path with tooltip
- Picker filters respect scope: bundle-scope offers file-in-bundle picker, external offers native file dialog

**Acceptance:**
- [x] AssetRef params render in node panel with scope badge and editable path
- [x] Changing scope switches between bundle/external modes
- [x] Bundle-scope embedded paths are read-only (system-allocated)
- [x] External-scope paths show native file picker
- [x] Canvas chrome shows AssetRef params similarly

**CP4 regression (fixed):** `button` and `bool` branches were accidentally nested inside the `asset_ref` branch in `node_panel.cpp` and `node_canvas_chrome.cpp`. CodeBox/Subgraph AssetRef params fell through to `schema["value"].get<bool>()` on a JSON object → `json::type_error` every frame in debug when those nodes were on canvas.

### CP5 — Build + tests (item 9)

| Deliverable | Files |
|-------------|-------|
| `asset_ref.cpp` in `add_executable(argus_ui ...)` | `CMakeLists.txt` |
| Smoke test for AssetRef | `examples/test_asset_ref_smoke.cpp` + test target |
| Extend boundary authoring smoke test for new format | `examples/test_boundary_authoring_smoke.cpp` |

**Acceptance:**
- [x] `argus_ui` builds and links with no new warnings
- [x] `argus_test_asset_ref_smoke` passes (AssetRef serialization, legacy conversion, resolve)
- [x] Existing tests still pass with dual-format classifiers

### Post-CP5 — Debug preview performance

**Problem:** Debug UI builds (`CMAKE_BUILD_TYPE=Debug`) were unusably slow with `OpenCVMoviePlayer` in-node previews — even graphs with only a movie player (no CodeBox/Subgraph). UI thread stalled; inner `fps` readout dropped below 1. Release builds were fine.

**Root cause:** On every graph tick, core `PreviewProcessor` clones/downscales frames; `SignalBridge` copied the full RGB buffer again; `PreviewView` copied again and called `glTexImage2D` every UI frame even when the preview had not changed. Argus-core at `-O0` made decode/downscale worse.

| Fix | Files |
|-----|-------|
| Zero-copy `cv_mat` — borrow core `DataPtr` in `PreviewFrame` | `signal_bridge.hpp/cpp` |
| Generation counter — skip ingest + GL upload when unchanged | `signal_bridge.*`, `graph_session.*`, `preview_view.*` |
| `peek_preview()` returns `shared_ptr<const PreviewFrame>` | `signal_bridge.*`, call sites |
| Debug UI + optimized core — `argus_apply_core_debug_perf()` adds `-O2` to `argus_core` / `argus_builtin_nodes` / `argus_gles` when UI is Debug | `cmake/argus_core_debug_perf.cmake`, `CMakeLists.txt` |

**Acceptance:**
- [x] `only movie.bundle` in `build-debug/argus_ui` — responsive UI, movie inner fps ~25
- [x] Release and Debug `argus_ui` build clean

---

## Out of scope (AR1)

- **Media node `path` params** → AssetRef (AR2) — GPU pipeline can adopt the pattern
- **User-renamable `embedded_N` on-disk paths**
- **Cloud / URL schemes** (`https:`, `s3:`)
- **Copy external file into bundle on link** (explicit import only)
- **Core-side AssetRef type** (no core changes needed)

---

## Sequencing

```
AR1 (shipped)  →  GPU pipeline (GP1…)
B5 (shipped)   —  parent ↑ exposure on `main`
```

AR1 and B5 both shipped on `main`. Next track: [plan-gpu-pipeline.md](plan-gpu-pipeline.md).

---

## Progress

| Checkpoint | Status | Files |
|------------|--------|-------|
| **CP1** — AssetRef type + classifiers | **Done** | `src/asset_ref.hpp`, `src/asset_ref.cpp`, updated `boundary_authoring.cpp` classifiers |
| **CP2** — Boundary creation + fork migration | **Done** | `boundary_authoring.cpp` (make_*, fork_*, materialize), test updated |
| **CP3** — Resolution + backward compat | **Done** | `boundary_authoring.cpp`, `asset_resolver.cpp`, `canvas_tabs.cpp`, `embedded_script_gc.cpp`, `graph_topology_session.cpp`, `node_defaults.cpp` |
| **CP4** — AssetRef UI widget | **Done** | `node_canvas_chrome.cpp/hpp`, `node_panel.cpp` (+ control-flow fix) |
| **CP5** — Build + tests | **Done** | `CMakeLists.txt`, `examples/test_asset_ref_smoke.cpp` |
| **Post-CP5** — Debug preview perf | **Done** | `signal_bridge.*`, `preview_view.*`, `graph_session.*`, `cmake/argus_core_debug_perf.cmake` |

---

## Acceptance (full AR1)

- [x] Embedded CodeBox: `{"script": {"scope": "bundle", "path": "scripts/embedded_0.lua"}}` — create, open, hot reload
- [x] External CodeBox: `{"script": {"scope": "external", "path": "…/lib.lua"}}` — rebind in Node tab + ctl
- [x] Subgraph embedded: `{"subgraph": {"scope": "bundle", "path": "graphs/e0.json"}}` + `subgraph_json` until save → materialize unchanged behavior
- [x] Subgraph disk: `{"subgraph": {"scope": "external", "path": "…/b.bundle"}}` — tab, open original, rebind
- [x] Legacy bundles with `bundle_ref` / `script_ref` / `subgraph_file` / `script` open without manual migration
- [x] Editing + re-saving a legacy bundle migrates to AssetRef format
- [x] Search paths from preferences work with AssetRef resolution
- [x] Debug build with video previews remains interactive (`build-debug/argus_ui`)
- [ ] GPU pipeline prerequisite doc updated — GP1 may assume AssetRef pickers
