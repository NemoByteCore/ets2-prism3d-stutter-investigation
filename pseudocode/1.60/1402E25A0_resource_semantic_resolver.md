# `1.60.1.7s:0x1402E25A0` — semantic/resource resolver

**Working name:** semantic/resource resolver  
**Confidence:** High  
**Status:** promising measurement target

> This file contains an intentionally normalized reconstruction for research. It is not raw decompiler output.

## FACT

The function receives a semantic/resource identifier and resolves a matching `r_resource_packet_entry_t`.

Current static analysis shows two broad behaviors:

- a simple/fast matching path can succeed immediately
- otherwise the function can iterate active resource packets for the current draw and inspect entries until the requested semantic is found

It is called from the resource-bundle construction path around `1.60.1.7s:0x1402D7D70`.

## Normalized pseudocode

```cpp
resolve_resource_semantic(context, semantic_id)
{
    if (fast_entry_matches(context, semantic_id))
        return fast_entry(context);

    for (packet in active_resource_packets(context))
    {
        for (entry in packet.entries)
        {
            if (entry.semantic_id == semantic_id)
                return entry;
        }
    }

    return null;
}
```

The exact container layout and loop structure are intentionally omitted here; the important research behavior is the repeated semantic lookup and fallback packet scan.

## INFERENCE

If one `resource_build_bundle` invocation requests the same semantic more than once, the fallback search work may be repeated unnecessarily.

This matters because the resolver is in a high-frequency per-bundle path, so even a modest cost can become significant when multiplied by many draws/resources in a heavy scene.

## HYPOTHESIS

A low-risk proof of concept is a **local memo with lifetime limited to one `resource_build_bundle` call**:

```cpp
if (local_memo.contains(semantic_id))
    return local_memo[semantic_id];

entry = original_resolver(...);
local_memo[semantic_id] = entry;
return entry;
```

Constraints:

- no cache across frames
- no cache across draws
- first lookup always follows Prism3D's original behavior
- only repeated semantic IDs within the same bundle can bypass a rescan

This is not yet justified as a patch until repeat rate and timing are measured.

## Next measurement

For every `resource_build_bundle` invocation, count:

- resolver calls
- unique semantic IDs
- repeated semantic IDs
- repeat ratio
- total resolver wall-time

A useful result would distinguish light and heavy scenes rather than only reporting a global lifetime total.
