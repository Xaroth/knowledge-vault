---
type: Design Document
title: "Testing & Tooling"
description: "Test strategy, real-world spec corpus, build tooling, CI gates, and the per-component definition of done."
tags: [design, esi, api-explorer, openapi, testing, tooling, ci]
timestamp: 2026-07-18T19:00:46Z
---

# 10. Testing & Tooling

What makes the library trustworthy to us and to external contributors, and concrete enough for a
medior engineer to scaffold CI on day one. Tooling choices follow the POC where it already made a good
call.

## 10.1 Test strategy by layer

### Parser (`@eve-online-tools/openapi-model`) — the highest-value tests
The model is the core contract, so it gets the most rigorous testing.

- **Snapshot normalization** against a **real-world spec corpus** (see [§10.2](#corpus)). Each spec →
  `NormalizedSpec` snapshot; a diff is a reviewed, intentional change.
- **Dialect coverage:** targeted fixtures for OpenAPI 3.0 *and* 3.1 constructs — `nullable` vs
  `type:[…,"null"]`, type arrays, `const`, `$defs`, `discriminator`, `allOf` merge + conflict,
  `readOnly`/`writeOnly`, cyclic refs.
- **Resilience:** deliberately broken specs (dangling `$ref`, conflicting `allOf`, missing
  `operationId`, wrong types) assert we produce diagnostics and a degraded-but-usable model, and never
  throw on a single bad node ([09](./09-quality-and-resilience.md#error-handling)).
- **Property-based** spot checks where cheap (e.g. cycle handling terminates for any ref graph).

### Renderer (`@eve-online-tools/openapi-explorer`)
- **Component tests** (Testing Library + jsdom): each slot renders its props, honours the a11y pattern,
  is wrapped in an error boundary, and is token-styled.
- **Interaction tests:** navigation + deep-link sync, controlled/uncontrolled selection precedence,
  Try It (fill → validate → send via a mock `fetcher` → render response), variant/discriminator
  selection, collapse persistence.
- **Accessibility:** `vitest-axe`/`jest-axe` on every view type — zero violations; a keyboard-only
  traversal test; contrast assertions on both themes.
- **Security:** the XSS corpus test from [09](./09-quality-and-resilience.md#security).
- **DOM contract:** assert the documented `data-oae-*` attributes exist on their elements (guards the
  public CSS API from accidental removal — see [11](./11-open-source-and-api-stability.md)).
- **SSR:** render each view with `renderToString` and assert no `window`/`document` access at module
  or render scope.

### Integration / smoke
- The **playground** builds and typechecks in CI (a real integration smoke test).
- An end-to-end render of the **live ESI spec fixture** asserts all operations/schemas render without
  error-boundary fallbacks.

## 10.2 The real-world spec corpus {#corpus}
A curated set committed as fixtures, chosen to stress different shapes:

| Spec | Why |
| --- | --- |
| Petstore 3.0 & 3.1 | Canonical baseline, both dialects. |
| **ESI 3.1** | Our target: large, OAuth2, heavy `x-*`, recursive schemas. |
| Stripe | Very large; deep `allOf`; discriminators; the classic stress test. |
| GitHub | Huge; webhooks; unusual patterns. |
| Hand-crafted "broken" specs | Resilience/diagnostics coverage. |

Large third-party specs are vendored as fixtures (not fetched in CI) for determinism. This corpus is
what lets a contributor's PR prove it didn't regress rendering for anyone.

## 10.3 Per-component "definition of done" {#component-dod}
Every slot component ([04](./04-renderer-component.md#slots)) is complete only when it:

1. renders all documented props, with sensible empty/missing handling;
2. implements its a11y pattern ([09](./09-quality-and-resilience.md#accessibility));
3. is wrapped in (or safe under) an error boundary;
4. styles exclusively via `--oae-*` tokens (no hard-coded colours/fonts);
5. exposes its documented `data-oae-*` hook(s);
6. has a snapshot test + at least one interaction test + an axe check;
7. contains no `dangerouslySetInnerHTML` and sanitizes any spec-derived URL.

Tickets reference this list rather than restating it.

## 10.4 Tooling

| Concern | Choice | Notes |
| --- | --- | --- |
| Monorepo | **pnpm workspaces + Turborepo** | As the POC. `apps/*`, `packages/*`. |
| Language | **TypeScript**, strict | ES2022, bundler resolution, `declaration` + maps. |
| Parser build | **tsup** | Dual ESM+CJS + `.d.ts`; subpath export for fixtures. |
| Renderer build | **Vite library mode** + `vite-plugin-dts` | ESM+CJS; compiled `style.css`; raw Sass partials shipped. |
| Test runner | **Vitest** + Testing Library + jsdom | `api: 'modern-compiler'` for Sass. |
| A11y tests | **vitest-axe** | Gating. |
| Lint/format | **ESLint 9 (flat)** + Prettier | Custom rule: ban `dangerouslySetInnerHTML` in renderer. |
| Bundle budget | **size-limit** | Per-entry budgets; markdown + highlighter must be separate lazy chunks and excluded from the core budget. |
| Versioning | **Changesets** | Independent versions per package ([11](./11-open-source-and-api-stability.md)). |
| CI | GitHub Actions | `format:check → lint → typecheck → test → build → size-limit`, on Node LTS. |

## 10.5 CI gates (must pass to merge)
Format, lint, typecheck, unit/component tests, a11y tests, SSR test, size-limit budget, and a
successful build of both packages + the playground. A changeset is required for any change to a
package's public surface.
