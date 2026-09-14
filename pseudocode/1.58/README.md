# ETS2 1.58.1.4s static reference

This build is used **only for static reverse engineering** as a pre-regression comparison point.

It is not used as a runtime test build in this project.

## Reference executable

```text
Version: 1.58.1.4s
SHA-256: AB9785331BF9970542C61A0108A4E677C9F7C00FD316D4C0F9AB116F6BE6C234
```

## Current status

The dedicated Ghidra project is being analyzed. No 1.58 function addresses are published yet because equivalents of the known 1.60 resource/render functions have not been mapped with sufficient evidence.

The first targets are counterparts of:

- `1.60.1.7s:0x1402D7D70` — `resource_build_bundle`
- `1.60.1.7s:0x1402E25A0` — semantic/resource resolver
- `1.60.1.7s:0x14144C770` — uniform/resource context + cache-key path
- `1.60.1.7s:0x1402942D0` — descriptor update/build region
- `1.60.1.7s:0x14029E1F0` — draw/state/root-table submission region

Once an equivalent is identified, add a dedicated note here and link it from `docs/function-map.md`.
