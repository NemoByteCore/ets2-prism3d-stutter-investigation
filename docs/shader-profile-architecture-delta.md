# 1.58 → 1.60 shader-profile / resource-layout architecture delta

This page documents a static architecture change found while comparing ETS2 1.58.1.4s with 1.60.1.7s.

The goal is not to claim a root cause prematurely, but to record the strongest regression-shaped structural delta found so far.

## Summary

The 1.60 rendering path introduces a broader shader-profile/resource-layout model that is not present in the mapped 1.58 flow.

```text
RFX pass
  ↓
shader profile
  ↓
resource bucketization
  ↓
cross-stage layout merge
  ↓
expanded uniform-builder representation
  ↓
profiled DX12 root signature
  ↓
generalized descriptor/root-table mapping
  ↓
draw submission
```

The profile table has now been extracted and decoded. This makes the 1.58/1.60 contrast more concrete: 1.58 derives a root signature from the actual pipeline layout, whereas 1.60 selects one of 13 fixed root-signature profiles.

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

`1.60.1.7s:0x140293700` consumes 5-byte profile entries.

Current decode:

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

The one observed `0xFF` count is the pixel SRV range in `lightpass`; it is serialized as an unbounded range while the internal per-draw bookkeeping uses 16 as its working count.

## Profile capacities

| Profile | Root params | Root CBVs | Resource slots | Sampler slots | Table root params |
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

`material`, `lightpass`, and `fullscreen` are the most expanded common graphics profiles in this table.

## 1.60 uses fixed profile capacity during descriptor allocation

`1.60.1.7s:0x1402942D0` resolves the selected profile through `r_shader_pipeline_t + 0x8A`, then uses the profile root-signature totals when reserving resource and sampler descriptor-heap slots.

The descriptor-heap allocator is `1.60.1.7s:0x14028F070`.

Therefore the reservation size is profile-capacity driven. For example, the `material` profile carries capacity for 20 resource descriptors and 20 sampler descriptors.

This does **not** mean every reserved slot is necessarily written every draw. That distinction needs runtime measurement.

## The mapped 1.58 model was layout-specific

`1.58.1.4s:0x1401ABC10` lazily builds a DX12 root signature from the actual `r_shader_pipeline_layout_t`.

The mapped 1.58 path derives descriptor ranges from real resource bindings and aggregates them into a compact model:

```text
CBV/SRV/UAV ranges -> one resource descriptor table
sampler ranges     -> one sampler descriptor table
```

The mapped 1.58 descriptor builder (`0x1401AC780`) uses those layout-specific root-signature counts when reserving descriptor heap space.

The allocator itself maps closely to the 1.60 allocator (`1.58:0x1401A6A20` ↔ `1.60:0x14028F070`), so the important difference is where the requested counts come from.

## Root binding also changed architecture

Mapped draw submission:

```text
1.58.1.4s:0x1401B4800
1.60.1.7s:0x14029E1F0
```

The mapped 1.58 path conditionally binds a compact resource table and sampler table.

The 1.60 path instead:

- selects one of 13 fixed profile root signatures,
- uses root CBVs as separate root parameters,
- tracks/caches multiple root-parameter values,
- iterates per-set descriptor-table metadata,
- handles SRV/UAV/sampler tables independently,
- conditionally binds the resulting table handles.

Common 1.60 profiles such as `material`, `fullscreen`, and `lightpass` contain four descriptor-table root parameters, plus multiple root CBVs.

## Current interpretation

The strongest static regression-shaped change is now more specific than “larger descriptor structures”:

```text
1.58:
actual pipeline layout
→ layout-specific root signature
→ exact descriptor capacities
→ compact resource/sampler table model

1.60:
RFX shader profile
→ fixed root-signature profile
→ fixed profile descriptor capacities
→ root CBVs + split per-set tables
→ generalized root-parameter submission
```

This creates two concrete runtime questions:

1. how much descriptor capacity is reserved but not actually written for typical draws,
2. how much additional root-CBV/table checking and binding occurs for the common profiles.

The previously observed ~95–96% redundant sampler descriptor copies make this branch especially worth measuring, but static evidence alone does not prove causality.

## Pipeline-cache/profile identity remains an open audit

The mapped 1.60 pipeline-cache/create function `0x1402E4D50` derives its pre-lookup identity from six shader identities, while the selected profile ID is stored on the created pipeline and later controls root-signature selection.

A cache hit returns before an explicit requested-profile comparison is visible in this function.

This is **not a confirmed bug**. The asset/data model may guarantee that one six-shader tuple can only occur with one profile.

The clean test is a runtime audit of:

```text
six-shader tuple -> observed profile ID
```

and to report only tuples observed with more than one profile.

## Next measurement

The next probe should be profile-aware rather than global. Useful counters include:

- draw count by profile ID,
- resource/sampler descriptor slots reserved by profile,
- actual descriptor writes/copies,
- root-CBV checks and actual root-CBV set calls,
- descriptor-table checks and actual binds,
- root-signature/profile switches,
- conflicting shader-tuple/profile observations.

No behavior patch should be attempted until those rates are known.