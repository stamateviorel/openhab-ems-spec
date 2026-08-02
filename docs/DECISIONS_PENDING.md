# Pending decisions — owner's answer pass

> **HISTORICAL RECORD — these questions have been answered. This page is no longer a
> to-do list.**
>
> It is kept because it is the **reasoning behind** the decisions: how each question was
> posed, what options were put, what evidence each recommendation rested on and what its
> tier was. None of that is reproduced in the answers, and a maintainer overturning a
> decision needs it.
>
> **The answers are in [`OWNER_DECISIONS.md`](OWNER_DECISIONS.md)** — 21 rows, D1–D20 plus
> the Part D remainder, answered by the owner of the reference implementation on
> 2026-08-02. They are **owner decisions in a reference implementation, not thread
> consensus**, and every requirement they changed says so on its `Source:` line.
>
> Read this page against that one, because they do not always agree:
>
> - **Three decisions departed from the recommendation below.** **D1** took the master stop
>   to halt evaluation as well as dispatch, where A3 recommended dispatch-only. **D11** made
>   a positive battery reading mean _charging_, where A5 recommended positive = discharge.
>   **D10** made surplus include reclaimable battery charging, where **the same A5**
>   recommended `surplus` = grid export only plus a second named `allocatableBudget`, and
>   said in terms that a charging battery is _not_ surplus. A5 is therefore one
>   recommendation the owner departed from **twice**, on both of its halves. In each case
>   the recommendation here is the preserved alternative, and the change's `design.md`
>   carries it with its argument — A5's sign half in
>   `define-participant-model` design.md §5.11, its surplus half in
>   `define-engine-contract` design.md §11.
> - **One recommendation was transcribed with a sign error and is corrected in the corpus.**
>   A14 below says the shed threshold is `min(month-to-date billed peak, minimum billable
>   demand)`, copied from the reference's `Math.min(mtdPeak, −minBillableW)` — where **both
>   terms are negative** under grid-positive-is-export. Restated in the unsigned watts the
>   requirement uses, that selector is `max`, and `define-grid-constraints` says `max`.
> - **Part A's own retraction stands**: the claim under A4 that `buildEngineDispatchSet`
>   proved lower-wins was wrong and was struck; the evidence that survives is
>   `sortByPriority` and the fixtures.
> - **Several recommendations were never put to the owner** and are therefore neither
>   accepted nor rejected — they are live options a maintainer may still take. The largest
>   are A11's band-derivation proposal (only its encoding half became D12, so the tie-break
>   and the band mechanics stay open), A7's changed-command rule, and the retainable-snapshot
>   proposal in A3's neighbourhood. Each is flagged where it lands in the relevant
>   `design.md`.
>
> Nothing below has been edited to match the answers. Where this page and the corpus differ,
> the corpus is normative and this page records what was recommended and why.

79 numbered open questions across 11 design files, reduced to **15 load-bearing decisions, 30 parameters, 22 already-answered, and 5 that are genuinely yours**. Duplicates are merged (the same decision was often filed on both sides of a seam), and everything traceable to openHAB core, to the emsmanager binding, to storm.house/masipila, or to the wave-1 prototype carries that evidence inline.

## How it was meant to be used

_Preserved as written, in the imperative it was drafted in — this is what was put to the
owner on 2026-08-02, not instructions to a reader today._

**The default answer is "take the recommendation."** Every recommendation is grounded, and the evidence tier is stated so you can see how hard it would be to defend in review: `[C]` openhab-core convention (what a maintainer weights highest after correctness), `[P]` your own production binding, `[T]` storm.house or masipila, `[X]` what the wave-1 prototype had to do to compile, `[J]` engineering judgement and nothing else. Read Part B, Part C and Part D once, approve or strike, and do not go back to them.

**Spend your attention on Part A.** Those 15 change the shape of the code or the model — getting one wrong is a rewrite, not a config edit. Each carries the options, the recommendation, one line of evidence, and one line on what breaks if you choose otherwise. Answer them in this order, because they have dependencies: A1 → A2 → A11 → A4 → A9 → A15 → A6 → A5 → A8, then the rest in any order. Eight of the fifteen ask you to change something your live binding does today; those are collected under "Departures" immediately after Part A so you see the bill in one place.

| Part | Contents | Count |
| --- | --- | --- |
| A | Load-bearing decisions | 15 |
| B | Parameters and defaults | 30 |
| C | Already answered — do not read | 22 |
| D | No basis to recommend — yours | 5 |

## Part A — the load-bearing decisions

### A1 · The declaration and contribution plane

**Q.** How does a participant get declared, what is its identity, and who wins when two mechanisms describe the same device?

| Option | Consequence |
| --- | --- |
| Item metadata only | No add-on can contribute; wave-3 profile payloads have nowhere to live |
| Programmatic SPI only | The "no Thing required" user-facing path disappears |
| Both, unified behind one SPI | Nothing above the seam knows which mechanism produced a participant |

**Recommendation.** Both, behind a single `Provider<EnergyParticipant>` SPI: `energy:` item metadata for users, an OSGi whiteboard for add-ons and scripts. **Identity is the Item name**, optionally overridden by an explicit `id`. A second declaration of a present identity is **a further statement about the same participant**, never a second participant and never a rejection. Precedence, stated once and used everywhere: **explicit metadata > contributed (ties by `service.ranking`) > discovered proposal**. Same rule for contributed algorithms, keyed on a `contributorId` the contributor supplies (that is what tells a script reload from a hijack). One exception, deliberate: the **write-side actuation sink is chosen by explicit configuration naming it**, never by ranking — a per-participant override is allowed.

Evidence — `[C]` `MetadataKey` is already `(namespace, itemName)`, so a binding can address a user-declared participant with no coordination; `StateDescriptionServiceImpl` is core's ranked-aggregator pattern for read-side precedence, and `defaultPersistence` is core's convention for naming a writer. `[P]` `MetadataParticipantScanner` merges whiteboard participants with tagged items, "metadata winning on duplicate ids", and has for months.

If otherwise — registry-generated ids make the two-mechanisms-one-device case unimplementable; duplicate-as-conflict turns "the user overrides the add-on's guess" into an error the user must hand-resolve; a ranked sink lets a site's writer be decided by a binding race.

Retires — PM-1, PM-5.1, EP-1, EP-3, EP-5.11, EP-4(i), M3, DP-5a, EC-15(a–c).

### A2 · Priority: direction, default, tie-break, multiplicity

**Q.** Does a lower priority number mean served-first or win-conflicts, what does an undeclared consumer get, and may several algorithms run at once?

| Option | Consequence |
| --- | --- |
| Lower = better | Matches every fixture and the allocation order that is running |
| Higher = better | Inverts every fixture in the corpus |
| No stated default | An undeclared consumer's placement is implementation-defined, which the requirement forbids |

**Recommendation.** **Lower number = better priority, in both senses — served first and wins conflicts.** Default **100** for a consumer that declares none. Tie-break: participant id ascending, stateless. **Several algorithms may be active at once**; conflicts between them resolve by this same rule. Rotation and fair-share are legitimate but belong to an algorithm that owns a priority, not to the ordering primitive. Do **not** force priority and the level scale to point the same way — priority is a rank (lower is better, like every rank), level is a magnitude (higher = more available); state each in prose.

Evidence — `[P]` `sortByPriority` sorts ascending and serves in that order, ties by id, javadoc "lower = served first — storm.house's `Prioritaet`"; `DEFAULT_PRIORITY = 100`. `[X]` the fixtures agree. `[X]` the prototype ships `DefaultSurplusAlgorithm` and `DeviceProtectionAlgorithm` together and one of its tests depends on both being active. `[C]` note for reviewers: OSGi `service.ranking` is higher-wins, but that governs _which provider is used_ (A1's precedence), not serving order — two orderings, two conventions.

If otherwise — higher-wins inverts the corpus; a stateful tie-break makes overload resolution unreproducible from a snapshot and kills fixture conformance; single-algorithm means a user script _replaces_ device protections instead of coexisting with them.

Retires — PM-5.2, EC-19, GC-5, EL-2c, EC-15(d).

**Caution, and it is the one real cost in this decision.** Your live `Controller` contract says both directions in one sentence — "lower runs first, higher wins on conflict" — and `EmsManagerBridgeHandler` implements it (ascending sort, then last write lands). Adopting lower-wins-conflict means reconciling that in the binding. The earlier claim that `buildEngineDispatchSet` proves lower-wins is **wrong** and has been struck: that method reads no priority number at all, it retains legacy requests during cutover.

### A3 · Shadow, the master stop, and the safety ladder

**Q.** Are shadow and stop one control or two, does the stop halt evaluation, and what does the stop outrank?

| Option | Consequence |
| --- | --- |
| Unified shadow+stop | You can never say "stop writing but keep showing me what you would do" |
| Two controls | Diagnosis survives the moment you hit stop |
| Stop halts evaluation | A stopped engine is a black box exactly when you need to know why you stopped it |

**Recommendation.** **Two controls.** Stop is a global boolean; shadow is global **plus** per-algorithm and per-participant sets (empty = global). Fresh install: **shadowed, stop disengaged**. The stop halts **dispatch only** — evaluation continues, and `stopped` is exposed in the context so a well-behaved contributed algorithm can no-op. Confirm the ladder **electrical limits > device protections > level gates > optimization**, and place the stop **above all of it, protections included** — device safety reverts to whoever owned it before the EMS. Mandatory rider: engaging the stop must be _reported_, not just logged, as leaving protections unenforced.

Evidence — `[P]` the binding ships exactly this two-control shape (bridge `shadowMode=true` default plus a per-controller `shadowMode()`), used for a staged cutover; `constrainOnOff`'s javadoc states the ladder outright — "the fuse outranks the freezer, deliberately" — and `applyBreakerGate` overrides every positive action regardless of origin. `[J]` physics agrees: exceeding the fuse trips the board including the fridge. A kill switch with exceptions is not a kill switch.

If otherwise — protections above limits means a config file can force the engine past the fuse; protections surviving the stop means the operator cannot make the engine stop writing; a unified control makes shadow's default collide with the stop's.

Retires — EC-2, EC-5, EC-6, E9, N4, N5.

### A4 · The cycle snapshot, and where the site level comes from

**Q.** What does one evaluation cycle see, may an algorithm keep it, and is the site energy level an input to the cycle or a product of it?

| Option | Consequence |
| --- | --- |
| Level is an input the engine reads | Level and snapshot can disagree inside one cycle |
| Level is a product of the cycle | The engine cannot read the grid twice and get two answers |
| Engine reads the published level Item | Impossible under shadow mode, which writes nothing |

**Recommendation.** The snapshot is an **immutable value, explicitly retainable**, carrying its own `timestamp()`; algorithms compare it and never read it as "now". Minimum content: cycle instant; one reading per declared provider by role plus derived surplus; per participant — current state, **the age of that state**, declared phases, its bookable power figure, whether a command is outstanding, whether its measurement is stale; the site energy level; the declared electrical limits and the measured load they are checked against. Extend by adding accessors, never by reordering. **The current level is a product of the cycle**, derived from the slow price-derived plan (an input) plus this cycle's own surplus. The engine gets it through an **in-process seam**; the Item is a published view, never the transport. **Two Items**, planned schedule and current level, not one.

Evidence — `[C]` core's own value types are immutable and freely retained, and cross-plane data inside core travels by OSGi service (`StateDescriptionService`, `TimeZoneProvider`, `PersistenceServiceRegistry`), never round-tripped through the item registry. `[C]` decisive on the two Items: `PersistenceManagerImpl.scheduleNextForecastForItem()` makes a published TimeSeries _become_ the Item's state at each timestamp, so on one shared Item core itself would overwrite your surplus-escalated current level at every slot boundary. `[P]` `ShadowEmsRunner` computes `siteLevel` from the same `surplus` variable the plan was built from, then publishes it.

If otherwise — level-as-input lets the engine gate on a surplus figure it is not itself using (invisible in tests, appears on partly-cloudy days); Item-as-transport makes a renamed item silently disable level gating and makes shadow mode impossible; omitting the state age forces protections to read items live mid-cycle, defeating one-snapshot-per-cycle.

Retires — EC-4a, EC-4b, EC-4d, EC-18, EL-14a, EL-14b, EL-15, I1, I2.

### A5 · Sign conventions, and surplus versus allocatable budget

**Q.** Which direction is positive for each role, and what exactly is "surplus"?

| Option | Consequence |
| --- | --- |
| Surplus = grid export only | Every surplus-driven load oscillates at the cycle rate unless hysteresis is hand-tuned |
| Surplus = export + steered loads' own draw | Fixes the recursion, but double-counts if used as the raw reading |
| Two named quantities | Both uses served, neither ambiguous |

**Recommendation.** One sign rule, stated once: **positive = power flowing in the direction the role names, toward the site boundary.** Grid positive = export; PV positive = production; **battery positive = discharge**; consumer positive = consumption. A setpoint uses the same convention as its reading, so a battery commanded −3000 W is charging. Corollary that makes it self-checking: `Σ providers − Σ consumers = 0` at the boundary. Then **name two quantities, not one**: `surplus` = grid export only, instantaneous, from the grid provider's signed reading; `allocatableBudget` = `surplus + Σ draw of participants the engine is currently steering`, which is what a surplus-allocation algorithm consumes. A charging battery is **not** surplus — it is a participant the same allocator serves by priority. Curtailed PV is **not** surplus — it is capacity, not power, and a load switched on against it imports. The snapshot carries both an instantaneous reading and a short average; the limit floor uses instantaneous, opportunistic dispatch uses the average.

Evidence — `[P]` `SetpointRequest.Kind.WATTS_BATTERY` states "Positive = discharge, negative = charge" and has driven a real hybrid inverter for months. `[P]` both quantities exist in your binding and were being cited as if they were one: `surplusFromGridNet(g) = max(0, g)` is the raw reading, `EvCoordinatorController`'s `budget = (totalDrawW + grid − gridSafetyMarginW) / active` is the add-back. They are different quantities; the spec needs both names.

If otherwise — ad-hoc per-role signs mean every add-on contributing a battery reading guesses, and the failure mode is a 100 % error that looks plausible on a chart; one conflated "surplus" either oscillates or double-counts the grid reading depending on which definition an implementer picks.

Retires — PM-5.11, EC-11, EL-8c, OO-6c, E6, E7.

Worth recording honestly in the rationale: your own binding is internally inconsistent on the battery _reading_ (`Site`/`EnergyReading` say `+ = charging`, `EnergyContext` says `+ = discharging`) and it never bit, because nothing in it computes with the battery reading. That is the argument for fixing the convention now rather than after something does.

### A6 · Power figures: what is booked, and measured versus estimated

**Q.** Which number does the limit floor book per class, does the planner use a different one, and may a consumer declare its own measurement?

| Option | Consequence |
| --- | --- |
| Reuse the on-threshold as the rating | Silently over-books, always in one direction — thresholds are set with margin |
| A declared rating per class | Correct, but must not invalidate existing declarations |
| Carry an error term / reserve | Introduces a tuning constant nobody can derive; floor output depends on history |

**Recommendation.** **A consumer (and a provider) may declare an Item carrying its live power**, and the engine prefers a fresh measurement over an estimate everywhere except admitting a load that is not yet running. Per class: **Controllable** → declared `max` (its commanded value once commanded); **Batch** → `ratedW` × curve, peak for admission; **Simple** → a declared **`ratedPower`, optional with a reported inference** — fall back to the on-threshold only when it is absent, and surface that as a declaration gap, never a rejection; **ModeControllable** → **none**, mode changes are exempt from the _planner's_ budget, with an _optional_ per-mode declared draw that upgrades a mode load into it. Planner books declared figures; the **runtime floor books `max(declared, measured)` for anything already running**, declared for anything not started. **Re-book from scratch every cycle — no carried reserve.** Disposition on a shortfall, worst-priority first, over exactly four outcomes (trim, defer, leave-untouched, refuse-with-reason): Controllable → trim to the largest value ≥ its declared minimum, else defer; Simple and not-yet-started Batch → defer; **Batch already running → leave untouched and re-book its draw as uncontrollable load the floor trims others against**; ModeControllable → per the exemption. Untrimmability never changes shed order — priority orders, shape only decides how.

Evidence — `[P]` `measure` exists on any consumer today and the docs state the principle: "running-detection and delivered-energy metering trust measurement over commanded state — the autonomy principle"; the taxonomy says outright "the planner must plan on measured power, not commanded values"; `PowerProfile.Batch` already carries `ratedW` separately from any switching threshold, so only Simple has the bug E4 names. `[P]` `applyBreakerGate` is exactly "positive actions overridden in priority order" and `planSurplusDispatch` is exactly "trim to fit within [min,max], else off". `[P]` the mode exemption is a limitation you already name honestly: "mode loads' draw is unknown to the surplus budget"; a declared per-mode number would be fiction for SG-ready, where mode 3 draws whatever the heat pump wants. `[X]` the prototype found `measureItemName` load-bearing in three places with no requirement behind it. Stability comes from smoothing the input (EWMA τ=30 s + a 5-min rolling average), not from a reserve.

If otherwise — on-threshold-as-rating over-books silently and refuses loads that would have fit; making `ratedPower` mandatory invalidates your live Simple declarations (and, combined with the reject-on-invalid rule, would break them at parse time); a carried reserve falsifies "the same inputs always resolve one overload the same way"; no per-consumer measurement leaves the surplus recursion with only hand-tuned hysteresis and makes "commands are envelopes" unverifiable.

Retires — PM-5.13, PM-5.14, GC-1, GC-2, EC-9, EC-10, EC-8, E4, E5, E14, N1.

### A7 · Acknowledgement handling

**Q.** Does the ACK window live in the engine or in per-device adapters, and what happens when it expires?

| Option | Consequence |
| --- | --- |
| Adapter-owned | Unenforceable on a site with no adapter; every adapter reinvents it |
| Engine-owned, expiry lapses | A silently-reset device gets corrected next cycle |
| Engine-owned, expiry quarantines | Reads safety-conservative; fails on real hardware |

**Recommendation.** **Core engine**, with `ActuationSink.tracksAcknowledgements()` as an opt-out for protocols with their own ACKs (EEBus, OCPP). On expiry, the command **lapses and actuation resumes**; flag the participant `unacknowledged` in the snapshot and report it. Quarantine only as an explicit per-participant opt-in. An acknowledgement is **exact equality of the control item's state to the commanded value under `State.equals`** (which already converts between compatible units), with an optional per-participant `ackTolerance`, absent by default. A **changed** command supersedes immediately — only an identical repeat is suppressed, and **a decrease is never suppressed**.

Evidence — `[P]` `SetpointDedupe`'s own comment is decisive on expiry: "after the window, any state divergence is treated as real (e.g. a charger silently resetting its charging current after a SuspendedEVSE cycle) and re-sent" — that behaviour is what recovers real chargers. `[P]` `shouldSend` suppresses only when `prev.equals(desiredValue)`, and the requirement's own words say devices must not be flooded with _repeats_. `[C]` `QuantityType.equals` compares after a unit-compatibility check, so "16 A" vs "16000 mA" is already an ACK with no invented tolerance.

If otherwise — quarantine-by-default makes every `autoupdate="false"` device with no feedback permanently unmanaged after one command, silently; suppress-all means the electrical floor cannot reduce a load for a full window, exactly during the transient it exists for; a default tolerance is a number with no derivation that silently accepts an off-target device.

Retires — EC-3, EC-3b, EC-3c, EC-3d, E8.

### A8 · The observability surface

**Q.** Is there a decision-outcome vocabulary, where do shadow runs, declaration errors and degraded sources appear, and what happens to a malformed declaration?

| Option | Consequence |
| --- | --- |
| Logs only | The corpus's own side-by-side validation scenario stays untestable |
| Core persists outcomes | Core would own persistence policy for a diagnostic, which it never does |
| Events + REST + a status Item | Rules, UI and REST all see the same thing; persistence stays the user's choice |

**Recommendation.** Define the vocabulary and put it in the API: **an outcome enum plus a free-text reason on every decision** — `APPLIED, SHADOWED, STOPPED, SUPERSEDED, DEFERRED, SUPPRESSED, WITHHELD, REJECTED` — published as **events on the event bus, deduplicated so an unchanged decision is not re-emitted**, with the current cycle readable over REST. Core writes no Item for decisions, so "shadow writes nothing" stays literally true. Alongside it, **one engine-published status Item** (String summary + Number count) that the energy pages read. Everything that needs to be visible lands here: a decision naming an unknown participant is **REJECTED, with the reason, and counted**; a **malformed declaration skips the whole participant — never partial-accept — reported through a `ConfigStatusProvider` for the `energy:` namespace, keyed on the item**; an **absent or unrecognised profile class is a configuration error, never defaulted to Simple**; a lost or degraded contributor, a stale safety input, and a protected participant whose state is not persisted (see A15) all report here too.

Evidence — `[C]` core's exact precedent is `RuleStatus` + `RuleStatusDetail` + `RuleStatusInfoEvent` + a REST resource; and `ConfigStatusProvider` / `ConfigStatusMessage{INFORMATION,WARNING,ERROR}` / `ConfigStatusInfoEvent` already exist, i18n'd, per-parameter, consumed by UIs — inventing an EMS-specific error surface is the reviewable defect. `[C]` core publishes events and persistence services subscribe; core never owns persistence policy for a diagnostic. `[P]` `SetpointRequest.reason` is already a short human phrase used for both the shadow line and the lastAction channel; `PeakShaving_Level`/`PeakShaving_Status` exist on your site precisely so a widget can render a banner without poking engine internals. `[P]` `PriorityScheduler.run` already catches a throwing controller, warns, skips its requests and never takes down the tick.

If otherwise — keep outcomes private and every consumer of a shadow run reverse-engineers log strings; partial-accept steers a device on incomplete intent; silent-ignore makes a typo'd script look like a working one; an unrecognised class read as Simple lets the engine switch a Batch programme mid-run.

Retires — EC-12, EC-13, EC-14, EP-5.4, EP-5.5, EP-5.6, UI-5a, UI-5b, FP-3b, B11, B12, E15, E18, N6.

### A9 · The engine-owned enumeration (the security boundary)

**Q.** Is the list of things only the engine may decide closed, and which side of it does each constraint fall on?

| Option | Consequence |
| --- | --- |
| Everything is an algorithm input | A contributed script can start a device its owner marked hands-off |
| Everything is engine-owned | No algorithm can lift a gate under deadline pressure, which your planner legitimately does |
| Split: prohibitions engine-owned, preferences are inputs | Both cases served, along one stated line |

**Recommendation.** **Prohibitions are engine-owned; preferences are algorithm inputs.** Engine-owned, and the list is closed: the `never` hands-off flag; **a user-declared per-consumer level gate ("run only at level ≥ N")**; the readiness interlock; device-protection constraints (min on/off, max off); and the enumeration of readings that count as **safety inputs** (a contributor may not claim that status for itself, and a forecast series is by construction not in it). Algorithm input, overridable: the **engine's own level-derived steering**, so a planner can still lift the engine's gate under deadline pressure. Lift `never` **off the Simple profile onto the consumer**, applying to all four classes.

Evidence — `[X]` the prototype's seam exists because the corpus contradicts itself, and its default is engine-enforced "because it is the safer reading". `[P]` your binding enforces the same shape structurally: gating lives in the asset handler below every controller, "the last line before an item is written". `[T]` storm.house scopes the gate per consumer regardless of class. `[J]` a user-declared "leave this device alone" that a contributed script can override is not a setting, it is a suggestion.

If otherwise — leaving the user's gate overridable makes the single most-cited use case ("the pool pump only when it's cheap") advisory, which also destroys the grounds on which per-domain levels were rejected (Part B); keeping `never` on Simple means the devices most in need of hands-off — an EV you drive manually, a dishwasher — cannot have it, so users fake it by deleting the declaration, which also deletes the measurement the limit floor needs. That is a safety regression caused by a modelling choice.

Retires — EC-16, PM-5.7, FP-3a, N7, A1, F13.

### A10 · Units, bounds and phases

**Q.** Are declared limits in watts or amps, must a controllable provider declare bounds, and how are phases named and enforced?

| Option | Consequence |
| --- | --- |
| Watts only | Every wallbox user hand-converts 6 A for their phase count, wrongly |
| Either dimension, engine converts | Declarations say what the hardware says |
| Optional provider clamp | The first unbounded controllable PV declaration is an unbounded setpoint write |

**Recommendation.** **Either dimension accepted** on consumers and providers; the engine converts current → power with declared nominal voltage × phase count; canonical internal unit is watts. Resolve the layering explicitly: **declared config values are bare numbers whose dimension and unit come from the schema** (`maxCurrentA`, `ratedW`, `demandKwh` — your existing vocabulary), while **item states are `QuantityType` and UoM does the conversion**. **`[min, max]` is mandatory for any controllable provider.** Phases are **integer indices 1, 2, 3**. A participant that declares no phases is **exempt from per-phase enforcement and reported as a declaration gap** whenever the site declares per-phase budgets; the site total still constrains it (judgement, marked as such). And state outright that **per-phase enforcement covers controlled load only unless a provider declares per-phase readings** — so add optional per-phase readings to the provider declaration.

Evidence — `[C]` `ConfigDescriptionParameter` carries `unit`/`unitLabel` for INTEGER/DECIMAL and **throws if you set them on TEXT/BOOLEAN** (line 186), which is core telling you that config is bare numbers plus a schema unit while state is QuantityType — that is the reconciliation between "mandate quantity-typed values" and your `ratedW` keys, and both survive it. `[P]` `budgetToAmps(budgetW, phases, phaseVoltage)` is the conversion, running; the battery provider's `min`/`max` are already part of the declaration and the handler refuses to write without them. `[P]` your context carries `totalAmpsL1/L2/L3` and computes `minBreakerHeadroomA` from measured per-phase load — real per-phase headroom exists in production and depends entirely on per-phase measurement. `[J]` 6 A is the IEC 61851 minimum single-phase charge current; rewording the wallbox scenario to watts would make the requirement lie about the device.

If otherwise — assuming every phase for an undeclared load over-counts a single-phase load 3× and starves a site the first time someone forgets a declaration; controlled-load-only with no per-phase provider reading means the phase nearest its limit is loaded by exactly what the engine cannot see.

Retires — PM-5.9, PM-5.12(a–c), PM-5.8, EC-20, A5, A9, E2, E3, B4.

### A11 · The level plane: names, encoding, bands, ranking input

**Q.** What are the four levels called, how are they encoded, what exactly is a band, and does the classifier rank price or the active objective's series?

| Option | Consequence |
| --- | --- |
| A band is a price threshold (share) | Cannot split tied slots; overflow and overlap impossible |
| A band is a slot count | Exact sizes; needs a tie rule and an overflow rule |
| Fraction resolved to a count | Both properties; retires six questions |

**Recommendation.** Names: **restricted / normal / encouraged / maximum**. Encoding fixed centrally, **0 = restricted … 3 = maximum**. A band is **configured as a fraction of the series, resolved to an integer slot count against the actual series length (nearest-rank ceiling), then cut from a single ascending `(price, start)` ranking** — cheapest band off the head, encouraged next, restricted off the tail. That makes the tie-break one sentence (price ascending, then slot start ascending), makes overflow and band-precedence structurally impossible, and removes the hours-versus-slots contradiction from band configuration entirely. Add a **spread guard**: no non-normal level where `max == min`. And **widen the classifier's input from "the price series" to "the active objective's ranking series"**, defaulting to the effective consumption price.

Evidence — `[P]` you already publish exactly those four strings (`ShadowEmsRunner.levelText`) and `pricePercentileLevel` ranks by count-of-strictly-cheaper; the spread guard is your own `price < max && price > min`, two comparisons, production-tested. `[T]` storm.house uses the same names and the same 0–3 direction ("0 bzw. 💸 … 3 bzw. 👍 … optimal") and configures bands as percentage shares. `[X]` `1/6 · 1/6 · 1/6` of 24 = 4/4/4 reproduces `expected-planned-levels.csv` slot for slot. `[corpus]` the _Numeric exchange_ scenario requires a central encoding, and the _Repeated prices_ scenario requires that some tied slots be excluded — which a pure price threshold can never do, and a pure count can. `[corpus]` _Carbon data as a first-class series_ already says the ranking uses that series "exactly as the cost objective uses the price series".

If otherwise — count-only bands need an overflow rule, a band-precedence rule and a unit, all of which then need fixtures; tie-inclusive bands grade all 12 hours of a flat negative-price day "maximum" and hand the engine a budget it cannot honour; a price-typed classifier input means a carbon site publishes a level saying "cheap now" while placement says "green later".

Retires — EL-1, EL-2a, EL-2b, EL-5, EL-6a, EL-6b, EL-6c, EL-7, OO-1, OO-3, A11/L11.

Names are free **today** and expensive the moment anything is published (Item state options, UI, translations). Honest caveat on the ranking widening: the corpus's text supports it, but all three production systems only ever ran price-based levels — widen the type now, keep price the only shipped input until someone runs a carbon site.

### A12 · The window selection API

**Q.** Is a window request a slot count or a duration, is selection curve-aware, may a window start mid-slot, and what comes back when there is not enough?

| Option | Consequence |
| --- | --- |
| Slot-count requests | Candidates cover unequal time; cost-per-hour vs total cost becomes a real question |
| Duration requests | Every candidate covers the same time; the question disappears |
| Defer curve-awareness | Wave 3 inherits a contradiction instead of a parameter |

**Recommendation.** **Requests carry a `Duration`**, so all candidates cover equal time and the unequal-comparison question is removed rather than answered; keep a slot-count entry point for the "N cheapest hours" case and compare those on duration-weighted mean price. Define **one calculation, `cost(window, weights)`, with flat weights as the default** — wave-1 callers pass nothing and get flat selection, wave 3 passes a profile curve, one implementation. **Windows start on slot boundaries only**, stated as a documented restriction of `cost(window, weights)` rather than as a free optimization: the optimum is provably at a boundary for a flat load, but that proof fails for a shaped one, so once weights exist this is a real restriction of the search space. **Contiguity means abutting** — each slot ends exactly where the next begins; a gap breaks it. When fewer eligible slots exist than requested, **always return the best partial answer, carrying `requested` versus `granted` in the result type**.

Evidence — `[T]` masipila's production `GenericOptimizer` takes float hours (`allowPeriod(3.25)`); the API has never had a slot count. `[X]` the prototype already implements duration selection with a partially-priced final slot, and `PriceSlot.abuts()` is one comparison. `[P]` a batch consumer's optional `shape` already "refines the start to shape-weighted price costing" on top of the same flat selection — the two requirements are only in tension if the calculation has no weight parameter. `[P]` production never returns silence: a batch load gets a forced latest-start fallback and consumers otherwise receive an explicit `HOLD`, never an OFF.

If otherwise — a slot-count API forces an unequal-time comparison rule into the spec and a conversion later; deferring the weights parameter means changing a signature in wave 3; returning nothing on a shortfall means a boiler that needs 3 h and can get 2 gets 0.

Retires — EL-11a, EL-11b, EL-11c, EL-11d, EL-11e, PP-1, PP-2, PP-3.

### A13 · Where the load curve lives, and in what time base

**Q.** Is a curve a property of the demand, the programme, or the profile — and is it stored as an absolute `TimeSeries` or a relative-time shape?

| Option | Consequence |
| --- | --- |
| Curve on the demand | Two demands on one dishwasher can declare different curves for one physical programme |
| Curve on the profile | Wave 1's `ratedW × runtimeHours` becomes the rectangular default profile |
| Stored as a TimeSeries | Timestamps must denote a fictitious epoch; `Policy.REPLACE` is meaningless unanchored |

**Recommendation.** The curve belongs to **the profile**. A consumer always has exactly one _active_ profile — implicit and unnamed in wave 1 — so wave 1 and wave 3 are one model, not two. The stored profile is a **relative-time shape** (offset-from-start + fraction of a scale factor); it becomes a `TimeSeries` **at the moment it is scheduled**, anchored by the planner at the candidate start. A **sample is the average power over the interval from its own timestamp to the next sample's**, the last interval ending at the declared runtime — spacing carried per sample, not implied. State the energy integration rule by name: **LEFT Riemann**. Cap sample count only as a declaration guard (say 1440), with no semantic meaning.

Evidence — `[P]` your taxonomy states both halves already: "Class 4's `ratedW × runtimeHours` is exactly the rectangular special case of this", and "a prototype profile is relative-time (an array/metadata), not yet an openHAB `TimeSeries`; the moment the program is scheduled or started it projects onto absolute time and becomes a natural `TimeSeries`". `[X]` the wave-1 prototype independently built the same thing. `[C]` core's `TimeSeries` is `(Instant, State)` pairs with no width and no relative mode, and `PersistenceExtensions.RiemannType` already defines `LEFT, MIDPOINT, RIGHT, TRAPEZOIDAL` with LEFT as core's own default — reuse the term instead of coining interval-average semantics.

If otherwise — curve-on-demand means a Batch consumer with no pending demand has no curve, so nothing can pre-cost a programme before the user asks; leaving sample semantics unstated means two conforming implementations cost the same dishwasher differently. State the cost honestly: a relative-time profile does **not** ride the price/forecast storage plane, which was the stated attraction of TimeSeries. It is a small config entity instead, which tasks 1.1 already anticipates.

Retires — NP-1, NP-2, NP-3, PM-5.5, PM-5.6, A7, A8.

### A14 · Capacity-tariff budget, and who owns look-ahead

**Q.** Is a demand budget denominated in estimated or billed quantities, when is the error reconciled, and does the engine contract or grid-constraints own replanning?

| Option | Consequence |
| --- | --- |
| Reconcile at period end | You find out you blew the month's peak after it is billed |
| Project and reconcile every cycle | Shaving happens while it still matters |
| Feedback loop inside the floor | The cycle stops being a pure function of its snapshot |

**Recommendation.** Denominate in **the billed, measured quantity — the meter's own slot average**. The planner books against a **projection**: the running slot average extrapolated to slot end, reconciled **every cycle**, shedding against `min(month-to-date billed peak, minimum billable demand)` less an explicit safety margin. Slots must align to the supplier's metering quarters in **wall-clock zone-local time**, not a rolling window — a rolling 15-minute average never matches a bill. Two things a naive answer misses, both in production and both load-bearing: the **minimum billable demand floor** (below it, peaks are free and there is nothing to shave) and the **month-to-date peak as free budget**. Ownership: **`define-grid-constraints` owns look-ahead and replanning; `define-engine-contract` owns the runtime floor and a deferral-to-report path only** — a deferral is terminal for the cycle, published as an outcome (A8), which the planner may consume on its own cadence.

Evidence — `[P]` this is the mechanism running against a real Belgian capacity-tariff bill: `CapacityTariffTracker` keeps a running average over the current 15-minute slot "aligned to :00/:15/:30/:45 — same as the supplier's metering quarters", commits at rollover, tracks the month-to-date peak, resets monthly; `CapacityTariffShavingController.wouldExceedMonthlyPeak` compares `projectedQuarterW` against `Math.min(mtdPeak, -minBillableW) - CAPACITY_SHED_MARGIN_W`. Norway's top-3 is the same shape with N-highest averaging. `[corpus]` the _Established peak as budget_ scenario already assumes the free-budget half. `[X]` a terminal floor is what keeps A6's stateless re-booking honest.

If otherwise — estimate-denominated budgets are reconciled against a bill that used a different number; a feedback loop inside the floor needs a fixed-point argument nobody has, to keep "the same inputs always resolve one overload the same way" true.

Retires — GC-3, GC-4a, EC-17, E16.

### A15 · Degraded inputs and device protections

**Q.** What counts as a stale safety input, what is the conservative safe state, and where does elapsed time for a protection come from?

| Option | Consequence |
| --- | --- |
| Pause everything on stale input | How a user learns to uninstall an EMS |
| Freeze and floor | Loads keep running at their declared minimum; nothing ramps |
| Engine-held protection timers | Every restart silently voids every duty-cycle guarantee |

**Recommendation.** Staleness is **both** an optional per-participant `maxAgeSeconds` and unreadable/`UNDEF`/`NULL`, which always trips; do not invent an age mechanism — say a site may use core's `expire` namespace to turn a frozen item into `UNDEF`. Safe state is **freeze-and-floor**: refuse any increase, floor Controllable loads at their declared minimum, drop ModeControllable to its most-restricted mode, turn Simple loads off — **all of it subject to device protections**, so a Simple load inside its minimum runtime is held, not shed; the safe state may refuse increases immediately but may only shed once protections permit. Never interrupt a running Batch programme. Elapsed time comes from **`Item.getLastStateChange()`**; when it is null, **start the clock at first observation** and report the participant as protection-unknown. **The engine keeps no protection timers of its own.** And restate mstormi's "forced restart" clause observably: whenever the engine observes a protected consumer transition OFF→ON that it did not command, minimum-runtime applies from that observed transition exactly as for an engine-initiated start.

Evidence — `[C]` `Item.getLastStateChange()` exists (Item.java:77) and `PersistenceManagerImpl.restoreItemStateOnStartup` restores `lastStateChange` explicitly for this purpose, so "does a compressor's cooldown survive a restart" already has a core answer: yes, **iff the item is persisted with `restoreOnStartup`**. `[C]` `ExpireManager`'s `expire` namespace already converts "silently froze hours ago" into "unreadable", which is the gap that currently makes this safety requirement inert. `[P]` your building already does floor-not-stop: "if the PowerTag bridge looks dead (no Amps update in 30 s), all chargers cap at 6 A". `[X]` the four candidate "forced restart" events are indistinguishable at the item level, so any definition naming them is unimplementable; a definition naming the transition is free.

If otherwise — engine-memory timers void protections on every restart; a persistence-query per cycle gives a safety function a hard dependency on a queryable persistence service; shedding a Simple load mid-minimum-runtime contradicts the ladder in A3, which on your site is the boiler and the airco group.

This is **not free**, despite core answering the mechanism: it departs from your binding (`constrainOnOff` currently leaves desired state untouched when `minutesInState` is NaN), and it makes a safety guarantee contingent on user persistence configuration. Required spec clause, therefore: the engine must **report** a protected participant whose state is not persisted, so the degraded guarantee is visible rather than assumed. That is A8's third customer.

Retires — EC-4c, PM-5.15, EP-5.7(a), EP-5.7(b), N2, N3, E12.

## Departures — where Part A asks you to change working behaviour

Eight, and they are the reason to read Part A rather than skim it. In each case the production choice is correct for your site's device population and does not generalise to the devices the requirements name.

- **A2** — lower-wins-conflict inverts your `Controller` contract ("lower runs first, higher wins on conflict"), which your bridge implements by iteration order. Reconciling it is binding work.
- **A6** — a declared `ratedPower` for Simple loads replaces the on-threshold inference. Kept **optional** with a reported gap precisely so your live declarations stay valid.
- **A15** — start-the-clock-at-first-observation instead of leaving desired state untouched on unknown age; and your staleness gate currently caps chargers at 6 A with no protection interlock.
- **A9** — `never` lifts off the Simple profile onto the consumer, and a user-declared level gate becomes engine-enforced.
- **A7** — ACK handling moves from asset handlers into the engine, with a 60 s default window instead of 15 s.
- **A5** — battery reading sign becomes `+ = discharge` everywhere; `Site` and `EnergyReading` currently say the opposite of `EnergyContext`.
- **A4** — 60 s tick with a 5 s debounce, not a 5 s tick (Part B row EC-1).
- **A11** — band configuration becomes fraction-resolved-to-count, not tie-inclusive percentile; your current form grades every tied slot alike.

Unaffected: your mapdb + influxdb setup, since mapdb's role here is `restoreOnStartup`, not TimeSeries.

## Part B — parameters and defaults

Reversible. Scan, strike anything you dislike, approve the rest in one go.

### Engine contract

| Question | Recommended value | Evidence |
| --- | --- | --- |
| EC-1 evaluation cadence | Tick **60 s** plus event-responsive re-evaluation on declared readings, debounced **5 s** | `[P]` your 5 s tick + ~1 s debounce exists because a control that waits a full interval reads as broken; `[T]` Kai's "prices possibly every minute" fixes the pull half at 60 s |
| EC-3a ACK window length | **60 s** engine-wide, per-participant override; not learned | `[P][X]` the invariant is 2–3 evaluation cycles (15 s at a 5 s tick, 60 s at a 60 s tick); slow OCPP chargers override |
| EC-7 what survives stop/resume | Clean slate for engine-held ephemeral state (ACK windows cleared); protections continue by construction under A15 | `[P]` `SetpointDedupe.forget(key)` already runs when the engine stops owning a device |
| PM-5.4 demand kinds | Energy amount + deadline + must-not-be-interrupted only; **drop the two examples from the prose** — SoC→kWh is converted by the declarer, runtime-in-window is wave 3 | `[X]` neither example is buildable; `[P]` production carries `demandKwh` + `deadlineHour` only |
| PM-5.10 level→_n_ modes | `modeIndex = ceil(level × (n−1) / 3)`, index 0 = most restricted; **for _n_=2 do not use it** — binary devices use the per-consumer "on at level ≥ N" gate | `[P]` `EnergyManagementService.modeIndex` verbatim, running; ceil not round, so a NORMAL level does not block a two-mode device |
| PM-5.16 / EP-2 naming | Park `EnergyProvider` for the combined naming pass; when it runs, participant side keeps `EnergyConsumer` and renames the provider role, data contributors get per-kind names (`PriceSource`, `ForecastSource`) | `[C]` `org.openhab.core.common.registry.Provider` already means "contributor of registry elements"; `[X]` the corpus's wave-2/3 changes are already per-kind |
| EP-5.10 unresolvable Item names | Parser accepts (item and metadata arrive in either order); engine treats it at evaluation as a degraded source; **WARNING, not ERROR** | `[C]` `ItemChannelLink` is core's precedent that a forward reference is legal and re-binds later |
| EP-4(i) data-contributor choice | Per-role configuration naming the preferred contributor, defaulting to highest `service.ranking` | `[C]` `StateDescriptionServiceImpl` sorts providers by rank descending |

### Energy levels

| Question | Recommended value | Evidence |
| --- | --- | --- |
| EL-4 one level or per-domain | **Site-global**, with the seam `level()` so a future `level(domain)` can default to it | `[T][P][X]` all three systems publish exactly one; the corpus has no domain model to key on, and the cited use case is served per-consumer |
| EL-9 level where there is no plan | **normal**, and publish the absence of the feed separately | `[P]` `pricePercentileLevel` returns 1 for a missing schedule and production runs on it; most-restrictive turns a failed HTTP fetch into a cold house |
| EL-10 when a count change takes effect | Immediately: re-derive over the whole published series, re-publish `[now, end)` as one `TimeSeries` with `Policy.REPLACE`, leave elapsed entries alone | `[P]` derivation is a pure function re-run every tick; `[C]` `Policy.REPLACE` already deletes exactly `[begin, end]`. Note: `PersistenceManagerImpl` converts those bounds with `ZoneId.systemDefault()` — worth one sentence in the requirement |
| EL-8a PV escalation | `escalation = graded`; `encouragedFrom` = site surplus threshold (W), `overcapacityFrom` = 2× it | `[T]` storm.house escalates on a configured watt threshold; `[P]` you add the second step; `[X]` the prototype defaults to `none`, leaving the requirement inert out of the box |
| EL-12a season expression | Fixed month-day ranges, user-editable, shipped defaults: **summer 15 May – 15 Sep, winter 1 Nov – 20 Mar, transition = the rest** | `[T]` storm.house's own dates — note they are neither meteorological nor astronomical, which excludes the other two options outright |
| EL-12b whose midnight starts a season | The **site** zone, from core's `TimeZoneProvider` | `[C]` a season is a heating-demand concept about the building, so it is a site property |
| EL-12c which date names a delivery day | The **market's** local date, in the market's zone; carry the zone on the price series, never infer it | `[X]` `dayahead-prices.csv` runs 2023-03-23T23:00Z → 2023-03-24T22:00Z, i.e. exactly the CET calendar day — not UTC and not masipila's own EET day |
| EL-13 zero and negative prices | Levels stay purely **relative** — say so in one sentence: a level is a rank within the horizon, never a claim about absolute price. Absolute-price behaviour reuses the escalation seam, opt-in | `[C]` `Number:EnergyPrice` is a signed QuantityType with no non-negativity constraint; renaming "blocked" → "restricted" (A11) removes half the sting |
| EL-3c SG-ready mode-1 time cap | Participant-side, explicitly out of core. **Fact-checked: the BWP SG-Ready spec says max 2 h per assertion and max 3 assertions per day** | `[X]` `SgReadyMode`'s javadoc already calls it a device-side obligation. Flag the gap: the participant model can express "max 2 h" as maxOff and **cannot** express "3× per day" |

### Grid constraints

| Question | Recommended value | Evidence |
| --- | --- | --- |
| Capacity shed margin | An explicit configurable watt margin subtracted from the shed threshold; seed from the production constant | `[P]` `CAPACITY_SHED_MARGIN_W` exists and is the tunable part of A14's otherwise fixed shape |

### Energy UI

| Question | Recommended value | Evidence |
| --- | --- | --- |
| UI-1 where the UI ships from | Framework-served via the core UI-component provider mechanism, **in a bundle separate from the engine** so sequencing it into MainUI later is a deletion, not a refactor | `[P]` `EnergyUiProvider` is doing exactly this on your building today — provider-served, never written to JSONDB, pages rebuilt live from `energy:` tags |
| UI-2 overview minimum set | Site level; live per-participant power; today's cost or self-consumption per the active objective; the active grid constraint's headroom (month-to-date peak for a capacity tariff); one status line for degraded sources and declaration errors | `[corpus]` the first three are its own candidate set; `[P]` the peak KPI is in your shipped `Now` tab; the status line is where A8 lands |

### Discovery

| Question | Recommended value | Evidence |
| --- | --- | --- |
| DP-1(i) which meter is the grid | Propose every `ElectricMeter` carrying a Power measurement as a **candidate**, rank one directly under a Location first, **never auto-assign even with a single candidate** | `[C]` `SemanticTags.csv` has no grid tag and no top-level/sub distinction; `[P]` your own site has seven indistinguishable PowerTags on one bridge |
| DP-1(ii) EVSE control granularity | Read amps-vs-watts from the setpoint Point's **Property tag** (`Current` vs `Power`), fall back to the item's UoM dimension, then confirm; read the range from `StateDescription` min/max, else confirm — never assume 32 A or 11 kW | `[C]` `Property,Current` (line 62) and `Property,Power` (line 87) are sibling first-class tags — the design note's "read from UoM" misses this |
| DP-1(iii) Generator / WindGenerator / UPS | Park; propose nothing unless the user opts in | `[corpus]` _No guessing without semantics_ already covers it |
| DP-1 rule shape (warning) | **The parent tag is not usable as the role rule — the leaf tag must be.** `EVSE` sits under `PowerSupply` (SemanticTags.csv:194) but is a consumer, and `UPS` sits there too but is neither | `[C]` verified in the CSV; any "everything under PowerSupply is a provider" rule proposes the wallbox as a generator |
| DP-2 where proposals live | An EMS-owned proposal registry **modelled on** the Inbox lifecycle (NEW / IGNORED / approve-becomes-declaration), keyed on participant identity, surfaced through the guided-setup requirement; approval writes `energy:` metadata | `[C]` `DiscoveryResult.getThingUID()` and `Inbox.add`'s dedupe-on-Thing-ID mean an item-keyed proposal cannot reuse the Inbox without a fake ThingUID; the lifecycle is still the right one and users already know it |

### Learning layer

| Question | Recommended value | Evidence |
| --- | --- | --- |
| LL-0 missing design file | Create `add-learning-layer/design.md` and move LL-1..LL-4 into it | `[corpus]` same defect class as N8, which was dispositioned Fixed for `define-energy-levels` by creating one |
| LL-1 propose-don't-override | Keep it; re-source it from _Propose, never silently activate_ rather than leaving it flagged as synthesized. Narrow the override clause: a learned value may auto-apply only where the parameter is user-configurable **and** the participant declares it learnable; safety-relevant limits never auto-apply | `[corpus][P][X]` three independent instances of one guardrail — discovery's requirement, your shadow-mode default, the prototype's four enforcements |
| LL-2 minimum data | A **per-learner contract** (each learner declares its items and minimum retained history), enforced by refusing to run a learner whose inputs are not persisted, reported on A8's surface. Thermal model: indoor temp, outdoor temp, heating power at `everyChange` + `everyMinute`, ≥ 30 days | `[C]` persistence is already configured per item and per strategy. The **30-day figure is judgement** — see Part D |
| LL-3 model-quality metric | **Two numbers from every learner**, not one: a normalized confidence in [0,1] from the estimator's own parameter uncertainty ("converged" = ≥ threshold), and a rolling prediction error in the model's own unit ("drift" = above a multiple of its converged value). Publish both as items. (Load-bearing for wave 4) | `[P]` `ThermalModelEstimator` already carries both and they are different things — a parameter covariance that shrinks, and a tracked `lastResidual()`. The corpus's two scenarios need one each |
| LL-4 learning trigger | Per completed run for load profiles; continuous/online for the thermal model; on-demand re-learn for both. **Not nightly** | `[P]` the thermal estimator is online by construction (`update(dt, …)` per sample) and a nightly batch throws away its recursive form; a partial run is not a curve |

## Part C — already answered

Do not spend a second on these. Each has an answer and a source.

| Question | Answer | Source |
| --- | --- | --- |
| EP-5.1 is the `energy:` schema core's to specify | Whoever owns the framework bundle owns the namespace, and publishes it as a `MetadataConfigDescriptionProvider` — machine-readable, not prose | `[C]` its javadoc: "Every extension which deals with specific metadata (in its own namespace) is expected to provide an implementation of this interface"; core does it for `autoupdate` and `expire`. **Highest leverage in the set** — it also discharges EP-5.2, EP-5.3, EP-5.5's surface and most of the guided-setup requirement, since UIs render the editor form from a config-description automatically |
| EP-5.8 does the core registry contract need precedence | No, and no core API change is needed: the participant registry is an **aggregating service over ranked providers**, not an element registry | `[C]` `AbstractRegistry.added()` is first-come-wins, but `StateDescriptionServiceImpl` and `ConfigDescriptionRegistry` are the ranked-aggregation pattern that exists for exactly this |
| EP-5.9 do script contributors have an unload hook | **Yes — the premise is false.** Keep the add-on/script equivalence as written | `[C]` `org.openhab.core.automation.module.script.providersupport`: `ProviderScriptExtension` "ensures that they are removed when the script is unloaded", `ProviderRegistry.removeAllAddedByScript()` |
| EP-5.12 what "equal terms" means | Same SPI as any contributor, no special-casing, with core's own default registered at `service.ranking = -2` so any alternative outranks it with zero configuration. Testable | `[C]` verbatim what `DefaultStateDescriptionFragmentProvider` does (`-2`, with `ChannelStateDescriptionProvider` at `-1`) |
| EP-5.3 / PP-5 / B3 unknown declared keys | Accept, ignore, log once at DEBUG. Reject nothing | `[C]` `ConfigDescriptionValidatorImpl.validate()` iterates the described parameters and never the supplied map — undescribed keys are silently accepted everywhere in openHAB today |
| OO-6a contributed-objective identity | First registration wins, duplicate id refused with a WARN — and the "a reloaded script must replace itself" objection is obsolete | `[C]` `AbstractRegistry.added()` logs "It exists already from provider …"; the `providersupport` unload hook removes the need for last-wins |
| OO-6b what makes terms "equal" for objectives | The `service.ranking` whiteboard, core's defaults at a negative ranking | `[C]` same pattern as EP-5.12 |
| FP-1b must the layered series require `ModifiablePersistenceService` | Require it; it costs nothing new, and the design note's premise is factually wrong | `[C]` `PersistenceManagerImpl.timeSeriesUpdated()` filters on `instanceof ModifiablePersistenceService` before doing anything — core persists **no** TimeSeries, future entries included, to a non-modifiable service |
| FP-1a the survey (task 2.4) | Modifiable: **influxdb, inmemory, jdbc, mongodb**. Not: **dynamodb, jpa, mapdb, rrd4j**. `inmemory` is the always-available one | `[C]` re-run against `/home/openhab/openhab-addons/bundles` |
| FP-2 writer precedence on a layered series | Caps composed at **read time** as a separate constraint series — options (a) and (b) are unimplementable, not merely worse | `[C]` a published TimeSeries carries **no writer identity at all**, so no write-time precedence rule can be evaluated; `[corpus]` _Price component composition_ already specifies read-time composition, with `min` instead of `+` |
| UI-3 aggregated-series mechanics (task 1.3) | Pre-computed rollup series, one point per day into dedicated items. The blocking survey is closable from core's SPI alone | `[C]` `FilterCriteria` carries only itemName/dates/paging/operator/ordering/state — **no aggregation field**, so the alternatives require changing an interface every persistence add-on implements. `[P]` `StatisticsRollupController` is the running alternative — and two non-obvious details belong in the requirement: snapshot _before_ the day boundary, and derive the increment from the last reading _before_ the reset |
| EL-2a is a numeric encoding fixed centrally | Yes — the _Numeric exchange_ scenario cannot be satisfied otherwise. The encoding itself is in A11 | `[corpus]` the requirement forecloses its own third option |
| EL-3a the SG-ready offset | There is no offset — ship a named mapping, not arithmetic | `[X]` `SgReadyMode` already pairs `mode()` 1–4 with `level().code()` 0–3; named enums plus a lookup are core's convention |
| EL-8b hysteresis on the current level | No dwell timer. Chatter is absorbed downstream by per-consumer minOn/minOff/maxOff, by requirement; if you still want a level-side deadband use two thresholds (engage high, release low), not a timer | `[corpus]` _Level-gated operation_'s own scenario ends "(protection times still respected)"; `[T]` storm.house caps it at the device; `[P]` `peakShaveActive(gridW, engageW, recoverW, active)` |
| PM-5.3 deadline forms | Both: a bare local time resolves to the next occurrence at or after the evaluation instant in the site zone from `TimeZoneProvider`; an absolute deadline is an ISO-8601 instant | `[C]` `TimeZoneProvider` is the convention for the site zone. Qualify it as that, **not** as "core never calls `ZoneId.systemDefault()`" — there are 17 such call sites in core, including `PersistenceManagerImpl` |
| PM-2 core vs add-on boundary | Confirm the split; the engine, the actuation sink and the level plane are core, regional constraint logic and price/forecast/optimization are add-ons | `[T]` Kai has fixed the direction ("belongs in openhab-core, not yet another binding"); `[X]` the prototype shipped it as an opt-in feature, so an installation that never installs it is byte-identical |
| OO-4 objective selected, data plane missing | Fall back to cost and report the degraded source; then to all-`NORMAL` plus surplus escalation | `[corpus]` _Graceful degradation on contributor loss_ already mandates this shape, and covers the harder case of a source vanishing after selection |
| EC-13 decisions naming an unknown participant | Reject, mark REJECTED with the reason, count it — falls out of A8 | `[P]` `PriorityScheduler.run` already does the analogue for a throwing controller |
| EC-20 / E2 per-phase declaration | Already closed — the _Phase declaration_ requirement now exists. The live half (E3) is in A10 | `[corpus]` |
| EC-19 priority direction filing | Answered by A2; replace the section with a pointer and fix engine-contract's wording ("higher-priority" → "better-priority") | `[corpus]` |
| OO-2 composite objectives | Out of v1 | `[corpus]` |
| Not questions at all | PM-3 (engine simplicity — use it as an acceptance test on A1: a GraalJS object must be a first-class algorithm, which `ScriptAlgorithmProofTest` already proves), PM-4 (validation methodology — promote to the implementation plan), EC-21, UI-4, DP-3, DP-5b, LL-5, EP-5.13, GC-5 | `[corpus]` each is a note, a stance, or a dependency already filed elsewhere |

Three corpus statements are now **factually stale** against current core and should be corrected whatever you decide: EP-5.9's "nothing reliably tells a script it is being unloaded"; EP-5.8's framing as needing a core-maintainer decision; DP-1(ii)'s "read from UoM/state-description" (the Property tag already carries it). Add the SG-Ready primary source from Part B, and the fixture's CET-day provenance from EL-12c, to `fixtures/README.md`.

## Part D — genuinely your call

Five. No recommendation, and no invented rationale to pad them.

- **GC-4(b) — is there a discriminating boiler/budget test vector?** The corpus's own constraint binds: any such vector must come from the thread or a production system and must not be invented. The current pair does not discriminate (9 + 3 = 12 > 10, so the budget degenerates to mutual exclusion), and what it turns on is whether anyone can supply a run where two loads _partially_ share a slot. Three honest paths: ask masipila or mstormi in the thread; **derive one from your own site** — four EV chargers sharing an ECO solar budget under a capacity tariff is literally a partial-fit vector, and a second production system is the corpus's own accepted provenance class, so this is not inventing; or ship without one and keep the non-discriminating note. Path two is cheapest and entirely in your hands.
- **OO-5 — what carbon credit does an exported kWh get when the feed-in price is negative?** The cost side is settled arithmetic and needs no decision: value a kWh at `p(t)` imported and `f(t)` exported, so with `f < 0` running now dominates and the self-consumption and cost objectives simply agree. Carbon is the open part — a naive carbon objective exports green power to displace grid generation, but a negative feed-in price is the market saying the grid does not want it and the realistic outcome is curtailment, which makes the displacement credit fictional. Crediting export only while `f ≥ 0` follows from that reasoning, but **no thread source and no production system says it**, and it is one term in one objective. Yours.
- **LL-2's retention figure.** The per-learner contract is grounded; the **30 days** is not. It is judgement sanity-checked against your estimator's "converges within hundreds of samples under varied inputs" — at 1-minute sampling that is hours of samples but weeks of varied outdoor conditions, and the binding constraint is the weather, not the sample count. Say so in the requirement rather than presenting 30 days as derived.
- **EP-2's rename target.** That the participant role should not be called `EnergyProvider` is grounded (`org.openhab.core.common.registry.Provider` already means contributor, and the EMS SPI will extend it). **Which word replaces it** — `EnergySource` or anything else — is taste, and Kai coined both of the names in question, so it is worth one line in the thread rather than a unilateral rename.
- **DP-4 — whether and when to float discovery's provenance in #3478.** Procedural, not technical, and it should follow A1 rather than precede it: discovery's value depends entirely on whether declarations are metadata (discovery writes metadata, obviously useful) or a programmatic SPI (discovery has nothing to write). Your explicit-go workflow applies — draft body plus exact command, show, wait, then post.

## Closing note

Answering these makes them **your decisions in a reference implementation**, not thread consensus. The corpus will record them that way: each answer lands in its requirement with the rationale and the evidence tier attached, and **the alternatives stay preserved in the design file** rather than being deleted once a choice is made. That is deliberate — a maintainer who engages later can overturn any of them by pointing at the option that was kept, and the cost of doing so is a rewrite of one requirement rather than an archaeology exercise. Two waves of implementation on top of an answered spec is a far better argument than 79 open questions and no code.
