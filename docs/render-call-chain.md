# Render / resource call chains

Current working high-interest chains for ETS2 `1.60.1.7s`.

The descriptor path below remains valid, but the completed whole-corpus diff identified an additional frame-level `render_queue_set_t` ownership/copy path that is now the first runtime-discriminator target.

## Frame render / queue-set path

Strong 1.58/1.60 caller counterpart:

```text
1.58.1.4s:0x141213E40
1.60.1.7s:0x1413C1AE0
```

The two caller bodies are 7503 B in both builds with ~0.980 normalized similarity and 56 outgoing calls in both.

The path contains the same broad render-frame construction flow, including scene queues used for effects such as distortion/sunshaft/no-AA/deferred work.

A preserved loop copies/appends queue-set entries:

```text
frame render construction
  ↓
iterate render_queue_set_t entries
  ↓
1.60.1.7s:0x14154AAB0  queue-set copy/append helper
  ↓
continue frame render construction
```

The corresponding helper is:

```text
1.58.1.4s:0x1413D5830   516 B
1.60.1.7s:0x14154AAB0  1370 B
```

Both perform the same broad queue-set copy/append role. 1.60 adds substantial p3mem-style ownership/refcount machinery.

The caller invokes the corresponding helper once per queue-set element in the relevant path, plus two additional helper calls outside the loop.

Current status: strongest new steady-state static candidate; runtime call rate and cost are not yet measured.

## `r_proto` lazy queue-mask resolution

Mapped pair:

```text
1.58.1.4s:0x141213A20
1.60.1.7s:0x1413C1470
```

1.58 consumes the cached mask directly. 1.60 checks for unresolved sentinel `0xFFFFFFFF`, resolves additional state/type information, writes the mask back and then continues filtering.

Because the result is cached, this path is currently treated as a likely first-use/streaming component rather than guaranteed steady-state overhead.

## Descriptor / resource path

```text
r_item
  ↓
1.60.1.7s:0x14144C770
  ↓
1.60.1.7s:0x1402D7D70
  ↓
r_resource_bundle_t
  ↓
1.60.1.7s:0x1402942D0
  ↓
DX12 descriptor update/build
  ↓
1.60.1.7s:0x14029E1F0
  ↓
state / root tables / draw submission
```

### `1.60.1.7s:0x14144C770`

Small (~421-byte) helper/context builder. It conditionally rebuilds provider/cache state when its compact key changes. The mapped 1.58 counterpart is algorithmically very similar.

### `1.60.1.7s:0x1402D7D70`

Working identification:

```cpp
r_device_t::resource_build_bundle(
    r_resource_bundle_t *,
    rendergraph_context_t *,
    const r_item_t &
)
```

### `1.60.1.7s:0x1402E25A0`

Semantic/resource resolver used during bundle construction. It resolves semantic/resource IDs to resource packet entries and may scan active packets when a simple hit is unavailable.

### Downstream descriptor path

`1.60.1.7s:0x1402942D0` builds descriptor/root-table update state, followed by `1.60.1.7s:0x14029E1F0` and generalized state/root-table/draw submission.

Runtime experiments proved very large fixed-profile sampler reservation/copy redundancy and safely removed roughly 95% of targeted sampler allocation/copy pressure. The heavy-scene slowdown still occurs, so this chain remains relevant but is no longer treated as the complete explanation.

## Broader p3mem context

The 1.60 global diff also maps a broad allocator/scope migration:

```text
1.60.1.7s:0x140117240  p3_alloc path
1.60.1.7s:0x140117400  scope lifetime/free helper
```

This infrastructure reaches both the frame queue-copy helper and active traffic code.

Do not hook these generic helpers globally for performance measurement; their fan-in is too large and observer effect would be difficult to control.