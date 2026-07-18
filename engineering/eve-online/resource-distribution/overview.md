---
type: Reference
title: Resource distribution — overview
description: EVE Online publishes client assets across two CDN hosts as content-addressed blobs, resolved from logical res:/ and app:/ paths through a chain of plain-text index files keyed by client build number.
tags: [eve-online, resfile, cdn, overview]
timestamp: 2026-07-18T21:40:00Z
---

# What the system is

The EVE Online client does not carry its assets inline. Icons, textures, sounds,
models, and even the client's own distribution files live on two Amazon-fronted
CDN hosts as immutable, content-addressed blobs. A running client — or any tool
that wants a specific asset — resolves a human-readable **logical path** to a
CDN **cache path** by walking a chain of plain-text index files that are
themselves pinned to a specific client **build number**.

Because every blob is named after a hash of its contents, an asset URL is
permanent: a given cache path always returns the same bytes. Only the index files
change between builds, remapping logical paths to whichever blob is current.

# The two CDN hosts

| Host | Serves | Role |
| --- | --- | --- |
| `https://binaries.eveonline.com` | build metadata and index files | the *what and where* — the maps |
| `https://resources.eveonline.com` | the asset blobs themselves | the *bytes* — content-addressed storage |

Both tools hard-code these as defaults and validate that any override stays on an
allowed host (HTTPS-only, no private addresses).

# Logical path namespaces

Logical paths are URI-style, with a scheme prefix that names the namespace:

* **`res:/…`** — game resources: `res:/ui/texture/icons/icons111_07.png`,
  `res:/dx9/model/ship/...`. This is the namespace application code references.
* **`app:/…`** — application/distribution files: the client binaries and the
  resfile index files themselves. `app:/resfileindex.txt` is the entry that a
  binaries manifest uses to point at the resource index.

Logical paths are matched **case-insensitively**; both tools lowercase them when
building their lookup keys.

# The resolution chain

Turning a logical `res:/` path into downloadable bytes takes four hops:

```
eveclient_<SERVER>.json          → buildNumber                     (binaries host)
eveonline_<buildNumber>.txt      → find app:/resfileindex.txt       (binaries host)
  (the per-platform "binaries manifest")   → its CDN path
resfileindex.txt                 → map every res:/… → cache path    (binaries host)
  (the "resfile index")
https://resources.eveonline.com/<cachePath>  → the asset bytes     (resources host)
```

1. **Build discovery.** A server name (`tranquility`) resolves to a client build
   number via `eveclient_<SERVER>.json`. See [build discovery](./build-discovery.md).
2. **Binaries manifest.** `eveonline_<build>.txt` lists the distribution files for
   that build, including the resfile index. The consumer scans it for the
   `app:/resfileindex.txt` entry and reads that entry's CDN path.
3. **Resfile index.** The resfile index (`resfileindex.txt`) is the large map:
   one line per resource, logical `res:/` path to CDN cache path. See
   [index file format](./index-format.md).
4. **Asset fetch.** The asset is fetched from the resources host at the cache
   path. Cache paths are content hashes; see [CDN cache paths](./cdn-cache.md).

The index files at hops 2 and 3 are the same comma-delimited text format, differing
only in which namespace prefix they carry and what they list.

# Why it is shaped this way

* **Immutable assets, mutable maps.** Content addressing lets the CDN and every
  client cache treat asset URLs as permanent. A patch ships as a new set of index
  files, not as mutated asset URLs.
* **Build-pinned.** Every index is scoped to one build number, so a client resolves
  a self-consistent set of assets and never mixes versions.
* **Launcher-compatible cache.** Because the cache path *is* the storage key, a
  downloaded blob can sit in the same on-disk layout the official launcher uses for
  its `ResFiles` cache — tools can share that directory.

# Citations

[1] `@eve-online-tools/eve-resfile` source — `packages/eve-resfile/src/constants.ts`,
`index-loader.ts`, `parse.ts`.

[2] `eve-resfile-proxy` source — `common/domain/domain.go`,
`vfs/manifest.go`, `service/manifest/loader.go`.
