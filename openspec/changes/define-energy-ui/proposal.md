# Define the out-of-the-box energy UI (setup + visualization)

**Wave 3** — the part users actually see; the thread's most repeated wish.

## Why

Three independent voices asked for it in one 2023 afternoon: Kai — "it would be nice to
have suitable widgets and whole pages for the Main UI that directly support a good
energy overview **out of the box**"
([1481931227](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931227));
lsiepel — "Dashboard/Insights … **deserves its own area/module**"
([1481931215](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931215));
mstormi — "Same on dashboard. Yes please let's join forces. It's easy to spend
helluvalot of time on UI design"
([1481931249](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931249)).
It is also the wish behind the recurring community question that keeps this topic alive.
Yet the corpus so far had zero UI requirements. This change keeps them minimal —
behaviour, not design — because mstormi's warning about UI time-sinks is real.

> **Provenance note.** The visualization requirements are thread-sourced; the
> _feasibility_ evidence (pages served and removed with a bundle, participant-driven
> live rebuild, fast long-range views) is **reference-production-sourced**, and the
> guided-setup requirement is **owner-directed (2026-07-31)** — both flagged inline.

## What changes

- Define the `energy-ui` capability: pages that exist without dashboard-building, a
  guided setup surface (declare participants and intent, confirm discovery proposals),
  participant-driven content, past and future in one view, and long-range views at
  bounded point counts.

## Non-goals

- Visual design, layout, theming — deliberately unspecified (design freedom; the
  time-sink warning).
- Widget-level composition for power users — the existing MainUI widget system already
  covers it.
- Where the UI code lives (core webui vs. framework-served pages) — **decided by the
  owner 2026-08-02 (D17, Part B UI-1)**: framework-served through core's UI-component
  provider mechanism, from a bundle separate from the engine, so moving into MainUI later
  is a deletion. The MainUI-native path is preserved in design.md §1 as the sequenced end
  state. Owner decision, not thread consensus.

## Impact

- Consumes the participant model, levels and both data planes; the first thing most
  users will judge the whole framework by.
