---
type: Design Document
title: "Two-Repo Architecture"
description: "The public/private split, the content seam, build-time clone, and the rebuild trigger."
tags: [design, esi, developer-portal, astro, architecture]
timestamp: 2026-07-21T12:20:44Z
---

# 2. Two-Repo Architecture

## 2.1 Overview

The portal is one Astro application split across two repositories plus a shared authoring package.
The split is by **visibility**, not by technology.

```
┌───────────────────────────────────────────────────────────────────────────┐
│  PUBLIC repo (esi-docs successor)        vanilla, unstyled Astro           │
│  ├─ src/content/docs/**        community-authored Markdown/MDX              │
│  ├─ docs internal structure    (directory + frontmatter → /docs sub-nav)   │
│  └─ minimal astro config       standalone-previewable, NO infra/licensing  │
└───────────────────────────────────────────────────────────────────────────┘
                   │  build-time clone (content only, unpinned)
                   ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  PRIVATE repo                            the real, styled site             │
│  ├─ global shell + top-level nav (/, /blog, /api-explorer, /status, …)     │
│  ├─ every non-docs surface (see 08-surfaces)                               │
│  ├─ theme, licensed fonts/imagery, design tokens                           │
│  ├─ auth (OAuth + session) + authed proxy routes → Pulsar                  │
│  ├─ infrastructure / deploy pipeline                                       │
│  └─ renders cloned /docs content inside the styled shell                   │
└───────────────────────────────────────────────────────────────────────────┘
                   ▲
                   │  npm dependency (both repos)
┌───────────────────────────────────────────────────────────────────────────┐
│  AUTHORING KIT (published open package)                                    │
│  └─ shared remark/rehype plugins + components (callout, tabs, code-group…) │
└───────────────────────────────────────────────────────────────────────────┘
```

## 2.2 The public repo

A **vanilla Astro** project. It owns only the `/docs` subtree: the Markdown/MDX content and its
internal structure. It depends on the shared **authoring kit** so contributors can `astro dev` and
preview their pages with baseline (unstyled) rendering of every supported construct. It contains no
theme, no licensed assets, no analytics, and nothing hosting-related. Contributors experience a small,
legible repository whose only concern is content.

## 2.3 The private repo

The full site. It owns the global shell and top-level navigation, every non-docs surface, the design
system and licensed assets, authentication, and all infrastructure. At build time it pulls the public
content in and renders it through its own styled `/docs` layout, applying the design-token overrides
from [07-design-system-and-theming.md](./07-design-system-and-theming.md).

## 2.4 The content seam

The seam is **one-directional**: content flows public → private; nothing flows back, and the public
repo never learns how or where the site is hosted.

### Ingest — build-time clone

The private build **clones the public repo's content** into its content directory before
`astro build`. Content is treated as **fluid**:

- It is **not pinned** to a commit in the private repo. There is no submodule bump, no version bump,
  no release step per doc change.
- **Functionality is versioned; content is not.** The private repo owns the versioned functionality;
  the public repo is a continuously-updated content source.
- Because the sub-nav within `/docs` is derived from the cloned content's directory + frontmatter
  (see [04-content-model.md](./04-content-model.md)), a new or moved doc appears on the next build
  with **no private change**.

> Design consequence: the private build must record the cloned content's commit SHA in its build
> metadata/logs, so "what content shipped" is answerable even though it is not pinned in history.

### Trigger — rebuild on content change *(open)*

A merge to the public repo must cause the private site to rebuild and redeploy, **without the public
repo containing any hosting knowledge**. The exact mechanism is downstream of the deploy pipeline
([03](./03-rendering-and-hosting.md)) and is left open ([11](./11-open-decisions.md)). The candidates,
all of which keep the public tree pristine:

- A **webhook configured in the public repo's *settings*** (not in the tracked tree) that POSTs to a
  private build hook.
- A **`repository_dispatch`** from a minimal public workflow to the private repo (accepted only if the
  small amount of coupling it puts in the public tree is judged acceptable).
- A low-frequency **scheduled rebuild** as a safety net so a missed event self-heals.

The leading direction is a settings-webhook or dispatch, chosen once the pipeline is chosen.

## 2.5 The authoring kit

A small **published, open package** consumed by both repos. It defines the closed set of Markdown
directives and components (callout, tabs, code-group, mermaid, …) as shared remark/rehype plugins plus
components, with **vanilla defaults** so the public repo builds and previews standalone. The private
site restyles them via the design-token contract. Details in
[05-docs-authoring-contract.md](./05-docs-authoring-contract.md). It carries no licensed assets, so it
can be public.

## 2.6 What this buys us

- Licensed assets and infrastructure are structurally impossible to leak into the public repo.
- Contributors get a tiny, content-only repository they can run locally.
- Docs ship continuously; the private repo is disturbed only when *functionality* changes.
- One language, one framework, one design system across the whole portal.
