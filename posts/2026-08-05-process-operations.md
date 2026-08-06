# Understanding `processOperations`

`processOperations` looks like a deposit check followed by six loops.
Formalizing it exposed two less obvious questions: how can the proof track state through every loop, and how much can the final result reveal about where execution stopped?

## The Goal

`processOperations` coordinates the consensus operations contained in a Gloas block.
Its role is not to implement the rules for each operation, but to invoke the appropriate handlers in the required order and pass the resulting Beacon State from one stage to the next.

Before processing any operations, the function checks the block’s legacy deposit list.
Gloas handles deposits through the newer deposit request path: deposits arrive as execution-layer requests and are processed in connection with the parent execution payload.
To pass `processOperations`, a block must have an empty legacy deposit list.
If the list is non-empty, the function rejects the block before any operation handler runs.

When the deposit check passes, the function processes six operation groups in sequence:

1. proposer slashings;
2. attester slashings;
3. attestations;
4. voluntary exits;
5. BLS-to-execution credential changes; and
6. payload attestations.

Each operation in a group is passed to that group’s specialized handler, which validates the operation and applies its corresponding state changes.
A BLS-to-execution credential change, for example, replaces a validator’s older BLS withdrawal credentials with an Ethereum execution address, allowing future withdrawals to be sent there.

The state produced by each group becomes the input to the next.
If a handler invocation fails, the remaining operations in that group and all later groups are not run.
Under the concrete Lean `EStateM` runner, the error result carries the state reached at the point of failure rather than automatically restoring the pre-state.

The goal of the proof is to characterize this orchestration precisely.
It establishes the behavior of the initial deposit check and proves that `processOperations` succeeds exactly when the deposit list is empty and all six operation groups succeed in sequence, ending with the state produced by the payload-attestation stage.

## What the Proof Establishes

The proof establishes two function-level results about Gloas `processOperations`.

First, it proves the exact behavior of the legacy-deposit check.
If the block’s legacy deposit list is non-empty, `processOperations` immediately returns an assertion error with the pre-state, before any operation handler runs:

```
body.deposits.size ≠ 0
→ processOperations returns .error (.assert _) pre
```

It follows that `processOperations` can succeed only when the legacy deposit list is empty.

Second, the proof characterizes successful orchestration.
`processOperations` succeeds if and only if the deposit list is empty and all six operation family loops succeed in the required order, with each loop receiving the state produced by the preceding loop.

The forward direction exposes five intermediate states, one after each of the first five operation families, connecting the original pre-state to the post-state produced by payload attestations.
The reverse direction shows that if this complete sequence succeeds, then `processOperations` returns `.ok () post`.

This characterization also covers the empty list case: if the deposit list and all six operation lists are empty, the function succeeds without changing the state.
If a handler fails, no complete success chain exists, so `processOperations` cannot return successfully.
The equivalence characterizes success, it does not classify the exact error or state returned by every possible handler failure.

The result is generic over `[Preset]`, `[HasherTag]`, `[Config]`, and `[CryptoBackend]` and requires no state well-formedness assumption.
These parameters supply the container limits, state representation, protocol configuration, and cryptographic implementation required by the concrete handlers, but the sequencing proof assumes no particular values or cryptographic behavior.

Because the handlers are treated as opaque, the proof establishes only the coordinator’s control flow.
It does not establish the validity of individual operations, handler postconditions, protocol invariant preservation, exact error states for handler failures, or complete `processBlock` correctness.

## How I Structured the Proof

I structured this as a proof of concrete execution rather than protocol correctness.
Although `processOperations` is polymorphic over its transition monad, the theorems specialize it to the `EStateM StateTransitionError State` runner used by the Gloas interface.
This makes the exact deposit gate error, successful result, intermediate states, and successful state threading visible in the theorem statements.

I introduced `ProcessOperationsRun` as a transparent name for this runner and `processOperationsForM` as a transparent name for processing one operation list from left to right through its handler.

The source function uses Lean `for` loops over `SSZList`, a size-limited collection used for Ethereum consensus data and Simple Serialize (SSZ) encoding.
An `SSZList` preserves a definite operation order and cannot exceed its protocol-defined capacity.

Rather than assuming how these loops execute, I proved a bridge between each source-level loop and `ForM.forM` over the `SSZList`’s underlying array, which Lean represents definitionally using `Array.foldlM`.
This fold processes items from left to right, passing each resulting state to the next handler and stopping when an error occurs.
The bridge connects the theorem directly to the source implementation.

The first public theorem, `processOperations_nonempty_deposits_error`, handles the opening deposit assertion.
It is deliberately one-directional: non-empty deposits imply an immediate assertion failure with the pre-state preserved.

A converse would be misleading.
When deposits are empty, an opaque handler could theoretically return an `.assert` error without changing the state.
Observing that result alone would not prove that the opening deposit check caused the failure.

The second public theorem, `processOperations_run_ok_iff`, is the main function-level result and is bidirectional.
Private bind decomposition lemmas split a successful sequence into two parts: the first action succeeds with an intermediate state, and the remaining actions succeed from that state.
Applying these lemmas repeatedly exposes the five intermediate states.
The reverse direction uses the same structure to reconstruct the complete successful run.

The characterization is intentionally sensitive to the coordinator’s implementation.
If the deposit gate, operation families, selected handlers, loop order, or `SSZList` iteration behavior changes, the bridge or sequencing proof should stop compiling instead of continuing to certify an outdated description.
Changes inside an opaque handler may not affect this proof because handler correctness is deliberately outside its scope.

## Takeaway

Proving properties of `processOperations` showed that a coordinator can have a precise formal characterization of its control flow without re-proving the operations it dispatches.
By treating each handler as an opaque action, the proof isolates what the coordinator controls: the deposit check, execution order, propagation of updated states, and conditions for success.

The key structural step was connecting the source-level `SSZList` loops to the left-to-right folds used in the theorem.
This keeps the result tied to the implementation.
The deposit theorem also illustrates an important limitation: an error result does not always identify which part of a computation produced it, so causal claims require more than the output alone.

## Upstream Work

Proof: [etheorem PR #57](https://github.com/etheorem/etheorem/pull/57) — `processOperations` operation-sequencing and state-propagation proofs
