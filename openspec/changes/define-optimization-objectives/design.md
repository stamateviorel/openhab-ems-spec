# Design notes — open questions

## 1. Do energy levels follow the objective?

`energy-levels` derives the 4-level signal from the **price** schedule (base) plus PV
escalation. Under the carbon objective, should levels derive from the green-share series
instead — or stay price-based while only load *placement* changes? Production systems
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

## 4. Feed-in under non-cost objectives

Negative feed-in prices (`price-data`) interact oddly with self-consumption and carbon
objectives (exporting green power vs. consuming it locally). Needs a worked example
before hardening.
