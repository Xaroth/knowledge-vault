---
type: Design Review
title: "Review Round 2 — Lead Architect"
description: "Lead architect's second round: rulings on the engineer's responses and the Definition of Ready."
tags: [design, esi, api-explorer, openapi, review-notes, architect, definition-of-ready]
timestamp: 2026-07-18T19:00:46Z
---

# Round 2 — Lead Architect Review

**Date:** 2026-07-18 · **Responding to:** [02-engineer-round1.md](./02-engineer-round1.md).

The engineer's round closes the substantive gaps. I accept both rebuttals, adjust one model decision,
add three items the exchange surfaced, and then define the "ready for implementation" bar so we know
when to stop reviewing and start building.

---

## Rulings on the engineer's responses

### A6 (virtualization) — **rebuttal accepted.** {#a6-final}
Agreed: no built-in virtualization in v1. The combination of *render-one-view* + *the `NavigationTree`
slot as an escape hatch* + *a stated perf budget* is the right YAGNI call. I want one guardrail: the
perf budget in `09` must include a concrete number to test against (e.g. interaction-to-render on the
ESI spec on mid-range hardware), so "measure first" has a threshold, not a vibe.

### A13 (function-valued labels) — **rebuttal accepted.** {#a13-final}
Function-valued dynamic labels are simpler and more type-safe than a template engine. Good. One
constraint: the `defaultLabels` object is itself part of the **public API** (people will spread and
tweak it), so it lives in a stable, documented export, and adding a required label key is a
**breaking change**. Note that in `11`.

### A2 (`allOf` merge-and-preserve) — **accept with a refinement.** {#a2-final}
Merging for display is right. Refinement: **do not silently last-wins on conflicts.** When two `allOf`
members disagree on a property's type/constraints, keep the first, drop the conflicting one from the
merged view, **and emit an `error`-level diagnostic** (not just `warn`) — a conflicting `allOf` is
usually a spec bug the author wants surfaced, and swallowing it makes us the thing that hid a real
problem. Otherwise concur.

### A11 (`fetcher`) / A3 (tiers) / A4 (DOM contract) / A7 (diagnostics) — **all accepted as written.**
These are the strongest resolutions in the set. The single `fetcher` seam, the two integration tiers,
the frozen `data-oae-*` contract, and the `diagnostics` channel are exactly the contracts a
maintainable OSS library needs. No further comment.

---

## New findings from this round

### A2.1 — The model needs a machine-readable version, not just semver on the package · severity: medium · status: resolved → [03](../03-parser-package.md#versions) / [11](../11-open-source-and-api-stability.md)
Because a host can pass a **pre-built `NormalizedSpec`** (the Dependency Inversion seam), the renderer
can receive a model built by a *different version* of the parser than it expects. Add a
`modelVersion: number` field to `NormalizedSpec` and have the renderer check it, rendering a clear
error on mismatch rather than crashing on a missing field. Cheap insurance for the one place versions
can silently diverge.

### A2.2 — Reaffirm the YAGNI boundary explicitly, in the plan · severity: low · status: resolved → [05](../05-feature-set.md#54-deferred-yagni--listed-not-built)
The engineer's "no plugin framework" instinct is correct and should be *written down*, not just
agreed in notes, so it survives us. The extension model is "swap a React component"; there is no
plugin lifecycle, no runtime discovery, no config DSL. Add this as an explicit anti-goal so a
well-meaning future contributor doesn't "helpfully" build a plugin system.

### A2.3 — Confirm YAML stays in scope, cheaply · severity: low · status: resolved → [03](../03-parser-package.md#input-formats)
The engineer didn't take my bait to cut YAML, and they're right not to: swagger-parser handles it for
free, and an OSS OpenAPI tool that rejects YAML would be baffling to users (most specs are YAML). Keep
it. This is the rare case where *not* cutting is the YAGNI-consistent choice, because the feature costs
us nothing and its absence would cost every user.

---

## Definition of Ready (the bar for "medior engineers can build this")

The plan is implementation-ready when all of the following are true. This is the exit criterion for
the design phase:

1. **Contracts are frozen on paper:**
   - `NormalizedSpec` (incl. cycles, `allOf`, discriminator, diagnostics, `modelVersion`) — `03`.
   - `<ApiExplorer>` props, incl. `fetcher`, controlled/uncontrolled selection, integration tiers — `04`.
   - The CSS public API: `--oae-*` tokens + `data-oae-*` attributes — `07`/`11`.
   - The `AuthConfig` + `fetcher` auth flow — `06`.
2. **Cross-cutting quality is specified with acceptance criteria:** a11y patterns, perf budget with a
   number, error/diagnostics behaviour, XSS rules — `09`.
3. **Testing + tooling are concrete enough to scaffold CI:** spec corpus, test types, build tools,
   size budget, per-component checklist — `10`.
4. **OSS obligations are decided (except the two owner decisions):** public surface, semver policy,
   licensing placeholder, contribution basics — `11`.
5. **A concrete file layout exists** for both packages and the playground — `12`.
6. **The two owner decisions are surfaced, not buried:** package naming/scope and governance — tracked
   in `11` and `notes/README`.

With `09`–`12` added and the targeted edits to `03`/`04`/`06`/`07` applied, I consider items 1–6 met.
**I'm satisfied the design is ready to be broken into implementation tickets.** Remaining risk is
execution risk (chiefly 3.1 normalization and Try It edge cases), which the phased migration in `08`
already front-loads.

## What I explicitly did *not* ask for (guarding against over-design)

To keep us honest about YAGNI: I did **not** request a plugin system, a theming GUI, runtime spec
validation UI, multi-spec catalogues, an i18n framework, virtualization, or a mocking server. Each was
considered and consciously left out. If a future reviewer wants any of them, the burden is on them to
show a concrete need — the default is no.
