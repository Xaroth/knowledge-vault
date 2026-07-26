---
type: Design Document
title: "The Source App — speki as it is today"
description: "A documented reference of the current speki CLI: purpose, commands, the cooperative lock, git integration, OKF, the tag/catalog system, and agent integration — the constraints every later decision respects."
tags: [design, speki, architecture, okf, tooling, cli]
timestamp: 2026-07-26T12:00:00Z
---

# 1. The Source App — speki as it is today

This document records the current speki so the rest of the plan is self-contained. It matters here
for two reasons: the design *extends* this app rather than replacing it, and the app's existing
safety model is precisely what any new agent interface must not weaken.

- **Repository:** `github.com/eve-online-tools/speki`
- **Local path (source of this analysis):** `/home/noorbergen/repos/eve-online.tools/speki`
- **Language / stack:** Go 1.25, CLI framework **kong** (not cobra), shells out to the `git` binary
  (no go-git), Markdown parsing via **goldmark**, config via `adrg/xdg`, self-update via
  `minio/selfupdate`.

## 1.1 Purpose

speki manages **Knowledge Vaults**: git-backed Markdown knowledge bases conforming to **OKF** (Open
Knowledge Format), spec version `0.1`. It solves two coupled problems:

1. **Format conformance** — validating that a vault's Markdown "bundle" follows OKF conventions.
2. **Cooperative multi-writer editing** — a lock-based git workflow so multiple devices/agents can
   safely share one vault without clobbering each other, using nothing more than a plain git remote
   (no server component).

A strong third theme: speki is deliberately **agent-facing**. It self-describes its protocol and
injects that guidance into AI coding agents.

## 1.2 Command surface

Commands are declared as kong structs under `internal/cli/<name>/`. Global flags: `-o/--output
text|json`, `--name <vault>`, `--timeout` (default 300s). Exit codes: `0` success, `1` operational
failure, `2` config/usage error.

| Command | What it does |
| --- | --- |
| `version` | Prints speki version (works without git). |
| `doctor` | Environment checks: git version in range, config; with `--name`, main-branch / lock / identity checks. |
| `guide` | Prints the agent-facing protocol (vaults, workflow, error codes). The canonical contract. |
| `resolve` | Lists configured vaults. |
| `catalog` | Content discovery by tag / type / path / query; `--all` spans every vault. |
| `clone` / `import` | Clone a repo and register it as a vault / register an already-cloned repo. |
| `default` | Sets the default vault. |
| `integrate <claude\|cursor>` | Installs the global session-start hook into the agent tool config. |
| `hook <claude\|cursor>` | Hidden; invoked by the installed hook — syncs default vault and emits the guide in the tool's context envelope. |
| `begin` | Sync main + claim the cooperative lock. Flags `--force`, `--wait`. |
| `submit` | Validate OKF → commit → push → release lock. `--message` required, `--strict`. |
| `abort` | Release lock without committing. `--hard` resets to origin + deletes untracked; `--yes` skips prompt. |
| `sync` | Read-only fast-forward of main, throttled to 1/hour. |
| `status` | Working-tree state, lock ownership/expiry, last sync. |
| `self-update` | Replaces the binary in place from GitHub releases. |
| `okf validate` | Validate OKF bundle conformance; `--strict`. |
| `okf spec` | Prints the embedded OKF rules. |

## 1.3 The cooperative multi-writer lock (the architectural heart)

Implemented in `internal/git/lock.go`.

- **Where it lives:** on the git **remote** (`origin`) under a custom ref `refs/speki/lock` —
  deliberately *outside* `refs/heads/*` so it never appears as a branch, in PR pickers, or in
  "delete merged branches" automation. Its payload is a JSON blob `{identifier, timestamp}` stored as
  a detached commit built out-of-tree via git plumbing (`hash-object` → `mktree` → `commit-tree`),
  attributed to a synthetic `speki@localhost` identity so it never depends on the user's git config.
- **Identity:** each device holds a random UUID `identifier` in local config; the lock is
  per-device.
- **Acquisition:** `Lock()` reads the current remote lock SHA, and if held by another device and not
  stale, returns `ErrAlreadyLocked`. Otherwise it pushes a new lock commit using
  `--force-with-lease=<ref>:<observed-sha>` — a **compare-and-swap** so two devices cannot both win a
  race.
- **Lease + staleness:** `LockMaxAge = 1h`. Re-running `begin` while holding the lock **renews** the
  lease; a lock older than an hour is *stale* and may be usurped (again via lease-guarded CAS).
- **Contention:** `begin` without `--wait` returns `ErrLockedByOther`; with `--wait` it polls every
  30s up to 5m.
- **Release:** `Unlock()` verifies the identifier, then deletes the ref with a lease-guard so a
  concurrent usurp is not silently clobbered.

`begin` → `submit` → `abort` wrap this as a transaction: `submit` validates OKF first and **keeps the
lock on validation failure** so the agent can fix and re-run; `abort --hard` resets to `origin/main`
and cleans untracked files.

## 1.4 Git integration and the "never raw git" constraint

speki shells out to `git` with a pinned environment (`LC_ALL=C`/`LANG=C` for stable English output,
`GIT_TERMINAL_PROMPT=0` so prompts fail fast instead of blocking). Supported git is `2.46.0 ≤ v <
3.0.0`. Clone URLs are validated against option injection and remote-helper transports.

The rule **"inside a vault, never use raw git"** is a *social/protocol constraint, not a technical
lock* — speki cannot physically prevent raw git. It is backed by (a) `requireRepo` ensuring
operations only ever hit the vault's own repository root, and (b) the constraint string injected into
every agent session. Crucially: **the agent can always still call `git` directly** — a fact that
matters for the interface decision in [03-agent-interface.md](./03-agent-interface.md).

## 1.5 OKF format (spec 0.1)

The spec is embedded in code (`internal/okf/validator/spec.go`), surfaced by `speki okf spec`. A
**bundle** is a tree of Markdown files. **Concept files** open with a `---` YAML frontmatter block;
required field is `type`, with `title`/`description`/`tags` recommended. Nested `index.md` must not
have frontmatter (only the root may, to declare `okf_version`). `log.md` should use ISO-8601 date
headings. Intra-bundle links must be explicitly relative/absolute and resolve within the bundle.
Validation (via goldmark for links) emits stable finding codes E1–E4 (errors) and W1–W5 (warnings,
including **W5 tag hygiene**); `--strict` promotes warnings to errors.

## 1.6 Tags, catalog, and indexing (the part this plan changes)

Two layers, both in `internal/okf/`:

- **Index builder** (`catalog/catalog.go`): walks the bundle once, reading **only each file's
  frontmatter head** (capped at the closing `---` or 256 lines) so cost is bounded by frontmatter
  size, not file size. Produces `[]Entry{Path, Type, Title, Description, Tags}`. The inverted tag
  index is computed **on the fly, never persisted**.
- **CLI** (`cli/catalog/catalog.go`): `--tags` shows the tag vocabulary + singletons; filter mode
  does AND-matching on `--tag` plus `--type`/`--path`/`--query`; default gives a per-directory
  rollup. `--query` is a **naive substring match over title + description only** — not the body.

**The three limitations this plan targets:**

1. **No persisted index** — the catalog re-walks and re-parses every invocation.
2. **No body / full-text search** — delegated to ripgrep; there is no ranking.
3. **Tags are the only first-class discovery dimension**, and the worry is that a single flat tag
   vocabulary drifts and dilutes across large and multiple vaults.

`NormalizeTag` (lowercase, collapse whitespace/`_`/`-` to a single `-`) is the canonicalization
contract any richer index must stay consistent with. The **shared walker** (`internal/okf/walk.go`)
and `ReadFrontmatterHead` are the natural extension points — anything new should route through them so
validator and catalog cannot drift apart.

## 1.7 Agent integration today

speki presents itself to agents through a **self-describing protocol** plus **auto-injected session
context**:

- **`speki guide`** (`internal/cli/guide/guide.go`) assembles the canonical contract: protocol
  version, OKF version, the *no-raw-git* constraint, vault-placement guidance, the configured vaults,
  an 8-step workflow (discover → catalog → read → begin → edit → submit → abort → status), and the
  full error catalog with stable codes + hints. The same `Build()` powers both `speki guide` and the
  session hook.
- **Injection:** `speki integrate claude|cursor` installs a **global session-start hook** (into
  `~/.claude/settings.json` or Cursor's hook config). The Claude matcher is
  `startup|resume|clear|compact` so guidance is re-injected on resumed/compacted sessions — the
  "reliably and persistently" behaviour from recent work. The hook invokes `speki hook <agent>` by
  absolute path, which best-effort syncs the default vault and prints only the guide, wrapped in the
  tool's context envelope. Installation is atomic, idempotent, and upgrades legacy hooks in place.

**There is no MCP server in the codebase today.** Integration is purely native session-start hooks
plus stable JSON output (`-o json` everywhere; structured `{error, message, hint}` envelopes). This
is the starting point the interface decision builds on.

## 1.8 Config and vault resolution

Config lives at `~/.config/speki/config.yaml` (device `identifier` + `vaults` list; each vault has
`path`, `name`, `default`, `readonly`). `Resolve(name)` selects a named vault or the default;
`WritableVault()` rejects read-only vaults *before* any lock is claimed. Note the two distinct config
scopes: the user-global config above, versus a per-vault `.speki/config.yaml` that only declares
`okf_version`.
