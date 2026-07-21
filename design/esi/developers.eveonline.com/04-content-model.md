---
type: Design Document
title: "Content Model & Navigation"
description: "The three-way content taxonomy and the ownership of navigation between the two repos."
tags: [design, esi, developer-portal, content, contentful]
timestamp: 2026-07-21T12:20:44Z
---

# 4. Content Model & Navigation

## 4.1 The three-way content taxonomy

Every piece of content on the portal is classified by *what it is*, which decides *where it lives* and
*who edits it*:

| Class | Examples | Source of truth | Editor | Visibility |
| --- | --- | --- | --- | --- |
| **Docs** | API guides, service references, community-tool listings | Public repo (Markdown/MDX) | Community + CCP, via PR | Public |
| **Editorial / marketing** | Footer, hero copy, license agreement, blog posts, announcements | **Contentful** | Non-dev editors | Private (licensed assets OK) |
| **Functional UI / logic** | Buttons, forms, nav behaviour, auth, app-management flows | Private repo code | Engineers | Private |

The rule of thumb: **if it is changeable prose or imagery, it belongs in Contentful; if it is
behaviour, it is code; if it is community documentation, it is the public repo.** A "log in" button is
code; the footer and the license agreement are Contentful.

## 4.2 Contentful — the primary CMS

Contentful remains the primary CMS for all editorial/marketing content. It was never the source of the
split-stack pain, and it gives non-developers an editorial workflow plus a **private** home for
licensed blog and marketing imagery.

- Astro pulls Contentful content **at build time**; a Contentful publish webhook triggers a rebuild
  (same trigger family as the docs content — see
  [02](./02-two-repo-architecture.md#24-the-content-seam)).
- Rich-text renders through the design system's components (replacing the current
  `@contentful/rich-text-react-renderer` wiring), so editorial content is styled identically to the
  rest of the site.
- If localized editorial content is ever needed, Contentful's native localization covers it without
  the site adopting locale routing ([09](./09-search-i18n-and-cross-cutting.md)).

## 4.3 Navigation ownership

Navigation is split to preserve transparent content updates:

- **The private repo owns the global/top-level navigation** — the shell and the site map: `/`,
  `/blog`, `/api-explorer`, `/status`, `/applications`, `/authorized-apps`, `/static-data`, `/docs`,
  and the license/legal links. This is functional structure and lives in code (with labels sourced
  from Contentful where they are editorial).
- **The public repo owns the internal structure of `/docs` only.** Within the docs section, the
  community defines its own sub-structure and sub-navigation.

### Docs sub-navigation is content-derived

Inside `/docs`, the sub-nav is **derived from the public repo's directory tree + per-file
frontmatter** (title, order, section-index), matching today's MkDocs Material auto-nav behaviour and
contributors' existing mental model ("drop a file in, it appears"). This is what makes
[G3 transparent updates](./01-motivation-and-goals.md#13-goals) work: a new doc changes the sub-nav
with no private-repo edit. The private `/docs` layout renders that derived tree inside the styled
shell.

```
Global nav (private, code + Contentful labels)
└─ /docs  ──► sub-nav derived from PUBLIC repo (dirs + frontmatter)
   ├─ guides/…
   ├─ services/{esi,static-data,sso,image-server,iec}/…
   ├─ community/…            (community-tool listings)
   └─ resources/…
```

## 4.4 Migration of today's leaked content

Content currently mislocated in the public `mkdocs.yml` is relocated by class: theme/fonts/analytics
and `overrides/` → private repo; hero/footer chrome and social links → **Contentful**; the docs
content itself stays in the public repo. After migration the public repo holds content only.
