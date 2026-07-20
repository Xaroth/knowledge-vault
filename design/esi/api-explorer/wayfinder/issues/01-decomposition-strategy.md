---
type: Wayfinder Ticket
title: "Decomposition strategy & ticket granularity"
description: "Decide what a buildable unit is and at what granularity the build plan cuts tickets."
tags: [design, esi, api-explorer, wayfinder, build-plan]
timestamp: 2026-07-20T00:00:00Z
---

# Decomposition strategy & ticket granularity

- **Type:** grilling
- **Status:** resolved
- **Blocked by:** —

## Question

What is a "buildable unit" in this plan, and at what granularity do we cut tickets?

Concretely: is the parser sliced per module (`resolve-input`, `original-spec-snapshot`,
`validate-spec`, `load-policy`, `normalize-schema`, `normalize-operations`, … per
[../../12](../../12-repository-and-module-layout.md#122)) or per capability? Is the renderer
sliced per slot component or per feature area? How is the ESI adapter cut (vendor renderers,
auth/fetcher wiring, labels, theme tokens)?

This decision sets the shape of every downstream ticket, so it is the spine of the plan and
the first thing to resolve.

## Answer

1. **Primary boundary — the design's own module/component structure.** One buildable unit per
   parser module ([../../12](../../12-repository-and-module-layout.md#122)) and per renderer
   slot/feature ([../../04](../../04-renderer-component.md)), with the per-component
   definition-of-done ([../../10](../../10-testing-and-tooling.md#103)) as the completion bar.

2. **Unit size — target ~1 PR.** Each unit is independently reviewable and mergeable.
   **Digestibility for both writer and reviewer is the governing rule**; a unit splits when a
   change stops being easy to digest.

3. **Splitting oversized units — by behavior/concern**, each sub-unit carrying its own
   fixtures and DoD. Canonical case: `normalize-schema` splits into {3.0↔3.1 convergence} +
   {`allOf` merge-and-preserve} + {cycle/recursion handling}.

4. **Walking skeletons — one thin vertical slice per seam, scheduled early on that seam:**
   - **parser→renderer** (`NormalizedSpec` contract): the real ESI spec through a minimal
     parser path into a minimal renderer that shows one operation — *before* the renderer's
     breadth is built, so the model contract is validated against a real consumer while it is
     still cheap to change ([../../03](../../03-parser-package.md#33)).
   - **renderer→ESI-adapter** (Tier-2 composed island): a minimal composed island proving
     slots/extensions/`fetcher`/auth cross the island boundary
     ([../../08](../../08-esi-integration-and-migration.md#84)), at the start of the adapter
     phase.

5. **Spikes are separate units from build units.** A spike is time-boxed and
   throwaway-permitted; its "done" is an *answer* (approach chosen, edge cases mapped,
   reference code linked), **not** production code held to the per-component DoD. The build
   ticket that implements the area is a distinct unit consuming that answer. This keeps
   exploratory code from being smuggled past the digestible-PR/DoD bar. (*Which* hard parts
   get a spike, and each spike's exit criterion, is
   [ticket 03](./03-spike-inventory-and-gates.md).)

6. **ESI adapter splits into digestible sub-units** — vendor-extension renderers /
   auth+`fetcher`+scope-request wiring / labels dict / theme tokens / flagged-page wiring —
   led by the renderer→adapter walking skeleton. The auth/fetcher/scope path is the fragile,
   security-relevant piece ([../../06](../../06-authentication-and-try-it.md)) and gets its
   own PR rather than riding along with theme tokens.

7. **The playground is out of the build's inner loop.** It is the demand-driven
   **stakeholder-update vehicle** — built/updated only when a stakeholder update is due
   (irregular cadence), **not** a DoD hook and **not** stood up speculatively. Day-to-day
   correctness rides on the test suite + real-spec fixtures
   ([../../10](../../10-testing-and-tooling.md#102)); playground smoketests are a *secondary*
   check only. This **intentionally diverges** from the design's "playground as CI harness,
   not an afterthought" framing ([../../12](../../12-repository-and-module-layout.md#124),
   [../../10](../../10-testing-and-tooling.md#105)) — carried as an input to
   [ticket 04](./04-quality-ci-oss-weave.md).

**Boundaries deferred to sibling tickets:** cross-cutting DoD composition (a11y / security /
perf / CI / OSS-prep) → [ticket 04](./04-quality-ci-oss-weave.md); the dependency-ordered
sequence of these units → [ticket 02](./02-build-order-and-dependencies.md); the
sizing/estimation layer → [ticket 05](./05-sizing-and-estimation.md).
