# Function map

Updated: **2026-09-18**

This page is a compact map of functions that still matter to the investigation. Addresses are **build-specific** and must never be reused across versions without re-identification.

Working names are research labels unless explicitly stated otherwise.

## Current 1.60.1.7s runtime chain

| Address | Working name | Current status |
|---|---|---|
| `0x1401C5280` | outer main-loop owner | Broad phase anchor |
| `0x1401C77C0` | main-loop iteration / LOOP | Tracks sustained frame-time increase |
| `0x1401C6CB0` | PACE / frame-clock bookkeeping | Negligible |
| `0x1401D72F0` | rendergraph/present coordinator | Broad render boundary |
| `0x14011F730` | frame-time WAIT helper | Measured; not the sustained owner |
| `0x14021FE20` | RG_CORE | Main measured render-side owner |
| `0x14021F560` | T1 helper | Major RG_CORE child |
| `0x14021F73C` | T1 pass-callback callsite | Owns nearly all sampled T1 variation |
| `0x140226A50` | outer callback thunk | Dispatch only; not substantive work |
| `0x1413BD3F0` | nested target thunk | Jumps to substantive winner |
| `0x1413BB140` | substantive nested winner | Major callback implementation |
| `0x14154C370` | RQ_PREP | Secondary measured winner child |
| `0x14154C9F0` | RQ_ONE | Dominant measured winner child |
| `0x14154CF60` | HEAD_DISPATCH | ~99.84% of sampled RQ_ONE time in accepted run |
| `0x1402D8D20` | downstream dispatch implementation | Accepted item-batch owner |
| `0x1402D8F6C` | indirect descriptor-build callsite | Runtime bridge into native DX12 descriptor builder |
| `0x1402942D0` | `dx12_device_t::shader_pipeline_build_descriptor_update_info(...)` | Current narrow measurement target |
| `0x14028F070` | descriptor heap allocation / cursor advance | Called for resource and sampler reservations inside descriptor builder |

Current path:

```text
RG_CORE
  -> T1 helper
    -> pass callback
      -> nested winner
        -> RQ_ONE
          -> HEAD_DISPATCH
            -> 0x1402D8D20
              -> [vtable+0x260] at 0x1402D8F6C
                -> native DX12 0x1402942D0
```

## Current static counterparts

| 1.60.1.7s | 1.58.1.4s | Notes |
|---|---|---|
| `0x1413BB140` | `0x14120D6B0` | Substantive nested winner |
| `0x14154C9F0` | `0x1413D7170` | RQ_ONE |
| `0x14154CF60` | `0x1413D7700` | HEAD_DISPATCH; both 491 bytes |
| `0x1402D8D20` | `0x1401EC530` | Downstream dispatch target |

The active `HEAD_DISPATCH` callsites are:

```text
NOSPLIT  1.60.1.7s:0x14154CFA7 -> 0x1402D8D20
RANGE    1.60.1.7s:0x14154D048 -> 0x1402D8D20
```

## Runtime-demoted functions kept for reference

| 1.60.1.7s address | Working name | Why demoted |
|---|---|---|
| `0x14154AAB0` | `render_queue_set_t` copy/append helper | 14 direct executions across 24,798 rendered frames |
| `0x1413C1470` | `r_proto` lazy render-queue mask path | No direct `E8` callsites at proposed boundary |
| `0x1408DC510` | `traffic_trajectory_t::update_neighbors_bits` | 85 total calls; absent around sustained heavy-state onset |
| `0x14028E320` | DX12 defrag/resource-pool candidate | Architecture is real; steady-state activation not established |

Important static counterparts retained for those demoted paths:

```text
1.58.1.4s:0x1413D5830 <-> 1.60.1.7s:0x14154AAB0
1.58.1.4s:0x141213A20 <-> 1.60.1.7s:0x1413C1470
1.58.1.4s:0x140815880 <-> 1.60.1.7s:0x1408DC510
```

Demotion changes runtime priority; it does not invalidate the static mapping.

## Descriptor / root-binding branch

This branch is now reconnected directly to the accepted runtime leaf. The fixed-profile architecture remains a validated optimization/regression component; the current narrow question is how much of the accepted downstream cost it owns.

| 1.60.1.7s address | Working name |
|---|---|
| `0x14144C770` | uniform/resource context setup + provider-cache check |
| `0x1402D7D70` | `r_device_t::resource_build_bundle(...)` |
| `0x1402E25A0` | semantic/resource resolver |
| `0x1402942D0` | native DX12 descriptor update/build stage |
| `0x14029E1F0` | generalized state/root-table submission |
| `0x14028F070` | descriptor heap allocation / cursor advance |
| `0x14028EBE0` | shader-visible descriptor heap setup |
| `0x14144C160` | composite uniform-builder merge helper |

Mapped 1.58 counterparts:

```text
1.58.1.4s:0x1401EB530 <-> 1.60.1.7s:0x1402D7D70
1.58.1.4s:0x1401F4FB0 <-> 1.60.1.7s:0x1402E25A0
1.58.1.4s:0x14129BF20 <-> 1.60.1.7s:0x14144C770
1.58.1.4s:0x1401AC780 <-> 1.60.1.7s:0x1402942D0
1.58.1.4s:0x1401B4800 <-> 1.60.1.7s:0x14029E1F0
```

See [`shader-profile-architecture-delta.md`](shader-profile-architecture-delta.md).

## p3mem context

The 1.60 whole-corpus diff identified a broad allocator/scope migration:

```text
1.60.1.7s:0x140117240  p3_alloc path
1.60.1.7s:0x140117400  scope lifetime/free helper
```

These have very large fan-in and are **structural context only**. They are not justified global hook/patch targets.

## Mapping discipline

For important mappings, keep:

- exact build and executable hash;
- build-specific address;
- caller/callee or cross-reference evidence;
- strings/types/dataflow supporting identification;
- whether the name is recovered, inferred or descriptive;
- confidence;
- counterpart when known.

The completed corpus comparison recovered **58,589 confirmed counterpart pairs**, but runtime measurement decides which mappings deserve deeper work.

See [`global-diff-summary.md`](global-diff-summary.md).
