# Design notes — open questions

Sections carrying an **ANSWERED** block were decided by the owner on 2026-08-02; the
record, with rationale and evidence, is `docs/OWNER_DECISIONS.md`. Those are **the owner's
decisions in a reference implementation, not thread consensus** — nobody in #3478 agreed
them — and the question each one answers, including every option that was not chosen, is
preserved underneath it. Sections without such a block are still open. One of them (§5) is
weaker than that: it rests on the owner's reasoning with no thread and no production
source behind it at all, and says so.

## 1. Do energy levels follow the objective?

> **STILL OPEN.** The decision pack recommended widening the classifier's input from "the
> price series" to "the active objective's ranking series", defaulting to the effective
> consumption price (its pack A11) — with its own caveat that all three production systems
> only ever ran price-based levels. The owner took only the encoding half of that pack
> (D12), so this interaction is undecided and the pack's recommendation is one live option.
> It is still task 1.1.

`energy-levels` derives the 4-level signal from the **price** schedule (base) plus PV
escalation. Under the carbon objective, should levels derive from the green-share series
instead — or stay price-based while only load _placement_ changes? Production systems
only ever ran price-based levels; undecided. This is the main interaction to settle with
`define-energy-levels`.

## 2. Composite objectives

Weighted mixes ("mostly cost, mild carbon preference") are plausible but nobody in the
thread asked for them; deliberately out of v1. If they come, they arrive as a contributed
objective (extensibility requirement) before they justify core complexity.

## 3. Naming

The thread converged on objective-neutral wording — masipila adopting mstormi's "best
hours" over "cheapest hours"
([1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)).
API and UI vocabulary should follow ("best window", not "cheapest window").

## 4. Objective availability without its data plane

Selecting the carbon objective when no carbon-intensity source is installed is
undefined as written. Options: hide unavailable objectives, fall back to cost with a
visible notice, or refuse selection. Interacts with the extension surface's degraded-
source reporting. Undecided.

## 5. Feed-in under non-cost objectives

> **ANSWERED — owner decision D20** (2026-08-02, `docs/OWNER_DECISIONS.md`), and this is
> **the weakest decision in the corpus**. Say so wherever it is repeated.
>
> _The cost half needed no decision._ Value a kilowatt-hour at the consumption price `p(t)`
> when imported and the feed-in price `f(t)` when exported; with `f < 0`, running the load
> now dominates, and the cost and self-consumption objectives simply agree. That is
> arithmetic, and it is stated in `define-price-providers` _Feed-in pricing_.
>
> _The carbon half is the decision._ **No carbon credit while the feed-in price is
> negative.** The reasoning: a naive carbon objective exports green power to displace grid
> generation, but a negative feed-in price is the market refusing that power, so the
> realistic outcome is curtailment and the displacement is fictional. Stated in _Carbon
> credit for exported energy_.
>
> **The honest flag, which the requirement carries too: no #3478 comment and no production
> system states this.** It is reasoning alone, which is precisely why the decision pack
> refused to recommend it and handed it to the owner, and why the owner's own record calls
> it the most overturnable decision in the set. It is also small: one term in one objective,
> so overturning it is a one-line change to the carbon objective and nothing else in the
> corpus moves.
>
> Alternative preserved: credit the export regardless of price (the naive reading, which is
> what any implementation does if this requirement is deleted), and the middle option of
> crediting it at a curtailment-discounted rate, which nobody has data for.
>
> The worked example task 2.1 asks for is still worth producing — the decision states the
> rule, it does not demonstrate it on a real day.

Negative feed-in prices (`price-data`) interact oddly with self-consumption and carbon
objectives (exporting green power vs. consuming it locally). Needs a worked example
before hardening.

## 6. What the wave-1 prototype surfaced

> **ANSWERED in part — owner decision D10**, with D11 behind it (2026-08-02,
> `docs/OWNER_DECISIONS.md`), for the third bullet only.
>
> _Surplus_ (E7): **grid export plus the battery charging the engine can reclaim.** Battery
> charging is a decision the EMS itself made rather than a fixed load, so it is available to
> a better-priority consumer — one whose priority number is **lower** than the battery's
> (D4), the battery carrying a priority on the same scale so the comparison has a second
> operand at all (`define-participant-model` _Controllable providers_) — which is what stops
> a solar-first EV charger idling while the
> battery absorbs everything, the behaviour the owner's own site shows. Curtailed PV is
> **not** counted: the most complete option, but it needs inverter curtailment reporting that
> many plants do not have. Signs come with it (D11): grid + = export, battery + = charging,
> PV + = producing, consumers + = consuming, devices that disagree normalised at the edge.
> Stated in _Selectable objective_ here and in `define-energy-levels` _Surplus escalation of
> the current level_; alternatives preserved where the question lives,
> `define-engine-contract` design.md §11.
>
> Still open inside that answer, by the owner's own note on D10: whether the figure is
> instantaneous, averaged or forecast, and whether an already-running managed consumer's own
> draw counts.
>
> The first two bullets are **not** owner decisions. The decision pack dispositioned both in
> its Part C — a contributed objective's identity by core's own first-registration-wins
> registry plus the script-unload hook that makes last-wins unnecessary, and "equal terms" by
> the `service.ranking` whiteboard with core's defaults at a negative ranking — and the
> owner's recorded decision on that part was no action, so no requirement here states them
> yet. They stay open on this page.

Opened by building the wave-1 slice against this corpus, not by #3478. Ids are the
prototype's own (see `docs/PROTOTYPE_FEEDBACK.md`).

- **A contributed objective needs an identity story (I6).** _Objective extensibility_ makes
  the objective an extension point that scripts may contribute. Wave 1's algorithm plane
  registers by id and the last registration silently replaces the previous one — which is
  what a reloaded script needs, and which also lets one contributor take over another's id
  unnoticed. The declaration plane works ownership out carefully; the algorithm plane,
  where a contributed objective would live, says nothing. Filed as I6 under
  `define-engine-contract`; noted here because objectives inherit whatever it answers.
- **"On equal terms" is undefined, and the three built-ins are its first test (N10).**
  `extension-surface` _Core-shipped defaults without privilege_ requires core defaults to be
  replaceable "on equal terms" without saying what makes terms equal — same registration
  mechanism, same priority range, displaceable by id are all live readings. This change
  ships three built-in objectives that a contributed one must be able to displace, so the
  reading bites here first.
- **The self-consumption objective rests on an undefined word (E7).** _Selectable
  objective_'s first scenario places a load "into the surplus". Nothing in the corpus
  defines surplus beyond its being derived from grid export: whether a charging battery's
  power or curtailed PV count towards it is unstated, and each answer places the load
  somewhere else. Filed as E7 under `define-engine-contract`.
