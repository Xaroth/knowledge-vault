---
type: Wayfinder Map
title: "API Explorer — Sequenced Build Plan (Map)"
description: "Wayfinder map charting the route from the finished API Explorer design to a sequenced, sized, de-risked implementation plan."
tags: [design, esi, api-explorer, openapi, wayfinder, build-plan]
timestamp: 2026-07-20T00:00:00Z
---

# API Explorer — Sequenced Build Plan

<!-- wayfinder:map -->

## Destination

A **sequenced, sized implementation plan** that decomposes the finished
[API Explorer design](../README.md) (docs `01`–`12`) into buildable, dependency-ordered,
de-risked tickets — a spec a team can pick up and build from. The outcome of this effort
is the *plan*, not the code.

## Notes

- **Domain.** A home-grown OpenAPI 3.0/3.1 explorer replacing `@stoplight/elements`: two
  packages — `openapi-model` (parser, pure TS) and `openapi-explorer` (renderer, React) —
  plus a private ESI adapter. Full design intent in [../](../README.md) (docs `01`–`12`);
  the adversarial design review is in [../notes/](../notes/README.md).
- **Starting point.** The design is at Definition of Ready — the round-2 architect declared
  it "ready to be broken into implementation tickets" and no review finding remains open.
  This effort turns that readiness into an actual plan.
- **Owner decisions ride along.** Governance/ownership and license
  ([../11](../11-open-source-and-api-stability.md#decisions-needed)) block the first public
  *publish*, not the build. They are tracked in **Not yet specified**, not as the focus.
- **Skills to consult each session.** `/grilling` + `/domain-modeling` for decision tickets;
  `/research` for research tickets. Writes to this vault go through **speki**
  (`speki begin` → `speki submit`); every file is OKF (frontmatter required), so tickets
  carry their `Type`/`Status`/`Blocked by` metadata in the body, not as bare files.
- **Deliverable override.** This map's destination is a deliverable spec, so *assembling the
  plan document* is in scope as the final step — an intentional override of wayfinder's
  plan-don't-do default (see Not yet specified).
- **POC is a reference, not proven code.** The ESI Explorer POC is an extremely rough draft;
  every assertion it makes (the snapshot technique, the "tested" serializer, the look-and-feel)
  is **validated before entering the product**. A DoD obligation on every unit that reuses POC
  work.

## Decisions so far

<!-- one line per closed ticket: gist + link -->

- [Does the ESI spec mark internal operations?](./issues/06-esi-x-internal-check.md) — **No**
  (`x-internal`/`x-hidden`/`deprecated` all absent); the `hideInternal` operation-filtering
  feature stays out of scope. (Side-note: live spec uses `x-required-roles`, design says
  `x-required-scope` — reconcile when planning the adapter's extension registry.)
- [Decomposition strategy & ticket granularity](./issues/01-decomposition-strategy.md) —
  units follow the design's **module/component** structure, ~**1 PR** each with
  **digestibility** as the governing rule; oversized units split **by behavior/concern**;
  **walking-skeleton** unit per seam early; **spikes are separate units** from build units;
  the **ESI adapter splits** into sub-units; the **playground is decoupled** to a
  demand-driven stakeholder-update track (CI rides on tests + fixtures).
- [Build order & dependency backbone](./issues/02-build-order-and-dependencies.md) — a neutral
  **dependency DAG**; front bottleneck **3.1 spike → draft `NormalizedSpec` → walking skeleton
  → freeze**, then **parser ‖ renderer breadth** against the frozen contract; **adapter opens
  on the seam**, parallel to renderer breadth; **everything built in isolation**, then
  **A/B-flagged side-by-side → parity gate → flip → soak → tear down old**. **Island-safety is
  a day-one invariant**; **Astro host-wrapper integration ruled out of scope**.
- [Spike inventory & exit gates](./issues/03-spike-inventory-and-gates.md) — **exactly one
  spike**: the bounded, time-boxed **schema-normalization spike** (3.1 convergence +
  snapshot-preservation + `allOf`-merge shape → locks the `NormalizedSpec` shape), with a
  split-valve for `allOf`. Everything else is a **build unit with a validation DoD** (Try It
  serializer, full `allOf`) or a **gate** (island mount; perf budget → ticket 04). Surfaced a
  new prototype ticket on ESI-scale look-and-feel.

## Not yet specified

- **Assemble the build-plan document** from the accumulated decisions. Graduates to a ticket
  once decomposition, build-order, spikes, quality-weave, sizing, **and the ESI-scale
  look-and-feel prototype** are decided — its shape depends on those answers.
- **Publish-gating owner decisions** — governance/ownership and license
  ([../11](../11-open-source-and-api-stability.md#decisions-needed)). Needed before the first
  public release; ride along as blockers to the *publish* step, not the build. Graduate only
  if this effort's scope is extended to the publish milestone.
- **SDE-pages repoint** off `ElementsPanel`/`Stack`/`Wrapper`
  ([../08](../08-esi-integration-and-migration.md)) — the design flags this as a non-blocking
  follow-up decoupled from the ESI cutover. Slots into the plan's cleanup phase once the
  build-order backbone lands.

## Out of scope

- **Writing the implementation code** — this effort produces the plan; building is the
  downstream work the plan enables.
- **The first public open-source release** — the publish/launch act and its governance +
  license sign-off. A separate launch effort; this plan gets the packages build- and
  publish-*ready*, not published.
- **Astro host-wrapper integration** — mounting the component as an Astro island in the real
  app ([../08 Phase 4](../08-esi-integration-and-migration.md#84)). The Astro rewrite lands
  *after* the API Explorer rework, so it is a separate later effort; this plan keeps the
  island-safe self-containment invariant that makes it cheap. Ruled out by
  [Build order & dependency backbone](./issues/02-build-order-and-dependencies.md).
