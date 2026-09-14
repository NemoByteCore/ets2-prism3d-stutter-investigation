# Function map

This page tracks the current working identifications for high-interest Prism3D functions. Addresses are **build-specific** and must never be reused across versions without re-identification.

## 1.60.1.7s

| Address | Working name | Confidence | Why it matters |
|---|---|---|---|
| `0x1402D7D70` | `r_device_t::resource_build_bundle(...)` | High | Core per-draw/per-bundle path: uniform work, resource resolution and bundle construction before descriptor submission. |
| `0x1402E25A0` | semantic/resource resolver | High | Resolves a semantic/resource ID to an `r_resource_packet_entry_t`; repeated lookup inside one bundle may be avoidable work. |
| `0x14144C770` | uniform/resource context setup + provider-cache check | Medium/High | Small, very hot function. Uses a compact cache key and conditionally rebuilds provider/cache state. High sample count does not imply high per-call cost. |
| `0x1402942D0` | descriptor update/build stage | Medium | Downstream consumer of built resource state; feeds DX12 descriptor construction/update. |
| `0x14029E1F0` | draw/state/root-table submission stage | Medium | Downstream state/root-table/draw path reached after descriptor work. |
| `0x14022E380` | material lookup helper | Medium | Small helper; currently not considered a primary bottleneck. |
| `0x1401DB850` | `pp_batch_data_t` index/helper | Medium | Small helper; currently not considered a primary bottleneck. |

### Current chain

```text
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
state / root tables / draw submission
```

## 1.58.1.4s

Static analysis is being built now. Equivalent functions have **not yet been mapped**, so no 1.58 addresses are published here until they are identified with evidence.

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
