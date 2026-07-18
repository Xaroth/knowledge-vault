---
type: Guide
title: Authoring OKF knowledge
description: The working know-how for writing OKF in this repository — the one hard rule, the conventions to apply, the knowledge lifecycle, and copy-paste templates.
tags: [okf, convention, guide, authoring]
timestamp: 2026-07-14T09:03:21Z
---

# Overview

OKF represents knowledge as a directory of markdown files with YAML
frontmatter. It is minimal by design: no schema registry, no runtime, no
SDK. Your job is to produce, maintain, and consume bundles **conformant
with the spec**, not your memory of it.

Always read the canonical [spec](./spec.md) before
non-trivial work. It is the verbatim OKF v0.1 specification and the source
of truth for every rule below.

# The one hard rule

A bundle is conformant (spec §9) iff: every non-reserved `.md` file has a
parseable YAML frontmatter block, and every such block has a **non-empty
`type`** field. Everything else is soft guidance. Consumers MUST tolerate
missing optional fields, unknown types, and broken links — never reject a
bundle over them.

# Conventions to apply

- **One concept = one file.** The file path (minus `.md`) is the concept ID.
- **Frontmatter:** `type` is required. Add `title`, `description`, `tags`,
  `timestamp` (ISO 8601) when they aid consumption; add `resource` (a
  canonical URI) only for concepts bound to a real asset — omit it for
  abstract concepts.
- **Body:** prefer structural markdown (headings, tables, lists, fenced
  code). Conventional headings: `# Schema`, `# Examples`, `# Citations`.
- **Cross-links:** standard markdown links in **relative** form, always
  written **explicitly** — a same- or child-directory link starts with
  `./` (`./spec.md`, `./api-route/index.md`) and a parent link with `../`
  (`../services/auth-api.md`). A bare `spec.md` is not allowed: Obsidian
  treats a bare target as a vault-wide wikilink lookup, not a
  path relative to the current file, so the link silently fails to
  resolve. A link to another directory targets its `index.md` explicitly
  (`./api-route/index.md`, not `./api-route/`) — Obsidian does not resolve
  a bare folder path to its index. The spec recommends absolute bundle-relative links
  (`/services/auth-api.md`, §5.1), but those break in GitHub and Obsidian —
  the two tools this repository is read in — because neither resolves a
  leading `/` against the bundle root. A link asserts a relationship; its
  *kind* lives in the surrounding prose, not the link.
- **Reserved files:** `index.md` (directory listing, no frontmatter —
  except the bundle-root index may carry only `okf_version`). The spec also
  reserves `log.md` for change history (§7), but this repository does not
  use it — git history is the change log. Never use these reserved names
  for concepts.

# Bundle location

This repository *is* the bundle: concepts live at the repo root and in
domain subdirectories, committed straight to `main` with git history as
the paper trail.
A standalone project that adopts OKF should use `.okf/` at its repository
root unless it already uses another location — knowledge as code.

# Lifecycle

OKF knowledge is produced, kept in sync with the things it describes, and
consumed as context.

- **Sources.** Concepts are derived from code (source, READMEs, docstrings,
  config), distilled from docs and wikis (with the originals linked under
  `# Citations`), or captured by hand (decisions, playbooks, metrics).
- **Organization.** Concepts are grouped into domain directories (e.g.
  `services/`, `datasets/`, `decisions/`), one concept per file, with an
  `index.md` per directory and `okf_version: "0.1"` on the root index.
- **Staying in sync.** When the thing a concept describes changes, its body
  and `timestamp` are updated, cross-links are fixed, and new concepts are
  added for new assets. A retired asset is marked with a `**Deprecation**`
  note rather than silently deleted, so the context survives. Git history
  is the record of what changed, when, and by whom.
- **Consumption.** A reader starts from the root `index.md` (progressive
  disclosure), then follows links only into the relevant concepts. Broken
  links are not-yet-written knowledge, not errors. Anything durable learned
  while working is written back into the bundle.

# Validation

Never eyeball conformance — run the deterministic checker. See
[validation](./validation.md) for how to run it and how to
read the output. Resolve every `ERROR` (hard §9 failures). Warnings are
soft; fix them when cheap, but they never block.

# Templates

## Concept

```markdown
---
type: <Concept type, e.g. Service, BigQuery Table, Metric, Playbook, Decision>
title: <Human-readable display name>
description: <Single sentence summarizing the concept.>
resource: <Canonical URI of the underlying asset — omit for abstract concepts>
tags: [<tag>, <tag>]
timestamp: <ISO 8601, e.g. 2026-06-14T10:00:00Z>
---

# Overview

<What this concept is and why it matters.>

# Schema

<Use for assets with fields/columns; otherwise replace with relevant sections.>

| Field | Type | Description |
|-------|------|-------------|
|       |      |             |

# Citations

[1] [<source title>](<url>)
```

## Index (`index.md` — no frontmatter)

```markdown
# <Directory / Group Heading>

* [<Title>](./<relative-url>) - <short description from the concept's frontmatter>
* [<Title>](./<relative-url>) - <short description>

# <Another Group>

* [<Subdirectory>](./<subdir>/index.md) - <short description of the subdirectory>
```
