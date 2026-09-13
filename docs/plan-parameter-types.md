# Parameter types — int, vec2, vec3, vec4

**Track:** core parameter model + UI widgets (all node types, GLSL effects first consumer)  
**Status:** **Shipped** on `main` (core [#63](https://github.com/yacfish/argus-core/pull/63), modular [#51](https://github.com/yacfish/argus-modular/pull/51) + [#52](https://github.com/yacfish/argus-modular/pull/52); pin `f9ac20a`)  
**Prerequisites:** GP4 runtime shader packs on `main`  
**Related:** [argus-effect-format.md](argus-effect-format.md) · [plan-gpu-pipeline.md](plan-gpu-pipeline.md) GP4 follow-ups · [plan-jxs-import.md](plan-jxs-import.md) · [plan-color-parameters.md](plan-color-parameters.md) (future)

**Layer tags:** `[core]` = argus-core · `[app]` = argus-modular · `[core&app]` = both

---

## Goal

Add first-class **`int`**, **`vec2`**, **`vec3`**, and **`vec4`** parameters end-to-end: core storage + JSON schema, GLSL uniform binding, palette defaults, Node tab + canvas widgets, external pack `ref.json`.

Unblocks Jitter/JXS shaders (`vec2 zoom`, `int boundmode`, RGBA tints as `vec4`, etc.) without treating everything as separate floats.

**Follow-on:** [plan-parameter-widgets.md](plan-parameter-widgets.md) — widget registry, CodeBox `ui` parity, enum, colors; [plan-color-parameters.md](plan-color-parameters.md).

---

## Today

| Piece | Status |
|-------|--------|
| `FloatParameter`, `BoolParameter`, `StringParameter` | `[core]` **(done)** |
| `IntParameter` | `[core]` **(done)** — C1 + C1.1 discrete float audit |
| `Vec2Parameter`, `Vec3Parameter`, `Vec4Parameter` | `[core]` **(done)** |
| `type: "int"` / `vec2` / `vec3` / `vec4` in UI | `[app]` **(done)** — inspector + canvas scrubbers |
| Palette defaults | `[app]` copies `schema["value"]` |
| External `ref.json` | `[core]` `float`, `bool`, `int`, `vec2`, `vec3`, `vec4` |
| GLSL uniforms | `[core]` float, bool, int, vec2/3/4 |

---

## JSON schema (target)

Same shape for built-ins, bundle `params`, palette `default_params`, and `ref.json` (after loader normalizes).

### `int`

```json
{ "name": "boundmode", "type": "int", "value": 0, "default": 0, "min": 0, "max": 4 }
```

### `vec2` / `vec3` / `vec4`

```json
{
  "name": "zoom",
  "type": "vec2",
  "value": [1.0, 1.0],
  "default": [1.0, 1.0],
  "min": [0.1, 0.1],
  "max": [4.0, 4.0]
}
```

| Field | Notes |
|-------|--------|
| `value` | Current value (palette seeds new nodes from this) |
| `default` | Factory default (equals `value` on a fresh node) |
| `min` / `max` | Per-component bounds; clamp on `set_value` |
| Arrays | Fixed length 2 / 3 / 4; reject wrong length on load |

`ref.json` keeps authoring field **`default`**; loader maps to core schema (`value` + `default` in `to_json()`).

---

## Core `[core]` — **(done)**

| Step | Item | Status |
|------|------|--------|
| C1 | `IntParameter` — mirror `FloatParameter` (value, default, min, max, clamp) | **(done)** |
| C1.1 | Audit discrete `FloatParameter` → `IntParameter` (table below) | **(done)** |
| C2 | `Vec2Parameter`, `Vec3Parameter`, `Vec4Parameter` | **(done)** |
| C3 | `ParameterBag::apply_params_object` / `to_params_object` — int + JSON arrays | **(done)** |
| C4 | `declare_parameter_from_json` — `int`, `vec2`, `vec3`, `vec4` | **(done)** |
| C5 | `GlslEffectRunner::bind_uniforms` — int + vec uniforms | **(done)** |
| C6 | `argus_test_parameter_types` smoke | **(done)** |
| C7 | Bump **argus-modular** `ARGUS_CORE_REF` | **(done)** |

### C1.1 — float → int audit `[core]`

Params that are **discrete** (counts, indices, enum modes, pixel coords) today use `FloatParameter` + `static_cast<int>` in `apply()`. Migrate after **C1** so schema `type` is `"int"` and UI shows integer scrubbers.

| Area | Param(s) | Notes |
|------|----------|--------|
| `glsl` **blend** | `mode` | `0` = over, `1` = multiply; update shader `uniform int u_mode` |
| **OpenCVCamera** | `device_index`, `width`, `height` | `fps` stays **float** |
| **OpenCV** flip | `flip_code` | `-1` / `0` / `1` |
| **OpenCV** crop / pad | `x`, `y`, `width`, `height`, `top`…`right` | pixel ints |
| **OpenCV** filter ops | `kernel_size`, `iterations`, `ksize`, `dx`, `dy`, `d` | odd sizes via `odd_kernel()` where needed |
| **OpenCV** color_map | `colormap` | `0`–`11` enum |
| **OpenCV** adaptive threshold | `block_size` | odd block size |
| **OpenCVWindow** | `wait_ms` | milliseconds |
| **PassthroughCpu** | `delay_ms` | milliseconds |
| **GlesMeshRenderer** | `width`, `height` | render target pixels; RGB stays float until vec3 |

**Keep float:** continuous knobs (brightness, gamma, mix, thresholds 0–1, sigma, angles, fps, etc.). **Defer vec:** `color_tint` `tint_r/g/b` → `vec3` in a follow-up within C2.

**Files (primary):** `include/argus/core/parameter.hpp`, `src/core/parameter.cpp`, `nodes/gles/glsl_effect_runner.cpp`, `nodes/gles/external_shader_pack.cpp`

---

## UI `[app]` — **(done)**

| Step | Item | Status |
|------|------|--------|
| U1 | `node_panel.cpp` — `int`, `vec2`, `vec3`, `vec4` widgets | **(done)** |
| U2 | `node_canvas_chrome.cpp` — same for in-node exposed params | **(done)** |
| U3 | `param_digit_drag.cpp` — `vector_field` (per-component scrub) | **(done)** |
| U4 | `node_palette.cpp` — no change needed | **(done)** |
| U5 | Manual smoke — palette + inspector round-trip | **(soon)** |

**Files (primary):** `src/node_panel.cpp`, `src/node_canvas_chrome.cpp`, `src/param_digit_drag.cpp`

---

## Implementation order

1. ~~Core C1–C5~~ **(done)**
2. ~~Modular U1–U4~~ **(done)**
3. ~~Docs — [argus-effect-format.md](argus-effect-format.md)~~ **(done)**
4. **GP4** — [JXS import](plan-jxs-import.md) maps Jitter params into this schema **(done)**
5. **Future** — [color parameter types](plan-color-parameters.md) (`color_rgb` / `color_rgba`, auto-converters, optional picker)

---

## Test plan

- [ ] New node from palette: params equal **`default`**, not `min` (regression for floats + new types)
- [ ] Bundle save/reload preserves `vec2` arrays
- [ ] External pack with `vec2` + `int` loads and appears in palette
- [ ] Live `set_parameter` updates GLSL uniform same frame
- [ ] Node tab + canvas controls stay in sync

---

## Follow-up (optional)

**`color_tint` consolidation** — see [plan-color-parameters.md](plan-color-parameters.md) (CR5: `color_rgb` `tint` + migration). Deferred from C1.1; vec3 scrubbers work today.

---

## Out of scope (shipped track — see linked plans)

- `mat3` / `mat4` uniforms  
- **Color types + picker** — [plan-color-parameters.md](plan-color-parameters.md)  
- Enum parameters (string + int)  
- CodeBox dynamic param declarations (separate track)

---

## Changelog

| Date | Summary |
|------|---------|
| 2026-07 | Shipped C1–C5 + U1–U4 on `main`; pin `f9ac20a` |
| 2026-07 | Plan opened; branches `feature/parameter-types` on core + modular |
