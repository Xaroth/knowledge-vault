---
type: Design Document
title: "Motivation & Goals"
description: "The split-stack pain that drives the re-platform, and the goals and non-goals that bound it."
tags: [design, esi, developer-portal, motivation, goals]
timestamp: 2026-07-21T12:20:44Z
---

# 1. Motivation & Goals

## 1.1 Where we are today

`developers.eveonline.com` is served by **two unrelated stacks**:

- **`developers-next`** — the portal application: Next.js 14, React 18, **Mantine** UI, **Contentful**
  CMS (GraphQL), `next-auth` + `oauth4webapi` for EVE SSO, `next-intl` (en-only), `@stoplight/elements`
  for the API Explorer, deployed on **AWS Amplify**, with a co-located Amplify backend (API Gateway +
  Lambda + DynamoDB).
- **`esi-docs`** — the documentation: **MkDocs Material** (Python), a **public** and
  community-contributable repository (`CONTRIBUTORS`, `LICENSE`, community PRs), served statically
  (GitHub-Pages-style, via `CNAME`).

The two share no code, no components, no build, and no design system.

## 1.2 Concrete pain points

These are the specific problems the re-platform is designed to eliminate.

### P1 — Every design change is done twice

A change to shared look-and-feel (navigation, footer, typography, colours) must be reimplemented
independently in React/Mantine and in MkDocs Material's Python/Jinja theme. There is no shared
component, token, or style. The two portals drift, and the cost of keeping them consistent is paid
continuously.

### P2 — Licensed material cannot live with the docs

`esi-docs` is public and open to community contribution. Licensed fonts and imagery may not be
committed to it. Today this forces awkward workarounds and keeps the public docs visually degraded
relative to the portal.

### P3 — Presentation and infrastructure leak into the public repo

`mkdocs.yml` carries theme configuration, licensed fonts, analytics (GTM), hero/footer chrome,
`overrides/`, custom CSS/JS, and content-generation hooks. Contributors editing docs are exposed to —
and can break — presentation and operational concerns that are not theirs.

### P4 — Two toolchains, two skill sets

Maintaining the portal requires fluency in both a modern React/TypeScript stack and a Python/MkDocs
stack. Tooling, CI, linting, and testing are duplicated and divergent.

## 1.3 Goals

- **G1** — One stack. The entire portal is built on **Astro**.
- **G2** — Separate concerns by *repository visibility*: a public content repo and a private styled
  repo, so licensed assets and infrastructure stay private while docs stay open and contributable.
- **G3** — Docs content updates flow to the live site **transparently** — no private-repo change per
  doc edit.
- **G4** — The public repo contains **nothing hosting-related** and is standalone-previewable.
- **G5** — Reach **feature and information-architecture parity** with today's portal.
- **G6** — Shed moving parts that the unification makes redundant (Mantine, `next-auth`, MkDocs,
  `next-intl` routing) rather than port them.
- **G7** — Unify theming with the API Explorer via a shared design-token contract.

## 1.4 Non-goals

- **NG1 — Visual / UX redesign is not in scope as a commitment.** Parity is the floor; a redesign is a
  separable, gated upside (see [10-migration.md](./10-migration.md) and
  [11-open-decisions.md](./11-open-decisions.md)). The re-platform's success never depends on a
  redesign landing.
- **NG2 — Designing Pulsar.** Backend logic folds into **Pulsar services**; Pulsar's own design is an
  external dependency, not specified here (see [06-auth-and-backend.md](./06-auth-and-backend.md)).
- **NG3 — Re-designing the API Explorer.** It is consumed as-designed
  ([API Explorer plan](../api-explorer/README.md)).
- **NG4 — Multi-language content.** The portal remains English-only
  ([09](./09-search-i18n-and-cross-cutting.md)).
- **NG5 — Building the static-data generation pipeline.** Static data is already produced and stored
  externally; the portal only serves links to it.

## 1.5 The parity contract

"Parity" means: every surface listed in [08-surfaces.md](./08-surfaces.md) works, every current URL
and feed keeps resolving, and the information architecture is preserved. Because the design system is
rebuilt Astro-native (Mantine is retired), the *visual language* will change even under parity — but
the *layout and IA* of each surface are reproduced. Where the re-platform leaves headroom, selective
redesign may be adopted; it is never a precondition for cutover.
