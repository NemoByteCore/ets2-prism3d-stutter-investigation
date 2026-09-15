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

The helper was the strongest new steady-state static candidate after the global diff, but `NemoRenderQueueProbe v0.2` found only **14 direct executions across 24,798 rendered frames**.

Observed direct callsite totals:

```text
0x013C2A22 = 12
0x013C3289 = 1
0x013C329D = 1
0x0145CF47 = 0
0x0154E4C4 = 0
```

**Conclusion:** the direct cost of this helper cannot explain a sustained `+3–8 ms/frame` regression. The static delta remains real, but the direct-cost theory is strongly demoted.

### `r_proto` lazy mask-resolution boundary

Target:

```text
1.60.1.7s:0x1413C1470
```

A whole-executable scan found no direct `E8 rel32` callsite to the proposed boundary, and the harvested call graph contains no incoming direct-call edge for it.

**Conclusion:** do not repeat the same direct-call probe without new xref/reachability evidence. This is a structural negative result, not proof that all related `r_proto` behavior is irrelevant.

### `traffic_trajectory_t::update_neighbors_bits`

Target:

```text
1.60.1.7s:0x1408DC510
```

Four direct callsites were validated and the runtime probe installed successfully.

Runtime totals:

```text
total calls = 85
ordinary gameplay calls ≈ 55
shutdown/unload-adjacent burst = 30
```

The same run captured a clean natural transition from approximately:

```text
baseline weighted mean ≈ 16.775 ms/frame
heavy weighted mean    ≈ 22.268 ms/frame
sustained delta        ≈ +5.49 ms/frame
```

There was an approximately **42.17 s call-free interval centered on the heavy-state onset**.

**Conclusion:** direct execution cost at this target is far too sparse to own the sustained frame budget. The later increase in call rate is more plausibly a marker of a busier scene than the direct cause of the regression.

## Coarse main-loop timing

After those negatives, the investigation switched to a runtime-first main-loop chain recovered from the available 1.60 corpus:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
```

`NemoFramePhaseProbe v0.1` measures:

```text
LOOP   = duration of 0x1401C77C0
PACE   = nested duration of 0x1401C6CB0
RENDER = nested duration of 0x1401D72F0
OTHER  = LOOP - PACE - RENDER
```

The probe completed with sane nesting/call counts:

```text
LOOP calls   = 29,590
PACE calls   = 29,590
RENDER calls = 29,588
bad_end      = 0
thread mismatch = 0
```

Two separate natural good→heavy transitions gave similar deltas.

### Transition A

```text
LOOP   +4.116 ms
RENDER +1.641 ms
OTHER  +2.475 ms
```

### Transition B

```text
LOOP   +3.745 ms
RENDER +2.007 ms
OTHER  +1.738 ms
```

`PACE` remained around `~0.001 ms/iteration` and is not a meaningful owner of the regression.

## Important interpretation caveat

The broad `RENDER` bucket is not pure active renderer work.

Static inspection of `0x1401D72F0` identified a nested frame-time wait helper at:

```text
1.60.1.7s:0x14011F730
```

The helper uses `Sleep()` followed by a short spin phase to reach the target frame time. Therefore changes in the v0.1 `RENDER` bucket can reflect either:

- more active render/present-side CPU work, or
- less/more time left for deliberate frame pacing wait.

This is why v0.1 does **not** yet justify attributing the observed `+1.6–2.0 ms` RENDER delta directly to renderer work.

## Current experiment

`NemoFramePhaseProbe v0.2` separates the nested wait helper from the broad render bucket.

Derived buckets:

```text
RENDER_ACTIVE = RENDER - WAIT
OTHER         = LOOP - PACE - RENDER
CPU_ACTIVE    = OTHER + RENDER_ACTIVE + PACE
```

The decisive question is now:

> During the natural heavy state, does `RENDER_ACTIVE` grow materially, or does the broad RENDER change mostly represent lost frame-pacing wait time while CPU work grows elsewhere?

Interpretation:

- `WAIT` falls while `RENDER_ACTIVE` stays roughly flat -> extra CPU work is mainly outside active render and consumes time previously available for pacing wait;
- `RENDER_ACTIVE` also rises materially -> the regression budget is split between active render-side work and `OTHER` main-loop work;
- neither bucket explains enough -> move to differential CPU stack sampling between sustained good and heavy windows.

## Current status

The investigation has moved from:

```text
static ranking -> isolated probes
```

to:

```text
runtime phase localization -> narrower stack/counter work -> exact 1.58/1.60 comparison -> patch
```

This reduces the chance of spending more time on visually attractive static deltas that are too infrequent to matter at runtime.