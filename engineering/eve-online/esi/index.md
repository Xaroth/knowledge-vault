# ESI

ESI — the EVE Swagger Interface — is EVE Online's official RESTful HTTP API for
third-party development. These pages describe the API as one of its consumers sees
it: the host and contract it is served from, how an application authenticates a
character, how behavior is pinned across time, and the caching, rate-limiting, and
pagination rules a well-behaved client lives by.

The authoritative, machine-readable definition of every route is the
[OpenAPI contract](./versioning.md); these pages document the enduring model
around it, not the route list itself.

# Reading order

* [Overview](./overview.md) - what ESI is, the `esi.evetech.net` host, the OpenAPI contract, datasources, and the `/meta/` surface including service health.
* [Authentication](./authentication.md) - EVE SSO: the OAuth 2.0 flows, endpoint discovery, scopes, and the JWT access token and its validation.
* [Versioning](./versioning.md) - the compatibility-date model that pins API-wide behavior, the OpenAPI contract it is expressed in, and how routes are deprecated.
* [Caching](./caching.md) - the cache headers a client honors, conditional requests, and event-driven invalidation.
* [Rate limiting](./rate-limiting.md) - the response-weighted floating-window limit, its headers, and the legacy error limit.
* [Pagination](./pagination.md) - the three pagination models a route may use: cursor-based, page-numbered, and from-ID.
* [Client etiquette](./client-etiquette.md) - identifying an application and the obligations that keep it from being banned.

# Source material

This area is distilled from EVE's official developer documentation and developer
blog, read as the authoritative description of the system:

* ESI, SSO, and related service docs under
  [developers.eveonline.com/docs](https://developers.eveonline.com/docs/) (source:
  [github.com/esi/esi-docs](https://github.com/esi/esi-docs)).
* The EVE developer blog at
  [developers.eveonline.com/blog](https://developers.eveonline.com/blog), for the
  posts that introduced the compatibility-date model, the OpenAPI contract,
  floating-window rate limiting, event-driven caching, and the health endpoint.
