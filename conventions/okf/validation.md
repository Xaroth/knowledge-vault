---
type: Reference
title: Validating OKF conformance
description: How to validate OKF v0.1 conformance in this vault with `speki okf validate`, and how to read its findings.
tags: [okf, convention, validation, tooling]
timestamp: 2026-07-29T20:49:13Z
---

# Overview

Conformance with the [OKF v0.1 spec](./spec.md) is checked deterministically by
`speki` — never by eyeballing. `speki okf validate` implements the format rules
speki enforces (`speki okf spec` prints them). [Using speki](./speki.md) covers
the surrounding read/edit/commit workflow.

# Running it

Validate a vault by name:

```bash
speki okf validate --name <vault>
```

`--path <dir>` validates a bundle root or subtree outside the configured vaults
instead (it cannot be combined with `--name`). `--strict` promotes every warning
to an error, for a zero-warning gate.

Add `-o json` for a machine-readable `{ok, errors, warnings}` envelope (useful in
CI). `speki submit` runs the same validation before it commits — and takes the
same `--strict` — so a standalone `speki okf validate` is an optional pre-check,
not a separate gate. `speki submit --dry-run` validates and lists what would be
published without committing.

# Interpreting the result

- **error** (`E*`) → a hard conformance failure; the bundle is non-conformant.
  Fix every one — `submit` refuses while any remain.
- **warning** (`W*`) → soft guidance: a missing recommended field, a non-explicit
  or broken link, a tag problem. Warnings never block a submit unless `--strict`
  is passed; fix them when cheap. Broken intra-bundle links in particular are
  explicitly tolerated by the spec (§5.3) as not-yet-written knowledge.

With `-o json`, the result reports `"ok": true` when there are no errors,
regardless of warnings.

# What it checks

| Code | Severity | Rule |
|------|----------|------|
| `E1` | error   | A file cannot be read, or a concept file has missing/malformed YAML frontmatter. |
| `E2` | error   | Concept frontmatter is missing the required `type` field. |
| `E3` | error   | A nested `index.md` contains frontmatter (not permitted; the bundle-root `index.md` may carry only `okf_version`). |
| `E4` | error   | The bundle declares an `okf_version` (in `.speki/config.yaml` or the root `index.md`) that this speki does not validate. The declaration is optional; absent means the supported version. |
| `W1` | warning | A recommended frontmatter field (`title` or `description`) is missing. |
| `W2` | warning | A link is ambiguous about its base directory: neither explicitly relative (`./`, `../`) nor absolute (leading `/`). |
| `W3` | warning | An intra-bundle link resolves to nothing inside the bundle — a missing target, or one escaping the root via `../`. |
| `W4` | warning | A non-empty `log.md` has no ISO 8601 date headings. |
| `W5` | warning | A concept file has a blank tag, or two tags that collide after normalization. |
