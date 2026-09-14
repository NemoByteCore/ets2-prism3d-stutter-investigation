# `1.60.1.7s:0x14144C770` — uniform/resource context setup

**Working name:** uniform/resource context setup + provider/cache check  
**Confidence:** Medium/High  
**Status:** hot helper, but not currently treated as the main bottleneck

> This file contains an intentionally normalized reconstruction for research. It is not raw decompiler output.

## FACT

The function is small (~421 bytes in the current corpus) and appears to prepare a compact context for the uniform/resource path.

A key is formed from two 16-bit values:

```text
key =
    rendergraph_context[+0xD0]
  | item_internal[+0x142] << 16
```

That value is compared with cached state around:

```text
item_internal[+0x11C]
```

When the key matches, the heavier provider/cache rebuild path is skipped.

The function leads into the broader resource-bundle construction path around `0x1402D7D70`.

## Normalized pseudocode

```cpp
prepare_uniform_resource_context(rg_context, item)
{
    key = combine_u16(
        rg_context.field_D0,
        item.internal.field_142
    );

    if (item.internal.cached_key != key)
    {
        item.internal.cached_key = key;
        rebuild_provider_cache_or_list(...);
    }

    return prepared_context(...);
}
```

## INFERENCE

A high sample count in this function can be explained by very high call frequency. That does **not** by itself show that each invocation is expensive.

The built-in key comparison already demonstrates that Prism3D avoids at least some repeated provider/cache setup when state is unchanged.

## HYPOTHESIS

The interesting question is not simply “how often is this function called?” but:

- how often does the cache key actually change?
- how often does the rebuild branch execute in light vs heavy scenes?
- how much wall-time belongs to the rebuild branch versus the fast path?

## Next measurement

Count:

- calls/s
- `cached_key == new_key`
- `cached_key != new_key`
- wall-time for fast path
- wall-time when rebuild occurs

Until that is measured, this function should not be described as the descriptor/resource builder itself.
