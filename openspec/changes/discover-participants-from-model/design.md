# Design notes — open questions

## 1. How complete a mapping is safe to propose?

The 1:1 table in the proposal is the confident core. Edge cases to decide:

- **Grid vs. sub-meter.** Several `ElectricMeter`/`Power` items may exist; which is the
  grid connection vs. a device sub-meter? Location in the model (top-level vs. inside an
  Equipment) is a hint, not a guarantee. Propose-and-confirm covers it.
- **`EVSE` control granularity.** A `Setpoint` says "controllable" but not amps-vs-watts
  or the range — read from UoM/state-description where present, else confirm.
- **`Generator`/`WindGenerator`/`UPS`.** Providers, but with roles the four-role model
  (grid/pv/battery) doesn't name yet — park until the provider roles settle.

## 2. Where does discovery live?

An openHAB `DiscoveryService`-style flow is the native analogy (Inbox → approve). Options:
propose into an Inbox-like review list; or generate a draft `energy:` metadata set the
user edits. Either keeps "propose, never activate" intact.

## 3. Relationship to the model's own gaps

This change is only as strong as openHAB's semantic coverage of energy equipment. If the
provider-role vocabulary (grid/pv/battery) or a future `Evse` capability grows, the
mapping grows with it. It should degrade gracefully, never guess (see the "No guessing"
requirement).

## 4. Provenance / status

New proposal, owner + assistant, not yet raised in #3478. If it survives local review it
is a good candidate to float in the thread as the "out-of-the-box" onboarding story Kai
has repeatedly gestured at — but it is explicitly _not_ presented as thread consensus.

## 5. What the wave-1 prototype surfaced

Opened by building the wave-1 slice against this corpus, not by #3478. Ids are the
prototype's own (see `docs/PROTOTYPE_FEEDBACK.md`); recorded, not answered.

- **_Explicit declaration wins_ is the corpus's only precedence rule, and wave 1 needs a
  different one (B9).** It settles an explicit `energy:` declaration against a **discovered**
  proposal. Wave 1 produces a conflict it does not cover — explicit versus **contributed** —
  and the prototype had to extrapolate this requirement to have any default at all, saying
  so in the code. `define-extension-points` now carries a deterministic-resolution
  requirement for the contributed case. Open for this change: are the two the same rule seen
  from two sides, or do they deliberately differ?
- **This requirement is keyed on the item; the wave-1 planes are keyed on the participant
  (A12, B8).** "an explicit declaration … for the same item" holds only while one item means
  one participant. Participant identity is undefined across the corpus and is being added in
  `define-participant-model`; until it lands, discovery cannot tell whether its proposal
  duplicates an existing declaration except by whatever that identity turns out to be. Worth
  re-reading this requirement once it does.
