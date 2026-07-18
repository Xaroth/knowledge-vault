---
type: Design Document
title: "Feature Set"
description: "Feature parity matrix against @stoplight/elements and the POC, gaps closed, and deferred features."
tags: [design, esi, api-explorer, openapi, feature-parity, yagni]
timestamp: 2026-07-18T19:00:46Z
---

# 5. Feature Set

The target is **parity or better** against two baselines: what we actually use from
`@stoplight/elements` today, and the ESI Explorer POC. This document is the checklist that defines
"done" for the renderer, and it is explicit about what we deliberately defer (YAGNI).

## 5.1 Parity matrix

Legend: ✅ present · ⚠️ partial/stubbed · ❌ absent · ➖ not applicable / non-goal.

| Capability | Elements (as we use it) | ESI Explorer POC | **Target** |
| --- | --- | --- | --- |
| **Rendering & model** | | | |
| OpenAPI 3.0 | ✅ | ✅ | ✅ |
| OpenAPI 3.1 (type arrays, `const`, `$defs`, null unions) | ✅ | ⚠️ accepts 3.x, no 3.1 constructs | ✅ |
| YAML input | ✅ | ❌ JSON only | ✅ (cheap via swagger-parser) |
| Pre-normalized model bypass | ❌ | ✅ | ✅ |
| **Navigation** | | | |
| Tag-grouped operation list | ✅ | ✅ | ✅ |
| Schemas / models section | ✅ | ✅ | ✅ |
| Collapsible sections (persisted) | ✅ | ✅ | ✅ |
| **Search / filter** | ✅ (via our SpotlightSearch) | ❌ none | ✅ built-in |
| Deep linking (URL hash) | ✅ (react-router) | ✅ (own, no router) | ✅ (own, no router) |
| **Operation view** | | | |
| Method + path + summary + markdown | ✅ | ✅ | ✅ |
| Parameter table (path/query/header/cookie) | ✅ | ✅ | ✅ |
| Request body (per media type) | ✅ | ✅ | ✅ |
| Responses (per status, tabs) | ✅ | ✅ | ✅ |
| Security / auth summary per op | ✅ | ✅ | ✅ |
| **Schema viewer** | | | |
| Nested objects / arrays / maps | ✅ | ✅ | ✅ |
| `oneOf`/`anyOf`/`allOf` variant selector | ✅ | ✅ | ✅ |
| Clickable model cross-links | ⚠️ | ✅ | ✅ |
| Enum values + `x-enum-descriptions` | ⚠️ (via our patch/addon) | ✅ | ✅ |
| Examples from spec (`example`/`examples`) | ✅ | ❌ always synthesized | ✅ prefer spec, fall back to synth |
| **Try It** | | | |
| Interactive request builder | ✅ | ✅ | ✅ |
| Live request execution | ✅ | ✅ | ✅ |
| Server / server-variable selector | ✅ | ✅ | ✅ |
| Query serialization (form/space/pipe/deepObject, explode) | ✅ | ✅ | ✅ |
| Pre-flight validation (required, `{path}` placeholders) | ⚠️ | ✅ | ✅ |
| **Authentication as a first-class input** | ❌ DOM injection | ⚠️ paste token | ✅ typed `auth` prop + `fetcher` |
| Request transport / proxy hook | ⚠️ `tryItCorsProxy` | ✅ `onRequest` | ✅ single `fetcher` seam |
| SSRF/trust guard on send | ❌ | ✅ | ✅ |
| **Code & responses** | | | |
| Request code samples (multi-language) | ✅ | ✅ (11 languages) | ✅ |
| Syntax highlighting (samples + responses) | ✅ (Prism) | ❌ plain `<pre>` | ✅ |
| Response example generation | ✅ | ✅ | ✅ |
| Markdown descriptions (lazy) | ✅ | ✅ | ✅ |
| **Vendor extensions** | | | |
| Operation / schema `x-*` rendering | ⚠️ addon only | ✅ registry | ✅ registry |
| **Service-level `x-*` rendering** | ❌ needs source patch | ✅ | ✅ (no patch) |
| **Non-functional** | | | |
| Server-side render | ❌ ssr:false | ✅ SSR-safe | ✅ |
| Self-contained styling (no UI-kit dep) | ❌ Mantine/Mosaic bridge | ✅ CSS vars + modules | ✅ |
| Island-safe (no external context) | ❌ | ⚠️ (POC is generic) | ✅ by design |
| Zero patches | ❌ 3 patches | ✅ | ✅ |
| No `react-router` | ❌ | ✅ | ✅ |

## 5.2 Gaps we close relative to the POC

The POC is a strong base but has known gaps. These become explicit work items:

1. **OpenAPI 3.1 constructs** — type arrays, `const`, `$defs`, null unions. **Required**, because the
   ESI spec is 3.1.0. ([03-parser-package.md](./03-parser-package.md#versions))
2. **Search** — the POC has no navigation search; we render one today via `SpotlightSearch`. Ship a
   built-in search box over operations + schemas.
3. **Syntax highlighting** — the POC renders code and responses as plain `<pre>`. Add a lightweight
   highlighter (kept off the critical path, like markdown), so we do not regress against Elements'
   Prism output.
4. **Spec-provided examples** — the POC always synthesizes example bodies from types and ignores the
   spec's own `example`/`examples`. Prefer the spec's examples, synthesize only as fallback.

## 5.3 Gaps we close relative to Elements

1. **First-class auth** replaces DOM injection (**P2**).
2. **Service-level vendor extensions** without a patch (**P1**).
3. **SSR** (**P4**), **no router dependency** (**P5**), **owned styling** (**P3/P6**).

## 5.4 Deferred (YAGNI — listed, not built)

Capabilities we consciously do not build until there is a concrete need. Recorded so reviewers see
they were considered, not missed:

- **Export / "download SDK"** — we set `hideExport` today; not used.
- **`x-internal` filtering (`hideInternal`)** — add only if ESI marks internal operations.
- **Mock server / `MockingProvider`** — Elements ships it; we do not use it.
- **Multi-document catalogue / Markdown "articles" tree** — Elements' `TableOfContents` platform
  features; out of scope ([01-motivation-and-goals.md](./01-motivation-and-goals.md#15-non-goals-yagni)).
- **Built-in OAuth flow** — token acquisition stays in the host; the component consumes a token.
- **Webhooks rendering** — the ESI spec has none today; add if/when it does.
- **Request/response validation of user input against schemas** — beyond required/placeholder checks;
  add only if users ask for it.

### Anti-goal: no plugin framework

Called out explicitly so it survives us ([notes A2.2](./notes/03-architect-round2.md)). Extensibility
is exactly two plain-React mechanisms — slot overrides (`components`) and vendor-extension components
(`extensions`) — and **nothing more**. No plugin lifecycle, no runtime plugin discovery, no
configuration DSL. A future contributor proposing a plugin system must first demonstrate a concrete
need the component model cannot meet; the default answer is no. See
[11](./11-open-source-and-api-stability.md#no-plugin-framework).

## 5.5 Definition of done (renderer)

- Every ✅ in the Target column of [§5.1](#51-parity-matrix) is implemented and tested.
- Both a 3.0 and a 3.1 fixture render correctly (including the live ESI 3.1 spec).
- The four POC gaps in [§5.2](#52-gaps-we-close-relative-to-the-poc) are closed.
- The component renders as a standalone Astro island with no external React context.
