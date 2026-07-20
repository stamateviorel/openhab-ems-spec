# Tasks

## 1. Alignment
- [ ] 1.1 Owner review of the mapping table (which Equipment→role/class inferences are safe)
- [ ] 1.2 Decide the review surface (Inbox-style approval vs. generated draft metadata) — design.md §2
- [ ] 1.3 Only after the participant model (wave 1) settles — this change depends on it

## 2. Specification hardening
- [ ] 2.1 Pin the exact Equipment→role and Equipment+Point→class mapping as a table
- [ ] 2.2 Decide handling of ambiguous cases (multiple grid meters, unknown setpoint units)

## 3. Prototype path (after 1.x)
- [ ] 3.1 Read the semantic model, emit a proposed participant set (pure, testable)
- [ ] 3.2 Confirm-then-seed flow: accepted proposals become `energy:` declarations
- [ ] 3.3 Verify explicit declarations always win over proposals
