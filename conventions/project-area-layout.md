---
type: Convention
title: Layout of a project area
description: How a project (a product feature) is laid out — a product-description root plus one folder per exposure surface (ESI, SDE, …), each organized by the same concepts.
tags: [okf, authoring, structure, project, product]
timestamp: 2026-07-14T09:32:09Z
---

# What a project area is

A **project** under [`project/`](../project/index.md) is product-level knowledge about one
EVE feature or initiative — what it *is* and *why it exists*, for a mixed audience
of product people, API consumers and data consumers. This page describes the shape
every project area takes, so a new one lands the same way.

# The shape

A project area has a **product-description root** and **one folder per exposure
surface** — the distinct ways the feature reaches the outside world (the public
API, the static-data export, …). Each surface is a product in its own right with
its own readers, so it gets its own area.

```
project/<feature>/
  index.md          three-part guide + source material
  overview.md       what the feature is, and its lifecycle
  <concept>.md      one page per product concept (e.g. skins, licenses, paragon-hub)
  esi/              the ESI surface — for API engineers and third-party devs
    index.md
    scope.md        what the API lets you do, and what it doesn't
    <concept>.md    the routes for that concept + the eve-proto domain behind them
  sde/              the Static Data Export surface — for data consumers
    index.md
    <concept>.md    the static definitions that concept depends on
```

Not every feature has every surface — add a surface folder only when the feature
actually reaches readers that way.

# The rules it follows

- **Root is product-only.** The root pages say what the feature *is*. They carry no
  routes, protos or endpoint detail, so a non-technical reader never wades through
  API mechanics. ([organize by product](./organize-by-product.md).)
- **One area per audience/surface.** Product, ESI and SDE readers each walk to a
  different door. A surface like ESI or the SDE is an audience, not a plumbing
  layer, so it earns a folder. ([organize by product](./organize-by-product.md).)
- **Every area is organized by the same concepts, and cross-links.** A concept
  (say `paragon-hub`) has a product page *and* an `esi/` page, each answering that
  audience's version of the question, linked to each other.
- **Scope is a per-surface cross-cutting page.** The read-only/monitoring shape of
  a surface lives in that surface's folder (`esi/scope.md`), not on a concept page.
- **Present tense, no project narrative.** Pages describe what is, not what is being
  built or changed. ([describe the present](./describe-current-state.md).)
- **Sources stay at the area level and stay light.** Link the driving design docs
  in the root `index.md` under *Source material*. Keep tickets out of the concept
  pages — they aren't the knowledge, only where it came from.
