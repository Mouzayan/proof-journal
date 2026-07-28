# Understanding `initiateBuilderExit`

## The Goal

`initiateBuilderExit` begins a builder’s exit from Gloas ePBS by setting the builder’s `withdrawableEpoch`.
In enshrined Proposer-Builder Separation (ePBS), builders construct execution payloads and proposers choose which payload to include in a block.

The function does not withdraw the builder’s funds or create a pending withdrawal.
Instead, it records the earliest epoch at which the builder’s remaining balance may become withdrawable.
This delay keeps the funds reserved while any outstanding builder obligations can be settled.

The function computes:

`withdrawableEpoch := currentEpoch + minBuilderWithdrawabilityDelay`

Both values are `UInt64`s.
If their mathematical sum exceeds the largest value that a `UInt64` can represent, the addition wraps around instead of storing the intended result.
`initiateBuilderExit` does not perform an explicit overflow check.

The goal is therefore to establish two separate results:

1. **An exact characterization of the function’s behavior:** If the builder index is in range, the function updates only that builder’s `withdrawableEpoch`, setting it to the result of the `UInt64` addition. If the index is out of range, the state is unchanged.

2. **A conditional no-overflow result:** If the mathematical sum of the current epoch and the withdrawal delay is less than `2^64`, the stored `UInt64` value equals the intended ordinary mathematical sum.

`Preset` and `Config` represent the protocol settings available to the function, including the builder withdrawal delay.
Because the general theorem allows these settings to vary, it cannot guarantee that the calculation never overflows.
However, overflow can be ruled out for the fixed settings defined by the test-oriented minimal configuration and the mainnet configuration, provided `currentEpoch` remains below a suitable limit.

## What the Proof Establishes
