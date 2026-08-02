# Design notes — open questions (answered 2026-08-02)

> **Decision pass, 2026-08-02.** The owner answered this change's open questions; the
> record is [`docs/OWNER_DECISIONS.md`](../../../docs/OWNER_DECISIONS.md). Everything
> marked _Answered_ is an **owner decision in a reference implementation, not thread
> consensus** — which matters more here than anywhere else in the corpus, because this
> change was never raised in #3478 at all (§4). Nothing was deleted.

## 1. How complete a mapping is safe to propose?

The 1:1 table in the proposal is the confident core. Edge cases to decide:

- **Grid vs. sub-meter.** Several `ElectricMeter`/`Power` items may exist; which is the
  grid connection vs. a device sub-meter? Location in the model (top-level vs. inside an
  Equipment) is a hint, not a guarantee. Propose-and-confirm covers it.
- **`EVSE` control granularity.** A `Setpoint` says "controllable" but not amps-vs-watts
  or the range — read from UoM/state-description where present, else confirm.
- **`Generator`/`WindGenerator`/`UPS`.** Providers, but with roles the four-role model
  (grid/pv/battery) doesn't name yet — park until the provider roles settle.

**Answered — D17** (Part B, DP-1(i), DP-1(ii), DP-1(iii) and the rule-shape warning), each
edge case separately.

_Grid vs. sub-meter (DP-1(i))._ Propose **every** `ElectricMeter` carrying a Power
measurement as a **candidate**, rank one directly under a Location first, and **never
auto-assign — not even when there is exactly one candidate**. `SemanticTags.csv` has no
grid tag and no top-level/sub distinction, so any auto-assignment is a guess; the owner's
own site has seven indistinguishable clamp meters on one bridge. _Alternative preserved:_
auto-assigning a single candidate, which is tempting and wrong for exactly that site.

_EVSE control granularity (DP-1(ii)), and a correction._ Read amps-versus-watts from the
setpoint Point's **Property tag** — `Current` and `Power` are sibling first-class tags in
`SemanticTags.csv` — falling back to the item's UoM dimension, then to confirmation; read
the range from the item's `StateDescription` min/max, else confirm. **Never assume 32 A or
11 kW.** The bullet above says "read from UoM/state-description", which misses that the
Property tag already carries the dimension; that phrasing is stale and is corrected here
rather than deleted.

_Generator / WindGenerator / UPS (DP-1(iii))._ **Park; propose nothing unless the user opts
in.** _No guessing without semantics_ already covers it.

_The rule shape — a warning worth a requirement._ **The parent tag is not usable as the
role rule; the leaf tag must be.** `EVSE` sits under `PowerSupply` but is a **consumer**,
and `UPS` sits there too and is neither. Any "everything under `PowerSupply` is a provider"
rule proposes the wallbox as a generator.

## 2. Where does discovery live?

An openHAB `DiscoveryService`-style flow is the native analogy (Inbox → approve). Options:
propose into an Inbox-like review list; or generate a draft `energy:` metadata set the
user edits. Either keeps "propose, never activate" intact.

**Answered — D17** (Part B, DP-2). _Decision:_ **an EMS-owned proposal registry modelled on
the Inbox lifecycle** — NEW / IGNORED / approve-becomes-declaration — **keyed on participant
identity** (the Item name, per D5), surfaced through `define-energy-ui`'s guided-setup
requirement, with approval writing `energy:` metadata. _Why an EMS-owned registry rather
than the Inbox itself:_ `DiscoveryResult.getThingUID()` and `Inbox.add`'s dedupe-on-Thing-ID
mean an item-keyed proposal cannot reuse the Inbox without inventing a fake `ThingUID`. The
**lifecycle** is still the right one, and users already know it. _Alternative preserved:_
generating draft `energy:` metadata for the user to edit — which is what approval does
anyway, so the two differ only in whether the draft exists before the user says yes.

## 3. Relationship to the model's own gaps

This change is only as strong as openHAB's semantic coverage of energy equipment. If the
provider-role vocabulary (grid/pv/battery) or a future `Evse` capability grows, the
mapping grows with it. It should degrade gracefully, never guess (see the "No guessing"
requirement).

Unchanged by the decision pass.

## 4. Provenance / status

New proposal, owner + assistant, not yet raised in #3478. If it survives local review it
is a good candidate to float in the thread as the "out-of-the-box" onboarding story Kai
has repeatedly gestured at — but it is explicitly _not_ presented as thread consensus.

**Not decided — recorded and gated (DP-4, Part D remainder).** Whether and when to float
this change's provenance in #3478 is **procedural, not technical**, and it should follow
D5 rather than precede it: discovery's value depends entirely on whether declarations are
metadata (discovery writes metadata — obviously useful) or a programmatic SPI (discovery
has nothing to write). D5 answered that in metadata's favour, so the case is now
arguable — but the owner's **explicit-go workflow applies**: draft body plus the exact
command, shown, waited on, then posted. Nothing has been posted, and this paragraph is not
a licence to post. Tracked as task 1.4.

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

**Both answered — D5.** They are **the same rule seen from two sides**: one precedence
chain, **explicit metadata > contributed (ties by `service.ranking`) > discovered
proposal**, used everywhere. And participant **identity is the Item name**, optionally
overridden by an explicit `id` — so the item-keyed wording above and the participant-keyed
planes of wave 1 coincide by construction rather than by luck, and discovery can tell
whether its proposal duplicates an existing declaration. _Alternative preserved:_ the two
rules deliberately differing — a maintainer who wants a discovered proposal to outrank a
contributed one changes one link in the chain, in `define-extension-points`
_Deterministic resolution between contributors_ and here.
