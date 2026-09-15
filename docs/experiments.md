# Experiments

Updated: **2026-09-15**

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

### NemoDX12SamplerReuse v0.4 — earlier copy-only stage

Copy-only canonicalization skipped roughly 95–96% of targeted sampler descriptor copies while retaining Prism's original allocation behavior.

**Interpretation:** redundancy was real, but allocation cursor/heap pressure remained. The allocator-side reuse experiments addressed that missing half.

## Runtime structural probe

### NemoShaderProfileProbe v0.2

This broad probe was useful structurally but too intrusive for unbiased absolute frametime attribution because it wrapped tens of thousands of direct D3D12 calls per active frame.

Useful structural observations:

- reserved resource capacity / actual resource-layout demand: ~7.62x median
- reserved sampler capacity / actual sampler demand: ~8.34x median
- `material` profile: ~87% of active-gameplay draws in the capture
- typical sampler reservation: ~18.0 slots/draw vs ~2.12 actual sampler bindings/draw
- shader tuple/profile audit: 524 requests, 377 unique tuples, 0 conflicts

**Status:** retain the structural evidence; do not repeat this design as a broad performance profiler.

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

A hypothesis-agnostic `1.58.1.4s ↔ 1.60.1.7s` corpus diff is now complete.

```text
confirmed counterpart pairs: 58,589
materially changed confirmed pairs: 9,308
strong anchored 1.60-only: 597
strong anchored 1.58-only: 375
ambiguous unmatched regions: 3,370
```

The strongest new steady-state candidate is the mapped `render_queue_set_t` copy path:

```text
1.58.1.4s:0x1413D5830   516 B
1.60.1.7s:0x14154AAB0  1370 B
```

The 1.60 helper adds substantial p3mem-style ownership/refcount machinery and is called repeatedly from a strongly conserved render-frame construction path.

See [`global-diff-summary.md`](global-diff-summary.md).

## Next experiment: narrow `render_queue_set` discriminator

### Phase 1 — count only

Target:

```text
1.60.1.7s:0x14154AAB0
```

Collect synchronized with existing rendered-frametime windows:

- helper calls/window
- queue-set count from the preserved caller if safe/read-only
- ownership/refcount-heavy branch entries if identifiable cheaply
- natural unload/ferry/teleport transitions as context
- existing passive camera markers only as context

Do **not** time/log every call yet.

Decision rule:

- strong correlation with ~16.7 ms vs sustained ~20–25 ms windows -> advance to timing
- essentially flat call/branch behavior -> demote this candidate and continue down the global ranking

### Phase 2 — low-overhead timing only after positive Phase 1

Prefer aggregate timing around the queue-copy section in the preserved caller or sampled helper timing over per-call logging.

Practical interpretation:

```text
heavy-minus-good >= ~1 ms/frame  -> strong primary-component evidence
~0.3–1 ms/frame                  -> real component, unlikely complete cause
< ~0.2–0.3 ms/frame              -> demote unless another strong correlation exists
```

Also check whether measured work drops across a naturally observed unload/ferry/teleport recovery.

### Phase 3 — optimization only after measured cost

If the path is material, inspect ownership semantics specifically in this frame path. Possible optimization classes include:

- remove a provably unnecessary copy
- reuse an existing queue-set
- move/borrow where frame lifetime guarantees safety
- avoid retain/release only where ownership equivalence is proven

Do **not** disable global p3mem refcounting.

### Phase 4 — state/reset axis

If the queue-copy path does not explain the reset behavior, next inspect the 1.60 `r_proto` lazy mask-resolution path:

```text
1.60.1.7s:0x1413C1470
```

Use counters for lazy-resolution hits/cache fills before considering timing.

### Phase 5 — broader p3mem only by active callsite

Do not hook generic `p3_alloc` globally.

Probe one confirmed active path at a time; current secondary candidate:

```text
traffic_trajectory_t::update_neighbors_bits
1.60.1.7s:0x1408DC510
```

The end goal is a measured budget rather than another single-theory guess:

```text
descriptor/root component       X ms
render_queue ownership          Y ms
traffic/other p3mem component   Z ms
remaining unexplained           R ms
```

## Experiment policy

- one significant variable at a time
- ordinary gameplay in one session; no requirement for hand-picked benchmark saves
- correlate against actual rendered frametime, not telemetry callback count
- prefer counters before timing, timing before patching
- if a candidate measures negligible cost, demote it immediately
- avoid broad per-D3D-call tracing