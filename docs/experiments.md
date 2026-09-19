# Experiments

Updated: **2026-09-18**

This file keeps the experiments that still matter to the current interpretation. Probe-by-probe chronology is intentionally omitted; old version numbers are not useful unless they changed the conclusion.

## Current runtime localization

The active runtime strategy is:

```text
broad frame phase
  -> measured owner
    -> sampled child timing
      -> matched-cardinality comparison
        -> recurse only into the measured owner
```

The current render-side chain is:

```text
RG_CORE
  -> T1 helper
    -> pass callback
      -> nested winner
        -> RQ_ONE
          -> HEAD_DISPATCH
            -> 0x1402D8D20
```

The no-split path is now accepted, and stable queue attribution shows one queue class carrying most sampled item work overall. The current narrow question is the native-DX12 descriptor-builder split reached from the accepted downstream path: descriptor-parent cost by stable queue, then resource reservation vs sampler reservation vs remaining descriptor-update work.

See [`rg-core-runtime-localization.md`](rg-core-runtime-localization.md).

## Stable queue attribution

A stable-index queue-work follow-up validated 113,391 queue identities across ordinary windows with 0 invalid derivations and 0 bucket overflow.

The dominant class, `queue_index 0 / tag 11`, carried about 80.7% of processed items and 80.0% of measured queue time overall, with `corr(total_items, queue0_items) ~= 0.973`.

This result is an attribution result, not proof of overdraw or duplicated geometry. Some matched-cardinality scenes move growth into other queues, and heavy scenery can legitimately produce more render items.

## Descriptor-builder discriminator

The accepted downstream path reaches the native-DX12 descriptor builder through `0x1402D8F6C [vtable+0x260]`. Static mapping identifies the implementation as `0x1402942D0`, with 1.58 counterpart `0x1401AC780`.

Inside 1.60 the resource and sampler reservation calls are:

```text
RESOURCE  0x1402943FF -> 0x14028F070
SAMPLER   0x140294438 -> 0x14028F070
```

The next narrow runtime experiment measures this parent and allocator split while retaining stable queue attribution. It does not change rendering behavior.

## Confirmed useful optimization: sampler-table reuse

The mapped 1.60 descriptor/root-binding architecture reserves fixed profile capacity well above actual binding demand.

A safe sampler-table reuse experiment observed roughly:

```text
requested sampler slots avoided    ~95%
sampler copy calls skipped         ~95%
safety/error counters                 0
```

This confirms a real optimization opportunity.

**Limit:** the broader sustained heavy-scene slowdown still occurs after this pressure is strongly reduced.

**Status:** useful optimization component, not the complete root cause.

## Broad structural profiler

A profile-aware D3D12 structural probe found:

```text
reserved resource capacity / actual demand  ~7.62x median
reserved sampler capacity / actual demand   ~8.34x median
material profile share                      ~87% of active draws
typical sampler reservation                 ~18.0 slots/draw
typical actual sampler bindings             ~2.12/draw
shader tuple/profile conflicts              0 in the measured audit
```

The design wrapped too many hot D3D calls to be trusted for absolute performance attribution.

**Status:** retain structural observations; do not repeat this broad hook design for timing.

## Runtime-demoted static candidates

### `render_queue_set_t` copy helper

```text
1.60.1.7s:0x14154AAB0
14 direct executions / 24,798 rendered frames
```

**Status:** direct sustained-cost theory demoted.

### `r_proto` proposed boundary

```text
1.60.1.7s:0x1413C1470
```

No direct `E8 rel32` callsites were found at the proposed boundary.

**Status:** do not repeat the same direct-call probe without new reachability evidence.

### `traffic_trajectory_t::update_neighbors_bits`

```text
1.60.1.7s:0x1408DC510
85 total calls
```

A sustained heavy-state onset occurred across a long interval with no calls to the target.

**Status:** direct sustained-cost theory demoted.

## Descriptor/resource experiments that should not be repeated as implemented

### Individual resource-descriptor reuse

Large redundancy was observed, but intercepting/buffering enormous numbers of individual writes made the experiment substantially slower.

**Conclusion:** the implementation is rejected; the existence of redundancy is not.

### Full descriptor-builder memoization

Observed:

```text
memo_hits   = 0
memo_misses = 35,170,169
```

**Conclusion:** complete draw/bundle state is too dynamic for this memoization shape.

### Resource-table reuse with the old decoder

The decoder rejected the correct path because a validation rule was too aggressive.

**Conclusion:** invalid as a broad negative result. Do not cite it as proof that useful resource-table redundancy is absent.

## Whole-corpus static comparison

The hypothesis-agnostic `1.58.1.4s ↔ 1.60.1.7s` corpus diff remains the static map:

```text
confirmed counterpart pairs:        58,589
materially changed confirmed pairs:  9,308
strong anchored 1.60-only:             597
strong anchored 1.58-only:             375
ambiguous unmatched regions:          3,370
```

Static ranking is no longer used to pick the next isolated target by appearance alone.

## Experiment policy

- one significant variable at a time;
- ordinary gameplay in one session is acceptable;
- correlate against actual rendered frametime, not telemetry callback count;
- prefer counters before timing, timing before patching;
- use sampled/aggregate timing in hot paths;
- matched-cardinality comparisons are preferred when workload count can vary;
- demote candidates immediately when runtime frequency/cost is negligible;
- avoid broad per-D3D-call tracing when a narrow counter can answer the question;
- finish one measured branch before opening secondary branches.
