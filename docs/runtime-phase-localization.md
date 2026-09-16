# Runtime phase localization

Updated: **2026-09-16**

This document summarizes the runtime pivot that followed the completed `1.58.1.4s ↔ 1.60.1.7s` whole-corpus diff.

The purpose of the pivot is simple: stop choosing isolated static candidates one by one and first measure **which CPU phase actually owns the missing milliseconds** in the sustained heavy state.

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

The target executed only **85 times** in the whole run. A natural transition from about **16.775 ms/frame** to **22.268 ms/frame** occurred across an approximately **42.17 s interval with no calls to the target at all**.

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

The WAIT helper is **conditional and sparse**, not a global once-per-frame pacing gate.

Using clean ordinary gameplay windows:

```text
             GOOD       HEAVY      DELTA
LOOP         16.683 ms  20.358 ms  +3.675 ms
RENDER_ACTIVE11.404 ms  13.507 ms  +2.103 ms
OTHER         5.253 ms   6.840 ms  +1.586 ms
WAIT          0.024 ms   0.009 ms  -0.015 ms / loop
CPU_ACTIVE   16.659 ms  20.349 ms  +3.690 ms
```

**Conclusion:** the v0.1 RENDER increase was not just lost/redistributed time in this measured wait helper. Active render-side CPU work genuinely rises.

## v0.3: split the active buckets

`v0.3` retained the accepted LOOP/PACE/RENDER/WAIT boundaries and added selected immediate direct-child timing inside `RENDER` using callsite-specific wrappers with non-overlap accounting.

It also derives:

```text
PRE_RENDER_OTHER  = (RENDER_begin - LOOP_begin) - PACE
POST_RENDER_OTHER = LOOP_end - RENDER_end
```

The structural accounting gate passed throughout the run: no overlap, reentry, bad-end, thread-mismatch or child-sum violations were observed.

The clean `LOOP <= 17 ms` baseline remained effectively unchanged from v0.2 (`16.685 ms` vs `16.683 ms`).

A clean sustained heavy episode compared with recovered ordinary gameplay:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
RENDER_ACTIVE                   9.983 ms   11.593 ms  +1.610 ms
OTHER                           6.667 ms    8.132 ms  +1.465 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
POST_RENDER_OTHER               0.381 ms    0.380 ms  ~0
RENDER_CHILD_SUM                6.645 ms    9.367 ms  +2.722 ms
RENDER_SELF_RESIDUAL            3.337 ms    2.226 ms  -1.112 ms
```

The selected immediate render-child delta was:

```text
RG_CORE 0x14021FE20       +2.693 ms
RG_PRESENT_RESOLVE        +0.021 ms
all other selected children approximately flat/tiny
```

**Conclusion:** the non-render growth is specifically pre-render work, while the selected render-side positive delta is overwhelmingly localized to `RG_CORE 0x14021FE20`.

## v0.4: cardinality/state sampling at RG_CORE

The `RG_CORE` pseudocode iterates a rendergraph execution-order array:

```text
order_base  = *(u32 **)(state + 0x158)
order_count = *(u64 *)(state + 0x160)
pass_base   = *(ptr **)(state + 0xB0)
pass_count  = *(u64 *)(state + 0xB8)
sync_flag   = *(u8 *)(state + 0x218)
```

`v0.4` reused the existing `RG_CORE` wrapper and sampled only `order_count`, `pass_count` and `sync_flag` at entry. No new game target detours were added.

Safety remained clean. The clean gameplay baseline was about `16.689 ms`, effectively unchanged from v0.3/v0.2.

Across nine clean heavy windows in three natural episodes:

```text
                              GOOD        HEAVY       DELTA
LOOP                          16.689 ms   20.116 ms   +3.427 ms
RENDER_ACTIVE                 10.123 ms   12.071 ms   +1.949 ms
PRE_RENDER_OTHER               6.177 ms    7.602 ms   +1.425 ms
RG_CORE                        5.896 ms    9.417 ms   +3.521 ms
order_count                  157.5       183.5       +26.0
pass_count                   158.5       184.5       +26.0
```

Two episodes showed large count jumps from roughly `145` to roughly `190+`. Therefore raw rendergraph cardinality can contribute to heavy episodes.

### Matched-cardinality negative

The same run also contains many smooth ~16.67 ms windows with order counts around `190–223`.

Matched-cardinality comparison (`order_avg 184..205`):

```text
                              SMOOTH      HEAVY
LOOP                          16.696 ms   19.851 ms
RENDER_ACTIVE                  9.642 ms   11.935 ms
PRE_RENDER_OTHER               6.615 ms    7.473 ms
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

At essentially the same pass/order count, `RG_CORE` is about **+2.91 ms slower** in the heavy state.

The synchronization flag was never active:

```text
sync_hits = 0 / 86,942
```

**Conclusion:** more passes can matter, but total pass count alone is not the root discriminator. The same count can execute substantially slower.

## Current interpretation

**FACT:** `PACE` is negligible.

**FACT:** the measured WAIT helper does not own the heavy-state regression.

**FACT:** non-render growth is pre-render, not post-render.

**FACT:** among the selected render children, `RG_CORE 0x14021FE20` owns essentially all positive heavy-state growth.

**FACT:** raw `order_count` / `pass_count` can rise with heavy state but are not sufficient to distinguish heavy from smooth.

**FACT:** the measured `sync_flag` path is inactive in the accepted v0.4 run.

**INFERENCE:** the strongest next discriminator is rendergraph pass composition and/or work performed inside individual pass paths.

## Next step

Do not return to broad static candidate roulette.

The next low-overhead experiment keeps all accepted timing/cardinality instrumentation and samples only the passes actually selected by the `RG_CORE` execution-order array, at low rate.

Aggregate per sample:

```text
pass type 1..7 counts
callback/work-object presence at pass+0x19A8
raw list count at pass+0x1338
type-4 item count at pass+0xDB0
type-7 reference count at pass+0x12D0
```

Primary comparison: **smooth vs heavy at matched total pass cardinality**.

If one composition/work metric separates the states, recurse only into that pass family/path. If the mix and work counts remain flat while `RG_CORE` still differs by ~3 ms, move to targeted branch timing or differential CPU stack sampling inside `RG_CORE`.

See [`rg-core-runtime-localization.md`](rg-core-runtime-localization.md).
