---
type: Guide
title: Using speki to work with the vault
description: How to read, edit, validate, and commit knowledge in a speki-managed vault — the begin/submit workflow, the no-raw-git rule, and the pitfalls to avoid.
tags: [okf, convention, guide, speki, tooling]
timestamp: 2026-07-18T19:25:58Z
---

# Overview

`speki` is the CLI that manages this vault. It enforces [OKF](./spec.md)
conformance and wraps git in a **cooperative, lock-based workflow**: it syncs to
the latest state, claims a lock so two writers do not collide, validates the
bundle, then commits and pushes. It is the only supported way to change vault
content.

Run `speki guide` for the machine-facing protocol and `speki okf spec` for the
format rules speki enforces. This page is the working know-how on top of those.

# The one hard rule

**Never use raw `git` inside a vault — not even `git status`.** All reads that
need history go through speki, and all edits go through `speki begin` →
`speki submit`. Raw git bypasses the lock, the validation gate, and the push
protocol, and puts the vault in a state speki did not expect. If you catch
yourself reaching for `git`, use the speki equivalent below instead.

# The workflow

Editing is a lock-bracketed cycle. Reading is not.

```
discover ─ resolve / catalog / status        (no lock)
read ─────  open the files                    (no lock)
begin ────  speki begin --name <vault>        (sync + claim lock)
edit ─────  edit Markdown under the vault path
validate ─  speki okf validate --name <vault> (dry run; optional but recommended)
submit ───  speki submit --name <vault> -m …  (validate + commit + push + release)
```

`speki abort --name <vault>` releases the lock without committing (add
`--hard --yes` to also discard local changes).

## Discover and read (no lock)

- `speki resolve -o json` — list configured vaults and their local paths.
- `speki catalog --name <vault> --tags` — the vault's tag vocabulary (what it
  covers). Then `--tag <t>` lists files under a tag; add `--all` to search every
  vault at once. **Start here before reading**, and reuse existing tags rather
  than coining near-duplicates.
- `speki status --name <vault> -o json` — lock ownership, working-tree state, and
  last sync time.
- Open the files the catalog points at with your normal tools (ripgrep, editor).
  No lock is needed to read.

## Edit and commit (lock required)

1. `speki begin --name <vault>` — syncs `main` to latest and claims the lock.
   Add `--wait` to queue if another device holds it.
2. Edit Markdown files under the vault path. Follow the
   [authoring guide](./authoring-guide.md) and the [spec](./spec.md); reuse tags
   from `speki catalog --tags`.
3. `speki okf validate --name <vault>` — a dry run before committing. Resolve
   every `error`; warnings are soft (see [validation](./validation.md)).
4. `speki submit --name <vault> --message "<msg>"` — validates OKF, commits,
   pushes, and releases the lock. On failure it reports findings; fix them and
   re-run.

# Choosing a vault

When several vaults are configured, put new content in the best-fitting one —
compare their names and tag vocabularies (`speki catalog --all --tags` shows
every vault at once). If the best fit is unclear, ask rather than guessing.

# Common pitfalls

- **Empty repository → `branch_not_found`.** `speki begin` needs an existing
  branch with at least one commit. A freshly created, never-committed repo must
  be initialized (a first commit on the default branch) before speki can drive
  it.
- **Dirty working tree → `begin` refuses (`dirty_working_tree`).** speki wants a
  clean tree before it locks. Resolve the pending changes first: `speki submit`
  them, `speki abort --hard --yes` to discard them, or — only when the pending
  changes are exactly what you intend to submit — `speki begin --force` to lock
  without discarding.
- **Structured errors.** Add `-o json` to any command for a
  `{error, message, hint}` envelope; the `hint` usually names the recovery.
- **Broken links are warnings, not errors.** An intra-bundle link to a missing
  file is `W3` — tolerated by OKF as not-yet-written knowledge. Fix the ones you
  can; a genuine forward reference may stay. Links to files outside the vault can
  never resolve — inline them as plain text instead of a link.
- **Locks are time-boxed.** `begin` claims a lock for a fixed window (about an
  hour). `submit` and `abort` release it; a stale lock can be re-acquired after
  it expires.

# Command reference

| Task | Command |
|------|---------|
| List vaults | `speki resolve -o json` |
| See coverage / tags | `speki catalog --name <vault> --tags` |
| Search across vaults | `speki catalog --all --tag <t>` |
| Check state | `speki status --name <vault> -o json` |
| Claim lock + sync | `speki begin --name <vault>` |
| Validate (dry run) | `speki okf validate --name <vault>` |
| Commit + push + unlock | `speki submit --name <vault> --message "<msg>"` |
| Release lock (no commit) | `speki abort --name <vault>` |
| Print the protocol | `speki guide` |
| Print the format rules | `speki okf spec` |
