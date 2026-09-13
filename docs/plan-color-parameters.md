# Color parameters — `color_rgb` / `color_rgba`

**Track:** semantic color types + parameter converters + optional color picker (Node tab inspector)  
**Status:** Not started  
**Depends on:** [plan-parameter-types.md](plan-parameter-types.md) **(done)** — `vec3` / `vec4` on `main`  
**Related:** [plan-parameter-types.md](plan-parameter-types.md) · [plan-parameter-widgets.md](plan-parameter-widgets.md) · [plan-gpu-pipeline.md](plan-gpu-pipeline.md) GP4 · JXS import · `color_tint` consolidation

**Layer tags:** `[core]` = argus-core · `[app]` = argus-modular · `[core&app]` = both

---

## Goal

Add first-class **color** parameter types that behave like `vec3` / `vec4` in GLSL and JSON, but are authored and edited as colors:

- **`color_rgb`** — linear RGB, components `0.0`–`1.0`, binds `uniform vec3`
- **`color_rgba`** — linear RGBA, components `0.0`–`1.0`, binds `uniform vec4`

Users set values from the **Node tab inspector** (primary). An **optional color picker widget** replaces the per-component scrubber row when enabled. **Parameter auto-converters** normalize legacy and interchange formats without breaking saved bundles.

---

## Motivation

| Today | Problem |
|-------|---------|
| `color_tint` uses `tint_r`, `tint_g`, `tint_b` floats | Three unrelated knobs; not obviously a color |
| `vec3` scrubbers for RGB | Works but no swatch, no hex entry, no picker |
| JXS `vec3` defaults like `1 0 0` | Correct type, poor authoring UX for colors |

Color types are **semantic** wrappers around the shipped vec types — not a new GLSL uniform family.

---

## JSON schema (target)

### `color_rgb`

```json
{
  "name": "tint",
  "type": "color_rgb",
  "value": [1.0, 0.5, 0.25],
  "default": [1.0, 1.0, 1.0],
  "min": [0.0, 0.0, 0.0],
  "max": [1.0, 1.0, 1.0]
}
```

### `color_rgba`

```json
{
  "name": "fill",
  "type": "color_rgba",
  "value": [1.0, 1.0, 1.0, 0.5],
  "default": [1.0, 1.0, 1.0, 1.0]
}
```

| Field | Notes |
|-------|--------|
| `value` / `default` | JSON array length 3 or 4; clamp per component on `set_value` |
| `min` / `max` | Optional per-channel bounds (default full `0`–`1`) |
| Storage | Same array shape as `vec3` / `vec4`; `type` discriminates UI + converters |
| `ref.json` | Authoring field `default` as today; loader maps to core schema |

### Optional UI hint (bundle / node `ui`)

```json
{
  "ui": {
    "params": {
      "tint": { "widget": "color_picker", "visible": true }
    }
  }
}
```

When `widget` is omitted, Node tab falls back to the existing **vec scrubber row** (U1–U3). Color picker is **opt-in**, not mandatory for every `color_*` param.

---

## Parameter auto-converters `[core]`

Converters run at **load / `apply_params_object`** time (not graph wire `ConversionRegistry`).

| From | To | Rule |
|------|-----|------|
| `vec3` + `type: color_rgb` | `color_rgb` | Accept array; clamp |
| `color_rgb` | `vec3` uniform | Bind as `vec3` (no shader change) |
| `vec4` | `color_rgba` | Same pattern |
| Legacy `tint_r`, `tint_g`, `tint_b` keys | `color_rgb` `tint` | One-time migration helper on known effects |
| Hex string `"#RRGGBB"` / `"#RRGGBBAA"` | `color_*` | Optional in JSON import / paste |
| `vec3` type + `ui.widget: color_picker` | display as color | UI-only until type migration |

Registry location TBD: methods on `ParameterBag` or small `ParameterConverter` table keyed by `(from_type, to_type)` — mirror spirit of `ConversionRegistry` but for **param values**, not port types.

---

## Core steps `[core]`

| Step | Item |
|------|------|
| CR1 | `ColorRgbParameter`, `ColorRgbaParameter` — wrap vec storage + clamp |
| CR2 | `to_json()` / `from_json()` / `apply_params_object` — `color_rgb`, `color_rgba` |
| CR3 | `declare_parameter_from_json` + `GlslEffectRunner::bind_uniforms` — bind as `vec3` / `vec4` |
| CR4 | Parameter auto-converters (table above) |
| CR5 | Migrate `color_tint` → single `color_rgb` `tint` (+ bundle migration) |
| CR6 | Headless smoke — round-trip + converter fixtures |

---

## UI steps `[app]`

| Step | Item |
|------|------|
| CU1 | **Node tab** (`node_panel.cpp`) — `color_rgb` / `color_rgba` branches |
| CU2 | Optional `widget: color_picker` — ImGui `ColorEdit3` / `ColorEdit4` (float, 0–1) |
| CU3 | Canvas chrome — compact swatch or defer to inspector-only v1 |
| CU4 | Hex paste / copy optional (inspector tooltip or secondary field) |

**Primary surface:** Node tab inspector. Canvas in-node controls are secondary (CU3 may defer).

---

## Implementation order

1. **CR1–CR3** — core types + GLSL bind (same uniforms as vec)  
2. **CU1–CU2** — inspector + optional picker  
3. **CR4** — auto-converters + legacy tint migration  
4. **CR5** — `color_tint` effect consolidation  
5. **JXS import** — map Jitter color-like `vec3`/`vec4` params; optional `widget` hint in emitted `ref.json`

---

## Out of scope (this track)

- HSV / HSL parameter types  
- sRGB vs linear toggle (assume linear `0`–`1` like today)  
- LUT / palette asset refs  
- `mat3` / `mat4` color transforms  
- Mandatory color picker on every vec3 (picker is **optional** via `ui.widget`)

---

## Changelog

| Date | Summary |
|------|---------|
| 2026-07 | Plan opened; deferred from shipped parameter-types track |
