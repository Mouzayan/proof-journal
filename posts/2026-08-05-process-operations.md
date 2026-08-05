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

## Upstream Work

Proof: [etheorem PR #57 — `processOperations`](https://github.com/etheorem/etheorem/pull/57) operation-sequencing and state-propagation proofs
