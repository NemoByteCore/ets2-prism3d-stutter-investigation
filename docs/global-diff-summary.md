# Whole-corpus 1.58 → 1.60 diff summary

Updated: **2026-09-16**

This document summarizes the completed hypothesis-agnostic static comparison between ETS2 `1.58.1.4s` and `1.60.1.7s`.

The earlier descriptor/root-binding work remains valid, but it was a deep comparison of one selected rendering branch. The global pass was performed specifically to avoid treating that local delta as proof of the complete regression.

## Corpus coverage

```text
1.58 function inventory: 66,820
1.60 function inventory: 68,834
confirmed counterpart pairs: 58,589
coverage of 1.58: 87.68%
coverage of 1.60: 85.12%
materially changed confirmed pairs: 9,308
strong anchored 1.60-only functions: 597
strong anchored 1.58-only functions: 375
ambiguous unmatched regions: 3,370
```

Remaining unmatched functions are deliberately **not** all classified as new/removed. They include reordered code, helper extraction/inlining, template duplication and regions where the available corpus does not support a unique mapping.

## Matching methodology

The first structural alignment used function order, size, call count and external-API identities to reduce the search space. During validation, a concrete false pair showed that these structural anchors cannot be treated as semantic proof.

Final counterpart recovery therefore requires independent evidence from combinations of:

- relocation-insensitive normalized pseudocode
- distinctive strings/types
- validated caller/callee continuity
- local uniqueness / margin over alternative candidates
- global callgraph recovery for moved functions

Large deltas are not promoted from positional alignment alone.

## Main global finding: broad `p3mem` migration

The 1.60 corpus introduces a broad memory/scope architecture absent from the mapped 1.58 corpus.

Observed static differences include:

```text
direct _malloc_base calls:
1.58: 4,212
1.60:   260

1.60 p3_alloc path:
0x140117240
~1,485 static incoming edges

1.60 p3mem lifetime/free helper:
0x140117400
~3,828 static incoming edges
```

Among a conservative set of mapped callers, `390 / 392` functions using the 1.60 `p3_alloc` path have 1.58 counterparts using `_malloc_base`.

Decompiler-visible atomic/refcount signatures also increase substantially in 1.60. This is strong evidence of a cross-cutting allocator/scope/ownership migration, but static prevalence alone does not establish material frametime cost.

## Strongest static regression-shaped candidate found by the pass

Very high-confidence counterpart:

```text
1.58.1.4s:0x1413D5830   516 B
1.60.1.7s:0x14154AAB0  1370 B
```

Both functions perform the same broad `render_queue_set_t` copy/append role, but the 1.60 implementation contains extensive ownership/refcount machinery absent from the 1.58 counterpart.

Decompiler-visible sites:

```text
1.58 helper: LOCK 0 / UNLOCK 0
1.60 helper: LOCK 32 / UNLOCK 16
```

These are syntactic code sites, **not** executed-per-call counts.

The helper is reached from a strongly conserved render-frame construction function:

```text
1.58.1.4s:0x141213E40  7503 B
1.60.1.7s:0x1413C1AE0  7503 B
normalized similarity ≈ 0.980
outgoing calls: 56 -> 56
```

That caller contains the same broad frame-render flow and invokes the copy helper in a preserved queue-set path.

### Runtime follow-up

The static mapping remains high confidence, but the direct-cost hypothesis did **not** survive runtime frequency measurement.

`NemoRenderQueueProbe v0.2` found only:

```text
14 direct calls across 24,798 rendered frames
```

**Conclusion:** this remains a valid and interesting 1.58→1.60 code delta, but direct execution of the helper cannot explain a sustained `+3–8 ms/frame` regression.

This is an important methodological result: a very strong static regression shape can still be irrelevant to the sustained runtime budget if its active frequency is too low.

## Other global-diff candidates and runtime follow-up

### Active-runtime `p3mem` example

Mapped `traffic_trajectory_t::update_neighbors_bits` path:

```text
1.58.1.4s:0x140815880  500 B
1.60.1.7s:0x1408DC510  774 B
```

The 1.60 version adds scope-backed temporary storage/refcount handling.

Runtime follow-up found only 85 total calls in the measured run, including a 30-call shutdown/unload-adjacent burst. A natural heavy-state onset occurred across an approximately 42 s interval with no calls to the target.

**Conclusion:** the path proves that the p3mem migration reaches active gameplay code, but this particular function is strongly demoted as a sustained direct-cost root cause.

### Existing shader-profile / descriptor / root-binding architecture

This branch remains real and runtime-relevant. Runtime work confirmed large fixed-capacity reservation/copy pressure, and sampler reuse safely removes roughly 95% of targeted sampler allocation/copy work.

However, the sustained heavy-scene slowdown can still occur with that optimization active. The descriptor branch is therefore a validated component, not a complete explanation.

### `r_proto` lazy render-queue mask resolution

Strong mapped pair:

```text
1.58.1.4s:0x141213A20   869 B
1.60.1.7s:0x1413C1470  1463 B
```

1.60 adds lazy resolution when the cached mask is `0xFFFFFFFF`, writes the resolved value back, then continues queue filtering.

The proposed exact 1.60 instrumentation boundary has no direct `E8 rel32` callsites and no incoming direct-call edge in the harvested graph. This does not disprove indirect/tail/inlined use, but the original direct-call experiment is closed without new reachability evidence.

### DX12 resource allocator / TLSF / defragmentation

1.60 contains additional DX12 resource-pool/TLSF/defragmentation implementation. The architecture change is real, but steady-state activation/frequency is not established, so it remains lower priority.

## Demoted leads

The global pass also prevented several large static deltas from being over-ranked. Examples inspected and demoted include:

- DirectStorage introduction — backend exists in both builds
- new SRW-lock candidate — traced to `-map_dump` / I/O-cache functionality
- TAA/rendergraph growth — mainly history-image acquire/init path
- several large KDOP/vegetation changes — editor/load/build paths
- traffic-semaphore growth — animated collision-shape initialization
- several model/unit/UI candidates — setup or configuration paths

See [`disproven-hypotheses.md`](disproven-hypotheses.md) for closed/demoted directions.

## What changed after the runtime negatives

The whole-corpus diff is **not discarded**. Its role changed.

Old workflow:

```text
static ranking -> choose next isolated candidate -> runtime probe
```

Current workflow:

```text
runtime phase localization
  -> identify the phase that owns the missing milliseconds
  -> differential stack/counter work inside that phase
  -> use the 58,589-pair counterpart map to compare the measured hotspot
  -> patch only after the runtime budget is quantified
```

This is a better use of the completed static map than continuing to select functions by size/novelty alone.

## Current runtime localization result

`NemoFramePhaseProbe v0.1` measures a main-loop chain around:

```text
0x1401C77C0  LOOP
0x1401C6CB0  PACE
0x1401D72F0  RENDER
```

Two separate natural good→heavy transitions showed:

```text
transition A: LOOP +4.116 ms, RENDER +1.641 ms, OTHER +2.475 ms
transition B: LOOP +3.745 ms, RENDER +2.007 ms, OTHER +1.738 ms
```

`PACE` stayed around `~0.001 ms/iteration`.

The broad `RENDER` bucket contains a nested deliberate frame-time wait helper at `1.60.1.7s:0x14011F730`, so the next probe separates `WAIT` from active render work.

See [`runtime-phase-localization.md`](runtime-phase-localization.md).

## Current question

The investigation is no longer asking:

> Which static delta looks most suspicious?

It is asking:

> Which coarse runtime phase actually gains the missing milliseconds in the heavy state, and what exact 1.58→1.60 code/data change inside that measured phase explains the delta?

Do not return to broad per-D3D-call profiling or generic p3mem hooks without new evidence.