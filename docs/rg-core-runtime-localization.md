# RG_CORE runtime localization

Updated: **2026-09-18**

This document keeps only the accepted evidence ladder for the render-side branch. Detailed probe-version chronology is intentionally omitted; the public question is **what has been measured, what was ruled down, and where the current leaf is**.

## Target build

```text
ETS2:      1.60.1.7s
revision:  26c95e307fd5
renderer:  native DX12
SHA-256:   1D61BA2337E4D8CED85A06E10566A4DF064A2A0919CCD5E51561972D2A04255E
```

`1.58.1.4s` remains static-only and is never run.

## Entry point

Broad phase timing localized the selected render-side growth to:

```text
1.60.1.7s:0x14021FE20  RG_CORE
```

A representative recovered-vs-heavy comparison:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

The measured WAIT helper is not the owner of this increase.

## Evidence ladder

| Stage | Address / path | Decisive evidence | Conclusion |
|---|---|---|---|
| Rendergraph core | `0x14021FE20` | ~+2.7 ms in the accepted broad split | Main measured render-side owner |
| Cardinality | `order_count` / `pass_count` | Smooth and heavy windows exist at nearly identical counts | Count alone is insufficient |
| Coarse pass mix | types/work counters | Matched composition still differs by several ms | Broad workload shape is insufficient |
| Type-1 helper | `0x14021F560` | Exact-cardinality contribution rises `2.373 -> 4.650 ms` at flat call count | Major RG_CORE owner |
| Pass callback | site `0x14021F73C` | T1 `206 -> 603 qpc`, callback `200 -> 596 qpc` | T1 variation is almost entirely callback time |
| Nested implementation | `0x1413BD3F0 -> 0x1413BB140` | Estimated contribution `1.421 -> 4.661 ms` while samples fall `649 -> 583` | Dominant callback implementation gets slower per call |
| Winner child | `RQ_ONE 0x14154C9F0` | `1.682 -> 4.535 ms` in exact `160/161` comparison | Dominant child of winner |
| RQ_ONE child | `HEAD_DISPATCH 0x14154CF60` | ~99.84% of sampled RQ_ONE time | Current measured leaf |

## Why pass count is not enough

Some heavy scenes genuinely contain more rendergraph passes. That is real, but it is not sufficient.

Matched-cardinality example:

```text
                              SMOOTH      HEAVY
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

The synchronization flag was inactive throughout the accepted cardinality run.

## Why T1 became the focus

At exact `order/pass = 168/169`:

```text
                              SMOOTH      HEAVY
RG_CORE                        5.225 ms    8.196 ms
T1 calls / RG_CORE            ~143.9      ~144.0
T1 estimated / RG_CORE         2.373 ms    4.650 ms
```

Type-4 and type-7 helper timing is effectively flat.

**FACT:** increased T1 frequency is not the explanation; per-call cost rises.

## Callback localization

Inside T1, the relevant paths were separated:

```text
device-facing +0x208
pass callback +0x8
final +0x108 tail
```

At exact `151/152` cardinality:

```text
T1 sample avg       206 -> 603 qpc
callback            200 -> 596 qpc
device +0x208         0 ->   0 qpc
derived final tail    2 ->   2 qpc
```

Across accepted windows, sampled T1 and callback duration correlate at about `0.9995`.

The outer callback target itself is only a thunk:

```text
1.60.1.7s:0x140226A50

MOV RCX,[RCX+0x110]
MOV RAX,[RCX]
JMP qword ptr [RAX+0x8]
```

Resolving the nested target identified the dominant implementation:

```text
1.60.1.7s:0x1413BD3F0
  -> JMP 0x1413BB140
```

## Winner split

At exact `order/pass = 160/161`:

```text
                              SMOOTH      HIGH-COST    DELTA
winner parent                  2.286 ms    5.743 ms    +3.456
RQ_ONE 0x14154C9F0            1.682 ms    4.535 ms    +2.854
RQ_PREP 0x14154C370           0.448 ms    1.023 ms    +0.575
winner residual                0.022 ms    0.022 ms    ~0
```

`RQ_ONE` sampled calls fall `496 -> 352`, while average sampled duration rises `635 -> 1940 qpc`.

**FACT:** `RQ_ONE` is the dominant child; `RQ_PREP` is secondary; winner-body residual is negligible.

## Current leaf: HEAD_DISPATCH

Splitting direct children inside `RQ_ONE` produced:

```text
RQ_ONE parent total_qpc      24,363,328
HEAD_DISPATCH total_qpc      24,324,287
VIEW_UPDATE calls                     0
CMD_ALLOC calls                       0
INNER_DISPATCH calls                  0
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

Across normal windows:

```text
corr(RQ_ONE parent, HEAD_DISPATCH) ~= 0.99999977
```

**FACT:** the accepted sampled `RQ_ONE` path is effectively entirely `HEAD_DISPATCH = 1.60.1.7s:0x14154CF60`.

## Static counterpart map

```text
1.60.1.7s:0x1413BB140  <->  1.58.1.4s:0x14120D6B0
1.60.1.7s:0x14154C9F0  <->  1.58.1.4s:0x1413D7170
1.60.1.7s:0x14154CF60  <->  1.58.1.4s:0x1413D7700
1.60.1.7s:0x1402D8D20  <->  1.58.1.4s:0x1401EC530
```

The current `HEAD_DISPATCH` counterpart is the same size in both builds (`491` bytes) and has very similar high-level control flow.

## Current discriminator

`HEAD_DISPATCH` contains two normal direct calls to the same downstream function:

```text
NOSPLIT  1.60.1.7s:0x14154CFA7 -> 0x1402D8D20
RANGE    1.60.1.7s:0x14154D048 -> 0x1402D8D20
```

The active experiment separates these two callsites and the residual body.

Decision:

```text
one path dominates and slows per call
  -> recurse into that path/shared target

mix shifts while per-call cost is stable
  -> composition effect

both calls slow similarly
  -> recurse into shared 0x1402D8D20

residual grows instead
  -> split HEAD_DISPATCH body
```

No behavior patch is justified yet.

## Secondary measured branches

Two branches are real but intentionally parked:

- `RQ_PREP 0x14154C370` — secondary contributor inside the current winner;
- `PRE_RENDER_OTHER` — roughly +1.5 ms in the accepted broad frame split.

They should not be opened in parallel while the current leaf continues to produce clean localization.
