---
type: Wayfinder Ticket
title: "Decomposition strategy & ticket granularity"
description: "Decide what a buildable unit is and at what granularity the build plan cuts tickets."
tags: [design, esi, api-explorer, wayfinder, build-plan]
timestamp: 2026-07-20T00:00:00Z
---

# Decomposition strategy & ticket granularity

- **Type:** grilling
- **Status:** claimed
- **Blocked by:** —

## Question

What is a "buildable unit" in this plan, and at what granularity do we cut tickets?

Concretely: is the parser sliced per module (`resolve-input`, `original-spec-snapshot`,
`validate-spec`, `load-policy`, `normalize-schema`, `normalize-operations`, … per
[../../12](../../12-repository-and-module-layout.md#122)) or per capability? Is the renderer
sliced per slot component or per feature area? How is the ESI adapter cut (vendor renderers,
auth/fetcher wiring, labels, theme tokens)?

This decision sets the shape of every downstream ticket, so it is the spine of the plan and
the first thing to resolve.
