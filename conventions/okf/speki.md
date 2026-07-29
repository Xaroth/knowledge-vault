---
type: Guide
title: Using speki to work with the vault
description: How to search, read, edit, validate, and commit knowledge in a speki-managed vault — the begin/submit workflow, the no-raw-git rule, recovery, and the pitfalls to avoid.
tags: [okf, convention, guide, speki, tooling]
timestamp: 2026-07-29T20:49:13Z
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
yourself reaching for `git`, use the speki equivalent below instead — including
for a diff (`speki status --files`, `speki submit --dry-run`) and for history
(`speki log`).

# The workflow

Editing is a lock-bracketed cycle. Searching and reading are not.

```
search ───  speki search "<words>" --name <vault>   (no lock)
read ─────  open the paths it returned              (no lock)
begin ────  speki begin --name <vault>              (sync + claim lock)
edit ─────  edit Markdown under the vault path
validate ─  speki okf validate --name <vault>       (dry run; optional but recommended)
submit ───  speki submit --name <vault> -m …        (validate + commit + push + release)
```

`speki abort --name <vault>` releases the lock without committing (add
`--hard --yes` to also discard local changes).

## Search and read (no lock)

- `speki search "<words>" --name <vault>` — ranked over titles, descriptions,
  tags and full text; a trailing `*` matches by prefix. Narrow with
  `--tag <t>` (repeatable, AND), `--type <t>`, or `--path <dir>`; widen with
  `--all` to rank every configured vault together. **Start here**, and do not
  grep the vault — search already covers full text.
- `speki search --vocabulary --name <vault>` — the vault's tag vocabulary (what
  it covers, and the words to reuse rather than coining near-duplicates). Use it
  before choosing tags or a `type`.
- Open the paths search returned with your normal reading tools; body matches
  carry the line to start at. No lock is needed to read.
- `speki log --name <vault>` — recent commits, newest first (`-n <count>`,
  `--file <path>`). Read it before writing to pick up the vault's
  [commit-message conventions](../../engineering/conventions/semantic-commit-messages.md).
- `speki status --name <vault> -o json` — lock ownership, working-tree state, and
  last sync time. `can_edit` is whether *this session* may edit; `held_by_me`
  only means this machine.

## Edit and commit (lock required)

1. `speki begin --name <vault>` — syncs `main` to latest and claims the lock.
   Add `--wait` to queue if another device holds it.
2. Edit Markdown files under the vault path. Follow the
   [authoring guide](./authoring-guide.md) and the [spec](./spec.md); reuse tags
   and types from `speki search --vocabulary`. `speki status --files` lists what
   you have changed.
3. `speki okf validate --name <vault>` — a dry run before committing. Resolve
   every `error`; warnings are soft (see [validation](./validation.md)).
4. `speki submit --name <vault> --message "<msg>"` — validates OKF, commits,
   pushes, and releases the lock. `--dry-run` validates and lists what would be
   published without committing or releasing. On failure it reports findings; fix
   them and re-run.

# Choosing a vault

When several vaults are configured, put new content in the best-fitting one —
compare their names and tag vocabularies (`speki search --vocabulary --all`
covers every vault at once). If the best fit is unclear, ask rather than
guessing.

# Common pitfalls

- **Push rejected → `fast_forward_required`.** `main` moved on origin while the
  lock was held. `speki sync --rebase --name <vault>` replays the local commits
  on top of origin's, then submit again. On `rebase_conflict`, remove every
  conflict marker from the named files and re-run.
- **`abort --hard` is destructive.** Plain `speki abort` drops the lock and keeps
  local edits; `--hard` also resets `main` to `origin/main` and permanently
  deletes untracked files and unpushed commits. Never pass `--hard` or `--yes`
  without an explicit go-ahead.
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
  hour); re-running it renews. `submit` and `abort` release it; a stale lock can
  be re-acquired after it expires.

# Command reference

| Task | Command |
|------|---------|
| Search content | `speki search "<words>" --name <vault>` |
| Search every vault | `speki search "<words>" --all` |
| See the tag vocabulary | `speki search --vocabulary --name <vault>` |
| Read recent history | `speki log --name <vault> -n <count>` |
| List vaults | `speki resolve -o json` |
| Check state | `speki status --name <vault> -o json` |
| List changed files | `speki status --name <vault> --files` |
| Claim lock + sync | `speki begin --name <vault>` |
| Validate (dry run) | `speki okf validate --name <vault>` |
| Commit + push + unlock | `speki submit --name <vault> --message "<msg>"` |
| Rebase after a rejected push | `speki sync --rebase --name <vault>` |
| Release lock (no commit) | `speki abort --name <vault>` |
| Print the protocol | `speki guide` |
| Print the format rules | `speki okf spec` |
