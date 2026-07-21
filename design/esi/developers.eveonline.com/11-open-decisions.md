---
type: Design Document
title: "Open Decisions"
description: "Decisions deliberately left open, with the criteria and dependencies to resolve them."
tags: [design, esi, developer-portal, decisions]
timestamp: 2026-07-21T12:20:44Z
---

# 11. Open Decisions

Decisions that are intentionally unresolved. Each records the options, what it depends on, and the
criteria to settle it. They do not block drafting the plan; they must be settled before the phases
they gate.

## D1 — Hosting target

**Question:** Where does the private hybrid Astro site run?

**Options (AWS family only; Cloudflare out):**
- AWS-native app inside our own cluster
- AWS Amplify (with SSR)
- AWS container environment (Fargate / ECS-style)
- our own Kubernetes cluster

**Constraint:** a server runtime is mandatory
([03.2](./03-rendering-and-hosting.md#32-consequence-a-server-runtime-is-mandatory)); static-only is
out. The design targets a **host-agnostic container** (Node adapter) so this choice stays reversible
([03.3](./03-rendering-and-hosting.md#33-host-agnostic-deployment-artifact)).

**Criteria:** ops ownership, CCP infra/security standards, cost, parity with existing deployment
practice.

**Gates:** the deploy pipeline, and therefore D2. Must be settled before the first cutover
([10](./10-migration.md)).

## D2 — Rebuild trigger

**Question:** What makes the private site rebuild+redeploy when public docs content changes, while the
public repo stays hosting-agnostic?

**Options:**
- Webhook configured in the public repo's **settings** (not tracked tree) → private build hook
- `repository_dispatch` from a minimal public workflow (accepts a little coupling in the public tree)
- Low-frequency **scheduled rebuild** as a fallback / self-heal

**Depends on:** D1 (the pipeline determines what a "build hook" is).

**Criteria:** deploy latency, robustness, and keeping the public tree free of hosting concerns.
Leaning webhook/dispatch, possibly with a cron fallback.

## D3 — Redesign headroom

**Question:** How much visual/UX redesign does the re-platform's headroom permit, beyond parity?

**Status:** Parity is the committed floor; redesign is a **separable, gated upside**
([parity contract](./01-motivation-and-goals.md#15-the-parity-contract)). The new Astro-native design
system ([07](./07-design-system-and-theming.md)) already shifts the visual language while holding IA
at parity.

**Resolution:** decided per-surface, *after* that surface migrates at parity, based on effort left and
value — never as a precondition for cutover.

## Dependencies referenced but out of scope

- **Pulsar services** — the target for backend logic
  ([06.3](./06-auth-and-backend.md#63-backend--folds-into-pulsar-services)). Undocumented in this
  vault; needs its own design/docs. Gates the authed surfaces' cutover.
