---
type: Convention
title: Write in American English
description: This repository is written in American English throughout — "color", "armor", "license", never the British spellings.
tags: [okf, authoring, style, language]
timestamp: 2026-07-14T12:00:00Z
---

# The principle

This repository is written in **American English** — page prose and titles alike.
When you name or describe something, reach for the American spelling; a page about
ship paint says **color**, even if a British author would reach for "colour" by
habit.

# Why

- **Consistency across the bundle.** One spelling convention keeps pages
  predictable and searchable for humans and LLMs alike — a reader (or a retrieval
  query) hits the same word every time.
- **Quoted identifiers are read literally.** When a page quotes a field name or
  value from a system it documents, it must match the source exactly. Writing in
  American English by default keeps a documented `color` from drifting to
  `colour`.

# Common cases

| Use (American) | Not (British) |
|----------------|---------------|
| `color`        | `colour`      |
| `armor`        | `armour`      |
| `center`       | `centre`      |
| `license` (noun & verb) | `licence` |
| `catalog`      | `catalogue`   |
| `-ize` (`customize`, `organize`) | `-ise` |

# In practice

- Write your own prose in American English throughout.
- Quote identifiers exactly as the system defines them. Because the systems this
  repository documents are themselves American-English, following the rule when
  authoring is what keeps those identifiers American in the first place.
