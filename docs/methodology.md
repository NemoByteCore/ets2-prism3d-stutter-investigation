# Methodology

## Core rules

1. Separate **FACT / INFERENCE / HYPOTHESIS**.
2. Keep ETS2 `1.60.1.7s` runtime work separate from `1.58.1.4s` static analysis.
3. Always include the game version with every address.
4. Measure before patching.
5. A sample in a function does not prove an individual call is expensive.
6. Record negative results and demoted leads so they are not rediscovered repeatedly.
7. Prefer fail-open, reversible experiments.
8. One significant variable per experiment where practical.
9. Do not promote a static delta without runtime frequency/cost evidence.
10. Do not call a measured component the complete root cause if it cannot explain the observed state/reset behavior.

## Runtime methodology

For `1.60.1.7s`:

- normal driving rather than synthetic snapshot-only tests
- continuous rendered-frametime capture
- counters before timing; timing before patching
- read-only instrumentation where practical
- low-overhead aggregation rather than per-call logging in hot paths
- fail-open behavior for patch experiments

### Hard constraint: experiments must fit ordinary gameplay

The active save does not provide arbitrary control over scene composition.

Do not require:

- separate hand-picked light/heavy saves
- exact scene reproduction on demand
- teleporting solely to stage a benchmark

Preferred design:

```text
ordinary gameplay
  ↓
continuous rendered frametime
  ↓
lightweight synchronized counters
  ↓
short windows
  ↓
post-hoc good/heavy classification
```

Natural unload/ferry/teleport transitions are especially useful because they can change the slow state without process restart.

## Important frame telemetry caveat

`SCS frame_start` is not guaranteed to be 1:1 with physically rendered frames. At ~45–50 rendered FPS, the callback can still run around 60 Hz and catch up.

Do not derive rendered-frame milliseconds by dividing fixed-rate samples by `frame_start` count.

Prefer:

- independently measured rendered frametime
- wall time
- calls/s
- synchronized counters
- aggregate or sampled call duration

## Observer-effect rule

`NemoShaderProfileProbe v0.2` demonstrated that broad direct-D3D hooking can create workload proportional to the exact scene complexity being measured.

Therefore future probes should avoid tens of thousands of wrappers per frame when a narrow internal hook/counter can answer the question.

For a new hot-path candidate, the preferred order is:

```text
1. count calls / branch entries
2. check correlation with good vs heavy windows
3. add aggregate or sampled timing only if positive
4. patch only after material cost is demonstrated
```

## Static comparison methodology

`1.58.1.4s` is a static-only reference and is never launched.

The completed global comparison covers the whole exported function/call/string/pseudocode corpus rather than only one selected rendering path.

### Counterpart recovery

The initial alignment can use structural properties such as:

- function order
- byte size
- outgoing-call count
- external API identity

These properties are **search-space scaffolding only**, not semantic proof.

A concrete false pair was found during the global pass even in a small positional gap, so final mapping requires independent evidence from combinations of:

- relocation-insensitive normalized pseudocode
- distinctive strings/types
- validated caller/callee continuity
- strong local or global uniqueness margin
- callgraph recovery for functions moved far from their previous binary position

Unmatched functions are not automatically called new/removed.

### Final global-diff inventory

```text
1.58 functions: 66,820
1.60 functions: 68,834
confirmed counterpart pairs: 58,589
materially changed confirmed pairs: 9,308
strong anchored 1.60-only: 597
strong anchored 1.58-only: 375
ambiguous unmatched regions: 3,370
```

See [`global-diff-summary.md`](global-diff-summary.md).

## Symptom-driven ranking

Static candidates are ranked against runtime constraints, not just code growth.

A strong steady-state candidate should plausibly match several of these observations:

- scene/work scaling
- CPU-side pre-submit location
- sustained rather than one-off cost
- compatibility with good/heavy state changes
- possible change across world rebuild/unload/ferry

Large functions that are editor/load/UI/setup paths are demoted even if their static deltas are dramatic.

## Current runtime discriminator

Current top target:

```text
1.60.1.7s:0x14154AAB0
```

First measure:

- helper calls/window
- queue-set count if safe/read-only
- cheap ownership/refcount-heavy branch entries if identifiable

Only after positive correlation should aggregate/sampled timing be added.

If the measured cost is negligible, demote the candidate rather than deepening the branch to rescue the theory.

## Result template

Every substantial result should record:

```text
Build:
Function/address:
Method:
Result:
FACT:
INFERENCE:
HYPOTHESIS:
Confidence:
Evidence against / uncertainty:
Next discriminator:
```

For runtime probes, record enough synchronized timing/context to compare naturally occurring good and heavy windows.

## Publication / sanitization

Do not commit SCS binaries, proprietary assets, giant raw decompiler dumps, private handoffs or local-only machine data.

Prefer:

- original analysis
- normalized pseudocode
- build-specific mappings
- reproducible methodology
- sanitized aggregate logs/results
- minimal excerpts required to explain a finding