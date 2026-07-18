# OKF Convention

The Open Knowledge Format itself — the standard every piece of knowledge in
this vault follows.

* [Specification](./spec.md) - the verbatim OKF v0.1 draft specification; the normative source of truth.
* [Authoring guide](./authoring-guide.md) - how to produce, maintain, and consume OKF bundles, with copy-paste templates.
* [Using speki](./speki.md) - the CLI that manages this vault: read, edit, validate, and commit knowledge through the begin → submit workflow. Start here to make any change.
* [Validation](./validation.md) - how to run the conformance checker and read its output.

# Tooling

* [scripts/okf_validate.py](./scripts/okf_validate.py) - the deterministic OKF v0.1 conformance checker (in this vault, validation and commits run through `speki` — see [Using speki](./speki.md)).
