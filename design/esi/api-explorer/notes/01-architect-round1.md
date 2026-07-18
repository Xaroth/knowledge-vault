---
type: Design Review
title: "Review Round 1 — Lead Architect"
description: "Lead architect's first-round critique: 14 findings on contracts, cross-cutting quality, and OSS viability."
tags: [design, esi, api-explorer, openapi, review-notes, architect]
timestamp: 2026-07-18T19:00:46Z
---

# Round 1 — Lead Architect Review

**Date:** 2026-07-18 · **Reviewing:** main docs `01`–`08` (initial draft).

The plan is directionally right: the two-package split is sound, the Islands constraint is correctly
identified as the load-bearing requirement, and the pain points are evidenced rather than asserted.
It reads as a good *research write-up*. It is **not yet an implementation plan**, and it is not yet
safe to open-source. The gaps below are what stand between "we understand the problem" and "a medior
engineer can build this and we can publish it."

The theme of this review: **for an open-source library, the contract *is* the product.** Every gap
below is ultimately about an under-specified contract — the data model, the DOM, the extension seam,
the failure behaviour, the security posture.

---

### A1 — Package names/scope are internal; blocks open-sourcing · severity: medium · status: resolved → [11](../11-open-source-and-api-stability.md#decisions-needed)

`@ccp/openapi-model` / `@ccp/openapi-explorer` are internal-scope placeholders. Open-sourcing needs a
neutral name and a published npm org, and a decision on where the repo lives and who governs it. Keep
ESI out of the names (already done — good). **Recommendation:** flag as an owner decision; keep a
placeholder but mark it clearly so it is not mistaken for settled.

### A2 — The normalized model is the core public contract, and it is under-specified · severity: high · status: resolved → [03](../03-parser-package.md)

`NormalizedSpec` is *the* stable API for this project: the renderer, every slot override, and every
third-party consumer depend on it. The draft sketches the happy path but is silent on the cases that
actually decide whether the model is good:

- **Cyclic/recursive schemas** (ESI has self-referential types). How is a cycle represented so the
  renderer doesn't recurse forever?
- **`allOf`** — preserved as an array, or merged for object display? Both have costs. Decide.
- **`discriminator`** on `oneOf`/`anyOf` — needed for a decent variant UX; not mentioned.
- **`not`, `readOnly`/`writeOnly`, `deprecated`, `nullable` interplay with 3.1 unions.**
- **Response headers, parameter `content` (vs `schema`), multiple `examples`, `example` vs `examples`.**

**Recommendation:** treat the model as a versioned public contract; specify each of the above
explicitly with a decision and rationale; add a "model stability" statement.

### A3 — The extensibility model doesn't survive the Astro boundary as written · severity: high · status: resolved → [04](../04-renderer-component.md#integration-tiers)

The plan's own killer constraint undercuts its own extensibility story, and the draft doesn't
reconcile them. Slot/extension overrides are React `ComponentType`s passed as props. **React function
props cannot cross an Astro island boundary** — only serializable props can. So overrides are only
possible when the consumer authors *their own React island* that wraps `<ApiExplorer>` and composes
overrides in React.

This isn't a flaw, but it must be stated as **the** model, with explicit integration tiers:

1. **Tier 1 — drop-in island:** serializable props only (`spec`, `theme`, `labels`, flags). Theming
   via CSS. No component overrides. This is what a `.astro` file can mount directly.
2. **Tier 2 — composed island:** the consumer writes a React wrapper (itself an island) and passes
   slot/extension overrides. Full power.

**Recommendation:** document both tiers; make clear which features each supports. Otherwise OSS users
hit a wall and file bugs.

### A4 — We criticize Stoplight's private classes, then leave our own DOM contract undefined · severity: high · status: resolved → [07](../07-styling-and-theming.md#dom-contract) / [11](../11-open-source-and-api-stability.md)

Pain point **P3** is "we fight Stoplight's private `.sl-*` classes." But overriders *will* target our
DOM too. If we don't define a **stable, documented, semver-covered** set of hooks, we recreate exactly
the problem we're escaping — just with our name on it. **Recommendation:** designate the CSS public
API explicitly: the `--oae-*` token set **and** a small set of stable `data-*` attributes / class
names on key elements. Everything else is private and may change between minor versions. This turns
"clear elements that can be overridden" (a stated goal) into a real, testable contract.

### A5 — Accessibility is entirely absent · severity: high · status: resolved → [09](../09-quality-and-resilience.md#accessibility)

An API reference is a reading-and-navigation tool; a11y is core, not polish — and for OSS it's table
stakes and a frequent source of contributions/complaints. Nothing in the plan addresses keyboard
operability, ARIA semantics for the tree/tabs/disclosure/select widgets, focus management on
navigation, visible focus, contrast of the shipped themes, or reduced-motion. **Recommendation:** make
a11y a first-class requirement with concrete acceptance criteria and a testing gate (axe).

### A6 — No performance/scale strategy for real specs · severity: high · status: refined by [E6](./02-engineer-round1.md#e6) → [09](../09-quality-and-resilience.md#performance)

ESI is 203 operations / 313 schemas, with deep recursive schemas. The draft implies "render one view
at a time" (good) but never states it, and says nothing about: nav rendering cost, bounded/lazy schema
expansion, cycle-guarded recursion with a depth cap + "expand" affordance, or memoization boundaries.
**Recommendation:** specify the performance model and a bundle/perf budget. (I expect the engineer to
push back on premature virtualization — fine, but the *strategy* must be explicit.)

### A7 — Failure behaviour is undefined; fatal for OSS · severity: high · status: resolved → [03](../03-parser-package.md#diagnostics) / [09](../09-quality-and-resilience.md#error-handling)

OSS users will feed this every malformed, half-valid, exotic spec in existence. The plan must define:
- **Parser resilience** — one bad operation/schema must not abort the whole parse. Collect and return
  **diagnostics** (warnings/errors with pointers) rather than throwing on first fault.
- **Render resilience** — per-view (and ideally per-node) error boundaries so one bad node renders a
  fallback, not a blank page. (Note Elements itself wraps components in `withErrorBoundary` — we
  should too.)
- **UX states** — explicit loading / empty / error / partial contracts.

**Recommendation:** add a `diagnostics` channel to the model and an error-boundary strategy to the
renderer.

### A8 — Untrusted spec content is an XSS surface · severity: high · status: resolved → [09](../09-quality-and-resilience.md#security)

SSRF is covered (good), but the bigger everyday risk is **content injection**: descriptions
(markdown), example values, URLs, and `x-*` are all attacker-controlled when a user loads an arbitrary
spec. **Recommendation, as hard requirements:** markdown renders with **raw HTML disabled** (never add
`rehype-raw`); all spec-derived URLs are protocol-allowlisted and links get `rel="noopener
noreferrer"`; no `dangerouslySetInnerHTML` of spec content anywhere; examples rendered as text, not
HTML. This is a security posture, not a nicety.

### A9 — Testing strategy is too thin to earn OSS trust · severity: high · status: resolved → [10](../10-testing-and-tooling.md)

"Both a 3.0 and a 3.1 fixture render" is not enough. **Recommendation:** a **real-world spec corpus**
(Petstore 3.0 + 3.1, Stripe, GitHub, ESI 3.1, plus deliberately-broken specs) driving parser snapshot
+ resilience tests; component/interaction tests for Try It and navigation; automated a11y tests; and
CI gating all of it. This is what makes external contributors trust a PR won't break rendering.

### A10 — No open-source release/stability policy · severity: high · status: resolved → [11](../11-open-source-and-api-stability.md)

"Open-sourced" is now a hard requirement, which imposes obligations the plan ignores: an explicit
**public API surface** (what's exported vs internal; deep imports discouraged), **semver + changesets**,
a **deprecation policy**, **React 18/19 peer support**, a **bundle-size budget**, dual **ESM/CJS**,
**license**, and a minimal **CONTRIBUTING**. (Per the owner's instruction, *user-facing* docs are
deferred until the as-built re-evaluation — this finding is about release *engineering*, not docs.)

### A11 — `onRequest` returning `Request | Response` is a design smell · severity: medium · status: conceded/refined by [E3](./02-engineer-round1.md#e3) → [06](../06-authentication-and-try-it.md)

A single prop whose return type changes the component's behaviour (you-fetch vs I-fetch) is ambiguous
and hard to type well. **Recommendation:** collapse to one clean seam. I lean toward a single
`fetcher(request) => Promise<Response>` (host owns transport entirely; default is `fetch`), which also
cleanly covers auth injection, proxying, and signing. Let the engineer pick the exact shape.

### A12 — CORS reality for Try It is unaddressed · severity: medium · status: resolved → [06](../06-authentication-and-try-it.md#cors) / [09](../09-quality-and-resilience.md)

Browser `fetch` to `esi.evetech.net` (or any third-party API) requires CORS. The plan must state the
operational requirement, the default behaviour when CORS blocks a call, clear error messaging, and the
`fetcher`/proxy escape hatch. Silent failure here is a top support burden for any "Try It" tool.

### A13 — i18n scope and mechanism need a decision · severity: medium · status: refined by [E7](./02-engineer-round1.md#e7) → [04](../04-renderer-component.md#labels)

The `labels` dict is right in spirit but needs: a typed shape with **English defaults**, deep-partial
merge, per-key fallback, and an **interpolation** story for dynamic strings ("requires N scopes").
Also a YAGNI call: **v1 ships English defaults + override-able strings, no bundled translations.**
**Recommendation:** confirm that scope and specify the mechanism.

### A14 — Controlled vs uncontrolled selection is ambiguous · severity: medium · status: resolved → [04](../04-renderer-component.md#navigation)

`deepLinking`, `initialSelection`, and `onSelectionChange` can conflict (who owns the current view —
the URL, the host, or internal state?). **Recommendation:** define the React-idiomatic contract and a
strict precedence: controlled (`selection` prop) > host-uncontrolled-with-default > URL hash >
internal default.

---

## Summary for the engineer

Please respond to A1–A14 (concede / rebut / refine), then add implementation-level detail I've
deliberately left open: the **concrete module layout**, **dependency choices** (markdown, syntax
highlighter), the **cycle representation** in the model, the exact **`fetcher` signature**, and the
**build/test tooling**. Push back hard on anything that smells like scope creep — I'd rather cut than
gold-plate. In particular I expect a YAGNI argument against virtualization (A6) and possibly against
YAML input.
