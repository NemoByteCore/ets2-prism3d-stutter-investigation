# Current findings

Updated: **2026-09-18**

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

### v0.6 — type-1 execution cost is the major measured owner

v0.6 sampled elapsed time in the type-1, type-4 and type-7 helper paths. The run remained structurally clean and retained normal ~16.67 ms windows.

Exact matched order/pass pair:

```text
                              SMOOTH      HEAVY
LOOP                          16.678 ms   19.330 ms
RG_CORE                        5.225 ms    8.196 ms
order / pass                  168 / 169   168 / 169
type-1 calls / RG_CORE        ~143.9      ~144.0
type-1 sample avg              ~16 us      ~32 us
type-1 estimated/RG_CORE       2.373 ms     4.650 ms
type-4 estimated/RG_CORE       0.004 ms     0.005 ms
type-7 estimated/RG_CORE       0.000 ms     0.000 ms
```

**FACT:** type-1 helper `1.60.1.7s:0x14021F560` owns a large fraction of the measured RG_CORE heavy-state delta.

**FACT:** this is not explained by more type-1 invocations; call count is essentially unchanged while sampled per-call cost rises strongly.

**FACT:** type 4 and type 7 are demoted as major owners in this episode.

## Current RG_CORE branch targets

Static inspection of `0x14021FE20` identifies three clean direct helper paths:

```text
0x14021F560  type-1 helper
0x14021F780  type-4 helper
0x1402DE540  type-7 per-reference helper
```

Type 1 dominates the observed pass mix. Its helper performs device-facing work and invokes a pass-specific callback when present, so it is the strongest first timing discriminator without assuming it is the answer.

The type-4 helper processes its item list and callback/device work. The type-7 helper executes once per type-7 reference.

### v0.7.1 — type-1 cost collapses onto the pass callback

The v0.7.1 drive did not reproduce the earlier canonical sustained 19–21+ ms state, so it does not replace the v0.6 heavy-state budget. It does answer the narrower intra-T1 question cleanly.

At exact matched `order/pass = 151/152`:

```text
                              LOW-COST    HIGH-COST
RG_CORE                        4.681 ms    9.393 ms
type-1 sample avg                206 qpc      603 qpc
device +0x208                      0 qpc        0 qpc
callback +0x8                    200 qpc      596 qpc
pre-tail                         204 qpc      601 qpc
derived final tail                 2 qpc        2 qpc
```

Across accepted full windows, `corr(type-1, callback) ≈ 0.9995`.

**FACT:** nearly all sampled type-1 time and type-1 cost variation is inside the indirect pass callback at `1.60.1.7s:0x14021F73C`.

**FACT:** the measured device +0x208 and final +0x108 tail paths are negligible compared with the callback.

### v0.8 — one callback target, still not the final implementation

v0.8 reproduced the sustained heavy scene state and found that **all 107,739 sampled T1 callbacks** targeted exactly one address:

```text
1.60.1.7s:0x140226A50
```

Exact matched `order/pass = 159/160`:

```text
                              SMOOTH      HEAVY
LOOP                          16.673 ms   21.716 ms
RG_CORE                        5.265 ms   11.121 ms
type-1 sample avg                207 qpc      601 qpc
outer callback avg               203 qpc      597 qpc
```

Targeted static inspection shows that `0x140226A50` is only:

```text
MOV RCX,[RCX+0x110]
MOV RAX,[RCX]
JMP qword ptr [RAX+0x8]
```

So there is no outer callback-target composition shift. The entire sampled population goes through one thunk, and the substantive implementation is one nested vtable dispatch deeper.

### v0.9 — one nested implementation dominates T1 variation

v0.9 resolved the final nested callback targets. The dominant target is:

```text
0x1413BD3F0 -> JMP 0x1413BB140
```

It accounts for about **60.6%** of sampled T1 time and correlates strongly with T1 estimated cost (`r ≈ 0.973`).

Exact `order/pass = 144/145`:

```text
                              LOW-COST    HIGH-COST
RG_CORE                        4.590 ms    8.580 ms
type-1 estimated/RG_CORE       2.245 ms    6.096 ms
winner estimated/RG_CORE       1.421 ms    4.661 ms
winner sampled calls             649         583
winner sample avg qpc             410        1354
```

**FACT:** the winner's cost increase is predominantly per-call slowdown, not increased call count.

Static mapping identifies the substantive 1.60 implementation at `0x1413BB140` and a close 1.58 counterpart at `0x14120D6B0`. A queue-data stride/layout change is present between versions, but remains only a static clue.

## Immediate technical direction

The next discriminator splits seven direct child calls inside `0x1413BB140` and reports residual body cost, only for the already-localized winning T1 path.

```text
winner callback
  -> direct-child timing + residual
  -> identify concrete child/body mechanism
  -> compare that mechanism 1.58 ↔ 1.60
  -> patch only after the missing-ms budget is localized
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