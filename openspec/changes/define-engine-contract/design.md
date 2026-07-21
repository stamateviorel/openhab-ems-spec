# Design notes — open questions

## 1. Evaluation cadence

Kai suggested pull-based scraping "possibly up to every minute", with push "in a future
iteration" ([1482078918](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482078918)).
The reference runs a short fixed tick plus a debounced re-evaluation on relevant item
changes. Open: fixed tick only, tick + event-responsive, and what the default cadence is.
The requirement stays cadence-neutral ("regular cadence") until decided.

## 2. Shadow granularity and the master stop

Is the master stop the same control as global shadow mode (the reference's approach), or
a separate mechanism? And is shadow global-only, or also per-algorithm/per-participant
(the reference supports both layers)? Needs a decision before API design.

## 3. Where acknowledgement handling lives

The ACK window could live in the core engine, or in per-device adapters (the reference
puts it in asset handlers so "controllers stay ignorant of item names and write
mechanics"). Related: whether actuation adapters are themselves an extension point (see
`define-extension-points`).

## 4. Context snapshot content

"Same consistent snapshot" needs a defined minimum: live powers, prices, forecasts,
levels, per-participant state. The reference's context object is prior art; the core
version should be decided with the participant-model mechanism (wave-1 design.md §1).

## 5. Provenance recap

Thread-sourced: central evaluation, conflict resolution, limits (budget/phase), shadow
mode, replaceable algorithm. Reference-sourced (flagged inline): master stop, ACK
actuation, and the "regardless of algorithm" limit generalization. Reviewers should
treat the flagged three as field-proven proposals, not thread consensus.
