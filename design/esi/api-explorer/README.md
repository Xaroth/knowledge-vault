---
type: Design Overview
title: "OpenAPI Explorer — Design Documentation"
description: "Index and executive summary for the home-grown OpenAPI Explorer that replaces @stoplight/elements."
tags: [design, esi, api-explorer, openapi, overview]
timestamp: 2026-07-18T19:00:46Z
---

# OpenAPI Explorer — Design Documentation

This directory describes a **home-grown, self-contained React component for rendering OpenAPI
3.0 / 3.1 documents**, intended to replace our current dependency on
[`@stoplight/elements`](https://github.com/stoplightio/elements).

The documents are written in a **finished-state** style: they describe the target product as if
it already exists, so they can be used as the source material for drafting development plans,
estimates, and tickets. They are *design intent*, not an implementation log.

## Why this exists

We render the ESI OpenAPI spec through `@stoplight/elements`. That integration works, but it has
accumulated significant friction — three `patch-package` patches, a ~350-line CSS override sheet
that fights Stoplight's internal class names, authentication injected by writing into a DOM input,
a hard dependency on `react-router`, and a component that cannot server-side render. On top of that,
a likely move to **Astro** (with its Islands architecture) breaks the way we currently share React
context across the page.

Rather than keep patching a black box, we build a component that we own, tailored to our needs, with
a clean extension surface. A working proof of concept — the
ESI Explorer — already validates the core of this approach and is
referenced throughout.

## The shape of the solution

Two packages, cleanly separated (mirroring the ESI Explorer POC):

| Package | Responsibility | Framework |
| --- | --- | --- |
| **`@eve-online-tools/openapi-model`** *(parser)* | Load, dereference, validate and **normalize** an OpenAPI 3.0/3.1 document into a stable, UI-friendly model. | None (pure TS) |
| **`@eve-online-tools/openapi-explorer`** *(renderer)* | A single self-contained React component, `<ApiExplorer>`, that renders the normalized model. Fully overridable, self-styled, island-safe. | React 18/19 |

ESI-specific behaviour (EVE SSO auth, `x-*` vendor extensions, compatibility dates, spec
pre-processing) lives **outside** both packages, in a thin adapter in this app, wired in through the
component's documented extension points. The renderer itself knows nothing about ESI.

> Package names are provisional. See [02-architecture.md](./02-architecture.md#naming).

## Reading order

1. **[01-motivation-and-goals.md](./01-motivation-and-goals.md)** — Why we are replacing Elements, the
   concrete pain points (with evidence), and the goals / non-goals that bound the work.
2. **[02-architecture.md](./02-architecture.md)** — The two-package split, data flow, the provider
   stack, and how the design satisfies the Astro Islands / self-containment constraint.
3. **[03-parser-package.md](./03-parser-package.md)** — `@eve-online-tools/openapi-model`: pipeline, the normalized
   data model, OpenAPI 3.0 vs 3.1 handling, `$ref`/extension recovery, and the load policy.
4. **[04-renderer-component.md](./04-renderer-component.md)** — `@eve-online-tools/openapi-explorer`: the public
   `<ApiExplorer>` API, the component-slot registry, the vendor-extension registry, navigation and
   deep linking.
5. **[05-feature-set.md](./05-feature-set.md)** — Feature parity matrix against Elements and the POC,
   and the gaps we deliberately close or defer.
6. **[06-authentication-and-try-it.md](./06-authentication-and-try-it.md)** — The Try It request
   builder and the first-class authentication hook that replaces today's DOM injection.
7. **[07-styling-and-theming.md](./07-styling-and-theming.md)** — Self-contained styling, the CSS
   custom-property token contract, and the override model for host integration.
8. **[08-esi-integration-and-migration.md](./08-esi-integration-and-migration.md)** — How this app
   wires ESI specifics in, and the migration path off `@stoplight/elements`.
9. **[09-quality-and-resilience.md](./09-quality-and-resilience.md)** — Accessibility, performance,
   error handling/diagnostics, and security — the cross-cutting requirements, with acceptance criteria.
10. **[10-testing-and-tooling.md](./10-testing-and-tooling.md)** — Test strategy, the real-world spec
    corpus, build tooling, CI gates, and the per-component definition of done.
11. **[11-open-source-and-api-stability.md](./11-open-source-and-api-stability.md)** — Public API
    surface, the CSS/DOM override contract, semver policy, and governance placeholders.
12. **[12-repository-and-module-layout.md](./12-repository-and-module-layout.md)** — Concrete monorepo
    and per-package file layout, plus the frozen dependency choices.

### Design-review dialogue

The plan was reviewed adversarially (lead architect → lead engineer → lead architect). The reasoning
— every criticism, rebuttal, and concession — is recorded in **[notes/](./notes/README.md)**, and the resolved
decisions are folded into docs `03`–`12`. Start at [notes/README.md](./notes/README.md) to follow the
trains of thought, including the two [open decisions](./11-open-source-and-api-stability.md#decisions-needed)
(package naming and governance) that need a human owner.

## Guiding principles

- **SOLID** — the renderer depends on the *normalized model* abstraction, never on raw OpenAPI
  parsing or on ESI. ESI specifics are injected, not baked in. Every visual part is a swappable slot.
- **YAGNI** — we build to reach or modestly exceed today's feature set and the POC. Speculative
  capabilities are listed explicitly as *deferred*, not implemented ahead of need.
- **Simple over complex, complex over complicated** — a hand-rolled, well-scoped component with a
  small, legible context surface is preferred over a heavyweight framework we must fight.
- **Self-containment** — the component owns its styling, its state, and its context. Everything it
  needs from the host arrives through props. Nothing leaks in or out.
