# Runtime phase localization

Updated: **2026-09-16**

This document summarizes the runtime pivot that followed the completed `1.58.1.4s ↔ 1.60.1.7s` whole-corpus diff.

The purpose of the pivot is simple: stop choosing isolated static candidates one by one and first measure **which coarse CPU phase actually owns the missing milliseconds** in the sustained heavy state.

## Ranked static candidates that were demoted at runtime

### `render_queue_set_t` copy helper

Target:

```text
1.60.1.7s:0x14154AAB0
```

`NemoRenderQueueProbe v0.2` found only **14 direct executions across 24,798 rendered frames**.

**Conclusion:** the static delta is real, but the direct execution cost cannot explain a sustained multi-millisecond-per-frame regression.

### `r_proto` lazy mask-resolution boundary

Target:

```text
1.60.1.7s:0x1413C1470
```

A whole-executable scan found no direct `E8 rel32` callsite to the proposed boundary, and the harvested call graph contains no incoming direct-call edge for it.

**Conclusion:** do not repeat the same direct-call probe without new xref/reachability evidence.

### `traffic_trajectory_t::update_neighbors_bits`

Target:

```text
1.60.1.7s:0x1408DC510
```

The target executed only **85 times** in the whole run, including a 30-call shutdown/unload-adjacent burst. A natural transition from about **16.775 ms/frame** to **22.268 ms/frame** occurred across an approximately **42.17 s interval with no calls to the target at all**.

**Conclusion:** direct execution cost at this target is far too sparse to own the sustained frame budget.

## Coarse main-loop timing

The active main-loop chain is:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
```

`NemoFramePhaseProbe v0.1` measured:

```text
LOOP   = duration of 0x1401C77C0
PACE   = nested duration of 0x1401C6CB0
RENDER = nested duration of 0x1401D72F0
OTHER  = LOOP - PACE - RENDER
```

Two natural heavy transitions showed similar coarse growth:

```text
transition A: LOOP +4.116 ms, RENDER +1.641 ms, OTHER +2.475 ms
transition B: LOOP +3.745 ms, RENDER +2.007 ms, OTHER +1.738 ms
```

`PACE` remained around `~0.001 ms/iteration` and is not a meaningful owner of the regression.

## v0.2: WAIT separation resolves the main ambiguity

Static inspection of `0x1401D72F0` identified a nested timing helper at:

```text
1.60.1.7s:0x14011F730
```

It uses `Sleep()` followed by a spin-until-target loop. `NemoFramePhaseProbe v0.2` therefore added:

```text
WAIT = 0x14011F730

RENDER_ACTIVE = RENDER - WAIT
OTHER         = LOOP - PACE - RENDER
CPU_ACTIVE    = OTHER + RENDER_ACTIVE + PACE
```

Instrumentation remained clean:

```text
LOOP   calls = 33,339
PACE   calls = 33,339
RENDER calls = 33,337
WAIT   calls =  2,092
bad_end = 0
thread mismatch = 0
```

The WAIT helper is therefore **conditional and sparse**, not a global once-per-frame pacing gate.

## Aggregate good-vs-heavy result

Using ordinary gameplay phase windows `8..111`, classifying clean good windows as `LOOP <= 17.0 ms` and heavy windows as `LOOP >= 19.0 ms`, then weighting by loop-call count:

```text
             GOOD       HEAVY      DELTA
LOOP         16.683 ms  20.358 ms  +3.675 ms
RENDER_ACTIVE11.404 ms  13.507 ms  +2.103 ms
OTHER         5.253 ms   6.840 ms  +1.586 ms
WAIT          0.024 ms   0.009 ms  -0.015 ms / loop
CPU_ACTIVE   16.659 ms  20.349 ms  +3.690 ms
```

At this coarse resolution, roughly **57%** of the measured heavy-state delta is in `RENDER_ACTIVE` and roughly **43%** is in `OTHER`.

The WAIT contribution is only tens of microseconds per loop and actually decreases slightly in heavy windows.

**Conclusion:** the v0.1 RENDER increase was not just lost/redistributed time in this measured sleep/spin helper. Active render-side CPU work genuinely rises.

## Natural episode checks

The aggregate split is not driven by one isolated outlier.

### Episode A

Immediate baseline vs full heavy/recovery episode:

```text
LOOP          +3.027 ms
RENDER_ACTIVE +1.370 ms
OTHER         +1.673 ms
WAIT          -0.017 ms
```

Central peak:

```text
LOOP          +3.870 ms
RENDER_ACTIVE +2.172 ms
OTHER         +1.712 ms
```

### Episode B

Immediate baseline vs heavy/recovery:

```text
LOOP          +1.561 ms
RENDER_ACTIVE +0.690 ms
OTHER         +0.882 ms
WAIT          -0.010 ms
```

Central peak:

```text
LOOP          +2.311 ms
RENDER_ACTIVE +1.265 ms
OTHER         +1.065 ms
```

The independent `NemoFrame` logger tracks both episodes in the same direction.

## Current interpretation

**FACT:** `PACE` is negligible.

**FACT:** the measured WAIT helper does not own the heavy-state regression.

**FACT:** both active render work and non-render residual main-loop work increase materially in heavy state.

**INFERENCE:** at this resolution the missing CPU budget is distributed across two coarse active buckets. A shared scene/state/cardinality driver may still be responsible for both, so this does **not** yet prove two independent bugs.

## Next step

The project now recurses only inside measured active buckets:

1. subdivide `RENDER_ACTIVE` inside `0x1401D72F0` using stable low-overhead boundaries;
2. subdivide `OTHER` inside `0x1401C77C0` into stable pre/post-render or equivalent subphases;
3. if the result remains distributed, use differential CPU stack sampling between sustained good and heavy windows;
4. add state/cardinality counters only inside a measured winning subphase;
5. map the measured hotspot back to the completed `1.58 ↔ 1.60` counterpart set;
6. patch only after a concrete millisecond budget exists.

This keeps the investigation runtime-driven instead of returning to visually attractive but potentially irrelevant static deltas.
