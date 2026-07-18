---
type: Reference
title: Resource distribution — index file format
description: Two tiers of plain-text index share one comma-delimited line format — a binaries manifest lists distribution files (including the resfile index) and the resfile index maps every res:/ path to a CDN cache path.
tags: [eve-online, resfile, cdn]
timestamp: 2026-07-18T21:40:00Z
---

# Overview

Two tiers of index file drive resolution, and both use the **same line format**.
They differ only in what they list and which namespace prefix their entries carry.

1. **The binaries manifest** (`app:/…` entries) lists a build's distribution files
   — the client binaries and, crucially, the resfile index files themselves.
2. **The resfile index** (`res:/…` entries) maps every game resource to its CDN
   cache path.

Both are fetched from the binaries host and are plain, newline-delimited text.

# Tier 1 — the binaries manifest

The binaries manifest is named per platform, scoped to the build number:

| Platform | Manifest filename |
| --- | --- |
| Windows | `eveonline_<buildNumber>.txt` |
| macOS | `eveonlinemacOS_<buildNumber>.txt` |

A consumer reads it to find the resfile index. The resfile index is registered in
the manifest under a well-known logical path, and its actual CDN location is that
entry's cache-path column. The Windows entry is:

```
app:/resfileindex.txt
```

macOS nests the same files under the app bundle, e.g.
`app:/EVE.app/Contents/Resources/build/resfileindex.txt`.

## Resfile index roles

A build publishes more than one resfile index, by role:

| Role | Windows logical path |
| --- | --- |
| Full | `app:/resfileindex.txt` |
| OS-specific | `app:/resfileindex_Windows.txt` |
| Prefetch | `app:/resfileindex_prefetch.txt` |

The bundler package uses only the **Full** index (`app:/resfileindex.txt`). The
proxy mounts the **Full** and **OS-specific** indexes together, layering the
OS-specific entries over the full set; it defines but does not load the prefetch
index. When one is absent (for example on a platform that does not publish it) it
is skipped rather than treated as an error.

# Tier 2 — the resfile index

The resfile index is the large map: one line per resource, keyed by its `res:/`
logical path. Its CDN location comes from the binaries-manifest entry above; it is
fetched from the binaries host at that cache path and parsed into a lookup map.

# The line format

Both tiers share one **comma-delimited** line format. Lines are split on newlines,
trimmed, and blank lines skipped. Each line has up to six columns:

```
<logicalPath>,<cdnPath>,<md5hex>,<size>,<compressedSize>,<mode>
```

| # | Column | Required | Notes |
| --- | --- | --- | --- |
| 0 | logical path | yes | e.g. `res:/ui/texture/icons/icons111_07.png` or `app:/resfileindex.txt`. Stored **lowercased** as the lookup key. |
| 1 | CDN cache path | yes | The content-addressed path on the resources host. Preserved **as-is** (not lowercased). |
| 2 | MD5 | no | 32 hex characters; the content checksum. |
| 3 | size | no | Decompressed size in bytes. |
| 4 | compressed size | no | Compressed (on-CDN) size in bytes. |
| 5 | mode | no | Opaque mode string. |

Only the first two columns are required. A line with fewer than two columns, or
with an empty logical path or cache path, is skipped.

## What each consumer reads

The two tools consume different subsets of the columns:

* **`eve-resfile` (bundler package)** reads only columns 0 and 1 — the logical
  path and the cache path. It discards the checksum, sizes, and mode; its job is
  URL resolution, not verification.
* **`eve-resfile-proxy`** parses all six columns into an entry record. It uses the
  MD5 for ETags and the sizes for directory listings and content-length, and can
  optionally verify a downloaded blob against the MD5.

# Prefix handling and namespaces

An index entry's logical path is matched against a namespace prefix (`res:/` or
`app:/`). When the index is loaded for a given namespace, entries not carrying that
prefix are skipped, and the prefix is stripped to form the filesystem-style lookup
key. Entries are keyed by the lowercased, prefix-stripped path; on a duplicate key
the first occurrence wins.

Because keys are lowercased, lookups are case-insensitive: `res:/Icons/64/Icon.PNG`
and `res:/icons/64/icon.png` resolve to the same entry. A lookup may include a
`?query` or `#fragment`, which is stripped before matching.

# Citations

[1] `@eve-online-tools/eve-resfile` — `src/parse.ts` (`parseResfileIndex`,
`parseFirstTwoColumns`, `findResfileIndexCdnPath`), `src/constants.ts`
(`resfileIndexPath = 'app:/resfileindex.txt'`), `src/lookup.ts` (normalization).

[2] `eve-resfile-proxy` — `vfs/manifest.go` (`Entry`, `parseIndexLine`,
`parseManifest`, prefixes), `common/platform/platform.go` (`ManifestPath`),
`common/resource/resource.go` (Full / OS-specific / Prefetch roles).
