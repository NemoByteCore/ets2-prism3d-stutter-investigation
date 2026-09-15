# Current findings

Updated: **2026-09-15**

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

**1.58.1.4s is not run.**

Primary symptom on the tested setup:

- light scenes can hold roughly ~16.67 ms / ~60 FPS
- heavier scene compositions commonly reach ~19–25 ms
- background motion can feel like `start → stop → start → stop`
- severity depends on scene composition
- unload/ferry/teleport transitions can sometimes restore ~16.67 ms without restarting the game

## Confirmed runtime findings

### Sustained slowdown is not primarily a wait/fence problem

`NemoFramePacingProbe v0.3` showed that in slower scenes the DXGI frame-latency gate and the tested fence waits are not consuming enough time to explain the sustained slowdown.

**FACT:** the CPU/render construction path reaches submission/present too late. The interesting work is earlier in the frame.

### The slowdown is scene-dependent

Changing loaded scene state can move the same running game between a slower ~19–25 ms state and ~16.67 ms behavior.

### Simple instancing-volume metrics do not explain it

`NemoInstanceStateProbe` showed highly dynamic instancing activity, but raw bytes/chunks/clusters did not track frametime strongly enough to explain the slowdown by themselves.

### Sampler descriptor redundancy is real

`NemoDX12SamplerReuse v0.4` safely skipped redundant sampler `CopyDescriptorsSimple` work in the tested flow.

Observed runs showed roughly **95–96%** of the targeted sampler copies were redundant. The user reported a small subjective improvement.

**Interpretation:** this optimization remains a plausible component of a final patch, but it does not by itself explain the whole regression.

## Important telemetry correction

`SCS frame_start` telemetry is not guaranteed to be 1:1 with physically rendered frames. At ~45–50 FPS, the callback can still run around 60 Hz and catch up.

Do not divide fixed-rate sample counts by `frame_start` count and call that value `ms/rendered-frame`.

Prefer:

- wall time
- samples/s
- exact call-duration instrumentation
- actual rendered-frame timing when testing frame-synchronous hypotheses

## Mapped render/resource chain

High-confidence mapped path in 1.60:

```text
RFX pass / shader profile
  ↓
resource bucketization + cross-stage merge
  ↓
composite uniform-builder merge when needed
  ↓
r_item
  ↓
1.60.1.7s:0x14144C770
  ↓
1.60.1.7s:0x1402D7D70
  ↓
r_resource_bundle_t
  ↓
1.60.1.7s:0x1402942D0
  ↓
DX12 descriptor update/build
  ↓
1.60.1.7s:0x14029E1F0
  ↓
generalized root-parameter submission
```

Important mapped counterparts:

```text
1.60.1.7s:0x1402D7D70  ↔  1.58.1.4s:0x1401EB530
1.60.1.7s:0x1402E25A0  ↔  1.58.1.4s:0x1401F4FB0
1.60.1.7s:0x14144C770  ↔  1.58.1.4s:0x14129BF20
1.60.1.7s:0x1402942D0  ↔  1.58.1.4s:0x1401AC780
1.60.1.7s:0x14029E1F0  ↔  1.58.1.4s:0x1401B4800
```

See [`function-map.md`](function-map.md) for mapping evidence.

## Confirmed static architecture delta

The strongest regression-shaped architectural difference found so far is now specific enough to test directly.

### 1.58 model

The mapped 1.58 DX12 path builds a root signature from the actual pipeline layout.

```text
actual pipeline layout
→ layout-specific root signature
→ layout-specific descriptor capacities
→ compact resource descriptor table
→ compact sampler descriptor table
```

CBV/SRV/UAV ranges are aggregated into the resource table in the mapped path.

### 1.60 model

1.60 introduces shader-profile selection and chooses one of **13 fixed DX12 root-signature profiles**.

```text
RFX shader profile
→ fixed root-signature profile
→ fixed descriptor capacities
→ individual root CBVs
→ split SRV/UAV/sampler tables by set/visibility
→ generalized root-parameter submission
```

Exact profile names:

```text
0  compute
1  fullscreen
2  simple0
3  simple1
4  simple2
5  simple3
6  simple4
7  simple1_shared
8  lightpass
9  material
10 material_lite
11 shadow
12 sky
```

Selected profile capacities:

| Profile | Root params | Root CBVs | Resource slots | Sampler slots | Table roots |
|---|---:|---:|---:|---:|---:|
| `material` | 11 | 7 | 20 | 20 | 4 |
| `lightpass` | 8 | 4 | 24 | 18 | 4 |
| `fullscreen` | 8 | 4 | 24 | 16 | 4 |
| `material_lite` | 6 | 4 | 6 | 6 | 2 |

Full table and decoding details are in [`shader-profile-architecture-delta.md`](shader-profile-architecture-delta.md).

## Descriptor reservation is profile-capacity driven in 1.60

`1.60.1.7s:0x1402942D0` resolves the selected profile and uses profile totals when reserving resource and sampler descriptor-heap slots.

For example, the `material` profile carries capacity for 20 resource descriptors and 20 samplers.

**FACT:** reservation is based on fixed profile capacity.

**Open runtime question:** how many of those reserved slots are actually written/copied on typical draws?

## Root binding is more fragmented in 1.60

Mapped draw submission:

```text
1.58.1.4s:0x1401B4800
1.60.1.7s:0x14029E1F0
```

The mapped 1.58 path conditionally binds a compact resource table and sampler table.

The mapped 1.60 path can instead:

- scan multiple root-CBV slots
- issue individual root-CBV updates
- track several cached root-parameter values
- handle SRV/UAV/sampler table classes independently
- bind multiple table roots per profile

This is a static architectural difference, not yet measured proof of the observed frametime loss.

## Pipeline-cache/profile identity audit remains open

Mapped pipeline cache/create path:

```text
1.60.1.7s:0x1402E4D50
```

The visible pre-lookup cache identity is derived from six shader identities, while the selected profile ID is stored on the created pipeline and later controls root-signature selection.

A cache hit returns before an explicit requested-profile comparison is visible in this function.

**HYPOTHESIS / audit target:** determine whether the same six-shader tuple can ever be requested with more than one profile ID.

This is **not a confirmed cache bug**. The asset/data model may guarantee one profile per shader tuple.

## Best current technical model

```text
heavy scene / more complex draw mix
→ more work through 1.60 shader-profile binding architecture
→ fixed profile-capacity reservation
→ expanded descriptor/root-binding bookkeeping
→ CPU reaches submission/present too late
```

This is the strongest current regression model, but runtime correlation is still required.

## Current runtime constraint

The active save does not provide arbitrary control over test scenes.

Do not design the main experiment around:

- hand-picked stable light/heavy scenes
- separate debug saves
- arbitrary teleporting solely for measurement
- reproducing two exact scene compositions on demand

The valid methodology must work during normal play on the current save.

## NemoShaderProfileProbe v0.1 status

`NemoShaderProfileProbe v0.1` built successfully, but its original test protocol is rejected for root-cause/patch decisions.

Reasons include:

- separate light/heavy runs do not fit the available save
- it lacks a sufficiently synchronized actual rendered-frametime signal
- decoded layout counts are not the same thing as actual descriptor writes/copies
- predicted root-CBV/table activity is not direct `SetGraphicsRoot*` instrumentation
- whole-run totals cannot show what changes exactly when frametime worsens

The v0.1 artifact remains an implementation reference only.

## Next step — NemoShaderProfileProbe v0.2

Design a **single-session, save-compatible, profile-aware runtime probe**.

Preferred experiment shape:

```text
one ordinary gameplay session
→ continuous actual rendered frametime
→ continuous profile/resource/root-binding counters
→ short synchronized windows
→ post-hoc split into good/bad frametime regions
```

High-value counters:

- draw/work count by profile ID
- descriptor-builder time
- resource/sampler slots reserved by profile
- actual descriptor writes/copies where directly instrumented
- actual root-CBV API calls where directly instrumented
- actual descriptor-table binds where directly instrumented
- root-signature/profile switches
- descriptor heap rollover/switches
- shader-tuple/profile conflicts
- actual rendered frametime aligned with the above

The useful question is not merely whether a visually heavier scene does more work. It is **which measured work changes disproportionately and synchronously when actual rendered frametime worsens inside the same session**.

## Evidence categories

Keep these separate:

```text
community reports:
1.58 good → 1.59/1.60 bad

our runtime evidence:
1.60 reproduced and profiled

our static evidence:
1.58 vs 1.60 binary/code comparison
```
