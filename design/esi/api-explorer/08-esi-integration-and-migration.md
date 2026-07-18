---
type: Design Document
title: "ESI Integration & Migration"
description: "How this app wires ESI specifics in, and the phased migration off @stoplight/elements."
tags: [design, esi, api-explorer, openapi, migration, vendor-extensions, integration]
timestamp: 2026-07-18T19:00:46Z
---

# 8. ESI Integration & Migration

How this app wires ESI specifics into the generic component, and how we migrate off
`@stoplight/elements`. The renderer and the parser know nothing about ESI; **all** ESI knowledge lives
in an app-owned adapter (the evolution of today's [src/components/esi/](../src/components/esi/)).

## 8.1 The ESI spec, precisely

Confirmed by inspecting the live document (via the POC fixture and the app's loader):

- **URL:** `https://esi.evetech.net/meta/openapi.json` with an `X-Compatibility-Date` header (or
  `?compatibility_date=YYYY-MM-DD` on the public download links). Host varies by tier:
  `esi.evetech.net` (live), `esi-test.evetech.net`, `esi-dev.evetech.net`
  ([src/lib/esi/util.ts](../src/lib/esi/util.ts)).
- **Version:** OpenAPI **3.1.0** — this is why the parser must handle 3.1 constructs
  ([03-parser-package.md](./03-parser-package.md#versions)).
- **Size / shape:** ~203 paths, 313 component schemas, 36 tags, a single server
  (`https://esi.evetech.net`), no webhooks.
- **Security:** one scheme, `OAuth2` (authorization-code, `https://login.eveonline.com/v2/oauth/authorize`),
  with a large scope list; no global security — operations opt in per-route.

## 8.2 ESI vendor extensions

The extensions present in the spec, and those synthesized client-side by the app's pre-processing
middlewares, each mapped to a renderer registered in the component's extension registry
([04-renderer-component.md](./04-renderer-component.md#extensions)):

| Extension | Origin | Entity | Rendered as |
| --- | --- | --- | --- |
| `x-compatibility-date` | spec | operation | (informational; drives compatibility handling) |
| `x-rate-limit` | spec | operation | Rate-limit row in the "Details" table |
| `x-cache-age` / `x-cache-mode` | spec | operation | Cache info (client/server, TTL vs event-based) |
| `x-required-roles` | spec | operation | Roles row in "Details" |
| `x-pagination` | spec | operation | Pagination row (e.g. cursor-based) |
| `x-enum-descriptions` | spec | schema | Enum value → description list (consumed by the model) |
| `x-common-model` | spec | schema | Named scalar-alias handling (consumed by the model) |
| `x-cache-info` | **synthesized** (`middlewares.ts`) | operation | Combined cache info in "Details" |
| `x-required-scope` | **synthesized** (from `security`) | operation | Authentication section + authorize action |
| `x-overview` | **synthesized** (from options) | service | Overview panel (useful links, downloads, changelog) |

Note the app is mid-refactor: it recently consolidated cache/rate-limit/scope/roles/pagination into a
single `RouteDetails` "Details" table
([route-details.tsx](../src/components/esi/api-spec/vendor/route-details.tsx)), keeping
`x-required-scope` separate (it also drives the auth action), plus `x-overview` and
`x-enum-descriptions`. The adapter carries these renderers over largely unchanged — they become
`ExtensionComponents` entries instead of `renderExtensionAddon` output, and the **service-level**
`x-overview` renders **without the `elements-core` patch**.

## 8.3 The ESI adapter — responsibility map

Everything ESI, in one place. Each item maps a current mechanism to its island-safe replacement:

| Concern | Today | Adapter (new) |
| --- | --- | --- |
| **Spec fetch** | `api-spec-provider` + react-query, `fetchESIOpenAPISpec` | Same fetch; parse with `@eve-online-tools/openapi-model` (server-side when possible) → `NormalizedSpec`. |
| **Compatibility date** | `compatibility-date-provider`, `?compatibility_date` query, newest default | Unchanged; selected date drives the fetch and re-normalization. |
| **Pre-processing** | `middlewares.ts` mutates the raw spec (scope/cache/overview/desc) | Keep as pre-parse middlewares on the raw document, **or** move synthesized extensions to post-normalize enrichment of `NormalizedSpec`. Both are adapter-local. |
| **Auth token** | DOM injection via `x-required-scope.tsx` | `auth` prop + `fetcher` from `character-auth-provider` ([06](./06-authentication-and-try-it.md)). |
| **Scope grants** | `useAccessRequestModal()` context | `onRequestScopes` callback opening the existing modal. |
| **Vendor rendering** | `renderExtensionAddon` + `elements-core` patch | `extensions` registry (covers service level, no patch). |
| **Search** | bespoke `SpotlightSearch` + `react-router` | built-in navigation search ([04](./04-renderer-component.md#navigation)). |
| **Routing** | `StoplightRouter` shim + `react-router` | built-in hash deep linking (no router). |
| **i18n** | `next-intl` `useTranslations` | `labels` dictionary built from `api-spec.*` keys in [en.json](../src/localization/en.json). |
| **Theme** | `theme.scss` Mantine/Mosaic bridge | `--oae-*` token block ([07](./07-styling-and-theming.md#77-how-the-esi-look-is-achieved)). |
| **Overview panels / changelog** | `overview.tsx`, `changelog.tsx` (Mantine `ElementsPanel`) | `x-overview` extension renderer using the component's own panel primitives or app components. |

## 8.4 Migration strategy

Incremental and low-risk. The generic packages can be built and proven before the app switches over.

### Phase 0 — Foundations (parallelizable)
- Stand up `@eve-online-tools/openapi-model` from the POC parser; **add OpenAPI 3.1 support** and a 3.1 test
  fixture (the live ESI spec). This is the highest-risk item, so it goes first.
- Stand up `@eve-online-tools/openapi-explorer` from the POC renderer; close the four POC gaps (search, syntax
  highlighting, spec examples, and consume the 3.1 model). Verify island-safety (mount with no
  external context).

### Phase 1 — ESI adapter, behind a flag
- Build the adapter ([§8.3](#83-the-esi-adapter--responsibility-map)): port the vendor renderers to
  the extension registry, wire `auth`/`onRequest`/`onRequestScopes`, build the `labels` dictionary,
  supply the EVE theme tokens.
- Render `/api-explorer` with the new component behind a feature flag, alongside the existing Elements
  path, so we can compare them directly.

### Phase 2 — Parity verification
- Walk the parity matrix ([05-feature-set.md](./05-feature-set.md#51-parity-matrix)) against the flagged
  page: every operation renders, Try It works with a real token and scope grant, deep links resolve,
  search works, all `x-*` render (including service-level `x-overview`), 3.1 schemas render correctly.
- Visual review against the current page.

### Phase 3 — Cutover & cleanup
- Flip the flag; delete [src/components/@stoplight/](../src/components/@stoplight/) (wrapper +
  `theme.scss`), the [patches/](../patches/) for Stoplight, and remove `@stoplight/elements`,
  `@stoplight/elements-core`, `react-router`, `react-router-dom` from
  [package.json](../package.json). Drop `patch-package` if nothing else needs it.
- The SDE pages that reuse `ElementsPanel`/`ElementsStack`/`ElementsWrapper`
  ([src/components/sde/](../src/components/sde/)) are **unrelated to the spec renderer** — repoint them
  at the component's own panel primitives or plain app components as a small follow-up; they do not
  block the ESI cutover.

### Phase 4 — Astro readiness (when the platform move happens)
- Because the component is already island-safe and SSR-capable, mounting `<ApiExplorer>` as an Astro
  island is a matter of resolving `auth`/`labels`/`theme`/`spec` in the host wrapper and passing them
  as props. No context crosses the boundary, so nothing breaks — this is the payoff of the
  self-containment constraint held throughout the design ([02-architecture.md](./02-architecture.md#islands)).

## 8.5 Risks & mitigations

| Risk | Mitigation |
| --- | --- |
| 3.1 normalization is subtler than expected | Do it first (Phase 0), with the live ESI spec as the fixture; it is the gating item. |
| Try It behaviour differs from Stoplight in edge cases (serialization, media types) | Port the POC's tested serializer; verify against real ESI operations in Phase 2. |
| Visual regressions vs the current polished look | Feature-flag side-by-side comparison in Phase 1–2; the token system reproduces the EVE look. |
| Hidden reliance on host context surfaces late | The island-safety check in Phase 0 (mount with no external context) catches it before the adapter is built. |
| SDE reuse of `Elements*` primitives | Explicitly decoupled as a non-blocking follow-up ([§8.4 Phase 3](#phase-3--cutover--cleanup)). |
