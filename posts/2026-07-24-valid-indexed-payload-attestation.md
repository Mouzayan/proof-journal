# Understanding `isValidIndexedPayloadAttestation`

## Upstream Work

Proof PR: [etheorem #38](https://github.com/etheorem/etheorem/pull/38) — characterizes indexed payload attestation validation, including per index checks, non strict ordering, and aggregate-signature verification.

## The Goal

`isValidIndexedPayloadAttestation` validates a vote made by a group of Ethereum validators about a block’s execution payload.

An attestation initially records participation as a compact sequence of bits.
Each `1` marks a participating seat on the Payload Timeliness Committee (PTC).
On the block-processing path, `getIndexedPayloadAttestation` maps the selected seats to explicit validator indices, such as validators `42`, `107`, and `301`, and sorts them before passing the indexed attestation to this function.

This lets the protocol:

- identify the validators involved;
- confirm that their indices are within the validator registry;
- retrieve their public keys; and
- verify their aggregate-signature.

In Ethereum’s BLS signature scheme, each participating validator signs the attestation using its own private key.
Those signatures are aggregated into a single signature, which can be checked using the validators’ public keys.

The goal is to characterize exactly when `isValidIndexedPayloadAttestation` accepts an indexed attestation according to its implemented checks.
In other words, the theorem should establish precisely when the function returns `true`.

The function is a _pure validation predicate_:
it examines the attestation and the supplied state, then produces a yes-or-no answer without modifying the blockchain state.

## What the Proof Establishes

The Lean theorem characterizes exactly when `isValidIndexedPayloadAttestation` returns `true`.
It proves both directions:

- If the function returns `true`, every validation condition has passed.
- If every validation condition holds, the function returns `true`.

The required conditions are:

- The participant list is non empty.
- The validator indices are nondecreasing: each index is greater than or equal to the one before it.
- Every validator index is within the validator registry.
- The configured cryptographic backend accepts the aggregate-signature for the exact public keys, payload vote, domain, and slot-derived epoch computed by the function.

This is a _backend-generic characterization_.
Once the structural checks pass, the theorem proves that the function sends the exact inputs it computes to the configured backend and returns its aggregate-signature verification result.
It does not prove that callers always construct indexed attestations correctly or that the backend or underlying BLS cryptography is sound.

## How I Structured the Proof

The proof is organized in two layers.

**Implementation-facing layer.** The first theorem, `isValidIndexedPayloadAttestation_eq_true_iff_checks`, unfolds the function and describes its result using the implementation’s literal checks.
These include the `Array.all` expressions for adjacent ordering and registry bounds, the non empty list check, and the aggregate-signature verification call.

**Readable layer.** Two bridge lemmas translate the `Array.all` expressions into clearer propositions about individual validator indices.
The second theorem, `isValidIndexedPayloadAttestation_eq_true_iff`, uses them to state that the list is non empty, every adjacent pair is nondecreasing, every validator index is in range, and the configured backend accepts the exact verification call constructed by the function.

This characterization also explains two edge cases.
A one-element list passes the ordering check because it has no adjacent pair to compare.
Duplicate indices are allowed because the check uses “less than or equal to” rather than strict “less than.”
This matches the possibility that one validator occupies multiple PTC seats.

The proof uses the same Gloas-local definitions of `getDomain` and `computeEpochAtSlot` as the implementation, ensuring that the theorem describes the code exactly.
It does not introduce mathlib as an additional dependency.

## Takeaway

The key design decision was to separate fidelity to the implementation from readability.
Proving the literal `Array.all` checks first kept the argument tied to the executable code, and the bridge lemmas then turned those checks into statements that are easier to understand and reuse.

The most surprising detail was that the required ordering is non strict.
Duplicate validator indices are therefore part of the behavior the theorem must preserve.
