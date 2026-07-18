---
type: Design Document
title: "Styling & Theming"
description: "Self-contained styling, the CSS custom-property token contract, and the stable DOM override contract."
tags: [design, esi, api-explorer, openapi, styling, theming, css-tokens]
timestamp: 2026-07-18T19:00:46Z
---

# 7. Styling & Theming

The component ships its own styling and exposes a small, documented override surface. It depends on no
UI kit and leaks no styles. This replaces the current ~350-line override war (pain points **P3**,
**P6**).

## 7.1 What we are replacing

Today's [theme.scss](../src/components/@stoplight/elements/theme.scss) exists only to make a
third-party component look like ours:

- It remaps Stoplight's HSL token system onto Mantine tokens.
- It `@nested-import`s Stoplight's entire stylesheet under a `[data-wrapper]` selector to contain it.
- It re-colours code blocks with `!important`.
- It drives a sticky-sidebar scroll animation by targeting private selectors (`.sl-elements`,
  `.sl-sticky`, `.sl-bg-canvas`, `[data-testid='two-column-right']`, `.sl-stack--{i}`, …).
- [wrapper.tsx](../src/components/@stoplight/elements/wrapper.tsx) injects a script to prime
  `localStorage['mosaic-theme']` so Mosaic's theme detection does not fight the app.

None of this is our styling — it is us fighting someone else's. When we own the component, the styling
*is* the component's, and this whole layer disappears.

## 7.2 Approach — CSS Modules + Sass + custom-property tokens

Ported from the POC, which already does this cleanly:

- **CSS Modules** for every component (`*.module.scss` imported as `styles`). No global class names,
  so nothing leaks into or out of the host. This is the structural guarantee of self-containment.
- **Sass** for authoring (nesting, mixins), compiled to a single shipped stylesheet.
- **CSS custom properties for all themeable values** — colours, method colours, fonts, spacing. Sass
  scalars are used only for non-themeable structure.

## 7.3 The token contract {#tokens}

The component reads its themeable values from CSS custom properties with sensible defaults. This is
the **public theming API** — the "clear elements that can be overridden" the goal calls for.

```scss
/* defaults shipped by the component; every one is override-friendly */
--oae-color-bg
--oae-color-surface
--oae-color-border
--oae-color-text
--oae-color-text-muted
--oae-color-accent            /* links, active nav, focus */
--oae-color-danger / --oae-color-success / --oae-color-warning
--oae-method-get / -post / -put / -patch / -delete / -head / -options
--oae-font-family
--oae-font-family-mono
--oae-radius
--oae-space-unit
/* …a small, documented, stable set */
```

Prefix (`--oae-`, provisional) namespaces the tokens so they cannot collide with a host's variables.
A host themes the component by setting these on the explorer root or any ancestor:

```css
.my-docs [data-oae-root] { --oae-color-accent: #1b9cff; --oae-font-family: "Shentox", sans-serif; }
```

No `!important`, no selector reverse-engineering, no forking.

## 7.3a The stable DOM contract {#dom-contract}

We criticized Stoplight for forcing us to target private `.sl-*` classes (pain point **P3**). We must
not recreate that problem under our own name ([notes A4](./notes/01-architect-round1.md#a4)). So the
component exposes a **deliberate, documented, semver-covered** set of DOM hooks — and nothing else is
fair game:

- **`data-oae-*` attributes** on structural elements: `data-oae-root`, `data-oae-nav`,
  `data-oae-operation`, `data-oae-method` (value = the HTTP method), `data-oae-schema`,
  `data-oae-tryit`, `data-oae-response`, … These are the sanctioned targets for host CSS that needs to
  reach past the tokens.
- **Internal class names are CSS-module-hashed** precisely so they *cannot* be targeted and can never
  become an accidental contract.

A test asserts the documented hooks exist ([10](./10-testing-and-tooling.md)), so removing one is a
caught, deliberate, breaking change ([11](./11-open-source-and-api-stability.md#css-dom-contract)). If
you need to style something that has no hook, that is a request to add a hook — not a reason to reach
into internals.

## 7.4 Theme selection

The `theme` prop sets `data-theme` on the component root (as the POC does). Token *values* live in
theme blocks scoped by that attribute:

```scss
[data-theme='light'] { --oae-color-bg: #fff;  --oae-color-text: #111; /* … */ }
[data-theme='dark']  { --oae-color-bg: #0d0f14; --oae-color-text: #e6e6e6; /* … */ }
```

Because the value is just an attribute + a matching CSS block, a host can define **its own** theme
name and block (`theme="eve-dark"`) without any component change. Light and dark are shipped;
anything else is host-defined.

## 7.5 Self-containment rules

- **No global resets.** The component does not touch `body`/`:root`. Its root is a normal flex/grid
  container that sets `box-sizing: border-box` on its own descendants only (the POC's rule).
- **No UI-kit dependency.** No Mantine, no Mosaic. The few interactive primitives it needs
  (collapse, tabs, tooltip, copy button) are small, purpose-built, and styled with tokens — so the
  component works identically whether or not the host uses Mantine. (Contrast the current
  [panel.tsx](../src/components/@stoplight/elements/panel.tsx), which is Mantine-based.)
- **Responsive by default.** The sidebar/stacked layout adapts at a breakpoint internally; there is no
  separate "responsive" mode to configure.
- **No scroll-timeline hacks against foreign DOM.** Sticky positioning is expressed against the
  component's *own* layout, which it controls — not reverse-engineered from Stoplight's markup.

## 7.6 Distribution

Following the POC's packaging so hosts have graded levels of control:

- `@eve-online-tools/openapi-explorer/style.css` — the compiled component stylesheet (import once).
- `@eve-online-tools/openapi-explorer/styles` — the raw Sass token partials, for hosts that want to theme in Sass.
- The token set in [§7.3](#tokens) — the primary, framework-free override surface (just CSS).

## 7.7 How the ESI look is achieved

The ESI adapter supplies an EVE-branded theme block (fonts, accent, method colours, dark surface) by
setting the `--oae-*` tokens — the same values today's `theme.scss` bridges from Mantine, but now
declared once as plain tokens instead of derived through three theming systems. No `localStorage`
priming script, no Mosaic, no Mantine bridge.
