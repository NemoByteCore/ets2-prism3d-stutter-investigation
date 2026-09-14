# ETS2 / Prism3D stutter investigation

Reverse-engineering investigation of scene-dependent CPU-side stutter in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

## Goal

Find the CPU-side work that makes some scenes miss the ~16.67 ms frame budget, identify a safe workaround or patch if possible, and document the investigation well enough for the ETS2 modding / reverse-engineering community and SCS Software to reproduce or continue it.

## Current model

```text
heavy scene
→ high per-draw / per-resource-bundle CPU work
→ uniform evaluation + semantic/resource resolution
→ descriptor construction
→ draw/state submission
→ CPU reaches Present too late
```

Current high-interest functions in **1.60.1.7s**:

- `0x1402D7D70` — `r_device_t::resource_build_bundle(...)`
- `0x1402E25A0` — semantic/resource resolver
- `0x14144C770` — small uniform/resource context + cache-key path
- descriptor update/build and submission path downstream

See [`docs/function-map.md`](docs/function-map.md) for the build-specific function map and [`pseudocode/`](pseudocode/) for normalized reconstructions with FACT / INFERENCE / HYPOTHESIS kept separate.

A DX12 sampler-descriptor optimization (`NemoDX12SamplerReuse v0.4`) has already shown that a large amount of descriptor work is redundant and can be skipped safely in the tested setup. It remains a candidate component of a final patch.

## Help wanted

Useful outside contributions include:

- identifying 1.58 counterparts of the mapped 1.60 functions
- checking the resource/buffer/render path for changes introduced in 1.59/1.60
- reviewing the normalized pseudocode and call-chain interpretation
- suggesting low-risk instrumentation for exact per-call timing and repeat-rate measurements
- reproducing the scene-dependent slowdown on other systems with clearly documented conditions

If you recognize one of the code patterns, please open an issue with the relevant build, address, evidence and reasoning. Concrete evidence is much more useful than generic performance suggestions.

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the format used in this project.

## Version policy

- **1.60.1.7s** — runtime profiling, probes and patch experiments.
- **1.58.1.4s** — static reverse-engineering reference only. It is **not run** and is used as a pre-regression comparison point.

Reference hashes:

```text
1.60.1.7s
1D61BA2337E4D8CED85A06E10566A4DF064A2A0919CCD5E51561972D2A04255E

1.58.1.4s
AB9785331BF9970542C61A0108A4E677C9F7C00FD316D4C0F9AB116F6BE6C234
```

## Repository layout

```text
.
├─ README.md
├─ CONTRIBUTING.md
├─ docs/
│  ├─ current-findings.md
│  ├─ function-map.md
│  ├─ methodology.md
│  ├─ timeline-1.58-1.60.md
│  ├─ render-call-chain.md
│  ├─ experiments.md
│  └─ disproven-hypotheses.md
└─ pseudocode/
   ├─ README.md
   ├─ 1.60/
   └─ 1.58/
```

The public repository intentionally stays small. Large raw decompiler exports, Ghidra project databases, game binaries and local workspace files are not published here.

## Evidence policy

Every result should record the build, function address, method, result, interpretation, confidence and next step.

Facts, inferences and hypotheses must remain explicitly separated. Addresses are build-specific and must never be assumed to match across versions without re-identification.

No SCS binaries or assets are committed. The repository contains original analysis, normalized pseudocode, mappings and research notes rather than bulk decompiler output.

## Runtime build currently under investigation

```text
ETS2:      1.60.1.7s
revision:  26c95e307fd5
renderer:  native DX12
EXE SHA-256:
1d61ba2337e4d8ced85a06e10566a4df064a2a0919ccd5e51561972d2a04255e
```

See [`docs/current-findings.md`](docs/current-findings.md) for the current technical state.