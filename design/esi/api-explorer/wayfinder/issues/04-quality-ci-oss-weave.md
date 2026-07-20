---
type: Wayfinder Ticket
title: "How quality, CI & OSS-prep thread into the plan"
description: "Decide whether cross-cutting quality/testing/CI/OSS-readiness work is woven per-ticket or split into standalone tickets."
tags: [design, esi, api-explorer, wayfinder, build-plan]
timestamp: 2026-07-20T00:00:00Z
---

# How quality, CI & OSS-prep thread into the plan

- **Type:** grilling
- **Status:** open
- **Blocked by:** 01

## Question

How do the cross-cutting requirements thread into the plan — accessibility, security,
performance, testing strategy + spec corpus, CI gates, and the public-API/semver/OSS-readiness
work ([../../09](../../09-quality-and-resilience.md), [../../10](../../10-testing-and-tooling.md),
[../../11](../../11-open-source-and-api-stability.md))?

Per-ticket definition-of-done woven into each buildable unit, standalone tickets, or a mix?
This determines whether quality is a gate on every ticket or scheduled as separate work, and
where the `data-oae-*` DOM contract, the a11y APG patterns, the XSS rules, and the CI budget
gates land in the sequence.

## Input from decomposition strategy (ticket 01)

The [decomposition decision](./01-decomposition-strategy.md) **decoupled the playground** from
the build's inner loop: it becomes a demand-driven stakeholder-update vehicle, not a DoD hook
and not a standing CI harness. So this ticket must decide what the CI smoke/typecheck gate
runs against instead — the working assumption is the **test suite + real-spec fixtures**
([../../10](../../10-testing-and-tooling.md#102)), with playground smoketests as a secondary
check only. This diverges from the design's "playground as CI harness"
([../../12](../../12-repository-and-module-layout.md#124),
[../../10](../../10-testing-and-tooling.md#105)); reconcile the CI-gate definition accordingly.
