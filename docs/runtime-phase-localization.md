# Runtime phase localization

Updated: **2026-09-18**

This note records the broad runtime localization that moved the project away from static-candidate roulette.

## Why the project pivoted

Several plausible static candidates failed simple runtime reachability/frequency checks:

```text
1.60.1.7s:0x14154AAB0
  14 direct executions across 24,798 rendered frames

1.60.1.7s:0x1413C1470
  no direct E8 callsites at the proposed boundary

1.60.1.7s:0x1408DC510
  85 total calls, including a sustained heavy-state onset with no calls
```

The lesson was straightforward: first locate the missing frame time, then use static comparison on the measured owner.

## Broad main-loop chain

```text
0x1401C5280  outer loop owner
  -> 0x1401C77C0  main-loop iteration
       -> 0x1401C6CB0  PACE / frame-clock bookkeeping
       -> 0x1401D72F0  RENDER / rendergraph-present coordinator
            -> 0x14011F730  measured WAIT helper
            -> 0x14021FE20  RG_CORE
```

Derived buckets:

```text
RENDER_ACTIVE     = RENDER - WAIT
PRE_RENDER_OTHER  = (RENDER_begin - LOOP_begin) - PACE
POST_RENDER_OTHER = LOOP_end - RENDER_end
```

## Accepted broad budget

Clean ordinary gameplay windows:

```text
             GOOD       HEAVY      DELTA
LOOP         16.683 ms  20.358 ms  +3.675 ms
RENDER_ACTIVE11.404 ms  13.507 ms  +2.103 ms
OTHER         5.253 ms   6.840 ms  +1.586 ms
WAIT          0.024 ms   0.009 ms  -0.015 ms
```

The measured WAIT helper is conditional and sparse. It does **not** explain the heavy-state regression.

A narrower render-child split then produced:

```text
                              RECOVERED   HEAVY      DELTA
LOOP                           16.672 ms   19.735 ms  +3.063 ms
RENDER_ACTIVE                   9.983 ms   11.593 ms  +1.610 ms
PRE_RENDER_OTHER                6.285 ms    7.752 ms  +1.467 ms
POST_RENDER_OTHER               0.381 ms    0.380 ms  ~0
RG_CORE                         6.271 ms    8.964 ms  +2.693 ms
```

Among the selected immediate render children, the positive growth was overwhelmingly localized to:

```text
1.60.1.7s:0x14021FE20  RG_CORE
```

## Accepted conclusions

**FACT:** `PACE` is negligible.

**FACT:** the measured WAIT helper does not own the sustained slowdown.

**FACT:** non-render growth is pre-render, not post-render.

**FACT:** the selected render-side growth is concentrated in `RG_CORE`.

**FACT:** the slowdown therefore has at least two measurable CPU-side components:

```text
PRE_RENDER_OTHER
RG_CORE
```

The current project strategy is to finish the `RG_CORE` branch to a concrete mechanism before opening `PRE_RENDER_OTHER`.

## Rendergraph cardinality caveat

Heavy scenes can contain more rendergraph work, but count alone is not sufficient.

Matched-cardinality example:

```text
                              SMOOTH      HEAVY
RG_CORE                        6.253 ms    9.161 ms
order_count                  192.88      191.16
pass_count                   193.88      192.16
```

The same broad rendergraph cardinality can execute about 3 ms slower.

That result is why the project moved from counting work to timing the paths that execute it.

## Current handoff

The render branch has since been narrowed much further:

```text
RG_CORE
  -> T1 helper
    -> pass callback
      -> nested winner
        -> RQ_ONE
          -> HEAD_DISPATCH
            -> shared downstream 0x1402D8D20
```

See [`rg-core-runtime-localization.md`](rg-core-runtime-localization.md) for the condensed evidence ladder and [`current-findings.md`](current-findings.md) for the current technical snapshot.

## Measurement rule

The broad phase results remain useful as a budget check even as the leaf changes.

A new leaf is only promoted when:

- it explains a material share of the parent delta;
- the measurement remains structurally clean;
- matched-cardinality or otherwise controlled comparisons support the result;
- increased call count alone is not silently substituted for per-call slowdown;
- a narrower discriminator exists before any behavior patch.
