# 1.58 → 1.60 shader-profile / resource-layout architecture delta

> **Scope update — 2026-09-15:** this document describes a high-confidence **local rendering-branch delta**. A later hypothesis-agnostic whole-corpus diff found additional cross-cutting 1.60 changes, especially p3mem ownership/scope machinery. The descriptor/root-binding branch remains real and runtime-relevant, but it is no longer treated as the complete or automatically dominant root-cause theory. See [`global-diff-summary.md`](global-diff-summary.md).

## Summary

The mapped 1.60 rendering path introduces a shader-profile/resource-layout model that differs materially from the mapped 1.58 flow.

```text
1.58:
actual pipeline layout
→ layout-specific root signature
→ layout-specific descriptor capacities
→ compact resource + sampler table model

1.60:
RFX shader profile
→ one of 13 fixed root-signature profiles
→ fixed profile descriptor capacities
→ individual root CBVs + split per-set tables
→ generalized root-parameter submission
```

This is a confirmed architectural difference in the mapped branch. Runtime experiments also confirmed that fixed profile sampler reservation/copy pressure is very large. However, removing roughly 95% of targeted sampler allocation/copy work does not eliminate the broader heavy-scene slowdown.

## Exact 1.60 profile IDs

| ID | Profile |
|---:|---|
| 0 | `compute` |
| 1 | `fullscreen` |
| 2 | `simple0` |
| 3 | `simple1` |
| 4 | `simple2` |
| 5 | `simple3` |
| 6 | `simple4` |
| 7 | `simple1_shared` |
| 8 | `lightpass` |
| 9 | `material` |
| 10 | `material_lite` |
| 11 | `shadow` |
| 12 | `sky` |

The static name table is at `1.60.1.7s:0x141D00BD0`; the root-profile descriptor table is at `1.60.1.7s:0x141D17590`.

## Profile descriptor format

`1.60.1.7s:0x140293700` consumes 5-byte entries:

```text
byte 0: kind
  0 = root CBV
  1 = SRV descriptor table
  2 = UAV descriptor table
  3 = sampler descriptor table

byte 1: visibility
  0 = all
  1 = vertex
  2 = pixel

byte 2: logical set / register space
byte 3: CBV shader register for kind 0
byte 4: descriptor count for table kinds
```

The observed `0xFF` count in the `lightpass` pixel-SRV range is serialized as an unbounded range while internal per-draw bookkeeping uses 16 as its working count.

## Profile capacities

| Profile | Root params | Root CBVs | Resource slots | Sampler slots | Table roots |
|---|---:|---:|---:|---:|---:|
| compute | 5 | 2 | 18 | 8 | 3 |
| fullscreen | 8 | 4 | 24 | 16 | 4 |
| simple0 | 2 | 2 | 0 | 0 | 0 |
| simple1 | 4 | 2 | 1 | 1 | 2 |
| simple2 | 4 | 2 | 2 | 2 | 2 |
| simple3 | 4 | 2 | 3 | 3 | 2 |
| simple4 | 4 | 2 | 4 | 4 | 2 |
| simple1_shared | 3 | 1 | 1 | 1 | 2 |
| lightpass | 8 | 4 | 24 | 18 | 4 |
| material | 11 | 7 | 20 | 20 | 4 |
| material_lite | 6 | 4 | 6 | 6 | 2 |
| shadow | 4 | 1 | 5 | 1 | 3 |
| sky | 5 | 3 | 8 | 8 | 2 |

## Fixed profile capacity drives 1.60 descriptor reservation

`1.60.1.7s:0x1402942D0` resolves the selected profile through pipeline state and uses profile totals when reserving resource and sampler descriptor spans via `1.60.1.7s:0x14028F070`.

For example, the `material` profile carries capacity for 20 resource descriptors and 20 sampler descriptors.

This does **not** mean every reserved slot is written every draw.

Runtime structural measurement found median active-gameplay ratios of roughly:

```text
reserved resource capacity / actual layout demand ~7.62x
reserved sampler capacity  / actual sampler demand ~8.34x
```

Typical sampler values were roughly `18.0 reserved slots/draw` versus `2.12 actual sampler bindings/draw`.

## Mapped 1.58 model

`1.58.1.4s:0x1401ABC10` lazily builds a DX12 root signature from the actual pipeline layout.

The mapped 1.58 path derives descriptor ranges from real resource bindings and aggregates them into a compact model:

```text
CBV/SRV/UAV ranges -> resource descriptor table
sampler ranges     -> sampler descriptor table
```

The mapped 1.58 descriptor builder (`1.58.1.4s:0x1401AC780`) reserves layout-specific counts. The allocator maps closely to `1.60.1.7s:0x14028F070`; the important difference is where requested counts come from.

## Root binding architecture also changed

Mapped draw submission:

```text
1.58.1.4s:0x1401B4800
1.60.1.7s:0x14029E1F0
```

The mapped 1.58 path conditionally binds a compact resource table and sampler table.

The 1.60 path can instead:

- select a fixed profile root signature
- use individual root CBVs
- track multiple cached root-parameter values
- iterate per-set descriptor-table metadata
- handle SRV/UAV/sampler tables independently
- bind multiple table roots

## Runtime result: sampler pressure is real and reducible

`NemoDX12SamplerAllocReuse` safely avoids roughly 95% of targeted sampler allocation/copy pressure in ordinary-play runs with safety counters remaining zero.

This validates the descriptor/sampler branch as a real optimization target.

**Important:** sustained heavy-scene slowdown still occurs with that optimization active. Therefore this branch is a component, not a complete explanation.

## Pipeline tuple/profile audit

A runtime audit observed:

```text
tuple requests          524
unique shader tuples    377
same-profile repeats    147
tuple/profile conflicts 0
```

This is negative evidence against the proposed practical six-shader-tuple/profile collision in the measured run. It does not prove the invariant globally, but the hypothesis is lower priority.

## Current place in the investigation

This branch remains a confirmed architectural change and a validated optimization target. Runtime localization moved much deeper through the rendergraph and then independently reconnected the accepted downstream leaf back to the native-DX12 descriptor builder:

```text
RG_CORE
  -> T1 helper
    -> pass callback
      -> nested winner
        -> RQ_ONE
          -> HEAD_DISPATCH
            -> downstream 0x1402D8D20
              -> [vtable+0x260] at 0x1402D8F6C
                -> 0x1402942D0 descriptor builder
```

Stable queue attribution also shows that the heavy-state multiplier is largely more legal per-item work, not merely fixed work becoming slower. The current question is therefore narrower: how much of that multiplied per-item workload is spent in fixed-profile resource reservation, sampler reservation, or the remaining descriptor-update path?

Sampler pressure is still a validated component rather than a complete explanation; the branch is now an active runtime discriminator rather than a parked static theory.

The public rule remains: static architecture differences and runtime structural ratios are evidence, but causal ranking comes from measured frame-budget ownership.
