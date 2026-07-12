---
title: "OAuth 2.0 and OpenID Connect (OIDC)"
category: concept
tags: [authentication, authorization, oauth, oidc, jwt, sessions, security, delegation, tokens]
sources: []
updated: 2026-07-12
---

# OAuth 2.0 and OpenID Connect (OIDC)

## What it is

**OAuth 2.0 is a delegated-*authorization* framework** — not an implementation of session tokens, and not the same axis as [[session-vs-jwt-auth]]. Its core problem:

> "How do I let App X access my data on Service Y **without giving App X my Service Y password**?"

Example: "Log in with Google," or letting a photo-printing site read your Google Photos. You never hand the printing site your Google password — Google issues it a **scoped, revocable access token** instead. That delegation is the whole point.

**OpenID Connect (OIDC)** is a thin *authentication* layer **on top of** OAuth 2.0. OAuth answers "can App X act on the user's behalf?"; OIDC adds "…and *who is* this user?" via an **ID token, which is always a JWT**.

## Common misconception

- ❌ "OAuth is an implementation of session tokens." / "OAuth = JWT."
- ✅ OAuth is orthogonal to the session-vs-JWT choice. It defines token **roles**, not token **formats**, and can use *either* implementation underneath.

## How it works

OAuth defines token roles, not implementations:

- **Access token** — used to call APIs. Can be implemented **either way**:
  - **Opaque / reference token** — a random string; the API validates it by calling the auth server's **introspection endpoint** (or a lookup). This is **stateful — session-like**.
  - **JWT** — self-contained, signature-verified **locally**. This is **stateless**.
- **Refresh token** — long-lived, exchanged for new access tokens; almost always **stateful** (stored server-side, revocable).
- **ID token** (OIDC only) — proves *who the user is*; **always a JWT**.

So OAuth **rides on top of** the session-vs-JWT decision — it doesn't equal either. JWT appears *inside* the OAuth/OIDC ecosystem as a token **format choice**.

### The layering

| Layer | What it answers | Artifacts |
|---|---|---|
| **OAuth 2.0** | *Delegated authorization* — can App X access this resource on the user's behalf? | Access token, refresh token, scopes |
| **OpenID Connect (OIDC)** | *Authentication* — who is this user? (built **on top of** OAuth) | **ID token (always a JWT)** |
| **Session vs JWT** | *Where does the token's state live?* | Opaque + store vs self-contained JWT |

## When to use / when not to

- **Use OAuth** when a **third party** (or a separate first-party client like an SPA/mobile app) needs delegated, scoped access to resources — instead of sharing credentials.
- **Use OIDC** when you also need to **authenticate the user** (get a verified identity), e.g. "Log in with Google/Okta/Auth0."
- **Don't** reach for OAuth for a simple monolith logging its own users in against its own DB — a plain session or first-party token is simpler. OAuth's delegation machinery earns its complexity when there are separate parties/clients.

## Trade-offs

- **Access token format** — opaque tokens give **instant, precise revocation** (introspection hits the auth server every call) at the cost of a per-call lookup; JWT access tokens are **fast and stateless** but inherit JWT's weak revocation (see [[session-vs-jwt-auth]]). A common middle ground: short-lived JWT access tokens + stateful refresh tokens.
- **Delegation vs simplicity** — OAuth solves credential-sharing and scoping, but adds authorization-server infrastructure, redirect flows, and token lifecycle management.

## Real-world examples

- **"Log in with Google/GitHub"** — OIDC on top of OAuth; the app receives an ID token (JWT) plus an access token.
- **BFF pattern** — from [[session-vs-jwt-auth]]: the SPA does OAuth, but the **backend holds the OAuth tokens** and hands the browser a plain `HttpOnly` **session cookie**. This is the cleanest illustration that OAuth ≠ JWT-in-the-browser — the OAuth tokens live server-side while the browser uses a session.
- **API access with scopes** — a service issued an access token scoped to `read:photos` cannot write or read anything else.

## See also
- [[session-vs-jwt-auth]] — the token-state axis OAuth rides on; BFF pattern
- [[service-to-service-auth]] — mTLS + propagated JWT internally, downstream of OAuth at the edge
- [[api-gateway-microservices-pattern]] — where OAuth token validation typically happens at the edge
