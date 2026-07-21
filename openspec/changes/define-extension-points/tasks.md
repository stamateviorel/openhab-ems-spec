# Tasks

## 1. Alignment
- [ ] 1.1 Maintainer discussion of the mechanism table (design.md §1) — Kai's "to what
      extent do new add-on types make sense" is the decision to land
- [ ] 1.2 Naming pass (§2: EnergyConsumer vs DemandDescription et al.)
- [ ] 1.3 Decide whether actuation adapters are contributable (§3, with engine-contract 1.3)

## 2. Specification hardening
- [ ] 2.1 Specify contributor lifecycle events the engine must tolerate (appear, update,
      disappear, reappear)
- [ ] 2.2 Define the selection/composition configuration surface (§4)

## 3. Prototype path (after 1.x)
- [ ] 3.1 One data provider contributed via plain items (mechanism a) and the same one
      via a typed service (mechanism d) — compare ergonomics
- [ ] 3.2 Script-registered demand description end-to-end
- [ ] 3.3 Kill a contributor mid-run; verify baseline fallback + degraded-source report
