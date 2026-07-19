# Add named profiles per device

**Wave 3** — one device, several ways to run. The July 2026 convergence, but the idea
is as old as the thread's second comment.

## Why

Real devices don't have one behavior: a washing machine has a full-size and a low-temp
program; a heat pump runs one DHW cycle a day in summer but all day in winter; PV
production curves differ by season. Requirement named by @mstormi (2026-07-19),
immediately seconded by @masipila from his own installation — and it closes the loop to
@jlaur's founding comment, which already wanted to "select the last 60 °C cottons
program" with its own mapped curve. The wave-1 model carries exactly one profile per
consumer; this change makes profiles named, plural, and selectable — with the curve
(TimeSeries) and its scale factor part of each profile.

## What changes

- Define the `device-profiles` capability: multiple named profiles per participant,
  profile contents (curve × scale + constraints), and active-profile selection
  (manual, scheduled/seasonal, or detected).

## Non-goals

- Learning profiles from history (wave 4 — this change only defines the container it
  would write into).
- Reworking the four classes — profiles parameterize a class, they don't replace it.

## Impact

- Extends the participant model. This is the change that stress-tests the wave-1
  declaration-mechanism decision: named profiles with TimeSeries payloads strain flat
  item metadata (see wave-1 design.md §1).
