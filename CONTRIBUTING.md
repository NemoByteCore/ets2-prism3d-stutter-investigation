# Contributing

This repository is an evidence-driven reverse-engineering investigation. Contributions are welcome when they help verify the 1.58 ↔ 1.60 architecture delta, improve runtime instrumentation, reproduce the slowdown, or correct the current function/pseudocode model.

## Best ways to help right now

Current high-value contribution areas:

1. **Runtime instrumentation review**
   - low-overhead measurement of actual rendered frametime
   - profile-aware draw/work counters
   - actual descriptor writes/copies
   - actual `SetGraphicsRoot*` calls
   - root-signature/profile switches
   - descriptor heap rollover/switch events

2. **Shader-tuple/profile invariant audit**
   - determine whether the same six-shader tuple can ever be requested with more than one 1.60 shader-profile ID
   - this is an audit target, **not a confirmed cache bug**

3. **Independent reproduction**
   - reproduce scene-dependent CPU-side slowdown on another system
   - document exact game build, renderer, settings and hardware
   - distinguish measured frametime from subjective impressions

4. **Static review**
   - review the mapped 1.58 ↔ 1.60 descriptor/root-binding path
   - challenge function identifications or architecture interpretations with concrete evidence

## Good contributions

Useful reports usually contain:

- exact ETS2 build
- executable hash when relevant
- function address(es) with the build clearly stated
- how the function was identified
- callers/callees, strings, types or dataflow that support the identification
- whether the claim is a **FACT**, **INFERENCE** or **HYPOTHESIS**
- confidence level
- what the finding changes or what should be tested next

For runtime measurements, also include:

- renderer
- relevant graphics/settings state
- hardware
- how actual rendered frametime was measured
- whether a number is a direct API/engine event or an inferred counter

## Current build policy

- `1.60.1.7s` — runtime profiling, probes and patch experiments
- `1.58.1.4s` — static-only pre-regression reference; it is **not run**

Addresses are build-specific. Never assume an address or function identity carries across versions without re-identification.

## Runtime experiment constraint

The active save does not provide arbitrary control over test scenes. Please do not propose experiments that require hand-picked repeatable light/heavy saves or arbitrary teleporting solely for testing.

Preferred experiments work during one ordinary gameplay session and correlate synchronized counters with actual rendered frametime afterward.

See [`docs/methodology.md`](docs/methodology.md).

## Please avoid

- uploading SCS executables, assets or other proprietary game files
- bulk raw decompiler dumps
- presenting speculation as a confirmed cause
- treating reserved descriptor capacity as proof of actual descriptor writes
- treating predicted root-binding activity as equivalent to directly hooked API calls
- repeating already-closed hypotheses without new evidence
- generic optimization advice unrelated to the measured path

## Pseudocode

Pseudocode in this repository is intentionally normalized and structural. It should explain the behavior relevant to the investigation without reproducing large raw decompiler listings.

When adding or correcting pseudocode, keep these categories separate where applicable:

```text
FACT
INFERENCE
HYPOTHESIS
```

## Opening an issue

Use a concise, evidence-oriented title, for example:

```text
[1.60] Profile-aware timing around descriptor update path
```

```text
[1.58/1.60] Review of mapped root-signature construction delta
```

```text
[Repro] Scene-dependent slowdown on Ryzen / Radeon system
```

Include the minimum evidence needed for another person to reproduce, verify or challenge the result. Corrections are explicitly welcome; working function names in this repository are research labels unless stated otherwise.
