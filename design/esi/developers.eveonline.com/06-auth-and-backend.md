---
type: Design Document
title: "Authentication & Backend"
description: "Astro-owned OAuth + session, and the offload of backend logic to Pulsar services."
tags: [design, esi, developer-portal, authentication, eve-sso, pulsar]
timestamp: 2026-07-21T12:20:44Z
---

# 6. Authentication & Backend

## 6.1 Two auth flows

The portal has two distinct authentication concerns; do not conflate them:

1. **Portal / developer identity** — logging a developer in, and authorizing the dynamic developer
   surfaces (application management, authorized apps). Owned by the portal (this document).
2. **API Explorer "Try It" ESI token** — obtaining an access token to call ESI from the browser via
   the dedicated `SPEC_VIEWER` SSO client. Owned by the
   [API Explorer plan](../api-explorer/README.md), not here.

## 6.2 Portal auth — Astro owns OAuth + session

The private site's **server routes own the EVE SSO OAuth flow and the session**:

- The SSO **authorization-code exchange** happens server-side; the **SSO client secret stays on the
  server** and never reaches the browser.
- A **httpOnly, secure session cookie** is issued and validated by the site's server routes.
- This replaces today's `next-auth` + `oauth4webapi` + `jose` wiring with Astro server endpoints
  (redirect → callback → session), against `login.eveonline.com` (prod) and
  `sisilogin.testeveonline.com` (test).

This is why the site is **hybrid, not static** ([03](./03-rendering-and-hosting.md)): the auth routes
and the authed surfaces require a server runtime.

```
browser ──/login──► Astro server route ──redirect──► EVE SSO
EVE SSO ──code──► Astro /callback (server) ──exchange (secret)──► tokens
Astro ──set httpOnly session cookie──► browser
browser (authed) ──► Astro authed route ──proxy(+token)──► Pulsar service
```

## 6.3 Backend — folds into Pulsar services

Today `developers-next` bundles an **AWS Amplify backend** (API Gateway + Lambda + DynamoDB) with
functions such as `sign-developer-agreement`, `mark-as-active`, `get-character-list`, and
`check-developers-validity` — i.e. developer-account and registration metadata.

**That backend logic folds into Pulsar services.** The portal becomes a pure frontend plus a thin
authenticated proxy: data operations (list/create/edit OAuth applications, authorized apps,
developer registration, agreement signing, validity) are **offloaded to Pulsar** and reached over
HTTP with the session's credentials attached server-side.

- Pulsar is an **external CCP platform**; its design, API surface, and timeline are **out of scope**
  here and are a **dependency** ([11](./11-open-decisions.md), [01 NG2](./01-motivation-and-goals.md#14-non-goals)).
- The Amplify/DynamoDB backend is **decommissioned** once its logic lives in Pulsar.
- The migration is **decoupled** from the frontend re-platform by the phased cutover
  ([10](./10-migration.md)): authed surfaces cut over only when their Pulsar services are ready.

## 6.4 External dependencies

| Dependency | Role |
| --- | --- |
| EVE SSO (`login.eveonline.com` / `sisilogin`) | OAuth, developer identity |
| ESI (`esi.evetech.net` / `-dev` / `-test`) | API surface (Try-It, spec) |
| Pulsar services | Developer/application data operations |
| Contentful | Editorial content ([04](./04-content-model.md)) |
| S3 | Static-data artifacts ([09](./09-search-i18n-and-cross-cutting.md)) |

## 6.5 Secrets

SSO client secrets (portal client and the API Explorer's `SPEC_VIEWER` client) live only in the
private site's server environment and/or the relevant Pulsar service — never in the public repo, never
in client bundles.
