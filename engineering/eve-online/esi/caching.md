---
type: Reference
title: ESI — caching
description: ESI data is cached and clients must honor it — conditional requests with ETag/If-None-Match yield 304s, and cache invalidation is event-driven from the live server, so Cache-Control is the meaningful signal while Expires remains only for backwards compatibility.
tags: [eve-online, esi, performance]
timestamp: 2026-07-19T11:15:00Z
---

# Responses are cached, and clients must honor it

Most ESI responses are cached, and a client is obligated to respect that cache
rather than re-request data it already holds. Ignoring caching is grounds for
being [banned from ESI](./client-etiquette.md). Some responses — notably `POST`
methods — may carry no cache headers even though ESI caches them internally.

# Cache headers

ESI communicates cache behavior with standard HTTP headers:

* **`Cache-Control`** — the primary, current signal for how a response may be
  cached. Clients rely on it.
* **`ETag`** — an opaque validator (a content hash) for a response. A client stores
  it and sends it back to make a conditional request.
* **`Last-Modified`** — when the resource last changed. Every page of a paginated
  resource carries an identical `Last-Modified`.
* **`Expires`** — retained for backwards compatibility only. It is **no longer
  meaningful** as a freshness signal, because expiry is no longer time-driven (see
  below). Clients use `Cache-Control`, not `Expires`.

# Conditional requests

Rather than re-download unchanged data, a client revalidates with a conditional
request. It sends the stored `ETag` back in an **`If-None-Match`** header; if the
resource is unchanged, ESI returns **`304 Not Modified`** with no body. This
transfers no payload and — because a `3XX` response is cheaper than a `2XX` — also
costs less against the [rate limit](./rate-limiting.md).

# Event-driven invalidation

ESI does not expire cached data on a fixed timer. Instead it **invalidates the
cache when an event from the live server signals that the underlying data actually
changed** — for example a change to a character's skills or skill queue. Between
events the cached value is authoritative, and the backend continuously
revalidates cached data in the background, correcting any drift it detects.

The practical consequence for a client: watch `Cache-Control` and revalidate with
conditional requests, rather than polling on a fixed interval in the hope that a
timer has elapsed. Polling faster than the data can change returns nothing new and
only spends rate-limit budget.

# Citations

[1] Cache headers and conditional requests —
`https://developers.eveonline.com/docs/services/esi/best-practices/`.

[2] Event-driven invalidation and the role of `Cache-Control` versus `Expires` —
`https://developers.eveonline.com/blog/smarter-caching-when-events-drive-invalidation`.
