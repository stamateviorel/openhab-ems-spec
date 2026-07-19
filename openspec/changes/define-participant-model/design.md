# Design notes — open questions first

These are the contested decisions. They are deliberately **not** encoded in the
requirements; they need maintainer alignment before any implementation change is
proposed.

## 1. Where does the declaration live?

Three options were named in the thread
([lsiepel, 1481931283](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931283)),
plus one from the sketch:

| Option | Sketch | Evidence |
|---|---|---|
| a. Item metadata namespace (`energy:`) | Kai's 2023 proposal | Implemented and field-proven in emsmanager; zero new concepts for users |
| b. "Power description" providers (analogous to state descriptions) | Kai's alternative ([1481931263](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931263)) | Lets bindings ship profiles programmatically; composes with (a) |
| c. Thing/channel annotation by add-ons | lsiepel | Rejected for "logical devices" without Things ([1481931263](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931263)) |
| d. New add-on types for EMS extensibility | Kai 2026 ([5016907260](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016907260)) | "we will have to work out to what extent this makes sense" |

Working hypothesis worth testing: (a) for user declaration, (b) as the SPI so add-ons
can contribute the same information, (d) for the pluggable pieces (price sources,
forecast sources, algorithms) — mirroring how persistence/transformation services are
add-on types today. Flat metadata demonstrably struggles with wave-3 *named profiles
carrying TimeSeries payloads* — one reason not to hard-commit to (a) alone.

## 2. Core vs. add-on boundary

Kai 2026: the final solution belongs in openhab-core, not "yet another binding"
([5016907260](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016907260)).
The reference implementation's internal seams suggest a concrete split to discuss:

- **Core:** participant model + registry, profile classes, energy levels, the engine
  contract (and the "direct communication in the background" lsiepel asked for in
  [1492968827](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1492968827)).
- **Add-ons:** price providers, forecast providers, regional tariff/constraint logic,
  optimization algorithms (masipila's "contributed algorithms",
  [1481931363](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931363)).

## 3. Engine simplicity

The sketch wanted the `EnergyManagementService` so simple it "could possibly even be a
rule template" ([1481931374](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931374)) —
scripts must be able to implement alternative algorithms. Whatever lands must keep the
engine swappable rather than monolithic.

## 4. Validation methodology (from the reference)

Before any engine variant steers a live building: unit tests on the pure logic, then a
shadow mode running beside the user's existing automation comparing every decision, then
live control behind a master kill switch. This proved out in production
([5014707411](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707411))
and should be a first-class acceptance path in the implementation plan, not an
afterthought.
