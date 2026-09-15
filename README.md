# ETS2 / Prism3D stutter investigation

Active reverse-engineering investigation of scene-dependent CPU-side stutter in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

> **Status:** the hypothesis-agnostic whole-corpus `1.58.1.4s ↔ 1.60.1.7s` static diff is complete. Several top-ranked static candidates were then demoted by runtime measurement, so the investigation has pivoted to **coarse main-loop phase localization** before choosing the next leaf function.

## What is being investigated

On the tested system, light scenes can hold roughly **16.67 ms / 60 FPS**, while heavier scene compositions can move into roughly **19–25 ms**. The slowdown is scene-dependent and can sometimes disappear after an unload/ferry/teleport transition without restarting the game.

Earlier runtime work showed that the sustained slowdown is **not primarily explained by the previously measured DXGI/fence wait gates**. Slow regions mainly contain more work rather than one obvious per-draw spike.

## Whole-corpus diff status

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

See [`docs/global-diff-summary.md`](docs/global-diff-summary.md) for methodology and caveats.

## Important runtime correction to the static ranking

The global diff found several attractive 1.60 changes, but runtime frequency checks showed that the strongest-looking individual candidates do **not** directly own the sustained frame budget.

### `render_queue_set_t` copy helper

```text
1.60.1.7s:0x14154AAB0
```

This helper looked highly regression-shaped statically because the 1.60 implementation adds substantial p3mem-style ownership/refcount machinery. Runtime measurement, however, found only **14 direct calls across 24,798 rendered frames**.

The static delta is real; the sustained direct-cost theory is strongly demoted.

### `r_proto` lazy-resolution boundary

```text
1.60.1.7s:0x1413C1470
```

No direct `E8 rel32` callsites or incoming direct-call edges were found for the proposed instrumentation boundary. This does not prove the entire subsystem irrelevant, but the original direct-call experiment is closed without new reachability evidence.

### traffic neighbor update

```text
traffic_trajectory_t::update_neighbors_bits
1.60.1.7s:0x1408DC510
```

The target executed only **85 times** in the whole run, including a 30-call shutdown/unload-adjacent burst. A natural transition from about **16.775 ms/frame** to **22.268 ms/frame** occurred across an approximately **42.17 s interval with no calls to the target at all**.

That makes the direct execution cost far too sparse to explain the sustained heavy state.

## First useful coarse localization result

The current main-loop chain is:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
```

`NemoFramePhaseProbe v0.1` measured:

```text
LOOP   = main-loop iteration
PACE   = frame-clock bookkeeping
RENDER = broad rendergraph/present coordinator
OTHER  = LOOP - PACE - RENDER
```

Two separate natural good→heavy transitions showed similar growth:

```text
transition A: LOOP +4.116 ms, RENDER +1.641 ms, OTHER +2.475 ms
transition B: LOOP +3.745 ms, RENDER +2.007 ms, OTHER +1.738 ms
```

`PACE` stayed around `~0.001 ms/iteration` and is effectively ruled out as the owner of the regression.

This is the first measurement that consistently assigns a real part of the heavy-state delta to a broad CPU phase rather than an isolated static candidate.

See [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md).

## Why RENDER is not yet the answer

The broad `RENDER` bucket includes a nested frame-time wait helper at:

```text
1.60.1.7s:0x14011F730
```

That helper uses `Sleep()` plus a short spin phase. Therefore the observed `RENDER` delta can mix active render work with deliberate pacing wait.

The current experiment, `NemoFramePhaseProbe v0.2`, separates that wait and derives:

```text
RENDER_ACTIVE = RENDER - WAIT
OTHER         = LOOP - PACE - RENDER
CPU_ACTIVE    = OTHER + RENDER_ACTIVE + PACE
```

The next decision is based on where the missing milliseconds remain after `WAIT` is removed.

## Descriptor/root-binding branch: still real, not the whole theory

The mapped 1.58/1.60 DX12 binding architecture difference remains confirmed. Runtime probing found large fixed-capacity over-reservation, and `NemoDX12SamplerAllocReuse` safely avoids roughly **95%** of targeted sampler allocation/copy pressure.

However, sustained heavy-scene slowdown still occurs with that optimization active. The descriptor branch is therefore a real optimization/regression component, **not a complete explanation**.

See [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md).

## Current next step

Do not return to broad static candidate roulette or broad per-D3D-call tracing.

Current order:

1. finish `WAIT` separation with the phase probe;
2. identify whether the heavy delta sits in `RENDER_ACTIVE`, `OTHER`, or both;
3. if coarse timing is still insufficient, compare differential CPU stack samples between sustained good and heavy windows;
4. add state/cardinality counters only inside the phase that actually owns the missing time;
5. map the measured hotspot back to the completed 1.58 ↔ 1.60 counterpart map;
6. patch only after a measured millisecond budget exists.

## Help wanted

Useful outside contributions are welcome, especially:

- independent reproduction of the scene-dependent slowdown
- review of the current main-loop / wait boundary interpretation
- low-overhead Windows x64 stack-sampling ideas suitable for good-vs-heavy differential analysis
- corrections to build-specific function mappings
- alternative explanations for the measured `OTHER` / active-render split

Please keep **FACT / INFERENCE / HYPOTHESIS** separate and identify the exact build for every address.

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Start here

- [`docs/current-findings.md`](docs/current-findings.md) — current technical state
- [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md) — latest runtime pivot and phase results
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