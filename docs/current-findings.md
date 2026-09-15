# Current findings

Updated: **2026-09-15**

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

## Primary runtime symptoms

On the tested setup:

- light scenes can hold roughly `16.67 ms / 60 FPS`
- heavier scene compositions commonly reach roughly `19–25 ms`
- the slower state can be sustained rather than a single hitch
- background motion can feel like `start -> stop -> start -> stop`
- severity strongly depends on scene composition
- unload/ferry/teleport transitions can sometimes restore ~16.67 ms without restarting the game

The camera-switch observation remains subjective/unresolved and is not used as the primary theory.

## Confirmed runtime constraints

### Sustained slowdown is not primarily a wait/fence problem

`NemoFramePacingProbe v0.3` showed that the tested DXGI frame-latency gate and fence waits are too small to explain the sustained slowdown.

**FACT:** CPU-side render construction reaches submission/present too late.

### Slow regions mainly contain more work

A broad structural probe was too intrusive for absolute timing, but within that run slow active-gameplay regions showed approximately:

```text
draws                 +37.6%
resource copy activity +50.2%
sampler copy activity  +46.3%
root CBV calls          +37.3%
root table calls        +38.2%
```

Per-draw rates remained comparatively flat. This favors repeated per-item/per-draw/per-queue costs that scale with scene complexity rather than one isolated stall.

### Simple instancing-volume metrics do not explain the slowdown

Raw instancing bytes/chunks/clusters were dynamic but did not track frametime strongly enough to explain the problem by themselves.

## Whole-corpus 1.58 ↔ 1.60 diff is complete

The investigation no longer relies on a descriptor-only static comparison.

```text
1.58 function inventory: 66,820
1.60 function inventory: 68,834
confirmed counterpart pairs: 58,589
coverage of 1.58: 87.68%
coverage of 1.60: 85.12%
materially changed confirmed pairs: 9,308
strong anchored 1.60-only: 597
strong anchored 1.58-only: 375
ambiguous unmatched regions: 3,370
```

Unmatched functions are deliberately not all classified as new/removed. See [`global-diff-summary.md`](global-diff-summary.md).

## Strongest new steady-state static candidate

Very high-confidence `render_queue_set_t` copy/append counterpart:

```text
1.58.1.4s:0x1413D5830   516 B
1.60.1.7s:0x14154AAB0  1370 B
```

Both perform the same broad role. The 1.60 implementation contains substantial p3mem-style ownership/refcount machinery absent from 1.58.

Decompiler-visible sites:

```text
1.58: LOCK 0 / UNLOCK 0
1.60: LOCK 32 / UNLOCK 16
```

These are syntactic code sites, not executed-per-call counts.

The helper is reached from a strongly conserved render-frame construction function:

```text
1.58.1.4s:0x141213E40  7503 B
1.60.1.7s:0x1413C1AE0  7503 B
normalized similarity ≈ 0.980
outgoing calls: 56 -> 56
```

The caller invokes the helper in a queue-set loop plus two additional calls outside the loop.

**FACT:** repeated frame logic is preserved while the 1.60 helper is materially heavier and ownership-aware.

**HYPOTHESIS:** this added per-copy work may contribute measurable CPU cost in complex scenes.

Runtime call rate, executed atomic/refcount path rate and aggregate cost remain unmeasured. This is not yet a confirmed root cause.

## Broad 1.60 `p3mem` allocator/scope migration

The global corpus shows a cross-cutting architecture change:

```text
direct _malloc_base calls:
1.58: 4,212
1.60:   260

1.60 p3_alloc path:       0x140117240  (~1,485 static incoming edges)
1.60 lifetime/free path:  0x140117400  (~3,828 static incoming edges)
```

Among a conservative mapped caller set, `390 / 392` functions using 1.60 `p3_alloc` have 1.58 counterparts using `_malloc_base`.

Decompiler-visible atomic/refcount signatures increase substantially in 1.60. The migration reaches both render construction and active traffic code.

One active traffic example:

```text
traffic_trajectory_t::update_neighbors_bits
1.58.1.4s:0x140815880  500 B
1.60.1.7s:0x1408DC510  774 B
```

The 1.60 path adds scope-backed temporary storage/refcount cleanup.

**Important:** static prevalence does not prove significant frametime cost. Do not patch global p3mem without runtime evidence.

## Descriptor / root-binding architecture remains a confirmed component

The mapped 1.58 DX12 path derives root signatures and descriptor capacities from the actual pipeline layout. The mapped 1.60 path selects one of 13 fixed shader/root-signature profiles with fixed capacities, separate root CBVs and split per-set tables.

High-confidence chain:

```text
RFX shader profile
  -> resource bucketization / cross-stage merge
  -> 1.60.1.7s:0x14144C770
  -> 1.60.1.7s:0x1402D7D70
  -> 1.60.1.7s:0x1402942D0
  -> descriptor update/build
  -> 1.60.1.7s:0x14029E1F0
  -> generalized root/table submission
```

Mapped counterparts include:

```text
1.60:0x1402D7D70 <-> 1.58:0x1401EB530
1.60:0x1402E25A0 <-> 1.58:0x1401F4FB0
1.60:0x14144C770 <-> 1.58:0x14129BF20
1.60:0x1402942D0 <-> 1.58:0x1401AC780
1.60:0x14029E1F0 <-> 1.58:0x1401B4800
```

See [`shader-profile-architecture-delta.md`](shader-profile-architecture-delta.md).

## Fixed profile reservation is confirmed at runtime

A broad structural probe showed median active-gameplay ratios around:

```text
reserved resource capacity / actual layout demand ~7.62x
reserved sampler capacity  / actual sampler demand ~8.34x
```

Typical sampler values were roughly:

```text
reserved ~18.0 slots/draw
actual   ~2.12 sampler bindings/draw
```

The `material` profile accounted for about 87% of active-gameplay draws in that capture.

The probe itself was too intrusive for unbiased absolute timing.

## Sampler allocation/copy pressure is real but not the complete cause

`NemoDX12SamplerAllocReuse v0.2` observed:

```text
requested_slots_original  = 2,251,985,241
allocated_slots_real      =    96,277,760
avoided_slots             = 2,156,696,505
sampler_copy_calls_seen   =   247,221,488
copy_calls_skipped        =   235,147,658
```

Derived:

```text
~95.77% requested sampler slots avoided
~95.12% sampler copy calls skipped
```

A later v0.3 run reproduced roughly `95.18%` avoided slots and `94.49%` skipped copies. Safety/error counters remained zero.

**FACT:** the fixed-profile sampler path creates large, safely reducible pressure.

**FACT:** the broader heavy-scene slowdown can still occur after that pressure is strongly reduced.

Therefore sampler reuse is a validated optimization component, not a complete fix.

## Other ranked global-diff candidates

### `r_proto` lazy render-queue mask resolution

Strong counterpart:

```text
1.58.1.4s:0x141213A20   869 B
1.60.1.7s:0x1413C1470  1463 B
```

1.60 adds lazy resolution when the cached mask is `0xFFFFFFFF`, then caches the result. This makes it more plausible as a streaming/first-use component than a permanent every-frame cost.

### DX12 resource allocator / TLSF / defragmentation

1.60 contains additional DX12 resource-pool/TLSF/defragmentation code, including a `dx12_pool_t::defragment_data(...)` candidate around `1.60.1.7s:0x14028E320`.

Steady-state activation/frequency is not established, so this remains lower priority.

## Important negative / demoted leads

Do not promote these again without new evidence:

- DirectStorage introduction — backend exists in both builds
- six-shader tuple/profile collision — `0` conflicts in the measured runtime audit
- new SRW-lock candidate — traced to `-map_dump` / I/O-cache functionality
- TAA/rendergraph growth — mainly history-image acquire/init path
- several large KDOP/vegetation changes — editor/load/build paths
- traffic-semaphore growth — animated collision-shape initialization
- several model/unit/UI candidates — setup/configuration rather than steady-state gameplay

## Current technical model

The evidence fits a cumulative model better than a one-bug model:

```text
heavy scene / more active work
  -> more repeated 1.60 CPU-side infrastructure work
     - render_queue ownership/refcount
     - descriptor/root-binding overhead
     - possibly other active p3mem paths
  -> CPU render construction reaches submit/present later

world rebuild / unload / ferry
  -> active scene/queue/scope state changes or is rebuilt
  -> repeated work may drop
  -> ~16.67 ms behavior can return without process restart
```

This is a hypothesis framework, not proof of causality.

## Exact next runtime step

Do not return to a broad D3D profiler.

First probe target:

```text
1.60.1.7s:0x14154AAB0
```

Collect low-overhead synchronized counters during ordinary gameplay:

1. helper calls/window
2. queue-set count from the preserved caller if safe/read-only
3. cheap count of ownership/refcount-heavy branch entries if identifiable
4. natural unload/ferry/teleport transitions as context

Only if counts/branch behavior correlate with good (~16.7 ms) versus sustained heavy (~20–25 ms) windows should aggregate or sampled timing be added.

If the measured cost is negligible, demote this branch immediately and continue down the global ranking.

## Telemetry caveat

`SCS frame_start` is not guaranteed to be 1:1 with physically rendered frames. Prefer actual rendered-frame timing, wall time, synchronized counters and narrow exact/aggregate duration measurements.