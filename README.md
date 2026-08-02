# openHAB Energy Management — community spec

A structured, source-credited requirements corpus for **native energy management in
openHAB**, seeded from the discussion in
[openhab-core#3478](https://github.com/openhab/openhab-core/issues/3478) and from three
production EMS implementations built by that thread's participants.

This repo is an [OpenSpec](https://github.com/Fission-AI/OpenSpec) planning repo: the
requirements live as reviewable **change proposals** under [`openspec/changes/`](openspec/changes/),
each one small enough to discuss on its own. Code lands elsewhere (openhab-core /
openhab-addons) — this repo is the shared plan that the code repos catch up to.

> **Status: seed.** Offered to the openHAB community as a starting point — happy to
> transfer this repo to the `openhab` organisation, move its content into a core
> discussion, or restructure it however the maintainers prefer. Every requirement is
> traceable to the person who first stated it (see [`docs/SOURCES.md`](docs/SOURCES.md)).
>
> **Nothing here is a community decision** — but it is no longer true that nothing here is
> decided. The corpus's own open questions were answered on 2026-08-02 by **one person, the
> owner of a reference implementation**, so that it could be built from at all. Those
> answers are labelled as his everywhere they appear, and every option they rejected is
> still in the file next to them. See _Where this stands_ below.

## Why this exists

The #3478 thread (2023 → today) already contains a remarkably complete requirements
set — Kai's architecture sketch, a four-step backlog that was agreed in 2023, four
device-class patterns proven in production, a four-level energy-availability model
running in two independent systems, price/forecast data-plane needs, and (July 2026)
named per-device profiles and an adaptive learning layer. What was missing is a place
where those requirements are **collected, credited and reviewable** instead of spread
over 90 comments. That is this repo.

It follows the structure @masipila proposed and @kaikreuzer approved in 2023
([backlog](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1492955803)):
step 1 (future-timestamped storage) has since **landed in openHAB 4.1** as TimeSeries +
the `forecast` persistence strategy; the changes here pick up at steps 2–4 and extend
them with everything learned since.

## The changes (reading order)

| Wave | Change | What it defines |
|---|---|---|
| 1 | [`define-participant-model`](openspec/changes/define-participant-model/) | The `energy:` namespace, providers/consumers, the four device classes + protections |
| 1 | [`define-engine-contract`](openspec/changes/define-engine-contract/) | The core-owned runtime: evaluation, conflict resolution, the electrical-limit floor, shadow mode, swappable algorithms (scripts first-class) |
| 1 | [`define-extension-points`](openspec/changes/define-extension-points/) | How add-ons and scripts contribute providers/consumers/algorithms at runtime |
| 1 | [`define-energy-levels`](openspec/changes/define-energy-levels/) | The 4-level energy-availability model (SG-ready / EVCC compatible) |
| 1† | [`discover-participants-from-model`](openspec/changes/discover-participants-from-model/) | Auto-propose the participant set from openHAB's semantic model (assisted setup) |
| 2 | [`define-price-providers`](openspec/changes/define-price-providers/) | Price data plane: components, tariffs, common calculations |
| 2 | [`define-forecast-providers`](openspec/changes/define-forecast-providers/) | Forecast data plane: solar/weather/demand as TimeSeries |
| 2 | [`define-optimization-objectives`](openspec/changes/define-optimization-objectives/) | Selectable objective: cost / self-consumption / carbon ("best hours", not "cheapest") |
| 3 | [`define-grid-constraints`](openspec/changes/define-grid-constraints/) | Capacity tariffs, peak fees, load balancing |
| 3 | [`add-named-profiles`](openspec/changes/add-named-profiles/) | Multiple named profiles per device (programs, seasons) |
| 3 | [`define-energy-ui`](openspec/changes/define-energy-ui/) | Out-of-the-box MainUI energy **setup + visualization**: guided setup surface, participant-driven pages, fast long ranges |
| 4 | [`add-learning-layer`](openspec/changes/add-learning-layer/) | Learned settings and learned profiles |

Open architecture questions (core vs. new add-on types, metadata vs. description
providers) are collected in the wave-1 [`design.md`](openspec/changes/define-participant-model/design.md) —
deliberately as questions, not answers.

† `discover-participants-from-model` is a **fresh proposal (2026-07-20), not yet raised
in #3478** — unlike the other changes, whose requirements each trace to a thread comment. It
rests on openHAB's existing semantic model and Kai's metadata/out-of-the-box intent; its
proposal.md says so up front.

**Executable acceptance vectors** live in [`fixtures/`](fixtures/) — @masipila's 2023
worked price/level/scheduling tables, extracted verbatim from the thread and
machine-verified internally consistent. A conforming implementation must reproduce them.

**Building from this corpus?** Read [`docs/PROTOTYPE_TRACK.md`](docs/PROTOTYPE_TRACK.md)
and [`docs/REVIEW_CHECKLIST.md`](docs/REVIEW_CHECKLIST.md) (the maintainers' own review
rules, which govern anything built here) first. It separates the _prototype track_ (open now — a shadow-only draft that makes
review cheap) from the _merge track_ (gated on maintainer decisions), and states the
rules: never resolve an open question, shadow only, fixtures are pass/fail, report
ambiguity instead of inventing.

## Where this stands

The corpus has been through three passes, and a reader can tell them apart at a glance
because each one labels itself on the `Source:` line of every requirement it touched.

**1. Collected and credited.** 100 requirements and 268 scenarios across the 12 changes
above, each requirement one SHALL with a `Source:` line naming the #3478 comment, the
production system or the core capability it rests on.

**2. Built once, and the build fed back.**
[`docs/PROTOTYPE_FEEDBACK.md`](docs/PROTOTYPE_FEEDBACK.md) catalogues what building a wave-1
prototype from this corpus surfaced — roughly seventy places where it was ambiguous,
contradictory or silent — and says, for each, whether a requirement was sharpened, a missing
requirement added, or an open question recorded. Requirements amended for that reason say so,
so **thread-sourced consensus stays distinguishable from prototype-driven repair**. No design
decision was resolved by that pass; it only made the gaps legible.

**3. The open questions were answered — by one person, and the corpus says so.** On
2026-08-02 the owner of the reference implementation answered the ~79 questions the design
files carried, in one session
([`docs/OWNER_DECISIONS.md`](docs/OWNER_DECISIONS.md), 21 rows, D1–D20 plus what stayed
open; the questions as they were put, with the options and evidence behind each
recommendation, are kept in
[`docs/DECISIONS_PENDING.md`](docs/DECISIONS_PENDING.md)). These are **owner decisions in a
reference implementation, not thread consensus**, and they are recorded as a source class of
their own ([`docs/SOURCES.md`](docs/SOURCES.md)): every requirement they changed names the
decision on its `Source:` line, and **every rejected alternative stays in the change's
`design.md`** — so a maintainer who disagrees overturns one by pointing at the option that
was kept, at the cost of rewriting one requirement rather than reconstructing the argument.

Five are flagged as weak on purpose:

- **Three departed from the recommendation put to the owner** — the master stop halting
  evaluation as well as dispatch (D1), the battery sign convention (D11), and surplus
  counting reclaimable battery charging (D10). Each keeps the recommendation it overrode,
  with its argument.
- **One rests on reasoning alone**, with no thread and no production source: no carbon
  credit at a negative feed-in price (D20). Its own requirement says so in its first
  sentence.
- **One commits to a test vector that does not exist yet** (D19). Nothing was invented to
  fill the gap; [`fixtures/`](fixtures/) still holds only the four thread-sourced vectors.

**What is still open is still open.** Of 107 design questions across the 12 changes, **81
are answered, 10 answered in part and 16 still open** — and the open ones say so rather than
being quietly settled. Among them: the price tie-break and the unit of the level-band
counts, the level names, the word that replaces `EnergyProvider`, which deadline forms are
admissible, and where a load curve belongs. Two requirements are consequently **not yet
fully implementable**: `define-energy-levels` _Level derivation_ needs a tie-break named and
is definite only on an hourly series, and its _Surplus escalation_ has no shipped threshold,
so it stays inert until a site configures one. Each says as much on its own `Source:` line
rather than leaving an implementer to find out.

A handful of cross-change reconciliations were also made — places where two decisions landed
on one behaviour from different changes and disagreed. They are marked as reconciliations,
not decisions, and collected as confirmation tasks in the affected `tasks.md` files.

## Reviewing a change

Per OpenSpec's [review guide](https://github.com/Fission-AI/OpenSpec/blob/main/docs/reviewing-changes.md),
read each change in this order — and quit early if something is wrong:

1. `proposal.md` — is this the right problem and scope?
1. `specs/…/spec.md` — is "done" defined correctly? (each requirement = one SHALL +
   scenarios; each carries a `Source:` link to the person who stated it)
1. `tasks.md` — is the plan of work sane?

Feedback: open an issue or a PR against the change folder here, or comment in
[#3478](https://github.com/openhab/openhab-core/issues/3478). When a change is agreed,
it gets archived into `openspec/specs/` — which starts empty on purpose and accumulates
only agreed material.

## Working with it

```bash
npm install -g @fission-ai/openspec@latest
openspec init   # wire up your own AI tool; then /opsx:explore or /opsx:propose
openspec validate --all --strict
```

[`openspec/config.yaml`](openspec/config.yaml) carries the openHAB
[coding guidelines](https://www.openhab.org/docs/developer/guidelines.html) and
[contribution process](https://github.com/openhab/openhab-core/blob/main/CONTRIBUTING.md)
as injected context — any AI generating artifacts here inherits them automatically.

This repo is also a registered-capable OpenSpec **store** (beta) — a planning repo that
code repos can reference. To work against it by name from anywhere:

```bash
git clone https://github.com/stamateviorel/openhab-ems-spec ~/openspec/openhab-ems-spec
openspec store register ~/openspec/openhab-ems-spec
openspec show define-participant-model --store openhab-ems-spec
```

(A code repo can later declare `references: [openhab-ems-spec]` in its own
`openspec/config.yaml` to give agents read-only access to these specs. Store tooling is
beta; the plain files work regardless.)

Prior art referenced throughout: [storm.house](https://storm.house/docs/) (@mstormi's
commercial EMS), [openhab-spot-price-optimizer](https://github.com/masipila/openhab-spot-price-optimizer/wiki)
(@masipila), [openhab-binding-emsmanager](https://github.com/stamateviorel/openhab-binding-emsmanager)
(@stamateviorel — a working realization of the 2023 sketch, used to pressure-test these
requirements in production).

## License

EPL-2.0 — same as openhab-core, so any text or code here can be lifted straight into
the openHAB projects.
