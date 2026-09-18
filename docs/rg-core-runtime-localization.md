# RG_CORE runtime localization

Updated: **2026-09-18**

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

## v0.6 — type-1 helper owns a large fraction of the delta

The v0.6 run passed all safety/accounting checks and preserved ordinary ~16.67 ms windows. It sampled every 16th execution of the three direct helpers.

The decisive result again comes from exact matched cardinality:

```text
order / pass                  168 / 169   168 / 169
                              SMOOTH      HEAVY
LOOP                          16.678 ms   19.330 ms
RG_CORE                        5.225 ms    8.196 ms
type-1 calls / RG_CORE        ~143.9      ~144.0
type-1 sample avg              ~16 us      ~32 us
type-1 estimated/RG_CORE       2.373 ms     4.650 ms
type-4 estimated/RG_CORE       0.004 ms     0.005 ms
type-7 estimated/RG_CORE       0.000 ms     0.000 ms
```

So RG_CORE rises by about **+2.97 ms** while type-1 call count is effectively unchanged. The sampled type-1 contribution rises by about **+2.28 ms**, roughly three quarters of that RG_CORE delta in this pair. Type 4 and type 7 remain effectively flat.

The same direction repeats in exact-cardinality groups `162/163`, `169/170`, `171/172`, and `173/174`.

**FACT:** the type-1 helper at `1.60.1.7s:0x14021F560` is a major owner of the heavy-state RG_CORE cost.

**FACT:** this is not simply more type-1 calls. At matched cardinality, effectively the same number of type-1 invocations becomes substantially more expensive per call.

**FACT:** type-4 and type-7 helpers are demoted as major owners for this episode.

**INFERENCE:** the next useful discriminator is inside the type-1 helper itself, especially its callback/device-facing work.

## v0.7.1 — T1 cost localizes to the pass callback

The corrected v0.7.1 probe aligned three internal measurements to the same sampled type-1 calls:

```text
0x14021F70F  device-facing +0x208 call
0x14021F73C  pass-specific callback through vtable +0x8
0x14021F773  pre-tail marker before the final +0x108 JMP
```

This drive did not reproduce the earlier canonical sustained 19–21+ ms heavy state, so it is not used to replace the v0.6 heavy-state budget. It is still decisive for the narrower intra-T1 question.

At exact matched order/pass cardinality `151 / 152`:

```text
                              LOW-COST    HIGH-COST
LOOP                          16.677 ms   17.503 ms
RG_CORE                        4.681 ms    9.393 ms
type-1 sample avg                206 qpc      603 qpc
device +0x208                      0 qpc        0 qpc
callback +0x8                    200 qpc      596 qpc
pre-tail                         204 qpc      601 qpc
derived final tail                 2 qpc        2 qpc
```

The type-1 increase is `+397 qpc`; the callback increase is `+396 qpc`.

Across accepted full windows, the sampled T1 duration and callback duration correlate at about `0.9995`. Several other exact-cardinality pairs repeat the same one-for-one behavior.

**FACT:** nearly all sampled type-1 time is inside the indirect pass callback at `1.60.1.7s:0x14021F73C`.

**FACT:** the measured device +0x208 path and final +0x108 tail path are negligible compared with the callback.

**FACT:** exact-cardinality T1 cost variation is mirrored almost one-for-one by callback-duration variation.

**INFERENCE:** the next useful discriminator is the actual indirect callback implementation, not another broad T1 split.

## v0.8 — one outer callback target, and it is only a thunk

v0.8 reproduced the canonical sustained heavy state, including several ~19–24 ms windows. Every one of **107,739 sampled T1 callbacks** resolved to the same outer callback target:

```text
1.60.1.7s:0x140226A50
```

At exact matched `order/pass = 159/160`:

```text
                              SMOOTH      HEAVY
LOOP                          16.673 ms   21.716 ms
RG_CORE                        5.265 ms   11.121 ms
type-1 sample avg                207 qpc      601 qpc
outer callback avg               203 qpc      597 qpc
```

So at identical rendergraph cardinality the T1 increase (`+394 qpc`) is mirrored essentially exactly by the one callback target (`+394 qpc`).

Targeted static inspection then showed that `0x140226A50` is not the substantive implementation:

```text
MOV RCX,[RCX+0x110]
MOV RAX,[RCX]
JMP qword ptr [RAX+0x8]
```

It is a dispatch thunk that loads a nested object from `callback_object + 0x110` and tail-jumps through that object's vtable slot `+0x8`.

**FACT:** there is no outer callback-target composition shift in the accepted v0.8 run.

**FACT:** the entire sampled population goes through one thunk.

**FACT:** the actual implementation is one indirect level deeper.

## v0.9 — dominant final implementation identified

v0.9 resolved the nested final callback targets and identified `0x1413BD3F0 -> JMP 0x1413BB140` as the dominant implementation path.

At exact `144/145` cardinality, its estimated contribution rises `1.421 -> 4.661 ms` while sampled count falls `649 -> 583`. The increase is therefore per-call cost rather than increased frequency.

## v0.10 — direct-child split localizes the winner to RQ_ONE

v0.10 split `0x1413BB140` into seven direct child calls plus residual. The run remained structurally clean.

At exact matched `order/pass = 160/161`:

```text
                              SMOOTH      HIGH-COST    DELTA
RG_CORE                        6.925 ms   11.358 ms    +4.433
winner parent                  2.286 ms    5.743 ms    +3.456
RQ_ONE 0x14154C9F0            1.682 ms    4.535 ms    +2.854
RQ_PREP 0x14154C370           0.448 ms    1.023 ms    +0.575
RQ_ALL                         0.127 ms    0.155 ms    +0.028
winner residual                0.022 ms    0.022 ms    ~0
```

`RQ_ONE` sampled calls fall `496 -> 352`, while average sampled duration rises `635 -> 1940 qpc`.

Across normal windows:

```text
corr(winner parent, RQ_ONE)  ≈ 0.9982
corr(winner parent, RQ_PREP) ≈ 0.9681
```

**FACT:** direct children explain essentially all measured winner cost.

**FACT:** winner-body residual is negligible.

**FACT:** `RQ_ONE = 0x14154C9F0` is the dominant owner of winner cost variation; `RQ_PREP` is secondary.

**FACT:** the RQ_ONE slowdown is predominantly per-call, not a higher invocation count.

The close static counterpart is:

```text
1.60 0x14154C9F0, size 1382
1.58 0x1413D7170, size 1417
```

## v0.11 direction

The next probe preserves the accepted chain and times four normal direct callsites inside `RQ_ONE`:

```text
0x14154CA29 -> 0x14154CF60   head dispatch
0x14154CC1B -> 0x1402158E0   view/state update
0x14154CDAE -> 0x140281C80   command allocation
0x14154CDE9 -> 0x14154CF60   inner per-entry dispatch
```

The two `0x14154CF60` sites stay separate because they occur at different structural positions.

No behavior patch is justified yet.
