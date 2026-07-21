# Tasks

## 1. Alignment
- [ ] 1.1 Maintainer/thread review — especially the three reference-sourced requirements
      (master stop, ACK actuation, limits-regardless-of-algorithm)
- [ ] 1.2 Decide cadence model (design.md §1) and shadow/stop granularity (§2)
- [ ] 1.3 Decide where ACK handling lives (§3) together with `define-extension-points`

## 2. Specification hardening
- [ ] 2.1 Define the minimum context-snapshot content (§4)
- [ ] 2.2 Reconcile the runtime limit floor with `define-grid-constraints` planning
      budgets — one declaration, two consumers of it

## 3. Prototype path (after 1.x)
- [ ] 3.1 Port the reference's controller/priority contract as a candidate core interface
- [ ] 3.2 Shadow-mode harness: decisions logged + comparable against existing automation
- [ ] 3.3 Script-algorithm proof: one trivial algorithm implemented as a JSR223 script
      driving the same engine
