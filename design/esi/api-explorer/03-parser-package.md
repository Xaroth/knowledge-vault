---
type: Design Document
title: "Parser Package — openapi-model"
description: "The parser/normalizer: pipeline, normalized data model, 3.0/3.1 handling, ref/extension recovery, diagnostics."
tags: [design, esi, api-explorer, openapi, parser, normalized-model, openapi-3-1, swagger-parser]
timestamp: 2026-07-18T19:00:46Z
---

# 3. Parser Package — `@eve-online-tools/openapi-model`

A framework-agnostic package that turns an OpenAPI 3.0/3.1 document into a stable, UI-friendly
**normalized model**. No React, no DOM. This is a hardened generalisation of the POC's
`@xaroth/openapi-parser`.

## 3.1 Responsibilities

- Accept a spec as a URL, a JSON/YAML string, or an object.
- Dereference `$ref`s (optionally validating the document).
- Reject non-3.x specs with an actionable message.
- Normalize into a `NormalizedSpec` that the renderer can consume directly — with `$ref` identity and
  vendor extensions preserved.
- Do all of the above safely (SSRF-hardened fetch policy for untrusted inputs).

It does **not** render anything, know about ESI, or manage caching/data-fetching lifecycles.

## 3.2 Pipeline

`loadSpec(input, options): Promise<LoadSpecResult>`

```
input (URL | string | object)
  │
  ├─ 1. resolve-input        fetch URL / parse JSON|YAML / accept object
  │                          (fetch gated by load policy)
  ├─ 2. snapshot             structuredClone of paths + components.schemas
  │                          BEFORE dereference (preserves $ref targets + x-*)
  ├─ 3. ownership guard      clone caller-owned objects; mutate parser-owned in place
  ├─ 4. dereference          SwaggerParser.dereference (default)
  │        or validate       SwaggerParser.validate      (options.validate === true)
  ├─ 5. version gate         require openapi 3.x; reject 2.x with guidance
  └─ 6. normalize            build NormalizedSpec (uses the snapshot from step 2)
        │
        ▼
  { raw: OpenAPIDocument, normalized: NormalizedSpec }
```

### Step 2 is the key idea: snapshot-before-dereference

swagger-parser fully inlines `$ref`s, which destroys two things the UI needs: **ref identity** (so a
property typed as `Character` can link to the `Character` model page) and **vendor extensions on
referenced nodes**. Before dereferencing, the parser deep-clones `paths` and `components.schemas`.
The normalizer later compares each dereferenced node against its pre-dereference original and
**re-attaches `$ref`** where the original was a reference, and recovers `x-*` from the original.

This is the POC's central architectural cleverness
(`original-spec-snapshot.ts`,
`normalize-schema.ts`),
and it is worth preserving verbatim.

### Contrast with Stoplight

Elements normalizes to an intermediate representation built on `@stoplight/types`
(`IHttpService` / `IHttpOperation`, produced by `transformOasToServiceNode`). That IR is powerful but
opaque, tied to Stoplight's ecosystem, and the reason vendor extensions needed a patch to surface. Our
`NormalizedSpec` is a small model we own and can extend without patching anyone.

## 3.3 The normalized data model

**This model is the project's core public contract.** The renderer, every slot/extension override,
and every third-party consumer depend on its shape, so it is versioned and treated as a stability
surface ([11](./11-open-source-and-api-stability.md#model-version)): any change to these types is a
**breaking** (major) change, and `NormalizedSpec` carries a `modelVersion` so a mismatched
pre-built model is detected rather than crashing the renderer.

(Shapes follow the POC's
`types.ts`; extension
notes for 3.1 are called out in [§3.5](#versions).)

```ts
interface NormalizedSpec {
  modelVersion: number                                    // bumped on any breaking shape change
  info: NormalizedInfo                                    // title, version, description?
  servers: NormalizedServer[]                             // url (variables resolved), description?
  tags: NormalizedTag[]                                   // name, description?
  operations: NormalizedOperation[]
  operationsById: ReadonlyMap<string, NormalizedOperation> // O(1) lookup, dedup-suffixed ids
  schemas: Record<string, NormalizedSchema>               // named component schemas
  securitySchemes: Record<string, NormalizedSecurityScheme>
}

interface NormalizedOperation {
  id: string                    // operationId, or generated `${method}_${path}`
  method: HttpMethod
  path: string
  summary?: string
  description?: string          // markdown
  tags: string[]
  parameters: NormalizedParameter[]  // path+operation params merged, path→op precedence
  requestBody?: NormalizedRequestBody
  responses: NormalizedResponse[]
  requiresAuth: boolean
  security: NormalizedSecurityRequirement[] // effective, after inheriting global security
  extensions: Record<string, unknown>       // x-* not consumed by the model
}

interface NormalizedSchema {
  type?: string | string[]      // string[] supports OpenAPI 3.1 type arrays
  format?: string
  title?: string
  description?: string
  required?: string[]
  properties?: Record<string, NormalizedSchema>
  items?: NormalizedSchema                    // arrays
  additionalProperties?: boolean | NormalizedSchema // maps
  enum?: unknown[]
  enumDescriptions?: string[]   // parallel to enum (from x-enum-descriptions)
  const?: unknown               // OpenAPI 3.1
  default?: unknown
  example?: unknown             // spec-provided; preferred over synthesized (§5.2)
  nullable?: boolean            // normalized from 3.0 nullable AND 3.1 null-union
  readOnly?: boolean            // badged; filtered out of Try It request bodies
  writeOnly?: boolean           // badged; filtered out of response views
  deprecated?: boolean
  $ref?: string                 // re-attached ref identity, e.g. #/components/schemas/Character
  circular?: boolean            // set on a cyclic ref; children NOT expanded (§3.10)
  oneOf?: NormalizedSchema[]
  anyOf?: NormalizedSchema[]
  allOf?: NormalizedSchema[]    // preserved; also merged into properties/required (§3.10)
  discriminator?: { propertyName: string; mapping?: Record<string, string> }
  not?: NormalizedSchema
  commonModel?: boolean         // ESI x-common-model named-scalar-alias convention
  invalid?: boolean             // node could not be normalized; raw kept, diagnostic emitted (§3.11)
  extensions: Record<string, unknown>
}
```

Design properties worth keeping:

- **`operationsById` is a `ReadonlyMap`** with deterministic de-duplication of colliding ids — safe,
  O(1) lookups for deep linking.
- **Parameters are pre-merged** (path + operation, with correct precedence) so the renderer never
  re-implements OpenAPI merge rules.
- **`extensions` carries every unconsumed `x-*`**, so the renderer's extension registry can render
  anything the spec author (or a pre-processing middleware) attaches — no parser change needed to
  support a new extension. This is the Open/Closed principle applied to vendor extensions.

## 3.4 Public API

Mirrors the POC's `index.ts`:

```ts
// Loading / normalizing
loadSpec(input, options?): Promise<LoadSpecResult>
normalizeSpec(rawDoc): NormalizedSpec        // if you already have a dereferenced doc
resolveInput(input, policy?): Promise<object>

// Helpers
refName(ref): string                          // "#/components/schemas/Foo" -> "Foo"
componentSchemaName(schema): string | undefined
isNormalizedSpec(x): x is NormalizedSpec      // lets the renderer accept either raw or normalized
visitNormalizedSchema(schema, visitor)        // cycle-tracking walker (examples, search indexing)

// Load policy (see §3.6)
assertFetchUrlAllowed(url, policy)
isBlockedFetchHost(host, policy)
type TrustMode = 'trusted' | 'untrusted'
```

`isNormalizedSpec` is what lets `<ApiExplorer>` accept **either** a raw spec (which it parses) **or** a
pre-built model (which it renders directly) — the Dependency Inversion seam in practice.

## 3.5 OpenAPI 3.0 vs 3.1 {#versions}

The POC accepts any `3.x` but types against `OpenAPIV3` (3.0) and does **not** handle 3.1-only
constructs — a real gap, because **the ESI spec is OpenAPI 3.1.0**. This package closes that gap. It
is the one place where we knowingly *extend* beyond the POC rather than port it.

Normalization must handle both dialects and converge them into the single `NormalizedSchema` shape:

| Construct | OpenAPI 3.0 | OpenAPI 3.1 | Normalized to |
| --- | --- | --- | --- |
| Nullability | `nullable: true` | `type: ["string","null"]` | `nullable: true` (+ `type` without `"null"`) |
| Multiple types | not allowed | `type: ["string","integer"]` | `type: string[]` |
| Constants | `enum: [x]` (single) | `const: x` | `const` (renderer shows single fixed value) |
| Reusable schemas | `components.schemas` | `components.schemas` **and** `$defs` | both resolved into `schemas` |
| Examples | `example` | `example` + `examples` | both surfaced (examples are a POC gap) |
| Exclusive bounds | `exclusiveMinimum: bool` | `exclusiveMinimum: number` | numeric range metadata |

The version is read from the document's `openapi` field; both `3.0.x` and `3.1.x` are accepted, `2.x`
is rejected with "Convert the spec to OpenAPI 3.x", and anything else is rejected as unsupported.
Fixtures for **both** dialects are part of the parser's test suite (the POC already ships an ESI 3.1
fixture and several smaller ones).

## 3.6 Load policy & security {#security}

Ported from the POC's `load-policy.ts`.
`trustMode: 'trusted' | 'untrusted'`:

- **trusted** (our own ESI host): normal fetching and external `$ref` resolution.
- **untrusted** (arbitrary user-supplied URL): blocks fetches to loopback, `.local`/`.internal`,
  IPv6 loopback/link-local/ULA, and private/reserved IPv4 ranges; supports a hostname allowlist; and
  disables external `$ref` resolution.

This matters because a public "paste your spec URL" affordance is an SSRF vector. Even if we do not
ship that on day one, the policy costs little to keep and prevents a whole bug class. The renderer
surfaces it as a Try It guard too (see
[06-authentication-and-try-it.md](./06-authentication-and-try-it.md)).

## 3.7 Input formats

- **JSON** — object, or string (fetched or parsed).
- **YAML** — swagger-parser handles YAML natively; the package should accept it (a small gain over
  the POC, whose input resolver is JSON-only). ESI serves JSON, so this is low priority but cheap.
- **Object** — passed through; caller-owned objects are cloned before any mutation (immutability
  guarantee), parser-owned inputs are mutated in place for efficiency.

## 3.8 The Buffer wart {#buffer}

swagger-parser expects Node's `Buffer`. In the browser this needs a polyfill
(`globalThis.Buffer = Buffer`), which the POC does in its playground entry. Two ways to avoid pushing
this onto every host:

1. **Parse on the server** (preferred here) — Next/Astro can run `loadSpec` server-side, so the
   browser never touches swagger-parser; only the plain-JSON `NormalizedSpec` is sent to the client.
2. If browser parsing is required, the package documents the one-line polyfill.

Because our SSR model already favours server-side parsing ([§2.8](./02-architecture.md#ssr--hydration-model)),
option 1 is the default and the wart effectively disappears for hosts.

## 3.10 Hard schema cases (the decisions that make the model good) {#cycles}

These are the cases the review flagged as under-specified ([notes A2](./notes/01-architect-round1.md)).
Each has a settled decision:

- **Cyclic / recursive schemas.** Normalization tracks visited component schemas by ref (extending
  the existing `visitNormalizedSchema` seen-set). On re-encountering one, it emits a node with `$ref`
  set and **`circular: true`**, and does **not** expand its children. The renderer shows a link to the
  model page and never recurses — an unbounded recursive schema can never hang the UI.
- **`allOf` — merge *and* preserve.** The normalizer emits merged `properties`/`required` (most
  `allOf` is "base + extension" and users want the combined object) **and** keeps the original members
  under `allOf` for slots that want them. On a genuine conflict (two members disagree on a property's
  type/constraints) the first wins, the conflicting member is dropped from the merged view, and an
  **`error`-level diagnostic** is emitted ([notes A2 final](./notes/03-architect-round2.md#a2-final))
  — a conflicting `allOf` is usually a real spec bug and we surface it rather than hide it.
- **`discriminator`.** Carried on the node so the variant selector labels branches by discriminator
  value (and `mapping`) instead of "Variant 1/2/3".
- **`readOnly` / `writeOnly` / `deprecated`.** Carried as flags; the renderer badges them and uses
  them to filter Try It request bodies (drop `readOnly`) and response views (drop `writeOnly`).
- **`not`, response headers, parameter `content` (vs `schema`), `example`/`examples`.** All carried
  into the model rather than dropped — this also closes the POC's "examples ignored" gap by preferring
  the spec's own examples and synthesizing only as a fallback.

## 3.11 Diagnostics & resilience {#diagnostics}

The parser **never throws on a single bad node** — essential for an open-source tool fed arbitrary
specs ([notes A7](./notes/01-architect-round1.md)). It returns diagnostics alongside the model:

```ts
interface Diagnostic { level: 'warn' | 'error'; message: string; pointer?: string /* JSON pointer */ }
interface LoadSpecResult { raw: object; normalized: NormalizedSpec; diagnostics: Diagnostic[] }
```

A node that cannot be normalized degrades to `{ invalid: true, raw }` and records a diagnostic, so the
rest of the document still renders. Only whole-document faults (not JSON/YAML, not OpenAPI 3.x) throw,
with an actionable message. The renderer's error-boundary strategy pairs with this — see
[09-quality-and-resilience.md](./09-quality-and-resilience.md#error-handling).

## 3.12 Build & distribution

Following the POC: dual ESM+CJS with `.d.ts` (tsup), strict TypeScript (ES2022, bundler resolution),
Vitest. The normalized model and helper types are the primary export; the ESI fixture (if kept for
tests) is a separate subpath export, never in the main bundle. `MODEL_VERSION` is exported so
consumers can assert compatibility ([11](./11-open-source-and-api-stability.md#model-version)).
