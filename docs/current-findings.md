# Current findings

Updated: **2026-09-16**

## Scope

Runtime / patch target:

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

**1.58.1.4s is never run.**

## Symptom

The tested setup can hold roughly `16.67 ms / 60 FPS` in light states and sustain roughly `19–25 ms` in heavier scene compositions. The state is scene-dependent rather than a single hitch and can sometimes clear after an unload/ferry/teleport transition.

## Whole-corpus diff

```text
1.58 functions:                    66,820
1.60 functions:                    68,834
confirmed counterpart pairs:       58,589
materially changed confirmed pairs: 9,308
strong anchored 1.60-only:            597
strong anchored 1.58-only:            375
ambiguous unmatched regions:         3,370
```

See [`global-diff-summary.md`](global-diff-summary.md).

The global diff remains the static map, but runtime measurement now decides which branches deserve deeper work.

## Important runtime-demoted static leads

- `0x14154AAB0` render-queue copy helper: only **14 direct calls across 24,798 rendered frames**.
- `0x1413C1470` r_proto boundary: no usable direct-call boundary for the proposed probe.
- `0x1408DC510` traffic neighbor update: only **85 total calls**, including a heavy-state onset interval with no calls.

These findings are why the project no longer promotes candidates merely because a 1.60 function became larger or gained new p3mem/refcount machinery.

## Runtime localization

Measured chain:

```text
0x1401C5280  outer loop owner
  -> 0x1401C77C0  LOOP
       -> 0x1401C6CB0  PACE
       -> 0x1401D72F0  RENDER
            -> 0x14011F730  WAIT
            -> 0x14021FE20  RG_CORE
```

### v0.2 — active render + other loop work both grow

```text
             GOOD       HEAVY      DELTA
LOOP         16.683 ms  20.358 ms  +3.675 ms
RENDER_ACTIVE11.404 ms  13.507 ms  +2.103 ms
OTHER         5.253 ms   6.840 ms  +1.586 ms
WAIT          0.024 ms   0.009 ms  -0.015 ms
```

PACE is negligible. The measured WAIT helper is sparse and does not own the slowdown.

### v0.3 — broad cost localized

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
POST_RENDER_OTHER               0.381 ms    0.380 ms  ~0
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

**FACT:** the non-render growth is pre-render work.

**FACT:** among the selected immediate render children, the positive heavy-state growth is concentrated almost entirely in `RG_CORE = 0x14021FE20`.

### v0.4 — raw rendergraph cardinality is not enough

Some heavy episodes do have more rendergraph passes. But matched-cardinality windows show:

```text
                              SMOOTH      HEAVY
LOOP                          16.696 ms   19.851 ms
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

The sampled synchronization flag was never active (`0 / 86,942`).

**FACT:** total order/pass count is not a sufficient heavy-state discriminator.

### v0.5 — matched composition still differs by several milliseconds

`v0.5` sampled actual execution-order-selected pass types and several exact work-count fields already read by RG_CORE:

```text
type 1..7 counts
callback-present count
pass + 0x1338 raw count
type-4 pass + 0xDB0 items
type-7 pass + 0x12D0 refs
```

The run was structurally clean: `97,290` LOOP calls, `97,287` RENDER/RG_CORE calls, only 3 no-render loops, and zero order mismatch, overlap, reentry, bad-end, thread-mismatch or child-sum accounting violations. Sync remained zero.

Decisive matched pair A:

```text
                              SMOOTH      HEAVY
LOOP                          16.568 ms   19.412 ms
RG_CORE                        6.832 ms    9.825 ms
order/pass                    159/160     159/160
T1                              134         134
T3                                1           1
T4                               10          10
T6                                5           5
T7                                8           8
callback-present                159         159
raw +0x1338                     288         282
type4 items                      10          10
type7 refs                        7           7
```

Decisive matched pair B:

```text
                              SMOOTH      HEAVY
LOOP                          16.704 ms   20.024 ms
RG_CORE                        5.126 ms    9.904 ms
order/pass                    158/159     158/159
T1                              134         134
T4                               10          10
T6                                5           5
T7                                8           8
```

A third `157/158` matched pair differs by about `+4.10 ms` in RG_CORE with essentially the same sampled mix/work counts.

**FACT:** the coarse pass composition/work fields measured by v0.5 are not sufficient to explain the heavy-state RG_CORE cost.

**INFERENCE:** broadly the same rendergraph work is becoming materially more expensive to execute; the next useful measurement is elapsed time inside execution paths rather than additional count fields.

See [`rg-core-runtime-localization.md`](rg-core-runtime-localization.md).

## Current RG_CORE branch targets

Static inspection of `0x14021FE20` identifies three clean direct helper paths:

```text
0x14021F560  type-1 helper
0x14021F780  type-4 helper
0x1402DE540  type-7 per-reference helper
```

Type 1 dominates the observed pass mix. Its helper performs device-facing work and invokes a pass-specific callback when present, so it is the strongest first timing discriminator without assuming it is the answer.

The type-4 helper processes its item list and callback/device work. The type-7 helper executes once per type-7 reference.

## Immediate technical direction

`v0.6` keeps accepted v0.3 timing and v0.4 cardinality, removes the v0.5 pass-mix scanner, and sampled-times only every 16th execution of the three helper callsites above.

```text
matched smooth vs heavy
  -> compare sampled type-1 / type-4 / type-7 helper duration
  -> recurse only into the measured winning branch
  -> if none explains the delta, isolate RG_CORE residual/tail or use differential CPU stack sampling
  -> map measured hotspot to 1.58 ↔ 1.60 counterpart
  -> patch only after a concrete missing-ms budget is localized
```

No behavior patch is justified yet.

## Descriptor/root-binding architecture

The mapped 1.58/1.60 DX12 descriptor/root-binding difference remains confirmed. `NemoDX12SamplerAllocReuse` safely removes roughly **95%** of targeted sampler allocation/copy pressure, but sustained heavy-scene slowdown still occurs with that optimization active.

Therefore the descriptor branch is a real optimization/regression component, not a complete explanation. See [`shader-profile-architecture-delta.md`](shader-profile-architecture-delta.md).

## Broad 1.60 p3mem migration

The global corpus still shows the broad allocator/scope migration:

```text
direct _malloc_base calls: 1.58 = 4,212; 1.60 = 260
1.60 p3_alloc path:      0x140117240  (~1,485 incoming edges)
1.60 lifetime/free path: 0x140117400  (~3,828 incoming edges)
```

This is structural context, not permission to hook or patch p3mem globally.

## Demoted / insufficient explanations

Do not promote these again without new evidence:

- DirectStorage introduction
- six-shader tuple/profile collision
- map-dump/I/O SRW-lock branch
- TAA history-init growth as sustained cause
- editor/load KDOP/vegetation candidates
- collision-init traffic semaphore candidate
- setup/config/UI/unit/model candidates
- direct-cost theory for `0x14154AAB0`
- repeated direct-call probing of `0x1413C1470`
- direct-cost theory for `0x1408DC510`
- measured WAIT helper as sustained owner
- raw RG_CORE pass/order count as a sufficient explanation
- v0.5 coarse pass mix / measured work-count fields as a sufficient explanation

## Telemetry caveat

`SCS frame_start` is not guaranteed to be 1:1 with physically rendered frames. Prefer rendered-frametime timing, wall time, synchronized counters and narrow aggregate duration measurements.