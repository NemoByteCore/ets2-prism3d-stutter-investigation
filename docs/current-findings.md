# Current findings

Updated: **2026-09-16**

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
- severity strongly depends on scene composition
- unload/ferry/teleport transitions can sometimes restore ~16.67 ms without restarting the game

The camera-switch observation remains subjective/unresolved and is not used as the primary theory.

## Confirmed runtime constraints

Earlier frame-pacing probes showed that the previously measured DXGI/fence wait gates are too small to explain the sustained slowdown by themselves.

A broad structural probe also showed that slow active-gameplay regions mainly contain **more work**, approximately:

```text
draws                  +37.6%
resource copy activity +50.2%
sampler copy activity  +46.3%
root CBV calls          +37.3%
root table calls        +38.2%
```

Per-draw rates remained comparatively flat. This favors costs that scale with active scene work rather than one isolated fixed stall.

## Whole-corpus 1.58 ↔ 1.60 diff is complete

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

See [`global-diff-summary.md`](global-diff-summary.md).

## Runtime follow-up demoted the strongest isolated static candidates

### `render_queue_set_t` copy helper

Static pair:

```text
1.58.1.4s:0x1413D5830   516 B
1.60.1.7s:0x14154AAB0  1370 B
```

The 1.60 helper contains substantial new p3mem-style ownership/refcount machinery and was initially the strongest steady-state static candidate.

`NemoRenderQueueProbe v0.2` found only **14 direct calls across 24,798 rendered frames**:

```text
0x013C2A22 = 12
0x013C3289 = 1
0x013C329D = 1
0x0145CF47 = 0
0x0154E4C4 = 0
```

**FACT:** the static delta is real.

**FACT:** direct runtime activity is far too sparse to explain a sustained multi-millisecond-per-frame regression.

**Decision:** strongly demote the direct-cost theory. Do not rescue it without new evidence.

### `r_proto` lazy queue-mask resolution boundary

Mapped pair:

```text
1.58.1.4s:0x141213A20   869 B
1.60.1.7s:0x1413C1470  1463 B
```

A whole-executable direct-call scan found no `E8 rel32` calls to the proposed 1.60 target. The harvested call graph also contains no incoming direct-call edge for that boundary.

**Decision:** the proposed direct-call experiment is closed. Indirect/tail/inlined use is not ruled out, but a new runtime attempt requires new reachability evidence first.

### `traffic_trajectory_t::update_neighbors_bits`

Mapped pair:

```text
1.58.1.4s:0x140815880  500 B
1.60.1.7s:0x1408DC510  774 B
```

Four direct callsites were validated. Runtime totals:

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

There was an approximately **42.17 s call-free interval centered on the heavy-state onset**.

**Decision:** direct execution cost at this target cannot plausibly own the sustained frame budget. The later increase in call rate is more likely a symptom of a busier scene than the direct cause.

## Current pivot: coarse main-loop phase localization

The active runtime chain is now:

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  main-loop iteration
          -> 0x1401C6CB0  frame-clock / duration bookkeeping
          -> 0x1401D72F0  rendergraph / present coordinator
```

`NemoFramePhaseProbe v0.1` measures:

```text
LOOP   = duration of 0x1401C77C0
PACE   = nested duration of 0x1401C6CB0
RENDER = nested duration of 0x1401D72F0
OTHER  = LOOP - PACE - RENDER
```

The probe completed with sane counts:

```text
LOOP calls   = 29,590
PACE calls   = 29,590
RENDER calls = 29,588
bad_end      = 0
thread mismatch = 0
```

Two independent natural good→heavy transitions showed similar growth:

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

**FACT:** a real `~1.7–2.5 ms` portion of the heavy-state increase appears in `OTHER`, outside the broad rendergraph/present bucket and outside frame-clock bookkeeping.

**FACT:** this is a phase-level localization result, not yet a root-cause function.

## Broad RENDER bucket includes deliberate waiting

Static inspection of `0x1401D72F0` identified a nested frame-time wait helper at:

```text
1.60.1.7s:0x14011F730
```

The helper uses `Sleep()` followed by a short spin phase to reach the target frame time.

Therefore the v0.1 `RENDER` value mixes:

- active render/present-side CPU work
- deliberate frame-time wait

The observed `+1.6–2.0 ms` RENDER delta cannot yet be attributed directly to renderer work.

## Current experiment: separate WAIT from active render work

`NemoFramePhaseProbe v0.2` adds the nested wait boundary and derives:

```text
RENDER_ACTIVE = RENDER - WAIT
OTHER         = LOOP - PACE - RENDER
CPU_ACTIVE    = OTHER + RENDER_ACTIVE + PACE
```

Decision logic:

- `WAIT` falls while `RENDER_ACTIVE` stays roughly flat -> extra CPU work is mainly elsewhere and consumes time previously available for pacing wait;
- `RENDER_ACTIVE` also rises materially -> the regression budget is split between active render work and `OTHER`;
- coarse buckets remain distributed/noisy -> use differential CPU stack sampling good vs heavy.

See [`runtime-phase-localization.md`](runtime-phase-localization.md).

## Descriptor / root-binding architecture remains a confirmed component

The mapped 1.58 DX12 path derives root signatures and descriptor capacities from actual pipeline layout. The mapped 1.60 path selects one of 13 fixed shader/root-signature profiles with fixed capacities, separate root CBVs and split per-set tables.

Runtime probing confirmed large fixed-capacity over-reservation. `NemoDX12SamplerAllocReuse` safely removed roughly **95%** of targeted sampler allocation/copy pressure with clean safety counters.

The broader heavy-scene slowdown still occurs with that optimization active.

**Conclusion:** descriptor/sampler work is a validated optimization component, not a complete explanation.

See [`shader-profile-architecture-delta.md`](shader-profile-architecture-delta.md).

## Broad 1.60 `p3mem` migration remains a structural fact

The global corpus shows:

```text
direct _malloc_base calls:
1.58: 4,212
1.60:   260

1.60 p3_alloc path:       0x140117240  (~1,485 static incoming edges)
1.60 lifetime/free path:  0x140117400  (~3,828 static incoming edges)
```

Among a conservative mapped caller set, `390 / 392` 1.60 `p3_alloc` callers have 1.58 counterparts using `_malloc_base`.

This remains important architectural context, but the recent negatives show why **static prevalence alone is not enough**. Generic p3mem hooks/patches are not justified.

## Important negative / demoted leads

Do not promote these again without new evidence:

- DirectStorage introduction — backend exists in both builds
- six-shader tuple/profile collision — `0` conflicts in the measured audit
- new SRW-lock candidate — traced to `-map_dump` / I/O-cache functionality
- TAA/rendergraph growth — mainly history-image acquire/init path
- several large KDOP/vegetation changes — editor/load/build paths
- traffic-semaphore growth — animated collision-shape initialization
- several model/unit/UI candidates — setup/configuration rather than steady-state gameplay
- direct-cost theory for `0x14154AAB0`
- repeat direct-call probing of `0x1413C1470` without new xrefs
- direct-cost theory for `0x1408DC510`

## Current technical direction

The project is no longer using:

```text
static ranking -> next attractive function -> runtime probe
```

Current workflow:

```text
coarse runtime phase localization
  -> good-vs-heavy differential stack/counter work inside the winning phase
  -> map measured hotspot to the completed 1.58 ↔ 1.60 counterpart set
  -> patch only after the missing-ms budget is measured
```

## Telemetry caveat

`SCS frame_start` is not guaranteed to be 1:1 with physically rendered frames. Prefer actual rendered-frametime timing, wall time, synchronized counters and narrow aggregate duration measurements.