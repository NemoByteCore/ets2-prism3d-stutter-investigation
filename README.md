# ETS2 / Prism3D stutter investigation

Active reverse-engineering investigation of scene-dependent CPU-side stutter in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

> **Status:** the hypothesis-agnostic whole-corpus `1.58.1.4s ↔ 1.60.1.7s` static diff is complete. Runtime follow-up demoted several top-ranked isolated candidates, and coarse main-loop timing now shows that the sustained heavy-state delta is split between **active render-side CPU work** and **other main-loop work**. The measured sleep/spin wait helper does not explain the slowdown.

## What is being investigated

On the tested system, light scenes can hold roughly **16.67 ms / 60 FPS**, while heavier scene compositions can move into roughly **19–25 ms**. The slowdown is scene-dependent and can sometimes disappear after an unload/ferry/teleport transition without restarting the game.

Earlier runtime work showed that the sustained slowdown is not primarily explained by the previously measured DXGI/fence wait gates. Slow regions mainly contain more work rather than one obvious fixed stall.

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

## Runtime correction to the static ranking

Three attractive isolated candidates were demoted by direct runtime evidence:

- `render_queue_set_t` copy helper `1.60.1.7s:0x14154AAB0`: only **14 direct calls across 24,798 rendered frames**;
- `r_proto` boundary `1.60.1.7s:0x1413C1470`: no direct-call boundary for the proposed experiment;
- `traffic_trajectory_t::update_neighbors_bits` `1.60.1.7s:0x1408DC510`: only **85 total calls**, with no calls for ~42 s centered on a natural heavy-state onset.

These results are why the investigation switched from isolated static-candidate probing to runtime phase localization.

## Coarse main-loop localization

Current measured chain:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
```

`NemoFramePhaseProbe v0.2` additionally measures a nested timing helper:

```text
WAIT = 0x14011F730
```

and derives:

```text
RENDER_ACTIVE = RENDER - WAIT
OTHER         = LOOP - PACE - RENDER
CPU_ACTIVE    = OTHER + RENDER_ACTIVE + PACE
```

Across clean ordinary-gameplay good (`LOOP <= 17 ms`) versus heavy (`LOOP >= 19 ms`) windows:

```text
             GOOD       HEAVY      DELTA
LOOP         16.683 ms  20.358 ms  +3.675 ms
RENDER_ACTIVE11.404 ms  13.507 ms  +2.103 ms
OTHER         5.253 ms   6.840 ms  +1.586 ms
WAIT          0.024 ms   0.009 ms  -0.015 ms / loop
```

The sleep/spin helper is conditional and sparse (`2,092` calls vs `33,339` main-loop calls) and contributes only tens of microseconds per normal gameplay loop.

**Current coarse split:** roughly **57% active render-side work / 43% other main-loop work**.

See [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md).

## Descriptor/root-binding branch: still real, not the whole theory

The mapped 1.58/1.60 DX12 binding architecture difference remains confirmed. Runtime probing found large fixed-capacity over-reservation, and `NemoDX12SamplerAllocReuse` safely avoids roughly **95%** of targeted sampler allocation/copy pressure.

However, sustained heavy-scene slowdown still occurs with that optimization active. The descriptor branch is therefore a real optimization/regression component, **not a complete explanation**.

See [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md).

## Current next step

Do not return to broad static candidate roulette or broad per-D3D-call tracing.

Current order:

1. subdivide `RENDER_ACTIVE` inside `0x1401D72F0` around stable low-overhead boundaries;
2. subdivide `OTHER` inside `0x1401C77C0` into stable pre/post-render or equivalent subphases;
3. if the cost remains distributed, compare differential CPU stack samples between sustained good and heavy windows;
4. add state/cardinality counters only inside the measured winning subphase;
5. map the measured hotspot back to the completed `1.58 ↔ 1.60` counterpart set;
6. patch only after a measured millisecond budget exists.

## Help wanted

Useful outside contributions are welcome, especially:

- independent reproduction of the scene-dependent slowdown
- review of the current main-loop / render-active split
- low-overhead Windows x64 stack-sampling ideas suitable for good-vs-heavy differential analysis
- corrections to build-specific function mappings
- alternative explanations for a shared scene/state driver increasing both `RENDER_ACTIVE` and `OTHER`

Please keep **FACT / INFERENCE / HYPOTHESIS** separate and identify the exact build for every address.

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Start here

- [`docs/current-findings.md`](docs/current-findings.md) — current technical state
- [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md) — latest runtime phase results
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
