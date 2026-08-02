# Define the engine contract

**Wave 1** — the core-owned spine: what actually runs the participants, and the safety
floor no algorithm can bypass.

## Why

Kai's placement is explicit: the framework belongs in openhab-core, with "new interfaces,
registries and providers", while the energy-management _algorithm_ "could possibly even be
as simple as a rule template, so that people could implement easily alternative energy
management algorithms and also do this through scripts"
([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)).
That split — swappable brains, core-owned guardrails — only works if the contract of the
core engine is specified: one evaluation loop, deterministic conflict resolution, hard
electrical limits enforced ahead of any optimization, shadow-first operation, and sane
actuation. Those runtime behaviours are what makes maintainers ship an EMS and users
trust it with a live house; without them the participant model is a data model, not a
system.

> **Provenance note.** Most requirements here trace to thread comments. Three carry
> **production-experience provenance from the reference implementation** instead —
> the master kill switch, acknowledgement-window actuation, and the "regardless of the
> active algorithm" generalization of limit enforcement — and are flagged inline. They
> are field-proven but should be reviewed as proposals, not thread consensus.
>
> **Prototype note.** Three requirements were later _sharpened_ — not answered — after the
> wave-1 prototype was built from this corpus: the limit floor must state the rule by which
> it resolves an overload (E13), the acknowledgement window must have a defined length and
> a defined behaviour on expiry (E8), and the enumeration of engine-owned responsibilities
> is stated in one place and stated to be complete (N7). Each carries that provenance on
> its Source line, and every value the sharpening left open is a numbered question in
> `design.md` — §3 for E8, §8 for E13, §16 for N7, and §§6–21 for everything else the
> prototype opened. See `docs/PROTOTYPE_FEEDBACK.md`.
>
> **Owner-decision note (2026-08-02).** Most of those questions have since been answered by
> the owner of the reference implementation (`docs/OWNER_DECISIONS.md`): the ladder and the
> master stop, shadow granularity, the trim-or-defer rule, what the floor books, the safe
> state on a degraded safety input, where protection timing comes from, the engine-owned
> enumeration, the acknowledgement semantics, the priority direction and the decision-outcome
> vocabulary. The same pass wrote in the reversible parameters as **stated defaults** — a
> 60-second tick with a 5-second debounce, a 60-second acknowledgement window, a clean slate
> for engine-held state across a stop — and settled that a deferral is terminal for the
> cycle, replanning belonging to `define-grid-constraints`. **They are owner decisions in a
> reference implementation, not thread
> consensus** — every requirement they touched says so on its Source line, and every
> rejected alternative stays in `design.md`. Two deserve a reviewer's attention up front:
> the master stop halts **everything** because the owner **overrode** the dispatch-only
> recommendation put to them (§6), and the priority direction **contradicts the reference
> binding's own live contract**, which now has to be reconciled (§19).

## What changes

- Define the `engine-contract` capability: central evaluation, deterministic conflict
  resolution, the constraint precedence ladder, the electrical-limit floor and what it
  books, the safe state on a degraded safety input, protection timing, shadow mode, master
  stop, actuation semantics, published decision outcomes, and algorithm replaceability with
  scripts as a first-class path.

## Non-goals

- Choosing the algorithm — this change makes algorithms swappable, it does not pick one.
- The extension/contribution mechanics (own change: `define-extension-points`).
- Cost-optimal scheduling under fee models — that is planning (`define-grid-constraints`);
  this change is the runtime enforcement floor beneath it.

## Impact

- Core interfaces + runtime behaviour contract; everything else in the corpus plugs into
  this spine.
