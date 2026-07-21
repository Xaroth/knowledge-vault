---
type: Design Document
title: "Search, i18n & Cross-cutting Concerns"
description: "Pagefind search, string externalization, static-data serving, and other cross-cutting concerns."
tags: [design, esi, developer-portal, search, i18n]
timestamp: 2026-07-21T12:20:44Z
---

# 9. Search, i18n & Cross-cutting Concerns

## 9.1 Search — Pagefind

MkDocs Material gave docs search for free; Astro static output does not. The portal uses
**Pagefind**:

- A **static search index** is built at deploy time and runs **entirely client-side** — no service,
  no API keys, nothing to operate. This fits the host-agnostic goal
  ([03](./03-rendering-and-hosting.md)) and the fluid-content model (the index is rebuilt on every
  deploy, so new docs are searchable as soon as they ship).
- Search covers **docs**, and can extend to **blog and marketing** content.
- The search UI is exposed through the natively-rebuilt **command palette**
  ([07.4](./07-design-system-and-theming.md#74-interactivity--react-islands-only-where-needed)).

Hosted search (e.g. Algolia DocSearch) was rejected to avoid another vendor and operational surface.

## 9.2 Internationalization — string externalization, en-only

The portal is **English-only**, matching today's reality (`next-intl` configured with a single `en`
locale, `localePrefix: 'never'`).

- **String externalization is retained**: functional UI strings live in a message catalog, so the
  site is i18n-*ready*.
- **No multi-locale routing** is built — that is speculative scope
  ([NG4](./01-motivation-and-goals.md#14-non-goals)).
- Editorial localization, if ever needed, is Contentful's job
  ([04.2](./04-content-model.md#42-contentful--the-primary-cms)); real Astro i18n routing can be added
  later if a concrete multi-language need appears.

## 9.3 Static-data serving

Static-data artifacts already live in **S3** (not in any repo) and are produced by an external
pipeline ([NG5](./01-motivation-and-goals.md#14-non-goals)). The current mechanism — an Amplify
rewrite to the S3 bucket plus a route that redirects the `latest/` URLs — is preserved in spirit:

- a **host-level rewrite** points the static-data paths at the S3 bucket, and
- a **thin server route** resolves the `latest` URLs to the current versioned objects.

Download URLs and the static-data feed stay stable.

## 9.4 Testing & tooling

One unified TypeScript toolchain across both repos (replacing the split React + Python toolchains):

- **Unit** tests for the parser/transform/authoring-kit logic and server routes.
- **End-to-end** tests (Playwright, carried over) across the key surfaces, including an authed flow.
- **Content validation** for the docs frontmatter contract
  ([05.5](./05-docs-authoring-contract.md#55-frontmatter-contract)), failing the build with
  contributor-facing messages.
- Shared lint/format config; the **authoring kit** and **design system** each have their own CI.

## 9.5 Accessibility & performance

The native design system is built to the accessibility bar Mantine provided (focus management,
semantics, contrast) as an explicit acceptance criterion, not an afterthought. Static-by-default
rendering plus fingerprinted CDN assets keep the portal fast; islands are hydrated selectively.
