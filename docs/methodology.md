# Methodology

## Core rules

1. Separate **FACT / INFERENCE / HYPOTHESIS**.
2. Do not repeat completed work without a concrete reason.
3. Change one meaningful variable per experiment where practical.
4. Keep ETS2 `1.60.1.7s` runtime work strictly separate from `1.58.1.4s` static analysis.
5. Always include the game version with function addresses, e.g. `1.60.1.7s:0x1402D7D70`.
6. Measure before patching.
7. A sampler is not a call tracer. A sample in a function does not prove an individual call is expensive.
8. Record negative results so rejected hypotheses are not repeatedly rediscovered.
9. Prefer non-destructive, fail-open experiments.
10. GitHub documents the investigation; it does not replace technical work.

## Runtime methodology

For `1.60.1.7s`:

- normal driving rather than synthetic snapshot-only tests
- automated probes where practical
- read-only instrumentation where practical
- exact counters / call duration when they answer the question better than broad sampling
- fail-open behavior for patch experiments
- actual rendered-frame timing when testing frame-synchronous hypotheses

### Hard constraint: experiments must fit the active save

The active save does **not** provide arbitrary control over scene composition.

Do not require:

- separate hand-picked light/heavy saves
- teleporting solely to stage a chosen test scene
- reproducing two exact scene compositions on demand
- maintaining special debug saves just for instrumentation

A two-run design such as:

```text
run A = chosen light scene
run B = chosen heavy scene
```

is not the preferred primary experiment here. It assumes scene-selection control that is not available and adds scene-composition confounding.

### Preferred design: within-session synchronized measurement

The next runtime probes should work during one ordinary gameplay session:

```text
normal gameplay
  ↓
continuous actual rendered frametime
  ↓
continuous counters / direct instrumentation
  ↓
short synchronized windows
  ↓
post-hoc split into good/bad frametime regions
```

The useful question is:

> Which measured work changes disproportionately and synchronously when rendered frametime worsens during the same session?

Natural unload/ferry/teleport transitions can be used as extra evidence if they occur naturally in the current save, but they are not a required test step.

### Current v0.2 measurement priorities

For the shader-profile / descriptor path, high-value measurements include:

- draw/work count by profile ID
- descriptor update/build time
- fixed resource/sampler capacity reserved by profile
- actual descriptor writes/copies where directly instrumented
- actual root-CBV `SetGraphicsRootConstantBufferView` calls
- actual descriptor-table bind calls
- root-signature/profile switches
- descriptor heap rollover/switch events
- shader-tuple/profile conflicts
- actual rendered frametime aligned with the same windows

Distinguish **real engine/API events** from counters inferred from decoded state.

## Important frame telemetry caveat

`SCS frame_start` is not 1:1 with physically rendered frames. When rendering falls to ~45–50 FPS, the telemetry callback can still run around 60 Hz and catch up.

Therefore do **not** divide fixed-rate sampler counts by `frame_start` count and call the result `ms/rendered-frame`.

Prefer:

- samples/s
- wall time
- exact call-duration instrumentation
- independently measured rendered-frame timing

## Static comparison methodology

`1.58.1.4s` is a static reference only and is never launched.

The current comparison concentrates on the mapped path around:

- `r_device_t::resource_build_bundle`
- semantic/resource resolution
- shader-profile selection
- descriptor reservation/update
- root-signature representation
- root-CBV / descriptor-table submission
- pipeline cache/profile identity

The purpose is to identify architectural changes introduced between the pre-regression and current paths, then design runtime measurements that can test those differences without assuming causality.

## Result template

Every substantial result should record:

```text
Build:
Function/address:
Method:
Result:
Interpretation:
Confidence:
Next step:
```

For runtime probes also record enough timing/context information to align the result with actual rendered performance.

## Publication / sanitization

Do not commit SCS binaries or proprietary assets.

Prefer:

- original analysis
- normalized pseudocode
- mappings
- reproducible methodology
- sanitized logs
- minimal excerpts necessary to explain a result

Do not publish private handoffs, local-only machine data or giant raw decompiler exports in this repository.
