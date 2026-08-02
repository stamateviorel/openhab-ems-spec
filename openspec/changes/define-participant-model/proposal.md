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
- Requirements state _what_ must be expressible. The declaration mechanism is now stated
  too — item metadata for users and a programmatic path for add-ons and scripts, both
  behind one provider interface — while the core/add-on boundary stays open in `design.md`.
- Amended after the wave-1 prototype: participant identity, a per-consumer power figure
  and a phase declaration are stated outright, because requirements in this and other
  changes already depend on them; the questions the prototype could not answer without
  deciding are recorded in `design.md` §5 (see `docs/PROTOTYPE_FEEDBACK.md`).
- Amended again on 2026-08-02 after the owner of the reference implementation answered
  most of those questions (`docs/OWNER_DECISIONS.md`). Identity, precedence, the priority
  scale, the per-class power figure, a declared measurement, units and phases, staleness,
  the observable "forced restart" and the reporting of malformed declarations are now
  definite; `never` lifts off the Simple profile onto every consumer class, which is a
  structural change and not a wording one. **These are owner decisions in a reference
  implementation, not thread consensus** — every requirement they touched says so on its
  Source line, and every rejected alternative stays in `design.md` so any of them can be
  overturned by rewriting one requirement.
- The same pass fixed the **sign convention for every provider role and for setpoints**
  (grid + = export, PV + = producing, battery + = charging, consumers + = consuming,
  devices that disagree normalised at the edge), which is what unblocks a definition of
  surplus anywhere in the corpus. It **inverts the battery direction the decision pack
  recommended**, so both directions are preserved in `design.md` §5.11 — the second of the
  two places where the owner overrode the recommendation put to them. Stated defaults
  landed with it: demand carries exactly three terms, and a level maps onto _n_ ordered
  modes by a stated formula that a two-mode device is excluded from.

## Non-goals

- Energy levels (own change, same wave), price/forecast data planes (wave 2),
  named per-device profile variants (wave 3), learning (wave 4).
- Choosing the optimization algorithm — the model must merely carry enough
  information for pluggable engines/algorithms to work with.
- Dashboards/UI (called out early as its own module; deliberately out of this change).

## Impact

- New concepts in openhab-core (interfaces + registry); no existing behavior changes.
- Bindings/scripts/add-ons can later implement providers without core changes.
