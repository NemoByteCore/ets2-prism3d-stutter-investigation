# Contributing

This repository is an evidence-driven reverse-engineering investigation. Contributions are welcome when they help verify build-specific mappings, reproduce the slowdown, improve low-overhead instrumentation, or challenge the current runtime model with concrete evidence.

## Best ways to help right now

### 1. Review the new whole-corpus diff

The `1.58.1.4s ↔ 1.60.1.7s` global comparison is summarized in [`docs/global-diff-summary.md`](docs/global-diff-summary.md).

High-value review targets:

- `render_queue_set_t` copy counterpart:
  - `1.58.1.4s:0x1413D5830`
  - `1.60.1.7s:0x14154AAB0`
- preserved frame-render caller:
  - `1.58.1.4s:0x141213E40`
  - `1.60.1.7s:0x1413C1AE0`
- p3mem allocator/scope interpretation
- `r_proto` lazy render-queue mask resolution:
  - `1.58.1.4s:0x141213A20`
  - `1.60.1.7s:0x1413C1470`

Corrections to mappings are explicitly welcome.

### 2. Low-overhead runtime instrumentation

The next runtime question is deliberately narrow: does `1.60.1.7s:0x14154AAB0` consume meaningful CPU time and scale with naturally occurring heavy frametime windows?

Useful ideas should avoid per-call logging in a very hot path. Preferred sequence:

```text
count calls / branch entries
-> correlate with rendered frametime
-> aggregate or sampled timing if positive
-> patch only after material cost is measured
```

### 3. Independent reproduction

Useful reports include:

- exact ETS2 build
- renderer
- CPU/GPU/driver/OS
- resolution/scaling and relevant graphics settings
- actual rendered frametime in good/heavy regions
- whether unload/ferry/teleport changes the state without restart
- measurement method

Distinguish measured values from subjective impressions.

### 4. Review retained descriptor/root-binding work

The mapped shader-profile/descriptor architecture remains valid and runtime-relevant, but it is no longer treated as the complete root-cause theory.

Review is still useful when it challenges a concrete mapping or proposes a low-overhead measurement.

## Good contributions

Useful static reports usually contain:

- exact build
- executable hash when relevant
- build-specific function address(es)
- how the function was identified
- callers/callees, strings, types or dataflow supporting the mapping
- **FACT / INFERENCE / HYPOTHESIS** separation
- confidence level
- evidence against / uncertainty
- what measurement would discriminate the claim

For runtime measurements, also include:

- renderer and relevant settings
- hardware
- how rendered frametime was measured
- whether a number is a direct event, decoded/inferred counter, aggregate duration or sample

## Build policy

- `1.60.1.7s` — runtime profiling, probes and patch experiments
- `1.58.1.4s` — static-only pre-regression reference; it is **never run**

Addresses are build-specific. Never carry an address/function identity between versions without re-identification.

## Runtime experiment constraint

The active test workflow does not depend on hand-picked repeatable light/heavy saves.

Preferred experiments work during one ordinary gameplay session and correlate synchronized counters with independently measured rendered frametime afterward.

Natural unload/ferry/teleport transitions are useful evidence because the slowdown can sometimes reset without process restart.

See [`docs/methodology.md`](docs/methodology.md).

## Please avoid

- uploading SCS executables, assets or other proprietary files
- bulk raw decompiler dumps
- presenting a static candidate as a confirmed performance cause
- treating code-site `LOCK/UNLOCK` counts as executed-per-call counts
- treating reserved descriptor capacity as actual writes
- broad direct-D3D hooking when a narrow counter can answer the question
- global p3mem patches before path-specific runtime measurement
- repeating closed/demoted hypotheses without new evidence
- generic optimization advice unrelated to a measured path

## Pseudocode

Pseudocode in this repository is intentionally normalized and structural. It should explain the behavior relevant to the investigation without reproducing large raw decompiler listings.

## Opening an issue

Useful title examples:

```text
[1.60] Low-overhead counter for render_queue_set copy path
[1.58/1.60] Review of render_queue_set counterpart mapping
[Repro] Scene-dependent slowdown on Ryzen / Radeon system
```

Include the minimum evidence needed for another person to reproduce, verify or challenge the result.