# ETS2 / Prism3D stutter investigation

Active reverse-engineering investigation of scene-dependent CPU-side stutter in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

> **Status:** the strongest regression-shaped static delta is now understood well enough to test directly at runtime. The next stage is a profile-aware, single-session probe synchronized with actual rendered frametime.

## What is being investigated

On the tested system, light scenes can hold roughly **16.67 ms / 60 FPS**, while heavier scene compositions can move into roughly **19–25 ms**. The slowdown is scene-dependent and can sometimes disappear after an unload/ferry/teleport transition without restarting the game.

Earlier runtime work showed that the sustained slowdown is **not primarily a DXGI wait / fence-wait problem**. The CPU reaches submission/present too late; the interesting work happens earlier in the render-construction path.

## Strongest current finding

The most important static difference found so far is the DX12 binding architecture used by the mapped 1.58 and 1.60 paths.

```text
1.58:
actual pipeline layout
→ layout-specific root signature
→ layout-specific descriptor capacities
→ compact resource table + sampler table model

1.60:
RFX shader profile
→ one of 13 fixed root-signature profiles
→ fixed profile descriptor capacities
→ individual root CBVs + split per-set tables
→ generalized root-parameter submission
```

The 13 profile definitions have been decoded. Common heavy profiles include:

| Profile | Root params | Root CBVs | Resource slots | Sampler slots | Table roots |
|---|---:|---:|---:|---:|---:|
| `material` | 11 | 7 | 20 | 20 | 4 |
| `lightpass` | 8 | 4 | 24 | 18 | 4 |
| `fullscreen` | 8 | 4 | 24 | 16 | 4 |

This is a **regression-shaped architectural change, not yet proof of the 3–8 ms slowdown**. The next runtime work is designed to measure whether fixed-profile reservation and expanded root binding actually rise disproportionately when rendered frametime gets worse.

See [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md) for the detailed static comparison.

## Current next step

Design and run **`NemoShaderProfileProbe v0.2`** around one normal gameplay session.

The active save does not allow arbitrary staging of repeatable “light” and “heavy” scenes, so the experiment must work during ordinary play:

```text
one normal gameplay session
→ actual rendered frametime
→ profile/resource/root-binding counters
→ short synchronized windows
→ post-hoc split into good/bad frametime regions
```

High-value measurements:

- draw/work count by shader profile
- descriptor-builder time
- fixed descriptor capacity reserved
- actual descriptor writes/copies where directly instrumented
- actual `SetGraphicsRoot*` calls where directly instrumented
- root-signature/profile switches
- descriptor heap rollover/switches
- shader-tuple/profile conflicts
- actual rendered frametime aligned with all of the above

`NemoShaderProfileProbe v0.1` exists only as an implementation reference; its original separate light/heavy-scene protocol is rejected and should not be used for root-cause or patch decisions.

## Help wanted

Useful outside contributions are very welcome. The most valuable help right now is:

- reviewing the mapped **1.58 ↔ 1.60 DX12 root-signature / descriptor path**
- suggesting low-overhead instrumentation for the v0.2 single-session probe
- reproducing the scene-dependent slowdown on other systems with exact build/renderer/settings documented
- checking whether one six-shader tuple can ever be requested with more than one 1.60 shader-profile ID
- reviewing normalized pseudocode and function identifications

If you have evidence, open an issue. A small, reproducible observation is more useful than generic performance advice.

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the evidence format.

## Start here

- [`docs/current-findings.md`](docs/current-findings.md) — current technical state
- [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md) — strongest 1.58 ↔ 1.60 architecture delta
- [`docs/function-map.md`](docs/function-map.md) — build-specific function map
- [`docs/methodology.md`](docs/methodology.md) — runtime/static methodology and evidence rules
- [`docs/experiments.md`](docs/experiments.md) — experiment summaries
- [`docs/disproven-hypotheses.md`](docs/disproven-hypotheses.md) — dead ends that should not be rediscovered
- [`pseudocode/`](pseudocode/) — normalized reconstructions

## Build policy

### Runtime / test / fix target

```text
ETS2:      1.60.1.7s
revision:  26c95e307fd5
renderer:  native DX12
SHA-256:   1D61BA2337E4D8CED85A06E10566A4DF064A2A0919CCD5E51561972D2A04255E
```

### Static-only reference

```text
ETS2:      1.58.1.4s
SHA-256:   AB9785331BF9970542C61A0108A4E677C9F7C00FD316D4C0F9AB116F6BE6C234
```

**1.58.1.4s is never run in this investigation.** It is used only as a pre-regression static comparison point.

## Evidence policy

Every result should keep **FACT / INFERENCE / HYPOTHESIS** separate and identify the build for every address.

No SCS executables, proprietary game assets, giant raw decompiler dumps, private handoffs or local-only data are published here. This repository contains original analysis, normalized pseudocode, mappings and reproducible research notes.

## Repository layout

```text
.
├─ README.md
├─ CONTRIBUTING.md
├─ docs/
│  ├─ current-findings.md
│  ├─ shader-profile-architecture-delta.md
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
