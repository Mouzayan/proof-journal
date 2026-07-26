# Understanding `is_valid_indexed_payload_attestation`

## The goal

The proof is about `isValidIndexedPayloadAttestation`, a function that validates a vote made by a group of Ethereum validators about a block’s execution payload.

An attestation initially records its participants in a compact form: a sequence of bits, with each `1` indicating that the validator at the corresponding position in the committee participated.

Before validation, these bits are expanded into explicit validator indices, for example, validators `42`, `107`, and `301`. This lets the protocol:

- identify the validators involved
- confirm that their indices are within the bounds of the validator registry
- retrieve their public keys
- verify their aggregate signature

In Ethereum’s BLS signature scheme, each participating validator signs the attestation using its own private key. Those individual signatures are then mathematically aggregated into a single signature. The validators’ public keys are used to verify that aggregate signature.

The goal is to prove that `isValidIndexedPayloadAttestation` correctly determines whether this expanded, or _indexed_, attestation is valid. i.e., it should return `true` precisely when the attestation is properly formed, refers to registered validators, and carries a valid aggregate signature from those validators.

The function is a _pure validation predicate_: it only examines the attestation and the information supplied to it. It produces a yes or no answer without modifying the blockchain state.

## What the proof should establish

The Lean theorem for `isValidIndexedPayloadAttestation` should characterize exactly when the function returns `true`. It should prove both directions:

- If the function returns `true`, every validation condition has passed.
- If every validation condition holds, the function returns `true`.

Examining the function and its dependencies identifies the conditions the theorem must capture:

- The participant list is non-empty.
- The validator indices are adjacent nondecreasing. Repeated indices are permitted because the same validator may occupy multiple PTC seats.
- Every validator index is within the validator registry.
- The configured cryptographic backend accepts the aggregate signature for the exact public keys, payload vote, domain, and slot-derived epoch computed by the function.

The theorem provides a backend-generic characterization of the function. In other words, it describes the function’s behavior independently of any particular cryptographic backend. Once the structural checks pass, it proves that the function sends the expected inputs to the configured backend and returns the backend’s aggregate-signature verification result.

It does not prove that callers construct indexed attestations correctly or that the backend or underlying BLS cryptography is sound.

## How the proof is structured

The proof is organized in two layers.

The first theorem, `isValidIndexedPayloadAttestation_eq_true_iff_checks`, unfolds the function and describes its result in terms of the checks performed by the implementation itself. These include the literal `Array.all` expressions used for adjacent ordering and registry bounds, together with the non-empty-list check and the aggregate-signature verification call.

Two bridge lemmas then translate those `Array.all` expressions into clearer propositions about individual validator indices. The second theorem, `isValidIndexedPayloadAttestation_eq_true_iff`, uses these lemmas to give a more readable characterization: every adjacent pair is nondecreasing, every validator index is in range, and the configured backend accepts the exact verification inputs constructed by the function.

This two-layer design keeps the proof closely connected to the executable implementation while also producing a theorem that is easy to understand and use elsewhere. Together, the two theorems cover every possible Boolean outcome, including empty input, failed ordering, out-of-range indices, backend rejection, and successful validation.

This also explains two edge cases. A one-element list passes the ordering check because it has no adjacent pair to compare. Duplicate indices are allowed because the ordering check uses “less than or equal to,” not strict “less than.”

The proof uses the same Gloas-local definitions of `getDomain` and `computeEpochAtSlot` as the implementation, ensuring that the theorem describes the code exactly. The result is one theorem that mirrors the code directly and another that expresses the same checks more clearly in terms of validator indices—all without requiring mathlib.

## Upstream work

Proof in [etheorem](https://github.com/etheorem/etheorem/pull/38)
