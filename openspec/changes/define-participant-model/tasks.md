# Tasks

## 1. Alignment

- [ ] 1.1 Maintainer review of the requirement set (thread or PR review) — including the
      owner decisions of 2026-08-02, which are a reference implementation's calls and not
      thread consensus (`docs/OWNER_DECISIONS.md`)
- [x] 1.2 Decide the declaration mechanism (design.md §1) — **both**: item metadata and a
      programmatic contribution path behind one provider interface (D5)
- [ ] 1.3 Decide the core vs. add-on boundary (design.md §2)

## 2. Specification hardening

- [ ] 2.1 Fold review outcomes back into this spec (OpenSpec change → archive cycle)
- [ ] 2.2 Cross-check the four classes against one more real installation that is
      neither storm.house nor the reference (volunteer wanted)
- [ ] 2.3 Close what the owner pass left open here: which provider roles may be
      controllable (design.md §5.8), the deadline forms (§5.3 — Part C recorded an answer,
      both forms with a bare local time resolving in the site zone, which D18 left
      unwritten), where the load curve belongs (§5.5) and its sample semantics (§5.6), and
      the word that replaces `EnergyProvider` (§5.16 — a thread question, not a unilateral
      rename). Closed by the parameter pass and no longer open here: the demand kinds
      (§5.4, D17 · PM-5.4), the level-to-_n_-modes mapping (§5.10, D17 · PM-5.10) and the
      per-role sign conventions (§5.11, D11)
- [ ] 2.4 Close the one edge the mode mapping leaves: the level gate is declared on Simple
      consumers (§5.7) while a two-mode ModeControllable consumer is steered by that same
      gate (§5.10) — either the gate's scope widens by one case, or a two-state device is
      declared Simple. Both readings recorded, neither decided
- [ ] 2.5 Confirm the three reconciliations made here that **no owner decision made**, each
      reversible in one place: a controllable provider carries a priority on the consumer
      scale, without which D10's "battery charging the engine can reclaim" has no second
      operand and reclaimability is undecidable (§5.8); the hands-off flag binds the
      engine's own electrical-limit floor and safe state, not only contributed algorithms as
      D13 framed it (§5.7); and what the floor books when a running load measures **below**
      its declaration, where pack A6's two sentences disagree (§5.14)

## 3. Prototype path (after 1.x)

- [ ] 3.1 Draft the core interfaces (participant, provider, consumer, profile)
- [ ] 3.2 Port the reference's metadata parser as one declaration-mechanism candidate
- [ ] 3.3 Shadow-mode harness so early adopters can validate without live control
- [ ] 3.4 Reconcile the reference binding with D4 — its `Controller` contract says "lower
      runs first, higher wins on conflict", which the _Priority_ requirement now
      contradicts (design.md §5.2)
