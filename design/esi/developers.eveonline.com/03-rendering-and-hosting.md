---
type: Design Document
title: "Rendering & Hosting"
description: "The hybrid rendering model and the host-agnostic AWS deployment artifact."
tags: [design, esi, developer-portal, astro, ssr, hosting]
timestamp: 2026-07-21T12:20:44Z
---

# 3. Rendering & Hosting

## 3.1 Rendering model — hybrid

The site is a **hybrid** Astro application: prerendered by default, server-rendered only where a
request genuinely needs a server.

- **Prerendered (static):** docs, blog, marketing/home, license agreement, status (see caveat), and
  the API Explorer shell. These are built once per deploy and served from cache/CDN.
- **Server-rendered (on demand):** authentication (OAuth redirect + callback + session), and the
  authenticated dynamic surfaces (application management, authorized apps) which proxy to Pulsar.
  See [06-auth-and-backend.md](./06-auth-and-backend.md).

Most of the portal is static; the dynamic surface is a small, well-bounded set of routes. This is
Astro's server output with per-route `prerender` defaults — static unless a route opts into
on-demand rendering.

## 3.2 Consequence: a server runtime is mandatory

Because auth and the authed surfaces run server-side, the site **cannot** be deployed as
static-only (plain S3 + CloudFront with no compute). The deploy target must provide a Node/edge
runtime. This rules out static-only hosting and shapes the packaging below.

## 3.3 Host-agnostic deployment artifact

The **hosting target is an open decision** constrained to the AWS family
([11-open-decisions.md](./11-open-decisions.md)):

- an AWS-native app inside our own cluster,
- AWS Amplify (with SSR),
- an AWS container environment (Fargate / ECS-style),
- our own Kubernetes cluster.

Cloudflare is explicitly out.

To keep those options genuinely open, the site is designed to build to a **host-agnostic artifact**:
a **container image running Astro's Node adapter**. A Node-server container runs unchanged on
Fargate/ECS and on our k8s cluster, and can also sit behind Amplify/CloudFront. The deploy pipeline
and the [rebuild trigger](./02-two-repo-architecture.md#24-the-content-seam) are chosen together with
the host; the artifact contract (a container serving the hybrid app) stays constant across them.

```
astro build (node adapter) ──► container image ──► { in-cluster app | Amplify | Fargate/ECS | k8s }
                                                    front: CDN for static assets
```

## 3.4 Caching and invalidation

Static output is fingerprinted and CDN-cached aggressively. Each deploy publishes a fresh set;
content changes reach users when the [content-triggered rebuild](./02-two-repo-architecture.md#24-the-content-seam)
redeploys. Server routes set their own cache policy (auth routes: no-store; status: short TTL).

## 3.5 Observability

Sentry is carried across from the current stack for error and performance monitoring of the server
routes and client islands. Build metadata records the deployed site version *and* the cloned docs
content SHA (per [2.4](./02-two-repo-architecture.md#24-the-content-seam)).
