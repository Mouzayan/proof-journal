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

1. **An exact characterization of the function’s behavior:** If the builder index is in range, the function updates only that builder’s `withdrawableEpoch` within the builder registry, setting it to the result of the `UInt64` addition. If the index is out of range, the function still succeeds, and every proved builder-registry element read and the registry size remain unchanged.

2. **Generic and configuration-specific no-overflow results:** For arbitrary protocol settings, if the mathematical sum of the current epoch and the withdrawal delay is less than `2^64`, the stored `UInt64` value equals the intended ordinary mathematical sum. For the repository’s paired minimal and mainnet Gloas configurations, the proof establishes that this addition cannot overflow, without requiring an additional epoch, slot, or no-overflow assumption.

`[Preset]` and `[Config]` provide the protocol settings used by the function.
The preset includes values such as the number of slots per epoch, while the configuration provides values such as the builder withdrawal delay.
Because the generic theorem allows these settings to vary, it cannot rule out overflow for every possible configuration without an explicit bound.
The fixed values in the paired minimal and mainnet configurations allow that bound to be proved automatically.
Consequently, the combined function-level results for those configurations require only that the builder index is in range.

## What the Proof Establishes

The Lean proof characterizes exactly how `initiateBuilderExit` affects the builder registry—the part of `BeaconState` that records registered ePBS builders and information about their identities, balances, and lifecycle status.

It covers both possible kinds of builder index:

- If the index identifies an existing builder, the function succeeds and updates only that builder’s `withdrawableEpoch`. The builder’s other fields, every other builder, and the size of the registry remain unchanged.
- If the index is out of range, the function also succeeds. Lean’s total `[i]!` semantics do not throw an error when the requested index does not exist, and every proved total element read and the registry size remain unchanged.

The proof also establishes that:

- the result holds for both cached and uncached state representations;
- the new `withdrawableEpoch` is calculated using the epoch derived from the pre-state, before the registry update;
- for arbitrary protocol settings, the theorem describes the result even when the `UInt64` addition wraps around; and
- for Etheorem’s minimal and mainnet configurations, the addition is proved not to wrap around. Each configuration uses its corresponding preset and protocol constants.

This is a _local characterization_ of one call to `initiateBuilderExit`, not a proof of the complete builder-exit process.
It does not prove that every unrelated `BeaconState` field remains unchanged, that `processBuilderExitRequest` always supplies an in-range index, that the builder eventually receives its withdrawal, or that the entire exit workflow is correct.
