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

## Primary runtime symptoms

On the tested setup:

- light scenes can hold roughly `16.67 ms / 60 FPS`
- heavier scene compositions commonly reach roughly `19–25 ms`
- the slower state can be sustained rather than a single hitch
- severity strongly depends on scene composition
- unload/ferry/teleport transitions can sometimes restore ~16.67 ms without restarting the game

## Whole-corpus diff status

```text
1.58 function inventory: 66,820
1.60 function inventory: 68,834
confirmed counterpart pairs: 58,589
coverage of 1.58: 87.68%
coverage of 1.60: 85.12%
materially changed confirmed pairs: 9,308
strong anchored 1.60-only: 597
strong anchored 1.58-only: 375
ambiguous unmatched regions: 3,370
```

See [`global-diff-summary.md`](global-diff-summary.md).

## Strong isolated static candidates were demoted by runtime measurement

### `render_queue_set_t` copy helper

```text
1.60.1.7s:0x14154AAB0
```

Despite a strong 1.58/1.60 static delta, runtime measurement found only **14 direct calls across 24,798 rendered frames**.

**Decision:** strongly demote the direct-cost theory.

### `r_proto` lazy-resolution boundary

```text
1.60.1.7s:0x1413C1470
```

No direct `E8 rel32` callsites or incoming direct-call edges were found for the proposed instrumentation boundary.

**Decision:** do not repeat the same direct-call probe without new reachability evidence.

### `traffic_trajectory_t::update_neighbors_bits`

```text
1.60.1.7s:0x1408DC510
```

The target executed only **85 times** in the whole run. A natural heavy-state transition occurred across an approximately **42.17 s interval with no calls to the target at all**.

**Decision:** direct execution cost is far too sparse to own the sustained frame budget.

## Runtime-first phase localization

The measured main-loop chain is:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
                -> 0x14021FE20  RG_CORE / rendergraph execution stage
```

`PACE` stays around `~0.001 ms/iteration` and is negligible.

`NemoFramePhaseProbe v0.2` separated the nested `Sleep()` + spin helper `0x14011F730`. In clean good-vs-heavy gameplay windows:

```text
             GOOD       HEAVY      DELTA
LOOP         16.683 ms  20.358 ms  +3.675 ms
RENDER_ACTIVE11.404 ms  13.507 ms  +2.103 ms
OTHER         5.253 ms   6.840 ms  +1.586 ms
WAIT          0.024 ms   0.009 ms  -0.015 ms / loop
```

**FACT:** the measured WAIT helper does not explain the heavy-state slowdown.

## v0.3: the broad buckets are now localized

`v0.3` split `OTHER` into pre/post-render portions and measured selected immediate children of `0x1401D72F0`.

A clean sustained heavy episode versus recovered ordinary gameplay:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
RENDER_ACTIVE                   9.983 ms   11.593 ms  +1.610 ms
OTHER                           6.667 ms    8.132 ms  +1.465 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
POST_RENDER_OTHER               0.381 ms    0.380 ms  ~0
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

The instrumentation safety/accounting counters stayed clean, and the clean baseline remained essentially identical to v0.2.

**FACT:** the non-render increase is specifically **pre-render work**.

**FACT:** post-render cleanup is flat.

**FACT:** among the selected direct RENDER children, the positive heavy-state delta is concentrated almost entirely in:

```text
RG_CORE = 1.60.1.7s:0x14021FE20
```

See [`rg-core-runtime-localization.md`](rg-core-runtime-localization.md).

## v0.4: raw rendergraph cardinality contributes, but is not sufficient

`RG_CORE` iterates a rendergraph execution-order array. `v0.4` sampled, read-only, at entry:

```text
order_count = state + 0x160
pass_count  = state + 0xB8
sync_flag   = state + 0x218
```

No new game target detours were added.

Across nine clean sustained heavy windows:

```text
                              GOOD        HEAVY       DELTA
LOOP                          16.689 ms   20.116 ms   +3.427 ms
RENDER_ACTIVE                 10.123 ms   12.071 ms   +1.949 ms
PRE_RENDER_OTHER               6.177 ms    7.602 ms   +1.425 ms
RG_CORE                        5.896 ms    9.417 ms   +3.521 ms
order_count                  157.5       183.5       +26.0
pass_count                   158.5       184.5       +26.0
```

Two heavy episodes show large count jumps from roughly `145` to roughly `190+`, so cardinality can contribute.

However, matched-cardinality windows show:

```text
                              SMOOTH      HEAVY
LOOP                          16.696 ms   19.851 ms
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

At essentially identical raw pass/order count, `RG_CORE` is about **2.91 ms slower** in the heavy state.

There are also smooth windows above 210 order entries with `RG_CORE` only around ~5.7–6.0 ms.

The sampled synchronization flag was never active:

```text
sync_hits = 0 / 86,942
```

**FACT:** raw pass count is not sufficient to explain the heavy state.

**INFERENCE:** pass composition and/or per-pass workload is now a stronger discriminator than total count.

## Current RG_CORE static discriminator

The `0x14021FE20` pseudocode dispatches execution-order-selected passes by a type at `pass + 0x8`, with supported types `1..7`.

Exact runtime-consumed fields suitable for low-rate read-only sampling include:

```text
pass + 0x1338  raw list count consumed before dispatch
pass + 0x19A8  callback/work-object pointer
pass + 0xDB0   count consumed by the type-4 helper
pass + 0x12D0  reference count iterated by type-7
```

The `+0x1338` field is intentionally not given a stronger semantic name than the pseudocode supports.

## Descriptor / root-binding architecture remains a confirmed component

The mapped 1.58 DX12 path derives root signatures and descriptor capacities from actual pipeline layout. The mapped 1.60 path selects fixed shader/root-signature profiles with fixed capacities.

Runtime probing confirmed large fixed-capacity over-reservation. `NemoDX12SamplerAllocReuse` safely removed roughly **95%** of targeted sampler allocation/copy pressure with clean safety counters.

The broader heavy-scene slowdown still occurs with that optimization active.

**Conclusion:** descriptor/sampler work is a validated optimization component, not a complete explanation.

See [`shader-profile-architecture-delta.md`](shader-profile-architecture-delta.md).

## Broad 1.60 `p3mem` migration remains a structural fact

The global corpus shows:

```text
direct _malloc_base calls:
1.58: 4,212
1.60:   260

1.60 p3_alloc path:       0x140117240  (~1,485 static incoming edges)
1.60 lifetime/free path:  0x140117400  (~3,828 static incoming edges)
```

This remains important architectural context, but runtime negatives show why static prevalence alone is not enough. Generic p3mem hooks/patches are not justified.

## Current technical direction

Immediate runtime question:

> At approximately the same total rendergraph cardinality, which pass types / exact per-pass work counts differ between smooth ~16.7 ms and heavy ~20 ms states?

Current workflow:

```text
matched-cardinality smooth vs heavy
  -> low-rate pass mix / work-count sampling inside RG_CORE
  -> recurse only into the separating pass family/path
  -> if no composition difference, branch timing or CPU stack sampling
  -> map the measured hotspot to the completed 1.58 ↔ 1.60 counterpart set
  -> patch only after the missing-ms budget is measured
```

## Important negative / demoted leads

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
- measured WAIT helper as the sustained regression owner
- raw `RG_CORE` pass/order count as a sufficient explanation

## Telemetry caveat

`SCS frame_start` is not guaranteed to be 1:1 with physically rendered frames. Prefer actual rendered-frametime timing, wall time, synchronized counters and narrow aggregate duration measurements.
