# `1.60.1.7s:0x1402942D0` — descriptor update/build stage

**Working name:** `dx12_device_t::shader_pipeline_build_descriptor_update_info(...)`  
**Confidence:** High  
**Status:** directly reconnected to the accepted runtime leaf

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

A diagnostic signature string in the 1.60 executable identifies this as `dx12_device_t::shader_pipeline_build_descriptor_update_info(...)`. The accepted downstream path reaches the corresponding virtual slot at `0x1402D8F6C [vtable+0x260]`.

The known static counterpart is:

```text
1.58.1.4s:0x1401AC780 <-> 1.60.1.7s:0x1402942D0
```

Within the 1.60 implementation, fixed resource and sampler descriptor reservations call the same allocator:

```text
RESOURCE  0x1402943FF -> 0x14028F070
SAMPLER   0x140294438 -> 0x14028F070
```

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

The 1.60 fixed-profile reservation model may multiply descriptor work when heavy scenes carry much larger legal render-item batches. Prior sampler reuse shows that sampler pressure alone is insufficient, so resource reservation and the remaining descriptor-update path must be measured separately.

## Next step

Measure the accepted runtime path narrowly:

- descriptor-parent time by stable queue class;
- resource-reservation versus sampler-reservation allocator cost;
- sampled fixed-profile capacities by queue/profile;
- residual descriptor update/write work after allocator cost.

Only then choose a behavior experiment.
