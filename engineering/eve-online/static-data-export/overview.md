---
type: Reference
title: Static Data Export — overview
description: The SDE is EVE's published dump of static game data, generated from the game client, versioned per Tranquility build, and distributed as JSON Lines and YAML archives from the EVE developer site.
tags: [eve-online, sde, static-data, overview]
timestamp: 2026-07-19T10:00:00Z
---

# What the SDE is

The **Static Data Export (SDE)** is CCP's published snapshot of EVE Online's
**static game data** — the reference tables that change only with a game update,
never during play. It covers item types and their attributes, groups and
categories, dogma, the New Eden map, factions, NPCs, localization, and more.

The SDE exists so third-party tools do not have to scrape or hard-code this data:
a tool downloads the export, keyed to a known build, and reads it directly. It is
the static counterpart to the live ESI API — where ESI serves dynamic,
per-character state, the SDE serves the fixed world definitions behind it.

# How it is built

The SDE is generated from the data already present in the game client. A single
mapping file controls which client data is exported and how it is structured, which
keeps tightly-scoped control over what is included — data that is not shipped to the
client is not in the SDE. A build is produced for a Tranquility deployment, so an
SDE build number corresponds to a client build.

# The two formats

The SDE is published in two interchangeable formats carrying the same data:

| Format | Notes |
| --- | --- |
| **JSON Lines** (`jsonl`) | One JSON object per line. Keys must be strings, so integer-keyed datasets use a `_key` / `_value` convention (see [record format](./record-format.md)). |
| **YAML** (`yaml`) | Supports integer keys natively. Large datasets such as `mapMoons` are memory-intensive and slow to parse. |

JSON Lines is the streaming-friendly format: a consumer can read it line by line
without holding a whole dataset in memory, which matters for the largest datasets.

# How it is distributed

Everything is served over HTTPS from the EVE developer site under
`https://developers.eveonline.com/static-data/tranquility/`:

* a small **manifest** (`latest.jsonl`) naming the current build,
* one **archive** per build and format (a `.zip` of per-dataset files),
* a **shorthand** URL per format that always resolves to the latest build,
* per-build **change** files and a **schema changelog**.

`tranquility` in the path is EVE's production server; the SDE is published for that
server. See [versioning and discovery](./versioning-and-discovery.md) for the exact
URLs and the caching model.

# Citations

[1] EVE developer documentation — Static Data:
`https://developers.eveonline.com/docs/services/static-data/`.

[2] `@eve-online-tools/eve-sde` — `src/constants.ts` (origin and URL builders).
