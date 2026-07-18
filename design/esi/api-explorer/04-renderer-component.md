---
type: Design Document
title: "Renderer Component — openapi-explorer"
description: "The public ApiExplorer API, slot and vendor-extension registries, navigation and deep linking."
tags: [design, esi, api-explorer, openapi, renderer, react, component-api, extensibility]
timestamp: 2026-07-18T19:00:46Z
---

# 4. Renderer Component — `@eve-online-tools/openapi-explorer`

A single self-contained React component, `<ApiExplorer>`, that renders a `NormalizedSpec`. Generic
(no ESI knowledge), self-styled, island-safe, and overridable at every visual seam. This is a
generalisation of the POC's
`@xaroth/esi-explorer`.

## 4.1 The public API

```ts
type ApiExplorerProps =
  // exactly one spec source (discriminated union — Dependency Inversion seam)
  ( { spec: string | object } | { normalized: NormalizedSpec } )
  & {
      // --- presentation ---
      theme?: string                        // sets data-theme; any host-defined theme block
      className?: string
      layout?: 'sidebar' | 'stacked'        // default 'sidebar'
      labels?: DeepPartial<ExplorerLabels>  // i18n dictionary; English defaults built in
      loader?: ComponentType                // shown while an internal async load resolves

      // --- overrides (see §4.4 / §4.5) ---
      components?: Partial<ExplorerComponents>   // swap any visual slot
      extensions?: ExtensionComponents           // render x-* vendor extensions

      // --- data / parsing ---
      loadOptions?: LoadSpecOptions              // forwarded to @eve-online-tools/openapi-model
      trustMode?: 'trusted' | 'untrusted'
      trustedServerOrigins?: string[]

      // --- Try It & auth (see §06) ---
      auth?: AuthConfig                          // token(s) supplied by the host
      fetcher?: Fetcher                          // (req: Request) => Promise<Response>; default: fetch
      onRequestScopes?: (scopes: string[]) => void  // e.g. open host's access-request modal
      hideTryIt?: boolean
      hideFields?: TryItFieldRef[]               // host-driven / hidden Try It fields

      // --- navigation (see §4.6) ---
      deepLinking?: boolean | { prefix?: string } // default true
      selection?: Selection                       // controlled
      defaultSelection?: Selection                // uncontrolled initial value
      onSelectionChange?: (selection: Selection) => void

      // --- visibility flags (parity with what we use from Elements) ---
      hideSchemas?: boolean
      hideServerInfo?: boolean
      hideSecurityInfo?: boolean
    }
```

### Design notes

- **One spec source, enforced by types.** A discriminated union means "raw or pre-normalized" is a
  compile-time choice, not a runtime guard. Hosts that parse server-side pass `normalized`; simple
  hosts pass `spec`.
- **Every host input is a prop.** There is no hidden context dependency — the [Islands
  rule](./02-architecture.md#islands) in force. `auth`, `labels`, `fetcher`, `onRequestScopes`,
  `theme` are exactly the things today's design pulls from host context, now passed explicitly.
- **The prop surface is deliberately smaller than Elements'.** We keep only what we use, plus the new
  first-class `auth`/`fetcher` hooks. See [§4.7](#vs-elements) for the mapping.
- **A single transport seam.** All request customisation (auth injection, proxying, signing, running
  the call yourself) goes through one prop, `fetcher(request) => Promise<Response>`, defaulting to
  `fetch`. This replaced an earlier ambiguous `onRequest` that could return either a `Request` or a
  `Response` ([notes A11](./notes/01-architect-round1.md)). See
  [06](./06-authentication-and-try-it.md).

## 4.1a Integration tiers {#integration-tiers}

Because React function props (slot/extension overrides) **cannot cross an Astro island boundary** —
only serializable props can ([02](./02-architecture.md#islands)) — there are two documented ways to
integrate, and each supports a different level of customisation. This distinction is essential for
open-source users ([notes A3](./notes/01-architect-round1.md#a3)).

| | **Tier 1 — drop-in island** | **Tier 2 — composed island** |
| --- | --- | --- |
| How | Mount `<ApiExplorer>` directly (e.g. from a `.astro` file). | Author your own React island that wraps `<ApiExplorer>`. |
| Inputs | Serializable props only: `spec`/`normalized`, `theme`, `labels`, flags, `deepLinking`. | Everything, including `components`, `extensions`, `fetcher`, callbacks. |
| Customisation | CSS tokens + `data-oae-*` hooks ([07](./07-styling-and-theming.md)). | Full slot/extension overrides + transport/auth hooks. |
| Use case | Quick embed, mostly-default docs. | Deep integration — ESI's adapter is Tier 2. |

**Override stability rule (Tier 2):** slot/extension components must be **stable references** (defined
at module scope or memoised). A new function identity each render remounts the subtree. Documented with
a one-line example in the package README.

## 4.2 Anatomy

`<ApiExplorer>` mounts its own provider stack ([§2.6](./02-architecture.md#providers)) and renders a
CSS-grid shell: a navigation `<aside>` and a `<main>` that shows exactly one view:

- **Overview** — `info` title + markdown description; the natural home for a service-level vendor
  extension slot (ESI's `x-overview`).
- **Route detail** — one operation: method + path + summary + markdown description, operation-level
  extension slots, the Try It panel, authentication summary, parameter table, request body, and
  responses.
- **Schema detail** — one named model, standalone and deep-linkable.

## 4.3 Component inventory

The default slots (each overridable via [§4.4](#slots)). This is the parity baseline — it matches or
exceeds what we render through Elements today:

| Slot | Renders |
| --- | --- |
| `NavigationTree` | Tag-grouped operations + a Schemas section + Overview entry; collapsible; search box ([§05](./05-feature-set.md)). |
| `Overview` | Service info + description (markdown). |
| `RouteDetail` / `RouteInfo` / `RouteSpec` | Operation view and its request/response tabs. |
| `SchemaView` | Recursive schema tree: objects, arrays (`items`), maps (`additionalProperties`), `oneOf`/`anyOf`/`allOf` via a variant selector; clickable model cross-links; 3.1 constructs. |
| `SchemaDetail` | Standalone model page. |
| `SchemaTypeLabel` | Type label; model names navigate to the model page. |
| `ParameterTable` | Params (name/in/type/description/enum/extensions). |
| `EnumValues` | "Possible values" with `x-enum-descriptions`. |
| `MarkdownContent` | `react-markdown` + `remark-breaks`, **lazy-loaded** off the critical path. |
| `TryItPanel` | Interactive request builder ([§06](./06-authentication-and-try-it.md)). |
| `CodeSamplePanel` / `RequestSamplePanel` | Multi-language request code samples. |
| `ResponseViewer` / `ResponseExamplePanel` | Response bodies / synthesized examples, syntax-highlighted. |

## 4.4 The component-slot registry {#slots}

The core extensibility mechanism, ported from the POC's `ExplorerComponents` registry.

```ts
interface ExplorerComponents {
  Overview: ComponentType<OverviewProps>
  NavigationTree: ComponentType<NavigationTreeProps>
  RouteDetail: ComponentType<RouteDetailProps>
  RouteInfo: ComponentType<RouteInfoProps>
  RouteSpec: ComponentType<RouteSpecProps>
  SchemaDetail: ComponentType<SchemaDetailProps>
  TryItPanel: ComponentType<TryItPanelProps>
  SchemaView: ComponentType<SchemaViewProps>
  ParameterTable: ComponentType<ParameterTableProps>
  SchemaTypeLabel: ComponentType<SchemaTypeLabelProps>
  MarkdownContent: ComponentType<MarkdownContentProps>
  EnumValues: ComponentType<EnumValuesProps>
  ResponseViewer: ComponentType<ResponseViewerProps>
}
```

Two properties make this powerful and are non-negotiable:

1. **Shallow-merge over defaults.** A host supplies `components={{ SchemaView: MySchemaView }}` and
   overrides just that slot; everything else keeps its default.
2. **Slots resolve each other through the registry, not direct imports.** `RouteSpec` renders schemas
   via `useExplorerComponents().SchemaView` — so overriding `SchemaView` cascades everywhere it is
   used. This is what makes "swap any part" actually compose (Open/Closed + Liskov: any replacement
   honouring the props contract slots in cleanly).

## 4.5 The vendor-extension registry {#extensions}

A parallel registry, keyed by `x-*` name, that renders vendor extensions at the entity where they
appear. This is how ESI specifics are injected **without the renderer knowing about ESI**, and it
replaces both Elements' `renderExtensionAddon` prop *and* the source patch we carry today.

```ts
type ExtensionComponents = Record<string, ComponentType<ExtensionComponentProps>>

interface ExtensionComponentProps {
  value: unknown          // the x-* value
  extensionKey: string    // e.g. "x-rate-limit"
  entity: 'service' | 'operation' | 'parameter' | 'schema' | 'response'
  entityName?: string     // operation id, schema name, …
}
```

At each rendering site, the component looks up any `extensions` present on the normalized node and
renders the registered component for each key. Contrast with today:

- **Elements** exposed a single `renderExtensionAddon` for operation/schema nodes, but **not** for the
  service node — which is why we patched `elements-core` to inject `NodeVendorExtensions` into
  `HttpServiceComponent` (pain point **P1**). Our registry renders extensions at **every** entity,
  including `service`, by design. No patch.
- The ESI adapter registers its renderers here (cache info, rate limit, required scope, required
  roles, pagination, overview, enum descriptions). See
  [08-esi-integration-and-migration.md](./08-esi-integration-and-migration.md).

## 4.6 Navigation, selection & deep linking {#navigation}

Selection is an explicit union (ported from the POC's `selection.ts`):

```ts
type Selection =
  | { type: 'overview' }
  | { type: 'route';  operationId: string }
  | { type: 'schema'; schemaName: string }
```

- **URL-hash deep linking by default** (`#/`, `#/route/<id>`, `#/models/<name>`, optional `prefix`),
  bidirectional-synced via `pushState`/`replaceState` + `hashchange`/`popstate`, guarded against
  feedback loops. This is our own implementation — it removes the `react-router` dependency
  (pain point **P5**) and the `StoplightRouter` compat shim entirely.
- **`deepLinking: false`** disables hash sync (for embeds that must not touch the URL).
- **Controlled / uncontrolled**, following the React idiom (`value`/`defaultValue`). A strict
  precedence resolves conflicts ([notes A14](./notes/01-architect-round1.md)):
  1. `selection` prop present → **controlled** (host owns it; the component calls `onSelectionChange`
     and never self-navigates).
  2. else `defaultSelection` → uncontrolled initial value.
  3. else URL hash (if `deepLinking`).
  4. else `{ type: 'overview' }`.
- Invalid ids resolve to a `route-not-found` / `schema-not-found` state rather than a blank screen.
- Collapse state persists in `sessionStorage` (SSR-guarded).

This subsumes the current app's bespoke `SpotlightSearch` + `StoplightRouter` layer with first-class,
in-component navigation and search.

## 4.7 Mapping from Elements' props {#vs-elements}

For reviewers who know today's surface. Left: what
api.tsx forwards to Elements. Right: our equivalent.

| Elements prop | Our equivalent | Notes |
| --- | --- | --- |
| `apiDescriptionDocument` / `apiDescriptionUrl` | `spec` / `normalized` | Union; also accepts a pre-built model. |
| `layout: sidebar/stacked/responsive` | `layout: sidebar/stacked` | Responsive behaviour is built into both, so the separate mode is dropped. |
| `renderExtensionAddon` | `extensions` registry | Now covers the service node too (no patch). |
| `router` / `outerRouter` / `basePath` / `staticRouterPath` | `deepLinking` / `selection` / `defaultSelection` / `onSelectionChange` | No `react-router`. |
| `hideTryIt` / `hideTryItPanel` | `hideTryIt` | Two flags collapsed to one; we do not use the distinction. |
| `hideSchemas` / `hideServerInfo` / `hideSecurityInfo` | same | Kept. |
| `hideSamples` | (samples always available, per-panel) | Not a global toggle we use. |
| `hideExport` | *(removed)* | We set `hideExport` today; export is a non-goal ([§01](./01-motivation-and-goals.md#15-non-goals-yagni)). |
| `hideInternal` | *(deferred)* | `x-internal` filtering; add only if ESI needs it. |
| `tryItCredentialsPolicy` / `tryItCorsProxy` | `fetcher` | One transport seam subsumes both — the host controls the fetch. |
| `logo` | slot override / `Overview` | Branding via slots, not a prop. |
| `maxRefDepth` | (parser concern) | Handled by the model's cycle tracking. |

Net effect: a **smaller, clearer** surface — the dead props are gone, the fragile ones
(`renderExtensionAddon`, DOM auth, router) become first-class, typed inputs.

## 4.8 Labels / i18n dictionary {#labels}

`labels?: DeepPartial<ExplorerLabels>` — a typed nested dictionary of every user-facing string, with
complete English defaults (`defaultLabels`, itself a public export). The host maps its own i18n
(`next-intl` today) into this object *before* rendering, so the component needs no i18n provider and
stays island-safe (pain point **P7**). The ESI adapter builds this from the existing `api-spec.*` keys
in en.json.

Decisions from review ([notes A13](./notes/01-architect-round1.md)):

- **Static strings are strings; dynamic strings are functions.** Interpolation uses type-safe
  functions rather than a template engine, e.g. `requiredScopes: (count: number) => string`. Zero
  dependencies, fully typed, trivial to override.
- **Deep-partial merge** over `defaultLabels`, so a host overrides only the keys it cares about; any
  missing key falls back to English.
- **English-only in v1, no bundled translations** (YAGNI). Translation is the host's job via `labels`.
- `defaultLabels` is part of the public API — adding a *required* label key is a breaking change
  ([11](./11-open-source-and-api-stability.md#model-version)).
