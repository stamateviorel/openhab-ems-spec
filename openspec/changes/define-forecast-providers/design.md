# Design notes — open questions

Sections carrying an **ANSWERED** block were decided by the owner on 2026-08-02; the
record, with rationale and evidence, is `docs/OWNER_DECISIONS.md`. Those are **the owner's
decisions in a reference implementation, not thread consensus**, and the question each one
answers — including every option that was not chosen — is preserved underneath it.

Sections 1 and 2 carry no such block on purpose. The decision pack that produced the
2026-08-02 answers dispositioned both in its Part C — already answered by core's own API
rather than by anything the owner had to choose, and in one case by contradicting this
page's premise — and the owner's recorded decision on that part was **no action**. So the
answers exist, but no requirement in this corpus states them yet, and these two sections
stand as written. Closing them is a documented follow-up, not a decision.

## 1. Overwriting past entries (feasibility dependency)

Future entries are native core capability: TimeSeries + the `forecast` persistence
strategy (openHAB 4.1). Overwriting _past or current_ entries — the §14a/inverter cap
written onto today's prediction — means modifying persisted history. Core ships the API
for it: `ModifiablePersistenceService` (`store(Item, ZonedDateTime, State)`,
`remove(FilterCriteria)`), so the layered-prediction requirement is implementable — but
only against persistence services that implement that interface. Open: survey which
services qualify (tasks 2.4), and whether the layered-series surface should require a
modifiable service or offer a fallback (e.g., a cap held in the engine's view without
rewriting stored history).

## 2. Writer precedence on a layered series

Multiple writers target the same series: the baseline generator, the live forecast
refresh, a cap writer, later the learning layer. Two stated behaviours can collide —
"fresh overwrites old" says a forecast refresh replaces entries; the cap scenario says
those same entries hold the capped value. As written, a refresh arriving after the cap
would silently erase it. Options: re-apply caps after any refresh, give writers a
precedence order, or model caps as a separate constraint series composed at read time.
Undecided — flagged for the prototype to surface (tasks 2.3).

## 3. What the wave-1 prototype surfaced

> **ANSWERED ELSEWHERE — owner decisions D13 and D16** (2026-08-02,
> `docs/OWNER_DECISIONS.md`), both of them by requirements that live in other changes; the
> questions are recorded here because this is where a forecast user meets them.
>
> _Which readings are safety inputs_ (E12, D13): neither the provider nor the consumer
> classifies them. **The engine owns the enumeration of what counts as a safety input, and a
> contributor may not claim that status for itself** — the same boundary that makes the
> `never` flag, the user level gate, the readiness interlock and device protections
> engine-owned. A contributed forecast series is therefore data-plane unless the engine's own
> enumeration says otherwise. The requirement is `define-engine-contract`'s.
>
> _Where degradation is reported_ (B12, D16 · pack A8): an outcome enum plus a reason on
> every decision, published as deduplicated events, with **one status Item** as the
> user-visible surface. So a site planning on a year-long baseline because a forecast add-on
> has been dark for a week finds out there. The requirement is `define-extension-points`'
> and `define-engine-contract`'s.
>
> Both bullets stay below as written, since neither change to this change's own requirements
> follows from them.

Opened by building the wave-1 slice against this corpus, not by #3478. Ids are the
prototype's own (see `docs/PROTOTYPE_FEEDBACK.md`).

- **Nothing says which of a contributor's readings are safety inputs (E12).**
  `extension-surface` _Graceful degradation on contributor loss_ splits contributor loss two
  ways: a data-plane source going away degrades to baselines — this change's _Forecast
  source fails_ scenario — while a lost safety input degrades to a conservative safe state.
  The corpus never says which side a given contributed reading falls on. A forecast series
  is assumed data-plane and never declared as such, and the safety branch has no staleness
  age behind it, so an implementation meeting a new source cannot classify it. Open: does a
  forecast provider's surface state that its readings are never safety inputs, or does the
  classification belong to whatever consumes the series?
- **Degradation has no reporting surface (B12).** _Source-agnostic consumption_ makes
  sources swappable and `extension-surface` requires the engine to "report the degraded
  source" — no change says where. A site planning on a year-long baseline because a forecast
  add-on has been dark for a week has no defined way to find that out, and the baseline
  fallback is precisely this change's headline behaviour. Recorded in
  `define-extension-points`; noted here because this is where a user would notice.
