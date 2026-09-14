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

- `1.60:FUN_14022E380` — appears to be a trivial material lookup
- `1.60:FUN_1401DB850` — appears to be a simple `pp_batch_data_t` indexer

## Current render/resource chain

```text
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
state / root tables / draw submission
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

This is currently a higher-priority investigation point than `14144C770` itself.

### `1.60:FUN_1402E25A0`

Current identification: semantic/resource resolver.

It receives a semantic/resource ID (`byte`) and resolves a matching `r_resource_packet_entry_t`. If there is no simple hit, it can iterate active resource packets for the draw and scan semantic IDs.

Promising future PoC:

**memoize semantic → resource_packet_entry only within one `resource_build_bundle` call.**

Rationale:

- no cache across frames
- no cache across draws
- low stale-state risk
- first lookup remains Prism's normal behavior
- repeated semantic IDs in the same bundle can avoid repeated scanning

First step: measure the repeat rate before patching.

## Uniform callback machinery

Static corpus currently shows roughly:

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
→ very high per-draw / per-resource-bundle work
→ uniform evaluation + semantic/resource resolution
→ descriptor construction
→ draw/state submission
→ CPU reaches Present too late
```

Highest-interest areas now:

- `1.60:FUN_1402D7D70` — resource bundle construction
- `1.60:FUN_1402E25A0` — semantic/resource resolution
- uniform callback machinery
- descriptor builder downstream

Potential future fixes remain hypotheses:

- local per-bundle semantic-resolution memo
- split/template the static part of a bundle per material/effect/pipeline
- keep dynamic uniforms/view/object state fresh

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
