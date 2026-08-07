# Understanding `initiateBuilderExit`

`initiateBuilderExit` looks like a one field state update.
Formalizing it exposed two less obvious questions: what happens when the builder index is out-of-range, and when can the epoch calculation overflow?

## Upstream Work

Proof PR: [etheorem #39](https://github.com/etheorem/etheorem/pull/39) — characterizes builder exit state transitions, out-of-range behavior, and no overflow guarantees.

## The Goal

`initiateBuilderExit` begins a builder’s exit from Gloas ePBS by setting the builder’s `withdrawableEpoch`.
In enshrined Proposer Builder Separation (ePBS), builders construct execution payloads and proposers choose which payload to include in a block.

The function does not withdraw the builder’s funds or create a pending withdrawal.
Instead, it records the earliest epoch at which the builder’s remaining balance may become withdrawable.
This delay keeps the funds reserved while any outstanding builder obligations can be settled.

The function computes:

`withdrawableEpoch := currentEpoch + minBuilderWithdrawabilityDelay`

Both are `UInt64` values.
If their mathematical sum exceeds the largest value that a `UInt64` can represent, the addition wraps around instead of storing the intended result.
`initiateBuilderExit` does not perform an explicit overflow check.

The goal is therefore to establish two separate results:

1. **An exact characterization of the function’s behavior:** If the builder index is in range, the function updates only that builder’s `withdrawableEpoch` within the builder registry, setting it to the result of the `UInt64` addition.
   If the index is out-of-range, the function still succeeds.
   Every builder registry lookup covered by the theorem returns the same value as before, and the registry size remains unchanged.

2. **Generic and configuration specific no overflow results:** For arbitrary protocol settings, if the mathematical sum of the current epoch and the withdrawal delay is less than `2^64`, the stored `UInt64` value equals the intended ordinary mathematical sum.
   For the repository’s paired minimal and mainnet Gloas configurations, the proof establishes that this addition cannot overflow without requiring an additional epoch, slot, or no overflow assumption.

## What the Proof Establishes

The Lean proof characterizes exactly how `initiateBuilderExit` affects the builder registry—the part of `BeaconState` that records registered ePBS builders and information about their identities, balances, and lifecycle status.

It covers both possible kinds of builder index:

- If the index identifies an existing builder, the function succeeds and updates only that builder’s `withdrawableEpoch`.
  The builder’s other fields, every other builder, and the size of the registry remain unchanged.
- If the index is out-of-range, the function also succeeds.
  Lean’s total `[i]!` semantics do not throw an error when the requested index does not exist.
  Every builder registry lookup covered by the theorem returns the same value as before, and the registry size remains unchanged.

The proof also establishes that:

- the new `withdrawableEpoch` is calculated using the epoch derived from the pre-state, before the registry update;
- for arbitrary protocol settings, the theorem describes the result even when the `UInt64` addition wraps around; and
- for Etheorem’s minimal and mainnet configurations, the addition is proved not to wrap around.

This is a _local characterization_ of one call to `initiateBuilderExit`, not a proof of the complete builder exit process.
It does not prove that every unrelated `BeaconState` field remains unchanged, that `processBuilderExitRequest` always supplies an in range index, that the builder eventually receives its withdrawal, or that the entire exit workflow is correct.

## How I Structured the Proof

I split the proof into three layers.

**State transition layer.** The first layer runs `initiateBuilderExit` in the concrete `EStateM StateTransitionError State` monad, the execution wrapper that carries the current state and records whether execution succeeds or returns an error.
It handles in range and out-of-range builder indices separately.

For an in range index, the proof shows that the selected builder’s post-state `withdrawableEpoch` is the `UInt64` sum of the epoch derived from the pre-state and the configured withdrawal delay.
For an out-of-range index, it proves _observational preservation_: internal cache bookkeeping may differ, but every builder registry lookup covered by the theorem, as well as the registry size, agrees with the pre-state.
The proof therefore compares values through `sszGet`, which exposes the meaningful stored values, rather than comparing the raw state representations.

**Arithmetic layer.** The second layer proves that the `UInt64` addition equals the intended `Nat` sum when the mathematical sum is less than `2^64`.
In other words, it treats the epoch and delay as ordinary unbounded natural numbers and requires their sum to fit within a `UInt64`.
Under that condition, the machine number addition cannot wrap around.
A function level corollary then combines this result with the state transition theorem to show that the builder’s stored `withdrawableEpoch` equals the intended mathematical sum.

**Configuration specific layer.** `[Preset]` and `[Config]` provide the protocol settings used by the function.
The preset includes values such as the number of slots per epoch, while the configuration provides values such as the builder withdrawal delay.

An arbitrary `Config` may specify an unsafe withdrawal delay, so the generic result requires the explicit `2^64` bound.
I proved configuration specific corollaries for the repository’s paired minimal and mainnet Gloas configurations.
These proofs establish the bound automatically from the fact that `state.slot` is a `UInt64` and from the fixed preset and configuration values.
The resulting corollaries require only an in range builder index, no additional epoch, slot, or no overflow hypothesis is needed.

## Takeaway

Formalizing this one field update required two notions of correctness: describing the state produced by the implementation and showing that its fixed width arithmetic has the intended mathematical meaning.
Keeping those questions separate produced a theorem that remains accurate for arbitrary configurations while supporting stronger no overflow results for minimal and mainnet.

The out-of-range case required an observational statement because raw state equality is not always the right specification.
Internal caches may differ even when every relevant protocol value remains unchanged, so the proof focuses on observable values through `sszGet`.
