# Evidence from shipped openHAB bindings

Everything in this corpus traces to [openhab-core#3478](https://github.com/openhab/openhab-core/issues/3478),
which is a fair thing for a maintainer to hold against it: one thread's opinion is not a community
spec. This page widens the provenance. Each row is a **shipped or proposed openHAB add-on** that had
to solve the same problem in production, found with a search across the `openhab` organisation on
2026-10-03.

Nothing here is a requirement yet. These are corroborations and contradictions to fold into the
named requirements on a later pass, recorded now so the sourcing is not reconstructed from memory.

A note on what the search found: across the whole organisation there is **exactly one design thread**
on energy management, #3478 itself. Everything else is binding-level work. That is itself worth
knowing — the design conversation has never had a second venue.

## Price resolution — the 15-minute market is live, not theoretical

| Source | What it shows |
|---|---|
| [`energidataservice` #18379](https://github.com/openhab/openhab-addons/pull/18379) — *Add support for variable spot price resolution* (merged 2025-04-19) | Preparatory work for 15-minute spot prices, done six months ahead of the change. A shipped binding treated resolution as a variable rather than a constant. |
| [`energidataservice` #19623](https://github.com/openhab/openhab-addons/pull/19623) — *Fix calculations for quarter-hourly spot prices* (merged 2025-11-06) | **Spot prices moved to 15-minute resolution on 1 October 2025 and the binding's calculations broke.** A regression of #18695, found in production. |

Bears on **_Time resolution_** (`define-price-providers`) and on `define-energy-levels` §7, where
hours-versus-slots is recorded as narrowed rather than answered. The corpus treats the slot unit as a
modelling choice; this is evidence that it is a live migration that has already broken working code
once, in an implementation written by the person who opened #3478.

## Price composition — total, spot and tax must be separable

| Source | What it shows |
|---|---|
| [`tibber` #19194](https://github.com/openhab/openhab-addons/pull/19194) — *Add energy price and taxes* (merged 2025-09-03) | Splits the price group into `total`, `spot` and `tax` because conflating them hid a real case: **a negative spot price under a positive total.** Shipped as a deliberate semantic break — "the notion of `spot-price` is changed, it is no longer holding the total price" — and pushed into a patch release to stop the confusion spreading. |

Bears on **_Price component composition_** (`define-price-providers`), whose *Components sum below
zero* scenario asserts exactly this and cites only the thread. Here is a binding that shipped the
same conclusion, under user pressure, and judged it urgent enough to break a channel's meaning.

## Provider abstraction — more than one source behind one interface

| Source | What it shows |
|---|---|
| [`aWATTar` #21843](https://github.com/openhab/openhab-addons/pull/21843) — *Add Energy-Charts API* (open, October 2026) | A second price provider behind **a common API interface**, with intervals derived from the returned timestamps so 15-minute data survives into the bridge's time series. |

Bears on **_Generic grid-price provider_** and on `define-extension-points` §1, the open question
about what carries a contribution. This is option (a) — a plain binding — being exercised right now
for the same data source the reference implementation reads.

## Capacity tariffs — the billed quantity is readable from the meter

| Source | What it shows |
|---|---|
| [`dsmr` #14279](https://github.com/openhab/openhab-addons/issues/14279) — *New channel types for peak demands (capacity tariff Belgium)* | Since 1 January 2023 the Belgian distribution tariff is billed on **the 15-minute average peak load of each month**, and the P1 port now publishes it directly: `1-0:1.4.0.255` current average demand, `1-0:1.6.0*255` maximum demand of the running month. |
| [`dsmr` #15038](https://github.com/openhab/openhab-addons/pull/15038) — *Add support for capacity tariff for Belgium* | Adds the two channels so a user "can steer your consumption" against them. |

Bears on **_Peak-based fee models_**, **_Capacity budget denominated in the billed quantity_** and
**_Metering slots follow the supplier's clock_** (`define-grid-constraints`), and it closes part of
that change's task 1.1, which asks for constraint archetypes to be collected from the community.

This is the strongest corroboration on the page, and it sharpens the requirement rather than merely
agreeing with it. The corpus has the planner **reconstruct** the billed quantity from power samples,
aligned to the supplier's quarters. On a meter with a P1 port, the billed quantity does not need
reconstructing — **the meter computes and publishes it**, including the month-to-date maximum the bill
is actually based on. A conforming implementation should prefer the meter's own figure where one
exists and fall back to reconstruction where it does not, and the corpus currently says nothing about
that choice.

## Device modes — SG-Ready is settable today

| Source | What it shows |
|---|---|
| [`stiebeleltron` #16758](https://github.com/openhab/openhab-addons/pull/16758) — *Add SG-Ready to the Stiebel-Eltron ISG binding* | SG-Ready states are writable on a shipped heat-pump binding. |

Bears on **_SG-ready mode mapping_** (`define-energy-levels`) and on **_Level-to-mode mapping for any
mode count_** (`define-participant-model`). The four-level scale maps onto something a user can
already drive from openHAB, which is the difference between a modelling convention and an interface.
