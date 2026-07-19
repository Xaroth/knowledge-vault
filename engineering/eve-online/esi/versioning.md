---
type: Reference
title: ESI — versioning
description: ESI pins behavior with a date, not a URL version — an API-wide X-Compatibility-Date header selects the API as it was on that day, expressed in an OpenAPI contract, with breaking changes gated behind newer dates and routes retired on an announced redirect-then-remove cadence.
tags: [eve-online, esi, api-stability]
timestamp: 2026-07-19T11:10:00Z
---

# Behavior is pinned by date

ESI routes carry no version in their path. Instead, a request declares the date at
which the application was written against the API, and ESI serves the behavior as
it was on that date. This applies **API-wide** — a single date governs every route
in the request, rather than each route carrying its own version number.

The date is sent as an **`X-Compatibility-Date`** header in ISO `YYYY-MM-DD` form:

```
X-Compatibility-Date: 2025-08-26
```

For clients that cannot set custom headers, a **`compatibility_date` query
parameter** carries the same value. The response echoes the
`X-Compatibility-Date` that was applied, so a client can confirm which behavior it
received. When a request sends no date, the **oldest available** compatibility date
is applied.

# What a new date gates

API changes take effect at a daily boundary (**11:00 UTC**). A change is either
**breaking** — visible only to requests using a compatibility date on or after the
change — or **non-breaking**, applied within existing dates because it cannot break
a conforming client:

| Requires a newer date (breaking) | Applies within existing dates (non-breaking) |
| --- | --- |
| A new route | A new optional parameter |
| A newly added or newly required parameter | A new response field |
| A type change to a parameter, field, or header | A new response header |
| Removing a parameter, field, header, or enum value | A new enum value |

A given compatibility date is supported for **at least one year**. Infrastructure
changes can occasionally shorten that, but only with advance notice.

An application therefore stays on a fixed date, reviews the changes introduced by
later dates when it chooses to, and moves its date forward deliberately — it is
never forced to absorb a breaking change it has not adopted.

# Preview access via a future date

Because behavior is keyed to a date, a route can be released to selected
applications ahead of general availability by granting them a **future**
compatibility date. The route is reachable only to a request carrying that date,
which makes the same mechanism serve early access as well as versioning.

# The OpenAPI contract

The machine-readable contract is an **OpenAPI** document, served under the
[`/meta/` surface](./overview.md):

* OpenAPI **3.1** (primary): `https://esi.evetech.net/meta/openapi.json`
* OpenAPI **3.0** (for tooling that requires it):
  `https://esi.evetech.net/meta/openapi-3.0.json`

OpenAPI is the only format the contract is published in. It expresses constructs
the older Swagger 2.0 format could not — notably `oneOf` for mutually exclusive
fields — and lets shared entities (a character, corporation, or alliance ID) be
defined once as reusable components rather than repeated per route. Route-level
notes in the spec also document each route's caching and rate-limit behavior.

# How routes are retired

A route is removed on an announced, **phased** cadence rather than cut off at once:

1. The route is first **redirected** to its modern replacement.
2. The redirect itself is removed only later (on the order of a month afterward),
   leaving a grace period during which old clients still function.

New routes and changes appear **only** in the OpenAPI contract. Retirements are
communicated in advance on the developer blog, always pointing at the replacement.

# Citations

[1] Compatibility-date model and change classification —
`https://developers.eveonline.com/docs/services/esi/overview/`,
`https://developers.eveonline.com/blog/changing-versions-v42-was-getting-out-of-hand`.

[2] OpenAPI contract —
`https://developers.eveonline.com/blog/changing-specs-from-swagger-to-openapi`,
`https://developers.eveonline.com/blog/goodbye-swagger-removing-the-last-remnants`.

[3] Preview via future compatibility date —
`https://developers.eveonline.com/blog/early-access-corporation-projects`.

[4] Phased route retirement —
`https://developers.eveonline.com/blog/spring-cleaning-legacy-routes-removed-24-march-2026`.
