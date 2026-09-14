# 1.58 → 1.60 shader-profile / resource-layout architecture delta

This page documents a static architecture change found while comparing ETS2 1.58.1.4s with 1.60.1.7s.

The goal is not to claim a root cause prematurely, but to record the strongest regression-shaped structural delta found so far.

## Summary

The 1.60 rendering path introduces a broader shader-profile/resource-layout model that is not present in the mapped 1.58 flow.

Current static model:

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

This helps explain why the downstream descriptor/update and root-table structures are much larger in 1.60.

## Confirmed static facts

### RFX pass shader profiles

The 1.60 RFX/pass path contains shader-profile parsing and validation that is absent from the corresponding mapped 1.58 flow.

The 1.60 parser recognizes 13 shader-profile names, including examples such as:

```text
material
simple0
compute
simple1_shared
simple1
simple2
simple3
simple4
fullscreen
material_lite
```

The selected profile ID is carried into the shader-pipeline representation and later selects one of 13 prebuilt DX12 root-signature profiles.

The exact definition of every profile still needs targeted static-data extraction.

### Three-way resource bucketization

The 1.60 resource metadata contains an additional selector that can place a resource into one of three binding buckets/sets.

Pipeline construction merges those bucketed bindings across shader stages and combines stage visibility.

This is a meaningful representation change compared with the simpler mapped 1.58 flow.

### Expanded `uniform_builder_t`

The corresponding builder stride changes from:

```text
1.58: 0x30 = 48 bytes
1.60: 0x40 = 64 bytes
```

1.60 also contains composite-builder machinery that can merge two differing builders and deduplicate callback entries.

This does not by itself prove that more callbacks execute per draw; it may primarily reflect new layout/setup requirements.

### Downstream descriptor/root-table state expansion

Previously mapped static deltas now fit this broader architecture:

```text
per-entry descriptor/update state:
1.58 = 0x1E0
1.60 = 0x2C0

temporary descriptor/update block:
1.58 = 0x18 bytes
1.60 = 0x98 bytes
```

The 1.60 block carries substantially more root-table/state information.

### Generalized root-table submission

The mapped 1.58 draw path uses a comparatively compact fixed descriptor-table flow.

The mapped 1.60 path instead:

- keeps a larger root-table cache,
- clears cached table state when root signature changes,
- iterates root-signature set descriptions,
- handles multiple descriptor-table classes per set,
- compares cached handles before binding.

That is an architectural broadening rather than only a structure-layout shift.

## Current interpretation

The semantic/resource resolver and the small context/cache helper are almost unchanged between 1.58 and 1.60, so they are currently weaker candidates for explaining the version regression.

The stronger structural candidate is the newer chain:

```text
shader-profile selection
→ resource bucketization/merge
→ profiled root signatures
→ generalized descriptor/root-table handling
```

This is still a static result. It does **not** prove that the new architecture accounts for the observed heavy-scene frametime increase.

## Next static work

Before designing the next runtime probe, the priority is to extract the complete 13-profile table and determine:

- profile name → ID mapping,
- root-signature/set/table layout for each profile,
- which profiles are significantly more complex,
- how profile selection is constrained relative to shader-combination caching.

Once that is known, runtime instrumentation can be made profile-aware rather than measuring only global descriptor-builder cost.
