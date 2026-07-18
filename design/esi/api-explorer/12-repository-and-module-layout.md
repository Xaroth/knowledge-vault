---
type: Design Document
title: "Repository & Module Layout"
description: "Concrete monorepo and per-package file layout, plus the frozen dependency choices."
tags: [design, esi, api-explorer, openapi, repository-layout, monorepo, dependencies]
timestamp: 2026-07-18T19:00:46Z
---

# 12. Repository & Module Layout

The concrete structure a medior engineer starts from. It mirrors the
[ESI Explorer POC](../../../xaroth/esi-explorer), which already has a clean, proven layout — this is
that structure, generalised and with the review's additions folded in.

## 12.1 Monorepo

```
openapi-explorer/                      # repo root; packages published under @eve-online-tools (see 11 §11.6)
├─ package.json                        # pnpm workspaces + turbo scripts
├─ pnpm-workspace.yaml                 # packages: apps/*, packages/*
├─ turbo.json                          # build/test/lint/typecheck task graph
├─ tsconfig.base.json                  # strict, ES2022, bundler resolution
├─ .changeset/                         # changesets (versioning)
├─ .github/workflows/ci.yml            # format→lint→typecheck→test→build→size-limit
├─ packages/
│  ├─ openapi-model/                   # @eve-online-tools/openapi-model (parser) — no React
│  └─ openapi-explorer/                # @eve-online-tools/openapi-explorer (renderer) — React
└─ apps/
   └─ playground/                      # Vite demo + manual harness + CI smoke test
```

Two packages, versioned independently ([11](./11-open-source-and-api-stability.md#model-version)).
The **ESI adapter is not here** — it lives in this app ([08](./08-esi-integration-and-migration.md)).

## 12.2 `packages/openapi-model` (parser)

```
src/
├─ index.ts                    # PUBLIC barrel (see 11 §11.1)
├─ load-spec.ts                # loadSpec(): orchestrates the pipeline (03 §3.2)
├─ resolve-input.ts            # URL | string(JSON/YAML) | object → object
├─ original-spec-snapshot.ts   # snapshot-before-dereference (the key trick, 03 §3.2)
├─ validate-spec.ts            # SwaggerParser dereference/validate; version gate
├─ load-policy.ts              # TrustMode, SSRF allow/block (03 §3.6)
├─ normalize.ts                # top-level: raw → NormalizedSpec + diagnostics
├─ normalize-operations.ts     # operations, param merge
├─ normalize-schema.ts         # schema tree; 3.0/3.1 convergence; allOf merge; cycles
├─ normalize-security.ts       # securitySchemes, effective security
├─ normalize-extensions.ts     # x-* collection (snapshot-recovered)
├─ diagnostics.ts              # Diagnostic type + collector (09 §9.3)
├─ visit-normalized-schema.ts  # shared cycle-tracking walker
├─ model-version.ts            # MODEL_VERSION constant (11 §11.4)
├─ types.ts                    # NormalizedSpec and all Normalized* (03 §3.3)
└─ __tests__/                  # fixtures/ (corpus, 10 §10.2) + specs
tsup.config.ts                 # dual ESM+CJS + d.ts; ./ + ./fixtures subpath
```

Build: **tsup**. Runtime deps: `@apidevtools/swagger-parser`, `openapi-types`. No React.

## 12.3 `packages/openapi-explorer` (renderer)

```
src/
├─ index.ts                       # PUBLIC barrel
├─ ApiExplorer.tsx                # entry: providers + layout shell (04 §4.2)
├─ context/
│  ├─ explorer-components.tsx     # slot registry + useExplorerComponents (04 §4.4)
│  ├─ extension-components.tsx    # x-* registry (04 §4.5)
│  ├─ try-it-values/              # external store + useSyncExternalStore (06 §6.4)
│  ├─ schema-navigation.tsx       # clickable model cross-links
│  └─ labels.tsx                  # defaultLabels + deep-partial merge (04 §4.8)
├─ selection/                     # Selection type, deep-linking, precedence (04 §4.6)
├─ components/                    # one folder per slot (default impls)
│  ├─ NavigationTree/  Overview/  RouteDetail/  RouteInfo/  RouteSpec/
│  ├─ SchemaView/  SchemaDetail/  SchemaTypeLabel/  ParameterTable/  EnumValues/
│  ├─ TryItPanel/  CodeSamplePanel/  ResponseViewer/
│  ├─ Markdown/                   # LAZY chunk: react-markdown + remark-* (no rehype-raw)
│  ├─ CodeHighlight/              # LAZY chunk: prism-react-renderer
│  ├─ primitives/                 # Collapse, Tabs, Tooltip, CopyButton, Select (token-styled)
│  └─ ErrorBoundary/              # per-view/per-node boundaries (09 §9.3)
├─ try-it/                        # buildRequest, serializeQuery, auth apply, validate, fetcher
├─ request-samples/              # per-language sample generators (registry)
├─ utils/                        # url-sanitize, schema-example, response-example
└─ __tests__/
styles/
├─ index.scss                    # imports tokens + component partials → style.css
├─ _tokens.scss                  # --oae-* defaults (07 §7.3) — PUBLIC
├─ themes/_light.scss _dark.scss # data-theme blocks (07 §7.4)
└─ _mixins.scss                  # focus-ring, truncate, scrollbar, respond-to
vite.config.ts                   # lib mode; entries: index, markdown; externalize react + model
```

Build: **Vite library mode** + `vite-plugin-dts`. Peer dep: `react`/`react-dom` `^18 || ^19`. Dep:
`@eve-online-tools/openapi-model` (caret). Lazy chunks: markdown, code-highlight.

## 12.4 `apps/playground`

Vite app that mounts `<ApiExplorer>` against several specs (ESI fixture as a static asset + a couple
of non-ESI specs to prove genericity). It is the manual test harness, the CI build/typecheck smoke
test, and the first working example an OSS reader will copy ([E6](./notes/02-engineer-round1.md#e6)).
During `vite serve` it aliases the workspace packages to their `src/` (as the POC does) so dev runs
against TS source with no prebuild.

## 12.5 Dependency decisions (frozen) {#dependencies}
Settled in review ([E2](./notes/02-engineer-round1.md#e2)) so they aren't re-litigated per PR:

| Need | Choice | Rationale |
| --- | --- | --- |
| Parse/dereference | `@apidevtools/swagger-parser` + `openapi-types` | Proven in POC; handles JSON+YAML+validate. |
| Markdown | `react-markdown` + `remark-gfm` + `remark-breaks`, **lazy** | Safe by default (no raw HTML); off critical path. |
| Syntax highlighting | `prism-react-renderer`, **lazy** | Runtime, tiny, no wasm — right for a drop-in lib. **Not Shiki** (build-time/wasm/async). |
| UI primitives | hand-built, token-styled | No UI-kit dependency (self-containment, 07). |
| Router / state lib | **none** | Own hash deep-linking + `useSyncExternalStore`. |

Everything else is React + TypeScript. This keeps the install small, the bundle tree-shakeable, and
the component portable across React apps, Astro islands, and plain Vite.
