# Experiments

## Confirmed useful optimization

### NemoDX12SamplerReuse v0.4

**Idea:** Prism copies sampler descriptors into fresh shader-visible tables per draw. Many tables are identical within the same frame. Keep allocations but skip redundant `CopyDescriptorsSimple` calls.

**Observed:**

- ~197.1M sampler copy calls
- ~186.7M skipped
- ~94.7% redundant
- later runs up to ~96% skipped
- no overflow/fail-open in the corrected merged version

**User observation:** slight subjective improvement.

**Status:** retain as a candidate part of a final patch unless later work finds a conflict.

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

## Planned runtime probe after 1.58 ↔ 1.60 static diff

If the static diff does not answer the regression question, instrument only a few exact points:

### `1.60:FUN_1402D7D70`

Measure:

- calls/s
- total wall-time/s
- average duration
- p95 duration

### `1.60:FUN_1402E25A0`

Measure:

- calls/s
- wall time
- repeated semantic IDs within the same bundle

### `1.60:FUN_14144C770`

Measure:

- calls/s
- number of times `cached_key != new_key`

### Uniform phase

Measure:

- uniform callbacks/s
- callbacks/bundle

Prefer exact counters over returning to a broad sampler when exact counters answer the question.
