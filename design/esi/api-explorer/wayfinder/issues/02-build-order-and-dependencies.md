---
type: Wayfinder Ticket
title: "Build order & dependency backbone"
description: "Formalize the design's phase sequencing and one-way package dependencies into the plan's ordered backbone."
tags: [design, esi, api-explorer, wayfinder, build-plan]
timestamp: 2026-07-20T00:00:00Z
---

# Build order & dependency backbone

- **Type:** grilling
- **Status:** resolved
- **Blocked by:** 01

## Question

Turn the design's Phase 0–4 sequencing
([../../08](../../08-esi-integration-and-migration.md#84)) and the strict one-way dependency
direction `adapter → explorer → model` ([../../02](../../02-architecture.md#21)) into the
plan's ordered backbone: the dependency-wired sequence of buildable units.

Must capture the hard gates: `NormalizedSpec` frozen before any consumer builds against it,
the model stabilizing before the renderer (renderer depends via a caret range, not a pinned
workspace version), and the provider-stack/slot-registry existing before individual slots.

## Answer

The backbone is a **dependency DAG**, not a linear list: each unit names its blockers, and
anything unblocked is takeable. It serves a solo builder (topologically sorted into one path)
or a team (parallel tracks) without hard-coding a serial order. Builder capacity is
unspecified; the neutral DAG handles both.

### Front of the plan — a deliberately serial bottleneck

1. **3.1 schema-normalization spike** — the highest-risk item
   ([../../03 §3.5](../../03-parser-package.md#35), [../../08 §8.5](../../08-esi-integration-and-migration.md#85));
   runs **first** and feeds *both* the model shape and the skeleton.
2. **Draft `NormalizedSpec`** (types + `MODEL_VERSION`) — build-against, explicitly not frozen.
3. **parser→renderer walking skeleton** — the real ESI spec through a minimal parser path into
   a minimal renderer showing one operation; stresses the draft contract against a real
   consumer.
4. **Freeze `NormalizedSpec`** + set the semver/`MODEL_VERSION` baseline — *earned* by the
   skeleton, not declared on paper. Breadth builds only against the frozen contract.

### Breadth — parallelizable once the contract is frozen

- **Parser track** — remaining modules against the frozen shape: `resolve-input`,
  `original-spec-snapshot` (precedes dereference), `validate-spec` + version gate,
  `load-policy`, the `normalize-schema` splits ({3.0↔3.1 convergence} — seeded by the spike —
  {`allOf` merge}, {cycle handling}), `normalize-operations`, `normalize-security`,
  `normalize-extensions`, `diagnostics`.
- **Renderer track** — **provider-stack + slot-registry first** (slots resolve through it),
  then slots/features: nav, operation view, schema viewer, Try It, responses, code samples,
  markdown, built-in search, syntax highlighting.
- The two tracks depend only on the frozen contract, not on each other.

**Spike placement rule:** every spike blocks its build unit and sits immediately before it on
the critical path; the 3.1 spike is the sole front-loaded exception (above). *Which* hard
parts get a spike, and each exit criterion, is [ticket 03](./03-spike-inventory-and-gates.md).

### Built in isolation

Everything above is built in a **standalone monorepo**, proven by tests + real-spec fixtures
(+ the demand-driven playground) — **no wiring into the existing app**.

**ESI adapter** opens when the **renderer→adapter seam** is ready (frozen contract +
slot-registry + the Try It/`fetcher`/auth seam) — *not* full renderer parity. Adapter
sub-units (vendor renderers, auth+`fetcher`+scopes, labels, theme tokens) run **parallel to**
the remaining renderer breadth, led by the renderer→adapter walking skeleton. Still isolated
from the app.

### Integration & cutover tail — the first and only touch of the existing codebase

1. **Side-by-side integration** on `/api-explorer` behind an **A/B feature flag** — the
   flagged-page wiring unit; first contact with the app.
2. **A/B parity verification** in the real app — the [../../05 §5.1 matrix](../../05-feature-set.md),
   real token/scope flow, deep links, search, 3.1 schemas. A **hard gate**.
3. **Flip default → soak** behind the flag.
4. **Tear-down / cleanup**, each its own small unit, all **blocked by the flip**: remove the
   `@stoplight` wrapper + `theme.scss`, drop the three `patch-package` patches, remove
   `@stoplight/elements(-core)` and `react-router(-dom)` + the router shim. No deletion until
   parity is signed off and the flag flipped — rollback stays trivial to the point of no return.

### Invariants and scope

- **Island-safe self-containment is an in-plan invariant** (goal 5), enforced from day one by
  the zero-context mount test — the component needs nothing but props, no external React
  context. This is *how* it is built, not a phase, and it is what makes a future Astro
  migration cheap rather than a full rewrite.
- **Out of scope:** the **Astro host-wrapper integration** (mounting as an Astro island in the
  real app). The Astro rewrite lands *after* the API Explorer rework, so it is a separate later
  effort; this plan only preserves the self-containment invariant. (Moved to the map's Out of
  scope.)
- **Deferred / non-blocking:** the **SDE-pages repoint** off `ElementsPanel`/`Stack`/`Wrapper`
  — decoupled follow-up, not on the cutover critical path (stays in Not yet specified).

**Boundaries to sibling tickets:** spike selection + exit criteria →
[ticket 03](./03-spike-inventory-and-gates.md); cross-cutting DoD/CI weave (incl. the
decoupled-playground CI question) → [ticket 04](./04-quality-ci-oss-weave.md);
sizing/estimation over these units → [ticket 05](./05-sizing-and-estimation.md).
