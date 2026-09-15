# Render / resource call chains

Updated: **2026-09-16**

Current working high-interest chains for ETS2 `1.60.1.7s`.

## Main-loop timing chain

Current coarse runtime-localization chain:

```text
0x1401C5280  outer loop owner
    ↓
0x1401C77C0  main-loop iteration / LOOP
    ├─> 0x1401C6CB0  frame-clock bookkeeping / PACE
    └─> 0x1401D72F0  rendergraph-present coordinator / RENDER
            └─> 0x14011F730  frame-time wait helper / WAIT
```

`NemoFramePhaseProbe v0.1` found that `PACE` is negligible while the heavy state consistently adds time to both `OTHER = LOOP - PACE - RENDER` and the broad `RENDER` bucket.

Two natural transitions:

```text
transition A: LOOP +4.116 ms, RENDER +1.641 ms, OTHER +2.475 ms
transition B: LOOP +3.745 ms, RENDER +2.007 ms, OTHER +1.738 ms
```

The nested `0x14011F730` wait helper uses `Sleep()` followed by a short spin phase, so `RENDER` is not pure active renderer work. The current v0.2 phase probe separates `WAIT` and derives `RENDER_ACTIVE = RENDER - WAIT`.

See [`runtime-phase-localization.md`](runtime-phase-localization.md).

## Frame render / queue-set path

Strong 1.58/1.60 function counterpart:

```text
1.58.1.4s:0x141213E40
1.60.1.7s:0x1413C1AE0
```

The two bodies are 7503 B in both builds with ~0.980 normalized similarity and 56 outgoing calls in both.

The path contains the same broad render-frame construction flow, including scene queues used for distortion/sunshaft/no-AA/deferred work.

A mapped queue-set copy/append helper is:

```text
1.58.1.4s:0x1413D5830   516 B
1.60.1.7s:0x14154AAB0  1370 B
```

The 1.60 implementation adds substantial p3mem-style ownership/refcount machinery.

### Runtime correction

`NemoRenderQueueProbe v0.2` found only **14 direct executions across 24,798 rendered frames**.

Current status: strong static regression-shaped delta, but the direct helper-cost theory is strongly demoted as an explanation for sustained multi-millisecond frame loss.

## `r_proto` lazy queue-mask resolution

Mapped pair:

```text
1.58.1.4s:0x141213A20
1.60.1.7s:0x1413C1470
```

1.58 consumes the cached mask directly. 1.60 checks for unresolved sentinel `0xFFFFFFFF`, resolves additional state/type information, writes the mask back and then continues filtering.

The proposed exact 1.60 instrumentation boundary has no direct `E8 rel32` callsites and no incoming direct-call edge in the harvested graph.

Current status: direct-call probe closed pending new indirect/tail/xref evidence.

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

Runtime experiments proved very large fixed-profile sampler reservation/copy redundancy and safely removed roughly 95% of targeted sampler allocation/copy pressure. The heavy-scene slowdown still occurs, so this chain remains relevant but is not treated as the complete explanation.

## Broader p3mem context

The 1.60 global diff maps a broad allocator/scope migration:

```text
1.60.1.7s:0x140117240  p3_alloc path
1.60.1.7s:0x140117400  scope lifetime/free helper
```

This infrastructure reaches both rendering and active traffic code.

A path-specific traffic test at `0x1408DC510` proved too sparse to explain sustained per-frame cost. Generic p3mem helpers therefore remain architectural context, not justified global hook/patch targets.

Do not hook these generic helpers globally for performance measurement; their fan-in is too large and observer effect would be difficult to control.