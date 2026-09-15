# 1.58 → 1.60 regression timeline

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

Frame-pacing work showed the sustained slowdown is not primarily a DXGI/fence-wait problem; CPU-side work reaches submission/present too late.

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

The global pass identified a broad 1.60 `p3mem` allocator/scope migration and a repeated `render_queue_set_t` copy path in normal frame construction as the strongest new steady-state candidate.

See [`global-diff-summary.md`](global-diff-summary.md).

### Stage 5 — current runtime discriminator

Current first target:

```text
1.60.1.7s:0x14154AAB0
```

The first experiment is intentionally narrow:

- calls/window
- queue-set count if safe/read-only
- cheap ownership/refcount-heavy branch count if identifiable
- correlation with naturally occurring good/heavy frametime windows

Timing and patching follow only if the count-stage result is positive.

## Current question

The investigation is no longer asking only:

> What changed in the descriptor path?

The current question is:

> Which of the confirmed 1.60 CPU-side changes actually accounts for a meaningful share of the missing ~3–8 ms/frame, and which state/world-lifetime changes explain why the slowdown can reset without restarting the process?