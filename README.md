# ETS2 / Prism3D stutter investigation

Active reverse-engineering investigation of scene-dependent CPU-side stutter in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

> **Status:** runtime localization has narrowed the render-side slowdown to `RQ_ONE = 1.60.1.7s:0x14154C9F0`, reached through `RG_CORE -> T1_HELPER -> pass callback -> nested winner 0x1413BB140`. In `v0.10`, exact matched `160/161` cardinality shows the winner rising `2.286 -> 5.743 ms`; `RQ_ONE` alone rises `1.682 -> 4.535 ms` while its sampled call count falls. The next discriminator splits the four normal direct callsites inside `RQ_ONE`.

## What is being investigated

On the tested system, light scenes can hold roughly **16.67 ms / 60 FPS**, while heavier scene compositions can move into roughly **19–25 ms**. The slower state can be sustained and strongly scene-dependent; unload/ferry/teleport transitions can sometimes restore ~16.67 ms without restarting the game.

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

See [`docs/global-diff-summary.md`](docs/global-diff-summary.md).

## Why runtime localization replaced static candidate roulette

Several attractive isolated static candidates were directly demoted:

- `render_queue_set_t` copy helper `0x14154AAB0`: only **14 direct calls across 24,798 rendered frames**;
- `r_proto` boundary `0x1413C1470`: no direct-call boundary for the proposed experiment;
- `traffic_trajectory_t::update_neighbors_bits` `0x1408DC510`: only **85 total calls**, including a long heavy-state onset interval with no calls.

The static mappings remain useful, but runtime evidence now decides where to recurse.

## Current measured chain

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  PACE / frame-clock bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
                -> 0x14011F730  measured WAIT helper
                -> 0x14021FE20  RG_CORE / rendergraph execution
```

### v0.2 — coarse active split

```text
             GOOD       HEAVY      DELTA
LOOP         16.683 ms  20.358 ms  +3.675 ms
RENDER_ACTIVE11.404 ms  13.507 ms  +2.103 ms
OTHER         5.253 ms   6.840 ms  +1.586 ms
WAIT          0.024 ms   0.009 ms  -0.015 ms
```

The measured sleep/spin WAIT helper does not own the slowdown.

### v0.3 — localization to PRE_RENDER_OTHER + RG_CORE

A clean sustained heavy episode versus recovered ordinary gameplay:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
POST_RENDER_OTHER               0.381 ms    0.380 ms  ~0
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

Among the selected immediate RENDER children, essentially all positive growth localized to `RG_CORE`. Non-render growth localized to pre-render work.

### v0.4 — pass count can rise, but count is not the discriminator

Across clean heavy windows, order/pass counts often rise substantially. But matched-cardinality windows are decisive:

```text
                              SMOOTH      HEAVY
LOOP                          16.696 ms   19.851 ms
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

The sampled sync flag was never active (`0 / 86,942`).

### v0.5 — even coarse pass composition matches

`v0.5` sampled actual execution-order-selected pass types and exact work-count fields already consumed by RG_CORE.

One matched pair:

```text
                              SMOOTH      HEAVY
LOOP                          16.568 ms   19.412 ms
RG_CORE                        6.832 ms    9.825 ms
order / pass                  159 / 160   159 / 160
 type 1                         134         134
 type 3                           1           1
 type 4                          10          10
 type 6                           5           5
 type 7                           8           8
 callback-present               159         159
 raw +0x1338                    288         282
 type4 items                     10          10
 type7 refs                       7           7
```

Another exact `158 / 159` matched pair has `RG_CORE` at roughly **5.13 ms smooth vs 9.90 ms heavy** while the sampled type mix remains essentially the same.

**Current conclusion:** the same broad rendergraph workload is becoming materially more expensive to execute. More pass-count fields are unlikely to answer why.

See [`docs/rg-core-runtime-localization.md`](docs/rg-core-runtime-localization.md).

## v0.6 result — type-1 helper wins

At exact matched order/pass cardinality `168 / 169`:

```text
                              SMOOTH      HEAVY
RG_CORE                        5.225 ms    8.196 ms
type-1 calls / RG_CORE        ~143.9      ~144.0
type-1 sample avg              ~16 us      ~32 us
type-1 estimated/RG_CORE       2.373 ms     4.650 ms
type-4 estimated/RG_CORE       0.004 ms     0.005 ms
type-7 estimated/RG_CORE       0.000 ms     0.000 ms
```

The type-1 path explains about **2.28 ms of the 2.97 ms RG_CORE increase** in this exact pair, while type-4/type-7 timing is effectively flat. The same direction repeats across several other exact-cardinality groups.

## v0.10 result — RQ_ONE is the dominant concrete child

The substantive winner `0x1413BB140` was split into seven direct children plus residual. At exact matched `order/pass = 160/161`:

```text
                              SMOOTH      HIGH-COST
RG_CORE                        6.925 ms   11.358 ms
winner 0x1413BB140             2.286 ms    5.743 ms
RQ_ONE 0x14154C9F0            1.682 ms    4.535 ms
RQ_PREP 0x14154C370           0.448 ms    1.023 ms
winner residual                0.022 ms    0.022 ms
```

The `RQ_ONE` contribution rises by about **2.85 ms** while its sampled call count falls `496 -> 352`; average sampled duration rises `635 -> 1940 qpc`.

Across normal windows, winner cost and `RQ_ONE` cost correlate at about `0.998`.

**FACT:** `RQ_ONE = 0x14154C9F0` is the dominant measured owner inside the winning callback implementation. `RQ_PREP` is a secondary contributor; winner-body residual is negligible.

Static mapping identifies a close 1.58 counterpart:

```text
1.60 0x14154C9F0  <->  1.58 0x1413D7170
```

## Current next step — split RQ_ONE

The next probe separately times four normal direct callsites inside `0x14154C9F0`:

```text
HEAD_DISPATCH_154CF60
VIEW_UPDATE_2158E0
CMD_ALLOC_281C80
INNER_DISPATCH_154CF60
```

and reports the remaining RQ_ONE residual. This should distinguish dispatch cost from view/state update, command allocation, or body work.

No behavior patch is justified yet.

## Descriptor/root-binding branch

The mapped 1.58/1.60 DX12 binding architecture difference remains confirmed. Runtime probing found large fixed-capacity over-reservation, and `NemoDX12SamplerAllocReuse` safely avoids roughly **95%** of targeted sampler allocation/copy pressure.

Sustained heavy-scene slowdown still occurs with that optimization active, so this is a real optimization/regression component, **not a complete explanation**.

See [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md).

## Help wanted

Useful outside contributions include independent reproduction, review of the `PRE_RENDER_OTHER + RG_CORE` localization, interpretation of the RG_CORE branch helpers, low-overhead Windows x64 sampling ideas, and corrections to build-specific mappings.

Please keep **FACT / INFERENCE / HYPOTHESIS** separate and identify the exact build for every address. See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Start here

- [`docs/current-findings.md`](docs/current-findings.md)
- [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md)
- [`docs/rg-core-runtime-localization.md`](docs/rg-core-runtime-localization.md)
- [`docs/global-diff-summary.md`](docs/global-diff-summary.md)
- [`docs/function-map.md`](docs/function-map.md)
- [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md)
- [`docs/methodology.md`](docs/methodology.md)
- [`docs/experiments.md`](docs/experiments.md)
- [`docs/disproven-hypotheses.md`](docs/disproven-hypotheses.md)
- [`pseudocode/`](pseudocode/)

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