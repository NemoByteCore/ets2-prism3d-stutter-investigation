# 1.58 → 1.60 regression timeline

Updated: **2026-09-16**

## Evidence categories

Keep these separate:

```text
community reports:
1.58 good → 1.59/1.60 bad

our runtime evidence:
1.60 reproduced and profiled

our static evidence:
1.58 vs 1.60 whole-corpus and targeted code comparison
```

Community reports are context, not our own runtime A/B measurement.

## Public regression context

Public reports collected during the investigation include:

- Reddit reports of constant stuttering after newer builds
- Steam discussions describing similar regressions
- SCS forum reports around 1.59/1.60

SCS also changed buffer/resource allocation behavior in 1.59-era builds and published fixes described as buffer-management/resource-allocator performance work during beta development.

These reports help define the suspected regression window but do not replace build-specific code/runtime evidence.

## Why 1.58 matters

`1.58.1.4s` is not a runtime target and is never launched in this project.

It is used as a static pre-regression snapshot against runtime/fix target `1.60.1.7s`.

## Investigation progression

### Stage 1 — runtime symptom characterization

The sustained slowdown was reproduced in 1.60:

- light scenes around ~16.67 ms
- heavy compositions around ~19–25 ms
- strong scene dependence
- unload/ferry/teleport can sometimes restore good behavior without restart

Frame-pacing work showed the previously measured DXGI/fence wait gates do not explain the sustained slowdown by themselves.

### Stage 2 — targeted descriptor/resource branch comparison

Initial static work focused on:

- `r_device_t::resource_build_bundle`
- semantic/resource resolution
- shader-profile selection
- descriptor reservation/update
- root-signature representation
- root-CBV/table submission

This found a real architecture delta: 1.58 uses layout-specific root-signature/capacity behavior in the mapped path, while 1.60 uses fixed shader/root-signature profiles with fixed capacities and generalized root binding.

### Stage 3 — runtime validation of the descriptor branch

Runtime probing confirmed large fixed-profile over-reservation and sampler pressure.

`NemoDX12SamplerAllocReuse` safely removes roughly 95% of targeted sampler allocation/copy work, but the broader heavy-scene slowdown still occurs.

Conclusion: descriptor/sampler work is a real component, not a complete explanation.

### Stage 4 — hypothesis-agnostic whole-corpus diff

The investigation then restarted the 1.58 ↔ 1.60 comparison globally to avoid tunnel vision.

Final inventory:

```text
confirmed counterpart pairs: 58,589
coverage: 87.68% of 1.58 / 85.12% of 1.60
materially changed confirmed pairs: 9,308
strong anchored 1.60-only: 597
strong anchored 1.58-only: 375
ambiguous unmatched regions: 3,370
```

The global pass identified a broad 1.60 `p3mem` allocator/scope migration and several regression-shaped candidates.

See [`global-diff-summary.md`](global-diff-summary.md).

### Stage 5 — runtime discrimination of top static candidates

The strongest-looking isolated candidates were tested before patching.

#### `render_queue_set_t` copy helper

```text
1.60.1.7s:0x14154AAB0
```

Runtime result:

```text
14 direct calls across 24,798 rendered frames
```

Conclusion: strong static delta, but direct cost far too sparse to explain sustained multi-millisecond frame loss.

#### `r_proto` lazy-resolution boundary

```text
1.60.1.7s:0x1413C1470
```

No direct `E8 rel32` callsites or incoming direct-call edge were found for the proposed hook boundary.

Conclusion: repeat direct-call probing closed pending new reachability evidence.

#### `traffic_trajectory_t::update_neighbors_bits`

```text
1.60.1.7s:0x1408DC510
```

Only 85 calls occurred in the full run, including 30 near shutdown/unload. The run still transitioned naturally from about `16.775 ms/frame` to `22.268 ms/frame`, with an approximately 42 s call-free interval centered on the heavy-state onset.

Conclusion: direct execution cost at this target cannot explain the sustained heavy state.

### Stage 6 — current runtime-first phase localization

After repeated isolated negatives, the project pivoted to a coarse main-loop chain:

```text
0x1401C5280
  -> 0x1401C77C0  LOOP
       -> 0x1401C6CB0  PACE
       -> 0x1401D72F0  RENDER
            -> 0x14011F730  WAIT
```

`NemoFramePhaseProbe v0.1` produced two similar natural good→heavy transitions:

```text
transition A: LOOP +4.116 ms, RENDER +1.641 ms, OTHER +2.475 ms
transition B: LOOP +3.745 ms, RENDER +2.007 ms, OTHER +1.738 ms
```

`PACE` stayed around `~0.001 ms/iteration`.

A nested frame-time wait helper was then identified inside the broad RENDER bucket, so the current v0.2 probe separates `WAIT` from active render work.

See [`runtime-phase-localization.md`](runtime-phase-localization.md).

## Current question

The investigation is no longer asking only:

> What changed between 1.58 and 1.60?

The whole-corpus map already answers that at scale.

The current question is:

> Which measured runtime phase actually gains the missing milliseconds in the heavy state, and what exact 1.58→1.60 code/data change inside that phase explains the measured budget?

Static comparison now follows runtime localization instead of selecting the next leaf function by visual attractiveness alone.