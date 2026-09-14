# Pseudocode notes

This directory contains **research-oriented normalized reconstructions**, not bulk decompiler output.

The goal is to make important Prism3D code paths understandable to other researchers without publishing giant raw Ghidra dumps.

Each note should separate:

- **FACT** — directly supported by static/runtime evidence
- **INFERENCE** — the current interpretation of those facts
- **HYPOTHESIS** — an idea that still needs measurement or testing

Addresses are always version-specific. Use the format:

```text
<build>:0x<address>
```

For example:

```text
1.60.1.7s:0x1402D7D70
```

Do not assume the same address or function layout in another ETS2 build.

## Current directories

- [`1.60/`](1.60/) — mapped high-interest functions from the actively profiled runtime build
- [`1.58/`](1.58/) — static-only pre-regression reference; mappings will be added as the binary comparison progresses

See [`../docs/function-map.md`](../docs/function-map.md) for the current cross-reference table.
