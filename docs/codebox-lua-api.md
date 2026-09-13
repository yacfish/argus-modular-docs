# CodeBox Lua API

**Status:** Spec — **first iteration** (B4 implements this contract)  
**UX:** [plan-codebox-ux.md](plan-codebox-ux.md) · **Boundaries:** [plan-boundaries.md](plan-boundaries.md) §4

Scripts declare their port surface in Lua. Core parses the script at **load / hot reload** and registers ports on the `CodeBox` node. Lua was chosen so authors avoid image processing in code; image **inputs** may exist as triggers only — no image **outputs** in v1.

---

## 1. Mandatory node surface (not declared in script)

Every `CodeBox` always has:

| Item | Role |
|------|------|
| `ctl` | Cold control inlet — key/value dict merges into node params (`enable`, path params, etc.) via `apply_control_dict` |
| `enable` | Standard param (default on) |

No default warm inlets or outlets. The script adds those.

---

## 2. Port declaration (evaluated at load)

Top-level tables in the script chunk:

```lua
inlets  = { argus.image, argus.dict }
outlets = { argus.dict }
```

| Constant | Direction (v1) | Meaning |
|----------|----------------|---------|
| `argus.image` | in only | Warm inlet — **trigger** on frame delivery; no pixel buffer in Lua |
| `argus.dict` | in or out | Warm inlet with dict body, or warm dict outlet |

**v1 rules:**

- `argus.image` in `outlets` → **load error**
- Unknown type tokens → load error
- Empty `outlets` is valid (logic-only / ctl-driven nodes)
- Port indices: warm inlets → `in0`, `in1`, …; outlets → `out0`, `out1`, …; **`ctl` always last** on canvas (cold, separate from warm indices)

On successful parse, core **reconfigures** the node’s warm ports. Hot reload may change the port list; incompatible wires are disconnected — see [plan-codebox-ux.md](plan-codebox-ux.md) §15.

---

## 3. Inlet handlers

One Lua function per declared warm inlet, by index:

```lua
function inlet_0()
  -- argus.image — trigger only (no image argument in v1)
end

function inlet_1(dict)
  -- argus.dict — Lua table (JSON-like)
end
```

Core calls the matching handler when that warm inlet receives data (push scheduler).

**Optional later:** `on_ctl(dict)` for custom ctl handling; v1 relies on standard param merge on `ctl`.

---

## 4. Params — read and change

### Read — `argus.param(key)`

```lua
local on = argus.param("enable")
local t  = argus.param("threshold")
```

Reads the current value from the node parameter bag (inspector, `ctl`, and defaults).

### CPU clock — `argus.cpu_time()`

```lua
local t = argus.cpu_time()  -- seconds (double)
local phase = t * argus.param("rate")
```

Returns monotonic **CPU clock time in seconds** — the same time base the app uses for the executor loop (`glfwGetTime()` in the UI). Updated every frame before graph ticks, so LFO phase stays correct when frame delivery slows under load.

**Not** frame index, **not** vsync interval — use this for time-based modulation triggered by `argus.image` inlets.

### Change — `on_param_change(key, val)`

Optional global handler. Core calls it when a key declared in `ui.params` changes:

```lua
function on_param_change(key, val)
  if key == "threshold" then
    print("threshold =", val)
  end
end
```

| Source | Calls `on_param_change`? |
|--------|-------------------------|
| Inspector / in-node widget edit | **Yes** — for `ui.params` keys |
| `ctl` dict merge | **Yes** — per changed `ui.params` key |
| Successful load / hot reload | **Yes** — once per `ui.params` key with its `default` (or type default) |
| `enable`, `script`, `script_ref` | **No** — use `argus.param` or inlet handlers |

Handler is optional; poll with `argus.param` when reactivity is not needed.

---

## 5. Outputs — `outlet(index, table)`

Publish a dict on declared outlet `index` (0-based among `outlets`):

```lua
outlet(0, { bypass = true, reason = "frame_pacing" })
```

- Not a Lua `return` — execution continues after `outlet`.
- Typical use: drive another node’s **`ctl`** inlet (control dict), e.g. `{ bypass = true }`.
- Invalid index or non-table → runtime error (surfaced in Node tab).

**No** image outlet in v1.

---

## 6. Inner previews — `publish_preview(key, value)`

Publish a value for a key declared in `ui.previews`:

```lua
publish_preview("rate", 12.5)
publish_preview("stats", { count = 12, bypass = true })
publish_preview("note", "ok")
```

- `key` must exist in `ui.previews`; unknown key → runtime error (Node tab).
- `value` type must match the declared preview `type` (`number`, `dict`, `text`).
- App reads via `CodeBox::sample_inner_preview(key)` (same C++ path as `OpenCVMoviePlayer::sample_inner_preview("fps")`).

Call from inlet handlers, `on_param_change`, or button handlers — whenever the script has new preview data.

---

## 7. Debug — `print(...)`

```lua
print("bypass =", bypass)
```

Routes to app log / stderr. Gated by app preference (e.g. **Debug → CodeBox script log**). Silent when off.

---

## 8. Example (stub template)

```lua
inlets  = { argus.image, argus.dict }
outlets = { argus.dict }

ui = {
  params = {
    threshold = { type = "number", default = 0.5, min = 0, max = 1, in_node = true },
  },
  buttons = {
    reset = "on_reset",
  },
  previews = {
    rate = { type = "number" },
  },
}

counter = 0

function on_param_change(key, val)
  if key == "threshold" then
    publish_preview("rate", val)
  end
end

function on_reset()
  -- button handler (name from ui.buttons)
end

function inlet_0()
  counter = counter + 1
  if counter >= 10 then counter = 0 end
  local bypass = counter >= 5
  outlet(0, { bypass = bypass })
  publish_preview("rate", counter)
  print("bypass =", bypass)
end

function inlet_1(dict)
  for key, val in pairs(dict) do
    print(key, val)
  end
end
```

---

## 9. Inspector UI — `ui` table (load time)

Optional top-level table. Parsed with `inlets`/`outlets`; drives Node tab and in-node chrome.

```lua
ui = {
  params = {
    threshold = { type = "number", default = 0.5, min = 0, max = 1, in_node = true },
    label     = { type = "string" },
    asset     = { type = "path" },
    verbose   = { type = "bool", default = false },
  },
  buttons = {
    reset = "on_reset",
  },
  previews = {
    stats = { type = "dict" },
    rate  = { type = "number" },
    note  = { type = "text" },
  },
}
```

| `type` (v1) | Param widget | Preview `kind` (app) |
|-------------|--------------|----------------------|
| `"number"` | slider / drag | — |
| `"string"` | text | — |
| `"path"` | bundle-relative / file picker | — |
| `"bool"` | toggle | — |
| preview `"number"` | — | `numbers` (`text` / `sparkline` viewer) |
| preview `"dict"` | — | `dict` (structured text; same family as monitor slots) |
| preview `"text"` | — | `default` (`text` viewer) |

**Deferred (not v1):** param types `int`, `float` (replaces v1 `"number"` — **no alias**), `vec2`–`vec4`, `enum`, `color_rgb` / `color_rgba` — [plan-parameter-widgets.md](plan-parameter-widgets.md).

Aligns with existing inner previews (e.g. `OpenCVMoviePlayer` `inner: fps` → `kind: numbers`, `ScalarF32`). **No** `cv_mat` previews from CodeBox in v1.

| Surface | Read / emit | React |
|---------|-------------|-------|
| Params | `argus.param("threshold")` | `on_param_change(key, val)` — optional; all `ui.params` keys |
| Buttons | — | global function named in `ui.buttons` value (e.g. `on_reset`) |
| Previews | app `sample_inner_preview(key)` | `publish_preview(key, value)` from script |

`in_node = true` sets default canvas visibility; user overrides in inspector (`ui.params.*.visible` in graph JSON, same as built-in nodes).

---

## 10. Core prototype (dev note — not user API)

ArgusCore today includes a **fixed-port** `CodeBox` prototype (`in1`/`in2`/`out0`/`out1`, `on_process()`, etc.) used for early headless tests and `overlay.lua` in demo bundles. That layout is **not** a shipped authoring API — B4 replaces it with this document. Demo scripts and tests are updated as part of B4; no dual-mode loader or user migration path.

---

## 11. Sandbox

- Stdlib: `table`, `string`, `math`, `utf8` — no `io` / `os` / arbitrary C API
- One VM per `CodeBox`
- Hot reload via bundle `watch.scripts` + `script_reloaded` signal