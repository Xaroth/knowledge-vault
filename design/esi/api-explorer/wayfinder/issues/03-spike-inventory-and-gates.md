---
type: Wayfinder Ticket
title: "Spike inventory & exit gates"
description: "Decide which flagged hard parts become up-front spike/validation tickets, and each spike's exit criterion."
tags: [design, esi, api-explorer, wayfinder, build-plan]
timestamp: 2026-07-20T00:00:00Z
---

# Spike inventory & exit gates

- **Type:** grilling
- **Status:** claimed
- **Blocked by:** —

## Question

Which of the design's flagged hard parts become explicit up-front spike/validation tickets
before committing the main build, and what is each spike's exit criterion?

Candidates called out in the design and review:

- **OpenAPI 3.1 normalization** — the gating risk; POC does not handle 3.1 constructs
  ([../../03](../../03-parser-package.md#35), [../../08](../../08-esi-integration-and-migration.md#85)).
- **snapshot-before-dereference** — preserve `$ref` identity + vendor extensions
  ([../../03](../../03-parser-package.md#32)).
- **`allOf` merge-and-preserve** — flagged "most likely to be contentious"
  ([../../03](../../03-parser-package.md#310)).
- **Try It query serialization** (`form`/`space`/`pipe`/`deepObject`, explode)
  ([../../06](../../06-authentication-and-try-it.md)).
- **Island zero-context mount** — de-risk late host-context reliance
  ([../../08](../../08-esi-integration-and-migration.md#85)).
- **Concrete perf budget** — mount <~200 ms, nav <~50 ms on the ESI spec
  ([../../09](../../09-quality-and-resilience.md#92)).
