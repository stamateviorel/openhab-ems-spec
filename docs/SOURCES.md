# Source ledger

Every requirement in this repo traces back to one of these origins. Permalinks are to
[openhab-core#3478](https://github.com/openhab/openhab-core/issues/3478) comments unless
stated otherwise. If you find your idea uncredited or miscredited, please open an issue —
that is a bug.

## People and what they contributed

| Who | Contribution | Key comments |
|---|---|---|
| **@jlaur** | Opened #3478; price components/elements model; program-mapped load curves (dishwasher); select-a-named-program idea; auto-mapping programs from persisted runs; gas / CO₂ / green-energy datasets; negative feed-in prices | [issue body](https://github.com/openhab/openhab-core/issues/3478), [1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141), [1482016387](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1482016387), [1489220814](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1489220814) |
| **@kaikreuzer** | 8-point needs summary; the architecture sketch: `energy:` metadata namespace, `EnergyManagementParticipant` registry, `EnergyProvider`/`EnergyConsumer`/`PowerProfile` interfaces, Simple/Controllable power profiles, generic `GridEnergyProvider`; "don't wait for me — drive it forward"; spec-driven process + core placement + new add-on types (2026) | [1481931209](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931209), [**1481931374** (the design)](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374), [2016826350](https://github.com/openhab/openhab-core/issues/3478#issuecomment-2016826350), [5012758420](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012758420), [5016907260](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016907260) |
| **@lsiepel** | Demand / Provider / Engine decomposition; lean interfaces, provider owns its complexity; the three integration options (metadata vs. new add-on type vs. Thing annotation); dashboards module | [1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215), [1481931283](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931283) |
| **@mstormi** (storm.house) | 4 energy levels ↔ SG-ready ↔ EVCC mapping; user-configurable hours per level; base-level-from-tariff + escalate-on-PV-excess; hour-chunk scheduling; AN-AUS consumer pattern (threshold, Schaltniveau, startklar, min/max runtime, cooldown, max-off, catch-up, priority); named profiles + TimeSeries×scale per profile; AI learning layer vision | [1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249), [1481931278](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931278), [5012316336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012316336), [5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379), [docs](https://storm.house/docs/) |
| **@masipila** (spot-price-optimizer) | Future-timestamped TimeSeries as THE storage model (fresh-overwrites-old); planned-schedule vs. current-mode decoupling; contributed-algorithms library; the four user stories (consecutive EV/boiler, N-cheapest-hours heat pump, SG-ready threshold); tariff composition (Caruna season); timeResolution & 15-minute readiness; multi-objective load balancing (maxPower + schedulingPriority); the 4-step backlog; heating need as a forecast TimeSeries with solar-forecast as passive-gain proxy (2026) | [1481931272](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931272), [1481931313](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931313), [1481931320](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931320), [1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363), [1492955803](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1492955803), [5017073428](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5017073428) |
| **@mherwege** | Belgian capacity tariff: billed monthly peak of 15-minute power buckets; engine flexibility for pluggable regional logic | [1481931336](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931336) |
| **@seime** | Norwegian tiered grid fee (average of 3 highest hours/month); limit-load-within-window need; region/date-dependent config updatable outside the release cycle | [1481931351](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931351) |
| **@J-N-K** | Money/currency as UoM (`MoneyQuantity` → landed as `Number:Currency` / `Number:EnergyPrice` in openHAB 4.1) | [1484171583](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1484171583) |
| **@wborn** | Generalize beyond electricity — resource management (heat, water, …) as a future direction | [1696830864](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1696830864) |
| **@stamateviorel** (openhab-binding-emsmanager) | Working realization of the 2023 sketch used to pressure-test these requirements: `energy:` namespace parser, the four profile classes, energy levels with percentile/seasonal windows, shadow-mode validation methodology, capacity-tariff controller, online-learned RC thermal model + pre-heat planner | [5005793264](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5005793264), [repo](https://github.com/stamateviorel/openhab-binding-emsmanager) |

## Already landed in openHAB core (build on, don't reinvent)

- **Future-timestamped TimeSeries + `forecast` persistence strategy** (openHAB 4.1) — the
  capability step 1 of the 2023 backlog asked for; originally discussed around
  [openhab-core#3000](https://github.com/openhab/openhab-core/pull/3000).
- **`Number:Currency` / `Number:EnergyPrice`** (openHAB 4.1) — the currency question from
  the thread, resolved ([confirmation](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1963625777)).
- **Item metadata namespaces** — the mechanism Kai's sketch proposed for marking energy
  participants.
- **Semantic model, persistence services, UoM** — assumed throughout.

## Adjacent prior art (referenced, not absorbed)

- [EVCC](https://evcc.io) charge modes (off / pv-surplus / min+pv / max) — level-mapping
  evidence in [1481931278](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931278).
- SG-ready (Smart-Grid-ready heat pump modes) — the 4-mode industry standard the level
  model maps onto.
- EEBus — deliberately split out of scope by
  [2016826350](https://github.com/openhab/openhab-core/issues/3478#issuecomment-2016826350);
  could later be one implementation behind the provider interfaces.
- OpenEMS "Natures" — the capability-interface analogy.
