# Designing compositional ePBS and FOCIL proofs in Heze

Heze inherits most of its fork-choice machinery from Gloas and adds FOCIL (Fork-Choice-Enforced Inclusion Lists) on top of it.
Looking at the individual functions separately is only part of the formal verification problem.
The broader verification question is what happens when the new FOCIL state and decisions are consumed by the ePBS (enshrined proposer-builder separation) machinery Heze inherited.

Most of the proofs I had been working on focused on characterizing individual functions: what they return, which branches they take, and how they update state.
This work adds another level.
Instead of only asking:

> What does this function do?

it also asks:

> What fact does this function establish for another component, and what happens when that fact is consumed later?

This led to a new proof direction in Etheorem: proving how independently characterized functions compose across state writes, later reads, subsystem boundaries, and even fork boundaries.

## Upstream Work

Proof roadmap PR: [etheorem #127](https://github.com/etheorem/etheorem/pull/127) — adds the Heze ePBS–FOCIL compositional proof plan to `PROOF_LEDGER.md`.

## The Goal

Gloas introduces the ePBS fork-choice machinery.
Heze is a relatively thin fork over Gloas: much of that machinery is inherited, while Heze adds the FOCIL state and logic needed for inclusion lists.

This creates an interesting verification boundary.

A FOCIL handler can write something into the fork-choice store, and later an inherited ePBS function can read that information and use it in a fork-choice decision.
When Heze modifies an inherited function, the surrounding inherited logic uses the Heze-specific version.
This is how FOCIL can affect ePBS behavior even inside machinery originally inherited from Gloas.

So proving each function independently does not by itself prove that the pieces fit together correctly.

The proof shape I settled on is modular:

`writer → state fact / invariant → reader → fork-choice consequence`

A writer first establishes a fact about the state it returns.
A later invariant can show that the fact remains available across later valid protocol transitions.
A reader consumes the fact without needing to know how it was originally produced.
Finally, a fork-choice theorem shows the protocol consequence.

One important decision is that these proofs remain separate.

I do not want every downstream theorem to depend directly on one specific writer if all it really needs is a store property.
The proofs need to be reusable and decoupled from unrelated parts of the implementation.

## The Three Layers

It made sense to reason about the work in three layers.

The first is **cross-fork refinement**.

Heze and Gloas have different `Store` types and separate Lean definitions, so a theorem about `Gloas.shouldExtendPayload` does not automatically prove the same thing about the Heze version.

Rather than trying to say the two stores are equal, the proposed `SharesEpbsView` condition compares only the parts of each store that the ePBS decision can observe.

Under that shared view, a recorded FOCIL `true` lets Heze continue through the same ePBS logic inherited from Gloas.
A recorded `false` stops the Heze extension decision early.
A missing FOCIL record is a Heze-specific assertion/error case.

The second layer is **state safety and invariants**.

This is where the write side and read side meet.

For example, when `onExecutionPayloadEnvelope` successfully processes a payload, Heze records both the payload and its FOCIL satisfaction result at the same block root.

The reader later expects that a verified payload has this result available.

The invariant is intentionally one-directional:

`verified payload → recorded FOCIL satisfaction result`

I originally thought of this as a symmetric pairing property, where the payload and its FOCIL result would always imply the existence of each other.
But the reader only needs one direction: if a payload is verified, its FOCIL satisfaction result must be available.
The reverse is not needed and is not true for every intermediate `Store`, because `recordPayloadInclusionListSatisfaction` can write the satisfaction result before the envelope handler commits the payload.

The invariant is therefore meant to hold for stores produced through valid protocol transitions, not for every possible `Heze.Store` value that could be constructed.

The third layer is **protocol consequences**.

These proofs answer the question: after all of this bookkeeping, what actually changes in fork choice?

The first protocol consequence we plan to prove starts with a recorded FOCIL `false`, meaning Heze has checked whether the revealed execution payload satisfies the applicable inclusion-list requirements and stored the answer `false`.

In ePBS fork choice, the same block root can be considered in two payload states: `FULL`, representing the candidate with its execution payload revealed, and `EMPTY`, representing the corresponding empty-payload candidate.
For the previous-slot comparison we care about, neither voting weight nor the block root gives fork choice a reason to prefer one over the other.
The decision therefore falls to the payload-status tiebreaker.

If `shouldExtendPayload` is `false`, the tiebreaker ranks `EMPTY` above `FULL`, so the comparison selects `EMPTY`.
In other words, the FOCIL result can directly affect which of those two fork-choice candidates wins the comparison.

This deliberately proves only that specific `EMPTY`/`FULL` comparison.
It does not yet prove that the full `getHead` traversal will encounter that particular pair or that the `EMPTY` candidate will determine the final head.

## The First Composition Story

The first end-to-end proof chain we will work through follows one FOCIL result through the system:

`onExecutionPayloadEnvelope`
→ payload + FOCIL satisfaction result
→ `isPayloadInclusionListSatisfied`
→ `shouldExtendPayload`
→ `getPayloadStatusTiebreaker`
→ one `EMPTY`/`FULL` `betterOf` comparison

The first step looks only at the store returned by a successful `onExecutionPayloadEnvelope` call and proves what is true at the root associated with that payload:

- the satisfaction map contains the FOCIL satisfaction result;
- the payload map contains the payload;
- `isPayloadVerified` is therefore `true`.

These facts can already be proved using the successful-envelope equation and the existing same-key `FcMap` laws.

The next step composes them with the reader.

`shouldExtendPayload` checks that the payload is verified before reading the FOCIL result through `isPayloadInclusionListSatisfied`.

If the recorded result is `false`, the extension decision is `false`.

If it is `true`, execution continues through the remaining ePBS logic inherited from Gloas.

A directly constructed `Heze.Store` can still contain a verified payload without its FOCIL satisfaction record.
The later `VerifiedImpliesRecorded` invariant should prove that this cannot happen in stores produced through valid protocol transitions.

This separation is important.
The local writer/reader proof does not have to wait for a proof about all stores reachable through valid protocol execution.

## Why I Kept the Proofs Modular

It would be possible to aim immediately for one large theorem that starts at payload reveal and ends somewhere inside `getHead`.

I decided against that because the intermediate facts are useful on their own.

For example, the tiebreaker theorem should depend on `shouldExtendPayload = false`, not specifically on `onExecutionPayloadEnvelope`.
A successful envelope is one way to establish that result, but not the only one.

Likewise, the reader safety theorem should depend on `VerifiedImpliesRecorded`, not on the entire history of how the store reached that state.

This keeps the dependencies small:

`writer → local fact`

`store invariant → reader safety`

`reader result → fork-choice consequence`

The pieces can then be combined when an end-to-end proof needs them.

This also makes failures easier to understand: if the writer changes, the writer theorem should fail; if the reader changes, the reader theorem should fail; and if fork choice changes how `EMPTY` and `FULL` are compared, the consequence theorem should fail.
One giant theorem would make those boundaries much harder to see.

## What Is New About This Direction

The new proof vertical composes existing local proofs across two kinds of boundaries.

One is the **subsystem boundary** between FOCIL and ePBS.

The other is the **fork boundary** between Gloas and Heze.

The cross-fork part is particularly interesting because inheritance does not make the proof transfer automatic.
Gloas and Heze declarations are separate Lean constants, and their store structures are different types.
The proof therefore has to state explicitly which observations are shared and which behavior Heze intentionally changes.

## Takeaway

The main design decision here was not to treat the ePBS–FOCIL interaction as one large proof target.

A writer establishes a fact.
An invariant can preserve it.
A reader relies on it.
Fork choice exposes its effect.
A separate cross-fork refinement shows which behavior Heze preserves from Gloas and what FOCIL changes.

Keeping those steps modular gives each theorem a smaller job and makes the results reusable in different proof chains.

Function characterization is still necessary, but once enough local contracts exist, the next level of verification is proving how those contracts compose into the protocol behavior we expect.
