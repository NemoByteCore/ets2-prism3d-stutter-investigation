# Methodology

## Core rules

1. Separate **FACT / INFERENCE / HYPOTHESIS**.
2. Do not repeat completed work without a concrete reason.
3. Change one meaningful variable per experiment.
4. Keep ETS2 `1.60.1.7s` runtime work strictly separate from `1.58.1.4s` static analysis.
5. Always include the game version with function addresses, e.g. `1.60:0x1402D7D70`.
6. Measure before patching.
7. A sampler is not a call tracer. A sample in a function does not prove an individual call is expensive.
8. Record negative results so rejected hypotheses are not repeatedly rediscovered.
9. Prefer non-destructive, fail-open experiments.
10. GitHub documents the investigation; it does not replace technical work.

## Runtime methodology

For `1.60.1.7s`:

- normal driving rather than synthetic F8-only snapshots
- automated probes where practical
- read-only instrumentation where practical
- exact counters / call duration when they answer the question better than broad sampling
- fail-open behavior for patch experiments

### Important frame telemetry caveat

`SCS frame_start` is not 1:1 with physically rendered frames. When rendering falls to ~45–50 FPS, the telemetry callback can still run around 60 Hz and catch up.

Therefore do **not** divide fixed-rate sampler counts by `frame_start` count and call the result `ms/rendered-frame`.

Prefer:

- samples/s
- wall time
- exact call-duration instrumentation

## Static comparison methodology

`1.58.1.4s` is a static reference only and is never launched.

The first comparison targets are the 1.60 equivalents around:

- `r_device_t::resource_build_bundle`
- semantic/resource resolver
- uniform evaluation / callback machinery
- resource packets
- allocator / buffer-management paths
- immediate callers and callees

The purpose is to identify code changes introduced between 1.58 and 1.59/1.60 that may explain the regression.

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

## Publication / sanitization

Do not commit SCS binaries or assets.

Prefer:

- original analysis
- pseudocode written from the investigation
- mappings
- scripts
- sanitized logs
- minimal excerpts necessary to explain a result

Larger decompiler excerpts should be reviewed before publication.
