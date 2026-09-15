# Function map

This page tracks current working identifications for high-interest Prism3D functions. Addresses are **build-specific** and must never be reused across versions without re-identification.

Working names are research labels unless explicitly stated otherwise.

## Current 1.60.1.7s runtime-localization map

| Address | Working name | Confidence | Current status |
|---|---|---|---|
| `0x1401C5280` | outer main-loop owner | Medium/High | Calls the measured main-loop iteration boundary. Current coarse runtime-localization anchor. |
| `0x1401C77C0` | main-loop iteration | High | `NemoFramePhaseProbe` LOOP boundary; tracks the sustained heavy-state increase. |
| `0x1401C6CB0` | frame-clock / duration bookkeeping | Medium/High | PACE boundary; measured at ~0.001 ms/iteration and effectively ruled out as the regression owner. |
| `0x1401D72F0` | rendergraph / present coordinator | Medium/High | Broad RENDER boundary. Contains both active work and a nested deliberate frame-time wait. |
| `0x14011F730` | frame-time wait helper | Medium/High | Uses `Sleep()` followed by a short spin phase; separated in `NemoFramePhaseProbe v0.2`. |
| `0x14154AAB0` | `render_queue_set_t` copy / append helper | High | Strong static 1.58 counterpart, but only 14 direct runtime calls across 24,798 rendered frames. Direct-cost theory strongly demoted. |
| `0x1413C1AE0` | render-frame construction function | High | Strong 1.58 counterpart. Important static context, but not treated as a per-frame timing boundary after the direct helper-frequency result. |
| `0x1413C1470` | render-queue / `r_proto` mask filter with lazy resolution | High | No direct `E8` callsites / incoming direct-call edge for the proposed hook boundary. Direct-call experiment closed pending new reachability evidence. |
| `0x140117240` | recovered `p3mem_scope_t::p3_alloc(...)` path | High | Central 1.60 allocator/scope path with ~1,485 static incoming edges. Structural context only; do not hook globally. |
| `0x140117400` | p3mem scope lifetime/free helper | Medium/High | ~3,828 static incoming edges. Structural context only; do not hook globally. |
| `0x1408DC510` | `traffic_trajectory_t::update_neighbors_bits` | High | Active traffic path, but only 85 total calls in the measured run and absent for ~42 s around a natural heavy-state onset. Direct-cost theory strongly demoted. |
| `0x14028E320` | `dx12_pool_t::defragment_data(...)` candidate | Medium/High | 1.60 DX12 resource allocator/TLSF/defrag path; steady-state activation/frequency still unproven. |

## Main-loop phase-localization chain

```text
0x1401C5280  outer loop owner
    -> 0x1401C77C0  LOOP
          -> 0x1401C6CB0  PACE / frame-clock bookkeeping
          -> 0x1401D72F0  RENDER / rendergraph-present coordinator
                -> 0x14011F730  WAIT / frame-time wait helper
```

`NemoFramePhaseProbe v0.1` found two similar natural good→heavy transitions:

```text
transition A: LOOP +4.116 ms, RENDER +1.641 ms, OTHER +2.475 ms
transition B: LOOP +3.745 ms, RENDER +2.007 ms, OTHER +1.738 ms
```

`PACE` stayed around `~0.001 ms/iteration`.

Because `RENDER` includes the nested wait helper, `NemoFramePhaseProbe v0.2` separates `WAIT` and derives `RENDER_ACTIVE = RENDER - WAIT`.

See [`runtime-phase-localization.md`](runtime-phase-localization.md).

## Render-queue counterpart set

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

The corresponding frame-construction function is:

```text
1.58.1.4s:0x141213E40  7503 B
1.60.1.7s:0x1413C1AE0  7503 B
```

Static mapping remains high confidence, but runtime measurement found only 14 direct helper calls across 24,798 rendered frames.

Current status: **strong static delta, strongly demoted sustained direct-cost theory**.

## `r_proto` lazy-resolution counterpart

```text
1.58.1.4s:0x141213A20   869 B
1.60.1.7s:0x1413C1470  1463 B
```

1.58 consumes the cached render-queue mask directly. 1.60 checks for sentinel `0xFFFFFFFF`; when unresolved, it follows additional state, resolves the mask/type, stores the result back and performs ownership cleanup before continuing.

The proposed exact 1.60 boundary has no direct `E8` callsites and no incoming direct-call edge in the harvested graph. That does not disprove indirect/tail/inlined use, but direct-call probing is not repeated without new xref evidence.

## Active traffic p3mem counterpart

```text
1.58.1.4s:0x140815880  500 B
1.60.1.7s:0x1408DC510  774 B
```

Recovered context identifies this family as `traffic_trajectory_t::update_neighbors_bits`. The 1.60 version adds thread-local p3mem scope acquisition/refcount and scope-backed temporary storage.

Runtime follow-up found only 85 total executions in the full run, with 30 in a final shutdown/unload-adjacent burst. A natural heavy-state onset occurred across an approximately 42 s call-free interval.

Current status: **proves p3mem reaches active gameplay code; does not support this function as a sustained direct-cost root cause**.

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
| `0x1402E4D50` | shader-pipeline cache/create path with profile state | Medium/High | Runtime audit observed 0 tuple/profile conflicts in the measured run. |
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
| `0x14129BF20` | `0x14144C770` | Exact 421-byte size and effectively identical context/cache-key algorithm after normalization. | High |
| `0x1401AC780` | `0x1402942D0` | Same recovered descriptor-update role and type fingerprint. | High |
| `0x1401B4800` | `0x14029E1F0` | Same unique draw diagnostic/context and downstream role. | High |
| `0x14014D570` | `0x14022E380` | Exact 90-byte `r_material_t` array helper. | High |
| `0x1400FC430` | `0x1401DB850` | Exact 90-byte `pp_batch_data_t` array helper. | High |

## Global-diff mapping note

The completed whole-corpus pass recovered **58,589 confirmed counterpart pairs** across the two builds. The initial structural skeleton was used only to constrain search space; it was not treated as semantic proof after a concrete false-pair case was found.

Final high-confidence recovery uses normalized pseudocode, distinctive strings/types, validated callgraph continuity and uniqueness/margin checks.

The recent runtime negatives do not invalidate the counterpart map; they change how it is used. The map now follows runtime localization instead of driving isolated target selection by static size/novelty alone.

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