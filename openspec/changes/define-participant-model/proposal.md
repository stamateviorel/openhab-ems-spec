# Define the energy participant model

**Wave 1** — the spine everything else attaches to. Mirrors steps 2 + 4 of the
[2023 backlog](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1492955803).

## Why

Every EMS discussed in #3478 — storm.house, spot-price-optimizer, emsmanager, and the
2023 architecture sketch — independently arrived at the same shape: **providers**
(grid / PV / battery) and **consumers** carrying a small **power profile** that tells a
central engine how the device may be steered. openHAB has no native way to express this;
today it lives in per-user rules. The 2023 sketch proposed the concepts
([design](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374));
one reference implementation has since validated them against a live building. This
change turns that converged shape into reviewable requirements.

## What changes

- Define the `energy-participants` capability: how existing Items are declared as
  energy providers/consumers, the provider roles, and the four consumer profile classes
  with their protection parameters.
- Requirements are mechanism-neutral: *that* users mark Items and *what* must be
  expressible — whether via item metadata, description providers, or a new add-on type
  is an open question in `design.md`.

## Non-goals

- Energy levels (own change, same wave), price/forecast data planes (wave 2),
  named per-device profile variants (wave 3), learning (wave 4).
- Choosing the optimization algorithm — the model must merely carry enough
  information for pluggable engines/algorithms to work with.
- Dashboards/UI (called out early as its own module; deliberately out of this change).

## Impact

- New concepts in openhab-core (interfaces + registry); no existing behavior changes.
- Bindings/scripts/add-ons can later implement providers without core changes.
