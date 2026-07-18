---
type: Convention
title: Organize by product, not technology
description: Group OKF concepts around what a reader perceives — a feature, a market, a thing they own — rather than around the technology or layer that implements it.
tags: [okf, authoring, structure, product]
timestamp: 2026-07-14T09:32:09Z
---

# The principle

When you carve a subject into concept pages, split it along the lines a **reader**
already has in their head — the *perceived product* — not along the technology or
implementation layer that happens to deliver it.

A human looking for "how does the marketplace work" wants a page called
**Paragon Hub**, not to reassemble that picture from a "database" page, a
"protobuf" page and a "cache" page. Organize for the question, not for the
plumbing.

# Why

- **Findability.** People search by subject ("licenses", "the market"), not by
  layer ("the read endpoints"). Product-shaped pages match how they look.
- **One place per concept.** Everything about a thing — what it is, and how it's
  exposed — lives together, so there's a single obvious home for each new fact.
- **Stable structure.** Products change shape slowly; technologies get replaced.
  Pages named after the product outlive the stack that implements them.
- **Better for LLMs too.** A concept-complete page is a clean unit of retrieval —
  the model gets the whole subject in one chunk instead of three partial ones.

# In practice

- **Name pages after concepts, not layers.** Prefer `paragon-hub`, `skins`,
  `licenses` over `routes`, `endpoints`, `api`, `database`.
- **Give distinct audiences their own area.** When a subject serves separate
  reader groups — the product audience, an API's consumers, a data export's
  consumers — each earns its own area (a product root, an `esi/` folder, an
  `sde/` folder). A surface like ESI or the SDE is itself a product with its own
  readers, so splitting there groups by *audience*, not by plumbing. Different
  readers walk to different doors.
- **Within an area, still organize by concept, and cross-link.** Each area repeats
  the same concepts (a `paragon-hub` page under the product root *and* under
  `esi/`), each answering that audience's version of the question. Link the
  matching pages so a reader following one concept finds it from whichever door
  they came in.
- **The real smell is splitting one concept across implementation layers.** A
  `protobuf.md` + `database.md` + `cache.md` for the same thing forces the reader
  to reassemble it. Audiences and surfaces deserve pages; internal layers nobody
  navigates by do not.
- **Cross-cutting concerns may still be their own page.** A genuinely orthogonal
  fact — a read-only/monitoring scope, an API-wide convention, a shared identifier
  scheme — earns its own page. The test: it applies *across* concepts rather than
  describing one.
- **Keep deep implementation on the engineering side.** Product pages explain the
  concept and its surface; how the machinery is built lives under engineering.
