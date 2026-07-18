---
type: Tool
title: eve-resfile
description: A build-tooling package with Vite, Rollup, and PostCSS plugins that resolve res:/ imports to EVE CDN assets — proxied on demand in dev, fetched and emitted as static bundle assets in production.
tags: [eve-online, resfile, tooling]
timestamp: 2026-07-18T21:40:00Z
---

# Overview

`@eve-online-tools/eve-resfile` lets a web project reference EVE client assets
through `res:/` import specifiers — `import iconUrl from "res:/ui/texture/icons/icons111_07.png"`
or `url(res:/…)` in CSS. It downloads and caches the [resfile index](../index-format.md),
then ships bundler plugins that resolve those imports. It reads only the logical
path and CDN cache-path columns of the index — its concern is URL resolution, not
verification.

It is published as `@eve-online-tools/eve-resfile` (MIT) from `packages/eve-resfile`
in the `node-packages` monorepo.

# Two runtime modes

| Phase | JS `import x from "res:/…"` | CSS `url(res:/…)` |
| --- | --- | --- |
| **Dev (Vite)** | virtual module → local proxy URL `/__eve_res__/…` | PostCSS rewrites to the proxy URL |
| **Production** | asset fetched at build time and emitted into the bundle | rewritten to the emitted asset's relative path |

* **Dev** never writes assets to disk. A Vite middleware serves
  `/__eve_res__/<encoded res path>` by looking the path up in the index and
  streaming the blob from the resources host on demand, with a one-day
  `Cache-Control`.
* **Production** fetches each referenced asset from the CDN at build time and emits
  it as a static bundle asset; JS and CSS then reference it by relative URL. A
  concurrency gate (default 8) throttles build-time fetches.

# The index load

The package performs the full resolution chain — build manifest → binaries manifest
→ resfile index — and caches the parsed map per build number as versioned JSON:

```
.cache/eve-resfile/<buildNumber>/
  build-index.txt       the raw binaries manifest
  resfileindex.txt      the raw resfile index
  resfile-map.json      { version, buildNumber, entries: { resPath: cdnPath } }
```

On read, a cache whose schema version or build number does not match is discarded
and rebuilt. Writes are atomic (temp file plus rename).

`loadResfileIndexData(options?)` exposes this loader as a standalone library for
scripts and tooling that do not need a bundler plugin.

# Configuration surface

Options (with defaults) include the CDN origins (`indexOrigin =
binaries.eveonline.com`, `assetOrigin = resources.eveonline.com`), an `allowedHosts`
allowlist, an optional pinned `buildNumber`, `cacheDir`, output directories
(`distDir`, `assetsDir`), fetch tuning (`fetchTimeoutMs`, `fetchConcurrency`), and
`missingResPath` behavior (`throw` or `warn-and-empty`). When a `res:/` path is
missing or a fetch fails under `warn-and-empty`, JS modules resolve to an empty
string.

## Security

Origins are validated at option resolution: HTTPS only (HTTP allowed only for
allowlisted `localhost`), host must be in `allowedHosts`, and private / link-local
addresses are rejected. This limits SSRF risk when origins come from untrusted
config. The fetch layer retries transient failures (up to 3 retries, exponential
backoff, honoring `Retry-After` on 429) but never retries a 404.

# Where each concern lives

| Concern | Module |
| --- | --- |
| Index download, cache, parse | `index-loader.ts`, `cache.ts`, `parse.ts` |
| `res:/` path normalization | `lookup.ts` |
| Vite JS resolution | `vite/plugin.ts` |
| Dev proxy middleware | `dev-middleware.ts` |
| Rollup asset emission | `asset-emit.ts`, `rollup/plugin.ts` |
| CSS `url(res:/…)` rewriting | `postcss/plugin.ts` |
| Shared handle across adapters | `integration-factory.ts` |

The Vite adapter emits assets via Rollup's `emitFile` during `load`; the Rollup
adapter, which cannot always emit companion assets during `load`, instead returns
sentinel strings and rewrites them to relative URLs in `writeBundle`.

# Citations

[1] `packages/eve-resfile/README.md`, `ARCHITECTURE.md`, and `src/` — `constants.ts`,
`index-loader.ts`, `parse.ts`, `cache.ts`, `lookup.ts`, `fetch.ts`,
`origin-validation.ts`, `content-type.ts`, the `vite/`, `rollup/`, `postcss/` entries.
