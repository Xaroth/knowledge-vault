# Static Data Export

The Static Data Export (SDE) is CCP's published dump of EVE Online's **static game
data** — the slowly-changing reference tables that only change with a game update:
item types, groups, dogma attributes, the map, factions, and more. It is
distributed as downloadable archives from the EVE developer site, versioned by
build number, in JSON Lines and YAML.

These pages describe the SDE as it is published and consumed: its distribution and
versioning, the on-the-wire record format, the datasets it contains, how changes
between builds are tracked, and the first-party tool that loads it at build time.

# Reading order

* [Overview](./overview.md) - what the SDE is, the two formats, and how it is distributed and built.
* [Versioning and discovery](./versioning-and-discovery.md) - build numbers, the `latest.jsonl` manifest, download URLs, and HTTP caching.
* [Record format](./record-format.md) - the one-object-per-line JSONL row shape, the `_key` / `_value` integer-key convention, YAML differences, and translation dictionaries.
* [Datasets](./datasets.md) - the per-dataset file model, map data and its coordinate system, and the structural shape of the data.
* [Change tracking](./change-tracking.md) - the per-build `changes/<build>.jsonl` diffs and the schema changelog.

# Tools

* [Tools](./tools/index.md) - first-party consumers of the SDE: the `eve-sde` build-time loader.

# Source material

This area is derived from the EVE developer documentation and a first-party tool:

* SDE documentation and downloads —
  [developers.eveonline.com/docs/services/static-data](https://developers.eveonline.com/docs/services/static-data/)
  and [developers.eveonline.com/static-data](https://developers.eveonline.com/static-data).
* `@eve-online-tools/eve-sde` — `packages/eve-sde` in
  [github.com/eve-online-tools/node-packages](https://github.com/eve-online-tools/node-packages).
