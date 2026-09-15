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
- background motion can feel like `start -> stop -> start -> stop`
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

## 1.60 shader-profile architecture is materially different from 1.58

High-confidence mapped chain in 1.60:

```text
RFX pass / shader profile
  -> resource bucketization + cross-stage merge
  -> composite uniform-builder merge when needed
  -> r_item
  -> 1.60.1.7s:0x14144C770
  -> 1.60.1.7s:0x1402D7D70
  -> r_resource_bundle_t
  -> 1.60.1.7s:0x1402942D0
  -> DX12 descriptor update/build
  -> 1.60.1.7s:0x14029E1F0
  -> generalized root-parameter submission
```

Important mapped counterparts:

```text
1.60.1.7s:0x1402D7D70  <->  1.58.1.4s:0x1401EB530
1.60.1.7s:0x1402E25A0  <->  1.58.1.4s:0x1401F4FB0
1.60.1.7s:0x14144C770  <->  1.58.1.4s:0x14129BF20
1.60.1.7s:0x1402942D0  <->  1.58.1.4s:0x1401AC780
1.60.1.7s:0x14029E1F0  <->  1.58.1.4s:0x1401B4800
```

The mapped 1.58 DX12 path builds a root signature from the actual pipeline layout. 1.60 instead selects one of **13 fixed DX12 root-signature profiles**.

```text
1.58:
actual layout
-> layout-specific root signature
-> layout-specific descriptor capacities
-> compact resource table + sampler table

1.60:
RFX shader profile
-> fixed root-signature profile
-> fixed descriptor capacities
-> individual root CBVs
-> split SRV/UAV/sampler tables by set/visibility
-> generalized root-parameter submission
```

Exact 1.60 profile names:

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

Selected fixed capacities:

| Profile | Root params | Root CBVs | Resource slots | Sampler slots | Table roots |
|---|---:|---:|---:|---:|---:|
| `material` | 11 | 7 | 20 | 20 | 4 |
| `lightpass` | 8 | 4 | 24 | 18 | 4 |
| `fullscreen` | 8 | 4 | 24 | 16 | 4 |
| `material_lite` | 6 | 4 | 6 | 6 | 2 |

See [`shader-profile-architecture-delta.md`](shader-profile-architecture-delta.md).

## Fixed profile reservation is now confirmed at runtime

`1.60.1.7s:0x1402942D0` passes fixed root-profile totals to the descriptor allocator. A broad structural runtime probe showed that, in active gameplay, fixed capacity is much larger than actual layout demand.

Median active-frame ratios from that run:

```text
reserved resource capacity / actual resource-layout demand ~7.62x
reserved sampler capacity  / actual sampler demand         ~8.34x
```

Typical per-draw sampler values were roughly:

```text
reserved sampler slots ~18.0/draw
actual sampler bindings ~2.12/draw
```

The `material` profile accounted for about 87% of active gameplay draws in that capture.

The probe itself was too intrusive for absolute frametime attribution, but these structural ratios are useful.

## Sampler heap pressure is a real optimization target

Static allocator reconstruction shows:

```text
1.60.1.7s:0x14028F070  descriptor allocation / cursor advance
sampler heap capacity   0x800 = 2,048 descriptors
resource heap capacity  0x80000 = 524,288 descriptors
```

The fixed profile sampler spans therefore matter much more for heap pressure than the same style of over-reservation on the much larger resource heap.

### Earlier copy-only result

`NemoDX12SamplerReuse v0.4` safely skipped roughly 95–96% of targeted sampler descriptor copies, but deliberately left Prism's original sampler allocations untouched. The user reported a small subjective improvement.

### Allocator-side reuse result

`NemoDX12SamplerAllocReuse v0.2` moved reuse before final sampler allocation/materialization.

Observed ordinary-play run:

```text
sampler_alloc_provisional = 125,772,004
unique_tables             =   5,089,396
duplicate_tables          = 108,312,238
zero_copy_tables          =  12,370,370
requested_slots_original  = 2,251,985,241
allocated_slots_real      =    96,277,760
avoided_slots             = 2,156,696,505
sampler_copy_calls_seen   =   247,221,488
copy_calls_skipped        =   235,147,658
```

Derived:

```text
~86.12% duplicate tables
~9.84% zero-copy tables
~4.05% real nonzero unique tables
~95.77% requested sampler slots avoided
~95.12% sampler copy calls skipped
```

All safety/error counters were zero.

A later v0.3 run reproduced the same scale of reduction (~95.18% slots avoided, ~94.49% copy calls skipped) with all safety counters still zero.

**FACT:** sampler allocation/copy pressure created by the fixed-profile path is large and safely reducible.

**Important:** the broader heavy-scene slowdown has still occurred in runs where sampler pressure was strongly reduced. Sampler reuse is therefore a validated optimization component, **not the complete root cause or complete fix**.

See [`experiments.md`](experiments.md) for experiment details.

## Root binding remains a plausible remaining cost

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

The broad structural probe showed this work scales largely with draw count rather than exploding per draw in slow regions. This remains relevant, but broad per-call tracing is too intrusive to use as an unbiased perf measurement.

## Pipeline-cache/profile collision hypothesis is lower priority

Mapped pipeline cache/create path:

```text
1.60.1.7s:0x1402E4D50
```

Static analysis raised a concern that the visible pre-lookup key is based on six shader identities while profile ID is stored separately on creation.

Runtime audit result:

```text
tuple requests          524
unique shader tuples    377
same-profile repeats    147
tuple/profile conflicts 0
```

This is negative evidence against a practical tuple/profile collision in the captured run. It does not prove the invariant globally, but the hypothesis is now lower priority and should not be presented as a confirmed cache bug.

## Important telemetry correction

`SCS frame_start` telemetry is not guaranteed to be 1:1 with physically rendered frames. At ~45–50 FPS, the callback can still run around 60 Hz and catch up.

Do not divide fixed-rate sample counts by `frame_start` count and call that value `ms/rendered-frame`.

Prefer actual rendered-frame timing, wall time and narrow exact counters.

## Runtime methodology constraint

The active save does not provide arbitrary control over test scenes. Main experiments must work during ordinary play in one session rather than requiring hand-picked light/heavy saves or exact scene reproduction.

Broad direct-D3D tracing was also shown to have a substantial observer effect, so future runtime work should prefer lightweight synchronized markers/counters and narrow hooks.

## Best current technical model

```text
heavy scene / more draw work
-> more traffic through the 1.60 fixed-profile descriptor/root-binding architecture
-> large roughly per-draw fixed-capacity bookkeeping
-> sampler allocation/copy pressure (now strongly optimized)
-> remaining descriptor/root-binding/render-construction work
-> CPU reaches submission/present too late
```

The fixed-profile architecture remains the strongest regression-shaped 1.58 -> 1.60 delta, but the successful sampler optimization demonstrates that no single measured sampler cost explains the entire slowdown.

## Current direction

Do not return to a broad profiler.

Current work should:

1. keep the validated sampler allocator/copy reuse component;
2. use lightweight same-session markers/counters around naturally occurring heavy states;
3. narrow the remaining descriptor/root-binding or scene-state cost without adding tens of thousands of hooked calls per frame;
4. keep FACT / INFERENCE / HYPOTHESIS separate.

## Evidence categories

Keep these separate:

```text
community reports:
1.58 good -> 1.59/1.60 bad

our runtime evidence:
1.60 reproduced and profiled
sampler allocator/copy pressure strongly reduced safely
heavy-state regression not fully fixed

our static evidence:
1.58 vs 1.60 binary/code comparison
fixed-profile root-signature architecture introduced in mapped 1.60 path
```
