# EVE Online

Engineering knowledge about EVE Online's client-facing systems, documented from
the outside — as a third-party tool sees them, without access to CCP's internal
sources.

# Areas

* [ESI](./esi/index.md) - EVE's official RESTful HTTP API for third-party development: the `esi.evetech.net` host and OpenAPI contract, EVE SSO authentication, compatibility-date versioning, and the caching, rate-limiting, and pagination rules its clients follow.
* [Resource distribution](./resource-distribution/index.md) - how the EVE client's binaries and resources are published to CDNs, discovered by build, indexed, and content-addressed — plus the tools that resolve `res:/` paths against it.
* [Static Data Export](./static-data-export/index.md) - how EVE's static game data (types, dogma, the map, …) is published as versioned JSON Lines and YAML archives, its record and change formats, and the tool that loads it at build time.
