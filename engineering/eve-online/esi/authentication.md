---
type: Reference
title: ESI — authentication
description: ESI authenticates a character through EVE SSO using OAuth 2.0 — the authorization-code flow (with PKCE for native apps), endpoints discovered from a well-known metadata document, and a scoped JWT access token the application validates against the SSO JWKS.
tags: [eve-online, esi, authentication, eve-sso]
timestamp: 2026-07-19T11:05:00Z
---

# EVE SSO

Authenticated ESI routes act on behalf of a specific character. The character
authorizes an application through **EVE Single Sign-On (SSO)**, an **OAuth 2.0**
provider at `https://login.eveonline.com`. The application never sees the
character's password; it receives tokens scoped to exactly what the character
consented to.

# The two flows

* **Authorization Code** — for applications that can keep a client secret on a
  server the user does not control (web backends).
* **Authorization Code with PKCE** — for applications that cannot keep a secret
  (native desktop, mobile, single-page apps). PKCE substitutes a per-request proof
  for the client secret.

# Discovering the endpoints

The SSO endpoints are **discovered, not hard-coded**. A metadata document at

```
https://login.eveonline.com/.well-known/oauth-authorization-server
```

returns the current authorization, token, and JWKS (signing-key) endpoint URLs. A
client fetches it periodically and caches the result, because the concrete URLs
can change. The endpoints it currently points at are the `/v2/oauth/authorize` and
`/v2/oauth/token` routes on the SSO host.

# The authorization request

The application sends the character to the authorization endpoint with:

* `response_type=code`
* `client_id` — the registered application ID
* `redirect_uri` — must match a callback registered for the application
* `scope` — a space-separated list of requested scopes
* `state` — an opaque anti-CSRF value the client verifies on return

A PKCE client additionally sends `code_challenge` (the base64url SHA-256 of a
random verifier, unpadded) and `code_challenge_method=S256`.

# Exchanging the code for tokens

The callback delivers an authorization `code`, which the application exchanges at
the token endpoint with a form-encoded POST (`grant_type=authorization_code` plus
the `code`):

* A confidential client authenticates with HTTP **Basic** auth, the username and
  password being its `client_id` and secret.
* A PKCE client sends the original `code_verifier` instead of a secret.

The response contains a short-lived **access token** and a long-lived **refresh
token**. The refresh token is exchanged (`grant_type=refresh_token`) for fresh
access tokens and remains valid until the character revokes access.

# The access token

The access token is a **JWT**, specific to one character and one set of scopes, and
valid only for a limited time. An application presents it to ESI as an
`Authorization: Bearer <token>` header. Because the token is a signed JWT, a
resource server validates it **locally**, without calling back to SSO:

* Verify the RSA signature against the keys published at the JWKS endpoint.
* `iss` (issuer) is `login.eveonline.com` (or its `https://` form).
* `aud` (audience) contains both the application's `client_id` and the literal
  string `"EVE Online"`.
* `exp` (expiry, a Unix timestamp) is in the future.

Useful claims inside the token:

* `sub` — the subject, formatted `CHARACTER:EVE:<character-id>`.
* `name` — the character name.
* `scp` — the array of granted scopes.

# The scope model

An application can reach only the data the character consented to at
authorization. Scopes not granted are inaccessible, and a character can revoke a
grant at any time, which invalidates the issued tokens. The scope each route
requires is declared in the [OpenAPI contract](./versioning.md).

# Citations

[1] EVE SSO documentation —
`https://developers.eveonline.com/docs/services/sso/`.
