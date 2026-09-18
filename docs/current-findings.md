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

The tested setup can hold roughly `16.67 ms / 60 FPS` in light states and sustain roughly `19–25 ms` in heavier scene compositions. The slower state is scene-dependent rather than a single hitch and can sometimes clear after an unload/ferry/teleport transition without restarting the process.

## Broad frame-budget localization

The accepted broad split is:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
POST_RENDER_OTHER               0.381 ms    0.380 ms  ~0
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

**FACT:** the measured WAIT helper does not own the sustained slowdown.

**FACT:** non-render growth is in `PRE_RENDER_OTHER`, not post-render cleanup.

**FACT:** the selected render-side positive growth is overwhelmingly in `RG_CORE = 1.60.1.7s:0x14021FE20`.

`PRE_RENDER_OTHER` remains a real secondary branch for later. The current investigation deliberately finishes the render branch first.

## Accepted render-side localization chain

```text
RG_CORE 0x14021FE20
  -> T1 helper 0x14021F560
    -> pass callback 0x14021F73C
      -> outer thunk 0x140226A50
        -> nested winner 0x1413BD3F0
          -> JMP 0x1413BB140
            -> RQ_ONE 0x14154C9F0
              -> HEAD_DISPATCH 0x14154CF60
                -> NOSPLIT 0x14154CFA7
                  -> downstream 0x1402D8D20
```

The important result is not the chain by itself; it is that each recursion step was selected by measured elapsed-time ownership rather than static appearance.

## Why the current leaf is strong

### Rendergraph count and coarse composition are insufficient

Heavy windows can have more rendergraph passes, but matched-cardinality windows remain several milliseconds apart.

At effectively equal `order/pass` count:

```text
                              SMOOTH      HEAVY
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

Matched pass-type/work composition also fails to explain the gap. The same broad workload can execute materially slower.

### T1 helper owns most of the measured RG_CORE delta

At exact `168/169` cardinality:

```text
                              SMOOTH      HEAVY
RG_CORE                        5.225 ms    8.196 ms
T1 calls / RG_CORE            ~143.9      ~144.0
T1 estimated / RG_CORE         2.373 ms    4.650 ms
```

Type-4 and type-7 helper timing is effectively flat in the same comparison.

### The pass callback owns the T1 variation

At exact `151/152` cardinality:

```text
T1 sample avg       206 -> 603 qpc
callback            200 -> 596 qpc
device +0x208         0 ->   0 qpc
derived final tail    2 ->   2 qpc
```

Across accepted windows, sampled T1 and callback duration correlate at about `0.9995`.

### One nested implementation became the dominant callback owner

The dominant nested target is:

```text
1.60.1.7s:0x1413BD3F0
  -> JMP 1.60.1.7s:0x1413BB140
```

At exact `144/145` cardinality:

```text
winner estimated / RG_CORE   1.421 -> 4.661 ms
winner sampled calls           649 -> 583
winner avg qpc                  410 -> 1354
```

The target becomes much more expensive per call even while sampled count falls.

### The winner collapses mostly onto RQ_ONE

At exact `160/161` cardinality:

```text
                              SMOOTH      HIGH-COST
winner parent                  2.286 ms    5.743 ms
RQ_ONE 0x14154C9F0            1.682 ms    4.535 ms
RQ_PREP 0x14154C370           0.448 ms    1.023 ms
winner residual                0.022 ms    0.022 ms
```

`RQ_ONE` is the dominant measured child; `RQ_PREP` is real but secondary and remains in backlog.

### RQ_ONE collapses almost completely onto HEAD_DISPATCH

Across the full accepted run:

```text
RQ_ONE parent total_qpc      24,363,328
HEAD_DISPATCH total_qpc      24,324,287
RQ_ONE residual_qpc              39,041
```

At exact `156/157` cardinality:

```text
                              LOW-COST    HIGH-COST
RQ_ONE parent                  1.855 ms    4.002 ms
HEAD_DISPATCH                  1.850 ms    3.997 ms
HEAD samples                     398         386
HEAD avg qpc                     842        1727
```

**FACT:** `HEAD_DISPATCH = 1.60.1.7s:0x14154CF60` accounts for about **99.84%** of sampled `RQ_ONE` time in this run.

**FACT:** the slowdown is predominantly per-call, not increased invocation frequency.

## Current static counterparts

```text
1.60.1.7s:0x1413BB140  <->  1.58.1.4s:0x14120D6B0
1.60.1.7s:0x14154C9F0  <->  1.58.1.4s:0x1413D7170
1.60.1.7s:0x14154CF60  <->  1.58.1.4s:0x1413D7700
1.60.1.7s:0x1402D8D20  <->  1.58.1.4s:0x1401EC530
```

The current `HEAD_DISPATCH` implementations are both 491 bytes and have very similar high-level structure. The useful question is therefore runtime path/cost behavior, not size alone.

## Current discriminator

The HEAD_DISPATCH path split is decisive:

```text
HEAD sampled                32,359
HEAD total_qpc          31,679,799
NOSPLIT calls               32,359
NOSPLIT total_qpc        31,593,076
RANGE calls                      0
HEAD residual_qpc           86,723
```

At exact `order/pass = 143/144`:

```text
HEAD_DISPATCH             2.585 -> 3.869 ms
NOSPLIT_DISPATCH          2.578 -> 3.863 ms
NOSPLIT samples             459 -> 407
NOSPLIT avg qpc             958 -> 1512
HEAD residual             ~0.007 ms flat
```

**FACT:** the accepted sampled HEAD_DISPATCH path is exclusively the no-split callsite.

**FACT:** RANGE was not observed.

**FACT:** essentially all material HEAD_DISPATCH variation is inherited from `1.60.1.7s:0x1402D8D20`.

Current downstream static counterpart:

```text
1.60.1.7s:0x1402D8D20  <->  1.58.1.4s:0x1401EC530
```

The next measurement splits the normal direct children inside `0x1402D8D20` and reports residual body cost.
No behavior patch is justified yet.

## Secondary validated branch: descriptor/root-binding architecture

The mapped 1.58/1.60 DX12 descriptor/root-binding architecture changed materially. Fixed-profile sampler reservation/copy pressure is real, and sampler-table reuse removes roughly **95%** of targeted allocation/copy work with clean safety counters.

The sustained scene-dependent slowdown still occurs afterward.

**Conclusion:** this is a real optimization/regression component, not the complete explanation.

See [`shader-profile-architecture-delta.md`](shader-profile-architecture-delta.md).

## Runtime-demoted leads

Do not promote these again without new evidence:

- direct sustained-cost theory for `1.60.1.7s:0x14154AAB0`;
- repeated direct-call probing of `1.60.1.7s:0x1413C1470`;
- direct sustained-cost theory for `1.60.1.7s:0x1408DC510`;
- measured WAIT helper as the sustained owner;
- raw rendergraph cardinality as a sufficient explanation;
- coarse pass-type/work composition as a sufficient explanation;
- DirectStorage introduction;
- broad D3D hooks for performance attribution.

See [`disproven-hypotheses.md`](disproven-hypotheses.md).

## Whole-corpus diff

The static comparison remains useful for counterpart recovery:

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

## Evidence discipline

Public conclusions should distinguish:

- **FACT** — directly measured or statically verified;
- **INFERENCE** — the most direct interpretation of measured evidence;
- **HYPOTHESIS** — a plausible mechanism still awaiting a discriminator.

Addresses are build-specific. Never carry them between versions without re-identification.
