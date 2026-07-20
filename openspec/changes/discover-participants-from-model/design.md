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
has repeatedly gestured at — but it is explicitly *not* presented as thread consensus.
