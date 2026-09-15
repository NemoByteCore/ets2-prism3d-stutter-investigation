# Experiments

Updated: **2026-09-16**

## Confirmed useful optimization

### NemoDX12SamplerAllocReuse v0.2 / v0.3

**Idea:** reuse same-frame sampler tables before materializing another full fixed-profile sampler allocation.

**v0.2 observed:**

```text
sampler_alloc_provisional = 125,772,004
unique_tables             =   5,089,396
duplicate_tables          = 108,312,238
zero_copy_tables          =  12,370,370
requested_slots_original  = 2,251,985,241
allocated_slots_real      =    96,277,760
avoided_slots             = 2,156,696,505
sampler_copy_calls_seen   =   247,221,488
copy_calls_skipped        =   235,147,658
```

Derived:

- ~86.12% duplicate tables
- ~9.84% zero-copy tables
- ~4.05% real nonzero unique tables
- ~95.77% requested sampler slots avoided
- ~95.12% sampler copy calls skipped

All safety/error counters were zero.

A later v0.3 run reproduced approximately:

- ~95.18% requested sampler slots avoided
- ~94.49% sampler copy calls skipped
- all safety/error counters zero

**Interpretation:** fixed-profile sampler reservation/copy pressure is real and safely reducible.

**Important limit:** the broader sustained heavy-scene slowdown can still occur after this pressure is strongly reduced. This is therefore a validated optimization component, not the complete root cause or complete fix.

## Broad structural profiler

### NemoShaderProfileProbe v0.2

This broad probe was useful structurally but too intrusive for unbiased absolute frametime attribution because it wrapped tens of thousands of direct D3D12 calls per active frame.

Useful structural observations:

- reserved resource capacity / actual resource-layout demand: ~7.62x median
- reserved sampler capacity / actual sampler demand: ~8.34x median
- `material` profile: ~87% of active-gameplay draws in the capture
- typical sampler reservation: ~18.0 slots/draw vs ~2.12 actual sampler bindings/draw
- shader tuple/profile audit: 524 requests, 377 unique tuples, 0 conflicts

**Status:** retain the structural evidence; do not repeat this design as a broad performance profiler.

## Runtime discrimination after the whole-corpus diff

### NemoRenderQueueProbe v0.2

Target:

```text
1.60.1.7s:0x14154AAB0
```

This was the strongest new steady-state static candidate after the whole-corpus diff.

Observed direct-call totals:

```text
0x013C2A22 = 12
0x013C3289 = 1
0x013C329D = 1
0x0145CF47 = 0
0x0154E4C4 = 0

total = 14 calls
```

The run contained `24,798` rendered frames.

**Interpretation:** the direct execution frequency is far too low to explain a sustained multi-millisecond-per-frame regression.

**Status:** direct-cost theory strongly demoted. Do not add deeper timing without new evidence.

### NemoRProtoProbe v0.1

Proposed target:

```text
1.60.1.7s:0x1413C1470
```

The whole executable contained no direct `E8 rel32` calls to the target, so the probe intentionally refused to install instrumentation.

Follow-up harvested callgraph inspection also found no incoming direct-call edge to the same boundary.

**Interpretation:** this is a structural reachability negative for the proposed hook boundary, not a runtime-frequency measurement of all related `r_proto` behavior.

**Status:** do not repeat direct-call probing without new xref/indirect-reachability evidence.

### NemoTrafficNeighborProbe v0.1

Target:

```text
traffic_trajectory_t::update_neighbors_bits
1.60.1.7s:0x1408DC510
```

Four direct callsites were validated and instrumented.

Observed:

```text
total calls = 85
ordinary gameplay calls ≈ 55
shutdown/unload-adjacent burst = 30
```

The same run captured a clean natural transition:

```text
baseline weighted mean ≈ 16.775 ms/frame
heavy weighted mean    ≈ 22.268 ms/frame
sustained delta        ≈ +5.49 ms/frame
```

There was an approximately `42.17 s` call-free interval centered on the heavy-state onset.

**Interpretation:** direct execution cost at this target cannot plausibly own the sustained frame budget. The later higher call rate is more likely correlated scene activity than direct causality.

**Status:** strongly demoted as a sustained direct-cost theory.

## Current runtime-first localization

### NemoFramePhaseProbe v0.1

Recovered main-loop chain:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
```

Measured buckets:

```text
LOOP   = duration of 0x1401C77C0
PACE   = nested duration of 0x1401C6CB0
RENDER = nested duration of 0x1401D72F0
OTHER  = LOOP - PACE - RENDER
```

Final call counts were sane:

```text
LOOP calls   = 29,590
PACE calls   = 29,590
RENDER calls = 29,588
bad_end      = 0
thread mismatch = 0
```

Two natural good→heavy transitions produced similar deltas:

```text
transition A:
LOOP   +4.116 ms
RENDER +1.641 ms
OTHER  +2.475 ms

transition B:
LOOP   +3.745 ms
RENDER +2.007 ms
OTHER  +1.738 ms
```

`PACE` remained around `~0.001 ms/iteration`.

**Interpretation:** `PACE` is not the owner. A repeatable `~1.7–2.5 ms` part of the heavy-state increase lands in `OTHER`, outside the broad rendergraph/present bucket.

**Important caveat:** `RENDER` includes a nested deliberate frame-time wait helper, so its `+1.6–2.0 ms` change is not yet equivalent to extra active renderer work.

### NemoFramePhaseProbe v0.2 — current experiment

Nested wait helper identified at:

```text
1.60.1.7s:0x14011F730
```

The helper uses `Sleep()` followed by a short spin phase.

v0.2 measures that wait separately and derives:

```text
RENDER_ACTIVE = RENDER - WAIT
OTHER         = LOOP - PACE - RENDER
CPU_ACTIVE    = OTHER + RENDER_ACTIVE + PACE
```

Decision rule:

- `WAIT` decreases while `RENDER_ACTIVE` stays roughly flat -> extra CPU work is primarily elsewhere and consumes pacing headroom;
- `RENDER_ACTIVE` also rises materially -> missing budget is split between active render-side work and `OTHER`;
- neither split explains enough -> differential CPU stack sampling good vs heavy.

See [`runtime-phase-localization.md`](runtime-phase-localization.md).

## Descriptor/resource experiments

### Resource Descriptor Reuse v0.5

Observed:

- ~47.2M hits
- ~9.95M misses
- ~112.2M writes skipped
- frametime degraded to ~28–30 ms

**Interpretation:** redundancy exists, but intercepting/buffering enormous numbers of individual writes made this implementation more expensive than the work removed.

**Status:** implementation rejected; underlying redundancy remains relevant.

### Full descriptor-builder memo v0.2

```text
memo_hits   = 0
memo_misses = 35,170,169
```

**Interpretation:** complete draw/bundle state is too dynamic for this form of memoization.

**Status:** rejected.

### Resource table reuse v0.3

The decoder skipped the correct path because validation of `set_count > root_set_count` was too aggressive.

**Status:** invalid as a broad negative result. Do not cite it as proof that useful resource-table redundancy is absent.

## Completed whole-corpus static phase

A hypothesis-agnostic `1.58.1.4s ↔ 1.60.1.7s` corpus diff is complete.

```text
confirmed counterpart pairs: 58,589
materially changed confirmed pairs: 9,308
strong anchored 1.60-only: 597
strong anchored 1.58-only: 375
ambiguous unmatched regions: 3,370
```

Static ranking remains useful for mapping measured runtime hotspots, but recent experiments show why it is no longer used to choose the next isolated target by itself.

## Experiment policy

- one significant variable at a time
- ordinary gameplay in one session; no requirement for hand-picked benchmark saves
- correlate against actual rendered frametime, not telemetry callback count
- prefer counters before timing, timing before patching
- if a candidate measures negligible frequency/cost, demote it immediately
- avoid broad per-D3D-call tracing
- after repeated isolated negatives, localize the owning runtime phase before choosing another leaf function