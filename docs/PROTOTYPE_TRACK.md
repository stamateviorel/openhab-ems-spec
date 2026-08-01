# Prototype track — rules of engagement

This page tells a coding assistant (or a human) how to build **from** this corpus without
overstepping it. It exists because every change's `tasks.md` opens with "maintainer
review": read literally, nothing could ever be built. Two tracks resolve that.

## The two tracks

| | **Prototype track** | **Merge track** |
|---|---|---|
| Goal | A throwaway draft that makes review cheap and surfaces spec ambiguities | Code that lands in openhab-core / openhab-addons |
| Gate | None — proceed to each change's `## 3. Prototype path` tasks | Maintainer decisions (each change's `## 1. Alignment` tasks) |
| Output | A local/private branch, offered for discussion | A pull request |
| Actuation | **Shadow only** — never writes to a device | Normal, after review |

**Only the prototype track is open today.** It is not a shortcut around the process: it
is step three of the process the maintainer proposed — requirements → PRD →
"have coding assistants work through it to come up with a meaningful implementation"
([#3478 comment 5012758420](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5012758420)).

## Hard rules

1. **Never resolve an open question.** Anything in a `design.md` is a maintainer's
   decision. Where a decision is unavoidable to compile, implement **all** framed options
   behind a seam (interface + two implementations) and leave the choice to configuration.
   Never silently pick one and move on.
1. **Shadow only.** The prototype computes and logs; it does not write to items. The
   master stop and shadow default from `engine-contract` are implemented first, not last.
1. **Fixtures are pass/fail.** Anything in [`fixtures/`](../fixtures/) is a conformance
   test, not an illustration. A level classifier that does not reproduce
   `expected-planned-levels.csv` from `dayahead-prices.csv` is wrong, not "close".
1. **Requirements are the contract; scenarios are the tests.** Each `#### Scenario:`
   should map to a unit test. If a scenario cannot be tested, say so in the report —
   that is a spec defect worth more than the code.
1. **Follow openHAB's rules from the start.** They are injected via
   [`openspec/config.yaml`](../openspec/config.yaml): Java 21, OSGi DS, SLF4J
   (`logger`, parameterized), `@NonNullByDefault`, `.internal` packages, JavaDoc +
   `@author`, Spotless, no thread creation, approved libraries only, EPL-2.0 headers,
   DCO sign-off. Retrofitting these later wastes the prototype's credibility.
   The governing reference is [`docs/REVIEW_CHECKLIST.md`](REVIEW_CHECKLIST.md) — the
   maintainers' own rules and 44-item review checklist. Note their stated priority:
   user-facing consistency, then architecture consistency, then runtime behaviour, and
   explicitly _not_ individual code style.
1. **Report ambiguity instead of inventing.** Every place the spec was unclear,
   contradictory or silent goes into the build report. Those become spec changes — the
   loop closing is the point.
1. **Nothing public without the owner's go.** No pushes to shared repos, no PRs, no
   comments on the thread.

## Definition of done (all must hold)

- Builds inside a real `openhab-core` checkout with its static analysis clean
  (priority-1 findings fail the build).
- Unit tests green, including one test per implemented scenario.
- **All four fixture vectors reproduced exactly.**
- A shadow-mode demonstration: decisions computed and logged, zero writes.
- Zero open questions resolved; every seam documented.
- A written report: what was built, what the spec got right, where it was ambiguous, and
  what should change in the corpus.

## Suggested first slice (wave 1)

`define-participant-model` + `define-engine-contract` + `define-energy-levels`, with
`define-extension-points` shaping the seams:

1. The participant model as core-style interfaces (participant, provider, consumer, the
   four profile classes).
1. **Both** declaration mechanisms behind one interface — item metadata (`energy:`) and a
   description-provider SPI — per wave-1 `design.md` §1. This is the seam that keeps the
   maintainer's decision open.
1. The engine: one evaluation pass over a consistent snapshot, deterministic
   priority-ordered conflict resolution, the electrical-limit floor, shadow gating,
   master stop.
1. The level classifier as a pure function — **gated on the fixtures**.
1. A trivial script-contributed algorithm, proving algorithms are replaceable
   (`engine-contract`: scripts first-class).

Waves 2–4 stay unbuilt until wave 1 has been through review.
