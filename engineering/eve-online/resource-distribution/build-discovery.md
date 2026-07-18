---
type: Reference
title: Resource distribution — build discovery
description: A server name resolves to a client build number through eveclient_<SERVER>.json on the binaries host; that build number scopes every index file, and unpinned consumers poll for build changes.
tags: [eve-online, resfile, cdn]
timestamp: 2026-07-18T21:40:00Z
---

# Overview

Every index file is scoped to one client **build number**, so the first step in
resolving any asset is finding the current build. A consumer starts from a
**server name** and reads a small JSON manifest that names the build.

# The build manifest

The build manifest lives on the binaries host at `eveclient_<SERVER>.json`, where
`<SERVER>` is the server's short code, uppercased:

| Server name | Short code | Manifest URL |
| --- | --- | --- |
| `tranquility` (production) | `TQ` | `https://binaries.eveonline.com/eveclient_TQ.json` |
| `singularity` (test) | `SISI` | `https://binaries.eveonline.com/eveclient_SISI.json` |

`tranquility` is the default. An unrecognized server name is passed through
uppercased rather than rejected, so other environments can be reached by name.

## Fields

The proxy reads four fields from the JSON; the bundler package reads only
`buildNumber`:

| Field | Type | Meaning |
| --- | --- | --- |
| `buildNumber` | string or number | The current build. **Preferred** identifier. |
| `build` | string | Legacy build identifier, superseded by `buildNumber`. |
| `protected` | bool | When true, the build requires an SSO token and cannot be served anonymously. |
| `platforms` | list | Platforms the build publishes (never includes Windows explicitly). |

A build number may arrive as a JSON number or string; consumers coerce it to a
string. `buildNumber` is the identifier going forward; `build` is the older form.

# Protected builds

When `protected` is true, the build's assets sit behind SSO authentication and
cannot be fetched anonymously. The proxy treats a protected build as a hard error
(it has no token to present) rather than serving partial results.

# Platforms

The `platforms` field lists the non-Windows platforms a build publishes (for
example macOS). Windows is always implicitly available and is appended
unconditionally by the proxy, so the effective platform set is
`{windows} ∪ platforms`. A consumer may restrict itself to a subset; the working
set is the intersection of the build's platforms with the configured ones, and an
empty intersection is an error.

Platform choice selects which binaries manifest and resfile index filenames are
read — see [index file format](./index-format.md).

# Pinning versus auto-update

* **Pinned.** A consumer may pin an explicit build number in configuration; the
  build manifest is then skipped entirely and the pinned build is used directly.
* **Latest.** With no pin, the consumer resolves the current build from the manifest.
  The proxy additionally polls the manifest on a fixed interval (every 5 minutes) and,
  when the build changes, reloads all index files and updates the build number it
  reports on responses. Pinning a build disables this watcher.

# Citations

[1] `eve-resfile-proxy` — `service/clientbuild/clientbuild.go` (URL construction,
`ClientBuild` struct), `service/service.go` (`resolveBuild`, platform intersection),
`service/watch.go` (5-minute poll).

[2] `@eve-online-tools/eve-resfile` — `src/constants.ts` (`eveClientManifest =
'eveclient_TQ.json'`), `src/parse.ts` (build-number parse), `src/index-loader.ts`
(`resolveBuildNumber`).
