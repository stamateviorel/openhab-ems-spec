# Define the price data plane

**Wave 2** — step 3 of the [2023 backlog](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1492955803),
now that its step 1 (future-timestamped TimeSeries) has landed in core (openHAB 4.1).

## Why

This is the issue's founding need: future energy prices from many sources behind one
interface, with calculations implemented once ([issue body](https://github.com/openhab/openhab-core/issues/3478)).
Reality documented in the thread: the consumer price is a **sum of components** from
possibly different sources (spot + grid tariff + taxes), tariffs are regional and
sometimes conditional ("winter Saturday daytime"), units and VAT vary, and feed-in can
even cost money. Several bindings already fetch prices; what is missing is the common
model and the shared math.

## What changes

- Define the `price-data` capability: prices as future-timestamped TimeSeries in
  `Number:EnergyPrice`, component composition, a generic configurable grid-price
  provider, shared window calculations, and feed-in pricing.

## Non-goals

- Currency exchange services (explicitly deferred in 2023; `Number:Currency` exists).
- Level derivation (wave 1 consumes this plane).
- Non-electricity resources (gas noted as future direction, wborn's generalization).

## Impact

- Core interfaces + possibly one generic core provider; existing price bindings
  (EnergiDataService, Tibber, ENTSO-E, …) become sources without breaking changes.
