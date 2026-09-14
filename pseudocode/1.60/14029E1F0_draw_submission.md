# `1.60.1.7s:0x14029E1F0` — draw/state submission stage

**Working name:** draw/state/root-table submission stage  
**Confidence:** Medium  
**Status:** downstream context for the current bottleneck investigation

> This file contains an intentionally normalized reconstruction for research. It is not raw decompiler output.

## FACT

This function is downstream of descriptor update/build and participates in the state/root-table/draw submission region of the native DX12 renderer.

Current chain:

```text
0x1402942D0
  ↓
DX12 descriptor update/build
  ↓
0x14029E1F0
  ↓
state / root tables / draw submission
```

The current evidence does not justify a precise recovered signature, so this repository uses a descriptive working name only.

## Normalized pseudocode

```cpp
submit_draw_state(draw_context, descriptor_state)
{
    apply_root_table_state(...);
    apply_required_draw_state(...);
    issue_or_prepare_draw_submission(...);
}
```

This is a structural model only.

## INFERENCE

Runtime evidence shows the heavy-scene slowdown occurs before Present rather than in DXGI/fence waits. This means the complete CPU path leading into submission matters, but current evidence points further upstream—bundle/resource/uniform work—as a stronger investigation target.

## HYPOTHESIS

Changes between 1.58 and 1.60 in state invalidation, root-table rebuild policy, or the inputs received from descriptor construction could still amplify per-draw CPU work.

## Next step

Identify the 1.58 counterpart and compare both the function itself and its immediate callers/callees before considering any runtime patch here.
