# Define the extension points

**Wave 1** — how everything that is *not* core plugs in. This change frames Kai's one
stated architectural uncertainty without answering it.

## Why

The agreed placement is a core framework with the surrounding functionality contributed
from outside: "we could have many bindings that each bring their own `EnergyProvider`"
([1483930195](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1483930195)),
providers and profiles "ideally implemented by bindings, but we could also have
implementations that are done through scripts", consumers "likely … fairly individual
scripts" ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)),
and in 2026: "we might introduce **new types of add-ons** to make the EMS extensible —
but we will have to work out to what extend this makes sense"
([5016907260](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016907260)).
For that to hold, the *behaviour* of the extension surface must be specified —
contribution at runtime, coexistence, lean interfaces, graceful degradation — while the
*mechanism* (plain bindings vs. new add-on types vs. scripts vs. service whiteboard)
stays an open design question.

## What changes

- Define the `extension-surface` capability: runtime contribution and removal, multiple
  contributors coexisting under user selection, contributor-owned complexity, graceful
  degradation, and core-shipped defaults that don't privilege themselves.

## Non-goals

- Deciding the add-on-type mechanism — that is THE open question, held in `design.md`.
- The engine behaviours themselves (`define-engine-contract`).
- Price/forecast specifics — those changes define the data; this one defines how their
  sources arrive and leave.

## Impact

- The contract that keeps the core lean and the ecosystem contributable; pairs with
  `define-engine-contract` as the wave-1 spine.
