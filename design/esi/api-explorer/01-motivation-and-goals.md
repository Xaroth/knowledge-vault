---
type: Design Document
title: "Motivation & Goals"
description: "Why we are replacing @stoplight/elements, the evidenced pain points, and the goals and non-goals."
tags: [design, esi, api-explorer, openapi, motivation, goals, stoplight]
timestamp: 2026-07-18T19:00:46Z
---

# 1. Motivation & Goals

## 1.1 Where we are today

The API Explorer page (`/api-explorer`) renders the ESI OpenAPI document through
`@stoplight/elements` v9.0.4. The integration is not a thin dependency — it is a substantial
abstraction layer plus a set of workarounds that reach into Elements' internals:

- A wrapper layer in [src/components/@stoplight/elements/](../src/components/@stoplight/elements/)
  (`api.tsx`, `wrapper.tsx`, `theme.scss`, `panel.tsx`, `stack.tsx`).
- The real consumer, [api-spec.tsx](../src/components/esi/api-spec/api-spec.tsx), which pre-processes
  the spec, injects a custom router and search, and feeds Elements an in-memory document.
- ESI vendor-extension renderers in
  [src/components/esi/api-spec/vendor/](../src/components/esi/api-spec/vendor/).
- Three `patch-package` patches under [patches/](../patches/).

It works. But the cost of keeping it working is high and rising, and an upcoming platform change
(Astro) invalidates a core assumption it relies on.

## 1.2 Concrete pain points

These are the specific, evidenced problems — not general dissatisfaction. Each is something the
replacement is designed to eliminate.

### P1 — We patch the library to render our data

Vendor extensions at the *service* level do not render without a source patch. We inject a
`NodeVendorExtensions` call into `HttpServiceComponent` across all three module formats of
`@stoplight/elements-core` (`patches/@stoplight+elements-core+9.0.4.patch`). We also carry a bug-fix
patch to `@stoplight/json-schema-viewer` (description resolution on combiner/choice nodes) and a
patch adding a missing `types` export to two package manifests. Patches are re-applied on every
install and silently rot on every upgrade.

### P2 — Authentication is injected through the DOM

There is no supported way to hand Elements a bearer token, so the token is written directly into
Stoplight's Try It input. [x-required-scope.tsx](../src/components/esi/api-spec/vendor/x-required-scope.tsx)
targets `[data-test="auth-try-it-row"] input[type="text"]`, waits for it with a `MutationObserver`,
sets `input.value = 'Bearer …'`, and dispatches a synthetic `input` event so React notices. This is
the single most fragile piece of the integration: it depends on Stoplight's private markup and
breaks on any DOM change in the library.

### P3 — A CSS override war against private class names

[theme.scss](../src/components/@stoplight/elements/theme.scss) is ~350 lines scoped under
`[data-wrapper='@stoplight/elements']`. It remaps Stoplight's HSL token system onto Mantine tokens,
`@nested-import`s Stoplight's full stylesheet, re-colours code blocks with `!important`, and drives a
sticky-sidebar scroll animation by targeting internal selectors: `.sl-elements`, `.sl-sticky`,
`.sl-bg-canvas`, `[data-testid='two-column-right']`, `.sl-stack--{i}`, `[data-test='security']`, and
more. We do not control this DOM contract, so every upgrade risks silently breaking layout and theme.

### P4 — It cannot server-side render

Elements relies on the DOM at module scope, so it is imported with `dynamic(…, { ssr: false })`
([api.tsx:45-54](../src/components/@stoplight/elements/api.tsx#L45-L54)). The result is a mandatory
client-only load with a loading-overlay flash, and no server-rendered content for the docs — poor for
first paint and for indexability of an API reference.

### P5 — A routing dependency we only carry for Elements

Elements navigates with `react-router`. We pull in `react-router` / `react-router-dom` v6 purely to
satisfy it, and bridge our own navigation through a `StoplightRouter` compat shim plus the
`outerRouter` prop. That is a whole routing library and a shim in service of one component.

### P6 — Deep coupling to Mantine and Mosaic theming

The component only looks right because `theme.scss` bridges Stoplight/Mosaic's design tokens to
Mantine's, and `wrapper.tsx` injects a script that primes `localStorage['mosaic-theme']` before
Mosaic's own theme detection can run. We are wiring three theming systems together (Stoplight,
Mosaic, Mantine) to render one screen.

### P7 — The Astro Islands boundary breaks context sharing

The current design leans on **host-provided React context**: the vendor renderers call
`useCharacterAuth()` and `useAccessRequestModal()`, [api-spec.tsx](../src/components/esi/api-spec/api-spec.tsx)
uses `next-intl`'s `useTranslations`, Mantine components read the Mantine provider, and the spec is
supplied through an `ApiSpecProvider` context. Under Astro's Islands architecture, **React context
does not cross the island boundary**. An interactive island can provide and consume its *own* context
internally, and can be wrapped in a provider, but it cannot reach a provider that lives outside the
island. Today's component would lose its auth, its translations, its theme, and its data at the
boundary.

## 1.3 What "better" means

The replacement must be a component we can reason about and evolve — where rendering our own data is a
first-class prop (not a patch), where authentication is a first-class input (not a DOM write), where
styling is ours (not an override war), and where the whole thing is a single self-contained island
that needs nothing from outside except its props.

## 1.4 Goals

1. **A configurable React component** with a well-defined props API — comparable in spirit to
   Elements' `APIProps`, but tailored to us and without the dead surface. See
   [04-renderer-component.md](./04-renderer-component.md).
2. **Feature parity or better** versus the ESI Explorer POC, and parity with the parts of Elements we
   actually use (navigation, operation view, schema viewer, Try It, code samples, markdown,
   responses). See [05-feature-set.md](./05-feature-set.md).
3. **First-class authentication hook**, so a logged-in user's token flows into Try It through a
   documented API rather than DOM injection. See
   [06-authentication-and-try-it.md](./06-authentication-and-try-it.md).
4. **Self-contained, overridable styling** — clearly named CSS custom properties and swappable
   sub-components, so the component drops into other sites and is themable without forking.
   See [07-styling-and-theming.md](./07-styling-and-theming.md).
5. **Island-safe self-containment** — no dependence on React context that lives outside the
   component. Everything from the host comes in via props (data, auth, labels/i18n, theme, callbacks).
   See [02-architecture.md](./02-architecture.md#islands).
6. **OpenAPI 3.0 *and* 3.1 support**, correctly — including 3.1 type arrays, `const`, `$defs`, and
   `null` unions, which the POC does not yet handle. See
   [03-parser-package.md](./03-parser-package.md#versions).
7. **Renders the ESI spec** as served from
   `https://esi.evetech.net/meta/openapi.json?compatibility_date=<date>` (OpenAPI 3.1.0, ~203 paths,
   313 schemas, OAuth2 / EVE SSO, and ESI's `x-*` extensions). See
   [08-esi-integration-and-migration.md](./08-esi-integration-and-migration.md).
8. **Parsing in a separate, framework-agnostic package**, reusing the POC's
   swagger-parser-based approach. See [03-parser-package.md](./03-parser-package.md).

## 1.5 Non-goals (YAGNI)

Explicitly out of scope, so the design stays honest about its bounds:

- **Not a spec editor.** Read-only rendering. No authoring, linting, or diffing UI.
- **Not a general API-docs platform.** No multi-document catalogues, no Markdown "articles" tree, no
  mock server. Elements ships these; we do not use them.
- **Not an OAuth client.** The component *consumes* a token via its auth hook; it does not run the SSO
  redirect/exchange. That stays in the host (as it does today).
- **Not a Swagger 2.0 renderer.** OpenAPI 3.x only, matching the POC, which rejects 2.x with guidance
  to convert.
- **No bundled data-fetching framework.** The component accepts a spec (URL or document) and can fetch
  a URL itself, but it does not impose react-query or a caching layer on hosts.
- **No design-system dependency.** It will not depend on Mantine, Mosaic, or any host UI kit. It ships
  its own minimal styling.

## 1.6 Success criteria

The work is done when:

- `/api-explorer` renders the live ESI spec with **no `patch-package` patches** and **no
  `react-router`** in the dependency tree for this feature.
- The docs page **server-renders** its static content; interactivity hydrates on top.
- A logged-in character's token reaches Try It through a **typed prop/callback**, with the
  access-request/scope flow intact.
- The component renders correctly when **embedded as a single Astro island** with no external React
  context available.
- Theming a host integration requires **only CSS custom properties and/or slot overrides** — no
  forking, no `!important` selector wars.
- OpenAPI **3.0 and 3.1** fixtures both render correctly, including 3.1-only schema constructs.
- The component is **accessible** (automated `axe` checks pass; full flow keyboard-operable),
  **resilient** (a malformed spec degrades with diagnostics rather than crashing), and **safe**
  (untrusted spec content cannot inject markup). See
  [09-quality-and-resilience.md](./09-quality-and-resilience.md).
- The library is **publishable**: an explicit public API surface, semver policy, a documented CSS/DOM
  override contract, and CI gating tests + a11y + bundle budget. See
  [10-testing-and-tooling.md](./10-testing-and-tooling.md) and
  [11-open-source-and-api-stability.md](./11-open-source-and-api-stability.md).

> **Note on scope:** open-sourcing became a firm requirement during review, which added the
> accessibility, resilience, security, testing, and release-engineering requirements now captured in
> docs `09`–`12`. The review dialogue that produced them is in [notes/](./notes/).
