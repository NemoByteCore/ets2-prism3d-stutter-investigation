# Disproven / closed hypotheses

Do not revisit these without new hard evidence, a changed build, or a materially better measurement method.

## Closed as primary root causes

- `g_traffic` A/B
- generic graphics/config tweaks
- texture budget/cache
- DXVK
- online services
- frame fence / DXGI wait
- Prism sleep
- stale instance result reuse
- defrag as the main root cause
- old resource descriptor v0.5 implementation
- full descriptor-builder memo v0.2
- resource-table reuse v0.3 as implemented by the old decoder

## Defrag

Static RE confirmed genuine defrag-related functions:

- `1.60:FUN_1412E3780`
- `1.60:FUN_1402E7AC0`
- `1.60:FUN_14028E320`

Live A/B did not improve frametime.

## Resource descriptor reuse v0.5

Observed redundancy was real:

- ~47.2M hits
- ~9.95M misses
- ~112.2M writes skipped

But the implementation intercepted/buffered enormous numbers of individual writes and became more expensive than the work it removed. Frametime reached ~28–30 ms.

**Conclusion:** do not repeat this implementation. The experiment does *not* prove resource redundancy is unimportant.

## Full descriptor-builder memo v0.2

Observed:

- `memo_hits = 0`
- `memo_misses = 35,170,169`

**Conclusion:** an entire draw/bundle is too dynamic for this form of memoization.

## Resource table reuse v0.3

This experiment did **not** prove there is no useful redundancy. The decoder skipped the correct path because validation of `set_count > root_set_count` was too aggressive.

**Conclusion:** implementation/test invalid for the broader hypothesis; do not cite it as proof that resource-table redundancy is absent.
