# Design notes — the mechanism question (deliberately open)

## 1. THE open question: what carries a contribution?

Kai 2026: "we might introduce new types of add-ons to make the EMS extensible — but we
will have to work out to what extend this makes sense"
([5016907260](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016907260)).
Candidate mechanisms, none decided:

| Mechanism | Precedent | Notes |
|---|---|---|
| a. Plain bindings publishing to channels/items | The 2023 sketch's default (price bindings → linked items) | Zero new infrastructure; contribution = items, selection = item configuration |
| b. New EMS add-on types with a dedicated SPI | persistence services / transformations / voice add-ons | First-class lifecycle + typed contracts; the "new types of add-ons" idea |
| c. Scripts / rule templates | automation add-ons, marketplace rule templates | Kai's first-class path for consumers and algorithms; API must be script-friendly |
| d. OSGi service whiteboard | core registries (Item/Thing/Metadata providers) | The runtime register/unregister behaviour in this change matches this precedent |

Likely not either/or: (a) for data-by-items, (d) beneath (b) for typed contributions,
(c) on top via a script-accessible registry. To be worked out with maintainers.

## 2. Naming

Kai himself questioned `EnergyConsumer` — "I am wondering whether something like
`DemandDescription` might be actually a better wording?"
([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)).
Parked here so the naming pass happens consciously.

## 3. Are actuation adapters an extension point?

Device-quirk handling on the *write* side (ACK windows, mode mapping — see
`define-engine-contract` §3) could itself be contributable, e.g. an EEBus add-on acting
as "one implementation behind the interfaces"
([2016826350](https://github.com/openhab/openhab-core/issues/3478#issuecomment-2016826350)).
Open.

## 4. Selection/composition UX

"User selects which contributions are used" needs a home: bridge-style configuration,
per-participant metadata, or MainUI flow. Ties to the discovery change
(`discover-participants-from-model`) for the propose-and-confirm surface.
