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
> discussion, or restructure it however the maintainers prefer. Nothing here is a
> decision; every requirement is traceable to the person who first stated it (see
> [`docs/SOURCES.md`](docs/SOURCES.md)).

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
| 1 | [`define-energy-levels`](openspec/changes/define-energy-levels/) | The 4-level energy-availability model (SG-ready / EVCC compatible) |
| 1† | [`discover-participants-from-model`](openspec/changes/discover-participants-from-model/) | Auto-propose the participant set from openHAB's semantic model (assisted setup) |
| 2 | [`define-price-providers`](openspec/changes/define-price-providers/) | Price data plane: components, tariffs, common calculations |
| 2 | [`define-forecast-providers`](openspec/changes/define-forecast-providers/) | Forecast data plane: solar/weather/demand as TimeSeries |
| 3 | [`define-grid-constraints`](openspec/changes/define-grid-constraints/) | Capacity tariffs, peak fees, load balancing |
| 3 | [`add-named-profiles`](openspec/changes/add-named-profiles/) | Multiple named profiles per device (programs, seasons) |
| 4 | [`add-learning-layer`](openspec/changes/add-learning-layer/) | Learned settings and learned profiles |

Open architecture questions (core vs. new add-on types, metadata vs. description
providers) are collected in the wave-1 [`design.md`](openspec/changes/define-participant-model/design.md) —
deliberately as questions, not answers.

† `discover-participants-from-model` is a **fresh proposal (2026-07-20), not yet raised in
#3478** — unlike the other changes, whose requirements each trace to a thread comment. It
rests on openHAB's existing semantic model and Kai's metadata/out-of-the-box intent; its
proposal.md says so up front.

## Reviewing a change

Per OpenSpec's [review guide](https://github.com/Fission-AI/OpenSpec/blob/main/docs/reviewing-changes.md),
read each change in this order — and quit early if something is wrong:

1. `proposal.md` — is this the right problem and scope?
2. `specs/…/spec.md` — is "done" defined correctly? (each requirement = one SHALL +
   scenarios; each carries a `Source:` link to the person who stated it)
3. `tasks.md` — is the plan of work sane?

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
