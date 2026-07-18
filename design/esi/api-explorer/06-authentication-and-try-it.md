---
type: Design Document
title: "Authentication & Try It"
description: "The Try It request builder and the first-class authentication hook that replaces DOM injection."
tags: [design, esi, api-explorer, openapi, authentication, try-it, eve-sso, fetcher]
timestamp: 2026-07-18T19:00:46Z
---

# 6. Authentication & Try It

This is the most important behavioural improvement over today's integration. It turns authentication
from a DOM hack (pain point **P2**) into a first-class, typed, island-safe input.

## 6.1 The problem being solved

Today, a logged-in character's token reaches ESI Try It calls by **writing into Stoplight's DOM**.
x-required-scope.tsx:

- selects `[data-test="auth-try-it-row"] input[type="text"]`,
- waits for it with a `MutationObserver` (10s timeout),
- sets `input.value = 'Bearer <token>'` and dispatches a synthetic `input` event,
- driven by `useEffect(() => setCurrentToken(auth.accessToken), [auth.accessToken])`.

It works only as long as Stoplight's private markup does not change, and it depends on
`useCharacterAuth()` / `useAccessRequestModal()` — host context that will not exist across an Astro
island boundary (**P7**). Both problems disappear when auth is a prop.

## 6.2 The authentication model

The host owns identity (EVE SSO login, token refresh, scope grants). The component owns *using* a
token to make a request. The seam between them is two props: `auth` (declarative values) and `fetcher`
(the transport hook).

```ts
interface AuthConfig {
  // A map from security-scheme name (as in the spec's securitySchemes) to a resolved value.
  // For ESI this is the single "OAuth2" scheme -> the character's bearer token.
  schemes: Record<string, AuthValue>
}

type AuthValue =
  | { type: 'bearer'; token: string }
  | { type: 'apiKey'; value: string }        // header/query name comes from the scheme
  | { type: 'basic';  username: string; password: string }

// The ONE transport seam. Default: globalThis.fetch.
type Fetcher = (request: Request) => Promise<Response>
```

### How it flows

1. The host resolves the current token (with refresh) and passes
   `auth={{ schemes: { OAuth2: { type: 'bearer', token } } }}`.
2. The component maps the operation's effective security to the matching `AuthConfig.schemes` entry and
   **applies it to the outgoing `Request`** — bearer/api-key/basic handled natively, exactly as the
   POC's
   `tryItAuth.ts`
   derives auth from `security` + `securitySchemes`. Simple hosts stop here — no `fetcher` needed.
3. The component calls **`fetcher(request)`** (default `fetch`). A host that needs more can read/clone
   the `Request`, inject a *fresh* token, rewrite the URL to a proxy, sign it, or route it anywhere —
   all in a few lines — and return the `Response`.

**Why one `fetcher` and not an interceptor** ([notes A11](./notes/01-architect-round1.md), resolved):
an earlier draft used `onRequest` returning `Request | Response`, whose behaviour changed with the
return type — ambiguous and hard to type. A single `fetcher(request) => Promise<Response>` is the
minimal, familiar seam; it subsumes Elements' `tryItCredentialsPolicy` *and* `tryItCorsProxy` *and*
today's DOM injection, and it is the clean place to inject a fresh token at send time (ESI tokens
expire; the host refreshes ~60s before expiry).

## 6.3 The scope / access-request flow

ESI operations require specific scopes. Today the flow is: derive `x-required-scope` from the
operation's `security` (in middlewares.ts), render
an "Authentication" section, and open the host's access-request modal via `useAccessRequestModal()`.

Island-safe replacement:

- The **required scopes** for an operation come from its `security` in the normalized model — the
  renderer computes and displays them natively (no need for the synthesized `x-required-scope` just to
  show them, though the ESI adapter may still add it for richer copy).
- When the user needs to grant a scope, the component calls the **`onRequestScopes(scopes)`** prop.
  The host opens its own modal / starts the SSO flow. The component never imports the modal context.
- Whether the current token *has* the required scopes is derived by the host (it knows the granted
  scopes) and reflected via `auth`; the component shows an "authorize / re-authorize" affordance when
  a required scope is absent and calls `onRequestScopes` on click.

So the two host-context calls in the current design (`useCharacterAuth`, `useAccessRequestModal`)
become two props (`auth`, `onRequestScopes`).

## 6.4 The Try It request builder

Ported from the POC, which is already solid
(`TryItPanel.tsx`
and `src/utils/`):

- **Fields** for path/query/header params (enum-aware selects), a server selector (from `servers`),
  and a request-body editor prefilled from a schema-derived example, with a media-type selector.
- **Shared value store** — a hand-rolled external store consumed via `useSyncExternalStore`, keyed
  `"<in>:<name>"`, with per-key subscriptions to avoid over-rendering. Exposed via a
  `useTryItValue(in, name)` hook so hosts can drive or hide fields (`hideFields` prop).
- **Reset semantics** (made explicit in review — [notes E4](./notes/02-engineer-round1.md#e4)):
  param and body values are **cleared on operation change** (they are operation-specific), while
  **server selection and per-scheme auth values persist** across operations (they are API-wide). This
  is predictable and matches how people actually explore an API.
- **Assembly** — path substitution + OpenAPI query serialization (`form`/`space`/`pipe`/`deepObject`,
  explode, arrays, objects, `allowEmptyValue`) + Content-Type injection.
- **Execution** — `AbortController`, a sequence counter, and stale-response rejection when the user
  navigates to another operation mid-request.

## 6.5 Pre-flight guards

Two validations run before Send is enabled (both from the POC):

1. **Input validation** (`validateTryItSend`) — blocks Send when required params are missing or a
   `{path}` placeholder is unresolved, with inline invalid-field messaging.
2. **Trust validation** (`validateTryItTrust`) — in `untrusted` mode, blocks Send unless the target
   origin is in `trustedServerOrigins` / `defaultServerUrl`, or a `fetcher` is present.
   This pairs with the parser's SSRF policy ([03-parser-package.md](./03-parser-package.md#security)).

For our ESI usage the host runs in `trusted` mode against `esi.evetech.net`, so the trust guard is a
no-op; it exists to make any future "paste a spec" affordance safe by construction.

## 6.5a CORS reality {#cors}

Browser `fetch` to a third-party API requires that API to send CORS headers
([notes A12](./notes/01-architect-round1.md)). Two facts shape the design:

- **ESI serves permissive CORS today**, so direct Try It works for us with the default `fetcher`.
- **Most third-party APIs do not**, so the *generic* component must not fail silently. A CORS failure
  (an opaque response / `TypeError` from `fetch`) renders an **actionable message** — "this API did
  not permit a browser request; supply a `fetcher` or proxy" — rather than a blank or a cryptic
  error. The `fetcher` seam is the documented escape hatch for hosts that need to proxy.

This is also part of the security posture — see
[09-quality-and-resilience.md](./09-quality-and-resilience.md#security).

## 6.6 What the ESI adapter provides

In this app, the adapter wires:

- `auth` from `character-auth-provider` (`getAccessToken()` with its refresh logic) — resolved *inside
  the island's host wrapper* and passed down as a prop.
- `fetcher` to attach a **fresh** `Authorization: Bearer` header at send time and to target the
  correct host (`esi.evetech.net` / `esi-test` / `esi-dev`) per tier.
- `onRequestScopes` to open the existing access-request modal and start the SSO grant.

This is a Tier-2 (React-composed) integration ([04](./04-renderer-component.md#integration-tiers)),
which is required for the slot/extension overrides and the `fetcher`/callbacks the adapter supplies.

No `data-test` selector, no `MutationObserver`, no synthetic events. See
[08-esi-integration-and-migration.md](./08-esi-integration-and-migration.md) for the full adapter map.
