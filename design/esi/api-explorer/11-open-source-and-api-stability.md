---
type: Design Document
title: "Open Source & API Stability"
description: "Public API surface, the CSS/DOM override contract, semver policy, and open-source governance decisions."
tags: [design, esi, api-explorer, openapi, open-source, api-stability, semver, governance]
timestamp: 2026-07-18T19:00:46Z
---

# 11. Open Source & API Stability

Open-sourcing is a hard requirement, and it changes the design: **every exported name and every DOM
hook becomes a contract we must keep or break deliberately.** This document defines the public surface,
the stability policy, and the governance placeholders.

> Per the owner's instruction, **user-facing documentation is deferred** to the as-built
> re-evaluation. This document covers release *engineering and policy*, not end-user docs.

## 11.1 The public API surface

Only what is exported from each package's top-level entry is public. Deep imports (into internal
modules) are unsupported and may break in any release — enforced by not publishing internal paths in
`exports` and by an ESLint boundary rule internally.

### `@eve-online-tools/openapi-model` (public)
- `loadSpec`, `normalizeSpec`, `resolveInput`
- `isNormalizedSpec`, `refName`, `componentSchemaName`, `visitNormalizedSchema`
- Load-policy: `assertFetchUrlAllowed`, `isBlockedFetchHost`, `TrustMode`
- All `Normalized*` types, `Diagnostic`, `LoadSpecResult`, `LoadSpecOptions`
- `MODEL_VERSION` constant ([§11.4](#model-version))

### `@eve-online-tools/openapi-explorer` (public)
- `ApiExplorer` (default component) + `ApiExplorerProps`
- The slot registry types: `ExplorerComponents` + every `*Props` interface (so overrides are typed)
- `ExtensionComponents`, `ExtensionComponentProps`
- `AuthConfig`, `AuthValue`, `Fetcher`, `Selection`, `TryItFieldRef`
- `ExplorerLabels`, `defaultLabels`
- `useTryItValue`, `useExplorerComponents` (for override authors)
- CSS: `./style.css`, `./styles` (Sass partials)

Anything not in these lists is internal.

## 11.2 The CSS / DOM contract {#css-dom-contract}
We criticized Stoplight for forcing us to target private `.sl-*` classes
([01](./01-motivation-and-goals.md), pain point **P3**). We will not recreate that. Two — and only two
— things are the public styling API, both semver-covered:

1. **CSS custom properties** — the `--oae-*` token set ([07](./07-styling-and-theming.md#tokens)).
2. **Stable DOM hooks** — a documented set of `data-oae-*` attributes on structural elements
   (`data-oae-root`, `data-oae-nav`, `data-oae-operation`, `data-oae-method`, `data-oae-schema`,
   `data-oae-tryit`, `data-oae-response`, …). A test asserts these exist
   ([10](./10-testing-and-tooling.md)).

Internal class names are **CSS-module-hashed** precisely so they *can't* be targeted and can't become
an accidental contract. If you find yourself needing to style something without a hook, that's a
request to add a hook — not a reason to target internals.

## 11.3 Extension model is "swap a component" — and nothing more {#no-plugin-framework}
An explicit **anti-goal** ([A2.2](./notes/03-architect-round2.md)): there is no plugin system, no
runtime plugin discovery, no configuration DSL, no lifecycle hooks. Extensibility is exactly two
mechanisms, both plain React:

- **Slot overrides** — replace a visual component via the `components` prop.
- **Vendor-extension components** — render `x-*` via the `extensions` prop.

Both require the consumer to compose in React (Tier 2 integration —
[04](./04-renderer-component.md#integration-tiers)). This is deliberately the *whole* extension story.
A future contributor proposing a plugin framework must first show a concrete need the component model
cannot meet; the default answer is no.

## 11.4 Versioning & stability policy {#model-version}
- **SemVer**, enforced via **Changesets**. The two packages **version independently** — the model will
  stabilize before the renderer, and a model patch must not force a renderer release. The renderer
  depends on the model via a **caret range**, not a pinned workspace version.
- **What counts as breaking:**
  - Model: any change to the `NormalizedSpec` shape; removing/renaming an export; adding a *required*
    label key to `defaultLabels`.
  - Renderer: removing/renaming a prop, slot, or `data-oae-*` hook; removing a `--oae-*` token;
    changing controlled/uncontrolled precedence.
- **`MODEL_VERSION`** ([A2.1](./notes/03-architect-round2.md)): `NormalizedSpec` carries a
  `modelVersion: number`. Because a host can pass a pre-built model built by a different parser
  version, the renderer checks it and renders a clear error on mismatch instead of crashing on a
  missing field. Bumped on any breaking model shape change.
- **React support:** peer dependency `^18 || ^19`.
- **Deprecation:** deprecated exports are marked `@deprecated` with a replacement, kept for one major,
  then removed.

## 11.5 Repository & contribution basics
- **License:** permissive OSS license — **owner decision** (MIT is the default expectation for a
  drop-in React library). Placeholder until confirmed.
- **`CONTRIBUTING.md`** (minimal for now): how to run the monorepo, the CI gates
  ([10](./10-testing-and-tooling.md#105-ci-gates-must-pass-to-merge)), the per-component
  checklist, and the "swap a component, don't build a framework" principle.
- **Issue/PR templates**, a code of conduct, and `CODEOWNERS` per the governance decision below.
- **Changelog** generated by Changesets.

## 11.6 Owner decisions {#decisions-needed}

### Decided

- **Package scope & names — DECIDED.** Packages publish under **`@eve-online-tools`**:
  `@eve-online-tools/openapi-model` and `@eve-online-tools/openapi-explorer`. If CCP chooses to publish
  under the company name, they move to the **`@ccpgames`** scope. The package *names* stay generic
  (no "eve"/"esi") so the toolkit reads as general-purpose; the ESI adapter stays private in this app.
  A scope move is a coordinated rename (publish under the new scope, deprecate the old with a pointer),
  not a breaking API change.

### Still open

- **Governance & ownership.** Which team owns the repo, release cadence, and issue triage once public;
  who is in `CODEOWNERS`; where the repo lives (and, tied to the scope choice, whether it lives under a
  CCP GitHub org). This does not block building the packages internally; it blocks the first public
  publish.

- **License.** Permissive OSS license — MIT is the default expectation for a drop-in React library
  ([§11.5](#115-repository--contribution-basics)); confirm before publish.
