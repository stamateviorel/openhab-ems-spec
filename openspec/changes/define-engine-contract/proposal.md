# Define the engine contract

**Wave 1** — the core-owned spine: what actually runs the participants, and the safety
floor no algorithm can bypass.

## Why

Kai's placement is explicit: the framework belongs in openhab-core, with "new interfaces,
registries and providers", while the energy-management *algorithm* "could possibly even be
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

## What changes

- Define the `engine-contract` capability: central evaluation, deterministic conflict
  resolution, the electrical-limit floor, shadow mode, master stop, actuation semantics,
  and algorithm replaceability with scripts as a first-class path.

## Non-goals

- Choosing the algorithm — this change makes algorithms swappable, it does not pick one.
- The extension/contribution mechanics (own change: `define-extension-points`).
- Cost-optimal scheduling under fee models — that is planning (`define-grid-constraints`);
  this change is the runtime enforcement floor beneath it.

## Impact

- Core interfaces + runtime behaviour contract; everything else in the corpus plugs into
  this spine.
