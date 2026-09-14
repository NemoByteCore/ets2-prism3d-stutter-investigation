# `1.60.1.7s:0x1402942D0` — descriptor update/build stage

**Working name:** descriptor update/build stage  
**Confidence:** Medium  
**Status:** downstream high-interest path

> This file contains an intentionally normalized reconstruction for research. It is not raw decompiler output.

## FACT

This function is reached downstream of `r_device_t::resource_build_bundle(...)` and sits on the path toward DX12 descriptor construction/update.

Current chain:

```text
0x1402D7D70  resource_build_bundle
  ↓
r_resource_bundle_t
  ↓
0x1402942D0
  ↓
DX12 descriptor update/build
  ↓
0x14029E1F0
```

The current evidence is sufficient to place it in the descriptor-building region, but not yet sufficient to publish a precise recovered signature.

## Normalized pseudocode

```cpp
update_or_build_descriptors(bundle, draw_context)
{
    consume_resolved_bundle_resources(bundle, ...);
    prepare_descriptor_state(...);
    update_descriptor_tables(...);
    pass_state_to_submission_path(...);
}
```

This is a structural model only.

## INFERENCE

The confirmed sampler-descriptor experiment shows that descriptor work in the native DX12 path can contain substantial redundancy. Therefore downstream descriptor construction remains relevant even though the current investigation has moved upstream toward bundle construction and semantic/resource resolution.

## HYPOTHESIS

Potential savings may come from separating truly dynamic descriptor state from repeated/static descriptor content, but no new patch should be attempted here until the 1.58↔1.60 static comparison and exact call/timing measurements are available.

## Next step

Map the 1.58 equivalent and compare:

- caller/callee structure
- descriptor-table setup
- allocation/update behavior
- resource packet inputs
- any new invalidation/rebuild logic introduced after 1.58
