# Render / resource call chain

Current working chain for ETS2 `1.60.1.7s`:

```text
r_item
  ↓
1.60:FUN_14144C770
  ↓
1.60:FUN_1402D7D70
  ↓
r_resource_bundle_t
  ↓
1.60:FUN_1402942D0
  ↓
DX12 descriptor update/build
  ↓
1.60:FUN_14029E1F0
  ↓
state / root tables / draw submission
```

## `1.60:FUN_14144C770`

Small (~421-byte) helper/context builder. It conditionally rebuilds a provider/cache list when a composite key changes.

```text
key =
    rendergraph_context[+0xD0]
  | item_internal[+0x142] << 16
```

Compared with a cached value around `item_internal[+0x11C]`.

This function should no longer be described as the main resource builder.

## `1.60:FUN_1402D7D70`

Current identification:

```cpp
r_device_t::resource_build_bundle(
    r_resource_bundle_t *,
    rendergraph_context_t *,
    const r_item_t &
)
```

This is currently the primary structural focus for resource/uniform work.

## `1.60:FUN_1402E25A0`

Current identification: semantic/resource resolver used during resource bundle construction.

It resolves semantic/resource IDs to `r_resource_packet_entry_t` objects and may scan active resource packets when a simple hit is unavailable.

Potential future optimization target: per-bundle semantic memoization, but only after measuring repeat frequency.

## Downstream descriptor path

`1.60:FUN_1402942D0` feeds descriptor update/build work, followed by `1.60:FUN_14029E1F0` and state/root-table/draw submission.

A separate sampler-descriptor optimization already proved that some descriptor work in this region is massively redundant.
