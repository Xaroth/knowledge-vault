---
type: Design Document
title: "Quality Attributes & Resilience"
description: "Accessibility, performance, error handling/diagnostics, and security requirements with acceptance criteria."
tags: [design, esi, api-explorer, openapi, accessibility, performance, security, resilience]
timestamp: 2026-07-18T19:00:46Z
---

# 9. Quality Attributes & Resilience

Cross-cutting requirements that apply to the whole renderer, with concrete acceptance criteria. These
came out of the review rounds ([notes/](./notes/README.md)) and are non-optional: for an open-source component,
these are what separate "renders our spec" from "trustworthy library."

## 9.1 Accessibility {#accessibility}

An API reference is a reading-and-navigation tool; accessibility is core function, not polish.

### Required ARIA patterns

| UI element | Pattern | Key requirements |
| --- | --- | --- |
| Navigation tree | [WAI-ARIA Tree View](https://www.w3.org/WAI/ARIA/apg/patterns/treeview/) | `role="tree"`/`treeitem`/`group`; roving `tabindex`; `↑`/`↓`/`←`/`→`/`Home`/`End`; `aria-expanded` on groups. |
| Response / body / sample tabs | [Tabs](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/) | `role="tablist"`/`tab`/`tabpanel`; arrow-key selection; `aria-selected`. |
| Collapsible panels | [Disclosure](https://www.w3.org/WAI/ARIA/apg/patterns/disclosure/) | Button toggles, `aria-expanded`, `aria-controls`. |
| Variant / enum / server selects | native `<select>` or [Listbox](https://www.w3.org/WAI/ARIA/apg/patterns/listbox/) | Prefer native `<select>` (free a11y) unless styling forces otherwise. |
| Copy buttons | button + live region | `aria-label`; announce "copied" via `aria-live="polite"`. |

### Behavioural requirements

- **Focus management:** on selection change (nav → view), move focus to the view's heading (with
  `tabindex=-1`) so keyboard/screen-reader users land on the new content.
- **Visible focus:** a `:focus-visible` ring on every interactive element, themable via a token
  (`--oae-color-accent`). Never `outline: none` without a replacement.
- **Motion:** honour `prefers-reduced-motion` — collapse/expand animations become instant.
- **Contrast:** both shipped themes (light, dark) meet WCAG 2.1 AA (4.5:1 text, 3:1 UI) — verified in
  tests ([10](./10-testing-and-tooling.md)).
- **Landmarks:** the component root is a `<section aria-label>`; nav is `<nav>`; the view is `<main>`
  *only if* the component owns the page region (configurable, to avoid duplicate `main` landmarks when
  embedded).

### Acceptance
Automated `axe` checks pass on every view type with zero violations; the full flow
(open → navigate → expand schema → Try It) is operable by keyboard alone; screen-reader smoke-tested
on at least one AT.

## 9.2 Performance & scale {#performance}

Target spec: ESI at 203 operations / 313 schemas, with deep recursive schemas.

### The model

- **Render one view at a time.** Only the selected operation/schema/overview is mounted. This is the
  primary performance decision and the reason the component stays cheap on large specs.
- **Normalize once.** Parsing/normalization is memoized (or done server-side); it never re-runs on
  re-render. The `NormalizedSpec` is treated as immutable.
- **Bounded, cycle-guarded schema expansion.** Schemas expand to `defaultExpandedDepth` (default: a
  small number, e.g. 2) with an explicit "expand" affordance deeper. Cycle detection
  ([03](./03-parser-package.md#cycles)) guarantees recursion terminates — an unbounded recursive
  schema must never hang the UI.
- **Memoized subtrees.** `SchemaView` rows and rendered markdown are `React.memo`'d; the Try It store
  uses per-key `useSyncExternalStore` subscriptions so a keystroke re-renders one field, not the panel.
- **Response cap** ([E5](./notes/02-engineer-round1.md#e5)): above a size threshold (e.g. ~256 KB),
  skip pretty-print + syntax highlighting and offer "show raw" — highlighting a multi-MB body is a
  real jank source.

### No built-in virtualization (v1) {#no-virtualization}
Deliberate ([A6 ruling](./notes/03-architect-round2.md#a6-final)). 203 nav rows + one view is not a
performance problem, and windowing fights find-in-page and a11y. A host with a pathological spec can
supply a virtualized `NavigationTree` via the slot registry. Revisit only with evidence.

### Perf budget (testable)
On the ESI spec, on mid-range hardware: initial mount < ~200 ms after the model is available;
navigation between operations < ~50 ms interaction-to-paint; no dropped frames expanding a typical
schema. These numbers are the threshold that makes "measure first" meaningful; tune once measured.

## 9.3 Error handling, diagnostics & resilience {#error-handling}

Open-source users will feed this every malformed and exotic spec that exists. The component degrades;
it does not blank out or crash.

### Parser diagnostics
`loadSpec`/`normalizeSpec` never throw on a single bad node. They return diagnostics alongside the
model ([03](./03-parser-package.md#diagnostics)):

```ts
interface Diagnostic { level: 'warn' | 'error'; message: string; pointer?: string /* JSON pointer */ }
interface LoadSpecResult { raw; normalized: NormalizedSpec; diagnostics: Diagnostic[] }
```

A node that can't be normalized degrades to `{ invalid: true, raw }` and records a diagnostic, rather
than aborting the parse. Fatal, whole-document failures (not JSON/YAML, not OpenAPI 3.x) still throw,
with an actionable message.

### Render resilience
- **Per-view error boundary:** a thrown render error in one operation/schema view shows a compact
  fallback ("This operation couldn't be rendered") with the error in development, and leaves the rest
  of the app usable. (Elements itself wraps components in `withErrorBoundary`; we do the same,
  deliberately.)
- **Per-row resilience** in `SchemaView`: a single malformed property renders an "unparseable" row,
  not a broken subtree.
- **Diagnostics surface:** the component can optionally render a dismissible diagnostics summary (off
  by default) so authors see spec problems; hosts can also read `diagnostics` and handle them.

### UX state contract
Explicit, documented states: `loading` (parsing/fetching) · `ready` · `empty` (valid spec, no
operations) · `error` (fatal). Each has a default rendering and a slot/override.

## 9.4 Security {#security}

The everyday threat is not SSRF (covered in [03](./03-parser-package.md#security)) but **untrusted
spec content**: descriptions, examples, URLs, and `x-*` are attacker-controlled whenever a user loads
an arbitrary spec. These are hard rules, enforced in code review and (where possible) lint.

- **No raw HTML from spec content.** Markdown renders with `react-markdown` and **`rehype-raw` is
  never added**. No `dangerouslySetInnerHTML` of any spec-derived string, anywhere. An ESLint rule
  bans `dangerouslySetInnerHTML` in the renderer package.
- **URL sanitization.** Every spec-derived URL (server URLs, external docs, OAuth URLs, links in
  markdown) passes an allowlist sanitizer (`http`, `https`, `mailto` only; anything else is dropped or
  rendered as inert text). External links get `rel="noopener noreferrer"` and `target` is set
  intentionally.
- **Examples as text.** Example/response bodies are rendered as text (highlighted), never interpreted
  as markup.
- **CORS / Try It safety** ([06](./06-authentication-and-try-it.md#cors)): a Try It request that fails
  CORS produces an actionable message, not a silent failure; the `untrusted` trust mode + `fetcher`
  seam bound what a "paste a spec" deployment can reach.
- **No secret persistence by default.** Try It auth values live in memory; persisting a token is a
  host decision made explicitly through the `auth` prop, not something the component does behind the
  user's back.

### Acceptance
A test loads a spec containing `<script>` / `javascript:` payloads in descriptions, titles, examples,
and server URLs, and asserts none execute and none render as live markup.
