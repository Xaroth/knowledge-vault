---
type: Wayfinder Ticket
title: "Build order & dependency backbone"
description: "Formalize the design's phase sequencing and one-way package dependencies into the plan's ordered backbone."
tags: [design, esi, api-explorer, wayfinder, build-plan]
timestamp: 2026-07-20T00:00:00Z
---

# Build order & dependency backbone

- **Type:** grilling
- **Status:** open
- **Blocked by:** 01

## Question

Turn the design's Phase 0–4 sequencing
([../../08](../../08-esi-integration-and-migration.md#84)) and the strict one-way dependency
direction `adapter → explorer → model` ([../../02](../../02-architecture.md#21)) into the
plan's ordered backbone: the dependency-wired sequence of buildable units.

Must capture the hard gates: `NormalizedSpec` frozen before any consumer builds against it,
the model stabilizing before the renderer (renderer depends via a caret range, not a pinned
workspace version), and the provider-stack/slot-registry existing before individual slots.
