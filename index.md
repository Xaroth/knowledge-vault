---
okf_version: "0.1"
---

# Personal Knowledge Vault

Knowledge kept in [OKF](./conventions/okf/index.md) format — Markdown files with
YAML frontmatter — and committed through [speki](./conventions/okf/speki.md). The
primary focus is feeding LLMs efficient, durable context; the secondary focus is
staying human-readable.

Changes go through `speki begin → submit`; git history is the paper trail of who
changed what, when, and why. Never drive the vault with raw git — see
[Using speki](./conventions/okf/speki.md).

# Areas

* [Conventions](./conventions/index.md) - the OKF format and authoring standards this vault follows, plus how to use speki.
* [Design](./design/index.md) - design documentation for systems and components, describing their intended shape.
* [Engineering](./engineering/index.md) - deep implementation knowledge: how systems are built, the formats they read and write, and the tools that work against them.
* [Project](./project/index.md) - product-level knowledge about features and initiatives.
