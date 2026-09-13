# Initial plan — argus-modular (argus-ui)

**Date**: June 2026  
**Status**: **Phases U1–U5 done**; **U6 editor polish** largely done. See [roadmap.md](roadmap.md) for what is next.

This document records the **bootstrap plan** copied from the original app README and what was delivered against it. It is the historical reference for the first editor + run-mode milestones.

**Engine dependency:** [argus-core](https://github.com/yacfish/argus-core) — linked as `third_party/argus-core` or sibling checkout.  
**UI stack (v1):** Dear ImGui + GLFW. Core stays UI-free.

**Authoritative UI design (argus-core):** `docs/internal/plan-graph-editor-ui.md` in argus-core.

---

## 1. What this repo is

**argus-modular** (working name **argus-ui**) is the visual application for Argus:

| Mode | Planned phases | Behaviour |
|------|----------------|-----------|
| **Run** | U1–U2 | Load bundle, always-on executor, live controls, throttled previews |
| **Edit** | U3–U5 | Graph canvas, wiring, node panel, save bundle |

It **imports argus-core** as a library. It does **not** duplicate graph, executor, or node code.

---

## 2. Planned repository layout

```
argus-modular/
├── README.md
├── CMakeLists.txt
├── docs/
│   ├── roadmap.md
│   └── plan_initial.md          # this file
├── third_party/
│   └── argus-core/              # submodule or sibling path
├── src/
│   ├── main.cpp
│   ├── app.cpp / app.hpp        # GLFW + ImGui lifecycle
│   ├── graph_session.cpp        # owns argus::Graph, load/save bundle
│   ├── signal_bridge.cpp        # SignalBus → UI state
│   ├── preview_view.cpp         # U1: cv_mat display
│   ├── graph_canvas.cpp         # U3: nodes, wires, palette
│   ├── node_panel.cpp           # U4: param visibility
│   └── edit_mode_panel.cpp      # U3–U5: edit shell, presets
└── cmake/
    └── argus_link_builtin_nodes.cmake
```

---

## 3. Application architecture

### 3.1 Threads

| Thread | Work |
|--------|------|
| **Main / UI** | GLFW events, ImGui frame, draw previews, `set_parameter` from widgets |
| **Executor** | `graph.start()` → internal thread runs push ticks at `tick_hz` |

The UI thread does **not** call `tick_once()` while the executor is running in run mode.

### 3.2 Core APIs used

| Task | API |
|------|-----|
| Load project | `graph.load_bundle(path)` |
| New empty doc | In-memory graph + `save_bundle_as` on first save |
| Save project | `graph.save_bundle(path)` |
| Preset | `graph.apply_preset_file(...)` / `save_preset_file(...)` |
| Executor | `graph.start()` after doc open; `poll_executor` in app loop (no manual Run/Stop UI) |
| Live param | `graph.set_parameter("node_name", "param", json)` |
| Node picker | `NodeFactoryRegistry::node_catalog_json()` |
| Debug wire | `graph.resolve_port_ref_json({node, port})` |
| Preview throttle | `graph.set_preview_max_hz(node_id, "out", hz)` |
| Script watch | `graph.poll_script_watch()` during ticks |

### 3.3 Test bundle

Fixture from argus-core:

```
third_party/argus-core/examples/bundles/demo.bundle/
```

---

## 4. Phased delivery (initial plan)

| Phase | Scope (planned) | Status |
|-------|-----------------|--------|
| **U1** | Run shell — load bundle, always-on executor, one CPU preview | **Done** |
| **U2** | Control panel — sliders/toggles from node `ui` + manifest `ui_defaults` | **Done** |
| **U3** | Graph canvas — custom ImGui canvas; read/write graph via `to_json()` | **Done** |
| **U4** | Node panel — param visibility, preview registration | **Done** |
| **U5** | Save bundle — `save_bundle`, preset picker, `save_preset_file` | **Done** |
| **U6** | Graph editor polish — selection, ports, delete, wiring UX | **Mostly done** |

---

## 5. U1 acceptance checklist

- [x] Window opens (GLFW + ImGui)
- [x] **File → Open bundle** loads `demo.bundle`
- [x] **Run** starts executor; **Stop** stops it
- [x] `parameter_changed` updates a debug panel when params change
- [x] At least one **CPU preview** (`cv_mat`) displays from `preview_updated`
- [x] No graph/executor code duplicated — only `argus_core` calls
- [x] Builds on macOS 12+ with argus-core on `main`

---

## 6. Delivered beyond the original checklist

| Item | Notes |
|------|-------|
| Run / Edit mode toggle | `AppMode` — edit while stopped, run locks executor |
| Empty startup document | No auto-load of `demo.bundle`; **Save as** on first save |
| Bundle sync on open | Editor layout restored from bundle entry graph |
| Self-connection block | Cannot wire a node to itself |
| Wire-drag UX | Lock outputs while dragging; skip duplicate inlets; ignore same-node inputs |
| Preset combo / Load ID fix | ImGui ID scope fix |
| `ui.params` preserved | Editor metadata kept when committing graph JSON |
| Node panel stability | Deferred `apply_changes` avoids crash on visibility toggle |

---

## 7. ImGui + OpenGL notes (v1)

- Reuse **GLFW** and **GLES** targets from the argus-core subproject.
- U1 preview path: `PreviewUpdatedEvent` with `kind == "cv_mat"` → upload RGB to ImGui texture (throttled by core).
- GLES previews (`gl_texture`) deferred — see [roadmap.md](roadmap.md).

---

## 8. Related plans (argus-core)

| Topic | Path in argus-core |
|-------|-------------------|
| UI phases (authoritative) | `docs/internal/plan-graph-editor-ui.md` |
| Core/UI split | `docs/internal/architecture-repos.md` |
| Bundles | `docs/internal/plan-app-bundle.md` |
| Signals / previews | `docs/internal/plan-signal-preview.md` |
| Audio (later, in core) | `docs/internal/plan-audio-nodes.md` |

---

**Next-quality bar:** [roadmap.md](roadmap.md) · [architecture.md](architecture.md)

*Historical bootstrap document. Active priorities live in [roadmap.md](roadmap.md).*