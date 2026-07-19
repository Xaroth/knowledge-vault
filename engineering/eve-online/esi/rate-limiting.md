---
type: Reference
title: ESI — rate limiting
description: ESI meters requests with a floating-window token bucket scoped per application and character, where the token cost of a request is weighted by its response class so errors cost most, reports consumption in response headers, and returns 429 with Retry-After when a bucket is empty.
tags: [eve-online, esi, resilience]
timestamp: 2026-07-19T11:20:00Z
---

# The floating-window model

ESI meters requests with a **floating-window token bucket**. Each request consumes
tokens from a bucket; the tokens a request spent are returned to the bucket once
the window has passed, so capacity continuously recovers rather than resetting at a
fixed instant.

The distinctive part is that a request's cost is **weighted by its response
class**, not counted flat. Failures — especially client errors — cost more, which
makes an error-generating loop exhaust its budget quickly instead of hammering the
API:

| Response class | Token cost | Rationale |
| --- | --- | --- |
| `2XX` success | 2 | the baseline cost of doing work |
| `3XX` (e.g. `304 Not Modified`) | 1 | cheapest — rewards conditional requests |
| `4XX` client error | 5 | most expensive — discourages hammering on user errors (a `429` itself is exempt) |
| `5XX` server error | 0 | a client is not penalized for ESI's own failures |

The specific token values and window length are current tuning and can change; the
enduring model is *weighted-by-response-class, returned-after-a-window*. A
conforming, well-cached application generally never notices the limit — it targets
buggy or abusive traffic.

# Bucket scope

A bucket is keyed by a **rate-limit group** (a set of related routes) combined with
a **user identity**, so one application's behavior on one character does not consume
another's budget:

* **Authenticated routes** — the identity is `<applicationID>:<characterID>`.
* **Unauthenticated routes** — the identity is the source IP (optionally combined
  with the application ID).

# The headers

Rate-limited routes report live consumption in response headers, so a client can
steer without guessing:

* `X-Ratelimit-Group` — which group the route belongs to.
* `X-Ratelimit-Limit` — the bucket size and window (a value such as `150/15m`).
* `X-Ratelimit-Remaining` — tokens left in the bucket.
* `X-Ratelimit-Used` — tokens consumed.
* `Retry-After` — seconds to wait, sent only on a `429`.

When a bucket is empty, ESI returns **HTTP 429 Too Many Requests** with a
`Retry-After` header. A client waits that long before retrying.

# The legacy error limit

Routes not yet migrated to the floating-window system fall under an older
**error-limiting** scheme. It counts non-`2xx`/`3xx` responses in a rolling window;
when the budget is exhausted, ESI returns **HTTP 420** on **all** ESI routes until
the window resets. Its state is reported in headers:

* `X-ESI-Error-Limit-Remain` — errors still allowed in the current window.
* `X-ESI-Error-Limit-Reset` — seconds until the window resets.

A client watches these and backs off before hitting `420`, exactly as it watches
the `X-Ratelimit-*` headers for `429`.

# Citations

[1] Floating-window model, token weights, bucket scope, and headers —
`https://developers.eveonline.com/docs/services/esi/rate-limiting/`,
`https://developers.eveonline.com/blog/hold-your-horses-introducing-rate-limiting-to-esi`.

[2] Legacy error-limit headers (`X-ESI-Error-Limit-*`, `420`) —
`https://developers.eveonline.com/docs/services/esi/best-practices/`.
