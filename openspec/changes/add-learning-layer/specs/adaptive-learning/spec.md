# adaptive-learning

## ADDED Requirements

### Requirement: Learned thermal model

The system SHALL be able to learn a building thermal model (heat-loss rate / thermal
mass) from persisted indoor temperature, weather and heating-power data, expose its
quality as a confidence and a prediction error, and use it to derive settings such as
maximum heating-off duration and pre-heat lead time.

#### Scenario: Max-off from the model

- **WHEN** the learned model's confidence has reached its converged threshold
- **THEN** the engine can compute how long heating may stay off before the configured
  comfort floor is reached, instead of using a hand-tuned constant

> Source: mstormi's learning-layer vision ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379));
> masipila's two-winters experience ([5017073428](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5017073428));
> prior art: online-learned RC model + pre-conditioning planner in the reference
> ([repo](https://github.com/stamateviorel/openhab-binding-emsmanager)); "error metric"
> split into confidence and prediction error by the owner 2026-08-02 (D17, Part B LL-3,
> docs/OWNER_DECISIONS.md) — alternative preserved in design.md §3.

### Requirement: Learned load profiles

The system SHALL be able to derive a device's load-curve profile from persisted runs
(automatically mapping a program from its recorded consumption) and publish it as a
named profile, updating it as new runs are recorded.

#### Scenario: Program mapped from history

- **WHEN** enough runs of the "60 °C cottons" program are persisted
- **THEN** a named profile with the measured curve becomes available for scheduling,
  replacing the flat-consumption assumption

#### Scenario: SG-ready device with richer telemetry

- **GIVEN** a heat pump controlled only through SG-ready modes but exposing live power or
  metrics over Modbus or a cloud interface
- **WHEN** that telemetry is persisted
- **THEN** the learning layer refines the device's consumption prediction from it, even
  though the control surface carries no power setpoint

> Source: jlaur's founding idea ([1481931141](https://github.com/openhab/openhab-core/issues/3478#issuecomment-1481931141));
> mstormi's "adapt/replace the consumption TimeSeries profile from recorded historic
> data" ([5016228379](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5016228379))
> and SG-ready-but-measured dynamic consumers
> ([5020830338](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5020830338));
> normalized-pattern observation (lsiepel,
> [5003008753](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5003008753)).
> What the learned curve is written into is a relative-time profile shape, not an absolute
> series, following the owner's decision of 2026-08-02 (D16 pack A13, docs/OWNER_DECISIONS.md
> — see `add-named-profiles`).

### Requirement: Learning runs when its subject completes or continuously

Learning SHALL be triggered by the shape of what it learns — per completed run for load
profiles, continuously for the thermal model, and on demand for either — and never as a
nightly batch over both.

#### Scenario: A partial run is not a curve

- **GIVEN** a washing programme the user aborted halfway
- **WHEN** the learning layer considers it
- **THEN** no profile is derived from it, the trigger being a completed run

#### Scenario: The thermal model updates as samples arrive

- **GIVEN** a thermal learner in its recursive/online form
- **WHEN** a new sample arrives
- **THEN** the model updates from it directly, rather than waiting for a nightly pass that
  would discard the recursion

> Source: `tasks.md` 2.2 asked the question (per run / nightly / on demand); decided by the
> owner 2026-08-02 (D17, Part B LL-4, docs/OWNER_DECISIONS.md) — grounded in the reference
> estimator, which is online by construction, and in the fact that a partial run is not a
> curve; the nightly-batch alternative is preserved in design.md §4. Not stated in the
> thread.

### Requirement: Each learner declares the data it needs

A learner SHALL declare the items it reads and the minimum retained history it requires,
and refuse to run — reporting why on the engine's observability surface — when a site's
persistence does not supply them.

#### Scenario: Thermal learner on an under-persisted site

- **GIVEN** a thermal model whose contract is indoor temperature, outdoor temperature and
  heating power persisted at `everyChange` plus `everyMinute` for at least 30 days, on a
  site that persists indoor temperature only
- **WHEN** the learner is asked to run
- **THEN** it does not run, and the missing inputs are reported rather than the model
  producing a confident number from data it does not have

#### Scenario: The requirement it produces is a proposal, not a write

- **GIVEN** a learner refusing to run for want of history
- **WHEN** the user reads the report
- **THEN** it names the items and the retention the learner needs, so the fix is a
  persistence configuration change rather than a guess

> Source: `tasks.md` 1.2 asked which persistence strategies a user must enable; decided by
> the owner 2026-08-02 (D17, Part B LL-2, docs/OWNER_DECISIONS.md) — per-learner contract,
> enforced by refusal, reported on the surface D16 pack A8 defines; alternatives preserved
> in design.md §2. **The 30-day retention figure is judgement, not derived** (Part D
> remainder, left undecided on purpose): it is sanity-checked against the reference
> estimator converging "within hundreds of samples under varied inputs" — weeks of varied
> weather rather than hours of samples — which supports "weeks" and does not derive "30". A
> site that finds less sufficient is evidence to move it, and moving it costs a
> configuration default. Not stated in the thread.

### Requirement: Model quality is two numbers, not one

Every learner SHALL publish two quality figures as items — a normalized confidence in
[0,1] from its own parameter uncertainty, and a rolling prediction error in the model's
own unit — so "converged" and "drifting" are separately observable.

#### Scenario: Converged

- **GIVEN** a learner whose confidence has risen to its configured threshold
- **WHEN** the engine asks whether the model may be used
- **THEN** the answer is yes, on the confidence figure alone

#### Scenario: Converged but drifting

- **GIVEN** a converged model whose rolling prediction error has risen above a multiple of
  its converged value, a sensor having been moved
- **WHEN** the engine asks whether the model may be used
- **THEN** the answer is no, and the state is distinguishable from "not yet converged"
  because the two figures are separate

> Source: `tasks.md` 2.1 asked what "converged" means uniformly; decided by the owner
> 2026-08-02 (D17, Part B LL-3, docs/OWNER_DECISIONS.md) — the reference estimator already
> carries both a shrinking parameter covariance and a tracked residual, and this change's
> two scenarios need one each; the single-quality-number alternative is preserved in
> design.md §3, together with why it cannot express "converged but drifting". Not stated in
> the thread.

### Requirement: Propose, don't override

Learned values SHALL be applied as reviewable proposals or clearly-flagged overrides —
auto-applying only where the parameter is user-configurable and the participant declares
it learnable, never silently replacing user configuration, and never auto-applying to a
safety-relevant limit.

#### Scenario: Model drifts

- **GIVEN** a previously converged model
- **WHEN** its rolling prediction error degrades (sensor moved, renovation)
- **THEN** the system falls back to the user-configured values and surfaces the change,
  rather than acting on a bad model

#### Scenario: A learnable setting applies, a protection does not

- **GIVEN** a learned pre-heat lead time on a participant that declares it learnable, and a
  learned minimum-off time that is a device protection
- **WHEN** both are produced
- **THEN** the lead time is applied and flagged, and the protection is offered as a
  proposal the user accepts or rejects

> Source: the same guardrail as `participant-discovery` _Propose, never silently activate_,
> which is where it is stated for discovery; **re-sourced there** from its previous
> "synthesized, needs explicit sign-off" flag by the owner 2026-08-02 (D17, Part B LL-1,
> docs/OWNER_DECISIONS.md), on the grounds that the same rule appears independently three
> times — discovery's requirement, the reference implementation's shadow-mode default and
> the wave-1 prototype's four enforcements; thread grounding remains the shadow-first
> validation culture
> ([5014707411](https://github.com/openhab/openhab-core/issues/3478#issuecomment-5014707411)).
> The narrowed auto-apply clause is the owner's; alternatives preserved in design.md §1.

### Requirement: On-premise learning

All learning SHALL run on locally persisted data without requiring cloud services.

#### Scenario: Offline site

- **WHEN** the installation has no internet beyond its forecast sources
- **THEN** thermal-model and profile learning still function on local persistence

> Source: openHAB's local-first principle; all named prior art (storm.house,
> spot-price-optimizer, reference) learns/operates on local data.
