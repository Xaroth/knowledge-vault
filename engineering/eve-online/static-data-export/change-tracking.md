---
type: Reference
title: Static Data Export — change tracking
description: Each build has a changes/<build>.jsonl file whose _meta record links it to the previous build and whose per-dataset records list added, changed, removed, and changedLocalization keys, while schema-changelog.yaml records structural schema changes.
tags: [eve-online, sde, static-data]
timestamp: 2026-07-19T10:00:00Z
---

# Overview

Two resources describe how the data moves between builds: a per-build **change
file** listing which rows changed, and a **schema changelog** recording structural
changes to the datasets themselves. Together they let a consumer update
incrementally and adapt to shape changes without re-reading everything.

# Per-build changes

Each build publishes a change file:

```
https://developers.eveonline.com/static-data/tranquility/changes/<buildNumber>.jsonl
```

It is a JSONL file. Its first record, keyed `_meta`, links the build to its
predecessor:

```json
{"_key": "_meta", "buildNumber": 3436472, "lastBuildNumber": 3435006, "releaseDate": "2026-07-17T11:04:22Z"}
```

`lastBuildNumber` names the build this one is a diff against, so a consumer can walk
the chain of change files to catch up across several builds.

Every other record is keyed by a **dataset name** and lists the affected row keys by
kind of change:

```json
{"_key": "certificates", "changed": [81, 2235]}
{"_key": "types", "added": [88001], "changed": [587], "removed": [12345]}
{"_key": "missions", "changedLocalization": [1552, 4816]}
```

| Field | Meaning |
| --- | --- |
| `added` | Row keys new in this build. |
| `changed` | Row keys whose value changed. |
| `removed` | Row keys no longer present. |
| `changedLocalization` | Rows whose only change is translated text. |

A dataset with no changes in a build has no record in that build's change file.

# Schema changelog

Structural changes — datasets added or removed, fields renamed, split, or dropped —
are recorded in a single running file:

```
https://developers.eveonline.com/static-data/tranquility/schema-changelog.yaml
```

Where the per-build change files track which *rows* moved, the schema changelog
tracks how the *shape* of the datasets changed, so a consumer can tell a routine
data update apart from one that needs code changes.

# Citations

[1] EVE developer documentation — Static Data (`changes/<build>.jsonl`, the `_meta`
record and `lastBuildNumber`, `schema-changelog.yaml`):
`https://developers.eveonline.com/docs/services/static-data/`.
