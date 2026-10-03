> **Note:** This is a read-only documentation mirror for **Argus Modular**.
> The full source code and active development happen in a private repository.

---

# argus-modular

Visual application for [argus-core](https://github.com/yacfish/argus-core): graph editor, bundles, live previews, and an always-on executor.

| Mode | What you get |
|------|----------------|
| **Operate** (default) | Graph ticks continuously; runtime controls, CPU previews, global monitor |
| **Edit** (`[EDIT]` menu) | Canvas wiring, floating node picker, Node tab inspector, undo/clipboard; executor keeps running |

Core owns graph logic, nodes, and the executor. This repo is **UI only** (Dear ImGui + GLFW).

## Screenshot

![Argus Modular UI](docs/screenshots/argus-modular-ui.jpeg)

Node graph with live OpenCV video player, grayscale processing, and dual `gl_window` previews.

## Depth, Kinect, and ML

These nodes are in the palette on current `main` (core pin `24038e9`):

| Node | Palette name | What it does |
|------|----------------|--------------|
| `KinectV1Source` | `k1_source` | Xbox 360 colour and depth. An empty `serial` claims a free camera and is filled in after the first open. Saving the bundle keeps that binding. The field is a text box in the Node tab. `fps` is a slider (0 keeps the camera rate) and the node shows a measured fps preview, like the other inputs. |
| `KinectV2Source` | `k2_source` | Same colour, depth, fps slider, and fps preview. Synthetic backend. |
| `DepthToPointCloud` | `depth_to_cloud` | `depth.u16` to an organized `pointcloud.xyz`. |
| `DepthFilePlayer` | `depth_player` | Plays a recorded folder: colour images, 16-bit depth PNGs in millimetres, `intrinsics.json`, `timestamps.txt`. |
| `DepthRecorder` | `depth_recorder` | Writes that same folder from a colour and depth pair. |
| `YoloDetector` | `yolo` | ONNX Runtime detection, segmentation, or pose (`task`). Needs `-DARGUS_ENABLE_ONNXRUNTIME=ON` in the core build. Segmentation masks travel on `detection.list` as COCO-style RLE. The timing preview is `process_ms`, the job duration, not a frame rate. |
| `MediaPipePose` | `pose` | BlazePose lite. Set `model_dir` to a folder with `pose_detection.onnx` and `pose_landmarks_detector_lite.onnx` (shipped in argus-core under `models/mediapipe`). Colour in, `skeleton.2d`, `skeleton.3d`, and an overlay preview out. The timing preview is `process_ms`. |
| `SkeletonOverlay` | `skeleton` | Draws a `skeleton.2d` pose onto a colour frame. It does not run a model. |

`ML/YoloCamera`, `ML/PoseCamera`, and the Depth templates under `templates/` are the bundled examples. The longer plan is docs/dev-docs/plan-new-modules.md (only available in the main repo).


**Docs**

| Doc | Contents |
|-----|----------|
| [docs/roadmap.md](docs/roadmap.md) | What to build next |
| [docs/architecture.md](docs/architecture.md) | System map — **(done)** / **(soon)** / **(deferred)** |
| [docs/keyboard-shortcuts.md](docs/keyboard-shortcuts.md) | Keyboard cheat sheet |
| docs/plan-live-topology.md (only available in the main repo) | Live topology — shipped + remaining polish |
| docs/plan-boundaries.md (only available in the main repo) | SubgraphNode + CodeBox — B1–B5 done |
| docs/plan-codebox-ux.md (only available in the main repo) | CodeBox authoring spec (B4 — shipped) |
| docs/codebox-lua-api.md (only available in the main repo) | CodeBox Lua API |
| docs/plan-app-preferences.md (only available in the main repo) | Argus menu, prefs, File extensions |
| docs/plan-graph-canvas-split.md (only available in the main repo) | Canvas module split (shipped) |
| docs/plan-gpu-pipeline.md (only available in the main repo) | GPU path, `gl_viewer`, GLSL lib, external packs |
| docs/argus-effect-format.md (only available in the main repo) | External folder shader pack format (GP4) |
| docs/ci-local.md (only available in the main repo) | Local smoke CI (`scripts/ci-local.sh`) |
| docs/plan-new-modules.md (only available in the main repo) | Depth, Kinect, YOLO, and the modules still to build |
| docs/plan_initial.md (only available in the main repo) | Bootstrap history (U1–U5) |

---

## Prerequisites

- CMake 3.25+, Ninja or Make, C++20 compiler
- OpenCV (same path as your argus-core build)
- argus-core — pinned in `cmake/resolve_argus_core.cmake` (`ARGUS_CORE_REF` = `24038e9…`, ML `process_ms` preview, core #85)

CMake resolves core in this order:

1. `-DARGUS_CORE_DIR=/path/to/argus-core` (local dev checkout)
2. git submodule `third_party/argus-core`
3. `$ENV{ARGUS_CORE_DIR}`
4. **FetchContent** from GitHub at `ARGUS_CORE_REF`

Fresh clone with no submodule fetches the pinned SHA automatically.

**Local dev** (sibling checkout):

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DARGUS_CORE_DIR=/path/to/argus-core
cmake --build build -j
```

**When argus-core ships a fix you need:** bump `ARGUS_CORE_REF` in this repo in the same UI PR (or immediately after core merges). Cursor agents are instructed to do this without being asked — see AGENTS.md (only available in the main repo).

```bash
./scripts/bump_argus_core_ref.sh -ud -m "short note" -cp   # merged on main: bump, docs, commit, push
./scripts/bump_argus_core_ref.sh -l -m "wip feature"        # local core only, pre-merge
```

---

## Build

**Always pass `-DCMAKE_BUILD_TYPE=Release`** (or `RelWithDebInfo`). A bare `cmake -S . -B build` leaves `CMAKE_BUILD_TYPE` empty on single-config generators (Ninja, Make) — the compiler runs with **no `-O` flags**. Media graphs then look ~10× slower than they should (e.g. ~2 fps instead of ~25 fps on a 1080p movie→OpenCV→GPU bundle) because per-pixel CPU work in this repo and linked core is unoptimized. OpenCV/ffmpeg decode can still look fine since those ship as optimized binaries.

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DARGUS_CORE_DIR=/path/to/argus-core   # omit if using FetchContent pin
cmake --build build -j
```

After reconfiguring, confirm:

```bash
grep CMAKE_BUILD_TYPE build/CMakeCache.txt
# expect: CMAKE_BUILD_TYPE:STRING=Release
```

`Debug` is for stepping through UI code only; expect slow in-node previews on movie/GPU graphs (see `cmake/argus_core_debug_perf.cmake` — core nodes stay at `-O2` in Debug UI builds).

Interactive build flags (window nodes, OpenCV HighGUI/videoio) match a desktop app — see root `CMakeLists.txt`.

**Smoke CI** (all `argus_test_*_smoke` binaries, Release):

```bash
./scripts/ci-local.sh
```

See docs/ci-local.md (only available in the main repo).

### Tick profiler (opt-in)

Per-node executor timing is available in **argus-core** when built with the current tree. Zero overhead when unset.

```bash
ARGUS_PROFILE_TICK=1 ./build/argus_ui --bundle /path/to/show.bundle
```

Once per second on **stderr**:

```
[tick-profile] ticks=27 avg_tick_ms=34.4
    OpenCVMoviePlayer: 6.88 ms/tick (1 calls/tick)
    CpuToGlesUpload: 6.81 ms/tick (1 calls/tick)
    ...
```

Use this when `ui_fps` ≈ `graph_fps` and you need to see which node type dominates tick time. Rebuild **both** ArgusModular and ArgusCore (`-DCMAKE_BUILD_TYPE=Release`) after changing profiler code.

---

## Run

```bash
./build/argus_ui
./build/argus_ui --bundle /path/to/show.bundle
```

Starts with an **empty document** if no bundle path is given. The executor **auto-starts** when a document is open. Use **File → Open bundle** or **Save as** for a new project.

Test fixture (from argus-core):

```
third_party/argus-core/examples/bundles/demo.bundle/
```

---

## Layout

```
src/
  app.*                  GLFW + ImGui lifecycle, [EDIT] toggle, executor poll
  graph_session.*        Graph, bundle I/O, presets, live topology wrappers
  signal_bridge.*        Core SignalBus → UI (params, previews)
  graph_canvas.*         Nodes, wires, picker, clipboard, undo
  boundary_authoring.*   Subgraph RAM embeds, save materialize, tab labels
  canvas_tabs.*          Subgraph tab bar registry + chrome
  subgraph_creation_panel.*  Embedded vs load-from-disk on new SubgraphNode
  edit_mode_panel.*      Side tabs: Monitor · Node · Presets
  preferences_panel.*    Preferences modal (Debug, UX remap)
  node_panel.*           Node tab inspector (params, previews, visibility)
  preview_view.*         CPU preview textures
  global_monitor_panel.* Full-size monitor tab
cmake/
  argus_link_builtin_nodes.cmake
docs/
  roadmap.md, architecture.md, plan-*.md
```

**Shell:** canvas left · side column right (Monitor / Node / Presets) · `[EDIT]` in menu bar · Debug in Preferences.

---

## Core docs (external)

| Topic | Location |
|-------|----------|
| UI design | argus-core `docs/internal/plan-graph-editor-ui.md` |
| Live topology (engine) | argus-core `docs/internal/plan-live-topology.md` |
| Bundles | argus-core `docs/internal/plan-app-bundle.md` |
| Local CI | argus-core `docs/ci-local.md` |

---

## License

| Component | License |
|-----------|---------|
| argus-core | Private |
| Dear ImGui | MIT |
| GLFW | zlib/libpng-style |
| argus-modular | Private |