# Parameter & viewer widgets — unified UI overhaul

**Track:** inspector + canvas chrome widgets, CodeBox `ui` schema, inner/port preview viewers  
**Status:** Not started — **deferred** after shipped [plan-parameter-types.md](plan-parameter-types.md)  
**Depends on:** [plan-parameter-types.md](plan-parameter-types.md) **(done)** on `main`  
**Related:** [plan-color-parameters.md](plan-color-parameters.md) · [plan-codebox-ux.md](plan-codebox-ux.md) · [codebox-lua-api.md](codebox-lua-api.md) · [argus-effect-format.md](argus-effect-format.md) · [plan-gpu-pipeline.md](plan-gpu-pipeline.md)

**Layer tags:** `[core]` = argus-core · `[app]` = argus-modular · `[core&app]` = both

---

## Goal

One coherent **parameter + preview widget** system across:

- Built-in nodes (Node tab + optional canvas chrome)
- GLSL / external shader packs (`ref.json` + `ui.params` overrides)
- **CodeBox** script `ui.params` / `ui.previews` (today the narrowest surface)

Ship missing **types** (vec2, enum, distinct int/float, colors) and **widgets** (combo, color picker, richer preview viewers) without each feature re-implementing ad hoc UI in `node_panel.cpp` / `node_canvas_chrome.cpp`.

---

## Motivation (today’s gaps)

| Area | Shipped | Gap |
|------|---------|-----|
| Core param types | `float`, `bool`, `string`, `int`, `vec2`–`vec4` | No `enum`, no semantic `color_*` ([plan-color-parameters.md](plan-color-parameters.md)) |
| CodeBox `ui.params` | `number`, `bool`, `string`, `path` only | No `vec2`, `int`, `float`, `enum`, `color_*`; `number` maps to `FloatParameter` only |
| CodeBox scripts | Workarounds (four floats for vec2 range; string for mode) | LFO / generator authoring friction |
| Widget selection | `ui.params.widget` hints (`slider`, `toggle`, `text`, `path`, `asset_ref`) | No `enum_combo`, `color_picker`, vec-specific layout policy |
| Preview viewers | `numbers`, `dict`, `text`, `cv_mat`, `image_auto`, … | No shared registry; CodeBox preview types lag built-in kinds |
| Type names | JSON `"type": "float"` in core; CodeBox v1 uses `"number"` | **Breaking:** CodeBox adopts `float` / `int` only — drop `"number"` (no alias; few scripts exist) |

Example pain (CodeBox LFO): vec2 range needs four `number` params; waveform mode is a free-form `string` instead of an enum combo.

---

## Target type system

### Shared JSON / core types (all node surfaces)

| Type | Storage | GLSL / wire | Notes |
|------|---------|-------------|--------|
| `float` | `FloatParameter` | `float` | Replaces CodeBox `"number"` — **no alias** |
| `int` | `IntParameter` | `int` | Discrete modes, counts, indices |
| `bool` | `BoolParameter` | `bool` | |
| `string` | `StringParameter` | — | Paths use `path` widget + AssetRef where applicable |
| `vec2` / `vec3` / `vec4` | `Vec*Parameter` | `vec*` | Already shipped for built-ins |
| `enum` | **new** `EnumParameter` | `int` uniform | String labels in UI; int value in JSON / ctl |
| `color_rgb` / `color_rgba` | **new** semantic colors | `vec3` / `vec4` | [plan-color-parameters.md](plan-color-parameters.md) |

### CodeBox `ui.params` (script declaration → runtime params)

Target parity with built-in schema (subset v1, grow over time):

```lua
ui = {
  params = {
    rate   = { type = "float", default = 0.25, min = 0, max = 20, in_node = true },
    mode   = { type = "enum", default = "sin", options = { "sin", "saw", "square" }, in_node = true },
    range  = { type = "vec2", default = { 0, 0 }, min = { -1, -1 }, max = { 1, 1 }, in_node = true },
    tint   = { type = "color_rgb", default = { 1, 1, 1 }, in_node = true },
    count  = { type = "int", default = 4, min = 1, max = 16, in_node = true },
  },
}
```

| CodeBox `type` | Maps to core | Lua `argus.param()` shape |
|----------------|--------------|---------------------------|
| `float` | `FloatParameter` | number |
| `int` | `IntParameter` | integer |
| `bool` | `BoolParameter` | boolean |
| `string` | `StringParameter` | string |
| `vec2` / `vec3` / `vec4` | `Vec*Parameter` | JSON array (Lua table array) |
| `enum` | `EnumParameter` | string label **or** int index (TBD — prefer label in Lua) |
| `color_rgb` / `color_rgba` | `Color*Parameter` | `{ r, g, b }` or array |

Optional **`widget`** override (same as bundle `ui.params` today): `slider`, `drag`, `toggle`, `enum_combo`, `color_picker`, `path`, …

---

## Widget registry (target architecture) `[app]`

Centralize widget dispatch instead of growing `if (type == …)` chains in two files.

| Concern | Today | Target |
|---------|-------|--------|
| Inspector rows | `node_panel.cpp` type branches | `draw_param_widget(profile, spec, value)` registry |
| Canvas chrome | `node_canvas_chrome.cpp` mirrors panel | Same registry, `ControlProfile::Canvas` |
| Visibility / layout | `control_visibility.cpp`, `ui.params.*` | Unchanged; registry reads resolved widget id |
| CodeBox merge | `boundary_authoring::merge_codebox_script_ui` | Map script `type` → default widget + core param ctor |

Suggested modules (names TBD):

- `param_widget_registry.hpp/cpp` — `(type, widget_hint) → draw + measure height`
- `preview_viewer_registry.hpp/cpp` — `(kind, viewer) → draw inner preview / monitor slot`

Built-in nodes, GLSL effects, and CodeBox all call the same registries.

---

## Preview / viewer overhaul `[app]`

| Step | Item |
|------|------|
| PV1 | Catalog existing `ui.previews` kinds + viewers (`text`, `sparkline`, `image_auto`, …) |
| PV2 | Registry API: measure + draw for in-node + monitor tab |
| PV3 | CodeBox `ui.previews` types map to same kinds (`number` → `numbers`, add `sparkline` opt-in) |
| PV4 | Optional: mini waveform / scope viewer for LFO debug (uses `publish_preview`) |

Defer **cv_mat** previews from CodeBox until image metadata API exists.

---

## Core steps `[core]`

| Step | Item |
|------|------|
| PW-C1 | `EnumParameter` — stored int + label table; JSON `{ type, value, options[] }` |
| PW-C2 | `CodeBox::sync_ui_params_from_declaration` — `int`, `float`, `vec2`–`vec4`, `enum` |
| PW-C3 | Lua sandbox — `argus.param` returns arrays for vec; enum as label string |
| PW-C4 | `on_param_change` / ctl merge — enum + vec JSON shapes |
| PW-C5 | Fold [plan-color-parameters.md](plan-color-parameters.md) CR1–CR4 under this track or keep as sub-milestone |
| PW-C6 | Smoke: script declares vec2 + enum → inspector round-trip → ctl dict |

---

## App steps `[app]`

| Step | Item |
|------|------|
| PW-A1 | Widget registry — extract from `node_panel` / `node_canvas_chrome` (float/int/bool/string/vec first) |
| PW-A2 | `enum_combo` widget — inspector + canvas |
| PW-A3 | Vec2 compact row — linked scrubbers (reuse parameter-types U1 layout) |
| PW-A4 | CodeBox script UI merge — new types + default widgets |
| PW-A5 | CodeBox: remove `"number"` type; `"float"` only — parser, stub template, API docs (breaking) |
| PW-A6 | Preview viewer registry + CodeBox preview type extensions |
| PW-A7 | Color picker widget — coordinate with [plan-color-parameters.md](plan-color-parameters.md) CU2 |

---

## Implementation order (suggested)

1. **PW-A1** — registry refactor (no new types; reduces duplicate code)  
2. **PW-C1, PW-A2** — `enum` end-to-end (unblocks CodeBox mode selectors)  
3. **PW-C2, PW-A3, PW-A4** — CodeBox `vec2` / `int` / `float`  
4. **PW-C5, PW-A7** — colors ([plan-color-parameters.md](plan-color-parameters.md))  
5. **PW-A6** — preview viewer unification  

Bump **argus-core** pin after each core slice lands ([AGENTS.md](../AGENTS.md) § Argus-core pin).

---

## Breaking changes (accepted)

CodeBox `ui.params` is **not** locked in — very few scripts exist. This track does **not** preserve v1 `"number"`:

- Param type `"number"` is **removed**; authors use `"float"` (and `"int"` where discrete).
- No parser alias, no migration shim in `sync_ui_params_from_declaration`.
- Update stub template, examples, and [codebox-lua-api.md](codebox-lua-api.md) in the same PR slice as PW-A5 / PW-C2.

Preview type `"number"` may rename to `"float"` or stay as a preview-only label — decide in PW-A6 (orthogonal to param typing).

---

## Out of scope (this track)

- HSV / HSL param types  
- sRGB vs linear toggle (assume linear 0–1 like today)  
- CodeBox `ui.params` for `mat3` / textures / asset refs beyond existing `path`  
- Integrated Lua IDE  
- imgui-node-editor migration  

---

## Acceptance (when done)

- [ ] CodeBox script can declare `vec2` range + `enum` mode without float/string workarounds  
- [ ] Inspector and canvas use the same widget registry for built-in + CodeBox params  
- [ ] `int` and `float` are distinct in CodeBox `ui.params`; `"number"` is gone (no compat alias)  
- [ ] `color_rgb` / optional picker available on built-ins and CodeBox (via color sub-plan)  
- [ ] Preview kinds share one viewer registry; CodeBox inner previews match built-in behavior  
- [ ] [codebox-lua-api.md](codebox-lua-api.md) type table updated; stub template uses enum + vec2 example  

---

## Changelog

| Date | Summary |
|------|---------|
| 2026-07 | Plan opened — deferred widget/type overhaul after parameter-types ship; captures CodeBox vec2/enum/color/int-float gaps |
| 2026-07 | Breaking stance: drop CodeBox `"number"`; `float` / `int` only (no alias) |
