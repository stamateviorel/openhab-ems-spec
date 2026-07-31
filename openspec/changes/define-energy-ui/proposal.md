# Define the out-of-the-box energy UI

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

> **Provenance note.** The requirements are thread-sourced; the *feasibility* evidence
> (pages served and removed with a bundle, participant-driven live rebuild, fast
> long-range views) is **reference-production-sourced** and flagged inline.

## What changes

- Define the `energy-ui` capability: pages that exist without dashboard-building, derive
  from the declared participants, show past and future in one view, and stay fast at
  long ranges.

## Non-goals

- Visual design, layout, theming — deliberately unspecified (design freedom; the
  time-sink warning).
- Widget-level composition for power users — the existing MainUI widget system already
  covers it.
- Where the UI code lives (core webui vs. framework-served pages) — open question in
  design.md.

## Impact

- Consumes the participant model, levels and both data planes; the first thing most
  users will judge the whole framework by.
