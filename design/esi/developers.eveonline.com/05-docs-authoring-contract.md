---
type: Design Document
title: "Docs Authoring Contract"
description: "The GFM + shared-component authoring vocabulary and the content-transform pipeline shared by both repos."
tags: [design, esi, developer-portal, authoring, markdown]
timestamp: 2026-07-21T12:20:44Z
---

# 5. Docs Authoring Contract

## 5.1 The problem

Community-authored docs must render correctly in **two places**: the *vanilla* public repo preview
(so contributors can check their work with `astro dev`) and the *styled* private site. Today's docs
also rely on rich MkDocs Material constructs (admonitions/callouts, content tabs, tooltips, mermaid)
and on content-generating hooks. The contract must preserve that richness without letting presentation
into the public repo.

## 5.2 The vocabulary — GFM + a closed component set

Contributors write **GitHub-Flavored Markdown** plus a **closed, documented set** of rich constructs:

| Construct | Purpose | Replaces (MkDocs) |
| --- | --- | --- |
| **Callout / admonition** | Notes, warnings, tips | `!!! note` admonitions |
| **Tabs** | Alternative views (languages, platforms) | `content.tabs` |
| **Code group** | Grouped, switchable code samples | tabbed code blocks |
| **Mermaid** | Diagrams | mermaid via CDN |
| **Snippet include** | Reusable content fragments | `generate-snippets.py` |

The set is **closed and versioned** — new constructs are added deliberately, not improvised — so the
authoring surface stays legible and both renderers stay in sync.

## 5.3 Implementation — the shared authoring kit

The vocabulary is implemented once, in the **published open authoring-kit package**
([02.5](./02-two-repo-architecture.md#25-the-authoring-kit)):

- **remark/rehype plugins** parse the directives/components from Markdown/MDX.
- **Components** provide **vanilla, unstyled defaults** so the public repo renders and previews every
  construct standalone.
- The **private site restyles** those components through the design-token contract
  ([07](./07-design-system-and-theming.md)) — same markup, portal-grade presentation.

Because the kit carries no licensed assets and no presentation, it is itself public and depended on by
both repos.

## 5.4 Content-transform hooks

Today's Python hooks are **content transforms**, not presentation, so they port into the shared
content pipeline (as remark/rehype steps or an Astro integration) rather than into private
presentation code:

- **Snippet generation / includes** (`generate-snippets.py`) → a snippet-include directive resolved at
  build from the public repo's `snippets/`.
- **Community-tools listing** (`community-tools.py`) → generated from the `docs/community/**` entries'
  frontmatter, producing the community-tool index as content.

Keeping these in the content pipeline preserves the public repo's standalone preview and the
transparent-update property.

## 5.5 Frontmatter contract

The public docs frontmatter is the interface the private `/docs` layout consumes. The recognised
fields (title, description, order, section-index, tags, and any redirect/alias needed to preserve
current URLs) are documented in the authoring kit and validated at build. Unknown fields are ignored;
missing required fields fail the build with a contributor-facing message.
