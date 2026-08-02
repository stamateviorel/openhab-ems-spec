# Prototype feedback — what building wave 1 surfaced

Every `> Source:` line in this corpus says where a requirement came from. Most say
[#3478](https://github.com/openhab/openhab-core/issues/3478) — a person, a comment, a
permalink. A few now say _"sharpened after the wave-1 prototype surfaced …"_ and name an
identifier from this page instead. **This page is what those identifiers mean.**

The distinction matters more than the content. Thread-sourced material is consensus
already reached by the people who run these systems. Everything catalogued here is the
opposite: it is the residue of a build, found by one implementation, agreed by nobody. A
reviewer must be able to tell the two apart at a glance, and the Source lines exist so
that they can.

## Where this came from

A throwaway wave-1 prototype was built from this corpus under
[`PROTOTYPE_TRACK.md`](PROTOTYPE_TRACK.md) — participant model, engine, level classifier,
declaration seams; shadow-only, never writing to a device. Its rules of engagement
included one that produced this page:

> **Never resolve an open question.** Where a decision is unavoidable to compile,
> implement **all** framed options behind a seam and leave the choice to configuration.

Building against a specification is the cheapest known way to find out where that
specification is silent, because a compiler will not accept a silence. Roughly seventy
places did not survive contact with a build. They are catalogued below.

The prototype's own two write-ups are the long form:

- `PROTOTYPE_REPORT.md` §5 — the defect list, grouped by change
- `README.md` §8 and §10.5 — the model's and the integration's own ambiguity tables

The identifiers are the prototype's, carried forward unchanged so that a reader holding
either document can find the corresponding corpus entry. The two documents overlap: the
same defect sometimes appears in both under different ids, and the aliases are recorded
below.

## How a defect was disposed of

The cardinal rule of this corpus applies to prototype feedback exactly as it applies to
everything else: **a defect is never folded in by deciding something.** Four dispositions
were available, and only the first two touch a requirement.

| Disposition | What it means | When it was used |
|---|---|---|
| **Sharpened** | An existing requirement now demands that something be _defined_, without saying what it must be | The requirement was under-specified in a way an implementer cannot route around |
| **Added** | A new requirement states something several existing requirements already depend on | Nobody could dispute the requirement's existence, only its shape |
| **Open question** | Recorded in the owning change's `design.md`, answered nowhere | **The default.** Anything whose repair would require picking a value, unit, direction, default or mechanism |
| **Rejected** | Reported, examined, found not to be a corpus defect | Three cases, each with its reason recorded below |

Nothing on this page resolves a design decision. Where the prototype had to behave _some_
way in order to compile, the behaviour it chose is recorded in the relevant `design.md`
under a "Prototype:" note — as **evidence of the cost of the silence, never as a
recommendation**. A maintainer answering one of these questions should feel no pull
towards what the prototype happened to do.

## The catalogue

Grouped by the change that owns the answer. "Landed" names the file that now carries it;
section numbers are the `design.md` sections written for this pass.

### `define-participant-model`

| Id | What the build hit | Disposition | Landed |
|---|---|---|---|
| A1 | The level gate is scoped to Simple consumers; the taxonomy it cites scopes it per consumer, so `never` — "leave this alone" — cannot be said about a Batch or Controllable device at all | Open question | design.md §5.7 |
| A2 | A deadline reads as recurring in one example and absolute in another; not stated which, or both | Sharpened (_Demand declaration_) + open question | spec.md; design.md §5.3 |
| A3 | Two of the requirement's own three examples — a state-of-charge target, a runtime-in-a-window — are not expressible in the shape it defines | Open question | design.md §5.4 |
| A4 | Priority has no direction, no default and no tie-break; the source note "lower runs first, higher wins on conflict" points both ways in one sentence | Sharpened (_Priority_) + open question | spec.md; design.md §5.2 |
| A5 | May a non-battery provider be controllable, and must a controllable provider declare a clamp? | Open question | design.md §5.8 |
| A6 | The level→mode mapping must exist "without translation logic in user rules" and is specified for no mode count other than four | Open question | design.md §5.10 |
| A7 | Is a load curve a property of the demand or of the programme? Both readings are in the text | Open question | design.md §5.5; `add-named-profiles` design.md |
| A8 | Load curves have no bound, no sample spacing, and no statement of whether samples are instantaneous power or interval averages | Open question | design.md §5.6; `add-named-profiles` design.md |
| A9 | The Controllable class is called a power profile and its own scenario is in amperes; provider clamps are power | Open question | design.md §5.9 |
| A10 | `EnergyProvider` names both a participant role here and lsiepel's data-contributor role on the extension surface — a second collision on top of the one Kai already raised | Open question (naming pass) | design.md §5.16; `define-extension-points` design.md §2 |
| A11 | Level names differ between the spec and the taxonomy it cites | Open question | `define-energy-levels` design.md §1 |
| A12 | Participant identity is undefined exactly where the registry merges statements from several sources about "the same participant" | **Added** (_Participant identity_) + open question | spec.md; design.md §5.1 |
| E2 | `engine-contract`'s "Phase-aware headroom" scenario opens "a consumer that declares which phase(s) it draws on", and nothing in the participant model can declare a phase | **Added** (_Phase declaration_) + open question | spec.md; design.md §5.12; `define-engine-contract` design.md §20 |
| E3 | Per-phase headroom also needs per-phase _measurement_; providers declare one aggregate power Item, so the load on a phase is not observable | Open question | design.md §5.12; `define-engine-contract` design.md §20 |
| E4 | A Simple consumer has no rated power, so a budget check has to reuse the surplus `onThreshold` as one | **Added** (_Consumer power figure_ — the model must _name_ the value the floor books; whether every class has one stays open) + open question | spec.md; design.md §5.13 |
| E6 | The sign convention is fixed for the grid reading only; a battery reading and a controllable setpoint both need one and have none | Sharpened (_Provider roles_) + open question | spec.md; design.md §5.11 |
| N1 | The per-consumer measurement Item is used by three requirements and declared by none | Open question | design.md §5.14 |
| N2 | `minOn` "doubles as catch-up time after a forced restart" — nothing defines a forced restart or how an engine would recognise one | Open question (for @mstormi) | design.md §5.15 |

### `define-engine-contract`

| Id | What the build hit | Disposition | Landed |
|---|---|---|---|
| E1 | "The higher-priority decision is applied" never says whether higher priority is a larger or a smaller number — A4 in the engine's public API | Open question (cross-reference) | design.md §19 |
| E5 | A ModeControllable mode carries no power semantics, so no budget can be enforced against a mode change | Open question | design.md §10; `define-grid-constraints` design.md §2 |
| E7 | "Surplus" is defined nowhere beyond being derived from grid export — battery, curtailed PV and a managed load's own draw are all unresolved | Open question | design.md §11 |
| E8 | "Unacknowledged" has no window, no comparison tolerance and no rule for a changed command; a never-echoing device blocks forever | Sharpened (_Acknowledgement-aware actuation_ — the window needs a defined length _and_ a defined behaviour on expiry; what expiry does is not chosen) + open question | spec.md; design.md §3 |
| E9 | The shadow default and the master-stop default are the same statement if the two are one control, which a design note offers as a live option | Open question | design.md §2 |
| E10 | Does disengaging the stop resume as if it never stopped, or from a clean slate? | Open question | design.md §7 |
| E11 | No default evaluation cadence and no event story | **Rejected** — already recorded | design.md §1 |
| E12 | "Goes stale" and "conservative safe state" both have no operational meaning — no age, no floor | Open question | `define-extension-points` design.md §5.7; `define-forecast-providers` design.md §3 |
| E13 | "Trimmed or deferred" never says which, when; a Switch cannot be trimmed at all | Sharpened (_Electrical limits outrank optimization_ — a defined rule settles what happens to each affected load; trim and defer are not fixed as the whole set) + open question | spec.md; design.md §8 |
| E14 | Enforce the budget against estimates or against measurements, and how is the error reconciled? | Open question | design.md §9; `define-grid-constraints` design.md §3 |
| E15 | Nothing says what happens to a decision naming an unknown participant | Open question | design.md §13 |
| E16 | `fixtures/README.md` attributes the boiler vector to conflict resolution, but reproducing it needs look-ahead replanning that no requirement describes | Open question + fixture note | design.md §17; `define-grid-constraints` design.md §4; `fixtures/README.md` |
| E17 | "The same consistent context snapshot" and nothing more; retention beyond a cycle unstated | Open question | design.md §4 |
| E18 | Shadow output has no surface and no rate, yet a requirement's own scenario asks a user to compare | Open question | design.md §14; `define-energy-ui` design.md §5 |
| I6 | The algorithm plane has no ownership statement; `registerAlgorithm` replaces silently | Open question | design.md §15; `define-optimization-objectives` design.md §6 |
| N3 | The protection durations have no stated source of elapsed time, which decides whether protections survive a restart | Open question | design.md §4 |
| N4 | Nothing says whether the master stop halts _evaluation_ or only dispatch — a stopped engine still runs third-party code every tick | Open question | design.md §6 |
| N5 | Nothing says what a protection should do while the stop is engaged: device safety and operator safety point opposite ways | Open question | design.md §5 |
| N6 | The corpus has no vocabulary for what became of a decision — applied, superseded, deferred, suppressed, withheld | Open question | design.md §12; `define-energy-ui` design.md §5 |
| N7 | "Engine-owned" is enumerated once and the enumeration is incomplete; it is the security boundary of the whole extension story | Sharpened (_Replaceable algorithm, scripts first-class_ — one enumeration, stated in one place and stated to be complete; membership stays open) + open question | spec.md; design.md §16 |

### `define-energy-levels`

| Id | What the build hit | Disposition | Landed |
|---|---|---|---|
| L1 | The level→number mapping exists only in a CSV, and it runs opposite to priority's — wave 1 ships two ordinal scales pointing opposite ways, neither stated in prose | Sharpened (_Four-level scale_ — a stated encoding governs any numeric exchange; whether one is fixed centrally at all stays open) + open question | spec.md; design.md §2 |
| L2 | The SG-ready offset is unstated; modes are 1–4 and level codes 0–3 | Open question | design.md §3 |
| L3 | No tie-break is defined; `fixtures/README.md` itself says one is needed | Sharpened (_Level derivation_) + open question | spec.md; design.md §6 |
| L4 | "Hours" versus "slots" — on 15-minute data "4 hours" is either 4 or 16 | Open question | design.md §7 |
| L5 | Counts that do not fit a partial day are undefined, and partial days are routine | Open question | design.md §6 |
| L6 | Band precedence is unspecified even without overflow | Open question | design.md §6 |
| L7 | "Escalates" has no magnitude, no threshold and no target level, so the requirement is inert until configured | Open question | design.md §8 |
| L8 | No hysteresis, so the published level chatters minute-to-minute on a partly cloudy day | Open question | design.md §8 |
| L9 | The current level outside the plan is undefined | Open question | design.md §9 |
| L11 | Level names differ between the spec and the taxonomy — same as A11 | Open question | design.md §1 |
| L12 | Percentile and fixed-count derivation are _identical_ on tie-free data, proved on the corpus fixture; the open question is tie handling plus adaptivity | Open question + task 2.1 reframed | design.md §5; tasks.md 2.1 |
| L13 | "Consecutive" is undefined on real data — may a series be non-uniform or gapped? | Open question (part already settled by `price-data` _Time resolution_) | design.md §11; `define-price-providers` design.md |
| L14 | Comparing windows of unequal covered time is undefined | Open question | design.md §11; `define-price-providers` design.md |
| L15 | Window start granularity is unstated | Open question | design.md §11 |
| L16 | Selection ignores the load's own shape; "cheapest N slots" is correct only for a flat load | Open question | design.md §11; `define-price-providers` design.md |
| L17 | Under-supply behaviour is unstated and user-visible, and the two candidate behaviours differ | Open question | design.md §11 |
| L19 | Price semantics — currency, taxes, fees, and above all negative prices, where "blocked = most expensive" becomes strange | Open question (feed-in half already covered) | design.md §13; `define-price-providers` design.md |
| L20 / L21 | Seasonal boundaries, time zone, and which local date names a delivery day that straddles two | Open question | design.md §12 |
| L22 | Rule-updatable hour counts have no re-derivation trigger | Open question | design.md §10; tasks.md 2.2 |
| L23 | The plan is a future-timestamped TimeSeries and the current level is an Item — one item or two, and what may be published under shadow | Open question | design.md §14 |
| N8 | This change was the only wave-1 change with no `design.md`, so its open questions lived where no reader was pointed | **Fixed** — `design.md` created | design.md |

### `define-extension-points`

| Id | What the build hit | Disposition | Landed |
|---|---|---|---|
| B1 | The `energy:` namespace is named by a requirement and specified nowhere — every key in the prototype's parser is invention the corpus cannot validate | Open question (for @kaikreuzer) | design.md §5.1 |
| B2 | The key vocabulary cannot express three of its own requirements — no consumer min/max, no `consecutive`, no level gate | **Added** (_Expressive declaration surface_) | spec.md |
| B3 | Provider `price` and `schedule` keys have nowhere to land in wave 1: accept and ignore, or reject? | Open question | design.md §5.3; `define-price-providers` design.md |
| B4 | Units are unstated and the vocabulary is internally inconsistent — some keys bake the unit into the name, others must carry it in the value | **Added** (as a qualifier on _Expressive declaration surface_ — the unit is fixed by a stated rule, which the declaration may carry or the specification may mandate) + open question | spec.md; design.md §5.2 |
| B5 | Deadlines are unreachable in their stated forms — A3 made concrete at the mechanism boundary | Open question | `define-participant-model` design.md §5.3 |
| B6 | No default profile class, and defaulting is safety-relevant: an unrecognised profile treated as Simple lets the engine switch a Batch programme | Open question | design.md §5.4 |
| B7 | No default priority, scale or direction — same as A4 | Open question | `define-participant-model` design.md §5.2 |
| B8 | Participant identity is undefined, and it is what makes cross-mechanism precedence work — same as A12 | **Added** (in `define-participant-model`) | `define-participant-model` spec.md |
| B9 | "Explicit declaration wins" is scoped to _discovery_; it says nothing about explicit versus add-on-contributed, which is the conflict wave 1 actually produces | **Added** (_Deterministic resolution between contributors_) | spec.md; `discover-participants-from-model` design.md §5 |
| B10 | "Multiple contributors, user selection" names no selection mechanism and no home for it | **Rejected** — already recorded | design.md §4, §5.11 |
| B11 | Failure semantics for a bad declaration are unspecified, and there is no notion of a declaration in error and no surface to show one | Open question | design.md §5.5; `define-energy-ui` design.md §5 |
| B12 | Degradation has no reporting surface, and the requirement covers data contributors but not a declaration contributor vanishing | Open question | design.md §5.6; `define-forecast-providers` design.md §3; `define-energy-ui` design.md §5 |
| B13 | openHAB's `AbstractRegistry` is registration-order dependent — the exact opposite of "explicit wins" | Open question (for the core registry maintainers) | design.md §5.8 |
| B14 | Script contributors have no unload hook, so the two contribution paths have genuinely different lifecycle guarantees that the corpus treats as equivalent | Open question | design.md §5.9 |
| B15 | Nothing says whether a declaration may name Items that do not exist | Open question | design.md §5.10 |
| E12 | Staleness has no age and the safe state has no floor, so the safety branch is inert by default in any faithful implementation | Open question | design.md §5.7 |
| N9 | Actuation adapters have no selection story — the single most consequential choice in the system | Open question | design.md §5.11 |
| N10 | "Core-shipped defaults on equal terms" has no observable definition | Open question | design.md §5.12; `define-optimization-objectives` design.md §6 |

### Cross-cutting

| Id | What the build hit | Disposition | Landed |
|---|---|---|---|
| I1 | _Level derivation_'s "PV escalation" scenario and _Central periodic evaluation_ are mutually recursive as written, and the corpus never says whether the site level is an engine **input** or a **product** of its own readings | Open question, recorded on both sides | `define-energy-levels` design.md §15; `define-engine-contract` design.md §4 |
| I2 | The level plane publishes Items; the engine consumes a level. Under shadow-only nothing may be published, so the only possible coupling is in-process — and the corpus describes none | Open question, recorded on both sides | `define-energy-levels` design.md §14; `define-engine-contract` design.md §18 |
| I3 | The seasonal derivation cannot be offered as configuration, because the corpus defines no season grammar, no zone and no straddling-day rule | Open question | `define-energy-levels` design.md §12 |
| I4 | Nothing says at which granularity a level applies; site-global versus per-domain is a signature, not a preference | Open question | `define-energy-levels` design.md §4 |
| I5 | `package-info.java` with `@NonNullByDefault` is against the codebase's own convention | **Rejected** — not a corpus defect | this page |
| L18 | The boiler fixture does not exercise the power budget it is filed under | Fixture note + open question | `fixtures/README.md`; `define-grid-constraints` design.md §4 |
| — | The fixture CSVs carry slot _starts_ only, so the last slot's end must be inferred | **Fixed** — slot width stated | `fixtures/README.md` |

## Aliases

The same defect reached the corpus under more than one identifier, because the model
report and the integration report were written at different times. A reviewer chasing one
should know about the other.

- **A4 ≡ E1 ≡ B7** — priority's direction, default and tie-break, seen from the model, the
  engine's public API and the declaration mechanism
- **A12 ≡ B8** — participant identity
- **A11 ≡ L11** — the level names
- **A3 ≡ B5** — the demand shape, and the same gap at the mechanism boundary
- **A6 ≡ L2** — the level→mode mapping and the SG-ready offset
- **E16 ≈ L18** — the boiler fixture: E16 is its attribution, L18 is its discriminating
  power. Both are true of the same file and neither implies the other
- **F13 → A1** — the prototype report cites "A1, F13" against the level gate and defines
  F13 nowhere: it appears twice as a bare cross-reference and in neither document's tables.
  A reviewer chasing it will find nothing, so it is carried here as an alias of A1 and
  `define-participant-model` design.md §5.7 says so inline

## The three rejected reports

Rejecting a report is a result, not an omission, and each is recorded so it is not
re-reported.

**E11 — no default cadence and no event story.** Not a new defect.
`define-engine-contract`'s design notes already carry the fixed-tick-versus-tick-plus-events
question and the cadence, and the requirement is deliberately mechanism-neutral about
both. The prototype's 60 s is a build artefact of having to pick something to run, not
evidence that the corpus is silent. Recorded as a re-report of an already-open question.

**B10 — "multiple contributors, user selection" names no mechanism.**
`define-extension-points` design.md §4 already states in as many words that
selection and composition need a home, and lists the candidates. The prototype's
configuration-PID `sources` + `precedence` is one implementation of §4's open question,
which makes it evidence for that question rather than a new one.

**I5 — `package-info.java` and `@NonNullByDefault`.** Not a defect in this corpus at all.
It is a conflict between the prototype's build brief and openhab-core's actual convention:
the tree contains zero `package-info.java` files outside the prototype bundle and annotates
every type instead, and the reactor's `-warn:+nullAnnotRedundant` makes the package-level
default emit a redundancy warning per class. This corpus specifies requirements, not Java
conventions; the resolution belongs to the prototype and to
[`REVIEW_CHECKLIST.md`](REVIEW_CHECKLIST.md), and nothing here needs to change. Recorded
because a future build will hit it again.

## One place a choice between remedies was made

E16 offered two: correct the boiler fixture's attribution, or add a replanning requirement
to `engine-contract` that would make the attribution true. This pass took the first,
because the second would answer the open question instead of recording it. It is the only
place in the pass where two offered remedies were weighed against each other rather than
both being recorded, it is named as such in
[`../fixtures/README.md`](../fixtures/README.md), and it is cheap for a maintainer to
reverse: reversing it means adding that requirement and restoring the table row, not
undoing anything.

## What this page does not do

It does not resolve anything. No direction, default, unit, sign, dimension, mechanism,
age, threshold or reporting surface was chosen anywhere in this pass. Where a requirement
was sharpened it now demands that something be _stated_ — and the statement itself is
still owed by a maintainer, in the `design.md` section the table names.

Two of the sharpened requirements are worth calling out because they read like decisions
and are not:

- _Priority_ now requires a total, deterministic order **whose direction is stated**. It
  does not say which direction. The fixtures imply one and the requirement's own source
  note implies the other; that contradiction is the point of the question, not something
  the sharpening settled.
- _Four-level scale_ now requires any numeric exchange of a level to be **governed by a
  stated encoding**. It does not say what the encoding is, nor even that one must be fixed
  centrally — `define-energy-levels` design.md §2 keeps "core numbers nothing and each
  mapping is stated by whoever exchanges it" live. The only encoding that exists anywhere
  today is the one `expected-planned-levels.csv` happens to bind, and it is recorded in
  that same section **as evidence, not as the answer**.

### Where the first draft of this pass ran over the line, and what was changed

An adversarial re-read of the pass looked for exactly one thing: a requirement that had
quietly picked something. It found none that picked a value, but six that **narrowed the
option set** — five of them by foreclosing an option the corpus's own `design.md` still
listed as live, which is the more dangerous shape, because the change then reads as
internally consistent to anyone who does not cross-check. All six were reworded rather than
defended. They are recorded here because a foreclosed option leaves no trace once it is
gone:

- _Consumer power figure_ opened "**Every** consumer SHALL carry a power figure", which
  killed `engine-contract` §10's live reading that mode changes are exempt from the budget,
  and contradicted the class definition of ModeControllable ("no power setpoint"). It now
  says the model must **name** which declared value is booked; whether every class has one
  is an open question in `define-participant-model` design.md §5.13.
- _Acknowledgement-aware actuation_ answered what happens when the window expires
  (actuation resumes), which is a behavioural pick — the safety-conservative reading is
  that an unobservable device is quarantined instead. It now requires the expiry behaviour
  to be **defined**, and `define-engine-contract` design.md §3 carries both readings plus a
  note that the never-give-up reading remains a maintainer's to choose.
- _Expressive declaration surface_'s unit scenario required the unit to travel **in the
  declaration**, which ruled out a canonical unit fixed by the specification — the shape
  openHAB core uses most. The rule now merely has to be stated; the third shape is in
  `define-extension-points` design.md §5.2.
- _Electrical limits outrank optimization_ promoted "trimmed or deferred" from an
  illustration into a closed two-valued outcome set, which its own design note had already
  shown to be one short. The rule now settles **what happens** to each affected load.
- _Priority_'s tie-break scenario demanded the same consumer win on every evaluation, which
  outlaws round-robin or fair-share rotation between equal priorities — a normal EMS
  behaviour. The scenario now only excludes incidental enumeration order, and whether the
  tie-break may be stateful is open in `define-participant-model` design.md §5.2.
- _Four-level scale_ presumed a normative numeric encoding exists; see above.

A seventh, milder one: _Replaceable algorithm_ had adopted the prototype's own proposed
wording ("exhaustive and authoritative") verbatim, which is both self-contradictory next to
"covering at least" and the implementer writing the specification's text for it. The
requirement now keeps its original engine-owned clause unchanged and adds only that the
enumeration is stated in one place and stated to be complete.

A maintainer who reads this page and concludes "so the prototype already decided" has read
it backwards.
