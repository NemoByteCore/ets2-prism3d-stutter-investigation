# Function map

This page tracks current working identifications for high-interest Prism3D functions. Addresses are **build-specific** and must never be reused across versions without re-identification.

Working names are research labels unless explicitly stated otherwise.

## Highest-priority 1.60.1.7s candidates after the global diff

| Address | Working name | Confidence | Why it matters |
|---|---|---|---|
| `0x14154AAB0` | `render_queue_set_t` copy / append helper | High | Strong 1.58 counterpart; called repeatedly from a conserved frame-render construction path; 1.60 adds substantial p3mem-style ownership/refcount machinery. Current top runtime-discriminator target. |
| `0x1413C1AE0` | render-frame construction caller | High | Strong counterpart to `1.58:0x141213E40`; 7503 B in both builds, ~0.980 normalized similarity, 56 outgoing calls in both; loops over queue sets and calls `0x14154AAB0`, plus two additional copies. |
| `0x1413C1470` | render-queue / `r_proto` mask filter with lazy resolution | High | Counterpart to `1.58:0x141213A20`; 1.60 resolves sentinel `0xFFFFFFFF`, caches the result, then continues filtering. Likely first-use/streaming-sensitive. |
| `0x140117240` | recovered `p3mem_scope_t::p3_alloc(...)` path | High | Central 1.60 allocator/scope path with ~1,485 static incoming edges; part of the broad 1.60 p3mem migration. |
| `0x140117400` | p3mem scope lifetime/free helper | Medium/High | ~3,828 static incoming edges; participates in scope ownership/refcount cleanup. Do not hook globally for performance measurement. |
| `0x1408DC510` | `traffic_trajectory_t::update_neighbors_bits` | High | Active traffic path; counterpart to `1.58:0x140815880`; 1.60 gains p3mem scope-backed temporary storage/refcount work. |
| `0x14028E320` | `dx12_pool_t::defragment_data(...)` candidate | Medium/High | 1.60 DX12 resource allocator/TLSF/defrag path; steady-state activation/frequency unknown. |

### New render-queue counterpart set

```text
1.58.1.4s:0x1413D5830   516 B
1.60.1.7s:0x14154AAB0  1370 B
```

Both perform the same broad `render_queue_set_t` copy/append role.

Decompiler-visible sites:

```text
1.58 helper: LOCK 0 / UNLOCK 0
1.60 helper: LOCK 32 / UNLOCK 16
```

These are code sites, **not** executed-per-call counts.

The corresponding frame caller is:

```text
1.58.1.4s:0x141213E40  7503 B
1.60.1.7s:0x1413C1AE0  7503 B
```

The preserved loop invokes the helper once per queue-set element in the relevant path, plus two additional helper calls outside the loop.

Current status: very strong static regression-shaped candidate; runtime cost unmeasured.

### `r_proto` lazy-resolution counterpart

```text
1.58.1.4s:0x141213A20   869 B
1.60.1.7s:0x1413C1470  1463 B
```

1.58 consumes the cached render-queue mask directly. 1.60 checks for sentinel `0xFFFFFFFF`; when unresolved, it follows additional state, resolves the mask/type, stores the result back and performs ownership cleanup before continuing.

Because the result is cached, this is currently ranked as a likely first-use/streaming component rather than a guaranteed every-frame cost.

### Active traffic p3mem counterpart

```text
1.58.1.4s:0x140815880  500 B
1.60.1.7s:0x1408DC510  774 B
```

Recovered context identifies this family as `traffic_trajectory_t::update_neighbors_bits`. The 1.60 version adds thread-local p3mem scope acquisition/refcount and scope-backed temporary storage.

This demonstrates that the p3mem migration reaches active gameplay code; it does not by itself prove material runtime cost.

## Existing 1.60 descriptor / rendering chain

| Address | Working name | Confidence | Why it matters |
|---|---|---|---|
| `0x1402D7D70` | `r_device_t::resource_build_bundle(...)` | High | Core per-draw/per-bundle path before descriptor submission. |
| `0x1402E25A0` | semantic/resource resolver | High | Resolves semantic/resource IDs to resource packet entries. |
| `0x14144C770` | uniform/resource context setup + provider-cache check | Medium/High | Small hot helper; near-identical algorithm to the 1.58 counterpart. |
| `0x1402942D0` | descriptor update/build stage | High | Recovered `shader_pipeline_build_descriptor_update_info` path; uses fixed profile capacities in 1.60. |
| `0x14029E1F0` | draw/state/root-table submission stage | High | Generalized root-CBV/table submission path. |
| `0x14028F070` | descriptor heap allocation / cursor advance | High | Reserves requested descriptor spans; sampler heap pressure is especially important because sampler heap capacity is 2048. |
| `0x14028EBE0` | shader-visible descriptor heap setup | High | Initializes resource heap capacity `0x80000` and sampler heap capacity `0x800`. |
| `0x14144CE50` | `uniform_builder_t` array lookup helper | High | 1.60 out-of-line lookup; equivalent work is inline in mapped 1.58 code. |
| `0x14144C160` | composite `uniform_builder_t` merge helper | Medium/High | Combines differing builders while merging/deduplicating callback entries. |
| `0x1402E4D50` | shader-pipeline cache/create path with profile state | Medium/High | Runtime audit observed 0 tuple/profile conflicts in the measured run; collision hypothesis is lower priority. |
| `0x140292620` | pipeline-state lookup/task path | Medium/High | Larger 1.60 path capable of queueing pipeline compile tasks; runtime significance not established. |
| `0x14022E380` | material lookup helper | High | Small array helper; not considered a primary bottleneck. |
| `0x1401DB850` | `pp_batch_data_t` index/helper | High | Small array helper; not considered a primary bottleneck. |

### Descriptor chain

```text
RFX pass / shader profile
  ↓
resource bucketization + cross-stage merge
  ↓
0x14144C160  composite uniform-builder merge when needed
  ↓
r_item
  ↓
0x14144C770
  ↓
0x1402D7D70  resource_build_bundle
  ↓
r_resource_bundle_t
  ↓
0x1402942D0  descriptor update/build
  ↓
0x14029E1F0  generalized root/table submission
```

## 1.58 descriptor-path counterparts

| 1.58 address | Counterpart in 1.60 | Evidence summary | Confidence |
|---|---|---|---|
| `0x1401EB530` | `0x1402D7D70` | Recovered symbol/type/assert context and near-identical normalized structure. | High |
| `0x1401F4FB0` | `0x1402E25A0` | Exact 342-byte size and effectively identical resource packet scan algorithm after normalization. | High |
| `0x14129BF20` | `0x14144C770` | Exact 421-byte size and effectively identical context/cache-key algorithm. | High |
| `0x1401AC780` | `0x1402942D0` | Same recovered descriptor-update role and type fingerprint. | High |
| `0x1401B4800` | `0x14029E1F0` | Same unique draw diagnostic/context and downstream role. | High |
| `0x14014D570` | `0x14022E380` | Exact 90-byte `r_material_t` array helper. | High |
| `0x1400FC430` | `0x1401DB850` | Exact 90-byte `pp_batch_data_t` array helper. | High |

## Global-diff mapping note

The completed whole-corpus pass recovered **58,589 confirmed counterpart pairs** across the two builds. The initial structural skeleton was used only to constrain search space; it was not treated as semantic proof after a concrete false-pair case was found.

Final high-confidence recovery uses normalized pseudocode, distinctive strings/types, validated callgraph continuity and uniqueness/margin checks.

See [`global-diff-summary.md`](global-diff-summary.md).

## Evidence discipline

For each important mapping, keep:

- exact build and executable hash
- address
- callers/callees or cross-references used for identification
- strings/types/dataflow supporting the identification
- whether the name is recovered, inferred or descriptive
- confidence level
- version counterpart when known

Addresses are never portable across builds without re-identification.