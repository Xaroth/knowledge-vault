---
type: Reference
title: Resource distribution — CDN cache paths
description: Assets are stored on the resources host under content-addressed, hash-named cache paths with a two-hex-digit shard directory; the MD5 in the index verifies bytes and drives ETags, and the layout matches the launcher's ResFiles cache.
tags: [eve-online, resfile, cdn]
timestamp: 2026-07-18T21:40:00Z
---

# Overview

The cache path in column 1 of an index entry is where an asset's bytes actually
live on the resources host. These paths are **content-addressed**: the filename is
derived from a hash of the contents, so a given path is immutable and a patch ships
as new paths rather than mutated ones.

# Cache path shape

A cache path shards by the first two hex digits of the hash and names the file by
hash and checksum:

```
<xx>/<hash>_<checksum>
```

For example:

```
7d/7d87a0a3a100cf9f_00b8308223bd89d4db39914d2c6488a3
```

The leading `7d` is the shard directory (the first two hex digits of the hash). The
consumer never *constructs* this path — it reads it verbatim from the index and
appends it to the resources host:

```
https://resources.eveonline.com/7d/7d87a0a3a100cf9f_00b8308223bd89d4db39914d2c6488a3
```

# Checksums and sizes

The index carries an optional **MD5** (32 hex characters) and two byte sizes
(decompressed and compressed) alongside each cache path.

* **Verification.** The MD5 lets a consumer confirm downloaded bytes match the
  index. The proxy can verify on read (comparing `md5.Sum(bytes)` to the entry's
  MD5) but leaves this **off by default**; the bundler package does not verify.
* **ETags.** The proxy serves the MD5 as a strong ETag, so conditional requests
  (`If-None-Match`) can return `304 Not Modified` without re-reading the blob.
* **Sizes.** The decompressed size answers `Content-Length` and `HEAD` without
  fetching bytes; the compressed size drives the compression-ratio column in
  directory listings.

An entry with no MD5 (a zero digest) is treated as "unset" — verification is
skipped and the ETag falls back to hashing the fetched body.

# Launcher-compatible on-disk cache

Because the cache path *is* the storage key, a downloaded blob can be stored under
that exact relative path on disk. This is the same layout the official EVE launcher
uses for its `ResFiles` cache, so a tool's cache directory and the launcher's can
be one and the same — a blob fetched by one is a cache hit for the other.

Both tools cache raw CDN bytes keyed by the CDN cache path. Because the key is a
content hash, a new asset version lands at a new key automatically; stale entries
are never overwritten in place, so the cache needs no invalidation.

# Content types

Assets are served with a MIME type derived from the logical path's extension. The
map covers the usual image, audio, video, and font types plus EVE-specific cases —
notably `.dds` textures (`image/vnd-ms.dds`) — and falls back to
`application/octet-stream` for unknown extensions.

# Citations

[1] `eve-resfile-proxy` — `vfs/fs.go` (checksum validation), `vfs/internal/md5/md5.go`
(digest parsing), `service/http/middleware/conditional/conditional.go` (ETags),
`vfs/fetch/cache/cache.go` and `cache/disk/disk.go` (byte cache keyed by CDN path),
`README.md` (launcher `ResFiles` compatibility, cache-path example).

[2] `@eve-online-tools/eve-resfile` — `src/content-type.ts` (MIME map, `.dds`),
`src/cache.ts` (per-build cache).
