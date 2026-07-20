---
type: Wayfinder Ticket
title: "Does the ESI spec mark internal operations?"
description: "Research whether the live ESI OpenAPI document uses x-internal (or equivalent), deciding if operation-filtering belongs in the plan."
tags: [design, esi, api-explorer, wayfinder, build-plan, research]
timestamp: 2026-07-20T00:00:00Z
---

# Does the ESI spec mark internal operations?

- **Type:** research
- **Status:** resolved
- **Blocked by:** —

## Question

Does the live ESI OpenAPI document mark internal operations with an `x-internal` vendor
extension (or any equivalent)?

Resolves whether the deferred operation-filtering feature (`hideInternal`,
[../../04](../../04-renderer-component.md#47), [../../05](../../05-feature-set.md#54)) belongs
in the build plan or stays out of scope. The design defers it explicitly: "add only if ESI
marks internal operations."

Source: the live spec at
`https://esi.evetech.net/meta/openapi.json?compatibility_date=<date>`.

## Answer

**No.** The live ESI spec (`?compatibility_date=2025-01-01`, OpenAPI 3.1.0, 182 paths,
202 schemas) contains **zero** internal/hidden markers: no `x-internal`, no `x-hidden`, no
`"deprecated": true`, and no naming convention distinguishing internal operations.

The only vendor extensions present are `x-cache-age`, `x-cache-mode`, `x-common-model`,
`x-compatibility-date`, `x-enum-descriptions`, `x-rate-limit`, and `x-required-roles`.
(Note: the live spec uses `x-required-roles`, whereas the design references
`x-required-scope` — worth reconciling when the adapter's extension registry is planned.)

**Consequence for the plan:** the operation-filtering feature (`hideInternal`) stays **out
of scope** — there is no signal in the ESI spec for it to act on. The design already gates it
on exactly this ("add only if ESI marks internal operations").
