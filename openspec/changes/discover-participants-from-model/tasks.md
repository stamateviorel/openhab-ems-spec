# Tasks

> **Decision pass 2026-08-02.** D5, D11 and D17 (Part B DP-1, DP-2) answered design.md §1,
> §2 and §5 (record: `docs/OWNER_DECISIONS.md`; owner decisions in a reference
> implementation, not thread consensus). §4's question — whether to float this change in
> #3478 — is **recorded and gated**, not decided (task 1.4).

## 1. Alignment

- [x] 1.1 Owner review of the mapping table (which Equipment→role/class inferences are safe)
      — **done (D17, Part B DP-1)**: leaf tag decides the role, meters are candidates only,
      EVSE granularity is read from the Property tag, `Generator`/`WindGenerator`/`UPS` are
      parked
- [x] 1.2 Decide the review surface (design.md §2) — **decided (D17, Part B DP-2): an
      EMS-owned proposal registry on the Inbox lifecycle**, keyed on participant identity,
      surfaced through `define-energy-ui` _Guided setup surface_; approval writes `energy:`
      metadata
- [ ] 1.3 Only after the participant model (wave 1) settles — this change depends on it;
      D5 supplies the piece it was waiting on (identity = Item name, one precedence chain)
- [ ] 1.4 **Gated, not decided (DP-4).** Whether and when to float this change's provenance
      in #3478 is procedural and follows D5 (declarations are metadata, so discovery has
      something to write). The owner's explicit-go workflow applies: draft the body plus the
      exact command, show it, wait for "go", then post. **Nothing has been posted**

## 2. Specification hardening

- [ ] 2.1 Pin the exact Equipment→role and Equipment+Point→class mapping as a table — with
      the leaf-tag rule stated at the top, since `EVSE` and `UPS` both sit under
      `PowerSupply`
- [x] 2.2 Decide handling of ambiguous cases (multiple grid meters, unknown setpoint units)
      — **decided (D17, Part B DP-1(i) and DP-1(ii))**: every meter is a ranked candidate
      and none is auto-assigned; setpoint dimension comes from the Property tag, then UoM,
      then the user — never an assumed 32 A or 11 kW

## 3. Prototype path (after 1.x)

- [ ] 3.1 Read the semantic model, emit a proposed participant set (pure, testable)
- [ ] 3.2 Confirm-then-seed flow: accepted proposals become `energy:` declarations, ignored
      ones stay ignored across re-runs
- [ ] 3.3 Verify the whole precedence chain from the discovery side: explicit metadata and
      contributed declarations both outrank a proposal
