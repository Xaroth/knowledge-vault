---
type: Convention
title: Describe the present, not the change
description: OKF concept pages state what a subject is now — its current, enduring shape — never the change, migration, or project that produced it.
tags: [okf, authoring, style, present-tense]
timestamp: 2026-07-14T09:32:09Z
---

# The principle

Every concept page describes what its subject **is now**. This repository is not a
changelog, a migration guide, or a project journal — it is a picture of the
current state. Someone reading a page should learn what the thing *is*, not the
story of how it came to be or what is being done to it.

# Why

- **Change-narrative rots.** Today's "new endpoint" is next year's plain endpoint.
  A page written as "we are adding X" is wrong the moment X ships, and misleading
  forever after. A page written as "X does Y" stays true.
- **The durable facts get buried.** Mixing "what changed" into "what is" forces
  every reader to subtract the history to find the fact they came for.
- **Same for humans and LLMs.** Present-tense statements of fact are clean units of
  knowledge; project status is noise to a model trying to answer a question.

# In practice

- **Write in the present tense.** "ESI exposes the market read-only." Not "we are
  adding endpoints", "this will be available", or "the routes being built".
- **Cut temporal and change words.** `new`, `now`, `today`, `recently`,
  `no longer`, `this work adds`, `being built`, `before/after`. If a sentence only
  makes sense relative to a moment in time, rewrite it.
- **Don't narrate the project.** Cite a source ticket or design doc as a
  *reference* for provenance if it helps, but the body describes the subject, not
  the initiative that created it.
- **State limitations as current facts.** "Layout component IDs are opaque
  numbers" — not "until ticket X ships, the IDs are opaque".
- **Planned but not real?** Describe the designed state plainly as the spec it is,
  or leave it out until it exists — don't write a status report.

# Where change does belong

Dated history has a home: git history and the tickets themselves. Keep it out of
the concept pages, which stay present-tense.
