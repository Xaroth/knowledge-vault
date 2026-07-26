---
type: Design Document
title: "Rollout Plan & Open Questions"
description: "A phased implementation plan for the speki agent-interface and search work, mapped onto the existing package layout, plus the decisions that need a human owner."
tags: [design, speki, build-plan, decisions, agent-interface, search]
timestamp: 2026-07-26T12:00:00Z
---

# 5. Rollout Plan & Open Questions

This document turns the two decisions
([03-agent-interface.md](./03-agent-interface.md),
[04-indexing-and-search.md](./04-indexing-and-search.md)) into a phased plan mapped onto speki's
existing packages. Phases are independent enough to ship separately; each is small and low-risk on its
own.

## 5.1 Phasing overview

| Phase | Track | Outcome | Risk |
| --- | --- | --- | --- |
| **A** | Interface | Harden the agent-native CLI surface (JSON + error next-steps everywhere). | Very low — additive. |
| **B** | Interface | Ship an Agent Skill / slash-command guidance layer; shrink the session hook. | Low — no change to the safety model. |
| **C** | Search | Persistent BM25 + frontmatter-filter index on SQLite FTS5; new `speki search`. | Medium — new dependency + on-disk state. |
| **D** | Search | Incremental (content-hash) index refresh + `speki catalog` backed by the index. | Low — builds on C. |
| **E** | Search (optional) | Local static-embedding vectors + RRF hybrid, behind a flag. | Medium — only if BM25 proves insufficient. |

## 5.2 Phase A — harden the agent-native CLI surface

Goal: make the CLI capture MCP's schema-validation benefit without any protocol layer.

- Audit every command for **`-o json` completeness** and a **stable, machine-readable error code +
  explicit next-step hint** on every failure path (extend `internal/cli/command/errors.go` and the
  `ErrorInfo` contract). The target: an agent can recover from any error using only the JSON, no
  prose.
- Ensure `submit`'s validation-failure path (keeps the lock, returns findings) emits those findings
  as structured JSON an agent can act on file-by-file.
- No new dependencies; no behavioural change to the lock or OKF.

## 5.3 Phase B — Skill / slash-command guidance layer

Goal: move the always-on guidance to lazy, trigger-based loading.

- Author a `SKILL.md` (and/or a `/speki-edit` slash-command) that packages the
  `begin → edit → submit → abort` workflow and the *no-raw-git* constraint, sourced from the same
  `guide.Build()` output so the Skill and `speki guide` never drift.
- Add a `speki integrate` mode that installs the Skill (alongside, or instead of, the session hook).
- **Shrink the session-start hook** to a minimal pointer ("a knowledge vault is configured; run
  `speki guide` or use the speki skill when editing") so unrelated sessions stop paying full-guide
  token cost. Keep the full-hook path available for hosts without Skills support.
- The Claude re-injection matcher (`startup|resume|clear|compact`) and atomic/idempotent install
  behaviour carry over unchanged.

## 5.4 Phase C — persistent BM25 + filter index

Goal: real ranked body search, zero service, pure Go.

- Add `modernc.org/sqlite` (pure-Go, CGo-free). Create an index package (e.g. `internal/okf/index/`)
  that:
  - reuses the **shared walker** (`internal/okf/walk.go`) and frontmatter parsing so it agrees with
    validator/catalog on bundle contents and **reuses `NormalizeTag`** for tag columns;
  - **chunks by Markdown heading hierarchy** (§4.5), storing each chunk with its body text (FTS5) plus
    filter columns (tags, type, title, path, section, mtime, content-hash);
  - stores the DB as a **local, gitignored cache** (XDG cache dir or `.speki/index/`, added to
    `.gitignore` on `clone`/`import` if managing an in-repo path).
- Add `speki search <query>` with `--tag`/`--type`/`--path` filters, `--limit`, and `-o json`,
  ranking by BM25 and filtering by frontmatter columns in one SQL statement. `--all` spans vaults with
  separate per-vault output (mirroring `catalog --all`).
- Keep ripgrep documented as the exact/regex escape hatch in the guide.

## 5.5 Phase D — incremental refresh + catalog on the index

Goal: retire "rebuild every run"; make discovery cheap at scale.

- On each read command, diff working-tree **content hashes** against the stored `(path, hash)` table
  and re-index only changed/new/deleted chunks (optionally seeded by `git diff --name-only`).
- Back `speki catalog` (tag vocabulary, rollups, filters) with the persisted index instead of a full
  re-walk — same output contract, far cheaper on large/multiple vaults.
- Decide the refresh trigger: lazily on read, on `sync`/`begin`, or an explicit `speki index`
  command (see open questions).

## 5.6 Phase E — optional local semantic search

Goal: bridge vocabulary gaps and enable "find related," still zero-service. **Only if BM25 + tags
prove insufficient in practice.**

- Add a near-pure-Go **static-embedding** encoder (model2vec-style: tokenizer + safetensors lookup +
  mean-pool), shipping the small (~30 MB) model as a release/self-update asset.
- Store vectors in a Go-native store (`chromem-go` or a brute-force cosine table) — *not* sqlite-vec,
  because modernc can't load the extension and CGo is to be avoided.
- Fuse with BM25 via **RRF** (~15 lines) behind a `--semantic`/`--hybrid` flag; default stays BM25 so
  the common path never loads the model.

## 5.7 Open questions (need a human owner)

1. **Index location.** In-repo gitignored `.speki/index/` (travels with a clone, obvious) vs an XDG
   cache dir keyed by vault path (never risks being committed, but is invisible). *Leaning: XDG cache*
   for zero chance of accidental commit across the multiple-vault setup.
2. **Refresh trigger & staleness.** Lazy-on-read (always fresh, adds latency to the first search) vs
   refresh-on-`sync`/`begin` vs explicit `speki index`. How stale is acceptable for a read-only
   `catalog`/`search`? *Leaning: refresh on read but bounded by content-hash diff, which is cheap.*
3. **Do we build Phase E at all, and when?** Requires committing to shipping and versioning an
   embedding model asset via `self-update`. Gate on evidence that BM25 + tags miss real queries.
4. **Skill vs hook coexistence.** Ship the Skill *instead of* the session hook, or both? Cursor's
   Skills/hook support and the known `additional_context` injection bug affect this
   ([01-source-app.md §1.7](./01-source-app.md)).
5. **FTS5 vs Bleve.** This plan recommends FTS5 for minimum surface area, but if Phase E is considered
   near-certain, Bleve's built-in KNN + RRF might justify a single-library stack from the start. A
   one-way-door dependency choice worth an explicit decision.
6. **`search` vs `catalog` surface.** Whether to keep them as separate commands or fold ranked
   full-text into `catalog --query` (which today is title/description-only). *Leaning: a distinct
   `search` command*, leaving `catalog` as the structural/tag discovery tool it is.
