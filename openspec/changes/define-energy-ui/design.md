# Design notes — open questions (answered 2026-08-02)

> **Decision pass, 2026-08-02.** The owner answered this change's open questions; the
> record is [`docs/OWNER_DECISIONS.md`](../../../docs/OWNER_DECISIONS.md). Everything
> marked _Answered_ is an **owner decision in a reference implementation, not thread
> consensus** — and for this change in particular that matters, because §2 was deliberately
> left to one round of user feedback rather than committee design. Nothing was deleted; the
> paths and candidates each section carried are left standing.

## 1. Where does the UI ship from?

Two proven paths, undecided:

| Path | Precedent | Trade-off |
|---|---|---|
| a. Built into MainUI (openhab-webui) | HA's energy dashboard model; native look, settings integration | Couples release cycles; webui maintainers own it |
| b. Served by the framework via the core UI-component provider mechanism | Running in production in the reference (pages appear/disappear with the bundle, no JSONDB) | UI ships with the EMS; webui stays untouched |

Path (b) is field-proven and keeps mstormi's "UI time-sink" risk contained; path (a) is
the more native end state. They can sequence: (b) first, (a) if/when adopted.

**Answered — D17** (Part B, UI-1). _Decision:_ **path (b), framework-served via the core
UI-component provider mechanism — and in a bundle separate from the engine**, so
sequencing into MainUI later is a **deletion, not a refactor**. The separate-bundle clause
is the part that is not in the table above and is the whole reason the answer is cheap to
reverse. _Evidence:_ this is running on the owner's building today — provider-served pages,
never written to JSONDB, rebuilt live from `energy:` tags. _Alternative preserved:_ path
(a) stands as the more native end state, and the sequencing note above is unchanged; a
maintainer who wants MainUI to own the pages deletes a bundle rather than untangling one.

## 2. What the overview must show (minimum set)

Candidate minimum: live flows (grid/PV/battery/consumers), current + planned energy
level, today's cost/self-consumption per the active objective, per-participant state.
Deliberately not specified as requirements yet — needs one round of user feedback, not
committee design (the time-sink warning).

**Answered — D17** (Part B, UI-2), and now specified as a requirement. _Decision:_ the
overview shows, at minimum — **the site level; live per-participant power; today's cost or
self-consumption per the active objective; the active grid constraint's headroom (the
month-to-date peak for a capacity tariff); and one status line for degraded sources and
declaration errors**. The first three are this section's own candidate set; the headroom KPI
is shipped in the reference implementation's `Now` tab; the status line is where the
observability surface (D16 pack A8) lands for a human. _Alternatives preserved:_ the
candidate set above, including the animated live-flow visual (§4), stands — the decision
adds two items and drops none, and the freeze-after-feedback intent of task 1.2 is
unchanged, since a minimum set is a floor rather than a design.

## 3. Aggregated-series mechanics

"Bounded point count" needs a mechanism: pre-computed rollup series (reference's
approach: one point/day written at day end), on-demand server-side aggregation, or
persistence-service-level support. Interacts with what persistence services can do —
survey needed (task 1.3).

**Answered — D18** (Part C, UI-3). _Decision:_ **pre-computed rollup series, one point per
day into dedicated items.** The blocking survey is closable from core's SPI alone:
`FilterCriteria` carries only itemName / dates / paging / operator / ordering / state —
**no aggregation field** — so the other two options require changing an interface every
persistence add-on implements. Two non-obvious details belong in the requirement rather
than in an implementation: **snapshot before the day boundary**, and **derive the increment
from the last reading before the reset**. _Alternatives preserved:_ on-demand server-side
aggregation and persistence-service-level support both stand above, and both become
available the day `FilterCriteria` grows an aggregation field.

## 4. Relation to existing community widgets

mstormi pointed at the community's animated energy widget as a starting point
([1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249));
the reference ships a standalone energy-flow widget. Whether the out-of-the-box page
embeds such a flow visual or stays plainer is design freedom, not spec.

Unchanged by the decision pass: still design freedom, and deliberately not a requirement.

## 5. Surfaces the wave-1 prototype found unnamed

Opened by building the wave-1 slice against this corpus, not by #3478. Ids are the
prototype's own (see `docs/PROTOTYPE_FEEDBACK.md`). Four kinds of user-visible information
exist in the wave-1 requirements with no stated place to appear. They are recorded here
because §2's minimum set is where they would land — not because this change alone should
answer them.

- **A degraded source (B12).** `extension-surface` requires the engine to report one and
  names no surface: Item, event, REST resource or log.
- **A declaration in error (B11).** The corpus has no notion of one. A mistyped declaration
  silently un-manages a device and there is nowhere it surfaces — which lands on _Guided
  setup surface_, the requirement that creates declarations in the first place.
- **What became of a decision (N6).** Applied, superseded, deferred, suppressed, withheld,
  stopped — the corpus has no vocabulary for it, though `engine-contract`'s _Side-by-side
  validation_ asks a user to compare the engine's decisions against their existing
  automation. Filed under `define-engine-contract`.
- **Shadow output (E18).** The same scenario needs shadow decisions to stay comparable over
  weeks; the only surface the requirement names is the log, and at a one-minute cycle the
  same decision is logged for hours. Filed under `define-engine-contract`.

**All four answered — D16 (pack A8), one surface for all of them.** The engine publishes an
**outcome enum plus a free-text reason on every decision**
(`APPLIED, SHADOWED, STOPPED, SUPERSEDED, DEFERRED, SUPPRESSED, WITHHELD, REJECTED`) as
**deduplicated events** — an unchanged decision is not re-emitted, which is exactly what
makes a weeks-long shadow run readable (E18) — with the current cycle readable over REST,
plus **one engine-published status Item** (String summary + Number count). Core writes no
Item for decisions, so "shadow writes nothing" stays literally true. A **malformed
declaration skips the whole participant** and is reported through a `ConfigStatusProvider`
for the `energy:` namespace, keyed on the item, which is what UIs already render
per-parameter (B11). This change's part is the human end of it: the status line in §2's
minimum set, and the declaration errors surfaced on _Guided setup surface_ where the
declaration was made. The normative requirements live in `define-engine-contract` and
`define-extension-points`. _Alternative preserved:_ logs only, rejected because it leaves
the corpus's own side-by-side validation scenario untestable.
