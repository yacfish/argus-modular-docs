# JXS → Argus shader pack import

**Track:** dev tooling — convert Cycling '74 Jitter `.jxs` into GP4 folder packs  
**Status:** **Good enough for now** — J1–J4.5 + J4.3 shipped; **150/150** runtime-verified (July 2026). Further import waves **paused** — resume from § Remaining when needed.  
**Depends on:** GP4 runtime packs **(done)** · [plan-parameter-types.md](plan-parameter-types.md) **(done)**  
**Related:** [argus-effect-format.md](argus-effect-format.md) · [plan-gpu-pipeline.md](plan-gpu-pipeline.md) GP4 follow-ups

**Script:** `argus-core/scripts/import_jitter_shaders.py`  
**Output:** `shaders/packs/<slug>/ref.json` + `fragment.glsl` [+ optional `vertex.glsl`]

---

## Goal

Import community / legacy Jitter shaders without hand-rewriting `ref.json` and GLSL for Argus `GlslShader` nodes.

One `.jxs` in → one pack folder out, discoverable like `demo_invert`.

---

## JXS inputs (reference)

[Jitter JXS format](https://docs.cycling74.com/userguide/jitter/jxs_file_format/) — XML root `<jittershader>`:

| Section | Import use |
|---------|------------|
| `<description>` | `ref.json` `description` |
| `<param>` without `state` | User parameters → `ref.json` `parameters` |
| `<param state="…">` | **Skip** — engine uniforms (matrices, `jit_position`, etc.) |
| `<param type="int">` + sampler bind | **Skip** — texture inlet (Argus single `in` hot texture) |
| `<language>` / `<program>` | GLSL sources (CDATA or `source="file.glsl"`) |
| `<bind param="…" program="fp|vp">` | Map params to programs; detect texture samplers |

### JXS → Argus param types

| JXS `type` | Argus `ref.json` `type` | Notes |
|------------|-------------------------|--------|
| `float`, `double` | `float` | |
| `int` (numeric) | `int` | Not texture index |
| `bool` | `bool` | |
| `vec2`–`vec4` | `vec2`–`vec4` | Space-separated `default` |
| `ivec*`, `bvec*`, `dvec*` | `int` / `bool` / `float` or skip | Best-effort v1 |
| `mat2`–`mat4` | **skip** | Out of scope |

Future: `vec3`/`vec4` with color semantics → [plan-color-parameters.md](plan-color-parameters.md).

---

## Pack slug (v2)

Folder id = **path under import root**, not `jittershader@name` alone:

| JXS path | Pack id |
|----------|---------|
| `color/cc.brcosa.jxs` | `color_cc_brcosa` |
| `math/op.sin.jxs` | `math_op_sin` |

Avoids collisions (199 Max shaders → 199 unique folders).

---

## GLSL transpilation

Argus single-input effects use a **built-in fullscreen quad VP** when `vertex.glsl` is omitted (`v_uv` → `sampler2D`).

| Jitter / legacy | Argus action | Status |
|-----------------|--------------|--------|
| External VP `sh.passthru*.vp.glsl`, `sh.passthrudim.vp.glsl` | **Omit** `vertex.glsl`; resolve from `shaders/shared/glsl/` | **(done)** |
| Custom VP (`op.binary.vp.glsl`, …) | Emit `vertex.glsl` | **(done)** — may not run without manual fix |
| `sampler2DRect` (uniforms **and** function params) | → `sampler2D` | **(done)** J3.1 |
| `texture2DRect` / `textureRect` | → `texture` | **(done)** J3.1 |
| `varying vec2 texcoord0` | → `in vec2 v_uv` | **(done)** J3.1 |
| Multi-texture composites (`tex0`/`tex1`, …) | `inputs` + `passes.samplers`; `texcoord0/1` → `v_uv` | **(done)** J4.1 |
| `gl_FragColor` / `outColor` | → `fragColor` | **(done)** |
| `#version 330 core` | → `#version 300 es` | **(done)** |
| `jit_PerVertex { … }` | → `in vec2 v_uv` | **(done)** |
| Pixel-space Rect UV offsets (FXAA, HDR taps) | `textureSize` / `argus_rect_texel()` normalization | **(done)** J3.2 |
| `texdim0` / passthrudim pixel scale | Strip `*texdim0`; normalized `v_uv` taps | **(done)** J3.3 |
| `varying vec2 TC`, `Pdev` | → `v_uv` (tiles uses `Pdev.xy`) | **(done)** J3.3 |
| Inline gaussian/median/kaleido/resample/sinefold VP | Fragment rewrites from `v_uv` + `argus_rect_texel()` | **(done)** J3.3 |
| `gl_TexCoord` / 3D inline VP | `custom_vertex_vp` → `argus_ready: false` | **(done)** J3.3 |

**Non-goals v1:** geometry shaders, `jit.gl.pix` gen patches, full matrix lighting stacks, multi-pass MRT graphs.

---

## `import_meta` in `ref.json`

Importer adds compatibility metadata (loader ignores unknown fields today):

```json
"import_meta": {
  "source_jxs": "hdr.downsample.filter.jxs",
  "argus_ready": true,
  "warnings": ["rect_pixel_coords", "mrt_category"]
}
```

| Field | Meaning |
|-------|---------|
| `argus_ready` | `false` when `multi_sampler` (>3 inlets), `multi_texture_inlet`, `geometry_shader`, `custom_vertex_vp`, `mrt_category`, `external_3d_vp`, or `volume_sampler` |
| `warnings` | `rect_pixel_coords`; `custom_vertex_vp`; `external_3d_vp`; `mrt_category`; `volume_sampler` (`sampler3D` / `texture3D`) |

Future: palette / discovery skips packs with `argus_ready: false` at load time (`load_pack_directory` in core).

---

## CLI

```bash
cd argus-core
python3 scripts/import_jitter_shaders.py path/to/effect.jxs -o shaders/packs/my_effect
python3 scripts/import_jitter_shaders.py /path/to/jitter/shaders -o shaders/packs --batch --recursive
```

| Flag | Purpose |
|------|---------|
| `-o DIR` | Output pack root |
| `--batch` | Import every `.jxs` in input directory |
| `--recursive` | Include subdirectories (required for Max bundle layout) |
| `--dry-run` | Print `ref.json` + GLSL to stdout |
| `--jxs-root` | Resolve relative `source="…"` beside each JXS |
| `--glsl-search DIR` | Extra GLSL include root (repeatable) |

---

## Steps

| Step | Item | Status |
|------|------|--------|
| J1 | Parse JXS XML — params, programs, binds | **(done)** |
| J2 | Emit `ref.json` v1 + path slugs | **(done)** |
| J3 | FP transpile — Rect types, `gl_FragColor`, version | **(done)** |
| J3.1 | Rect in function params + `texture2DRect` | **(done)** |
| J3.2 | Pixel-space UV normalization for Rect helpers | **(done)** |
| J3.3 | Remaining `rect_pixel_coords` on `argus_ready` packs — texdim0, TC/`Pdev`, gaussian/median VP, false-positive centering; `custom_vertex_vp` for 3D inline VP | **(done)** |
| J4.1 | Multi-texture composites — 2–3 inlets, `u_input` + `u_texN`, hot/cold roles | **(done)** |
| J4 | VP strategy — omit passthrough / emit custom | **(done)** |
| J5 | Fixture + unit tests | **(done)** |
| J6 | Max bundle smoke (199 JXS → 199 packs) | **(done)** |
| J7 | Palette respects `import_meta.argus_ready` | **(done)** |
| J8 | Runtime pack validation — `argus_validate_import_packs` + `validate_jxs_import.sh` | **(done)** |
| J4.2 | Generator inline-VP lighting → fragment rewrites (`stripes`, `bricks` + AA) | **(done)** |
| J4.5 | Runtime-fail fix wave — tighten `argus_ready` + importer gaps | **(done)** 143/143 execute_ok |
| J9 | Optional: `vec3` → `color_rgb` hint | Deferred — [plan-color-parameters.md](plan-color-parameters.md) |

### Max bundle smoke (July 2026)

Path: `/Applications/Max.app/.../media/jitter/shaders`

| Metric | Result |
|--------|--------|
| JXS files | 199 |
| Packs written | 199 |
| `sampler2DRect` left in FP | **0** |
| `import_meta.argus_ready: false` | **49** (custom VP, geometry, MRT, volume, >3 samplers) |
| `rect_pixel_coords` warning | **86** total; **0** on `argus_ready` packs |
| `argus_ready: true` | **150** (+2 generators spiderweb/crosstile) |
| Runtime `execute_ok` | **150** (validated headless GLES) |

### Runtime validation (July 2026)

Headless GLES gate: `argus-core/build/examples/argus_validate_import_packs` — compile + one execute pass per pack (gray dummy textures, default params).

```bash
# Fixture smoke (CI)
argus-core/scripts/validate_jxs_import.sh

# Full Max reimport tree (dev)
argus_validate_import_packs /private/tmp/argus_jxs_max_import_v4
```

| Metric | Importer (`ref.json`) | Runtime (GLES) |
|--------|----------------------:|---------------:|
| Total packs | 199 | 199 |
| Candidates | 163 `argus_ready` | same filter |
| **Actually runs** | — | **150** (matches `argus_ready`) |

`argus_ready` is necessary and sufficient after J4.5 (July 2026).

---

## J4.5 — Runtime-fail fix wave

**Goal:** `execute_ok == argus_ready` on the Max import tree (163/163), measured by `argus_validate_import_packs`.

**Strategy:** Fix what the importer can rewrite; downgrade `argus_ready` for packs that need runtime features Argus does not have (MRT, 3D volumes, full material lighting).

### Wave A — `argus_ready` tightening (honest exclusions)

Block additional warning tags in `import_meta` (palette already skips `false`):

| Add to `blocked` set | Reason | Packs |
|----------------------|--------|------:|
| `mrt_category` | `gl_FragData`, multi-output, `gl_LightSource` | 4 |
| `external_3d_vp` (new) | Emitted `vertex.glsl` with mesh lighting varyings (`P`, `N`, `T`, …) + FP reads them | 2 materials + 2 generators* |

\* `generator_gn_spiderweb`, `generator_gn_crosstile` — until Wave D rewrites FP to `v_uv`.

**Outcome:** −8 false positives from the 163 set (155 honest ready). Materials/MRT stay importable on disk but hidden from palette.

### Wave B — `texdim0` remnants (importer)

`strip_texdim0_scales` misses patterns where `texdim0` is used in `mod`, `mix`, or as a standalone varying.

| Pack | Fix |
|------|-----|
| `transition_tr_dissolve` | `mod(v_uv, texdim0)` → `fract(v_uv)` |
| `transition_tr_vignettes` | `texdim0*scale` → `scale` (normalized UV) |
| `texdisplace_td_repos` | `mod(coord,texdim0)` / fold → normalized equivalents |
| `mrt_mrt_motion_glitch` | `vtc *= texdim0` → drop scale |
| `generator_gn_gloop` | remove `varying texdim0`; `v_uv/texdim0` → `v_uv` |

**Tests:** unit cases in `test_import_jitter_shaders.py` per pattern. **+5 execute_ok** (if not downgraded in A).

### Wave C — Convolution tap grid

| Pack | Fix |
|------|-----|
| `convolution_cf_laplace` | Extend `detect_convolution_vp_rewrite` for `texcoord01`…`texcoord21` 3×3 laplace grid → `argus_rect_texel()` offsets from `v_uv` |

**+1 execute_ok**.

### Wave D — Generator FP rewrites (extends J4.2)

| Pack | Fix |
|------|-----|
| `generator_gn_gloop` | (also Wave B) |
| `generator_gn_spiderweb` | Rewrite FP: `T.st` → `v_uv`; drop `Cs` lighting dependency |
| `generator_gn_crosstile` | Rewrite FP: `N`/`T` → `v_uv` procedural tiling |

Strip or omit incompatible `vertex.glsl` when FP is fully procedural. **+3 execute_ok** (or keep in ready set if Wave A deferred).

### Wave E — Math / builtins / legacy IO

| Pack | Root cause | Fix |
|------|------------|-----|
| `math_op_and`, `math_op_or` | `bvec4` arithmetic invalid in GLSL ES 3.0 | Rewrite to `notEqual(a,zero) && notEqual(b,zero)` / logical OR variant |
| `math_op_invsqrt` | Jitter `invsqrt()` builtin | `1.0 / sqrt(x)` |
| `color_cc_exrdisplay` | `In.TEX0.xy` jit varyings struct | → `v_uv` |
| `vertdisplace_vd_gnoise_2d`, `vd_gnoise_3d` | `gl_Color` from VP | → `vec4(1.0)` or derive from `v_uv` |

**+6 execute_ok**.

### Wave F — Texdisplace / deferred

| Pack | Action |
|------|--------|
| `texdisplace_td_plane3d` | `texture3D` + `texture2D` — **downgrade** (`volume_sampler` warning); needs 3D texture asset |
| `texdisplace_td_rota` | Complex bound-check macro block — rewrite or **downgrade** after audit |

### Execution order

```
J8 (validator) → A (tighten) → B+C+E (importer) → D (generators, overlaps J4.2) → F (defer/downgrade)
```

### Acceptance

```bash
python3 scripts/import_jitter_shaders.py --batch --recursive \
  -o /tmp/argus_jxs_validate \
  /Applications/Max.app/.../media/jitter/shaders

build/examples/argus_validate_import_packs /tmp/argus_jxs_validate
# expect: execute_ok == candidates == argus_ready count
```

Wire full-tree gate into CI once green (`ARGUS_JXS_IMPORT_ROOT` required in BedroomPC reimport job).

---

## Paused — good enough for now (July 2026)

**Decision:** Stop active JXS import work. **150/150** `argus_ready` packs (all runtime-verified) is sufficient for palette breadth; remaining 49 packs need features Argus does not have yet (MRT, geometry shaders, mesh VP, 3D volumes) or per-shader manual effort with low ROI.

**What you have today:**

| Category | Ready | Notes |
|----------|------:|-------|
| All Max JXS | 199 imported | Full tree at prefs `search_paths` or `/private/tmp/argus_jxs_max_import_v4` |
| Palette-usable (`argus_ready`) | **150** | Headless GLES compile + execute |
| Generators | **11/11** | Including gnoise, spiderweb, crosstile |
| Blocked honestly | 49 | `import_meta.argus_ready: false` |

**Resume later (if needed):** Waves E/F leftovers — `fx_blobby`, texdisplace/vertdisplace custom VP, material `mat_*` procedural rewrites; MRT/geometry/volume remain **deferred** until runtime support exists.

---

## Test plan

- [x] Unit tests — `scripts/test_import_jitter_shaders.py`
- [x] Max bundle — 199/199 packs, no slug collisions
- [x] Import single-inlet slab → pack loads via `discover_external_shader_packs` (prefs search path + palette)
- [ ] Params round-trip in Node tab
- [x] Palette filter on `import_meta.argus_ready`
- [x] Runtime validation — `argus_validate_import_packs` on fixture packs (CI)
- [x] Full Max tree — `execute_ok` matches `argus_ready` (J4.5)

---

## Changelog

| Date | Summary |
|------|---------|
| 2026-07 | **Paused** — good enough at **150/150**; active import work stopped |
| 2026-07 | **Shipped** J4.3 — spiderweb/crosstile generator rewrites; **150/150** runtime gate |
| 2026-07 | **Shipped** gnoise gradperm pack asset; **148/148** runtime gate |
| 2026-07 | **Shipped** J4.2 — generator stripes/bricks lighting rewrites; **147/147** runtime gate |
| 2026-07 | **Shipped** J4.5 C/E/F — cross5 laplace, logical ops, invsqrt, exrdisplay, gl_Color, td.rota, volume_sampler; **143/143** runtime gate |
| 2026-07 | J4.5 A+B: block `mrt_category`/`external_3d_vp`; passthrudim texdim0; 144→143 ready after volume downgrade |
| 2026-07 | J8 runtime validator (`argus_validate_import_packs`); measured 142/163 execute_ok; J4.5 plan |
| 2026-07 | **Shipped** J4.1 on core `main` (#70); `argus_ready` 84→163 |
| 2026-07 | **Shipped** J3.3 on `main` (core #69, modular #56); ready+rect 19→0 |
| 2026-07 | **Shipped** J1–J3.2 on `main` (core #68, modular #55) |
| 2026-07 | J3.2 pixel UV normalization (`argus_rect_texel`); rect warnings 165→109 |
| 2026-07 | J3.1 Rect function params; `import_meta`; Max 199/199 smoke |
| 2026-07 | v2 slugs, VP resolve, fractional defaults, `--recursive` |
| 2026-07 | Plan opened; script on `feature/jxs-import` |
