# openHAB review rules — the governing reference

Anything built from this corpus is reviewed by openHAB maintainers against the rules
below. They are **not our rules**; they are reproduced here (with sources) so an
implementer inherits them instead of discovering them in review.

Sources, in order of authority:

1. [Rules for openHAB maintainers](https://github.com/openhab/openhab-distro/discussions/1505)
   — org-wide, @kaikreuzer
1. [Specific rules for openHAB Add-ons maintainers](https://github.com/openhab/openhab-addons/discussions/14694)
   — @kaikreuzer
1. [Review Checklist](https://github.com/openhab/openhab-addons/wiki/Review-Checklist)
   — the concrete per-PR checklist
1. [Coding Guidelines](https://www.openhab.org/docs/developer/guidelines.html)
   — already injected via [`../openspec/config.yaml`](../openspec/config.yaml)

## What reviewers actually weight (read this first)

From the add-ons maintainer rules, in the maintainers' own priority order:

1. **Consistency towards the user** — coherent modelling, proper documentation.
1. **Architecture consistency** — correct use of openHAB APIs, using provided features
   (shared HTTP client, system channel types, existing registries) rather than
   re-inventing them.
1. **Consistent runtime behaviour** — correct status handling, clean shutdown, proper
   logging.
1. **…and explicitly _not so much_ individual code style.**

That ordering matters more than any single item below: a change that is stylistically
perfect but models things inconsistently or re-invents a core API will be rejected, while
style nits are the least of a reviewer's concerns. Design for the first three.

## The checklist (all 44 items, verbatim in substance)

Marked for relevance, because this corpus targets **openhab-core**, which has no Things,
channels or bindings of its own: **[C]** applies to core code · **[B]** binding-specific,
applies only if/when an add-on is built from this corpus · **[C/B]** both.

### Packaging, licensing, docs

1. **[C/B]** Correct bundle name in `pom.xml` (e.g. `openHAB Core :: Bundles :: …`).
1. **[C/B]** New bundles registered in the main `pom.xml` **and** the Karaf feature
   (`src/main/feature`).
1. **[C/B]** EPLv2 license verified, `NOTICE` file present.
1. **[C/B]** All dependencies listed in the `NOTICE` file.
1. **[B]** README based on the official template, consistent sections.
1. **[B]** README formatting: a new line after every sentence.
1. **[B]** README: capitalise section headers (e.g. "Thing Configuration").
1. **[B]** README documents every thing type id, channel id and configuration key.
1. **[C/B]** i18n: provide the original language only; translations go through Crowdin.
1. **[C/B]** Correct copyright year — `mvn -lp :<artifactId> license:format`.

### Modelling and user-facing consistency

1. **[B]** Thing/Channel labels under 25 characters, 2–3 words, capitalised.
1. **[B]** Semantic tags added to Things and Channels where applicable.
1. **[B]** Specify units for Thing config parameters (e.g. `unit="s"`).
1. **[B]** Add min/max values for Thing config parameters.
1. **[B]** Add `context` tags to config parameters (e.g. `network-address`).
1. **[B]** Use Units of Measure in channel declarations (e.g. `Number:Temperature`).
1. **[B]** Specify the representation property in discovery results.

### Runtime behaviour

1. **[B]** `Handler.initialize()` returns quickly with a valid Thing status.
1. **[B]** Handle `REFRESH` commands.
1. **[B]** Clean up asynchronous futures from `initialize()` in `dispose()`.
1. **[C/B]** Threads set as daemon (`Thread.setDaemon(true)`) — where a thread is
   unavoidable at all; core guidelines say use the shared schedulers.
1. **[C/B]** Threads named (`Thread.setName()` or constructor).
1. **[C/B]** Examine `synchronized` methods carefully to prevent deadlocks.
1. **[C/B]** No need to check whether a `Future` is already cancelled before cancelling.

### Code

1. **[C/B]** `@NonNullByDefault` on all classes except DTOs.
1. **[C/B]** Refactor duplicate code where possible.
1. **[C/B]** camelCase for non-static fields — no underscores or prefixes.
1. **[C/B]** Prefer primitives over wrappers (`int` over `Integer`).
1. **[C/B]** Every `byte[]`↔`String` conversion specifies a `Charset`; same for streams.
1. **[C/B]** Sockets and streams use try-with-resources.
1. **[C/B]** Use lambdas for `Runnable`s.
1. **[C/B]** Cache the results of `getConfigAs()` / `getConfiguration()`.
1. **[C/B]** Annotate unavoidable compiler warnings with `@SuppressWarnings`.

### Exceptions and logging

1. **[C/B]** Custom exceptions extend `Exception` (checked).
1. **[C/B]** Do not throw `RuntimeException` for expected errors.
1. **[C/B]** Do not catch generic `Exception`; catch `RuntimeException` for unexpected
   errors.
1. **[C/B]** Special catch for `InterruptedIOException` alongside `IOException`.
1. **[C/B]** Return immediately when catching `InterruptedException` /
   `InterruptedIOException`.
1. **[C/B]** Conservative log levels — primarily `debug`; `error` for bugs or
   misconfiguration.
1. **[C/B]** Log stack traces only for severe errors that indicate a bug.
1. **[B]** Do not log when Things go offline — the framework logs the `updateStatus()`
   text.

### Verification before submitting

1. **[C/B]** Review checkstyle warnings in `target/code-analysis/report.html`.
1. **[C/B]** `mvn spotless:check -Dspotless.check.skip=false`.
1. **[C/B]** Validate JavaDoc with `mvn javadoc:javadoc`.

## Merge-time rules (org-wide)

Relevant to how a contribution eventually lands, and to how we should behave if we ever
hold a maintainer role:

- Maintainers do not merge their own PRs without another maintainer's approval.
- **squash+merge** is the default; never "create merge commit"; rebase+merge only when
  distinct commits genuinely belong in history.
- When merging multiple commits, clean up the message (no stacked sign-offs).
- PR builds should pass before merging; if ignored, say why in a comment.
- **Sign-off is mandatory**, except the small-patch exception (typos, single-line
  doc/comment changes).
- Merges set the milestone, and decide release-note visibility via the `bug` /
  `enhancement` label with an imperative, descriptive title.

## Known link rot

The add-ons maintainer discussion's first two links were reported broken in 2024
([comment](https://github.com/openhab/openhab-addons/discussions/14694#discussioncomment-8478234));
the working URLs are the ones listed at the top of this page.
