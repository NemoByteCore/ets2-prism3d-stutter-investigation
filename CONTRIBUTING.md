# Contributing

This is an evidence-driven reverse-engineering investigation. Contributions are useful when they verify a build-specific mapping, reproduce the slowdown on another system, improve low-overhead instrumentation, or challenge the current runtime model with concrete evidence.

## Best ways to help right now

### 1. Reproduce the scene-dependent slowdown

Useful reports include:

- exact ETS2 build;
- renderer;
- CPU / GPU / driver / Windows version;
- resolution, scaling and relevant graphics settings;
- actual rendered frametime in light and heavy scenes;
- whether unload/ferry/teleport changes the state without restarting;
- measurement method.

Please distinguish measured values from subjective impressions.

### 2. Review the current measured leaf

The active render-side path is:

```text
RG_CORE
  -> T1 helper
    -> pass callback
      -> nested winner
        -> RQ_ONE
          -> HEAD_DISPATCH 1.60.1.7s:0x14154CF60
            -> 1.60.1.7s:0x1402D8D20
```

Current static counterparts:

```text
1.60.1.7s:0x14154CF60  <->  1.58.1.4s:0x1413D7700
1.60.1.7s:0x1402D8D20  <->  1.58.1.4s:0x1401EC530
```

High-value review includes:

- confirming or challenging those counterpart mappings;
- identifying semantic differences in the downstream routine;
- checking the no-split vs ranged path interpretation;
- spotting a lower-overhead discriminator than the current one.

### 3. Suggest low-overhead instrumentation

The project has already seen substantial observer effect from broad hot-path hooks.

Preferred order:

```text
cheap count / path identity
  -> aggregate or sampled timing
    -> matched-work comparison
      -> behavior patch only after a material owner is measured
```

Per-call logging in hot paths is generally not useful.

### 4. Audit public mappings and documentation

Addresses are build-specific. Corrections are welcome when an address is:

- missing an exact build qualifier;
- presented as portable across versions;
- mapped without enough static evidence;
- described with a stale conclusion.

## Build policy

- `1.60.1.7s` — runtime profiling, probes and patch experiments;
- `1.58.1.4s` — static-only pre-regression reference; it is **never run**.

Never carry an address/function identity between versions without re-identification.

## Evidence format

Useful reports should separate:

- **FACT** — directly measured or statically verified;
- **INFERENCE** — the most direct interpretation of the evidence;
- **HYPOTHESIS** — a plausible mechanism still awaiting a discriminator.

For static findings, include:

- exact build;
- executable hash when relevant;
- build-specific address;
- how the function was identified;
- callers/callees, strings, types or dataflow supporting the mapping;
- uncertainty or evidence against.

For runtime findings, also include:

- renderer/settings/hardware;
- how rendered frametime was measured;
- whether a value is a direct event, aggregate duration, sampled duration or inferred estimate.

## Please avoid

- uploading SCS executables, assets or other proprietary files;
- bulk raw decompiler dumps;
- presenting a static candidate as a confirmed performance cause;
- broad direct-D3D hooking when a narrow measurement can answer the question;
- global p3mem patches before path-specific runtime evidence;
- repeating closed/demoted hypotheses without new evidence;
- generic optimization advice unrelated to a measured path.

See:

- [`docs/current-findings.md`](docs/current-findings.md)
- [`docs/rg-core-runtime-localization.md`](docs/rg-core-runtime-localization.md)
- [`docs/methodology.md`](docs/methodology.md)
- [`docs/disproven-hypotheses.md`](docs/disproven-hypotheses.md)

## Pseudocode

Public pseudocode should be normalized and structural. It should explain the behavior relevant to the investigation without reproducing large raw decompiler listings.
