---
title: "CSRF (Cross-Site Request Forgery)"
category: concept
tags: [security, web, browser, cookies, csrf, authentication]
sources: []
updated: 2026-07-12
---

# CSRF (Cross-Site Request Forgery)

## What it is

CSRF tricks a user's browser into sending a **state-changing request** to a site the user is authenticated to, using the user's **automatically-attached credentials** — without the user's intent. It is a "write" attack: it never needs to read the response.

> **Root cause:** the browser attaches cookies based on the request's **destination**, regardless of **who initiated it** (ambient authority).

## The mechanism

User is logged into `bank.com` (session cookie in browser). User visits `evil.com`, which contains:

```html
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker">
  <input name="amount" value="10000">
</form>
<script>document.forms[0].submit()</script>
```

The browser sends the POST to `bank.com` **with the session cookie attached** — because the request goes to `bank.com`, and that is whose cookie it is. The server sees a valid authenticated request and executes the transfer. The user clicked nothing.

## Why the Same-Origin Policy doesn't stop it

- SOP **hides the response** from `evil.com`'s JavaScript — the attacker can't *read* the result.
- But the **request still executed**. The damage happens on **send**, not on read.

This is exactly why **CORS is not a CSRF defense**: CORS governs *reading responses*; CSRF only needs *sending a request*. See `[[cors]]`.

## Why it's fundamentally a cookie problem

CSRF works **only** because of **ambient authority** — credentials the browser sends automatically. This determines which auth designs are vulnerable:

| Auth mechanism | CSRF-vulnerable? | Why |
|---|---|---|
| Session cookie (`[[session-vs-jwt-auth]]`) | **Yes** | sent automatically by browser |
| JWT in `Authorization: Bearer` header | **No** | attacker's page cannot set that header cross-origin; not automatic |
| JWT in a cookie | **Yes** | still a cookie — automatic |

"We use JWT" does **not** imply CSRF-safe. It depends on **where the token lives**: header (safe) vs cookie (vulnerable).

## Defenses

### 1. `SameSite` cookies — modern first line

Tells the browser when to attach the cookie based on the request's **initiator**, attacking ambient authority at the root.

| Value | Behavior | Effect on CSRF |
|---|---|---|
| `Strict` | Sent only on same-site requests | Fully blocks CSRF; breaks cross-site navigation (link from email arrives logged out) |
| `Lax` | Same-site + top-level GET navigations | Blocks cross-site POST/PUT/DELETE/forms/XHR while keeping link-clicks. **Browser default today.** |
| `None` | All cross-site requests (must be `Secure`) | No protection — explicit opt-out |

**Catch:** `Lax` still sends the cookie on a **top-level GET navigation**. If the app performs state changes via GET (`GET /transfer?...` — which it never should), `Lax` won't protect it. Hence the HTTP contract matters: **GET must be safe/idempotent; mutations must be POST/PUT/DELETE.**

### 2. Synchronizer (CSRF) token — the classic

Server embeds a random, unguessable token in the rendered form; verifies it on every state-changing request:

```html
<input type="hidden" name="csrf_token" value="a8f3...unpredictable">
```

`evil.com` can *send* the POST but **cannot read** the token (SOP blocks reading `bank.com`'s page — here SOP *does* help), so it can't supply a valid one.

- **Cost:** server stores/tracks tokens → **stateful** (cuts against stateless `[[session-vs-jwt-auth]]`).

### 3. Double-submit cookie — stateless variant

Server sets a random value in **both** a cookie and expects it echoed in a **header/hidden field**; verifies header == cookie. Attacker can't read the cookie (SOP) to copy it, and can't set a custom cross-origin header → can't match. **No server storage.**

- **Caveat:** weaker with subdomain/XSS footholds (an attacker on any `*.myapp.com` may write the cookie). Sign the token to harden.

### 4. Custom-header requirement — JSON APIs

Require a custom header (e.g. `X-Requested-With`) on mutations. A cross-origin `<form>` can't set custom headers, and setting one via `fetch`/XHR triggers a CORS preflight the server can reject. This is why **pure JSON APIs with `Authorization: Bearer` are largely CSRF-immune** — the token isn't ambient.

## How to choose

Layered default:

```
1. SameSite=Lax (or Strict)                        ← baseline, on by default
2. + CSRF token (synchronizer or double-submit)    ← defense in depth for cookie auth
3. Enforce GET = safe, mutations = POST/PUT/DELETE  ← makes SameSite=Lax sufficient
```

| Setup | CSRF strategy |
|---|---|
| Server-rendered, session cookies (`[[session-vs-jwt-auth]]`) | `SameSite=Lax` + synchronizer token |
| SPA, token in `Authorization` header | Largely immune — no ambient auth |
| SPA, HttpOnly cookie / **BFF pattern** | `SameSite` + double-submit or custom header |
| Public JSON API, Bearer tokens | No CSRF concern (still need `[[oauth-oidc]]` authz) |

## Trade-offs

- **XSS defense and CSRF defense pull in tension.** Putting the token in an **HttpOnly cookie** protects it from XSS but **reintroduces ambient authority → CSRF is back**. The BFF pattern from `[[session-vs-jwt-auth]]` does exactly this, so a BFF must pair its cookie with `SameSite` + a CSRF token. You need **both** defenses; neither substitutes for the other.
- Synchronizer tokens = stronger but stateful; double-submit = stateless but weaker under subdomain/XSS.
- CSRF only concerns **cookie/ambient** auth; header-based bearer tokens sidestep it entirely — but then you own token storage/XSS risk instead.

## Real-world examples

- Auto-submitting hidden form on a malicious page targeting a bank transfer endpoint (classic).
- GET-based state change (`GET /logout`, `GET /delete?id=`) defeating `SameSite=Lax` — argument for strict HTTP-method discipline.

## See also

- [[xss]] — strictly more powerful; defeats most CSRF defenses by reading the token
- [[cors]] — often mistaken for a CSRF defense; it is not (governs reads, not sends)
- [[session-vs-jwt-auth]] — cookie vs header token determines CSRF exposure; BFF re-opens it
- [[oauth-oidc]] — authorization layer; orthogonal to CSRF
- [[service-to-service-auth]] — no CSRF between services (no browser/ambient cookies)
- [[reverse-proxy]] — same-origin routing that also simplifies cookie scoping
