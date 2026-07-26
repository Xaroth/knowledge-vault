---
type: Design Document
title: "Agent Interface — CLI vs MCP vs Skills"
description: "The decision on how speki talks to an AI agent: keep the CLI as the sole mutation path, add an Agent Skill / slash-command guidance layer, harden the JSON/error surface, and do not build an MCP server. With token economics, fool-proofing analysis, and cited sources."
tags: [design, speki, agent-interface, mcp, cli, guide, decisions]
timestamp: 2026-07-26T12:00:00Z
---

# 3. Agent Interface — CLI vs MCP vs Skills

**Decision:** keep the **plain CLI** as speki's sole mutation path; add an **Agent Skill /
slash-command** layer to carry the workflow guidance and the *no-raw-git* constraint on demand;
**harden the agent-native surface** (`-o json` everywhere, every error a stable code + explicit next
step). **Do not build an MCP server** for the mutation path.

This is a deliberate, evidence-backed choice, not a default. The rationale below is balanced — it
includes the cases where MCP would legitimately win, none of which speki currently hits.

## 3.1 The three candidates

1. **Plain CLI + session hook (today).** The agent runs `speki begin`, `speki submit`, … as shell
   commands; a session-start hook injects protocol guidance text every session.
2. **A command-line MCP server.** speki exposes an MCP stdio server with typed tools
   (`vault_begin`, `vault_submit`, …) that the agent calls through the client's tool interface.
3. **Agent Skills / slash-commands.** A `SKILL.md` (or `/speki-edit` slash-command) packages the
   procedure and constraint; it loads lazily on a trigger and shells out to the existing `speki`
   binary.

These are not mutually exclusive. The recommendation is **1 + 3**: CLI for actions, Skill for
guidance.

## 3.2 Why not MCP — the token / context economics

MCP's defining cost is that every connected server injects **all** its tool definitions (names,
descriptions, full JSON schemas) into context on every turn, before the user says anything.
Practitioner and vendor measurements:

- **~200–800 tokens per well-documented tool**
  ([Layered](https://layered.dev/mcp-tool-schema-bloat-the-hidden-token-tax-and-how-to-fix-it/)); an
  open MCP spec issue frames it as "~1000 tokens/tool consumed per session"
  ([modelcontextprotocol#2808](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2808)).
- A single large server can cost **10,000–17,000+ tokens per request**, and seven servers ≈ **67,300
  tokens — 33.7% of a 200k window — before the first message**
  ([Junia](https://www.junia.ai/blog/mcp-context-window-problem)).
- Independent comparisons put MCP at **4–32× more tokens** than equivalent CLI calls
  ([MindStudio](https://www.mindstudio.ai/blog/mcp-vs-cli-ai-agents-token-costs-when-to-use)).

For speki's ~8–10 subcommands, an MCP server is a rough **2k–8k tokens of always-on overhead per
session whether or not the user touches a vault**. A CLI costs ~0 until invoked (progressive
disclosure via `--help` on demand). A Skill costs only its short description until a trigger fires.

That Anthropic is actively mitigating this in Claude Code — **MCP Tool Search** lazy-loads tool
definitions once they would exceed ~10% of context
([mcp.directory](https://mcp.directory/blog/mcp-context-bloat-fix-2026-tool-search-code-mode-progressive-disclosure))
— is itself an admission that eager tool loading is a real problem.

## 3.3 Why the CLI is the right primary interface

The strongest primary source is Anthropic's own **"Code execution with MCP"** (Nov 2025), which names
exactly these problems and recommends presenting tools as **code/APIs in a filesystem** the agent
explores and calls from a code-execution loop, loading only what it needs — reporting a workflow going
from **150,000 → 2,000 tokens (98.7% reduction)**
([Anthropic Engineering](https://www.anthropic.com/engineering/code-execution-with-mcp); analysis:
[Simon Willison](https://simonwillison.net/2025/Nov/4/code-execution-with-mcp/)). **A CLI on `$PATH`
is already the low-tech version of exactly that**: a tool the agent invokes from the shell with zero
up-front schema cost.

Reinforcing points:

- **Agents already know shell + composability.** Models are heavily trained on CLIs; a CLI composes
  with `rg`, pipes, and git, and is discoverable via `--help` on demand
  ([apideck](https://www.apideck.com/blog/mcp-server-eating-context-window-cli-alternative)).
- **Community consensus on when MCP wins:** when there is *no existing CLI*, or when you need
  **per-user OAuth, multi-tenant auth, audit trails, centralized governance, or remote access** —
  enterprise SaaS connectivity
  ([StackOne](https://www.stackone.com/blog/mcp-vs-cli-for-ai-agents/)). speki has **none** of these:
  it is a local process against a local git repo.

speki already leans into the good-CLI-for-agents pattern — `-o json` everywhere, a versioned protocol
contract (`speki guide`), machine-readable error envelopes. Those are the right instincts to double
down on.

## 3.4 The Skills layer — the actual improvement to make

Anthropic's **Agent Skills** (Oct 2025 open standard) are a folder with a `SKILL.md` (YAML
frontmatter + Markdown) plus optional scripts. Their defining property is **lazy loading**: only the
compact name + description sits in context; the full procedure loads when a trigger matches
([IntuitionLabs](https://intuitionlabs.ai/articles/claude-skills-vs-mcp)). The clean division of
labour, repeated across sources: **"Skills are procedural knowledge (how to do a task); MCP is the
connection layer (access to external systems). A skill can describe a procedure whose steps call
tools."** ([morphllm](https://www.morphllm.com/claude-code-skills-mcp-plugins))

This maps *exactly* onto speki. The `begin → edit → submit → abort` workflow and the *no-raw-git*
constraint are **procedural knowledge** — the textbook Skill use case. Today that guidance is injected
by an **always-on session-start hook**, spending tokens every session regardless of whether a vault is
touched. Repackaging it as a Skill / slash-command:

- carries the identical guidance but **loads only when relevant** (aligning with the lazy-loading
  rationale),
- **still shells out to the existing `speki` binary**, so nothing about the safety model changes,
- coexists with the current hook (the hook can shrink to a one-line pointer, or remain for hosts
  without Skills support).

This is the single highest-leverage change available on the interface side.

## 3.5 Fool-proofing — where MCP does and does not help

Established patterns for safe agent tool use, and where speki stands:

- **Validation-before-commit / transactions** — do risky work behind a gate that validates first.
  *speki already does this* (`submit` validates OKF, then commit/push/release as a unit; `abort`
  recovers). This is the recommended shape
  ([WorkOS](https://workos.com/blog/agent-experience-oujuh)).
- **Structured errors that prescribe the next action** — stable machine-readable codes plus a
  `suggested_action`/`recoverable` field: "shift from describing what went wrong to prescribing what
  to try next." speki's error contract is aligned; the lever is ensuring **every** error carries an
  explicit next step.
- **Idempotency** — mutating operations safe to call twice (speki's `begin` renews rather than
  double-acquires — already correct).

**Does MCP or CLI help fool-proofing more?** MCP has one genuine edge: JSON-Schema input validation
(enum/min/max/required) rejects a malformed call *before* the handler runs and advertises valid
options up-front, vs free-form flags validated after the fact
([Nearform](https://nearform.com/digital-community/implementing-model-context-protocol-mcp-tips-tricks-and-pitfalls/)).
**But that is not speki's threat model.** "Fool-proof" for speki means the agent must not *bypass the
workflow* (raw `git push` around the lock) or *corrupt the vault* — and **a tool schema does nothing
about either**, because the agent can always still call raw `git` regardless of transport. The real
enforcement is (a) OKF validation gating `submit` and the lock-respecting push path, and (b) strong,
well-timed guidance. A CLI with `-o json`, strict subcommand parsing, and validation-before-commit
captures ~95% of the schema benefit without the token tax. **Net: for correctness of an individual
call MCP schemas help marginally; for the safety property speki actually cares about, neither
transport is the enforcement point.**

## 3.6 Multi-writer safety stays in the commands

There is little formal "agent-facing lock" prior art; the durable patterns are the classic ones, and
speki's cooperative lock (§1.3) is already a sound instance. The load-bearing conclusion: **this logic
must stay inside speki's commands, not be re-expressed as MCP tools.** An MCP tool boundary would be a
thin wrapper around the same git operations — it adds no concurrency safety the CLI lacks and cannot
stop raw `git`. Keep the lock exactly where it is.

## 3.7 If speki ever did go MCP — the Go options

Recorded for completeness, since a future remote-vault requirement could revisit this. Two viable
libraries, both making a small server (define an input struct + handler per tool, register, `Run` over
stdio):

- **Official `github.com/modelcontextprotocol/go-sdk`** (maintained with Google; stdio via
  `mcp.StdioTransport`, tools via `AddTool`) — the forward-looking choice
  ([repo](https://github.com/modelcontextprotocol/go-sdk)).
- **`github.com/mark3labs/mcp-go`** (older, widely used; `server.ServeStdio`, built-in signal
  handling) ([repo](https://github.com/mark3labs/mcp-go)).

Effort to expose N tools is low with both — but note you would be **duplicating the CLI's command
surface behind schemas and maintaining two entry points to the same internal packages**. That
maintenance cost, for no safety gain, is the core reason to defer.

## 3.8 Recommendation

1. **Keep the CLI as the primary and only mutation path.** It is token-cheap, composable,
   agent-native, and where the transactional safety and cooperative lock correctly live.
2. **Add a Skill / slash-command guidance layer** that packages the workflow and *no-raw-git*
   constraint and loads lazily; shrink the always-on session hook to a minimal pointer. Biggest
   bang-for-buck.
3. **Harden the agent-native CLI surface** instead of adopting MCP: `-o json` on every command, every
   error carrying a stable code **and** an explicit next-step hint, validation-before-commit
   preserved. This captures MCP's schema-validation benefit without the context tax.
4. **Skip MCP** unless a requirement appears that *only* MCP serves (per-user OAuth, hosted/remote
   vaults, an MCP-only host). For a local, single-binary, git-backed vault, MCP adds up-front token
   overhead and a second surface while providing no extra protection against the threats that matter.

**Key sources:** [Anthropic — Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp)
· [MCP token overhead (Junia)](https://www.junia.ai/blog/mcp-context-window-problem)
· [spec issue #2808](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2808)
· [Skills vs MCP (IntuitionLabs)](https://intuitionlabs.ai/articles/claude-skills-vs-mcp)
· [WorkOS — agent tool design](https://workos.com/blog/agent-experience-oujuh)
· [go-sdk](https://github.com/modelcontextprotocol/go-sdk) · [mark3labs/mcp-go](https://github.com/mark3labs/mcp-go)
