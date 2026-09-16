# RG_CORE runtime localization

Updated: **2026-09-16**

This note records the accepted runtime evidence that narrowed the scene-dependent slowdown from a broad render bucket to a specific rendergraph execution stage and then tested raw rendergraph cardinality as the next discriminator.

## Target build

```text
ETS2:      1.60.1.7s
revision:  26c95e307fd5
renderer:  native DX12
SHA-256:   1D61BA2337E4D8CED85A06E10566A4DF064A2A0919CCD5E51561972D2A04255E
```

Reference `1.58.1.4s` remains static-only and is never run.

## v0.3 — immediate render-child localization

The accepted main-loop boundaries are:

```text
LOOP   0x1401C77C0
PACE   0x1401C6CB0
RENDER 0x1401D72F0
WAIT   0x14011F730
```

`v0.3` retained those boundaries and measured eight selected immediate direct children of `RENDER` with non-overlap accounting. It also derived:

```text
PRE_RENDER_OTHER  = (RENDER_begin - LOOP_begin) - PACE
POST_RENDER_OTHER = LOOP_end - RENDER_end
```

The structural safety gate was clean throughout the run:

```text
order_mismatch                     0
child_active_reentry               0
child_overlap_violation            0
child_bad_end                       0
child_thread_mismatch               0
render_child_sum_gt_render_active   0
other_split_residual                0
```

The clean baseline remained effectively unchanged from v0.2 (`16.685 ms` vs `16.683 ms` LOOP), so no material observer-effect signal was visible.

A clean sustained heavy episode compared with recovered ordinary gameplay produced:

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

The selected render-child delta was overwhelmingly concentrated in:

```text
RG_CORE  1.60.1.7s:0x14021FE20
6.271 ms -> 8.964 ms   +2.693 ms
```

The other selected children were approximately flat or tiny by comparison.

**FACT:** the non-render growth is pre-render work, not post-render cleanup.

**FACT:** among the measured immediate render children, `RG_CORE` owns essentially all positive heavy-state growth.

## Static shape of RG_CORE

`RG_CORE = 0x14021FE20` is a ~2279-byte rendergraph execution routine.

The 1.60 pseudocode shows:

```text
order_base  = *(u32 **)(state + 0x158)
order_count = *(u64 *)(state + 0x160)
pass_base   = *(ptr **)(state + 0xB0)
pass_count  = *(u64 *)(state + 0xB8)
sync_flag   = *(u8 *)(state + 0x218)
```

It iterates the execution-order array, selects a pass from the pass array, validates the index, and dispatches according to a pass type stored at `pass + 0x8`.

## v0.4 — raw cardinality/state sampling

`v0.4` added no new game target detours. The existing `RG_CORE` wrapper read only:

```text
order_count
pass_count
sync_flag
```

at function entry and aggregated them per ~5-second window.

Safety remained clean. The clean `LOOP <= 17 ms` baseline was ~`16.689 ms`, essentially unchanged from v0.3/v0.2.

Across nine clean sustained heavy windows in three natural episodes:

```text
                              GOOD        HEAVY       DELTA
LOOP                          16.689 ms   20.116 ms   +3.427 ms
RENDER_ACTIVE                 10.123 ms   12.071 ms   +1.949 ms
PRE_RENDER_OTHER               6.177 ms    7.602 ms   +1.425 ms
RG_CORE                        5.896 ms    9.417 ms   +3.521 ms
order_count                  157.5       183.5       +26.0
pass_count                   158.5       184.5       +26.0
sync_hits                      0           0           0
```

Two episodes showed large cardinality jumps from roughly `145` to roughly `190+`, so raw pass/order count can contribute.

## Cardinality is not sufficient

One heavy episode already showed a large `RG_CORE` increase with only a modest count increase:

```text
episode 1 pre:   RG_CORE ~8.257 ms, order ~160.1
episode 1 heavy: RG_CORE ~9.952 ms, order ~167.5
```

More decisively, the same run contains many smooth ~16.67 ms windows at high cardinality, including order counts around `190–223`.

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

At essentially the same raw rendergraph cardinality:

```text
RG_CORE heavy - smooth ≈ +2.91 ms
```

There are also smooth windows above 210 order entries with `RG_CORE` around only ~5.7–6.0 ms.

**FACT:** more passes can contribute to heavy episodes.

**FACT:** total pass/order count by itself does not discriminate heavy from smooth.

**FACT:** `sync_flag` was never active in this run (`0 / 86,942` samples), so this synchronization path does not explain the measured heavy states.

## Current discriminator: pass composition / per-pass work

Static inspection of `RG_CORE` shows pass types `1..7` and several exact work-count fields consumed by the runtime path:

```text
pass + 0x1338  raw list count consumed before switch dispatch
pass + 0x19A8  callback/work-object pointer
pass + 0xDB0   count consumed by the type-4 helper
pass + 0x12D0  reference count iterated by the type-7 path
```

The `+0x1338` field is intentionally left with a neutral label because the available pseudocode supports its use as a count but not a stronger semantic name.

The next low-overhead runtime discriminator therefore samples, at low rate, only the passes selected by the actual execution-order array and aggregates:

```text
type1..type7 counts
callback-present count
raw +0x1338 count
type-4 +0xDB0 item count
type-7 +0x12D0 reference count
```

The primary comparison is **smooth vs heavy at matched total pass cardinality**.

If one pass family or work-count metric separates the two states, recurse only into that path. If composition/work counts remain flat while `RG_CORE` still differs by ~3 ms, move to targeted branch timing or differential CPU stack sampling inside `RG_CORE`.

No behavior patch is justified yet.
