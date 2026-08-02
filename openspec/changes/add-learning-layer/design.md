# Design notes — open questions (created and answered 2026-08-02)

This change had no design file at all — the defect the decision pass filed as LL-0, the
same class as the one already fixed for `define-energy-levels`. Its four open questions
lived only in `tasks.md` and in one requirement's `Source:` line, where a reader meeting
the requirements first would never find them. They are collected here, each with what was
decided.

> **Decision pass, 2026-08-02.** The owner answered all four (D17, Part B) and left one
> figure explicitly undecided (Part D). The record is
> [`docs/OWNER_DECISIONS.md`](../../../docs/OWNER_DECISIONS.md). These are **owner
> decisions in a reference implementation, not thread consensus**, and one of them —
> §2's retention figure — is **judgement, not derived**, which the requirement says out
> loud. Nothing was deleted; every alternative each question carried is preserved.

## 1. Is "propose, don't override" the corpus's own guardrail or a synthesis? (LL-1)

_Propose, don't override_ was written as a synthesis of the thread's shadow-first
validation culture rather than as anything a participant stated verbatim, and its own
`Source:` line flags it "for explicit review as the only requirement here not stated
verbatim by a thread participant". That flag is the question: is the guardrail load-bearing
enough to keep, and if it is, what exactly may a learned value do on its own?

**Answered — D17** (Part B, LL-1). _Decision:_ **keep it, and re-source it** from
`participant-discovery` _Propose, never silently activate_ rather than leaving it flagged as
a synthesis — the same guardrail appears independently three times (discovery's requirement,
the reference implementation's shadow-mode default, the wave-1 prototype's four
enforcements), which is what makes it corpus behaviour rather than invention. _And narrow
the override clause:_ a learned value may **auto-apply only where the parameter is
user-configurable AND the participant declares it learnable**; **safety-relevant limits
never auto-apply**. _Alternatives preserved:_ dropping the guardrail as unsourced (rejected
— it is the one thing standing between a learner and a user's configuration) and letting
any learned value auto-apply once converged (rejected — it makes a drifting model a silent
actuator).

## 2. What data does a learner need before it may run? (LL-2)

`tasks.md` 1.2 asked which persistence strategies a user must enable. Nothing in the
requirements says what a learner needs, so a site with the wrong persistence configuration
gets a learner that runs and produces confident nonsense.

**Answered — D17** (Part B, LL-2), _with its central number left as judgement (Part D)._
_Decision:_ a **per-learner contract** — each learner declares the items it reads and the
minimum retained history it needs — **enforced by refusing to run a learner whose inputs
are not persisted**, and reported on the engine's observability surface (D16 pack A8). For
the thermal model: indoor temperature, outdoor temperature and heating power at
`everyChange` + `everyMinute`, over **at least 30 days**.

**⚠ The 30-day figure is judgement, not derived, and the requirement says so.** It is
sanity-checked against the reference estimator's "converges within hundreds of samples under
varied inputs" — at 1-minute sampling that is _hours_ of samples but _weeks_ of varied
outdoor conditions, so the binding constraint is the weather, not the sample count. That
reasoning supports "weeks"; it does not derive "30". A site that finds 14 days sufficient,
or 60 insufficient, is evidence this should move, and moving it costs a configuration
default. _Alternatives preserved:_ a site-wide minimum instead of a per-learner contract
(simpler, but wrong for any learner with different inputs) and running anyway on whatever
history exists, reporting low confidence instead of refusing (defensible, and rejected here
because a confident-looking number from three days of data is worse than no number).

## 3. What does "converged" mean, and what does "drifted" mean? (LL-3)

Two requirements need it and neither defines it: _Learned thermal model_ says the model
"expose its quality (error metric)" and its scenario turns on "converged within its error
bound", while _Propose, don't override_'s scenario turns on the error metric **degrading**.
One number cannot serve both readings.

**Answered — D17** (Part B, LL-3). _Decision:_ **two numbers from every learner, not one** —
a normalized **confidence** in [0,1] derived from the estimator's own parameter uncertainty
("converged" = at or above a threshold), and a rolling **prediction error** in the model's
own unit ("drift" = above a multiple of its converged value). **Both published as items.**
They are genuinely different quantities: a parameter covariance that shrinks as the fit
settles, and a tracked residual that grows when the world changes underneath a settled fit.
The reference estimator already carries both. _Alternative preserved:_ one combined quality
number, which is what the requirements implied and which cannot express "converged but
drifting" — the exact state a moved sensor or a renovation produces.

## 4. When does learning run? (LL-4)

`tasks.md` 2.2 asked: per run, nightly, or on demand? The two halves of this change have
different shapes — a load profile is only meaningful over a completed run, while a thermal
model is a continuous estimation problem.

**Answered — D17** (Part B, LL-4). _Decision:_ **per completed run for load profiles;
continuous/online for the thermal model; on-demand re-learn for both. Explicitly not
nightly.** A partial run is not a curve, and a nightly batch throws away the recursive form
the online estimator is built on. _Alternative preserved:_ a nightly batch for both, which
is the obvious operational shape and is rejected on those two grounds rather than on taste.
