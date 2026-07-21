---
type: Design Document
title: "Migration"
description: "The phased, route-by-route cutover from developers-next to the new Astro site."
tags: [design, esi, developer-portal, migration]
timestamp: 2026-07-21T12:20:44Z
---

# 10. Migration

## 10.1 Strategy — phased, route-by-route

`developers.eveonline.com` is live, and the [Pulsar backend migration](./06-auth-and-backend.md#63-backend--folds-into-pulsar-services)
lands on its own timeline. The cutover is therefore **phased, route-by-route behind host-level
rewrites**, not a big-bang switch:

- A routing layer (host rewrites) sends each path to **either** the old `developers-next` app **or**
  the new Astro site.
- Surfaces migrate one at a time; the old app serves everything not yet cut over.
- This delivers value incrementally, bounds risk per surface, and **decouples the frontend
  re-platform from Pulsar's timeline** — authed surfaces wait for their Pulsar services without
  blocking everything else.

The cost is running both stacks during the transition and operating the routing layer; this is
accepted in exchange for safety on a live property.

## 10.2 Suggested sequence

Ordered by risk and dependency (least-coupled first):

1. **Docs** (`/docs`) — highest value, no auth, exercises the full two-repo seam
   ([02](./02-two-repo-architecture.md)) and the authoring kit ([05](./05-docs-authoring-contract.md)).
2. **Blog + marketing/home + license** — static, Contentful-backed
   ([04](./04-content-model.md)).
3. **Static data + status + feeds** — static/rewrite surfaces
   ([09](./09-search-i18n-and-cross-cutting.md)); preserve URLs.
4. **API Explorer** — mount the island in the new shell
   ([08.4](./08-surfaces.md#84-api-explorer-boundary)).
5. **Auth** — bring up Astro server OAuth + session
   ([06.2](./06-auth-and-backend.md#62-portal-auth--astro-owns-oauth--session)).
6. **Authed surfaces** (application management, authorized apps) — **last**, gated on their Pulsar
   services being ready.

Sequence is indicative, not fixed; each phase is independently shippable behind the rewrite layer.

## 10.3 Cutover and rollback

Each surface cuts over by flipping its rewrite to the new site once it meets parity and passes its
e2e checks; rollback is flipping the rewrite back. The old app is decommissioned only after the final
surface (the authed Pulsar-backed ones) is cut over and stable. The Amplify/DynamoDB backend is
decommissioned in lockstep with its Pulsar replacement.

## 10.4 Redesign gating

Redesign is **not** part of migration. Each surface migrates **at parity** first. Only where a
migrated surface leaves clear headroom is selective redesign considered, as separate follow-up work —
never as a precondition for that surface's cutover
([parity contract](./01-motivation-and-goals.md#15-the-parity-contract)).
