# Design notes — open questions

## 1. Where does the UI ship from?

Two proven paths, undecided:

| Path | Precedent | Trade-off |
|---|---|---|
| a. Built into MainUI (openhab-webui) | HA's energy dashboard model; native look, settings integration | Couples release cycles; webui maintainers own it |
| b. Served by the framework via the core UI-component provider mechanism | Running in production in the reference (pages appear/disappear with the bundle, no JSONDB) | UI ships with the EMS; webui stays untouched |

Path (b) is field-proven and keeps mstormi's "UI time-sink" risk contained; path (a) is
the more native end state. They can sequence: (b) first, (a) if/when adopted.

## 2. What the overview must show (minimum set)

Candidate minimum: live flows (grid/PV/battery/consumers), current + planned energy
level, today's cost/self-consumption per the active objective, per-participant state.
Deliberately not specified as requirements yet — needs one round of user feedback, not
committee design (the time-sink warning).

## 3. Aggregated-series mechanics

"Bounded point count" needs a mechanism: pre-computed rollup series (reference's
approach: one point/day written at day end), on-demand server-side aggregation, or
persistence-service-level support. Interacts with what persistence services can do —
survey needed (task 1.3).

## 4. Relation to existing community widgets

mstormi pointed at the community's animated energy widget as a starting point
([1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249));
the reference ships a standalone energy-flow widget. Whether the out-of-the-box page
embeds such a flow visual or stays plainer is design freedom, not spec.
