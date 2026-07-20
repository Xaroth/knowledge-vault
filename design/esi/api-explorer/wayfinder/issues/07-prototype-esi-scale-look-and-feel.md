---
type: Wayfinder Ticket
title: "Prototype: surviving ESI scale — final look-and-feel & Try It UX"
description: "Prototype the product's look-and-feel at ESI's real scale (>200 routes/schemas) and the Try It UX, locking the design enough to build the affected renderer slots against."
tags: [design, esi, api-explorer, wayfinder, build-plan, prototype]
timestamp: 2026-07-21T00:00:00Z
---

# Prototype: surviving ESI scale — final look-and-feel & Try It UX

- **Type:** prototype
- **Status:** open
- **Blocked by:** —

## Question

The map's destination assumes a *finished* design to decompose, but the look-and-feel is **not
actually settled at ESI's scale**: the live spec is >200 routes and >200 schemas, and the POC's
design — good enough as a POC — showed real issues at that volume. Because this is
approach-shaping (it changes *how* Try It works, not just how it looks), it is settled with a
concrete artifact before the affected build units start.

Pin down, via a cheap concrete artifact to react to (see the `/prototype` skill):

- **Scale behaviour** — navigation / findability across >200 operations and 36 tags,
  information density, and rendering large / deeply-nested schema trees without the UI
  drowning. This is the primary driver.
- **Try It UX** — how the request builder, auth, and response view work *within* the
  scale-driven layout.
- **Customization surface** — the "moderately customizable by users" selling point: pin down
  *what* is customizable and *for whom* (end-user of the docs page vs integrating developer),
  so the visual slots aren't built into a corner. Relate to the existing developer-facing
  customization story (slot overrides + `--oae-*` tokens, [../../04](../../04-renderer-component.md),
  [../../07](../../07-styling-and-theming.md)).

**Output:** a concrete artifact (outline / mockup / stub UI) that locks the look-and-feel
enough to build the affected renderer visual slots and the Try It unit against — linked from
this ticket as an asset, not pasted in.

**Approach (decided in [ticket 03](./03-spike-inventory-and-gates.md)):** start here (a
prototype now); refine iteratively via the demand-driven playground / stakeholder updates
during execution; pivot to deferring (fog) only if the prototype stalls.

**Dependencies:** blocks the *final plan assembly* and should resolve before sizing the
visual-slot / Try It units, so it informs
[Sizing & estimation approach](./05-sizing-and-estimation.md).
