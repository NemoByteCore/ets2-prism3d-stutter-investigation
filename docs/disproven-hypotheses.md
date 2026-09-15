# Disproven / closed / demoted hypotheses

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
- DirectStorage as a newly introduced 1.60 feature

## Lower-priority / demoted, not fully disproven

These may be real code paths or real costs, but current evidence does not support treating them as the main sustained gameplay slowdown:

- six-shader-tuple/profile cache collision — 0 conflicts observed in the measured audit
- TAA/rendergraph growth — heavier added work is mainly behind missing/invalid history-image state
- new SRW-lock candidate — traced to `-map_dump` / I/O-cache functionality
- several large KDOP/vegetation deltas — traced to editor/load/build paths
- traffic-semaphore growth — animated collision-shape initialization
- several large model/unit/UI deltas — setup/configuration paths
- Linux/Vulkan swapchain/image changes — not the current Windows native-DX12 steady-state target

## DirectStorage introduction

The DirectStorage backend, `-nodstorage` handling and relevant factory code are present in both compared builds.

**Conclusion:** DirectStorage is not a credible `1.58 -> 1.60` introduction boundary.

## Defrag

Static RE confirmed genuine defrag-related functions, including `1.60.1.7s:0x14028E320` in the newer DX12 resource allocator/defrag family.

Previous live A/B did not improve frametime.

**Conclusion:** do not treat defrag as the established primary cause. The newer allocator architecture can still be revisited only if activation/frequency evidence justifies it.

## Shader tuple / profile collision

Runtime audit:

```text
tuple requests          524
unique shader tuples    377
same-profile repeats    147
tuple/profile conflicts 0
```

**Conclusion:** negative evidence against the proposed practical collision in the captured run. This does not prove the invariant globally, but it is no longer a leading hypothesis.

## TAA / rendergraph growth

A real 1.58/1.60 TAA/rendergraph delta was inspected. The added heavier branch is mainly reached when TAA history state/image is missing or invalid and performs acquire/storage initialization. 1.58 already contains analogous acquisition under the corresponding condition.

**Conclusion:** currently looks more like history initialization/reallocation than a generic new per-frame cost.

## SRW-lock candidate

A newly observed `AcquireSRWLockExclusive` / `ReleaseSRWLockExclusive` change initially looked regression-shaped.

Caller analysis tied it to `-map_dump` / I/O-cache behavior.

**Conclusion:** reject as an ordinary-gameplay explanation.

## Large KDOP / vegetation candidates

Several visually dramatic static deltas were manually classified as editor undo/redo or asset/object build/load paths.

**Conclusion:** they remain valid changed code but should not be ranked by size alone as steady-state candidates.

## Traffic semaphore delta

The inspected growth is associated with creation/initialization of animated collision shapes.

**Conclusion:** likely load/init work rather than the sustained frame loop.

## Resource descriptor reuse v0.5

Observed redundancy was real:

- ~47.2M hits
- ~9.95M misses
- ~112.2M writes skipped

But the implementation intercepted/buffered enormous numbers of individual writes and became more expensive than the work it removed. Frametime reached ~28–30 ms.

**Conclusion:** do not repeat this implementation. The experiment does *not* prove resource redundancy is unimportant.

## Full descriptor-builder memo v0.2

Observed:

```text
memo_hits   = 0
memo_misses = 35,170,169
```

**Conclusion:** an entire draw/bundle is too dynamic for this form of memoization.

## Resource table reuse v0.3

This experiment did **not** prove there is no useful redundancy. The decoder skipped the correct path because validation of `set_count > root_set_count` was too aggressive.

**Conclusion:** implementation/test invalid for the broader hypothesis; do not cite it as proof that resource-table redundancy is absent.

## Camera switching

The observation that cabin -> third-person can sometimes appear to improve a bad state remains **unresolved**, not disproven.

A marker-enabled run recorded real camera changes but did not reproduce the sustained heavy state.

**Conclusion:** keep camera events as passive context only. Do not build camera-specific instrumentation unless a heavy-state transition is actually aligned with a camera event.