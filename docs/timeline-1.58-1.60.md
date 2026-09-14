# 1.58 → 1.60 regression timeline

## Evidence categories

Keep these separate:

```text
community reports:
1.58 good → 1.59/1.60 bad

our runtime evidence:
1.60 reproduced and profiled

our static evidence:
1.58 vs 1.60 binary/code comparison
```

Community reports are context, not our own runtime A/B measurement.

## Public regression context

Public reports collected during the investigation include:

- Reddit reports of constant stuttering after newer builds
- Steam discussions describing similar regressions
- SCS forum reports around 1.59/1.60

SCS also changed buffer/resource allocation behavior in 1.59-era builds and published fixes described as buffer-management and resource-allocator performance fixes during the beta period.

There were also official comments about a problem stressing the Windows Memory Manager during beta development.

## Why 1.58 matters

`1.58.1.4s` is not a target game version and is not launched in this project.

It is used as a static **pre-regression code snapshot** to compare against `1.60.1.7s`.

Primary question:

> Did the resource/buffer/render path acquire a local code change between 1.58 and 1.59/1.60 that can be bypassed, memoized, disabled, patched, or at least precisely documented?

## Static comparison targets

Initial 1.58 ↔ 1.60 matching focuses on:

- `r_device_t::resource_build_bundle`
- semantic/resource resolver
- uniform evaluation / callbacks
- resource packet handling
- allocator / buffer-management paths
- nearby callers/callees

The 1.58 project, output, names, and mappings must remain separate from 1.60 artifacts.
