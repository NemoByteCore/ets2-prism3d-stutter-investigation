# ETS2 / Prism3D stutter investigation

Evidence-driven reverse engineering of a scene-dependent CPU-side slowdown in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

## Current status

The render-side slowdown has been narrowed from the whole frame to one small dispatch path:

```text
main loop
  -> RENDER
    -> RG_CORE 0x14021FE20
      -> T1 helper 0x14021F560
        -> pass callback 0x14021F73C
          -> nested winner 0x1413BB140
            -> RQ_ONE 0x14154C9F0
              -> HEAD_DISPATCH 0x14154CF60
                -> 0x1402D8D20
```

The current question is no longer "which subsystem is slow?" but whether the cost inside `HEAD_DISPATCH` comes from its no-split path or ranged path to the shared downstream routine `1.60.1.7s:0x1402D8D20`.

## Strongest runtime evidence

Light scenes can hold roughly **16.67 ms / 60 FPS** while heavier scene compositions can sustain roughly **19–25 ms**.

A broad phase split showed two independent growth areas:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

The current render-side leaf is much narrower. At exact matched `order/pass = 156/157`:

```text
                              LOW-COST    HIGH-COST
RQ_ONE parent                  1.855 ms    4.002 ms
HEAD_DISPATCH                  1.850 ms    3.997 ms
HEAD samples                     398         386
HEAD avg qpc                     842        1727
RQ_ONE residual                0.005 ms    0.005 ms
```

Across the full accepted run, `HEAD_DISPATCH` accounts for about **99.84%** of sampled `RQ_ONE` time, while sampled call count can fall as per-call cost rises. This is strong evidence for a per-call slowdown rather than simply more invocations.

## What has already been ruled down

Runtime evidence has demoted several attractive explanations:

- the measured WAIT helper does not own the sustained slowdown;
- raw rendergraph pass/order count is not sufficient;
- coarse pass-type/work composition is not sufficient;
- type-4 and type-7 helper paths are negligible in the accepted RG_CORE split;
- direct execution of `1.60.1.7s:0x14154AAB0` is far too sparse;
- the proposed direct-call boundary at `1.60.1.7s:0x1413C1470` has no direct callsites;
- `traffic_trajectory_t::update_neighbors_bits` is too sparse to own the sustained frame budget.

See [`docs/disproven-hypotheses.md`](docs/disproven-hypotheses.md).

## Static comparison

The current measured functions have close static counterparts in the 1.58 reference build:

```text
1.60.1.7s:0x14154C9F0  <->  1.58.1.4s:0x1413D7170
1.60.1.7s:0x14154CF60  <->  1.58.1.4s:0x1413D7700
1.60.1.7s:0x1402D8D20  <->  1.58.1.4s:0x1401EC530
```

The whole-corpus static diff remains the map; runtime measurement decides where to recurse.

## Secondary findings kept for later

Two real branches are intentionally not being opened in parallel:

- `PRE_RENDER_OTHER` contributes roughly +1.5 ms in the accepted broad split;
- the DX12 descriptor/root-binding branch contains real fixed-capacity sampler pressure, and sampler-table reuse removes roughly 95% of targeted allocation/copy work, but does **not** eliminate the broader heavy-state slowdown.

See [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md).

## Read this first

- [`docs/current-findings.md`](docs/current-findings.md) — current technical snapshot
- [`docs/rg-core-runtime-localization.md`](docs/rg-core-runtime-localization.md) — condensed evidence ladder for the render branch
- [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md) — broad frame-budget localization
- [`docs/global-diff-summary.md`](docs/global-diff-summary.md) — 1.58 ↔ 1.60 corpus map
- [`docs/methodology.md`](docs/methodology.md) — measurement rules
- [`docs/disproven-hypotheses.md`](docs/disproven-hypotheses.md) — branches not to repeat
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — useful ways to help

## Build policy

Runtime / profiling / patch target:

```text
ETS2:      1.60.1.7s
revision:  26c95e307fd5
renderer:  native DX12
SHA-256:   1D61BA2337E4D8CED85A06E10566A4DF064A2A0919CCD5E51561972D2A04255E
```

Static-only reference:

```text
ETS2:      1.58.1.4s
SHA-256:   AB9785331BF9970542C61A0108A4E677C9F7C00FD316D4C0F9AB116F6BE6C234
```

**1.58.1.4s is never run in this investigation.**

## Publication policy

No SCS executables, proprietary assets, giant raw decompiler dumps, private handoffs or local-only data are published here. Public notes contain original analysis, normalized pseudocode, mappings and reproducible measurements.
