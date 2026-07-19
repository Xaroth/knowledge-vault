---
type: Reference
title: ESI — pagination
description: A large ESI collection is paged by one of three models — opaque cursor tokens (before/after), a 1-based page number with an X-Pages count, or a from-ID walk backward through record IDs — each with its own duplicate-handling rule.
tags: [eve-online, esi]
timestamp: 2026-07-19T11:25:00Z
---

# Three pagination models

A route that returns a large collection pages it with one of three models. Which a
route uses is declared in the [OpenAPI contract](./versioning.md). All three can
surface **duplicate records** across requests when a record changes between calls,
so each defines how a client reconciles them.

# Cursor-based

Records are ordered by last-modification date, most recent last. The client walks
them with opaque **cursor tokens** that mark a position and must be treated as
black boxes — never decoded or constructed.

* `limit` — the maximum records to return (fewer may come back).
* `after` — return records following a token.
* `before` — return records preceding a token.
* Omitting both returns the most recent records.

A response carries a cursor object with `before` and `after` tokens alongside the
data. Duplicate handling: on a `before` walk, keep the record already stored; on an
`after` walk, replace it with the newer version.

# Page-numbered (`X-Pages`)

The client requests a **1-based `page`** parameter. The response header
**`X-Pages`** gives the total number of pages; the client iterates from 1 through
that count.

Because cache expiry can fall between page fetches and produce duplicates, a client
checks how close page 1 is to its cache expiry and, if it is near, waits for the
refresh before fetching the remaining pages.

# From-ID

The client passes a **`from_id`** — the ID of a record — and receives that record
plus **older** records; records are ordered most-recent-first. Omitting `from_id`
returns the most recent records. The walk ends when a response contains only the
`from_id` record itself (a single record); an empty response means there are no
records.

# Citations

[1] Cursor-based, X-Pages, and from-ID pagination —
`https://developers.eveonline.com/docs/services/esi/pagination/cursor-based/`,
`https://developers.eveonline.com/docs/services/esi/pagination/x-pages/`,
`https://developers.eveonline.com/docs/services/esi/pagination/from-id/`.
