---
type: Design Overview
title: "developers.eveonline.com Rework — Design Documentation"
description: "Index and executive summary for re-platforming the EVE developer portal onto a two-repo Astro architecture."
tags: [design, esi, developer-portal, astro, overview]
timestamp: 2026-07-21T12:20:44Z
---

# developers.eveonline.com Rework — Design Documentation

This directory describes a **re-platform of the EVE developer portal** (`developers.eveonline.com`)
onto a **single, coherent Astro stack split across two repositories** — one public and
community-contributable, one private and fully styled.

The documents are written in a **finished-state** style: they describe the target portal as if it
already exists, so they can be used as source material for development plans, estimates, and tickets.
They are *design intent*, not an implementation log, per
[describe the present, not the change](../../conventions/describe-current-state.md).

## Why this exists

The portal is served by two unrelated stacks: `developers-next` (Next.js 14 + React + Mantine +
Contentful, on AWS Amplify) and `esi-docs` (Python / MkDocs Material, public, GitHub-Pages-style
hosting). The two share nothing. Every design change is done twice, in two languages and two
toolchains, and licensed material (fonts, imagery) cannot live in the public docs repo. Rather than
maintain two portals, we unify on **Astro** and separate concerns by *repository visibility* instead
of by *technology*.

## The shape of the solution

Two Astro repositories, one shared authoring kit:

| Repo | Owns | Visibility |
| --- | --- | --- |
| **Public** (esi-docs successor) | The `/docs` subtree content and its internal structure. Vanilla, unstyled Astro; standalone-previewable. | Public / community |
| **Private** | The global shell + top-level navigation, every non-docs surface, theme, licensed assets, auth, and all infrastructure. | Private |
| **Authoring kit** (published package) | The closed set of Markdown directives/components both repos render. | Public package |

The private site **build-time clones** the public content and applies its theme. Docs content is
*fluid* — unpinned, unversioned, and updated transparently; only functionality is versioned. The
public repo contains **nothing hosting-related**.

## Reading order

1. **[01-motivation-and-goals.md](./01-motivation-and-goals.md)** — The split-stack pain, evidenced,
   and the goals / non-goals that bound the work (parity is the floor).
2. **[02-two-repo-architecture.md](./02-two-repo-architecture.md)** — The public/private split, the
   content seam, build-time clone, and the rebuild trigger.
3. **[03-rendering-and-hosting.md](./03-rendering-and-hosting.md)** — The hybrid rendering model and
   the host-agnostic AWS deployment artifact.
4. **[04-content-model.md](./04-content-model.md)** — The three-way content taxonomy and navigation
   ownership.
5. **[05-docs-authoring-contract.md](./05-docs-authoring-contract.md)** — The GFM + shared-component
   authoring vocabulary and the content-transform pipeline.
6. **[06-auth-and-backend.md](./06-auth-and-backend.md)** — Astro-owned OAuth + session, and the
   offload of backend logic to Pulsar services.
7. **[07-design-system-and-theming.md](./07-design-system-and-theming.md)** — The Astro-native design
   system and the token contract shared with the API Explorer.
8. **[08-surfaces.md](./08-surfaces.md)** — Per-surface fate across the whole portal.
9. **[09-search-i18n-and-cross-cutting.md](./09-search-i18n-and-cross-cutting.md)** — Pagefind search,
   string externalization, and static-data serving.
10. **[10-migration.md](./10-migration.md)** — The phased, route-by-route cutover.
11. **[11-open-decisions.md](./11-open-decisions.md)** — The decisions deliberately left open, with
    the criteria to resolve them.

## Relationship to the API Explorer plan

The [API Explorer](../api-explorer/README.md) is a **consumed dependency**, not re-designed here. It
is mounted as a React island inside the private site and shares this portal's design-token contract.
Its own plan governs its internals, including the Try-It authentication hook and the `SPEC_VIEWER`
SSO client.

## Guiding principles

- **Separate by visibility, not by technology** — one stack (Astro), split into public and private
  repos so licensed assets and infrastructure stay private while content stays open.
- **Content is fluid; functionality is versioned** — public docs flow into the site continuously
  without a private-repo change.
- **Parity first** — the re-platform reproduces today's features and information architecture; visual
  and IA redesign is a separable, gated upside, never on the critical path.
- **YAGNI** — we shed moving parts (Mantine, next-auth, MkDocs) rather than port them wholesale, and
  we do not build i18n routing or speculative capabilities ahead of need.
