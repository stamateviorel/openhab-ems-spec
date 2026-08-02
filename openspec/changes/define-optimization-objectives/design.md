# Design notes — open questions

## 1. Do energy levels follow the objective?

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

Negative feed-in prices (`price-data`) interact oddly with self-consumption and carbon
objectives (exporting green power vs. consuming it locally). Needs a worked example
before hardening.

## 6. What the wave-1 prototype surfaced

Opened by building the wave-1 slice against this corpus, not by #3478. Ids are the
prototype's own (see `docs/PROTOTYPE_FEEDBACK.md`); recorded, not answered.

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
