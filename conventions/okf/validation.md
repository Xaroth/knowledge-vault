---
type: Reference
title: Validating OKF conformance
description: How to run the deterministic OKF v0.1 conformance checker vendored in this repository, and how to read its output.
tags: [okf, convention, validation, tooling]
timestamp: 2026-07-14T09:03:21Z
---

# Overview

Conformance with the [OKF v0.1 spec](./spec.md) is checked
by a deterministic Python script — never by eyeballing. The checker lives
at [scripts/okf_validate.py](./scripts/okf_validate.py) and
implements the §9 rules verbatim.

# Running it

Run it against the bundle you want to check (the repository root is itself
the bundle):

```bash
uv run conventions/okf/scripts/okf_validate.py . --strict
```

If `uv` is unavailable, fall back to pip + python:

```bash
python3 -m pip install --quiet pyyaml && \
python3 conventions/okf/scripts/okf_validate.py . --strict
```

Add `--json` for machine-readable output (useful in CI). Point the first
argument at any subdirectory to validate just that scope.

# Interpreting the result

- **ERROR** → a hard §9 conformance failure: no parseable frontmatter, or a
  missing/empty `type`. The bundle is non-conformant. Fix every one.
- **warn** → soft guidance: a missing recommended field, a non-ISO log
  date, or a broken cross-link. Never blocks; broken links in particular
  are explicitly tolerated by the spec (§5.3). Fix when cheap.

The exit code is non-zero if any error is present, or — with `--strict` —
if any warning is present. That makes `--strict` the right mode for a CI
gate and for finishing any pass over the bundle.

# What it checks

| Rule | Severity | Source |
|------|----------|--------|
| Non-reserved `.md` has parseable YAML frontmatter | ERROR | §9.1 |
| Frontmatter has a non-empty `type` | ERROR | §9.2 |
| Recommended field (`title`, `description`, `tags`, `timestamp`) present | warn | §4.1 |
| `index.md` carries no frontmatter (root may carry only `okf_version`) | warn | §6 / §11 |
| `log.md` date headings are ISO 8601 `YYYY-MM-DD` | warn | §7 |
| Bundle-internal cross-links resolve | warn | §5.3 |
