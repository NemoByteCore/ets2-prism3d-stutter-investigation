# Experiments

Updated: **2026-09-15**

## Confirmed useful optimization

### NemoDX12SamplerAllocReuse v0.2 / v0.3

**Idea:** the 1.60 descriptor builder reserves sampler-table capacity from fixed shader-profile spans even when the final table is identical to one already built in the same frame, or receives no sampler writes at all. Reuse same-frame sampler tables before materializing another full sampler-heap allocation.

The patch keeps full original spans for genuinely unique nonzero tables. It does not shrink root-signature ranges, rewind allocator cursors, reuse tokens across frame generations, or touch the resource/CBV-SRV-UAV heap.

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
- ~95.77% of requested sampler slots avoided
- ~95.12% of sampler copy calls skipped

All safety/error counters were zero.

A later v0.3 run with lightweight camera markers reproduced the same order of magnitude:

- ~95.18% requested sampler slots avoided
- ~94.49% sampler copy calls skipped
- all safety/error counters zero

**Interpretation:** fixed-profile sampler reservation creates a large, real allocator/copy pressure that is safely reducible. This is now a validated optimization component.

**Important limit:** previous runs still reproduced the broader heavy-scene slowdown after sampler pressure was reduced. This is therefore not the complete root cause or complete fix.

**Status:** retain as a strong candidate component of a final patch.

### NemoDX12SamplerReuse v0.4 — earlier copy-only stage

**Idea:** keep Prism's original allocations but canonicalize duplicate same-frame sampler tables and skip redundant `CopyDescriptorsSimple` calls.

**Observed:**

- ~197.1M sampler copy calls
- ~186.7M skipped
- ~94.7% redundant
- later runs up to ~96% skipped
- no overflow/fail-open in the corrected merged version

**User observation:** slight subjective improvement.

**Interpretation:** this proved the redundancy, but because original sampler allocations remained, allocator cursor/heap pressure also remained. The later `SamplerAllocReuse` experiment addresses that missing half.

## Runtime structural probe

### NemoShaderProfileProbe v0.2

This broad probe was useful structurally but too intrusive for absolute frametime attribution because it wrapped tens of thousands of D3D12 calls per active frame.

Useful structural observations:

- median reserved resource capacity / actual resource-layout demand: ~7.62x
- median reserved sampler capacity / actual sampler demand: ~8.34x
- `material` profile: ~87% of active gameplay draws in the captured run
- typical sampler reservation: ~18.0 slots/draw versus ~2.12 actual sampler bindings/draw
- shader-tuple/profile audit: 524 requests, 377 unique tuples, 0 profile conflicts

The 0-conflict result is negative evidence against the proposed six-shader-tuple/profile cache-collision hypothesis, though it does not prove the invariant globally.

**Status:** do not repeat as a broad performance profiler; retain its structural results.

## Descriptor/resource experiments

### Resource Descriptor Reuse v0.5

**Observed:**

- ~47.2M hits
- ~9.95M misses
- ~112.2M writes skipped
- frametime degraded to ~28–30 ms

**Interpretation:** redundancy exists, but intercepting/buffering a huge number of individual writes made the implementation too expensive.

**Status:** implementation rejected; underlying redundancy remains relevant.

### Full descriptor-builder memo v0.2

**Observed:**

- `memo_hits = 0`
- `memo_misses = 35,170,169`

**Interpretation:** complete draw/bundle state is too dynamic for full memoization.

**Status:** rejected.

### Resource table reuse v0.3

The decoder skipped the correct path because validation of `set_count > root_set_count` was too aggressive.

**Interpretation:** this run cannot be used to prove resource-table redundancy is absent.

**Status:** invalid as a broader negative result.

## Current experiment policy

Avoid returning to broad instrumentation when a narrow counter or targeted patch can answer the question. The valid runtime method must also work on the existing save without requiring arbitrary staged light/heavy scenes.

Current high-value direction:

```text
validated sampler allocation/copy optimization
+ continue narrowing the remaining heavy-scene cost
+ correlate only lightweight, synchronized markers/counters with naturally occurring slow states
```
