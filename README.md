# ETS2 / Prism3D stutter investigation

Active reverse-engineering investigation of scene-dependent CPU-side stutter in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

> **Status:** the hypothesis-agnostic whole-corpus `1.58.1.4s ↔ 1.60.1.7s` static diff is complete. Runtime phase localization has now narrowed the sustained heavy-state slowdown to **pre-render main-loop work** plus a specific rendergraph execution stage, `RG_CORE = 1.60.1.7s:0x14021FE20`. Raw rendergraph pass/order count can contribute, but matched-cardinality windows prove that count alone is not sufficient: essentially the same number of passes can execute about **2.9 ms slower** in a heavy state. The current discriminator is therefore **pass composition / per-pass work**, not another broad static search.

## What is being investigated

On the tested system, light scenes can hold roughly **16.67 ms / 60 FPS**, while heavier scene compositions can move into roughly **19–25 ms**. The slowdown is scene-dependent and can sometimes disappear after an unload/ferry/teleport transition without restarting the game.

Earlier runtime work showed that the sustained slowdown is not primarily explained by the measured DXGI/fence/wait gates. Slow regions mainly contain more active CPU work rather than one obvious fixed stall.

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

## Runtime localization through v0.4

Measured chain:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
                -> 0x14021FE20  RG_CORE / rendergraph execution stage
```

`NemoFramePhaseProbe v0.2` separated the nested wait helper `0x14011F730` and showed that it does **not** own the slowdown.

`v0.3` then split the remaining work. In a clean sustained heavy episode:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
RENDER_ACTIVE                   9.983 ms   11.593 ms  +1.610 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
POST_RENDER_OTHER               0.381 ms    0.380 ms  ~0
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

Among the selected immediate render children, essentially all positive heavy-state growth localized to **`RG_CORE 0x14021FE20`**. The non-render growth localized to **pre-render work**, not post-render cleanup.

`v0.4` sampled rendergraph cardinality at `RG_CORE` entry with no new target detours. Across nine clean heavy windows:

```text
                              GOOD        HEAVY       DELTA
LOOP                          16.689 ms   20.116 ms   +3.427 ms
RENDER_ACTIVE                 10.123 ms   12.071 ms   +1.949 ms
PRE_RENDER_OTHER               6.177 ms    7.602 ms   +1.425 ms
RG_CORE                        5.896 ms    9.417 ms   +3.521 ms
order_count                  157.5       183.5       +26.0
pass_count                   158.5       184.5       +26.0
```

Two heavy episodes showed large count jumps (~145 → ~190+), so pass cardinality can contribute.

However, matched-cardinality windows are decisive:

```text
                              SMOOTH      HEAVY
LOOP                          16.696 ms   19.851 ms
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

At essentially the **same pass count**, `RG_CORE` can be about **+2.91 ms slower**. The synchronization flag was never active in the run (`0 / 86,942`).

**Current conclusion:** raw pass count is a contributor in some episodes, but not the root discriminator. The next question is which pass types / per-pass work differ at the same total cardinality.

See [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md) and [`docs/rg-core-runtime-localization.md`](docs/rg-core-runtime-localization.md).

## Descriptor/root-binding branch: still real, not the whole theory

The mapped 1.58/1.60 DX12 binding architecture difference remains confirmed. Runtime probing found large fixed-capacity over-reservation, and `NemoDX12SamplerAllocReuse` safely avoids roughly **95%** of targeted sampler allocation/copy pressure.

However, sustained heavy-scene slowdown still occurs with that optimization active. The descriptor branch is therefore a real optimization/regression component, **not a complete explanation**.

See [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md).

## Current next step

Do not return to broad static candidate roulette or broad per-D3D-call tracing.

Current order:

1. compare smooth vs heavy windows at **matched rendergraph cardinality**;
2. sample the `RG_CORE` execution-order-selected passes at low rate;
3. aggregate pass types `1..7` and a few exact per-pass work-count fields already consumed by `RG_CORE`;
4. if one pass family/work count separates smooth from heavy, recurse only into that path;
5. if pass mix/work counts stay flat, move to branch timing or differential CPU stack sampling inside `RG_CORE`;
6. patch only after a concrete millisecond budget is localized.

## Help wanted

Useful outside contributions are welcome, especially:

- independent reproduction of the scene-dependent slowdown
- review of the `PRE_RENDER_OTHER` + `RG_CORE` localization
- interpretation of rendergraph pass-type composition around `0x14021FE20`
- low-overhead Windows x64 stack-sampling ideas suitable for matched-cardinality smooth-vs-heavy analysis
- corrections to build-specific function mappings

Please keep **FACT / INFERENCE / HYPOTHESIS** separate and identify the exact build for every address.

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Start here

- [`docs/current-findings.md`](docs/current-findings.md) — current technical state
- [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md) — runtime phase results
- [`docs/rg-core-runtime-localization.md`](docs/rg-core-runtime-localization.md) — RG_CORE and cardinality results
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
