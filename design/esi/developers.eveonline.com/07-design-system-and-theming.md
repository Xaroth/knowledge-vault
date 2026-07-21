---
type: Design Document
title: "Design System & Theming"
description: "The Astro-native design system and the token contract shared with the API Explorer."
tags: [design, esi, developer-portal, design-system, theming]
timestamp: 2026-07-21T12:20:44Z
---

# 7. Design System & Theming

## 7.1 Decision — a new Astro-native design system

Mantine is **retired**. The portal gets a **new, Astro-native design system** built on plain Astro
components and a single token layer, rather than porting Mantine's React components wholesale.

Rationale:

- **One token system.** Mantine's theme and the API Explorer's CSS-custom-property tokens are two
  competing systems today. A single native token layer unifies the whole portal, the docs, and the
  API Explorer.
- **Lighter output.** Static pages (docs, blog, marketing) ship no React/Mantine runtime; interactive
  behaviour is added only where needed.
- **No framework to fight.** Astro-native components match the simple, mostly-static nature of the
  portal.

The cost is real: this is more work than porting, and a new design system inevitably shifts the visual
language. Per the [parity contract](./01-motivation-and-goals.md#15-the-parity-contract), the
**IA and layout of each surface are reproduced at parity** even as the visuals are rebuilt; this
document's rebuild is the main place where "parity floor" and "redesign stretch" blur, and it is
called out as the primary parity risk in [10-migration.md](./10-migration.md).

## 7.2 The token contract

The design system is expressed as **CSS custom properties** (colour, typography, spacing, radius,
elevation, motion). This is deliberately the **same contract the API Explorer already speaks**, so the
Explorer, mounted as an island, themes from the portal's tokens with no adapter.

- Tokens are the single source of truth; components consume tokens, never hard-coded values.
- The authoring-kit components ([05](./05-docs-authoring-contract.md)) resolve their styling from the
  same tokens, so community docs look native inside the portal.

## 7.3 Licensed assets

Licensed **fonts and imagery** live in the **private repo** (and Contentful for editorial imagery),
loaded through the token/theme layer. They are structurally absent from the public repo and the
authoring kit — resolving [P2](./01-motivation-and-goals.md#12-concrete-pain-points).

## 7.4 Interactivity — React islands only where needed

React survives only as **islands** for genuinely interactive surfaces:

- The **API Explorer** (its own component).
- **Application management** forms and **authorized-apps** views (authed, dynamic).
- A **command palette** (replacing Mantine's spotlight), rebuilt natively.

Everything else — the shell, navigation chrome, docs, blog, marketing — is static Astro. Icons come
from an Astro-friendly icon approach rather than the current FontAwesome React wiring.

## 7.5 Storybook / component workbench

A component workbench (Storybook or equivalent) is retained in the private repo for the design-system
and island components, so the new system is developed and reviewed in isolation.
