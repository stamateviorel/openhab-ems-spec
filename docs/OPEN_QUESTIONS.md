# Open questions across the corpus

Generated 2026-09-27 from the state markers in each change's `design.md`.
Every section in this corpus declares one of **Answered**, **Open**, **Narrowed** or
**Context**; this page collects the ones that are not closed.

It exists because the questions that need someone else were spread across twelve files, where
a provenance note and an unanswered architectural question looked the same.

**18 sections are open or only narrowed.** Three of them the corpus cannot answer for
itself.

## These need a core maintainer

Nothing downstream of them can be settled here, and answering them is cheap for whoever has the
authority: each is one decision, and every alternative is preserved in the section.

| Change | Question | State |
|---|---|---|
| [`define-extension-points`](../openspec/changes/define-extension-points/design.md) | 2. Naming | OPEN — and it is Kai's own question, unanswered since 2023 |
| [`define-participant-model`](../openspec/changes/define-participant-model/design.md) | 2. Core vs. add-on boundary | OPEN — and only a core maintainer can close it |
| [`define-participant-model`](../openspec/changes/define-participant-model/design.md) | 3. Engine simplicity | OPEN — and the corpus has grown away from it |

## Everything else still open

| Change | Section | State |
|---|---|---|
| [`define-energy-levels`](../openspec/changes/define-energy-levels/design.md) | 1. Level names — the spec and the taxonomy it cites disagree (A11 / L11) | STILL OPEN |
| [`define-energy-levels`](../openspec/changes/define-energy-levels/design.md) | 5. Derivation: what the prototype proved task 2.1 is actually about (L12) | STILL OPEN |
| [`define-energy-levels`](../openspec/changes/define-energy-levels/design.md) | 6. Tie-break, band precedence and counts that do not fit (L3, L5, L6) | STILL OPEN |
| [`define-energy-levels`](../openspec/changes/define-energy-levels/design.md) | 7. Hours or slots (L4) | NARROWED, not answered |
| [`define-energy-ui`](../openspec/changes/define-energy-ui/design.md) | 4. Relation to existing community widgets | OPEN |
| [`define-energy-ui`](../openspec/changes/define-energy-ui/design.md) | 5. Surfaces the wave-1 prototype found unnamed | OPEN |
| [`define-extension-points`](../openspec/changes/define-extension-points/design.md) | 3. Are actuation adapters an extension point? | OPEN |
| [`define-forecast-providers`](../openspec/changes/define-forecast-providers/design.md) | 1. Overwriting past entries (feasibility dependency) | OPEN — dispositioned, not decided |
| [`define-forecast-providers`](../openspec/changes/define-forecast-providers/design.md) | 2. Writer precedence on a layered series | OPEN — dispositioned, not decided |
| [`define-optimization-objectives`](../openspec/changes/define-optimization-objectives/design.md) | 1. Do energy levels follow the objective? | STILL OPEN |
| [`define-optimization-objectives`](../openspec/changes/define-optimization-objectives/design.md) | 2. Composite objectives | OPEN — deliberately deferred past v1 |
| [`define-optimization-objectives`](../openspec/changes/define-optimization-objectives/design.md) | 3. Naming | OPEN |
| [`define-price-providers`](../openspec/changes/define-price-providers/design.md) | 5. Price keys that arrive before the price plane (B3) | STILL OPEN |
| [`discover-participants-from-model`](../openspec/changes/discover-participants-from-model/design.md) | 3. Relationship to the model's own gaps | OPEN |
| [`discover-participants-from-model`](../openspec/changes/discover-participants-from-model/design.md) | 5. What the wave-1 prototype surfaced | OPEN |

