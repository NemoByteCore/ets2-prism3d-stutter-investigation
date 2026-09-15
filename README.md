# ETS2 / Prism3D stutter investigation

Active reverse-engineering investigation of scene-dependent CPU-side stutter in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

> **Status:** the hypothesis-agnostic whole-corpus `1.58.1.4s ↔ 1.60.1.7s` static diff is complete. The next stage is a narrow runtime discriminator for the strongest new steady-state candidate, not another broad profiler.

## What is being investigated

On the tested system, light scenes can hold roughly **16.67 ms / 60 FPS**, while heavier scene compositions can move into roughly **19–25 ms**. The slowdown is scene-dependent and can sometimes disappear after an unload/ferry/teleport transition without restarting the game.

Runtime work shows that the sustained slowdown is **not primarily a DXGI wait / fence-wait problem**. The CPU reaches submission/present too late; the interesting work happens earlier in render/simulation construction.

Slow active-gameplay regions mainly show **more work**, not one obvious per-draw explosion. This favors repeated per-item/per-draw/per-queue CPU costs that scale with scene complexity.

## Whole-corpus diff status

The static comparison is no longer limited to the previously mapped descriptor branch.

```text
1.58 functions: 66,820
1.60 functions: 68,834
confirmed counterparts: 58,589
coverage: 87.68% / 85.12%
materially changed confirmed pairs: 9,308
strong anchored 1.60-only: 597
strong anchored 1.58-only: 375
ambiguous unmatched regions: 3,370
```

Unmatched functions are deliberately not all called new/removed. See [`docs/global-diff-summary.md`](docs/global-diff-summary.md) for methodology, caveats and the current ranking.

## Strongest new candidate

The global pass found a very high-confidence `render_queue_set_t` copy counterpart:

```text
1.58.1.4s:0x1413D5830   516 B
1.60.1.7s:0x14154AAB0  1370 B
```

The 1.60 implementation performs the same broad copy/append role but contains substantial new p3mem-style ownership/refcount machinery.

It is reached from a strongly conserved render-frame construction path:

```text
1.58.1.4s:0x141213E40  7503 B
1.60.1.7s:0x1413C1AE0  7503 B
```

The caller invokes the helper in a queue-set loop plus two additional calls outside the loop.

This is currently the strongest new **steady-state regression-shaped static candidate**, but its runtime cost has not yet been measured. It is not presented as a confirmed root cause.

## Broader 1.60 `p3mem` migration

The global diff also found a cross-cutting allocator/scope migration:

```text
direct _malloc_base calls:
1.58: 4,212
1.60:   260

1.60 p3_alloc path:        0x140117240  (~1,485 static incoming edges)
1.60 scope lifetime/free:  0x140117400  (~3,828 static incoming edges)
```

Among a conservative mapped set, `390 / 392` 1.60 `p3_alloc` callers have 1.58 counterparts using `_malloc_base`.

The migration reaches active render and traffic paths, but static prevalence alone does not establish frametime cost.

## Descriptor/root-binding branch: still real, no longer the whole theory

The mapped 1.58/1.60 DX12 binding architecture difference remains confirmed:

```text
1.58:
actual pipeline layout
→ layout-specific root signature/capacity
→ compact resource + sampler tables

1.60:
RFX shader profile
→ one of 13 fixed root-signature profiles
→ fixed capacities
→ individual root CBVs + split per-set tables
→ generalized root-parameter submission
```

Runtime probing confirmed large fixed-capacity over-reservation. `NemoDX12SamplerAllocReuse` safely avoids roughly **95%** of targeted sampler allocation/copy pressure.

However, sustained heavy-scene slowdown still occurs with that optimization active. The descriptor branch is therefore a real optimization/regression component, **not a complete explanation**.

See [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md).

## Current next step

Do **not** repeat `NemoShaderProfileProbe v0.2` as a broad performance profiler; its direct D3D hooks create a substantial observer effect.

The next runtime test is a minimal probe around:

```text
1.60.1.7s:0x14154AAB0
```

First measure only low-overhead values during ordinary gameplay:

- helper calls per synchronized frametime window
- queue-set count if safe/read-only
- cheap count of the ownership/refcount-heavy path if identifiable
- natural unload/ferry/teleport transitions as context

If those values correlate with the ~16.7 ms → ~20–25 ms state change, follow with aggregate/sampled timing. If not, demote the candidate immediately and continue down the global ranking.

## Help wanted

Useful outside contributions are welcome, especially:

- review of the new whole-corpus mapping and `render_queue_set` counterpart
- independent reproduction of the scene-dependent slowdown
- low-overhead ideas for measuring a very hot helper without per-call logging
- review of p3mem ownership semantics in the mapped frame path
- corrections to build-specific function mappings

Please keep **FACT / INFERENCE / HYPOTHESIS** separate and identify the exact build for every address.

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Start here

- [`docs/current-findings.md`](docs/current-findings.md) — current technical state
- [`docs/global-diff-summary.md`](docs/global-diff-summary.md) — completed whole-corpus 1.58 ↔ 1.60 comparison
- [`docs/function-map.md`](docs/function-map.md) — build-specific function map
- [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md) — descriptor/root-binding branch
- [`docs/methodology.md`](docs/methodology.md) — evidence and experiment rules
- [`docs/experiments.md`](docs/experiments.md) — experiment summaries
- [`docs/disproven-hypotheses.md`](docs/disproven-hypotheses.md) — closed/demoted directions
- [`pseudocode/`](pseudocode/) — normalized reconstructions

## Build policy

### Runtime / test / fix target

```text
ETS2:      1.60.1.7s
revision:  26c95e307fd5
renderer:  native DX12
SHA-256:   1D61BA2337E4D8CED85A06E10566A4DF064A2A0919CCD5E51561972D2A04255E
```

### Static-only reference

```text
ETS2:      1.58.1.4s
SHA-256:   AB9785331BF9970542C61A0108A4E677C9F7C00FD316D4C0F9AB116F6BE6C234
```

**1.58.1.4s is never run in this investigation.**

## Evidence / publication policy

No SCS executables, proprietary assets, giant raw decompiler dumps, private handoffs or local-only data are published here. This repository contains original analysis, normalized pseudocode, mappings and reproducible research notes.