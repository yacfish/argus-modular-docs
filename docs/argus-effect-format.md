# External shader packs (GP4)

Folder-based shader packs for runtime-loaded `GlslShader` effects. Each pack is one directory; the **folder name** is the canonical effect id (`params.shader` / palette `glsl_<id>`).

## Layout

```
shaders/packs/
  demo_invert/
    ref.json        # metadata (parameters, ports, category)
    vertex.glsl     # optional — omit to use Argus built-in fullscreen quad VP
    fragment.glsl   # required
```

Future: same layout inside a `.bundle` archive or zip; only the container changes.

## ref.json (v1)

```json
{
  "argus_effect": "1",
  "id": "demo_invert",
  "category": "external",
  "description": "Short label for picker / docs",
  "inputs": [
    { "name": "in", "type": "gles.texture", "role": "hot" }
  ],
  "outputs": [
    { "name": "out", "type": "gles.texture" }
  ],
  "parameters": [
    { "name": "amount", "type": "float", "default": 1.0, "min": 0.0, "max": 2.0 },
    { "name": "mode", "type": "int", "default": 0, "min": 0, "max": 1 },
    { "name": "zoom", "type": "vec2", "default": [1.0, 1.0], "min": [0.1, 0.1], "max": [4.0, 4.0] }
  ]
}
```

| Field | Notes |
|-------|--------|
| `argus_effect` | Schema version (`"1"`) |
| `id` | Informational; **folder name wins** if they differ (stderr warning on load) |
| `inputs` / `outputs` | Optional — default single `in` (hot) / `out`; JXS composites may declare `in` + `tex1`… (`hot`/`cold`) |
| `parameters` | Optional — `float`, `bool`, `int`, `vec2`, `vec3`, `vec4` |
| `import_meta` | Optional — importer-only: `argus_ready`, `warnings` (loader ignores today) |
| `passes` | Optional — default single pass; see built-in `GlslEffect` pass schema |

GLSL filenames are **fixed** (`vertex.glsl`, `fragment.glsl`) — not listed in JSON.

## Discovery

On `register_builtin_glsl_effects()` / `discover_external_shader_packs()`:

1. Built-in catalog registers first (GP3).
2. External packs load from (first match wins per pack id; built-ins beat externals):
   - `<argus_ui executable>/shaders/packs/` (staged at build time; primary)
   - `shaders/packs/` found by walking up from the process **current working directory** (`main` sets cwd to the executable directory)
   - **Preferences → Documents → Search paths** — each path is tried as `<path>/shaders/packs/` and as `<path>` (pack root)
   - `ARGUS_SHADER_PACK_DIR` — optional dev override (`:`-separated pack roots)

No terminal environment variables are required for normal use. Add a search path in preferences (e.g. your argus-core checkout or a custom packs folder) or rely on packs copied next to the binary at build time.

Example preference search path:

```
/Users/you/ArgusCore          → loads .../ArgusCore/shaders/packs/demo_invert/
/path/to/my-packs-root        → loads child folders like rota2/ directly
```

Pack folder layout under a root:

```
packs-root/
  demo_invert/
    ref.json
    fragment.glsl
    vertex.glsl
```

External ids that collide with a built-in id are **skipped** (built-in wins).

## Palette

Same as built-ins: UI expands `GlslShaderLibrary::effect_catalog()` → `glsl_<id>` entries (`demo_invert` → `glsl_demo_invert`).

## JXS import

See [plan-jxs-import.md](plan-jxs-import.md). `scripts/import_jitter_shaders.py` (argus-core) emits:

- transpiled `fragment.glsl` (+ optional `vertex.glsl`)
- `ref.json` from JXS `<param>` metadata
- `import_meta` — `argus_ready` + warnings (`multi_sampler`, `rect_pixel_coords`, …)
- output folder name = path slug (e.g. `color_cc_brcosa`)

## Example

Shipped with argus-core: [shaders/packs/demo_invert/](https://github.com/yacfish/argus-core/tree/main/shaders/packs/demo_invert) (invert, id mismatch in ref.json to exercise folder-name rule).
