# Roadmap — argus-modular

**Updated**: July 2026  
**Purpose**: Single planning doc for the UI repo. System map: [architecture.md](architecture.md).

**Engine:** [argus-core](https://github.com/yacfish/argus-core) — pin `23559df…` (`ARGUS_CORE_REF`, J4.3 spiderweb/crosstile)  
**Bootstrap history:** [plan_initial.md](plan_initial.md)  
**Feature tracks:** [plan-boundaries.md](plan-boundaries.md) · [plan-asset-ref.md](plan-asset-ref.md) · [plan-gpu-pipeline.md](plan-gpu-pipeline.md) · [plan-parameter-types.md](plan-parameter-types.md) · [plan-parameter-widgets.md](plan-parameter-widgets.md) · [plan-color-parameters.md](plan-color-parameters.md) · [plan-jxs-import.md](plan-jxs-import.md) · [plan-app-preferences.md](plan-app-preferences.md) · [plan-graph-canvas-split.md](plan-graph-canvas-split.md)  
**UX polish history:** `next_on_UX-*.md` (UX-A–D, merged)  
**Live topology:** [plan-live-topology.md](plan-live-topology.md) — core shipped; UI polish remains

~~[plan-ui-ux.md](plan-ui-ux.md)~~ — **closed**; merged here + architecture.

---

## Product principles

| Principle | Today | Target | Tag |
|-----------|-------|--------|-----|
| Executor while document open | Always-on; no manual transport UI | Same; `[EDIT]` is layout-only | **(done)** |
| Topology edits (interactive) | Live `add_node` / connect / hot-swap | Same | **(done)** |
| LT polish | — | `topology_changed`, fatal Restart graph | **(soon)** [plan-live-topology.md](plan-live-topology.md) |
| Parameter tweaks | Live `set_parameter` | Same | **(done)** |
| One document | Single bundle | Same | **(done)** |

---

## Layout (shipped)

- **Canvas left** — graph editor (`GraphCanvas` + split modules per [plan-graph-canvas-split.md](plan-graph-canvas-split.md)).
- **Side column** — **Monitor** · **Node** (inspector) · **Presets** (no Debug tab — Debug in **Preferences → Debug**).
- **Node tab = inspector** — all params, visibility, preview routing, parent **↑** exposure toggles (Boundaries).
- **Add nodes** — floating picker (`⌘N`, double-click empty canvas, double-click name to replace).
- **Menu `[EDIT]`** — enable/disable edit (UX-D).
- **Argus / File menus** — About, Preferences, recent bundles, templates, reload — [plan-app-preferences.md](plan-app-preferences.md).

**(deferred)** Docking, status bar — untracked § below.

---

## Keyboard shortcuts

[docs/keyboard-shortcuts.md](keyboard-shortcuts.md)

---

## Palette tiers (catalog)

| Tier | Scope |
|------|--------|
| **P0** | CPU vision, `SubgraphNode`, `CodeBox` — CPU loops **(done)**; Boundaries B1–B5 **(done)** [plan-boundaries.md](plan-boundaries.md) |
| **P1** | GLES upload/shader/window — catalog **(done)**; **`glsl_*` palette + GP3/GP3.2 + external packs (GP4 runtime)** **(done)** [plan-gpu-pipeline.md](plan-gpu-pipeline.md) |
| **P2** | BlobDetector, Sketch2D, recorder, … — **(deferred)** |
| **P3** | Vulkan, compute — **(deferred)** / build-gated; Inlet/Outlet subgraph-only |

---

## Completed

| Phase | Summary |
|-------|---------|
| U1–U5 | Bootstrap — [plan_initial.md](plan_initial.md) |
| U6 partial | Selection, wiring, empty doc |
| **UX-A–D** | Shell, canvas UX, clipboard, panel polish — `next_on_UX-*.md` |
| **GC-1–GC-4** | Graph canvas split — [plan-graph-canvas-split.md](plan-graph-canvas-split.md) |
| **PREF-1–4** | Argus menu, File extensions, Preferences modal, UX remap — [plan-app-preferences.md](plan-app-preferences.md) |
| **Docs** | [keyboard-shortcuts.md](keyboard-shortcuts.md), [architecture.md](architecture.md) |
| **B1** (branch) | Subgraph creation panel, `boundary_authoring`, RAM embedded params — [plan-boundaries.md](plan-boundaries.md) |
| **B2** | Canvas tab bar, per-tab isolation, drill/undo/save glue — [plan-boundaries.md](plan-boundaries.md) |
| **B3** | Inlet/Outlet inner authoring, parent port relay, multi-tab save — [plan-boundaries.md](plan-boundaries.md) |
| **B4** | CodeBox script authoring — creation panel, port inference, `ui` previews/buttons, rebind, GC — [plan-boundaries.md](plan-boundaries.md) · [plan-codebox-ux.md](plan-codebox-ux.md) |
| **AR1** | AssetRef unified `bundle` / `external` path params; dual-format migration; asset resolver; debug preview perf — [plan-asset-ref.md](plan-asset-ref.md) · merged [#41](https://github.com/yacfish/argus-modular/pull/41) |
| **B5** | Parent exposure — `visible_in_parent`, **↑** affordance, parent `SubgraphNode` surfacing — [plan-boundaries.md](plan-boundaries.md) · `c6453e1…` |
| **GP2** | `gl_viewer` GPU previews — merged PR #44 — [plan-gpu-pipeline.md](plan-gpu-pipeline.md) |
| **GP3** | GLSL effect runner + `glsl_*` palette (`GlslShader`) — [plan-gpu-pipeline.md](plan-gpu-pipeline.md) · PR #49 |
| **GP3.2** | Tier 2 GLSL effects (`hue_shift`, `threshold`, `vignette`, `sharpen`, `color_tint`) — core #59 — [plan-gpu-pipeline.md](plan-gpu-pipeline.md) |
| **GP4** | External folder shader packs — runtime load, exe + preferences search paths, palette expand — [argus-effect-format.md](argus-effect-format.md) · modular #50, core #61 + #62 |
| **JXS** | Max JXS → GP4 import — **150/150** runtime-verified, **paused** (good enough) — [plan-jxs-import.md](plan-jxs-import.md) |
| **GP5** | Local CI — `scripts/ci-local.sh` (modular smokes + core GLSL tests) — [ci-local.md](ci-local.md) · modular #50, core #60 |
| **Perf** | Media-graph UI fixes (palette cache, GLES/preview throttle); Release build docs; `ARGUS_PROFILE_TICK` profiler — README, AGENTS.md |

---

## Active / next (tracked)

| # | Track | Item | Status | Doc |
|---|-------|------|--------|-----|
| 15 | **GPU pipeline** | GP4 follow-ups — `color_rgb` / bundle zip | **(soon)** | [plan-gpu-pipeline.md](plan-gpu-pipeline.md) · [plan-color-parameters.md](plan-color-parameters.md) |
| 17 | **Color parameters** | `color_rgb` / `color_rgba`, auto-converters, optional picker (Node tab) | Not started | [plan-color-parameters.md](plan-color-parameters.md) |
| 18 | **Parameter widgets** | Widget registry; CodeBox `vec2` / `enum` / `int`/`float`; preview viewers — [plan-parameter-widgets.md](plan-parameter-widgets.md) | Not started | [plan-parameter-widgets.md](plan-parameter-widgets.md) |
| 16 | **Parameters** | `int`, `vec2`, `vec3`, `vec4` — core types + UI widgets | **(done)** | [plan-parameter-types.md](plan-parameter-types.md) |
| 11 | **LT polish** | `topology_changed` in SignalBridge; fatal Restart graph | Not started | [plan-live-topology.md](plan-live-topology.md) LT-UI-4 |
| 14 | — | Timeline / automation | Not started | Bundle `timelines/` |

---

## Deferred (tracked, explicit no)

| Item | Rationale |
|------|-----------|
| `⌘⇧V` paste replace | Design before implement — [next_on_UX-C.md](next_on_UX-C.md) |
| Integrated Lua IDE | External editor + file watch — [plan-codebox-ux.md](plan-codebox-ux.md) |
| imgui-node-editor migration | Custom canvas sufficient |
| Multi-window / multi-graph | Single document v1 |
| Collaboration / cloud | Out of scope |

---

## Blocked on platform / core

| Item | Blocker |
|------|---------|
| GPU pipeline **VLK** spike | MoltenVK + shaderc on macOS 12 |
| GPU pipeline **MTL** | No core Metal nodes |
| `GlesComputeModule` UI | EGL — Linux CI |
| ANGLE EGL on macOS 12 | Homebrew `gn` build failure |

---

## Suggested workflow before merge

1. **Mac:** `./scripts/ci-local.sh --extended` in argus-core.
2. **Linux:** `./scripts/ci-local-remote.sh --screen` from Mac.
3. **UI:** `./scripts/ci-local.sh` in argus-modular; empty doc → add nodes → save → run → preview.

---

*Core roadmap: argus-core `docs/internal/roadmap.md`*