# Tasks

> **Decision pass 2026-08-02.** D5, D13, D16, D17 and D18 answered design.md §1–§6 (record:
> `docs/OWNER_DECISIONS.md`; owner decisions in a reference implementation, not thread
> consensus). Tasks a decision closed are checked and carry what was decided; what stayed
> open is unchanged.

## 1. Alignment

- [ ] 1.1 Maintainer discussion of the mechanism table (design.md §1) — **decided for the
      reference implementation by D5** (both mechanisms behind one provider SPI, identity =
      Item name, one precedence chain); Kai's "to what extent do new add-on types make
      sense" is still the thread's to land
- [ ] 1.2 Naming pass (§2: EnergyConsumer vs DemandDescription et al.) — D17 parks the
      rename for one combined pass; **which word replaces `EnergyProvider` is left as a
      thread question** (Part D remainder), worth one line in #3478 rather than a
      unilateral rename
- [ ] 1.3 Decide whether actuation adapters are contributable (§3, with engine-contract 1.3)
      — D5 settled the _selection_ half (named in configuration, never ranked);
      contributability itself is still open
- [x] 1.4 Decide whether the `energy:` metadata schema is core's to specify (§5.1) —
      **decided (D18, Part C EP-5.1): core owns the namespace and publishes it as a
      `MetadataConfigDescriptionProvider`**, machine-readable rather than prose. Worth
      confirming in the thread, since the question was addressed to @kaikreuzer

## 2. Specification hardening

- [ ] 2.1 Specify contributor lifecycle events the engine must tolerate (appear, update,
      disappear, reappear) — note that scripts _do_ have an unload hook
      (`ProviderScriptExtension`), and that D5's `contributorId` makes a re-registration a
      further statement rather than a duplicate
- [ ] 2.2 Define the selection/composition configuration surface (§4) — the rule is decided
      (per-role configuration, defaulting to highest `service.ranking`); the surface itself
      is not
- [x] 2.3 Answer the reporting-surface questions the wave-1 prototype surfaced: a
      declaration in error, a degraded source, and the safe state a lost safety input
      degrades to (§5.5–§5.7) — **decided (D16 pack A8, D14)**: skip the whole participant
      and report through `ConfigStatusProvider`; report degradation on the engine's
      outcome/status surface; freeze-and-floor as the safe state
- [x] 2.4 Decide how a site picks the component that turns decisions into writes (§5.11) —
      **decided (D5's exception): one sink named in configuration, per-participant
      override, never a ranking**. Requirement added: _The actuation sink is named, never
      ranked_

- [x] 2.5 Say what a declaration naming an Item that does not exist means (§5.10) —
      **decided (D17, Part B EP-5.10): a runtime condition, not a declaration error.** The
      parser accepts it, the engine reports a degraded source at warning level, and it binds
      if the Item appears later. Requirement added: _A declared Item name that does not
      resolve is a runtime condition_, whose third scenario draws the line against
      `energy-participants` _Malformed declarations are reported, never partially accepted_

## 3. Prototype path (after 1.x)

- [ ] 3.1 One data provider contributed via plain items (mechanism a) and the same one
      via a typed service (mechanism d) — compare ergonomics
- [ ] 3.2 Script-registered demand description end-to-end, including unload and reload
      under the same `contributorId`
- [ ] 3.3 Kill a contributor mid-run; verify baseline fallback + degraded-source report
- [ ] 3.4 Prove the security boundary (§6): a contributed algorithm cannot start a `never`
      consumer or lift a user-declared level gate, and can lift the engine's own
      level-derived steering
