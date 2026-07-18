---
type: Design Review
title: "Review Round 1 — Lead Engineer"
description: "Lead engineer's response: concessions, two YAGNI rebuttals, and implementation-level detail."
tags: [design, esi, api-explorer, openapi, review-notes, engineer]
timestamp: 2026-07-18T19:00:46Z
---

# Round 1 — Lead Engineer Review

**Date:** 2026-07-18 · **Responding to:** [01-architect-round1.md](./01-architect-round1.md), then
adding implementation-level findings.

Broadly agree. Most of A1–A14 are real and I'll concede them. I push back on two (A6 virtualization,
and partially A13) and refine several into concrete, buildable shapes. Then I add the things a medior
engineer needs to actually start: module layout, dependency choices, and the fiddly model details.

My cross-cutting worry, mirroring the architect's: **we must not let "open-source and extensible"
metastasize into a plugin framework.** The extension seam is "swap a React component" — nothing more.
No plugin lifecycle, no registry DSL, no runtime discovery. That's the YAGNI line I'll defend.

---

## Responses to architect findings

### A1 — Naming · concede
Agreed, owner decision. I'll add one engineering constraint: whatever the scope, the **two packages
version independently** (the model will stabilize before the renderer does), so they must be separate
publishable units from day one, not a single package with subpath exports.

### A2 — Model under-specified · concede, with concrete decisions {#a2}
This is the most important finding. Proposed resolutions (to land in `03`):

- **Cycles:** during normalization, track visited component schemas by ref. On re-encountering one,
  emit a node with `$ref` set and **`circular: true`**, and **do not** expand `properties`/`items`.
  The renderer renders it as a link to the model page, never recursing. Implementation: extend the
  POC's `visitNormalizedSchema` seen-set, which already exists for example generation.
- **`allOf`:** **merge for object display, preserve the source.** Most `allOf` in the wild is
  "base + extensions" and users want the merged object. Emit a merged `properties`/`required`, and
  keep the original parts under `allOf` for slots that want them. Conflicting keys: last-wins, and
  record a diagnostic.
- **`discriminator`:** carry it on the node (`discriminator?: { propertyName, mapping? }`) so the
  variant selector can label branches by discriminator value instead of "Variant 1/2/3".
- **`readOnly`/`writeOnly`/`deprecated`:** carry as booleans; renderer badges them and can filter
  read-only fields out of Try It request bodies.
- **`not`:** carry raw; render a compact "must not match" note. Low effort, avoids data loss.
- **Parameter `content` vs `schema`, response headers, `example` vs `examples`:** normalize all into
  the model (`NormalizedParameter` gets optional `content`; `NormalizedResponse` gets `headers[]`;
  both examples surfaced, `examples` preferred). This closes the POC's "examples ignored" gap in the
  same pass.

### A3 — Extensibility across the boundary · concede (this is the model) {#a3}
Correct and important. Two additions for `04`:
- Overrides must be **stable references** (defined at module scope or `useMemo`'d), or the component
  remounts on every render. Document this as a rule with a one-line example.
- Explicitly reject a non-React (serializable/JSON) extension mechanism as out of scope — it would be
  a second, worse API. Tier-1 users get CSS theming + flags; Tier-2 users get React composition.
  That's the whole story.

### A4 — Our own DOM contract · concede {#a4}
Agreed, and this is genuinely important for OSS. Concrete proposal (→ `07`/`11`): a **frozen, tested**
set of `data-oae-*` attributes on structural elements (`data-oae-root`, `data-oae-nav`,
`data-oae-operation`, `data-oae-method`, `data-oae-schema`, `data-oae-tryit`, …) plus the `--oae-*`
tokens. These are covered by semver; internal class names (CSS-module hashed) are not and must never
be targeted. Add a test that asserts the documented attributes exist (guards against accidental
removal).

### A5 — Accessibility · concede
No pushback. I'll make it concrete in `09`: the specific ARIA patterns (tree, tablist, disclosure,
listbox), roving-tabindex for the nav, focus-move-to-heading on selection change, `prefers-reduced-
motion` honoured for the collapse animation, and `jest-axe`/`vitest-axe` in CI. Cost is real but
front-loading it is far cheaper than retrofitting.

### A6 — Performance · **refine / partial rebut** {#e6}
Agree the *strategy* must be explicit; **rebut premature virtualization.** 203 nav rows and one
rendered operation view is not a performance problem — measure before adding windowing complexity
(which fights find-in-page and a11y). What actually matters, and what I'll specify in `09`:
- **Render one view at a time** (already the shape) — say so explicitly.
- **Cycle-guarded, depth-bounded schema expansion** with a default `defaultExpandedDepth` and an
  "expand" affordance. This is the real correctness+perf item (an un-bounded recursive schema is an
  actual hang, not a micro-opt).
- **Memoize** normalization (do it once) and heavy subtrees (`React.memo` on `SchemaView` rows).
- **Escape hatch:** because `NavigationTree` is a slot, a host with a pathological 5000-operation
  spec can drop in a virtualized tree themselves. We don't pay that complexity for everyone.
So: performance budget + these rules, **no built-in virtualization in v1.**

### A7 — Failure behaviour · concede {#e7-err}
Strongly agree. Concrete: parser returns `{ normalized, diagnostics: Diagnostic[] }` where
`Diagnostic = { level: 'warn'|'error', message, pointer? }`; normalization never throws on a single
bad node — it degrades that node (`invalid: true`, keep raw) and records a diagnostic. Renderer wraps
each view and each top-level schema row in an error boundary that renders a compact fallback +
(in dev) the error. Top-level states: `loading | ready | empty | error`.

### A8 — XSS · concede
No pushback — these become hard rules in `09` and lint-enforced where possible (an ESLint rule banning
`dangerouslySetInnerHTML` in the renderer package). `react-markdown` v9 already refuses raw HTML unless
`rehype-raw` is added; the rule is simply "never add it." Add a URL-sanitizer util (allowlist
`http`/`https`/`mailto`) used for every spec-derived href, and `rel="noopener noreferrer"` on external
links.

### A9 — Testing · concede → `10`
Agreed. One addition: keep the **playground app** (from the POC) as both a manual test harness and the
canonical living example for OSS users — it doubles as a smoke test in CI (build + typecheck).

### A10 — Release engineering · concede → `11`
Agreed. Engineering specifics: **changesets** for versioning, **tsup** (model) + **vite lib**
(renderer) for dual ESM/CJS + types, `"sideEffects": false` except the CSS file, `size-limit` in CI
for the bundle budget, React `^18 || ^19` as a peer dep. Monorepo via **pnpm + turbo** (as the POC
already is).

### A11 — `onRequest` smell · concede, resolved to a single `fetcher` {#e3}
Agreed, and I'll pick the shape: **one prop, `fetcher?: (request: Request) => Promise<Response>`,
defaulting to `globalThis.fetch`.** Rationale: it's the minimal, familiar seam — the host can read the
`Request`, clone/modify it (inject a fresh token, rewrite the URL to a proxy), and call `fetch`, all in
a few lines; or route it anywhere. It subsumes Elements' `tryItCredentialsPolicy` *and*
`tryItCorsProxy` *and* today's DOM injection with one well-typed function. Auth values from the `auth`
prop are applied by the component *before* `fetcher` is called, so simple hosts don't even need a
`fetcher`. Drops the `Request | Response` union entirely.

### A12 — CORS · concede → `06`/`09`
Agreed. Note for `06`: ESI does serve permissive CORS today, so direct Try It works for us; but the
*general* component must detect the opaque/`TypeError` CORS failure and render an actionable message
("this API did not permit a browser request; supply a `fetcher`/proxy"), because most third-party APIs
won't allow it.

### A13 — i18n · concede structure, **refine mechanism** {#e7}
Agree on English-defaults + no bundled translations (YAGNI). For interpolation, **rebut** a
mini-template engine; instead the `labels` type uses **plain strings for static text and functions for
dynamic text**, e.g. `requiredScopes: (count: number) => string`. Type-safe, zero-dependency, trivial
to override. Deep-partial merge over the built-in English `defaultLabels`.

### A14 — Controlled/uncontrolled · concede
Agreed; I'll write the precedence table in `04` and rename `initialSelection` → `defaultSelection` to
match React convention (`value`/`defaultValue`). Precedence: `selection` (controlled) >
`defaultSelection` (uncontrolled initial) > URL hash (if `deepLinking`) > `{ type: 'overview' }`.

---

## Engineer's own findings

### E1 — No concrete module layout · severity: high · status: resolved → [12](../12-repository-and-module-layout.md)
A medior engineer can't start from the current docs — there's no file tree. Added `12` with the full
monorepo layout, per-package `src/` structure (mirroring the POC), and where each concept lives.

### E2 — Dependency choices are hand-wavy · severity: high · status: resolved → [12](../12-repository-and-module-layout.md#dependencies) {#e2}
Pin the decisions so they're not re-litigated per-PR:
- **Markdown:** `react-markdown` + `remark-gfm` + `remark-breaks`, **lazy-loaded** as its own chunk.
  No `rehype-raw` (see A8).
- **Syntax highlighting:** `prism-react-renderer` (runtime, ~tiny, no wasm), **not** Shiki (build-time
  orientation + wasm + async is wrong for a drop-in React lib). Lazy-loaded like markdown.
- **Parser:** `@apidevtools/swagger-parser` + `openapi-types` (as POC). Server-side parse preferred to
  dodge the `Buffer` polyfill.
- **No** UI kit, router, or state library. The only runtime React dep beyond the above is React itself
  (peer).

### E3 — Cross-package versioning · severity: medium · status: resolved → [11](../11-open-source-and-api-stability.md)
Renderer depends on model via a **caret range**, not `workspace:*` pinned — so a model patch doesn't
force a renderer release. The model's `NormalizedSpec` shape change = model **major**.

### E4 — Try It value-store lifecycle needs defined reset semantics · severity: medium · status: resolved → [06](../06-authentication-and-try-it.md) {#e4}
Specify: param/body values are **cleared on operation change** (they're operation-specific); server
selection and per-scheme auth values **persist** across operations (they're API-wide). This matches
the POC's store but the reset rule was implicit — make it explicit so behaviour is predictable.

### E5 — Response/example rendering perf cap · severity: low · status: resolved → [09](../09-quality-and-resilience.md#performance) {#e5}
Pretty-printing + highlighting a multi-MB response body will jank. Cap: above a size threshold, skip
highlighting and offer "show raw". Cheap guard, avoids a real footgun.

### E6 — Playground is a deliverable, not an afterthought · severity: medium · status: resolved → [12](../12-repository-and-module-layout.md)
It's the manual test harness, the CI smoke test, and the first example OSS users read. Keep it in the
monorepo, load the ESI fixture as a static asset (don't inline), and add a couple of non-ESI specs to
prove genericity.

### E7 — "Definition of ready" for each component · severity: medium · status: resolved → [05](../05-feature-set.md#55-definition-of-done-renderer) + [10](../10-testing-and-tooling.md)
Each slot component needs acceptance criteria (props honoured, a11y pattern, error-boundary wrapped,
token-styled, snapshot + interaction test). Rather than restate per component, `10` defines the
**per-component checklist** every slot must satisfy, so tickets can reference it.

---

## Handoff to the architect

I believe A1–A14 are now either conceded with a concrete resolution or refined with a defensible YAGNI
argument (A6 no virtualization, A13 function-valued labels). New docs `09`–`12` carry the net-new
content; targeted edits go into `03`/`04`/`06`/`07`. Please sanity-check the two rebuttals, confirm the
`fetcher` seam is acceptable as the *only* transport hook, and confirm the model decisions in
[A2](#a2) — especially `allOf` merge-and-preserve, which is the one most likely to be contentious.
