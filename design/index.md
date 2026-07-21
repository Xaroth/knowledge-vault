# Design

Design documentation for systems and components — the intended shape of a thing
as it is designed, before or during its build. Pages here describe the design in
the present tense, per
[describe the present, not the change](../conventions/describe-current-state.md).

# ESI

* [API Explorer](./esi/api-explorer/README.md) - design for a home-grown, self-contained OpenAPI 3.0/3.1 renderer to replace `@stoplight/elements`: a two-package split (parser + renderer), an Astro-islands-safe component, and the adversarial design review behind it.
* [developers.eveonline.com Rework](./esi/developers.eveonline.com/README.md) - re-platform of the EVE developer portal onto a two-repo Astro architecture (public content + private styled site), unifying the split Next.js/MkDocs stacks, keeping licensed assets and infrastructure private while docs stay open and community-contributable.
