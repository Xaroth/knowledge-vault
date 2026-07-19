---
type: Reference
title: Static Data Export — versioning and discovery
description: The SDE is versioned by an integer build number; the latest.jsonl manifest names the current build in its "sde" record, archives are fetched by build number and format, and all resources support ETag / Last-Modified with a short cache.
tags: [eve-online, sde, static-data]
timestamp: 2026-07-19T10:00:00Z
---

# Overview

The SDE is versioned by an integer **build number**. Every archive, change file,
and manifest is scoped to one build. A consumer either pins a build number or
discovers the current one from a small manifest, then fetches the matching archive.

# The manifest

The current build is named in a one-line JSONL manifest:

```
https://developers.eveonline.com/static-data/tranquility/latest.jsonl
```

It is an SDE-shaped JSONL file (see [record format](./record-format.md)). The
current build lives in the record whose `_key` is `sde`:

```json
{"_key": "sde", "buildNumber": 3436472, "releaseDate": "2026-07-17T11:04:22Z"}
```

A consumer reads the `sde` record's `buildNumber` to learn the latest build, and
`releaseDate` (ISO 8601) for when it was published. The build number here is
illustrative — it advances with each release.

# Downloading a build

Each build and format is a single `.zip` archive of per-dataset files:

```
https://developers.eveonline.com/static-data/tranquility/eve-online-static-data-<buildNumber>-<variant>.zip
```

`<variant>` is `jsonl` or `yaml`. Two **shorthand** URLs always resolve to the
latest build without first reading the manifest:

```
.../static-data/tranquility/eve-online-static-data-latest-jsonl.zip
.../static-data/tranquility/eve-online-static-data-latest-yaml.zip
```

Pinning an explicit build number gives a stable, reproducible download; the
shorthand latest URLs float forward as new builds publish.

# HTTP caching

All resources support `ETag` and `Last-Modified`, so conditional requests
(`If-None-Match`, `If-Modified-Since`) return `304 Not Modified` when a build is
unchanged. Non-static resources (the manifest and the latest-shorthand redirects)
are cached for about 5 minutes; the build-numbered archives are immutable, so they
can be cached indefinitely.

# Citations

[1] EVE developer documentation — Static Data (manifest, download URLs, caching):
`https://developers.eveonline.com/docs/services/static-data/`.

[2] `@eve-online-tools/eve-sde` — `src/build-resolver.ts` (reads the `sde` record
from `latest.jsonl`), `src/constants.ts` (`sdeLatestUrl`, `sdeZipUrl`).
