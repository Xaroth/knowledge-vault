---
type: Design Document
title: "Architecture"
description: "The two-package split, data flow, provider stack, and the Astro Islands / self-containment constraint."
tags: [design, esi, api-explorer, openapi, architecture, astro, islands, ssr]
timestamp: 2026-07-18T19:00:46Z
---

# 2. Architecture

## 2.1 Overview

The system is two packages plus a thin, app-owned adapter:

```
┌─────────────────────────────────────────────────────────────────────┐
│  developers-next (this app)  /  future Astro site                   │
│                                                                     │
│   ESI adapter (app-owned, NOT a shared package)                     │
│   ├─ spec loading + pre-processing (compatibility_date, middlewares)│
│   ├─ EVE SSO token → auth hook                                      │
│   ├─ x-* vendor extension renderers (cache, rate-limit, scope, …)   │
│   └─ i18n label dictionary                                          │
│                     │ props only                                    │
│                     ▼                                               │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │  openapi-explorer   <ApiExplorer/>   (React island)       │     │
│   │  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐│     │
│   │  │ Navigation   │  │ Operation view│  │ Schema viewer    ││     │
│   │  ├──────────────┤  ├───────────────┤  ├──────────────────┤│     │
│   │  │ Try It panel │  │ Code samples  │  │ Vendor ext slots ││     │
│   │  └──────────────┘  └───────────────┘  └──────────────────┘│     │
│   │  self-contained context: components · extensions · try-it │     │
│   │  selection · theme                                        │     │
│   └───────────────────────────────────────────────────────────┘     │
│                     ▲ NormalizedSpec                                │
│                     │                                               │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │  openapi-model   (pure TypeScript, no React)              │     │
│   │  loadSpec → resolve · snapshot · dereference · validate · │     │
│   │  version-gate (3.0/3.1) · normalize → NormalizedSpec      │     │
│   └───────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

The dependency direction is strictly one-way: **adapter → explorer → model**. The model never
imports React; the explorer never imports the parser's raw OpenAPI handling at render time (it
consumes the normalized model); neither package knows anything about ESI.

This is the same clean separation the [ESI Explorer POC](../../../xaroth/esi-explorer) already proves,
generalised and hardened.

## 2.2 Why two packages

- **Single Responsibility.** Parsing/normalizing an OpenAPI document and *rendering* a normalized
  model are different concerns with different change cadences, test strategies, and dependencies
  (swagger-parser + Node-ish concerns vs React + DOM). Splitting them keeps each testable in
  isolation — the POC has 16 parser test files and 49 UI test files precisely because the boundary is
  clean.
- **Dependency Inversion.** The renderer depends on the *`NormalizedSpec` abstraction*, not on how it
  was produced. A host can pass a pre-built `NormalizedSpec` and skip the parser entirely (the POC's
  `normalized` prop does this), or swap the parser wholesale.
- **Reuse.** The normalized model is useful beyond this component — search indexing, SDK/codegen
  inputs, changelog diffing. Keeping it framework-free makes those reuses possible.

## 2.3 Naming

Packages publish under the **`@eve-online-tools`** npm scope, with the option to move to
**`@ccpgames`** if CCP chooses to publish under the company name
([11 §11.6](./11-open-source-and-api-stability.md#decisions-needed)):

- **`@eve-online-tools/openapi-model`** — the parser/normalizer. (POC name: `@xaroth/openapi-parser`.) Named
  "model" rather than "parser" because its product — the normalized model — is the valuable artifact;
  parsing is an implementation detail.
- **`@eve-online-tools/openapi-explorer`** — the renderer. (POC name: `@xaroth/esi-explorer`.) Named *without*
  "esi" on purpose: the renderer is generic; ESI is a consumer.

Neither package name contains "eve" or "esi" beyond the scope — the scope signals stewardship, the
package names stay generic so the components read as a general-purpose OpenAPI toolkit.

The **ESI adapter is not a package** — it is a folder in this app (evolved from today's
[src/components/esi/](../src/components/esi/)). It is the only place ESI knowledge lives.

## 2.4 The Islands / self-containment constraint {#islands}

This constraint shapes the whole design, so it is stated precisely.

**Rule:** the component must not read any React context that is provided *outside* its own root. It
may freely create and consume context *inside* itself, and it may be wrapped by a host in a provider
whose value it receives *as props*. It must not assume a host `ContextProvider` is an ancestor across
an island boundary.

Astro renders each interactive component as an isolated island. Two islands (or an island and the
surrounding app) do **not** share a React tree, so a `Context.Provider` in one is invisible to a
consumer in another. Anything the component needs from the host must therefore cross the boundary as
**serializable props or explicit callbacks**, not as ambient context.

### What this forbids (and how today's design violates it)

| Today relies on host context… | …which fails across an island boundary | Island-safe replacement |
| --- | --- | --- |
| `useCharacterAuth()` in vendor renderers | auth provider is outside the island | `auth` prop + `fetcher` hook ([§06](./06-authentication-and-try-it.md)) |
| `useAccessRequestModal()` | modal context is outside | `onRequestScopes(scopes)` callback prop |
| `next-intl` `useTranslations` | i18n provider is outside | `labels`/`messages` dictionary prop ([§2.7](#i18n)) |
| Mantine provider (theme/components) | provider is outside | self-contained styling, no UI-kit dependency ([§07](./07-styling-and-theming.md)) |
| `ApiSpecProvider` context | provider is outside | `spec` / `normalized` prop |

### What it permits

The component is free to use React context **internally** — indeed it does, for its slot registry,
extension registry, Try It value store, selection, and theme (see [§2.6](#providers)). These
providers live *inside* `<ApiExplorer>`, so they are always in-tree for their consumers regardless of
where the island is mounted. A host may also wrap `<ApiExplorer>` in its own provider *inside the same
island* — that is allowed, because it does not cross a boundary.

### Consequence

`<ApiExplorer>` is designed to be **mountable as a single Astro island with zero external React
context**. Every input is a prop. This is the litmus test for every API decision in these documents.

## 2.5 Data flow

1. **Host obtains a spec.** Either a URL, a raw document object/string, or (in this app) an already
   fetched-and-pre-processed document. See the ESI flow in
   [08-esi-integration-and-migration.md](./08-esi-integration-and-migration.md).
2. **Parse + normalize.** `@eve-online-tools/openapi-model`'s `loadSpec()` produces a `NormalizedSpec`. This can
   happen on the server (it is pure TS) or in the browser. The host may pass the `NormalizedSpec`
   straight into the component to avoid parsing twice.
3. **Render.** `<ApiExplorer>` renders the normalized model. Selection (which operation/schema is
   shown) is derived from the URL hash by default, or host-controlled via props.
4. **Try It.** On send, the component builds a request from the operation + user input, applies auth,
   and calls the host-provided `fetcher` (defaulting to a direct `fetch`). See
   [06-authentication-and-try-it.md](./06-authentication-and-try-it.md).

Because step 2 is server-capable, the static reference content (navigation, operations, schemas,
markdown) can be **server-rendered**, with interactivity (Try It, collapse state, deep-link sync)
hydrating on the client. This directly answers pain point **P4**.

## 2.6 The internal provider stack {#providers}

All of these live **inside** `<ApiExplorer>` and are therefore island-safe. Kept intentionally small
(Simple over complex; no global state library):

| Context | Purpose | Notes |
| --- | --- | --- |
| `ExplorerComponentsProvider` | The slot registry — which component renders each part. | Enables cascading overrides; see [§04](./04-renderer-component.md#slots). |
| `ExtensionComponentsProvider` | The `x-*` vendor-extension renderer registry. | ESI renderers plug in here; see [§04](./04-renderer-component.md#extensions). |
| `TryItValuesProvider` | Persisted Try It field values, as an external store consumed via `useSyncExternalStore`. | Fine-grained per-field subscriptions; survives route changes. |
| `SchemaNavigationProvider` | Lets a rendered type label link to its model page. | Enables clickable model cross-links. |
| `ThemeScope` | Sets `data-theme` and owns the CSS-variable scope. | No provider dependency; just a scoping element ([§07](./07-styling-and-theming.md)). |

This is the same lightweight, hand-rolled approach the POC uses — no Redux/Zustand. Selection state
is local and URL-hash-synced.

## 2.7 Internationalisation without host context {#i18n}

Because `next-intl` context cannot cross the island boundary (**P7**), the component takes its user-
facing strings as a **`labels` dictionary prop** with complete English defaults built in. The host
maps its own i18n system to that dictionary *before* rendering the island. This keeps the component
translatable while owning no i18n framework and depending on no i18n provider. See
[04-renderer-component.md](./04-renderer-component.md#labels).

## 2.8 SSR / hydration model

- **Model layer:** pure TS, runs anywhere. Parse on the server when possible.
- **Renderer:** authored to be SSR-safe. All `window`/`document`/`localStorage`/`sessionStorage`
  access is guarded (`typeof … === 'undefined'`) and deferred to effects — exactly as the POC already
  does for deep-linking, nav-collapse, and sample selection. No module-scope DOM access (the root
  cause of Elements' SSR incompatibility).
- **Hydration:** static reference content renders server-side; interactive affordances attach on
  mount. Deep-link hash reading happens lazily on the client.

## 2.9 Dependency budget

Deliberately small, to keep the component portable and the bundle disciplined (the POC is the
reference point):

- **Model:** `@apidevtools/swagger-parser` (dereference/validate) + `openapi-types` (types only).
  Requires a `Buffer` polyfill in the browser — a known swagger-parser wart (see
  [03-parser-package.md](./03-parser-package.md#buffer)); parsing server-side avoids it.
- **Renderer:** React (peer dependency, `^18 || ^19`), plus markdown (`react-markdown` + `remark-*`)
  loaded as a **lazy chunk** so it stays off the critical path, and a lightweight syntax highlighter
  for code (a gap to close vs the POC — see [05-feature-set.md](./05-feature-set.md)). No UI kit, no
  router, no state library.

Removed relative to today: `@stoplight/elements`, `@stoplight/elements-core`, `@stoplight/mosaic`
(transitively), `react-router`, `react-router-dom`, `patch-package` (for this feature), and the
Mantine dependency *of the docs component* (the app keeps Mantine elsewhere).
