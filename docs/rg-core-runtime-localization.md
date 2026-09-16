# RG_CORE runtime localization

Updated: **2026-09-16**

This note tracks the runtime narrowing of the scene-dependent CPU slowdown into the rendergraph execution stage and records the discriminators that have already been tested.

## Target build

```text
ETS2:      1.60.1.7s
revision:  26c95e307fd5
renderer:  native DX12
SHA-256:   1D61BA2337E4D8CED85A06E10566A4DF064A2A0919CCD5E51561972D2A04255E
```

`1.58.1.4s` remains static-only and is never run.

## v0.3 — render child localization

The accepted measured chain is:

```text
0x1401C77C0  LOOP
  -> 0x1401C6CB0  PACE
  -> 0x1401D72F0  RENDER
       -> 0x14011F730  WAIT
       -> 0x14021FE20  RG_CORE
```

`v0.3` also derives:

```text
PRE_RENDER_OTHER  = (RENDER_begin - LOOP_begin) - PACE
POST_RENDER_OTHER = LOOP_end - RENDER_end
```

A clean sustained heavy episode versus recovered ordinary gameplay produced:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
RENDER_ACTIVE                   9.983 ms   11.593 ms  +1.610 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
POST_RENDER_OTHER               0.381 ms    0.380 ms  ~0
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

All overlap/reentry/thread/accounting safety counters stayed clean, and the smooth baseline was effectively unchanged from v0.2.

**FACT:** the non-render growth is pre-render work, not post-render cleanup.

**FACT:** among the selected immediate render children, the positive heavy-state growth is concentrated almost entirely in `RG_CORE = 0x14021FE20`.

## Static shape of RG_CORE

`RG_CORE` is a ~2279-byte rendergraph execution routine. It iterates an execution-order array, resolves each selected pass, and dispatches by a type at `pass + 0x8`.

Relevant state fields:

```text
order_base  = state + 0x158
order_count = state + 0x160
pass_base   = state + 0xB0
pass_count  = state + 0xB8
sync_flag   = state + 0x218
```

## v0.4 — raw cardinality is real but insufficient

`v0.4` sampled only `order_count`, `pass_count`, and `sync_flag` at the already-instrumented RG_CORE entry.

Across clean sustained heavy windows:

```text
                              GOOD        HEAVY       DELTA
LOOP                          16.689 ms   20.116 ms   +3.427 ms
PRE_RENDER_OTHER               6.177 ms    7.602 ms   +1.425 ms
RG_CORE                        5.896 ms    9.417 ms   +3.521 ms
order_count                  157.5       183.5       +26.0
pass_count                   158.5       184.5       +26.0
```

Some heavy episodes genuinely contain more passes. However, matched-cardinality windows showed:

```text
                              SMOOTH      HEAVY
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

The synchronization flag was never active (`0 / 86,942`).

**FACT:** total pass/order count is not sufficient to discriminate heavy from smooth.

## v0.5 — coarse pass composition/work is also insufficient

`v0.5` kept the accepted timing/cardinality telemetry and, every 16th RG_CORE invocation, sampled only passes selected by the real execution-order array. It aggregated:

```text
pass types 1..7
callback-present count
raw pass+0x1338 count
type-4 pass+0xDB0 item count
type-7 pass+0x12D0 reference count
```

The run was structurally clean:

```text
LOOP calls                         97,290
RENDER calls                       97,287
RG_CORE calls                      97,287
loops_without_render                    3
order_mismatch                          0
child_active_reentry                    0
child_overlap_violation                 0
child_bad_end                            0
child_thread_mismatch                    0
render_child_sum_gt_render_active       0
sync hits                               0
```

The decisive result is matched **cardinality and composition**, not a broad good-vs-heavy average.

### Example A — exact 159/160 order/pass count

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
 type4 +0xDB0                    10          10
 type7 +0x12D0                    7           7
```

At effectively identical sampled work composition, heavy `RG_CORE` is about **+2.99 ms** slower.

### Example B — exact 158/159 order/pass count

```text
                              SMOOTH      HEAVY
LOOP                          16.704 ms   20.024 ms
RG_CORE                        5.126 ms    9.904 ms
order / pass                  158 / 159   158 / 159
 type 1                         134         134
 type 4                          10          10
 type 6                           5           5
 type 7                           8           8
```

Here the same coarse pass composition differs by about **+4.78 ms** inside RG_CORE.

Another matched pair at `157 / 158` differs by about **+4.10 ms** in RG_CORE with essentially the same sampled type/work counts.

A matched-cardinality aggregate over pass counts around `150..175` tells the same story: smooth RG_CORE is roughly `5.9 ms`, while the heavy subset is roughly `9.8 ms`, despite only small differences in the sampled type counts and raw work-count fields.

**FACT:** neither total cardinality nor the measured coarse pass-type/work-count composition explains the full heavy-state cost.

**INFERENCE:** the same broad rendergraph work is becoming materially more expensive to execute. The next discriminator must time execution paths rather than add more count fields.

## Static branch targets for the next timing step

The `RG_CORE` switch contains three clean direct helper paths suitable for narrow timing:

```text
0x14021F560  type-1 helper
0x14021F780  type-4 helper
0x1402DE540  type-7 per-reference helper
```

The type-1 helper is particularly interesting because type 1 dominates the pass mix and the helper includes device-facing work plus a pass-specific callback when present.

The type-4 helper consumes the type-4 item list and also invokes pass/device work.

The type-7 helper is called once per reference on the type-7 path.

## v0.6 discriminator

The next probe keeps v0.3 timing and v0.4 cardinality, removes the v0.5 pass-mix scan, and samples only every 16th execution of the three direct helper callsites above.

The branch timings are nested inside RG_CORE and are **not** added to the RENDER child sum, preserving the existing non-overlap accounting model.

Acceptance logic:

- if type-1 timing owns most of the smooth-vs-heavy RG_CORE delta, recurse inside `0x14021F560` (especially callback/device-facing work);
- if type 4 or type 7 separates the states, recurse only into that branch;
- if none of the three explains the missing milliseconds, treat the remaining RG_CORE body/tail/type-2/3/5/6 paths as the residual and move to targeted branch timing or differential CPU stack sampling.

No behavior patch is justified yet.