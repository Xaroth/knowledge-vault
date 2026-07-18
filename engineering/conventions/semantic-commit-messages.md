---
type: Convention
title: Semantic commit messages
description: Commit subjects follow Conventional Commits — a required lowercase type, an optional parenthesized scope, and a concise present-tense description — with a blank-line body, breaking-change markers, and the squash-merge PR number appended.
tags: [convention, git, commit-messages]
timestamp: 2026-07-18T22:20:00Z
---

# The principle

Every commit message is **semantic**: its subject declares *what kind* of change
it is and *what area* it touches, in a fixed grammar, before saying what it does.
This is the [Conventional Commits](https://www.conventionalcommits.org/) shape. A
reader — or a tool generating changelogs and version bumps — can classify a commit
from its subject alone, without reading the diff.

# The grammar

```
<type>(<scope>): <description>

<body>

<footer(s)>
```

Only the first line is required. It is a **subject line**: one line, no trailing
period, kept short (aim for ≤ 72 characters before any PR suffix).

* **`<type>`** — required, lowercase, from the [allowed set](#types).
* **`(<scope>)`** — optional, in parentheses, lowercase. See [scope](#scope).
* **`: `** — a colon followed by a single space.
* **`<description>`** — the change in one concise, present-tense clause. See
  [the description](#the-description).

# Types

A commit's type is one of a small, closed set. Pick the type that describes the
*intent* of the change.

| Type | Use for |
| --- | --- |
| `feat` | A new capability or user-facing behavior. |
| `fix` | A bug fix. |
| `chore` | Maintenance with no product behavior change — dependency bumps, config, infrastructure, dashboards, release scheduling, tooling. |
| `refactor` | Restructuring code without changing its behavior. |
| `docs` | Documentation-only changes. |
| `revert` | Reverting a previous commit. |
| `test` | Adding or adjusting tests only. |
| `build` | Build system or packaging changes. |
| `ci` | CI/pipeline configuration changes. |
| `perf` | A change made to improve performance. |
| `style` | Formatting or non-semantic code style only. |

`feat`, `fix`, and `chore` carry the vast majority of changes; `refactor`, `docs`,
and `revert` are common; the rest are used as needed. Do not invent new types
(`add`, `change`, `remove`, `update` are not types — fold them into `feat`, `fix`,
`chore`, or `refactor`).

# Scope

The scope names the part of the system the change touches. It is optional but
encouraged when a commit is localized to one area.

* Lowercase, inside parentheses, immediately after the type: `feat(cache): …`.
* Multi-word scopes are **kebab-case**: `chore(rate-limit): …`,
  `fix(mercenary-den): …`.
* A scope may be a **dotted path** to name a nested package or domain:
  `chore(middleware.errorlimit): …`, `chore(users.characters): …`.
* Prefer a stable, recognizable name — a package, module, subsystem, or domain
  concept — over an ad-hoc one. Omit the scope when a change is broad or spans
  many areas.

# The description

The description is a single clause stating the change.

* **Present tense, lowercase start, no trailing period.**
  `fix: strip trailing slash from the proxied path` — not
  `Fix: Strip trailing slash.`.
* **Say what the commit does**, concisely. For a `fix`, describing the *symptom*
  being resolved is idiomatic and acceptable
  (`fix: client-closed request is recorded as a 504`).
* Keep it short enough to read at a glance; move detail into the body.

# Breaking changes

A change that breaks a consumer is marked in **either or both** of two ways:

* Append `!` after the type/scope, before the colon:
  `feat!: …`, `chore(api)!: …`.
* Add a `BREAKING CHANGE:` footer describing what breaks and how to adapt.

```
feat(cache)!: rework how cache is configured per route

BREAKING CHANGE: removed "Conditional"; set the cache definition in
your handler instead.
```

# Body and footers

Everything after the subject is optional.

* **Body** — separated from the subject by one blank line. Explains *what* and
  *why* (not *how*), wrapped at roughly 72 columns. Use it whenever the subject
  cannot carry the reasoning.
* **Footers** — separated from the body by a blank line. `BREAKING CHANGE:` is a
  footer; issue and cross-references also belong here.

# The pull-request suffix

Squash-merged commits carry the merged PR number in parentheses at the end of the
subject: `feat: add character list for user (#3)`. This is the merge tooling's
doing — write the subject without it, and let the squash merge append it.

# Checklist

A conforming commit subject:

- [ ] starts with an allowed lowercase `type`
- [ ] has a lowercase, kebab-case or dotted `(scope)` when localized — or none
- [ ] uses `: ` (colon + one space) after the type/scope
- [ ] has a present-tense, lowercase description with no trailing period
- [ ] fits on one short line (PR suffix aside)
- [ ] marks breaking changes with `!` and/or a `BREAKING CHANGE:` footer
- [ ] puts any explanation in a blank-line-separated body, not the subject
