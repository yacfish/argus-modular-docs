# Roadmap — argus-modular

**Updated**: October 2026  
**Purpose**: Single planning doc for the UI repo. System map: [architecture.md](architecture.md).

**Engine:** [argus-core](https://github.com/yacfish/argus-core) — pin `ba50fdc…` (`ARGUS_CORE_REF`, pose metric previews, core #89)  
**Bootstrap history:** plan_initial.md (only available in the main repo)  
**Feature tracks:** plan-new-modules.md (only available in the main repo) · plan-boundaries.md (only available in the main repo) · plan-asset-ref.md (only available in the main repo) · plan-gpu-pipeline.md (only available in the main repo) · plan-parameter-types.md (only available in the main repo) · plan-parameter-widgets.md (only available in the main repo) · plan-color-parameters.md (only available in the main repo) · plan-jxs-import.md (only available in the main repo) · plan-app-preferences.md (only available in the main repo) · plan-graph-canvas-split.md (only available in the main repo) · plan-port-tooltips.md (only available in the main repo)  
**UX polish history:** `next_on_UX-*.md` (UX-A–D, merged)  
**Live topology:** plan-live-topology.md (only available in the main repo) — core shipped; UI polish remains

~~plan-ui-ux.md (only available in the main repo)~~ — **closed**; merged here + architecture.

---

## Product principles

| Principle | Today | Target | Tag |
|-----------|-------|--------|-----|
| Executor while document open | Always-on; no manual transport UI | Same; `[EDIT]` is layout-only | **(done)** |
| Topology edits (interactive) | Live `add_node` / connect / hot-swap | Same | **(done)** |
| LT polish | — | `topology_changed`, fatal Restart graph | **(soon)** plan-live-topology.md (only available in the main repo) |
| Parameter tweaks | Live `set_parameter` | Same | **(done)** |
| One document | Single bundle | Same | **(done)** |

---

## Layout (shipped)

- **Canvas left** — graph editor (`GraphCanvas` + split modules per plan-graph-canvas-split.md (only available in the main repo)).
- **Side column** — **Monitor** · **Node** (inspector) · **Presets** (no Debug tab — Debug in **Preferences → Debug**).
- **Node tab = inspector** — all params, visibility, preview routing, parent **↑** exposure toggles (Boundaries).
- **Add nodes** — floating picker (`⌘N`, double-click empty canvas, double-click name to replace).
- **Menu `[EDIT]`** — enable/disable edit (UX-D).
- **Argus / File menus** — About, Preferences, recent bundles, templates, reload — plan-app-preferences.md (only available in the main repo).

**(deferred)** Docking, status bar — untracked § below.

---

## Keyboard shortcuts

[docs/keyboard-shortcuts.md](keyboard-shortcuts.md)

---

## Palette tiers (catalog)

| Tier | Scope |
|------|--------|
| **P0** | CPU vision, `SubgraphNode`, `CodeBox` — CPU loops **(done)**; Boundaries B1–B5 **(done)** plan-boundaries.md (only available in the main repo) |
| **P1** | GLES upload/shader/window — catalog **(done)**; **`glsl_*` palette + GP3/GP3.2 + external packs (GP4 runtime)** **(done)** plan-gpu-pipeline.md (only available in the main repo) |
| **P2** | BlobDetector, CentroidFilter, Sketch2D, movie recorder — in the palette **(done)** |
| **P3** | Vulkan, compute — **(deferred)** / build-gated; Inlet/Outlet subgraph-only |

---

## Completed

| Phase | Summary |
|-------|---------|
| U1–U5 | Bootstrap — plan_initial.md (only available in the main repo) |
| U6 partial | Selection, wiring, empty doc |
| **UX-A–D** | Shell, canvas UX, clipboard, panel polish — `next_on_UX-*.md` |
| **GC-1–GC-4** | Graph canvas split — plan-graph-canvas-split.md (only available in the main repo) |
| **PREF-1–4** | Argus menu, File extensions, Preferences modal, UX remap — plan-app-preferences.md (only available in the main repo) |
| **Docs** | [keyboard-shortcuts.md](keyboard-shortcuts.md), [architecture.md](architecture.md) |
| **B1** (branch) | Subgraph creation panel, `boundary_authoring`, RAM embedded params — plan-boundaries.md (only available in the main repo) |
| **B2** | Canvas tab bar, per-tab isolation, drill/undo/save glue — plan-boundaries.md (only available in the main repo) |
| **B3** | Inlet/Outlet inner authoring, parent port relay, multi-tab save — plan-boundaries.md (only available in the main repo) |
| **B4** | CodeBox script authoring — creation panel, port inference, `ui` previews/buttons, rebind, GC — plan-boundaries.md (only available in the main repo) · plan-codebox-ux.md (only available in the main repo) |
| **AR1** | AssetRef unified `bundle` / `external` path params; dual-format migration; asset resolver; debug preview perf — plan-asset-ref.md (only available in the main repo) · merged [#41](https://github.com/yacfish/argus-modular/pull/41) |
| **B5** | Parent exposure — `visible_in_parent`, **↑** affordance, parent `SubgraphNode` surfacing — plan-boundaries.md (only available in the main repo) · `c6453e1…` |
| **GP2** | `gl_viewer` GPU previews — merged PR #44 — plan-gpu-pipeline.md (only available in the main repo) |
| **GP3** | GLSL effect runner + `glsl_*` palette (`GlslShader`) — plan-gpu-pipeline.md (only available in the main repo) · PR #49 |
| **GP3.2** | Tier 2 GLSL effects (`hue_shift`, `threshold`, `vignette`, `sharpen`, `color_tint`) — core #59 — plan-gpu-pipeline.md (only available in the main repo) |
| **GP4** | External folder shader packs — runtime load, exe + preferences search paths, palette expand — argus-effect-format.md (only available in the main repo) · modular #50, core #61 + #62 |
| **JXS** | Max JXS → GP4 import — **150/150** runtime-verified, **paused** (good enough) — plan-jxs-import.md (only available in the main repo) |
| **GP5** | Local CI — `scripts/ci-local.sh` (modular smokes + core GLSL tests) — ci-local.md (only available in the main repo) · modular #50, core #60 |
| **Perf** | Media-graph UI fixes (palette cache, GLES/preview throttle); Release build docs; `ARGUS_PROFILE_TICK` profiler — README, AGENTS.md |
| **Depth / Kinect** | `KinectV1Source` (serial latch, fps cap), `KinectV2Source` synthetic, `DepthToPointCloud`, `DepthFilePlayer`, `DepthRecorder` — plan-new-modules.md (only available in the main repo) M0 |
| **ML** | `YoloDetector` (detect / segment / pose, CPU + CoreML), `MediaPipePose` lite ONNX. The pose overlay is `out3`. Inner previews are `process_ms`, `valid_joints`, and `hip_z_m` (core #89, modular #81). The Kinect reading is in plan-new-modules.md (only available in the main repo) M2 |
| **Port tooltips** | Hover labels in edit mode — plan-port-tooltips.md (only available in the main repo) · modular #77, #78 |
| **Stale nodes** | A card whose type is missing, or whose saved name is not the current default, opens as a pale orange port-less placeholder and its wires are dropped — core #88, modular #80 |

---

## Active / next (tracked)

| # | Track | Item | Status | Doc |
|---|-------|------|--------|-----|
| 19 | **ML / depth** | Metre check is good enough (`hip_z_m` 1.75 versus a 2.0 m tape, put down to the test setup). `pose.se3` and CPU SLAM wait. TensorRT and gsplat training stay off this Mac | Later | plan-new-modules.md (only available in the main repo) |
| 20 | **GL world** | `gl_world` plus draw nodes (`gl_plane`, `gl_pointcloud`, `gl_light_source`) on the existing GLES path. No scene-graph library | Next | plan-gpu-pipeline.md (only available in the main repo) |
| 15 | **GPU pipeline** | GP4 follow-ups — `color_rgb` / bundle zip | **(soon)** | plan-gpu-pipeline.md (only available in the main repo) · plan-color-parameters.md (only available in the main repo) |
| 17 | **Color parameters** | `color_rgb` / `color_rgba`, auto-converters, optional picker (Node tab) | Not started | plan-color-parameters.md (only available in the main repo) |
| 18 | **Parameter widgets** | Widget registry; CodeBox `vec2` / `enum` / `int`/`float`; preview viewers — plan-parameter-widgets.md (only available in the main repo) | Not started | plan-parameter-widgets.md (only available in the main repo) |
| 16 | **Parameters** | `int`, `vec2`, `vec3`, `vec4` — core types + UI widgets | **(done)** | plan-parameter-types.md (only available in the main repo) |
| 11 | **LT polish** | `topology_changed` in SignalBridge; fatal Restart graph | Not started | plan-live-topology.md (only available in the main repo) LT-UI-4 |
| 14 | — | Timeline / automation | Not started | Bundle `timelines/` |

---

## Deferred (tracked, explicit no)

| Item | Rationale |
|------|-----------|
| `⌘⇧V` paste replace | Design before implement — next_on_UX-C.md (only available in the main repo) |
| Integrated Lua IDE | External editor + file watch — plan-codebox-ux.md (only available in the main repo) |
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