---
type: Reference
title: Static Data Export — record format
description: Each dataset is a keyed collection; in JSON Lines every line is one object carrying a _key, with integer-keyed rows using the _key / _value pair and object rows carrying their fields inline, while YAML keeps native keys and translation dictionaries hold localized strings.
tags: [eve-online, sde, static-data]
timestamp: 2026-07-19T10:00:00Z
---

# Overview

Every dataset is a **keyed collection** — a map from a row key (usually an integer
ID) to a row value. The two distribution formats encode that map differently:
YAML keeps native keys, while JSON Lines works around JSON's string-only keys with
a small convention.

# JSON Lines rows

In the `jsonl` variant each dataset file has **one JSON object per line**, and every
line carries a **`_key`** field holding the row's key. Because JSON object keys must
be strings, integer-keyed datasets cannot be a single JSON object; instead each row
is its own line and the integer key rides in `_key`.

A row takes one of two shapes:

* **Object row** — the row's value is a record. All fields other than `_key` are the
  value:

  ```json
  {"_key": 587, "groupID": 25, "name": {"en": "Rifter"}}
  ```

* **Value row** — the row's value is a scalar or array, carried in a single
  `_value` field alongside `_key`:

  ```json
  {"_key": 582, "_value": [1, 2, 3]}
  ```

A line with exactly `_key` and `_value` is a value row; any other line is an object
row whose value is its remaining fields. Keys may be integers or strings.

# YAML rows

The `yaml` variant represents each dataset as a native YAML mapping with integer or
string keys directly — there is no `_key` / `_value` indirection. The data is
identical; only the encoding differs. YAML is convenient for hand-inspection but
memory-intensive and slow to parse for the largest datasets (for example
`mapMoons`), which is the motivation for the streaming-friendly JSON Lines variant.

# Translation dictionaries

Localized text is a **translation dictionary**: an object keyed by language code
whose values are the translated strings, always including English under `en`:

```json
{"en": "Rifter", "de": "Rifter", "fr": "Rifter", "ja": "リフター"}
```

These appear as field values inside object rows (a `name` or `description` field,
for example). A consumer that needs only some languages can drop the rest, keeping
`en` as a fallback.

# Citations

[1] EVE developer documentation — Static Data (integer-key `_key` / `_value`
handling, YAML integer keys, `mapMoons`):
`https://developers.eveonline.com/docs/services/static-data/`.

[2] `@eve-online-tools/eve-sde` — `src/processors/parse-jsonl.ts` (object vs value
rows), `src/translation-dict.ts` (translation-dictionary detection).
