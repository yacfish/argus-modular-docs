# Bypass eligibility — Phase 2 continuation

**Branch:** merged to `main` (June 2026)  
**Parent plan:** [plan-bypass-eligibility.md](plan-bypass-eligibility.md)  
**Status:** Complete — all work packages merged (core PR #41, app PR #12)

Phase 1 merged capability gating (core PR #40) and UI hiding (app PR #11). Phase 2 core work (exclusions, undef eligibility, hybrid relay, tests) lands on `fix/bypass-relay-audit`.

**Converter semantics:** **Hybrid** — `relay_primary_hot_media()` when runtime types match; otherwise fall through to normal `process()` (minimal convert).

---

## Work packages

### WP1 — Core exclusions (ArgusCore: `fix/bypass-relay-audit`) — **Done**

**Goal:** `supports_bypass_relay` returns `false` for `SubgraphNode` and `CodeBox`.

| Task | File(s) | Notes |
|------|---------|-------|
| Type denylist in `supports_bypass_relay` | `src/core/node_bypass.cpp` | Early return on `node.type_name()` |
| Extend unit tests | `examples/test_bypass_eligibility.cpp` | Expect false for both types |
| Bump `ARGUS_CORE_REF` | ArgusModular `cmake/resolve_argus_core.cmake` | After core merge |

**Acceptance:** Palette add + bundle load never show bypass on Subgraph/CodeBox.

---

### WP2 — `undef` inlet + known converters (ArgusCore) — **Done**

**Goal:** Wiring does not affect bypass visibility; converters with `undef` inlet stay eligible.

| Task | File(s) | Notes |
|------|---------|-------|
| When inlet type is `undef`, check conversion path to primary outlet | `node_bypass.cpp` | Aligns with decision #1 |
| Test `PassthroughCPU` / upload nodes with `undef` inlet | `test_bypass_eligibility.cpp` | |

**Acceptance:** `CpuToGlesUpload` shows bypass before any wire exists.

---

### WP3 — Relay audit + typed helper (ArgusCore) — **Done**

**Goal:** Replace blind `relay_input` on bypass paths where mismatched types can crash or corrupt downstream.

| Task | File(s) | Notes |
|------|---------|-------|
| Add `relay_primary_hot_media(Node&)` | `node_ctl.hpp`, new impl | Primary hot in → first out; `are_compatible` guard |
| Per B/C/hybrid: convert fallback or no-op | Converter `process()` bypass branches | `cpu_to_gles_upload`, `gles_to_cpu_download`, `opencv_processor`, … |
| Audit bypass branches | `nodes/*.cpp` | Table in plan-inlets §4.4 |

**Acceptance:** `test_push_executor` bypass tests pass; no wrong-type publish on grayscale/convert bypass.

---

### WP4 — Executor regression test (ArgusCore) — **Done**

**Goal:** Close Step 4 gap — movie player ignores bypass for publishing.

| Task | File(s) | Notes |
|------|---------|-------|
| Add test: `OpenCVMoviePlayer` with `bypass: true` publishes same as `bypass: false` | `test_bypass_eligibility.cpp` or `test_push_executor.cpp` | Frame count / payload equivalence |

---

### WP5 — App cleanup (ArgusModular: `fix/bypass-eligibility-ui-2`) — **Done**

**Goal:** Align app with final core API; remove override path.

| Task | File(s) | Notes |
|------|---------|-------|
| Remove author override in `should_show_parameter` | `control_visibility.cpp` | Decision: no `visible: true` escape for unsupported bypass |
| Re-pin core after WP1–WP3 | `resolve_argus_core.cmake` | |
| Manual smoke checklist | — | See parent plan Step 4 |

**Acceptance:** Unsupported bypass never shown even if bundle JSON has `bypass.visible: true` (normalize on load already forces false).

---

### WP6 — Docs (both repos)

| Task | Repo | File |
|------|------|------|
| §4.4 eligibility + `supports_bypass_relay` | ArgusCore | `docs/internal/plan-inlets-push-executor.md` |
| Mark Step 5 complete | ArgusModular | `plan-bypass-eligibility.md` |
| Control profile note | ArgusModular | `plan-ui-ux.md` |

---

## Suggested PR order

```
ArgusCore  fix/bypass-relay-audit
  ├─ WP1 exclusions (Subgraph, CodeBox)
  ├─ WP2 undef + converters
  ├─ WP3 relay helper + audit
  ├─ WP4 movie publish test
  └─ WP6 ArgusCore doc §4.4

ArgusModular  fix/bypass-eligibility-ui-2
  ├─ WP6 docs (this branch — can land first)
  ├─ Pin new ARGUS_CORE_REF
  └─ WP5 app cleanup + smoke notes
```

Docs-only commits on `fix/bypass-eligibility-ui-2` can merge before the core PR; app code changes wait on the core pin.

---

## Smoke checklist

Automated (`argus_test_bypass_ui_smoke`) — **passed**:

- [x] Palette defaults: `OpenCVMoviePlayer`, `OpenCVProcessor`, `SubgraphNode`, `CodeBox`
- [x] `only_bypass` profile on movie player, processor, subgraph, converter
- [x] Author override removed (`visible: true` ignored when unsupported)
- [x] Bundle normalize clears stale `bypass.visible: true` on movie player

Manual (ImGui canvas / run controls) — optional:

- [ ] Canvas chrome shows/hides bypass toggles live
- [ ] Toggle bypass on `OpenCVProcessor` in running graph — relay works
- [ ] Toggle bypass on `CpuToGlesUpload` — hybrid convert fallback visible

---

## Out of scope (v1)

- Subgraph boundary relay map
- CodeBox script-aware bypass
- `argus.dict` ctl bypass
- Author override for unsupported nodes