# Function map

This page tracks the current working identifications for high-interest Prism3D functions. Addresses are **build-specific** and must never be reused across versions without re-identification.

## 1.60.1.7s

| Address | Working name | Confidence | Why it matters |
|---|---|---|---|
| `0x1402D7D70` | `r_device_t::resource_build_bundle(...)` | High | Core per-draw/per-bundle path: uniform work, resource resolution and bundle construction before descriptor submission. |
| `0x1402E25A0` | semantic/resource resolver | High | Resolves a semantic/resource ID to an `r_resource_packet_entry_t`; repeated lookup inside one bundle may be avoidable work. |
| `0x14144C770` | uniform/resource context setup + provider-cache check | Medium/High | Small, very hot function. Uses a compact cache key and conditionally rebuilds provider/cache state. High sample count does not imply high per-call cost. |
| `0x1402942D0` | descriptor update/build stage | High | Recovered `shader_pipeline_build_descriptor_update_info` path; builds the per-draw descriptor/root-table state consumed downstream. |
| `0x14029E1F0` | draw/state/root-table submission stage | High | Generalized state/root-table/draw path reached after descriptor work. |
| `0x14022E380` | material lookup helper | High | Small `r_material_t` array helper; not considered a primary bottleneck. |
| `0x1401DB850` | `pp_batch_data_t` index/helper | High | Small array helper; not considered a primary bottleneck. |
| `0x14144CE50` | `uniform_builder_t` array lookup helper | High | 97-byte bounds-checked lookup called from the 1.60 bundle loop; equivalent work is inline in the mapped 1.58 function. |
| `0x14144C160` | composite `uniform_builder_t` merge helper | Medium/High | Combines differing builders while merging/deduplicating callback entries as 1.60 cross-stage resource layouts are assembled. |
| `0x1402E4D50` | shader-pipeline cache/create path with profile state | Medium/High | Cache/create path uses six shader identities and stores the selected shader-profile ID in the created pipeline; exact cache-key/profile invariant is still under investigation. |
| `0x140292620` | pipeline-state lookup/task path | Medium/High | Larger 1.60 path that can enqueue `compile_pipeline_task_t`; runtime significance is not yet established. |

### Current chain

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
0x1402942D0
  ↓
DX12 descriptor update/build
  ↓
0x14029E1F0
  ↓
generalized state / root tables / draw submission
```

## 1.58.1.4s

Corpus-driven static matching identifies the pre-regression counterparts with high confidence:

| 1.58 address | Counterpart in 1.60 | Evidence summary | Confidence |
|---|---|---|---|
| `0x1401EB530` | `0x1402D7D70` | Same recovered `resource_build_bundle` symbol string, same rare type/assert strings, same broad call-chain shape and near-identical normalized structure. | High |
| `0x1401F4FB0` | `0x1402E25A0` | Exact 342-byte size and effectively identical semantic/resource packet scan algorithm after relocation/layout normalization. | High |
| `0x14129BF20` | `0x14144C770` | Exact 421-byte size and effectively identical context/cache-key algorithm. | High |
| `0x1401AC780` | `0x1402942D0` | Same recovered `shader_pipeline_build_descriptor_update_info` symbol string and same descriptor/resource type fingerprint. | High |
| `0x1401B4800` | `0x14029E1F0` | Same unique `draw_set_primitive_type` diagnostic/symbol and same downstream draw role. | High |
| `0x14014D570` | `0x14022E380` | Exact 90-byte `r_material_t` array helper. | High |
| `0x1400FC430` | `0x1401DB850` | Exact 90-byte `pp_batch_data_t` array helper. | High |

### Important static-diff note

The semantic/resource resolver and the small context/cache helper are algorithmically almost unchanged between 1.58 and 1.60. The strongest structural delta is now understood as a broader 1.60 shader-profile/resource-layout architecture:

- RFX passes gain shader-profile selection/validation
- resources can be split across three binding buckets and merged across shader stages
- `uniform_builder_t` grows from `0x30` to `0x40` and 1.60 gains composite-builder merge machinery
- the mapped descriptor/update payload grows substantially
- the per-entry state stride visible in the descriptor builder grows from `0x1E0` to `0x2C0`
- the temporary descriptor/update block grows from `0x18` bytes in 1.58 to `0x98` bytes in 1.60
- 1.60 zero-initializes 16 additional 64-bit table/state slots
- the mapped draw submission function grows from 1388 to 1984 bytes and moves from a compact fixed table path to generalized root-signature set/table traversal with cached root-table state

See `docs/shader-profile-architecture-delta.md` for the current architecture-level interpretation.

These are static facts. Their runtime cost still requires direct measurement before any patch is justified.

Reference executable SHA-256:

```text
AB9785331BF9970542C61A0108A4E677C9F7C00FD316D4C0F9AB116F6BE6C234
```

## Evidence discipline

Each mapping should eventually include:

- build and executable hash
- address
- callers/callees or cross-references used for identification
- relevant strings/types/dataflow
- whether the name is recovered, inferred, or purely descriptive
- confidence level
- version counterpart when known

## Notes

The names in this repository are working names used for research. Unless explicitly stated otherwise, they are not claimed to be official SCS symbol names.
