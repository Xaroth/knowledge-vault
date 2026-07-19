---
type: Tool
title: eve-sde
description: A build-time loader that resolves an SDE build, downloads and caches the JSON Lines archive, parses only the requested datasets, and runs processors that transform them into small generated artifacts, with Vite and Rollup plugins and a lock file that skips unchanged builds.
tags: [eve-online, sde, static-data, tooling]
timestamp: 2026-07-19T10:00:00Z
---

# Overview

`@eve-online-tools/eve-sde` downloads and processes the [SDE](../index.md) at
**build time**, so an app ships only the small slices of static data it needs
rather than the full export. It resolves a build, downloads and caches the archive,
extracts and parses selected datasets, and either dumps them as JSON or runs
custom **processors** that transform them into generated artifacts.

It consumes the **JSON Lines** distribution only, from the **`tranquility`** server;
it does not read the YAML, CSV, or sqlite variants. It is published as
`@eve-online-tools/eve-sde` (MIT) from `packages/eve-sde` in the `node-packages`
monorepo, and is used by other packages in that repo.

# Build resolution and download

The loader either uses a pinned `buildNumber` or reads the current one from the
[`latest.jsonl` manifest](../versioning-and-discovery.md) (the `sde` record). It
then downloads the archive from
`https://developers.eveonline.com/static-data/tranquility/eve-online-static-data-<buildNumber>-jsonl.zip`.

The only integrity check is the ZIP magic-byte signature (`PK\x03\x04`); no
checksum or hash is verified. A corrupt archive is deleted and re-downloaded. The
fetch layer retries transient failures (default 3 retries on 502/503/504, with
backoff and a 60-second timeout).

# Caching

Downloaded and derived data is cached per build under `.cache/eve-sde/<buildNumber>/`:

```
.cache/eve-sde/<buildNumber>/
  archive.zip                     the downloaded SDE archive
  raw/<dataset>.jsonl             a dataset extracted verbatim
  parsed/<dataset>.<format>.json  the parsed, keyed dataset
```

A dataset is loaded from the in-memory cache, then the parsed cache, then extracted
and parsed from the archive on a miss. The parsed cache key encodes the JSON
formatting so a change in output spacing does not reuse a stale blob.

# Reading datasets

Datasets are addressed by **bare name** (no extension); passing `types.jsonl` is an
error. Two access models are offered:

* **Whole-dataset** — `load(dataset)` materializes the full parsed dataset as a
  keyed object. Suitable for small datasets.
* **Streaming** — `loadStream(dataset)` yields one row at a time and never holds the
  whole dataset in memory. Suitable for large datasets such as `mapMoons`.

Rows follow the SDE [record format](../record-format.md): object rows and
`_key` / `_value` rows are preserved through parsing and re-serialization, so
generated output stays shape-compatible with the source.

# Processors and generated output

A **processor** (`{ id, version, run(ctx) }`) reads datasets and writes generated
artifacts into the output directory. Its context exposes dataset loading, streaming
readers and writers, path resolution, and access to earlier processors' output, so
processors can chain.

The built-in **`stripFields`** processor trims a dataset down: it filters rows by
key, deletes named fields, and reduces [translation dictionaries](../record-format.md)
to a chosen set of languages (with a required fallback), re-emitting the dataset in
SDE JSON Lines shape. Its `version` is derived from its options, so changing what it
strips automatically invalidates cached output.

# Skip-unchanged builds

After a run the loader writes a lock file (`.sde-lock.json`) recording the build
number, the datasets dumped, and the processors and their versions. On the next run
it regenerates only when that fingerprint differs (or `force` is set) — an unchanged
build, dataset set, and processor set is skipped entirely.

# Integration

* **Vite** and **Rollup** plugins wrap the same core `processSde` pipeline and run it
  during the build. The Rollup plugin needs an explicit project `root` for relative
  paths, since Rollup does not supply one.
* A slim `/processors` entry point exposes just the JSON Lines parsing helpers, so an
  app's runtime can read the generated artifacts without pulling in the download and
  Node-only code.

# Citations

[1] `packages/eve-sde/README.md`, `ARCHITECTURE.md`, and `src/` — `constants.ts`,
`build-resolver.ts`, `archive.ts`, `cache.ts`, `fetch.ts`, `load-table.ts`,
`stream-table.ts`, `processor.ts`, `fingerprint.ts`, `processors/` (`parse-jsonl.ts`,
`strip-fields.ts`), `translation-dict.ts`, the `vite/` and `rollup/` entries.
