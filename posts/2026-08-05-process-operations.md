# Understanding `processOperations`

## The Goal

`processOperations` coordinates the processing of the consensus operations contained in a Gloas block.
Its role is not to implement the rules for each operation itself, but to invoke the appropriate handlers in the order required by the protocol and pass the resulting Beacon State from one stage to the next.

Before processing any operations, the function checks the block’s legacy deposit list.
Gloas handles deposits through the newer deposit-request path: deposits arrive as execution-layer requests and are processed in connection with the parent execution payload.
That work belongs to another stage of the broader block-processing pipeline, rather than to `processOperations`.
A block reaching this function must therefore have an empty legacy deposit list.
If the list is non-empty, the function rejects the block immediately, before any consensus operation can modify the state.

When the deposit check passes, the function processes six groups of operations in sequence:

1. proposer slashings;
2. attester slashings;
3. attestations;
4. voluntary exits;
5. BLS-to-execution credential changes; and
6. payload attestations.

Each group is delegated to a specialized handler responsible for validating its operations and applying the corresponding state changes.
A BLS-to-execution credential change, for example, replaces a validator’s older BLS-based withdrawal credentials with an Ethereum execution address, allowing future withdrawals to be sent to that address.

The state produced by each group becomes the input to the next.
If any handler fails, processing stops and the remaining groups are not run.
State changes made by earlier handlers are not automatically undone: the concrete Lean `EStateM` runner carries the current state forward and includes the state reached at the point of failure in its error result.
The proof confirms this failure-state behavior for `processOperations` under the concrete Lean `EStateM` runner, but makes no broader claim about rollback behavior elsewhere.

The goal of the proof is to characterize this orchestration precisely.
The proof establishes that a non-empty legacy deposit list always causes immediate rejection.
It then shows that every successful call corresponds to all six operation groups completing successfully in the specified order, with each handler receiving the state produced by the preceding stage.
Conversely, if the deposit list is empty and the six handlers succeed in sequence, `processOperations` succeeds and returns the state produced by the final payload-attestation stage.

The proof therefore captures the function’s control flow directly: the initial deposit check, the required order of execution, the propagation of intermediate states, the point at which processing stops after a failure, and the conditions under which the complete operation-processing pipeline succeeds.

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
`processOperations` succeeds if and only if the deposit list is empty and all six operation-family loops succeed in the required order, with each loop receiving the state produced by the preceding loop.

The forward direction shows that every successful run has the following structure:

- The deposit list was empty.
- Proposer slashings succeeded from the pre-state.
- Attester slashings succeeded from the resulting state.
- Attestations succeeded next.
- Voluntary exits succeeded next.
- BLS-to-execution changes succeeded next.
- Payload attestations succeeded last and produced the final post-state.

The reverse direction shows that if the deposit list is empty and this sequence of loops succeeds, then `processOperations` returns `.ok () post`.

Five intermediate states, one after each of the first five operation families, make the ordering and state threading explicit.
If all six operation lists are empty, their folds succeed without changing the state.
If any handler invocation in the actual sequence fails, no complete success chain exists, so `processOperations` cannot return successfully.
The public theorems do not characterize the exact error or state returned by a later handler failure.
Under `EStateM`, state changes made before a failure are not automatically rolled back.

The result is generic over `[CryptoBackend]` and `[Config]` and requires no state well-formedness assumption.
The `[CryptoBackend]` and `[Config]` parameters provide the cryptographic implementation and protocol configuration required by the handlers, but the proof assumes no particular values or cryptographic behavior.

Because the handlers are treated as opaque, the proof establishes only the coordinator’s control flow.
It does not establish the validity of individual operations, handler postconditions, protocol-invariant preservation, exact error states for handler failures, or complete `processBlock` correctness.

## How I Structured the Proof

I treated this as an exact-runner proof rather than a protocol-correctness proof.
It describes how the Lean function executes: when it succeeds, how it handles the deposit check, and how it passes state between operation groups.
It does not prove the correctness of Ethereum’s broader protocol rules or the individual operations.

Although `processOperations` is polymorphic over its transition monad and can work with different state-and-error wrappers, the theorems specialize it to the concrete `EStateM StateTransitionError State` runner used by the Gloas interface.
This specialization allows the proof to describe exact `.ok` and `.error` results, intermediate states, and short-circuit behavior.

`EStateM StateTransitionError State` carries a Beacon State through the computation.
It can either succeed with an updated state or fail with a state-transition error, recording the state reached when the error occurred.

I introduced `ProcessOperationsRun` as a transparent name for this concrete runner.
I also introduced `processOperationsForM` as a transparent name for processing one operation list from left to right through its handler.

The source function uses Lean for loops over `SSZList`.
An `SSZList` is a size-limited collection used for Ethereum consensus data and Simple Serialize (SSZ) encoding.
It preserves a definite operation order and cannot exceed its protocol-defined capacity.

Rather than assuming how these loops execute, I proved a bridge between each source-level loop and `ForM.forM` over the `SSZList`’s underlying array, which Lean represents definitionally using `Array.foldlM`.
`ForM.forM` processes the items from left to right: it runs the handler on the first item, passes the resulting state to the next, and stops if an error occurs.
The bridge therefore connects the theorem’s folds directly to the loops in the source implementation.

The first public theorem, `processOperations_nonempty_deposits_error`, handles the opening deposit assertion.
It is deliberately one-directional: non-empty deposits imply an immediate assertion failure with the pre-state preserved.

A converse would be misleading.
When deposits are empty, an opaque operation handler could theoretically return an `.assert` error without changing the state.
Observing that result alone would not prove that the opening deposit check caused the failure.

The second public theorem, `processOperations_run_ok_iff`, is the main function-level result and is bidirectional.
Private bind-decomposition lemmas split a successful sequence into two parts: the first action succeeds with an intermediate state, and the remaining actions succeed from that state.
Applying these lemmas repeatedly exposes the five intermediate states.
The reverse direction uses the same structure to reconstruct the complete successful run.

`[Config]` and `[CryptoBackend]` are not assumptions about the coordinator’s correctness.
They remain in the theorem because the concrete handlers require them in order for the function to be defined.
The proof assumes no particular configuration values or cryptographic behavior, nor does it rely on the correctness of native cryptographic code reached through the Foreign Function Interface (FFI), cache equivalence, or the handlers themselves.

I did not add a theorem classifying every possible failure inside an operation handler.
The immediate deposit-rejection theorem and the successful-run equivalence cover the coordinator-level scope.
Ordinary `EStateM` bind semantics ensure that when an action fails, later actions do not run.

The characterization is intentionally sensitive to the implementation.
The bridge identifies the source-level for loops with the six folds described by the theorem, and the sequencing proof shows that those folds run in order while threading the resulting states between them.
If the deposit gate, operation families, loop order, handlers, or `SSZList` iteration behavior changes, the proof should stop compiling instead of continuing to certify an outdated description of the function.

## Upstream Work

Proof: [etheorem PR #57](https://github.com/etheorem/etheorem/pull/57) — `processOperations` operation-sequencing and state-propagation proofs
