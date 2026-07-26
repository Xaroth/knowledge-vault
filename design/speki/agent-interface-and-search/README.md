---
type: Design Overview
title: "Speki — Agent Interface & Vault Search Design"
description: "How speki should present itself to AI agents (CLI vs MCP vs Skills) and how it should index large knowledge vaults for fast agent search, with the research and decisions behind both."
tags: [design, speki, agent-interface, indexing, search, okf, overview, decisions]
timestamp: 2026-07-26T12:00:00Z
---

# Speki — Agent Interface & Vault Search Design

This directory holds a design plan for the next evolution of **speki**, the CLI that manages
git-backed Markdown knowledge vaults (the vault you are reading this in is one). It answers two
open questions:

1. **How should speki talk to an AI agent?** Keep the current plain-CLI-plus-session-hook model,
   move to a command-line **MCP server**, or adopt **Agent Skills / slash-commands** — under the
   hard requirements that the interface be *fool-proof*, *multi-writer-safe*, and give *quick local
   access* to a vault.
2. **How should speki index a large (and multi-vault) knowledge base** so an agent can search it
   quickly and reliably? Tags alone are the current mechanism, and the concern is that they will not
   survive scale. What full-text and/or vector alternatives fit the *zero-service, pure-Go-or-simple*
   constraint?

The documents describe the **intended shape** of the solution in the present tense, per
[describe the present, not the change](../../../conventions/describe-current-state.md). They are
design intent to draft plans, estimates, and tickets from — not an implementation log.

## The source app (documented, because it constrains everything)

This plan is written *for* an existing application, and the design only makes sense against how that
app already works. The current speki is documented in full in
[01-source-app.md](./01-source-app.md) so this plan stays useful even to a reader who has never seen
the codebase. In one paragraph:

> **speki** is a Go 1.25 CLI (kong-based, shells out to the `git` binary — no go-git) that manages
> **OKF** (Open Knowledge Format) vaults: git-backed Markdown knowledge bases. Its defining feature
> is a **cooperative multi-writer lock** stored on the git remote under `refs/speki/lock`, wrapped in
> a transactional edit session — `speki begin` (sync + claim lock) → edit Markdown → `speki submit`
> (validate OKF, commit, push, release) → `speki abort` (recover). It is explicitly **agent-facing**:
> it self-describes its protocol via `speki guide` and injects that guidance into Claude Code / Cursor
> through a session-start hook. Discovery is via **tags** in YAML frontmatter, surfaced by
> `speki catalog`; there is no persisted index and no body/full-text search (that is delegated to
> ripgrep). Source: `github.com/eve-online-tools/speki`, local path
> `/home/noorbergen/repos/eve-online.tools/speki`.

## The two decisions, up front

Both research tracks converged on clear, evidence-backed recommendations. They are stated here so the
rest of the plan can be read as justification and detail.

| Question | Decision | One-line why |
| --- | --- | --- |
| **Agent interface** | **Keep the CLI as the sole mutation path; add an Agent Skill / slash-command layer for guidance; harden the JSON/error surface. Do *not* build an MCP server.** | Anthropic's own late-2025 guidance points away from eager MCP tool-loading and toward filesystem/CLI-style interaction; MCP would add a persistent per-session token tax and a second surface to maintain while providing *no* extra protection against the threats speki actually faces (the agent can call raw `git` regardless of transport). |
| **Vault search** | **Add a persistent BM25 full-text + frontmatter-filter index on `modernc.org/sqlite` FTS5 (pure-Go, single file, gitignored cache). Keep tags as filter columns, not the primary ranker. Treat local static-embedding vectors as an optional, later phase.** | A real ranked body index is the highest-value, lowest-risk fix for "tags don't scale / no full-text / rebuild every run," stays zero-service and pure-Go, and keeps tags earning their keep as filters. Vectors only earn their complexity for vocabulary-gap / "find related" queries. |

## Reading order

1. **[01-source-app.md](./01-source-app.md)** — The current speki, documented: purpose, command
   surface, the cooperative lock, git integration, the OKF format, the tag/catalog system, and how it
   integrates with agents today. The constraints every later decision must respect.
2. **[02-goals-and-constraints.md](./02-goals-and-constraints.md)** — What "fool-proof",
   "multi-writer-safe", and "quick local access" concretely require; the real threat model; goals and
   non-goals.
3. **[03-agent-interface.md](./03-agent-interface.md)** — CLI vs MCP server vs Skills, the token /
   context economics, the fool-proofing analysis, and the recommendation with rationale and cited
   sources.
4. **[04-indexing-and-search.md](./04-indexing-and-search.md)** — Tags vs embedded full-text (Bleve,
   SQLite FTS5, trigram) vs local vectors (static embeddings, chromem-go, sqlite-vec); the
   chunking/metadata/incremental-index strategy; the recommendation with cited sources.
5. **[05-rollout-and-open-questions.md](./05-rollout-and-open-questions.md)** — A phased
   implementation plan mapped onto speki's existing packages, plus the decisions that need a human
   owner.

## Guiding principles

- **The agent is already a good shell user.** Prefer interfaces the model already knows (a CLI,
  files on disk) over protocol machinery that must be loaded before it can be used.
- **Safety lives in the transaction, not the transport.** `begin`/`submit`/`abort` + OKF validation +
  the cooperative lock are the enforcement points; no tool schema replaces them.
- **Zero-service, single-binary.** Anything added must run locally with no daemon and, wherever
  possible, no CGo — the vault must stay a plain git repo plus one static binary.
- **Tags are a filter, not a search engine.** The upgrade path keeps tags and adds ranking around
  them, rather than replacing the mechanism people already use.
- **Indexes are disposable.** The search index is a local, gitignored, content-hash-incremental
  cache — never committed, never merge-conflicting across the multiple vaults.
