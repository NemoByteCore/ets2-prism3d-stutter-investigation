# Whole-corpus 1.58 → 1.60 diff summary

Updated: **2026-09-15**

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

## Strongest new steady-state candidate: `render_queue_set_t` copy path

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

That caller contains the same broad frame-render flow and invokes the copy helper once per queue-set element in a preserved loop, plus two additional helper calls outside the loop.

**FACT:** the repeated frame path is preserved while the 1.60 copy helper is materially heavier and ownership-aware.

**HYPOTHESIS:** the added per-copy ownership/refcount work may contribute measurable CPU render-construction cost in complex scenes.

Runtime frequency and cost have not yet been measured, so this is **not a confirmed root cause**.

## Other ranked candidates

### Broader active-runtime `p3mem` use

The migration also reaches active traffic code. A mapped `traffic_trajectory_t::update_neighbors_bits` path grows from:

```text
1.58.1.4s:0x140815880  500 B
1.60.1.7s:0x1408DC510  774 B
```

The 1.60 version adds scope-backed temporary storage/refcount handling. Execution frequency and cost remain unmeasured.

### Existing shader-profile / descriptor / root-binding architecture

This branch remains real and runtime-relevant. Runtime work confirmed large fixed-capacity reservation/copy pressure, and sampler reuse safely removes roughly 95% of targeted sampler allocation/copy work.

However, the sustained heavy-scene slowdown can still occur with that optimization active. The descriptor branch is therefore a validated component, not a complete explanation.

### `r_proto` lazy render-queue mask resolution

Strong mapped pair:

```text
1.58.1.4s:0x141213A20   869 B
1.60.1.7s:0x1413C1470  1463 B
```

1.60 adds lazy resolution when the cached mask is `0xFFFFFFFF`, writes the resolved value back, then continues queue filtering. Because the result is cached, this is currently more plausible as a first-use/streaming component than a permanent every-frame cost.

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

## Current symptom-driven model

The evidence currently fits a cumulative model better than a single static bug:

```text
heavy scene / more active work
  -> more repeated 1.60 CPU-side infrastructure work
     - render_queue ownership/refcount
     - descriptor/root-binding overhead
     - possibly other active p3mem paths
  -> render construction reaches submit/present later

world rebuild / unload / ferry
  -> active scene/queue/scope state changes or is rebuilt
  -> repeated work can drop
  -> good ~16.67 ms behavior can return without process restart
```

This is a framework for experiments, not proof of causality.

## Next runtime discriminator

The next probe should be deliberately narrow around:

```text
1.60.1.7s:0x14154AAB0
```

First collect low-overhead counts aligned with existing rendered-frametime windows:

- helper calls/window
- queue-set count from the preserved caller if safe/read-only
- cheap count of ownership/refcount-heavy branch entries if identifiable
- passive transition/camera markers only as context

Only if count/branch behavior correlates with naturally occurring heavy windows should aggregate or sampled timing be added.

Do not return to a broad per-D3D-call profiler.