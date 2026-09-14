# `1.60.1.7s:0x1402D7D70` — resource bundle construction

**Working name:** `r_device_t::resource_build_bundle(...)`  
**Confidence:** High  
**Status:** high-priority investigation target

> This file contains an intentionally normalized reconstruction for research. It is not raw decompiler output.

## FACT

Current static analysis identifies this function as the core resource-bundle construction path reached for rendered items before downstream descriptor/state submission.

Observed responsibilities include:

- consuming an `r_item_t`-related context
- building/evaluating uniform state
- resolving resources used by the bundle
- producing/updating an `r_resource_bundle_t`
- feeding downstream descriptor construction/update

The current surrounding chain is:

```text
r_item
  ↓
0x14144C770
  ↓
0x1402D7D70
  ↓
r_resource_bundle_t
  ↓
0x1402942D0
  ↓
DX12 descriptor update/build
```

## Normalized pseudocode

```cpp
resource_build_bundle(bundle, rendergraph_context, item)
{
    prepare_bundle_context(bundle, rendergraph_context, item);

    evaluate_required_uniforms(...);

    for (each required resource semantic)
    {
        entry = resolve_resource_semantic(...);
        attach_resolved_resource(bundle, entry, ...);
    }

    finalize_bundle_state(bundle, ...);
}
```

This is deliberately structural. It describes the behavior relevant to the investigation without reproducing the full decompiler listing.

## INFERENCE

Because heavy scenes reach submission/present late while fence/DXGI waits largely disappear, very frequent work in this per-draw/per-bundle region is a plausible contributor to CPU-side frame time.

`0x1402D7D70` is currently more interesting than the much smaller `0x14144C770` helper because it contains the broader uniform/resource-building work rather than only a compact cache check/context setup.

## HYPOTHESES TO TEST

1. The same semantic/resource IDs may be resolved repeatedly inside one bundle construction.
2. A local per-call memo could remove repeated scans while keeping cache lifetime narrow enough to avoid stale state.
3. Some static portion of bundle construction may be reusable per material/effect/pipeline while object/view-dependent uniforms stay dynamic.

These are hypotheses, not confirmed optimizations.

## Next measurement

Instrument this exact function for:

- calls/s
- total wall-time/s
- average call duration
- p95 call duration
- resource-resolver calls per bundle
- uniform callbacks per bundle

Only after measuring those should a patch be attempted.
