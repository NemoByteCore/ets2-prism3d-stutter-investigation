# Contributing

This repository is an evidence-driven reverse-engineering investigation. Contributions are welcome, especially when they help identify version differences or narrow the CPU-side stutter path.

## Good contributions

Useful reports usually contain:

- exact ETS2 build
- executable hash when relevant
- function address(es) with the build clearly stated
- how the function was identified
- callers/callees, strings, types or dataflow that support the identification
- whether the claim is a FACT, INFERENCE or HYPOTHESIS
- confidence level
- what the finding changes or what should be tested next

For runtime measurements, include the relevant renderer/settings and distinguish measured values from subjective impressions.

## Current priority

The main static comparison is between:

- `1.58.1.4s` — static-only pre-regression reference
- `1.60.1.7s` — current runtime/reverse-engineering target

The highest-priority 1.60 functions are tracked in [`docs/function-map.md`](docs/function-map.md).

If you identify a 1.58 counterpart, please include enough evidence that another person can independently verify the mapping.

## Please avoid

- uploading SCS executables, assets or other proprietary game files
- bulk raw decompiler dumps
- assuming an address from one build refers to the same function in another build
- presenting speculation as a confirmed cause
- repeating already-closed hypotheses without new evidence
- generic optimization advice unrelated to the measured path

## Pseudocode

Pseudocode in this repository is intentionally normalized and structural. It should explain the behavior relevant to the investigation without reproducing large raw decompiler listings.

When adding or correcting pseudocode, keep these sections separate where applicable:

```text
FACT
INFERENCE
HYPOTHESIS
```

## Issues

For a new finding, open an issue with a concise title such as:

```text
[1.58] Possible counterpart of 1.60:0x1402E25A0
```

or:

```text
[1.60] Runtime timing for resource_build_bundle
```

Include the minimum evidence needed to reproduce or challenge the result. Corrections are welcome; the working function names in this repository are research labels unless explicitly stated otherwise.