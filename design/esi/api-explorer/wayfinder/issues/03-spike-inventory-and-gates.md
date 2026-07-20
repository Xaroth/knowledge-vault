---
type: Wayfinder Ticket
title: "Spike inventory & exit gates"
description: "Decide which flagged hard parts become up-front spike/validation tickets, and each spike's exit criterion."
tags: [design, esi, api-explorer, wayfinder, build-plan]
timestamp: 2026-07-20T00:00:00Z
---

# Spike inventory & exit gates

- **Type:** grilling
- **Status:** resolved
- **Blocked by:** —

## Question

Which of the design's flagged hard parts become explicit up-front spike/validation tickets
before committing the main build, and what is each spike's exit criterion?

Candidates called out in the design and review:

- **OpenAPI 3.1 normalization** — the gating risk; POC does not handle 3.1 constructs
  ([../../03](../../03-parser-package.md#35), [../../08](../../08-esi-integration-and-migration.md#85)).
- **snapshot-before-dereference** — preserve `$ref` identity + vendor extensions
  ([../../03](../../03-parser-package.md#32)).
- **`allOf` merge-and-preserve** — flagged "most likely to be contentious"
  ([../../03](../../03-parser-package.md#310)).
- **Try It query serialization** (`form`/`space`/`pipe`/`deepObject`, explode)
  ([../../06](../../06-authentication-and-try-it.md)).
- **Island zero-context mount** — de-risk late host-context reliance
  ([../../08](../../08-esi-integration-and-migration.md#85)).
- **Concrete perf budget** — mount <~200 ms, nav <~50 ms on the ESI spec
  ([../../09](../../09-quality-and-resilience.md#92)).

## Answer

**The bar for a spike:** a candidate earns a spike only if it has **contract risk** (could
change the `NormalizedSpec` shape or a public API), **genuine approach-uncertainty** (we don't
know *how* we'd build it), or offers **cheap cross-cutting validation** of a risk lots of later
code would otherwise assume. "Fiddly but known / portable from the POC" is a build unit, not a
spike.

### Exactly one spike — the schema-normalization spike

Front-loaded per [ticket 02](./02-build-order-and-dependencies.md); its output feeds the
`NormalizedSpec` draft.

- **Scope:** 3.0↔3.1 convergence **+** snapshot-preservation (`$ref` identity + vendor
  extensions) **+** `allOf`-merge *shape* — all locking the one `NormalizedSpec` schema-node
  shape, exercised on the **live ESI spec + one 3.0 fixture**.
- **Bounded by deliverable, not coverage:** it proves the shape and enumerates the
  convergence/edge-case rules with resolutions; it does **not** implement production
  normalization — that is the build units' job.
- **Time-boxed, with a split valve:** if the box runs out with a sub-area still contested
  (most likely `allOf`), peel that area into its own spike rather than let the one spike
  sprawl.
- **Exit criterion:** the convergence table ([../../03 §3.5](../../03-parser-package.md#35))
  proven on live-ESI + a 3.0 fixture; every edge case enumerated with its resolution;
  cyclic/recursive schemas confirmed handled; the `NormalizedSpec` schema-node shape locked
  enough to seed the draft.

### Not spikes — build units with a validation DoD

- **Try It query serializer** — OpenAPI parameter serialization is spec-defined, so no
  approach-uncertainty; DoD validates `form`/`space`/`pipe`/`deepObject` + explode against
  **real ESI operations** (POC serializer treated as a reference to check, not trust).
- **Full `allOf` implementation** — approach already decided
  ([../../03 §3.10](../../03-parser-package.md#310)); DoD carries a **deep-`allOf`
  (Stripe-style) fixture** to validate first-wins + the `error` diagnostic against nasty input.

### Not spikes — gates/tests owned elsewhere

- **Island zero-context mount** — a gate on the walking skeleton and a CI invariant
  (island-safety is a day-one invariant per [ticket 02](./02-build-order-and-dependencies.md)),
  not an exploratory experiment.
- **Perf budget** (mount <~200 ms, nav <~50 ms) — the *number* + CI budget gate belong to
  [ticket 04](./04-quality-ci-oss-weave.md); *feasibility* is measured on the walking skeleton
  once it renders the real spec. Nothing to measure up-front, so no spike.

### Standing principle (recorded in the map's Notes)

The ESI Explorer POC is a **rough-draft reference, not proven code** — every assertion it makes
(the snapshot technique, the "tested" serializer, etc.) is **validated before entering the
product**. A DoD obligation on every unit that reuses POC work.

### Surfaced

The product's look-and-feel is **not** actually settled at ESI's scale (>200 routes, >200
schemas), where the POC's design showed issues — surfaced as
[Prototype: surviving ESI scale — final look-and-feel & Try It UX](./07-prototype-esi-scale-look-and-feel.md).
