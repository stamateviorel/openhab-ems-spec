# Prototype feedback — what building wave 2 surfaced

Companion to [`PROTOTYPE_FEEDBACK.md`](PROTOTYPE_FEEDBACK.md), which is the wave-1 catalogue and is
not superseded by this one.

The same distinction applies and matters more than the content. Thread-sourced material is consensus
already reached by the people who run these systems. Everything on this page is the opposite: the
residue of a build, found by one implementation, agreed by nobody. The `> Source:` lines exist so a
reviewer can tell the two apart at a glance.

Wave 2 built the price plane, the forecast plane, the objectives and the join between them
(`STAGE2_REPORT.md`). It surfaced seventeen places where this corpus was ambiguous, contradictory or
silent. **Five needed an owner decision** and are answered in
[`OWNER_DECISIONS.md`](OWNER_DECISIONS.md) as D31–D35 and D36–D38. The remaining twelve are
corrections the corpus needs, catalogued here.

## The five that became decisions

| Finding | Decision |
|---|---|
| §5.9 a forecast surplus is required by O1 and defined nowhere | D31 |
| §5.7 a refresh landing on a cap erases it, and three options are framed with none chosen | D32 |
| §5.8 what an objective does when its data plane is absent | D33 |
| §5.11 whether the four-level signal follows the objective or stays price-based | D34 |
| §5.17 whether removing the shipped currency default went too far | D35 |
| §5.1 `ct/kWh` is asked for twice and cannot be expressed | D36 |
| §5.3 the adjustment pipeline has no stated order | D37 |
| §5.16 the tie-break is defeated by floating point | D38 |

## The twelve corrections

Each names what the corpus says now, what building it showed, and what should change. None of these
is a design choice; they are places where the text asks for something impossible, undefined, or
different from what it says elsewhere.

| id | Where | What building it showed | What the corpus should say |
|---|---|---|---|
| W2-1 | *Price component composition* | `QuantityType.toUnit` cannot convert a price even within one currency — the corpus's "the currency question is already solved by `Number:EnergyPrice`" is true for **carrying** a price and false for **converting** one | A footnote drawing that distinction, so an implementer does not plan on a conversion that is not there |
| W2-2 | *Price component composition* | Composition across components whose slots do not line up is mandated and undefined. Until it is defined, the alignment is configuration rather than behaviour | A scenario for two components on different geometries |
| W2-3 | *Prices as future-timestamped series*, *Forecasts as future-timestamped series* | A `TimeSeries`' last entry has no width, and every plane needs one. This is a framework-level gap, not a modelling preference | Both should state what the last entry's interval is |
| W2-4 | *Layered prediction series* | The layer ledger is not durable. The finding is sharper than "two options are impossible": all four are implementable, two are **not durable**, and the durable one is the option that never writes the cap into the prediction — which D32 has now made the default | State the durability property per option, not feasibility |
| W2-5 | *Export share* | `ExportShare` has no ranking-time definition — it is meaningful at dispatch and undefined when ranking a future slot | Either define it at ranking time or say it is a dispatch-time quantity only |
| W2-6 | *define-extension-points* | Objectives and algorithms disagree about what a duplicate contributed id means | One rule for contributed identity, applied to both |
| W2-7 | *define-price-providers* | The source SPI is pull-only, so nothing can notice that new prices have arrived; a site must configure a refresh cadence or tomorrow's prices are not picked up on the day they land | Say that the SPI is pull-only and that a cadence is therefore required, or add a way for a source to announce |
| W2-8 | *define-extension-points* | The price plane's conditions are registry-side only, so a **source** cannot report its own configuration problem | Whether a source reports its own conditions, and through what |
| W2-9 | *Shared window calculations* | A curve over a non-consecutive selection is undefined — a load curve assumes a run, and the ranked-slots form has no run | Say that a curve applies only to consecutive selections, or define what it means over a gap |
| W2-10 | *define-optimization-objectives* | The corpus's design §4 still reads "Undecided." where the implementation ships a default | Record the decision and where the default lives, either way it goes (now D33) |
| W2-11 | *Prices as future-timestamped series* | A source that throws is a normal event, not a fault, and the registry must survive it | State that a contributed source may fail and what the plane does when it does |
| W2-12 | *Shared window calculations* | `RankedSlotsSelection` ignores the load curve, documented and defensible — a slot's position in the run is not known until the set is chosen, so weighting it would be circular | Say so, rather than leaving a reader to assume the curve applies everywhere |

## What is deliberately still open

- **The forecast surplus stays off by default.** D31 defined what it means, not whether it runs.
- **W2-5, W2-9 and W2-12** are recorded as limitations rather than answered; each is defensible as it
  stands and none blocks an implementation.
