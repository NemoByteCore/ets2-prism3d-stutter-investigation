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

The target executed only **85 times** in the whole run, including a 30-call shutdown/unload-adjacent burst. A natural transition from about **16.775 ms/frame** to **22.268 ms/frame** occurred across an approximately **42.17 s interval with no calls to the target at all**.

**Decision:** direct execution cost is far too sparse to own the sustained frame budget.

## Runtime-first phase localization

The current main-loop chain is:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
```

`NemoFramePhaseProbe v0.1` first showed that heavy-state growth is split between the broad render coordinator and residual main-loop work.

`PACE` stayed around `~0.001 ms/iteration` and is negligible.

## v0.2 resolves the WAIT ambiguity

`NemoFramePhaseProbe v0.2` added a nested boundary around:

```text
1.60.1.7s:0x14011F730
```

This function is statically a `Sleep()` + spin-until-target timing helper.

Derived buckets:

```text
RENDER_ACTIVE = RENDER - WAIT
OTHER         = LOOP - PACE - RENDER
CPU_ACTIVE    = OTHER + RENDER_ACTIVE + PACE
```

Instrumentation completed cleanly:

```text
LOOP   calls = 33,339
PACE   calls = 33,339
RENDER calls = 33,337
WAIT   calls =  2,092
bad_end = 0
thread mismatch = 0
```

The WAIT helper is conditional and sparse rather than a once-per-frame global pacing gate.

## Clean good-vs-heavy result

Using ordinary gameplay windows, classifying good as `LOOP <= 17.0 ms` and heavy as `LOOP >= 19.0 ms`, weighted by loop-call count:

```text
             GOOD       HEAVY      DELTA
LOOP         16.683 ms  20.358 ms  +3.675 ms
RENDER_ACTIVE11.404 ms  13.507 ms  +2.103 ms
OTHER         5.253 ms   6.840 ms  +1.586 ms
WAIT          0.024 ms   0.009 ms  -0.015 ms / loop
CPU_ACTIVE   16.659 ms  20.349 ms  +3.690 ms
```

**FACT:** measured WAIT does not explain the heavy-state slowdown.

**FACT:** active render-side CPU work genuinely rises by about `+2.10 ms` in the aggregate comparison.

**FACT:** residual non-render main-loop work also rises by about `+1.59 ms`.

At this coarse resolution the measured delta is roughly **57% `RENDER_ACTIVE` / 43% `OTHER`**.

Two separate natural episodes independently show the same direction. See [`runtime-phase-localization.md`](runtime-phase-localization.md).

## Interpretation

The regression is no longer consistent with one coarse wait/pacing bucket consuming the missing time.

The missing CPU budget is distributed across active render work and other main-loop work at this resolution.

A shared scene/state/cardinality driver may increase both buckets, so this is **not** proof of two separate bugs.

## Descriptor / root-binding architecture remains a confirmed component

The mapped 1.58 DX12 path derives root signatures and descriptor capacities from actual pipeline layout. The mapped 1.60 path selects one of 13 fixed shader/root-signature profiles with fixed capacities, separate root CBVs and split per-set tables.

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

This remains important architectural context, but recent runtime negatives show why static prevalence alone is not enough. Generic p3mem hooks/patches are not justified.

## Current technical direction

Current workflow:

```text
coarse runtime phase localization
  -> subdivide measured active buckets
  -> good-vs-heavy differential stack/counter work if still distributed
  -> map measured hotspot to the completed 1.58 ↔ 1.60 counterpart set
  -> patch only after the missing-ms budget is measured
```

Immediate next work:

1. split `RENDER_ACTIVE` inside `0x1401D72F0` around stable low-overhead boundaries;
2. split `OTHER` inside `0x1401C77C0` into stable pre/post-render or equivalent subphases;
3. use differential CPU stack sampling if the subphase split is still broad/noisy.

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

## Telemetry caveat

`SCS frame_start` is not guaranteed to be 1:1 with physically rendered frames. Prefer actual rendered-frametime timing, wall time, synchronized counters and narrow aggregate duration measurements.
