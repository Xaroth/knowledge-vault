---
type: Tool
title: eve-resfile-proxy
description: A local HTTP proxy that resolves the EVE resfile indexes into a browsable read-only filesystem, serving CDN assets by path with directory listings, ETags, conditional requests, optional aliasing and read-time transforms, and a launcher-compatible disk cache.
tags: [eve-online, resfile, tooling, architecture]
timestamp: 2026-07-18T21:40:00Z
---

# Overview

`eve-resfile-proxy` is a local HTTP server that turns the [resource distribution](../index.md)
system into an ordinary read-only filesystem. It resolves the current build,
loads the [index files](../index-format.md), and serves any asset by path — `GET
/ui/texture/icons/icons111_07.png` — with directory listings, ETags, conditional
requests, and an optional disk cache. It reads all six index columns, using the
MD5 for ETags and the sizes for listings and content-length.

The module is `github.com/eve-online-tools/eve-resfile-proxy`.

# Virtual filesystem

The proxy composes the resource tree from stacked `fs.FS` layers, innermost first:

```
manifest  — the CDN-backed resfile index, fetched lazily per file
  → alias      — optional path and extension aliases (off unless configured)
    → transform — optional read-time transforms (off unless configured)
```

* **Manifest layer.** Backs each logical path with a lazily-fetched CDN blob.
  `Stat` and directory listings read only index metadata, so they never fetch bytes;
  directories are synthesized from the flat index by scanning path prefixes.
* **Mux (overlay).** Multiple manifests are overlaid at mount points — the full and
  OS-specific resfile indexes at the resource root, and (in full-tree mode) the
  binaries manifests under `/app/<platform>`. Earlier mounts win on key collision;
  listings merge across mounts.
* **Alias layer.** Optional path and extension rewrites (e.g. `favicon.ico` →
  `ui/texture/icons/icons111_07.png`), resolved iteratively so rules can stack. A
  real underlying file always wins over an alias.
* **Transform layer.** Optional read-time transforms (external command or WASM
  module) applied to matching paths, with their outputs cached separately.

Paths are cleaned and lowercased before lookup, matching the lowercased index keys.

# HTTP surface

The server (default `:8080`) runs a middleware chain
`heartbeat → method → index → load → conditional → respond`:

| Endpoint | Behavior |
| --- | --- |
| `GET /healthz`, `GET /livez` | liveness — `200 ok` |
| `GET\|HEAD /<path>` | the asset bytes; other methods → `405` |
| `GET /<path>/` | HTML directory listing (nginx-style) when index listing is enabled |

A missing path returns `404`; an upstream CDN/fetch failure surfaces as `502`.
Responses carry `Content-Type` (extension-derived), `Cache-Control: public,
max-age=3600`, an `ETag` (the manifest MD5), `X-Eve-Build` (current build number),
and `X-Cache-Status: HIT|MISS`. Conditional requests (`If-None-Match`,
`If-Modified-Since`) return `304`. `HEAD` returns headers only, using the index's
size for `Content-Length` without fetching bytes.

# Caching

With a cache directory configured, a read-through cache stores raw CDN bytes keyed
by CDN cache path — the same layout the EVE launcher's `ResFiles` cache uses, so the
directories can be shared. Caching is **off by default**. Transform outputs are
cached separately under `_transformed/<ruleName>/<cdnPath>` with an MD5 sidecar.

# Configuration

Configured by YAML file and/or CLI flags (flags override the file):

| Key | Meaning |
| --- | --- |
| `server` | EVE server name (default `tranquility`) → drives build discovery |
| `build` | pin a specific build; omit for latest + 5-minute auto-update |
| `platforms` | restrict loaded platforms (e.g. `windows`, `macos`) |
| `cache` | on-disk cache directory (enables caching) |
| `addr` | HTTP listen address (default `:8080`) |
| `full_tree` | expose resources under `/res/` and binaries under `/app/` |
| `no_index` / `index_listing` | toggle directory listings (on by default) |
| `aliases` | path / extension alias rules |
| `transforms`, `transform_limits` | read-time transform rules and resource limits |

Removed legacy keys (`rewrites`, `aliases.paths`, `aliases.extensions`, and the
`from`/`to` alias field names) are rejected with an explicit error.

# Fetch behavior

CDN requests use a shared HTTP client (1-minute timeout, `User-Agent:
github.com/eve-online-tools/eve-resfile-proxy`) with retries (up to 3, exponential
backoff from 250ms, honoring `Retry-After`) on 429/5xx, a concurrency cap of 4, and
a response-body size limit. Checksum verification against the index MD5 is available
but disabled in the running proxy.

# Citations

[1] `eve-resfile-proxy` — `README.md`, `service/service.go`, `service/manifest/loader.go`,
`vfs/` (`manifest.go`, `fs.go`, `mux/`, `alias/`, `transform/`, `fetch/`),
`service/http/` (`server.go`, middleware, `handler/respond.go`),
`cache/`, `cmd/eve-resfile-proxy/`, `example-config.yaml`.
