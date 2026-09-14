# Current findings

## Scope

Target runtime build:

```text
ETS2:      1.60.1.7s
revision:  26c95e307fd5
renderer:  native DX12
EXE SHA-256:
1d61ba2337e4d8ced85a06e10566a4df064a2a0919ccd5e51561972d2a04255e
```

Primary symptom:

- light scenes: ~16.67 ms / ~60 FPS
- heavy scenes: typically ~19–25 ms
- background motion can feel like `start → stop → start → stop`
- severity depends on scene composition
- unload/teleport/ferry can restore ~16.67 ms without restarting the game

## Confirmed findings

### Sustained slowdown is not primarily a wait/fence problem

`NemoFramePacingProbe v0.3` showed:

- light scene (~16.67 ms): DXGI frame-latency gate can wait ~16 ms and frame fence has headroom
- heavy scene (~20–22 ms): DXGI gate waits almost nothing, 3-frame fence waits almost nothing, transfer wait is marginal, Prism sleep is marginal

**Interpretation:** CPU/render construction reaches submission/present too late. The slowdown is created earlier in the frame.

### The problem is scene-dependent

Ferry/teleport unloading can change a heavy ~19–20 ms state back to ~16.67 ms. Sleep/time-skip is not an equivalent reset.

### Instancing is active but simple instancing-volume metrics do not explain the slowdown

Known 1.60 functions:

- `1.60:0x1409FEDC0` — task ctor
- `1.60:0x1409FCCD0` — cluster extraction
- `1.60:0x1409FF640` — task execute
- `1.60:0x1409FF7F0` — worker per-item
- `1.60:0x140B5E870`, `1.60:0x140B5F4A0` — packers
- `1.60:0x140659440` — consume
- `1.60:0x140659930`, `1.60:0x140659680` — publish/rebuild render buffers

`NemoInstanceStateProbe` showed highly dynamic results, rare pending tasks, and weak correlation between raw bytes/chunks/clusters and frametime.

**Interpretation:** stale-result reuse and simple raw-instancing-volume explanations are not sufficient.

### Sampler descriptor redundancy is real and patchable

`NemoDX12SamplerReuse v0.4` keeps allocations but skips redundant `CopyDescriptorsSimple` calls for identical sampler tables within the tested flow.

Observed totals included:

- ~197.1M sampler copy calls
- ~186.7M skipped
- ~94.7% redundant
- later runs up to ~96% skipped
- no overflow/fail-open in the corrected merged version

The user reported a small subjective improvement.

**Status:** keep this optimization as a candidate component of a final patch unless later work shows a conflict.

## Important methodology correction

`SCS frame_start` telemetry is not 1:1 with physically rendered frames. At ~45–50 FPS, callback telemetry can still run around 60 Hz and catch up.

Therefore fixed-rate CPU samples must not be divided by `frame_start` count and labeled `ms/rendered-frame`.

Use instead:

- samples/s
- wall time
- exact call-duration instrumentation

## Current reverse-engineering correction

`1.60:FUN_14144C770` is only ~421 bytes and does not look like a large descriptor/resource builder. It creates a small context and conditionally rebuilds a provider/cache list when a key changes.

Model:

```text
key =
    rendergraph_context[+0xD0]       // ushort
  | item_internal[+0x142] << 16      // ushort
```

compared against a cached value around:

```text
item_internal[+0x11C]
```

If the key matches, heavier work is skipped.

**Interpretation:** a high fixed-rate sample count here can represent very high call frequency, not expensive individual calls.

Additional small helpers:

- `1.60:FUN_14022E380` — trivial `r_material_t` lookup
- `1.60:FUN_1401DB850` — simple `pp_batch_data_t` array helper

## Current render/resource chain

```text
RFX pass / shader profile
  ↓
resource bucketization + cross-stage merge
  ↓
composite uniform-builder merge when needed
  ↓
r_item
  ↓
1.60:FUN_14144C770
  ↓
1.60:FUN_1402D7D70
  ↓
r_resource_bundle_t
  ↓
1.60:FUN_1402942D0
  ↓
DX12 descriptor update/build
  ↓
1.60:FUN_14029E1F0
  ↓
generalized root-table/state submission
```

### `1.60:FUN_1402D7D70`

Current identification:

```cpp
r_device_t::resource_build_bundle(
    r_resource_bundle_t *,
    rendergraph_context_t *,
    const r_item_t &
)
```

### `1.60:FUN_1402E25A0`

Current identification: semantic/resource resolver.

It receives a semantic/resource ID (`byte`) and resolves a matching `r_resource_packet_entry_t`. If there is no simple hit, it can iterate active resource packets for the draw and scan semantic IDs.

A local per-bundle semantic memo remains a possible optimization experiment, but the 1.58 comparison now shows that this resolver algorithm already existed in essentially the same form before the reported regression.

## 1.58 ↔ 1.60 static comparison

High-confidence counterpart mapping now gives:

```text
1.60.1.7s:0x1402D7D70  ↔  1.58.1.4s:0x1401EB530
1.60.1.7s:0x1402E25A0  ↔  1.58.1.4s:0x1401F4FB0
1.60.1.7s:0x14144C770  ↔  1.58.1.4s:0x14129BF20
1.60.1.7s:0x1402942D0  ↔  1.58.1.4s:0x1401AC780
1.60.1.7s:0x14029E1F0  ↔  1.58.1.4s:0x1401B4800
```

See `docs/function-map.md` for the full mapping evidence.

### FACT — resolver and context helper are conserved

The semantic/resource resolver is the same 342-byte algorithm in both builds after accounting for relocated data/layout offsets. The small context/cache helper is likewise effectively identical at 421 bytes.

**Interpretation:** neither function currently looks like a newly introduced 1.60 algorithmic regression by itself, even though both can still be hot enough to optimize.

### FACT — one bundle-loop lookup moved out-of-line

The 1.58 `resource_build_bundle` performs its `uniform_builder_t` array lookup inline. The mapped 1.60 function calls a separate 97-byte helper at `1.60.1.7s:0x14144CE50` inside the corresponding repeated loop.

This adds a real function-call boundary in 1.60. Runtime call rate and exact timing are still required before assigning significance.

### FACT — descriptor/root-table state expanded substantially

The strongest static change appears downstream:

- the corresponding per-entry state stride in the descriptor builder grows from `0x1E0` in 1.58 to `0x2C0` in 1.60
- the same arena allocator is asked for `0x18` bytes in the 1.58 descriptor builder but `0x98` bytes in 1.60
- the 1.60 block explicitly zeroes 16 additional 64-bit table/state slots
- the mapped draw/state submission function grows from 1388 to 1984 bytes
- 1.58 uses a compact fixed descriptor-table path, while 1.60 iterates root-signature set descriptors and tracks a larger cached root-table state

These are static facts, not proof of runtime cost.

### FACT — pipeline-state lookup architecture changed

The mapped pipeline-state lookup path grows from a 180-byte synchronous routing function in 1.58 to a 394-byte 1.60 path that can enqueue `compile_pipeline_task_t` work and manage a bounded task queue.

Again, this requires hit/miss/task-rate measurement before it can be treated as a contributor to sustained slowdown.

## Deep static architecture delta

A deeper pass shows that the descriptor/root-table expansion is part of a wider 1.60 redesign rather than an isolated backend change.

### FACT — RFX passes gain shader-profile selection

The 1.60 RFX/pass path contains shader-profile parsing and validation absent from the mapped 1.58 flow.

The parser recognizes 13 shader-profile names. The selected profile is carried into the shader-pipeline representation and later chooses one of 13 prebuilt DX12 root-signature profiles.

The full profile table still needs targeted static-data extraction.

### FACT — resources gain three-way bucketization

1.60 resource metadata carries an additional selector that can assign a resource to one of three binding buckets/sets.

Pipeline construction merges those bucketed bindings across shader stages and combines visibility.

### FACT — `uniform_builder_t` expands and gains composite merge support

The corresponding builder stride changes:

```text
1.58: 0x30 = 48 bytes
1.60: 0x40 = 64 bytes
```

A 1.60 helper at `0x14144C160` can combine differing builders while merging/deduplicating callback entries.

This does not yet prove that more uniform callbacks execute per draw.

### FACT — generalized profile-driven root-table handling

The new architecture currently reads as:

```text
RFX shader profile
→ 3-way resource buckets
→ cross-stage layout merge
→ composite uniform builders when needed
→ pipeline profile ID
→ one of 13 DX12 root-signature profiles
→ generalized set/table mapping
→ cached root-table handles
→ conditional binds
```

See `docs/shader-profile-architecture-delta.md` for the dedicated summary.

## Open static question: pipeline cache vs shader profile

A high-interest 1.60 pipeline cache/create path is `0x1402E4D50`.

Current static reading shows the lookup key built from six shader identities, while the selected shader-profile ID is stored in the created pipeline object.

It is not yet established whether profile ID also participates indirectly in the cache identity or whether the engine enforces an invariant that one six-shader combination can only ever use one profile.

**This is an open question, not a confirmed cache bug.**

## Uniform callback machinery

Static 1.60 corpus currently shows roughly:

- ~280 registrations
- ~272 distinct names
- ~249 distinct targets

Examples include:

- `material_diffuse`
- `material_specular`
- `material_environment`
- `paint`
- `tint`
- `anim_params`
- `transform_mvp_matrix`
- `transform_camera_offset`
- `fwd_lights_data`
- `shadowmap_bias`

This suggests cost may be distributed across many callbacks rather than concentrated in one obvious function.

## Best current technical model

```text
heavy scene
→ many draws/material passes
→ 1.60 shader-profile/resource-layout architecture
→ bucket merge + expanded descriptor/update state
→ generalized profiled root-table submission
→ CPU reaches Present too late
```

This is a working regression model, not yet proof of the exact cost mechanism.

Highest-interest areas now:

- complete definition of the 13 shader/root-signature profiles
- shader-combination ↔ profile cache invariant around `0x1402E4D50`
- profile-dependent descriptor/update complexity
- `0x1402942D0` and `0x14029E1F0` as later profile-aware runtime measurement targets
- `0x14144C160` / expanded `uniform_builder_t` representation as a secondary static branch

The semantic resolver remains a plausible optimization target, but it is no longer the strongest static candidate for explaining the 1.58 → 1.60 regression.

## Next step

Do **not** start with the previously planned generic runtime probe yet.

First perform targeted static-data/Ghidra extraction of the 13 shader-profile definitions:

- profile name → numeric ID
- sets/root parameters per profile
- descriptor-table classes/ranges per set
- profile complexity differences
- profile-assignment invariants relative to shader-pipeline caching

After that, design a profile-aware runtime probe that can report descriptor/root-table cost and bind activity by profile rather than only global function timings.

## Version-comparison policy

`1.58.1.4s` is used only as a static pre-regression snapshot. It is not launched.

Evidence categories must stay separate:

```text
community reports:
1.58 good → 1.59/1.60 bad

our runtime evidence:
1.60 reproduced and profiled

our static evidence:
1.58 vs 1.60 binary/code comparison
```
