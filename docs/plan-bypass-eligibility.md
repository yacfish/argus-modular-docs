# Plan: Bypass eligibility (core + app)

**Status:** Complete — merged to `main` (argus-core PR #41, argus-modular PR #12, June 2026)  
**Repos:** ArgusCore (capability + relay), ArgusModular (UI visibility)  
**Phase 2 log:** [plan-bypass-eligibility-phase2.md](plan-bypass-eligibility-phase2.md)

---

## Problem

`bypass` is exposed on **every** node because `register_standard_ports()` always adds it, and the app shows it whenever the control profile allows. That is wrong for:

- **Sources** — camera, videoplayer, imageloader (no hot media inlet to relay)
- **Sinks** — window outputs, recorder (bypass is documented no-op)
- **Analyzers** — e.g. `BlobDetector` (`cpu.rgb8` → `blob.list`; bypass does nothing useful)
- **Some converters** — blind `relay_input()` can publish the wrong payload type
- **SubgraphNode / CodeBox** — boundary/script semantics deferred; bypass hidden in v1

`enable` remains useful on sources/sinks. Only **bypass visibility and semantics** need tightening.

---

## Goal

**Bypass is offered only when the node can meaningfully relay a primary media stream:**

> Primary hot inlet type is compatible with primary outlet type, **or** a registered conversion path exists between them.

Sources, sinks, non-relay nodes, **SubgraphNode**, and **CodeBox** must not show bypass in the UI (canvas + Node tab defaults + run controls). Core must treat `bypass: true` as no-op when unsupported.

---

## Progress (June 2026)

| Step | Repo | Branch / PR | Status |
|------|------|-------------|--------|
| Plan doc | ArgusModular | `main` | **Done** |
| **Step 1** — `supports_bypass_relay`, gate `is_bypass()`, introspection | ArgusCore | PR #40 → `01ee1c2` | **Done** |
| **Step 2** — Relay audit + typed relay helper | ArgusCore | `fix/bypass-relay-audit` | **Done** (hybrid relay) |
| **Step 3** — App visibility + defaults | ArgusModular | PR #11 | **Done** |
| **Step 4** — Tests | Both | — | **Partial** (core unit matrix + movie publish test; app manual smoke pending) |
| **Step 5** — Docs cross-links | Both | — | **Done** |

### What landed in Phase 1

**ArgusCore (`01ee1c2`):**

- `argus::node_ctl::supports_bypass_relay(const Node&)`
- `Node::is_bypass()` returns `false` when unsupported (Option A)
- `"supports_bypass": bool` on `node_ports_to_json`
- `examples/test_bypass_eligibility.cpp` — source/sink/processor/analyzer/converter matrix

**ArgusModular (PR #11):**

- `control_visibility.cpp`, `node_canvas_chrome.cpp`, `control_panel.cpp`, `node_panel.cpp` — hide bypass when unsupported
- `node_defaults.cpp` — default `bypass.visible` from topology on palette add
- `graph_session.cpp` — `normalize_bypass_param_ui()` on bundle load
- `ARGUS_CORE_REF` bumped to `01ee1c21…`

### Still open after Phase 1

- Step 2 relay audit (`relay_primary_hot_media`, typed relay vs blind `relay_input`)
- Explicit hide for **SubgraphNode** and **CodeBox** (may currently show bypass — same-type ports make them eligible)
- `undef` inlet eligibility for known converters (decision below)
- Converter runtime semantics (B vs C — implications documented below)
- Movie-player publish regression test (`bypass: true` must not change output)
- App smoke checklist (manual)
- ArgusCore `plan-inlets-push-executor.md` §4.4 cross-link

---

## Eligibility rule (v1)

### 1. Port selection

| Term | Definition |
|------|------------|
| **Primary media inlet** | First input port where `name != "ctl"` and `type != argus.dict` and `role == hot` |
| **Primary media outlet** | First output port (any type) |

Using **hot** first inlet keeps `OpenCVAdd` bypass = relay `a` → `out` (cold overlay `b` ignored), matching existing bypass behavior.

### 2. Supported when

All must hold:

1. Primary media inlet **exists**
2. Primary media outlet **exists**
3. `compatible(in_type, out_type)` **OR** `ConversionRegistry::find_path(in_type, out_type)` is non-empty
4. Node type is **not** `SubgraphNode` or `CodeBox` (v1 exclusion)

Where:

- `compatible` = `DataTypeRegistry::are_compatible` (equal types, or either `undef`)
- **`undef` on inlet:** eligibility is **static** — wiring does not change the decision. Show bypass for **known converter types** whose declared in/out types have a conversion path (e.g. `CpuToGlesUpload`, `GlesToCpuDownload`, `PassthroughCPU`). Do **not** defer visibility until the port resolves at wire time.

### 3. Explicit exclusions

Even if types match, **not supported** when:

| Class | Examples | Reason |
|-------|----------|--------|
| Pure source | `OpenCVCamera`, `OpenCVMoviePlayer`, `OpenCVImageLoader`, `GlesGltfLoader` | No primary media inlet |
| Pure sink | `GlesWindowOutput`, `OpenCVWindow`, `VulkanWindowOutput`, `OpenCVMovieRecorder` | No relay outlet / documented no-op |
| Analyzer | `BlobDetector`, `CentroidFilter` | In/out types unrelated; no conversion path |
| Multi-out source | `GlesGltfLoader` | No inlet |
| **SubgraphNode** | — | Boundary relay map deferred; **hide bypass v1** |
| **CodeBox** | — | Script-declared ports (B4); bypass relay N/A — **hide bypass v1** |

### 4. Should work (user examples)

| Node | In → out | Why |
|------|----------|-----|
| `OpenCVProcessor`, `OpenCVMirror` | `cpu.rgb8` → `cpu.rgb8` | Same type |
| `OpenCVProcessor` (grayscale) | `cpu.rgb8` → `cpu.gray8` | Conversion path in registry |
| `CpuToGlesUpload` | `cpu.rgb8` → `gles.texture` | Conversion path |
| `GlesToCpuDownload` | `gles.texture` → `cpu.rgb8` | Conversion path |
| `PassthroughCPU` | `undef`/typed → same | Compatible / known converter |

`argus.dict` ctl in/out bypass is **out of scope for v1** (no dict outputs on builtins today).

---

## Decisions (resolved)

| # | Question | Decision |
|---|----------|----------|
| 1 | **`undef` primary inlet** | **Show for known converters.** Wiring must not gate visibility — eligibility comes from declared port types + conversion registry. |
| 2 | **Converter bypass semantics** | **Hybrid (C + B fallback)** — relay when runtime types match; else normal/fast-path convert. |
| 3 | **Author override** | **Not needed.** No `ui.params.bypass.visible: true` escape hatch for unsupported nodes; remove if present in Phase 2 cleanup. |
| 4 | **SubgraphNode** | **Hide bypass in v1.** Same for **CodeBox**. |

### Converter bypass: B vs C

Applies to nodes where inlet and outlet types differ but a conversion path exists (`CpuToGlesUpload`, `GlesToCpuDownload`, grayscale `OpenCVProcessor`, etc.).

#### Option B — bypass still converts

When `bypass: true`, the node runs its normal `process()` on a **fast path** that performs the minimum work to produce a correctly typed outlet payload (e.g. upload CPU→GLES, download GLES→CPU, grayscale convert).

| Implication | Detail |
|-------------|--------|
| **User expectation** | “Bypass” on a converter means *skip extra processing*, not *skip the type change*. On `CpuToGlesUpload`, bypass ≈ upload only, no downstream shader ops inside that node. |
| **Downstream safety** | Outlet type always matches declaration; no wrong-type publishes. |
| **Performance** | Converters still pay conversion cost; bypass saves algorithm-specific work only. |
| **UI honesty** | Bypass toggle on converters **does something visible** whenever input is present. |
| **Implementation** | Per-node `if (is_bypass()) { … minimal convert … return; }` — no shared blind `relay_input`. |

#### Option C — relay only when types already match at runtime

Static eligibility (Step 1) still shows bypass when a conversion path exists. At runtime, bypass relays **only if** `latest(in)` is already `are_compatible` with the primary outlet type.

| Implication | Detail |
|-------------|--------|
| **User expectation** | Bypass on a converter is “passthrough when types already align”; otherwise it appears inert or must fall through. |
| **Downstream safety** | Avoids blind copy of mismatched payloads (fixes crash class seen on type-changing OpenCV nodes). |
| **Performance** | True zero-copy relay when types match (e.g. `cpu.rgb8` → `cpu.rgb8` processor configured for grayscale but receiving rgb8). |
| **UI honesty** | Bypass may **look broken** on converters when inlet/outlet types differ — needs tooltip/copy or hybrid fallback. |
| **Implementation** | Shared `relay_primary_hot_media()` with runtime type check; if incompatible, no-op or fall through to normal `process()`. |

#### Recommendation for Phase 2

- **Default: C** for relay helper + runtime type guard (safety).
- **Hybrid:** when C cannot relay, **fall through to B** (normal/fast-path convert) rather than no-op — avoids dead toggles on converters.
- Document per-node behavior in ArgusCore §4.4 table after implementation.

---

## Implementation steps

### Step 1 — Core capability API (ArgusCore) — **Done**

See Progress table. Delivered in PR #40 (`01ee1c2`).

---

### Step 2 — Relay audit (ArgusCore) — **Remaining**

| Action | Nodes |
|--------|-------|
| No code change needed | Processors already relay primary in → out |
| Confirm no-op | Sinks (already no-op or early return) |
| Ignore bypass param | Sources — gating via `is_bypass()` is enough |
| Fix or hide | `BlobDetector` — hidden via Step 1 |
| **Hide v1** | `SubgraphNode`, `CodeBox` — add to `supports_bypass_relay` exclusions |
| Safe relay | Replace blind `relay_input` with typed check per B/C decision |
| **`undef` inlet** | Treat known converter types as eligible without wire-time resolution |

**New helper:** `node_ctl::relay_primary_hot_media(Node&)` — primary hot in + first out; publish only when `are_compatible` (C), with optional convert fallback (hybrid).

---

### Step 3 — App UI (ArgusModular) — **Done**

Single source: `supports_bypass_relay(live_node)` / `ports.supports_bypass`.

| Location | Status |
|----------|--------|
| `control_visibility.cpp` | Done |
| `node_canvas_chrome.cpp` | Done |
| `node_defaults.cpp` / add-node | Done |
| `graph_session.cpp` normalize on load | Done |
| Node tab inspector | Done |

**Phase 2 app follow-up:** author override removed; bundle normalize forces `bypass.visible: false` on unsupported nodes.

---

### Step 4 — Tests — **Partial**

**ArgusCore — done:**

- `test_bypass_eligibility.cpp` matrix
- `is_bypass()` gated on `OpenCVMoviePlayer`

**ArgusCore — remaining:**

- `bypass: true` on movie player does not change publish behavior (executor-level)
- `test_push_executor` bypass relay regression re-run after Step 2
- SubgraphNode / CodeBox → `supports_bypass_relay` false

**ArgusModular (manual / smoke) — remaining:**

- Add `OpenCVMoviePlayer` → no bypass in canvas or Node tab
- Add `OpenCVProcessor` → bypass shown
- Add `SubgraphNode` / `CodeBox` → no bypass
- `only_bypass` profile shows bypass only on eligible nodes

---

### Step 5 — Docs — **In progress**

- [x] This plan — status + decisions
- [x] [plan-bypass-eligibility-phase2.md](plan-bypass-eligibility-phase2.md) — continuation tasks
- [x] [plan-ui-ux.md](plan-ui-ux.md) — control profile cross-link
- [ ] ArgusCore `plan-inlets-push-executor.md` §4.4 — reference `supports_bypass_relay` (Phase 2 core PR)

---

## PR stack

| Order | Repo | Branch | Scope | Status |
|-------|------|--------|-------|--------|
| 1 | ArgusCore | `fix/bypass-eligibility-core` | Step 1 | **Merged** |
| 2 | ArgusCore | `fix/bypass-relay-audit` | Step 2 | **Merged** (PR #41) |
| 3 | ArgusModular | `fix/bypass-eligibility-ui` | Step 3 | **Merged** (PR #11) |
| 4 | ArgusModular | `fix/bypass-eligibility-ui-2` | Docs + app cleanup + smoke | **Merged** (PR #12) |
| 5 | ArgusModular | — | Plan on `main` | **Done** |

---

## Review checklist

- [x] Eligibility rule matches product intent (sources/sinks off; processors/converters on)
- [x] Primary inlet = first **hot** media port (not cold overlay)
- [x] Converter bypass semantics chosen — **hybrid (C + B fallback)**
- [x] Gate `is_bypass()` in core (not UI-only)
- [x] SubgraphNode + CodeBox hidden v1
- [x] `undef` inlet: show for known converters, wiring-independent
- [x] No author override for unsupported bypass