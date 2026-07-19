---
type: Reference
title: ESI — client etiquette
description: A well-behaved ESI client identifies itself with a descriptive User-Agent carrying contact details, requests only the data it uses, and honors the caching and rate-limit signals ESI sends — circumventing caching or ignoring limits is grounds for being banned.
tags: [eve-online, esi]
timestamp: 2026-07-19T11:30:00Z
---

# Identify the application

Every ESI request should identify the application making it, so CCP can attribute
traffic and reach the operator of a misbehaving client instead of blocking it
blind. The identity is carried one of three ways, depending on what the client can
set:

* **`User-Agent` header** — preferred for non-browser applications.
* **`X-User-Agent` header** — for browser applications, where `User-Agent` is a
  forbidden header the page cannot set.
* **`user_agent` query parameter** — a fallback when no header can be set.

A useful value carries, in rough order of importance: a **contact email**, the
**application name and version**, and optionally a source-code URL, a Discord
handle, or an EVE character name. For example:

```
MyMarketTool/1.4.2 (ops@example.com; +https://github.com/example/my-market-tool)
```

# Request only what you use

Fetch the specific data the application actually needs rather than sweeping whole
datasets it will discard — for example a single region's market orders instead of
every region. Over-broad polling strains both ESI and the live server, and a route
under a short cache window returns nothing new when fetched faster than it can
change.

# Honor the signals ESI sends

ESI tells a client how to behave; a client is obligated to listen:

* Respect [caching](./caching.md): observe `Cache-Control`, revalidate with
  conditional requests, and do not re-request data before it can have changed.
* Respect [rate and error limits](./rate-limiting.md): watch the `X-Ratelimit-*`
  and `X-ESI-Error-Limit-*` headers and back off on `429` / `420` per `Retry-After`.

**Circumventing ESI caching, or ignoring the rate- and error-limit information it
transmits, can get an application banned from ESI.** Good etiquette is not merely
courtesy — it is a condition of access.

# Citations

[1] User-Agent identification, caching obligations, and the ban policy —
`https://developers.eveonline.com/docs/services/esi/best-practices/`.

[2] Requesting only needed data and respecting cache windows —
`https://developers.eveonline.com/blog/market-orders-rate-limit-rolls-out-on-february-24-2026`.
