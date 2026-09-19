# ETS2 / Prism3D stutter investigation

Evidence-driven reverse engineering of a scene-dependent CPU-side slowdown in **Euro Truck Simulator 2 1.60.1.7s** using the native DX12 renderer.

> [!IMPORTANT]
> **Project status: temporarily paused.**
>
> Active investigation work is currently suspended. The project is **not abandoned**: the existing findings, measurements, mappings and documentation remain valid project history and will stay available here. Work may resume later from the current checkpoint.

## Current status

The render-side slowdown has been narrowed from the whole frame to one small dispatch path:

```text
main loop
  -> RENDER
    -> RG_CORE 0x14021FE20
      -> T1 helper 0x14021F560
        -> pass callback 0x14021F73C
          -> nested winner 0x1413BB140
            -> RQ_ONE 0x14154C9F0
              -> HEAD_DISPATCH 0x14154CF60
                -> NOSPLIT 0x14154CFA7
                  -> 0x1402D8D20
```

The downstream routine is not simply doing fixed work much more slowly. At matched rendergraph cardinality, heavy windows can carry **several times more render items per downstream call**. Stable queue attribution now shows one queue class carrying about **81% of sampled item work overall**, while also confirming that scene composition can shift growth into other queues.

The accepted downstream path also reaches the native-DX12 descriptor builder through the indirect call at `0x1402D8F6C [vtable+0x260]`. The current narrow experiment measures that descriptor-builder cost by stable queue and separates resource-reservation from sampler-reservation pressure without changing rendering behavior.

## Strongest runtime evidence

Light scenes can hold roughly **16.67 ms / 60 FPS** while heavier scene compositions can sustain roughly **19–25 ms**.

A broad phase split showed two independent growth areas:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

The current render-side leaf is much narrower. At exact matched `order/pass = 156/157`:

```text
                              LOW-COST    HIGH-COST
RQ_ONE parent                  1.855 ms    4.002 ms
HEAD_DISPATCH                  1.850 ms    3.997 ms
HEAD samples                     398         386
HEAD avg qpc                     842        1727
RQ_ONE residual                0.005 ms    0.005 ms
```

Across the full accepted run, `HEAD_DISPATCH` accounts for about **99.84%** of sampled `RQ_ONE` time, while sampled call count can fall as per-call cost rises. This is strong evidence for a per-call slowdown rather than simply more invocations.

The downstream split exposed the mechanism more clearly. At exact matched `order/pass = 168/169`:

```text
                              LOW-COST    HIGH-COST
downstream parent              1.520 ms    5.796 ms
sampled parent calls             488         443
processed render items        29,641      92,128
items / parent                  60.7       208.0
BUNDLE_BUILD                    0.856 ms    3.341 ms
```

The parent gets about `3.6x` more expensive per sampled call while processing about `3.4x` more items. Normalized parent cost per item rises only about **6%**. The per-item `BUNDLE_BUILD = 0x1402D7D70` timing stays roughly flat while its total cost scales with item count.

This means most of the earlier apparent "per-call slowdown" is actually **more render-item work inside each call**, not constant work suddenly taking 3–4x longer.

A follow-up queue-work run strengthened that result: across ordinary windows, downstream cost per rendered frame correlates with average queue item count at about `0.97`. Exact-cardinality examples show item averages such as `76 -> 201`, `72 -> 186`, and `49 -> 160` while sampled queue-call count often falls.

Raw `queue_data_t*` addresses are not stable identities: most sampled calls use transient addresses. Stable indexing from the active render-queue-set owner solved that problem: across ordinary windows, 113,391 identities validated with 0 invalid derivations and 0 bucket overflow. `queue_index 0 / tag 11` carries about 80.7% of item work overall, with total item workload and queue-0 workload correlating at about `0.973`.

This does **not** establish that the extra items are duplicates or invalid geometry, so the project is not pursuing a blind queue cap.

## What has already been ruled down

Runtime evidence has demoted several attractive explanations:

- the measured WAIT helper does not own the sustained slowdown;
- raw rendergraph pass/order count is not sufficient;
- coarse pass-type/work composition is not sufficient;
- type-4 and type-7 helper paths are negligible in the accepted RG_CORE split;
- direct execution of `1.60.1.7s:0x14154AAB0` is far too sparse;
- the proposed direct-call boundary at `1.60.1.7s:0x1413C1470` has no direct callsites;
- `traffic_trajectory_t::update_neighbors_bits` is too sparse to own the sustained frame budget.

See [`docs/disproven-hypotheses.md`](docs/disproven-hypotheses.md).

## Static comparison

The current measured functions have close static counterparts in the 1.58 reference build:

```text
1.60.1.7s:0x14154C9F0  <->  1.58.1.4s:0x1413D7170
1.60.1.7s:0x14154CF60  <->  1.58.1.4s:0x1413D7700
1.60.1.7s:0x1402D8D20  <->  1.58.1.4s:0x1401EC530
```

The whole-corpus static diff remains the map; runtime measurement decides where to recurse.

The accepted downstream path now reconnects directly to the mapped descriptor architecture:

```text
0x1402D8D20
  -> 0x1402D8F6C [vtable+0x260]
    -> native DX12 0x1402942D0
       RESOURCE 0x1402943FF -> 0x14028F070
       SAMPLER  0x140294438 -> 0x14028F070
```

The known descriptor-builder counterpart is `1.58.1.4s:0x1401AC780`.

## Secondary findings kept for later

Two other real branches are intentionally not being opened in parallel:

- `PRE_RENDER_OTHER` contributes roughly +1.5 ms in the accepted broad split;
- `RQ_PREP` is a measured but secondary child of the nested winner.

The DX12 descriptor/root-binding branch is no longer merely parked as a secondary static lead: the accepted runtime leaf now reaches it directly. Prior sampler-table reuse still shows that sampler pressure alone is not the complete explanation.

See [`docs/shader-profile-architecture-delta.md`](docs/shader-profile-architecture-delta.md).

## Read this first

- [`docs/current-findings.md`](docs/current-findings.md) — current technical snapshot
- [`docs/rg-core-runtime-localization.md`](docs/rg-core-runtime-localization.md) — condensed evidence ladder for the render branch
- [`docs/runtime-phase-localization.md`](docs/runtime-phase-localization.md) — broad frame-budget localization
- [`docs/global-diff-summary.md`](docs/global-diff-summary.md) — 1.58 ↔ 1.60 corpus map
- [`docs/methodology.md`](docs/methodology.md) — measurement rules
- [`docs/disproven-hypotheses.md`](docs/disproven-hypotheses.md) — branches not to repeat
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — useful ways to help

## Build policy

Runtime / profiling / patch target:

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

**1.58.1.4s is never run in this investigation.**

## Publication policy

No SCS executables, proprietary assets, giant raw decompiler dumps, private handoffs or local-only data are published here. Public notes contain original analysis, normalized pseudocode, mappings and reproducible measurements.
