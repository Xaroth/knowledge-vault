---
type: Design Document
title: "Goals, Constraints & Threat Model"
description: "What fool-proof, multi-writer-safe, and quick local access concretely require for speki; the real threat model; goals and non-goals for both the agent interface and the search work."
tags: [design, speki, goals, decisions, agent-interface, search]
timestamp: 2026-07-26T12:00:00Z
---

# 2. Goals, Constraints & Threat Model

The two design tracks share one set of requirements. Naming them precisely is what lets the interface
decision reject MCP on evidence rather than taste.

## 2.1 The three hard requirements, made concrete

### R1 — Fool-proof

An agent (or a careless human driving one) must not be able to *easily* corrupt the vault or bypass
the safe workflow. Concretely this decomposes into:

- **Bad content cannot be published.** A malformed OKF bundle must be rejected before it reaches the
  remote. *(Already met: `submit` validates before commit.)*
- **The workflow cannot be silently skipped.** Editing without a lock, or pushing around the lock,
  must be hard to do by accident and obvious when attempted.
- **Mistakes are recoverable.** Any failed or abandoned edit session has a clean, single-command
  recovery. *(Already met: `abort` / `abort --hard`.)*
- **Errors tell the agent what to do next.** Every failure carries a stable code *and* an explicit
  next step, so the agent self-corrects instead of thrashing.

### R2 — Multi-writer-safe

Multiple devices and multiple agents editing one vault must never clobber each other. The current
cooperative lock (`refs/speki/lock`, 1h lease, CAS via `--force-with-lease`, `--wait` queueing) is a
sound instance of optimistic/cooperative concurrency exposed as a transactional edit session. **This
requirement is met today and the design must not regress it.** The key downstream consequence: the
locking logic must stay *inside speki's commands*, because that is the only place it can be enforced.

### R3 — Quick local access

Reading and searching a vault must be fast and must not depend on any network round-trip or running
service. This is where the search work lives: discovery today is tag-only with no persisted index and
no body search, and "quick" must survive vaults of thousands of files across several repos.

## 2.2 The real threat model (why it decides the interface)

The threat is **not** "the agent constructs a syntactically invalid command" — that is a minor,
already-handled concern. The threats that actually matter are:

1. **Workflow bypass** — the agent runs raw `git push` / `git reset` and clobbers the lock or
   another writer's work.
2. **Vault corruption** — invalid OKF or a broken bundle reaches the remote.
3. **Cross-writer clobber** — two writers race.

The decisive observation, carried through [03-agent-interface.md](./03-agent-interface.md): **none of
these is prevented by the shape of the tool interface.** A JSON tool schema does not stop an agent
from calling raw `git`; only the transactional command + server-side/validation checks do. Therefore
the interface should be chosen on *ergonomics, token cost, and maintenance*, and the safety
properties should be reinforced *inside the commands* — not migrated into a tool-schema layer that
cannot enforce them anyway.

## 2.3 Portability / build constraints

- **Zero-service.** No daemon, no separate database server, no cloud API required at read/query time.
  A vault stays "a git repo + one static binary."
- **Prefer pure Go, avoid CGo.** speki ships as a self-updating single binary across platforms
  (including Windows, where the test suite already carries fixes). Pure-Go dependencies keep
  cross-compilation and `self-update` trivial; CGo/native libraries are a last resort and must be
  justified.
- **Route through the shared walker.** New indexing must reuse `internal/okf/walk.go` and
  `ReadFrontmatterHead`/frontmatter parsing so validator, catalog, and any new index stay in
  agreement on what a bundle contains and how tags normalize.

## 2.4 Goals

**Agent interface**

1. Keep the interface **token-cheap**: near-zero always-on context cost when the user is not editing a
   vault.
2. Keep it **composable** with the shell and git the agent already knows.
3. Make the safe workflow and the *no-raw-git* constraint **load exactly when relevant**, not on
   every unrelated session.
4. Make the machine-readable surface (`-o json`, error codes + next-step hints) complete enough that
   an agent rarely needs prose to recover.

**Search**

5. Add **ranked full-text (body) search** as a first-class, persisted capability.
6. **Keep tags** as a first-class *filter* dimension joined to that search — do not replace them.
7. Keep the index **fresh incrementally** and **disposable** (local, gitignored, rebuildable).
8. Leave a **clean seam for optional local semantic (vector) search** without requiring it up front.

## 2.5 Non-goals (YAGNI)

- **Not an MCP server** for the mutation path — see [03](./03-agent-interface.md) for the full
  rationale. (Reconsidered only if a future requirement appears that *only* MCP serves: per-user
  OAuth, multi-tenant/hosted access, remote non-local vaults, or a host that can consume *only* MCP
  tools.)
- **Not a hosted search service, vector DB server, or cloud embedding dependency.**
- **Not a replacement for the agent's own grep/read loop.** speki's search *complements* agentic
  search with ranking the agent cannot cheaply reproduce; it does not try to own exploration.
- **Not a change to the OKF format or the lock protocol.** Those are stable and out of scope.
- **Not committing the search index to git.**
