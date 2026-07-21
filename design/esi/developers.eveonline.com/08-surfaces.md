---
type: Design Document
title: "Surfaces"
description: "Per-surface fate across the whole portal under the re-platform."
tags: [design, esi, developer-portal, surfaces]
timestamp: 2026-07-21T12:20:44Z
---

# 8. Surfaces

Every portal surface and its fate under the re-platform. Rendering is per
[03](./03-rendering-and-hosting.md); "parity" is per
[the parity contract](./01-motivation-and-goals.md#15-the-parity-contract).

## 8.1 Surface inventory

| Surface | Current route(s) | Rendering | Content source | Fate |
| --- | --- | --- | --- | --- |
| **Home / landing** | `/` | Static | Contentful (copy) + code | Parity; native DS |
| **Docs** | external (esi-docs) → `/docs` | Static | Public repo | Re-homed under `/docs`, content-derived nav |
| **Blog** | `/blog`, `/blog/[slug]`, `/blog/page/[page]` | Static | Contentful | Parity; native DS + RSS |
| **API Explorer** | `/api-explorer` (+ `feed.xml`) | Static shell + island | ESI spec | Consumed as-designed ([plan](../api-explorer/README.md)) |
| **Application management** | `/applications`, `/applications/create`, `/applications/details/[app_id]` | Server (authed) | Pulsar | Parity; proxy to Pulsar |
| **Authorized apps** | `/authorized-apps/[character_id]/[app_id]` | Server (authed) | Pulsar | Parity; proxy to Pulsar |
| **ESI status** | `/status` | Static + short-TTL data | ESI status source | Parity |
| **Static data** | `/static-data` (+ `latest` zips, `feed.xml`) | Static page + rewrite/redirect | S3 | Parity; host rewrite → S3 + `latest` redirect route |
| **Auth** | `/login`, `/logout`, `/api/auth/*`, callback, info | Server | EVE SSO | Rebuilt on Astro server routes ([06](./06-auth-and-backend.md)) |
| **License agreement** | `/license-agreement` | Static | Contentful | Parity; editorial in Contentful |
| **Feeds** | `/feed.xml`, `/blog/.../feed.xml`, `/api-explorer/feed.xml`, `/static-data/feed.xml` | Static/build | derived | Parity; URLs preserved |

## 8.2 URL and feed stability

All current URLs and feeds keep resolving. Where the docs re-home changes a path, the docs frontmatter
carries a redirect/alias ([05.5](./05-docs-authoring-contract.md#55-frontmatter-contract)) so old
links survive. Feed URLs are preserved exactly.

## 8.3 Authed surfaces depend on Pulsar

Application management and authorized apps cannot cut over until their **Pulsar services** exist
([06.3](./06-auth-and-backend.md#63-backend--folds-into-pulsar-services)). The phased migration
([10](./10-migration.md)) sequences these last for that reason.

## 8.4 API Explorer boundary

The Explorer is mounted as a **React island** in the private site, themed from the portal tokens
([07](./07-design-system-and-theming.md)). Its spec loading, vendor extensions, and Try-It auth hook
are governed by its own plan. The portal provides only the page shell, navigation, and token context.
