---
type: Design Review Index
title: "Review Notes — Index"
description: "Index of the adversarial design-review dialogue (architect and engineer) behind the plan."
tags: [design, esi, api-explorer, openapi, review-notes, process]
timestamp: 2026-07-18T19:00:46Z
---

# Review Notes

This directory captures the **design-review dialogue** behind the plan in [../](../README.md). It exists so the
reasoning is followable, not just the conclusions. The main docs (`01`–`12`) hold the *resolved*
design; these notes hold the *argument* that got them there.

## Protocol

Reviews alternate between two roles, each challenging the last:

1. **Lead architect** — boundaries, contracts, cross-cutting quality, long-term maintainability,
   open-source viability.
2. **Lead engineer** — implementability, concrete APIs, dependency choices, effort, footguns.

Each round is a file. Each finding has:

- an **ID** (`A#` architect, `E#` engineer; second-round suffixes like `A2.#`),
- a **severity** (blocker / high / medium / low),
- a **recommendation**, and
- a **status**, updated by later rounds: `open` → `conceded` / `rebutted` / `refined` / `deferred` /
  `resolved`.

A later reviewer responds to earlier findings by ID (concede / rebut / refine) before adding their
own. When the dialogue converges, the resolved change is written into the main docs and the finding is
marked `resolved` with a pointer to where it landed.

## Rounds

| Round | Role | File |
| --- | --- | --- |
| 1 | Architect | [01-architect-round1.md](./01-architect-round1.md) |
| 1 | Engineer | [02-engineer-round1.md](./02-engineer-round1.md) |
| 2 | Architect | [03-architect-round2.md](./03-architect-round2.md) |

## Outcome

The resolved decisions from these rounds were applied to the main plan, notably the new documents:

- [09-quality-and-resilience.md](../09-quality-and-resilience.md) — accessibility, performance,
  error handling, security.
- [10-testing-and-tooling.md](../10-testing-and-tooling.md) — test strategy, spec corpus, build, CI.
- [11-open-source-and-api-stability.md](../11-open-source-and-api-stability.md) — public API surface,
  semver, the CSS/DOM override contract, governance placeholders.
- [12-repository-and-module-layout.md](../12-repository-and-module-layout.md) — concrete monorepo and
  per-package file layout, dependency choices.

Plus targeted edits to `03`, `04`, `06`, `07` (see the individual rounds for what changed and why).

## Owner decisions

Tracked in [11-open-source-and-api-stability.md](../11-open-source-and-api-stability.md#decisions-needed):

1. **Package scope / names — DECIDED.** Published under **`@eve-online-tools`**
   (`@eve-online-tools/openapi-model`, `@eve-online-tools/openapi-explorer`), moving to **`@ccpgames`**
   if CCP publishes under the company name. Package names stay generic.
2. **Governance / ownership — open.** Which team owns the repo, its release cadence, and its issue
   triage once open-sourced.
3. **License — open.** MIT is the default expectation; confirm before publish.
