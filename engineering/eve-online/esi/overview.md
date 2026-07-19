---
type: Reference
title: ESI — overview
description: ESI is EVE Online's official RESTful HTTP API, served from esi.evetech.net with an OpenAPI contract, scoped per game datasource (tranquility / singularity), with spec and service-health routes under a /meta/ surface.
tags: [eve-online, esi, overview]
timestamp: 2026-07-19T11:00:00Z
---

# What ESI is

ESI — the **EVE Swagger Interface** — is EVE Online's official RESTful HTTP API for
third-party development. It exposes game state (characters, corporations,
the market, the map, assets, and much more) as JSON over HTTP. Most routes read
data; some write it. The majority require an authenticated character and specific
consented scopes; a minority are public.

It is served from the host **`https://esi.evetech.net/`**. Requests carry an
[`X-Compatibility-Date`](./versioning.md) rather than a version number in the path,
so routes are version-agnostic (`/characters/{id}/…`, `/markets/{region_id}/orders/`).

# The contract is the OpenAPI spec

ESI does not document its routes in prose. The authoritative, machine-readable
contract is an **OpenAPI document** served under the `/meta/` surface, and the
[API Explorer](https://developers.eveonline.com/api-explorer) renders it for
humans. Every route, parameter, response schema, required scope, and per-route
cache and rate-limit note lives in that spec. See
[versioning](./versioning.md) for the spec formats and how it is pinned in time.

# Datasources

ESI serves the state of a specific **datasource** — an EVE server:

* **`tranquility`** — the single live game world.
* **`singularity`** — the public test server.

A client selects the datasource per request (as a `datasource` query parameter).
The same segment names the server elsewhere in EVE's developer surface — for
example in the [Static Data Export](../static-data-export/index.md) path
(`.../static-data/tranquility/…`).

# The `/meta/` surface

Cross-cutting, non-game endpoints live under `/meta/`:

* The **OpenAPI contract** itself (see [versioning](./versioning.md)).
* **Service health** at `/meta/status`, which reports the health of every route
  individually. Each route is classified as one of **OK**, **Degraded**, **Down**,
  or **Recovering**, derived from its observed server-error (`5XX`) rate. A public
  status dashboard at
  [developers.eveonline.com/status](https://developers.eveonline.com/status)
  surfaces the same data.

# The rules a client lives by

Four enduring concerns govern every ESI client, each with its own page:

* [Authentication](./authentication.md) — obtaining and validating a character's
  access token through EVE SSO.
* [Caching](./caching.md) — honoring cache headers and using conditional requests
  so unchanged data is not refetched.
* [Rate limiting](./rate-limiting.md) — staying within the request budget.
* [Client etiquette](./client-etiquette.md) — identifying the application and the
  obligations that keep it in good standing.

# Citations

[1] ESI overview and endpoints —
`https://developers.eveonline.com/docs/services/esi/overview/`,
`https://developers.eveonline.com/docs/services/esi/endpoints/`.

[2] Health endpoint and `esi.evetech.net` host —
`https://developers.eveonline.com/blog/a-better-view-on-status-improving-esi-health-monitoring`.
