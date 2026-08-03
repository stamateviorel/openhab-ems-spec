# Tasks

## 1. Alignment

- [ ] 1.1 Maintainer/thread review — especially the three reference-sourced requirements
      (master stop, ACK actuation, limits-regardless-of-algorithm) and the owner decisions
      of 2026-08-02, which are a reference implementation's calls and not thread consensus
      (`docs/OWNER_DECISIONS.md`)
- [x] 1.2 Decide the cadence model (design.md §1) — a 60 s tick plus event-responsive
      re-evaluation on declared readings, debounced 5 s, all three stated as defaults a
      site may change (D17, Part B EC-1); shadow/stop granularity is settled — two
      controls, shadow global plus per-algorithm and per-participant, fresh install
      shadowed with the stop disengaged (§2, D3)
- [ ] 1.3 Decide where ACK handling lives (§3) together with `define-extension-points` —
      still open, as are the default tolerance value and the changed-command rule; what
      counts as an acknowledgement and what expiry does are settled (D8)
- [x] 1.4 Decide what the master stop covers — it halts **everything**, evaluation and
      contributed algorithms included (§6, D1, overriding the dispatch-only
      recommendation); protections are not enforced while it is engaged and that is
      reported (§5, D1/D2); what survives a resume is answered per state item (§7)
- [x] 1.5 Settle the engine-owned enumeration's membership — prohibitions are engine-owned
      and the list is closed (hands-off flag, user level gate, readiness interlock, device
      protections, the safety-input enumeration); the engine's own level-derived steering
      is the one overridable member (§16, D13)
- [x] 1.6 Resolve the level-plane coupling with `define-energy-levels` — the level is a
      **product** of the cycle, computed by the engine from its own snapshot, and reaches
      it through an in-process seam rather than a published Item (§4 I1, §18 I2, D6)

## 2. Specification hardening

- [ ] 2.1 Define the minimum context-snapshot content (§4), including whether an algorithm
      may retain a snapshot beyond its cycle (E17) — the source of elapsed time for the
      protection durations is settled (`Item.getLastStateChange()`, no engine-held timers,
      D7)
- [ ] 2.2 Reconcile the runtime limit floor with `define-grid-constraints` planning
      budgets — one declaration, two consumers of it; the replanning half is settled: a
      deferral is terminal for the cycle and travels to the planner as a published outcome,
      this change keeping the runtime floor only (§17, E16, D16 / pack A14)
- [x] 2.3 State the trim-or-defer rule (§8, D9) and what the floor books (§9, D15 —
      `max(declared, measured)` while running, declared while not, re-booked every cycle);
      mode changes are exempt from the planner's budget with an optional per-mode draw
      (§10, D15). Surplus (§11) is answered separately and only in part: grid export plus
      reclaimable battery charging (D10, on D11's signs), with the instantaneous-versus-
      averaged question and the running-consumer recursion still open
- [x] 2.4 Define the decision-outcome vocabulary (§12), the shadow-output surface (§14) and
      the handling of a decision naming an unknown participant (§13) — an outcome enum plus
      a reason, published as deduplicated events, the current cycle over REST, one status
      Item (D16 / pack A8); **which component provides each surface** settled 2026-08-03
      (D23, §23): events stay with the framework, the status Item and the REST view move to
      a separate publishing component, so the framework writes no Item at all
- [ ] 2.5 Define algorithm identity and ownership — who may replace whose id (§15); the
      multiplicity half is settled (several algorithms may be active at once, D4)
- [x] 2.6 Settle the tie-break between two _decisions_ of equal priority on one device —
      D4 fixes the tie-break for ordering participants only, and conflict resolution groups
      by participant so that id always ties; **decided 2026-08-03 (D30, §19): algorithm id
      ascending, then the rendered action**, with the `"OFF" < "ON"` consequence recorded as
      an accepted accident rather than a designed safety ordering
- [ ] 2.7 Confirm the four cross-change reconciliations this change carries that **no owner
      decision made** — each is a place where two decisions landed on the same behaviour
      from different changes, and each is reversible in one place: the ladder's two stated
      exceptions, a running Batch programme and a hands-off consumer, which sit above the
      electrical-limit rung the ladder otherwise declares absolute (§5, §8); `stopped`
      being reachable only for the cycle in flight when the stop lands, since D1 halts the
      evaluation that would otherwise produce decisions to mark (§12); deferral of a Simple
      load that is **already running** meaning a switch-off (§8); and the acknowledgement
      tolerance band being absolute in the control Item's dimension rather than
      proportional (§3)

## 3. Prototype path (after 1.x)

- [ ] 3.1 Port the reference's controller/priority contract as a candidate core interface
- [ ] 3.2 Shadow-mode harness: decisions logged + comparable against existing automation
- [ ] 3.3 Script-algorithm proof: one trivial algorithm implemented as a JSR223 script
      driving the same engine
- [ ] 3.4 Build the second component D23 names (`org.openhab.core.energy.publish` in the
      reference): it consumes the framework's decision events and owns every write — the
      status Item, the REST view, and the current level and planned TimeSeries of
      `define-energy-levels`. Two things to prove alongside it: the framework still reports
      by itself when the component is absent, and the no-write proof over the framework
      still holds with the event tokens deliberately removed from its forbidden list (§23)
- [ ] 3.5 Carry D28's persistence dependency into the framework and prove the report tells
      "history is not being kept" from "no change observed yet" — the one case where a null
      `lastStateChange` has two causes with different remedies (§4)
