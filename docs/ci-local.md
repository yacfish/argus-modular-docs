# Local CI (argus-modular)

App smoke tests for roadmap **#10** and GPU pipeline **GP5** (modular slice).

```bash
./scripts/ci-local.sh
```

With a sibling argus-core checkout:

```bash
ARGUS_CORE_DIR=/path/to/argus-core ./scripts/ci-local.sh
```

## What it covers

| Step | Detail |
|------|--------|
| Configure | `-DCMAKE_BUILD_TYPE=Release` (required for representative perf) |
| Build | All targets in the chosen build dir (`build-ci-local` by default) |
| Run | Ten `argus_test_*_smoke` binaries (palette, boundaries, clipboard, prefs, assets, …) |

`argus_test_boundary_authoring_smoke` includes **GLSL palette** checks (`glsl_levels`, tier-2 `glsl_*` entries).

## GLES executor smoke (core)

Headless **movie → GlslShader → readback** graphs live in **argus-core** (`argus_test_gles_nodes`, `argus_test_glsl_effect_runner`). Run via:

```bash
cd ../ArgusCore && ./scripts/ci-local.sh --profile linux
```

(GP5 follow-up: add `argus_test_glsl_*` to core `ci-local.sh` / re-enabled GitHub workflow.)

**Done in argus-core** `ci-local.sh` / archived workflow — `argus_test_glsl_shader_library`, `argus_test_glsl_effect_runner`.

## Options

```
./scripts/ci-local.sh --skip-configure --skip-build   # re-run tests only
./scripts/ci-local.sh --build-dir build               # reuse main dev build tree
```

## Platform matrix

See [plan-gpu-pipeline.md](plan-gpu-pipeline.md) § Platform matrix (GL) — macOS 12 desktop GL primary; Linux Mesa EGL headless for core GLES CI.
