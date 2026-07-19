---
type: Reference
title: Static Data Export — datasets
description: The SDE archive holds one file per dataset named by the dataset (types, groups, dogmaAttributes, map-prefixed map files, …); map data shares one metric coordinate system, and the data is structured to mirror what the client itself carries.
tags: [eve-online, sde, static-data]
timestamp: 2026-07-19T10:00:00Z
---

# Overview

An SDE archive is a flat set of **per-dataset files**, one file per dataset, named
after the dataset with the format's extension — `types.jsonl`, `groups.jsonl`,
`dogmaAttributes.jsonl`, and so on (`.yaml` in the YAML variant). A consumer reads
only the datasets it needs.

Dataset names are not a fixed enumeration in tooling; they are resolved against the
archive's contents at read time. The set below is representative, not exhaustive.

# Representative datasets

| Dataset | Holds |
| --- | --- |
| `types` | Item and ship type definitions. |
| `groups`, `categories` | The type taxonomy. |
| `dogmaAttributes`, `dogmaEffects`, `dogmaUnits` | The dogma system — attributes, effects, and their units. |
| `masteries`, `typeBonus` | Type-related data split out of `types`. |
| `factions`, `agentTypes` | NPC factions and agent types. |
| `dbuffCollections`, `dynamicItemAttributes` | Buff collections and abyssal/dynamic item attributes. |
| `translationLanguages` | The set of supported languages. |
| `map`-prefixed files | The New Eden map (see below). |

# Map data

Map data lives in the **`map`-prefixed** datasets — `mapRegions`,
`mapConstellations`, `mapSolarSystems`, `mapMoons`, and related files. Every level
of the map shares **one coordinate system**: positions are relative to the New Eden
cluster center, in meters (scale `1.0` = 1 meter), as an `{x, y, z}` object on each
row.

```json
{"_key": 30000142, "position": {"x": -9.06e16, "y": 6.37e16, "z": 1.17e17}}
```

Solar-system keys in the range **30,000,000–30,999,999** are the playable New Eden
cluster; other ranges cover wormhole space, abyssal space, and internal systems. The
same regions, constellations, and systems are also reachable live through ESI's
`/universe/…` routes.

# Data shape

The SDE is structured to mirror the data the client itself carries, exported through
a controlled mapping. Two consequences follow from that:

* **Only client-visible data is present.** A field the game does not ship to the
  client is not in the SDE.
* **The data is organized into the dataset files above**, rather than into separate
  bulk-data or universe trees. Map and universe information is part of the
  `map`-prefixed files, following the same keyed-row structure as every other
  dataset.

# Citations

[1] EVE developer documentation — Static Data and the Map Data guide (dataset
files, map coordinate system, New Eden ID range):
`https://developers.eveonline.com/docs/services/static-data/`,
`https://developers.eveonline.com/docs/guides/map-data/`.

[2] `@eve-online-tools/eve-sde` — `README.md` (dataset examples: `types`, `groups`,
`dogmaAttributes`, `mapMoons`, `factions`, `translationLanguages`).
