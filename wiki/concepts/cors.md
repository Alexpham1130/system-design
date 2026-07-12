---
title: "CORS (Cross-Origin Resource Sharing)"
category: concept
tags: [security, web, browser, http, same-origin-policy]
sources: []
updated: 2026-07-12
---

# CORS (Cross-Origin Resource Sharing)

## What it is

CORS is a browser mechanism that lets a server **relax** the Same-Origin Policy (SOP) and allow a web page from one origin to read responses from a different origin. It is a *controlled relaxation of a default block*, not a feature that adds access.

> **Mental model:** SOP is the wall. CORS is the door the server chooses to open. The browser enforces both.

## The Same-Origin Policy underneath it

An **origin** = `scheme + host + port`. All three must match to be "same-origin."

| URL A | URL B | Same origin? | Why |
|---|---|---|---|
| `https://myapp.com` | `https://api.myapp.com` | No | different host |
| `https://myapp.com` | `http://myapp.com` | No | different scheme |
| `https://myapp.com` | `https://myapp.com:8443` | No | different port |
| `https://myapp.com/a` | `https://myapp.com/b` | Yes | path is irrelevant |

Registrable domain (`myapp.com`) does **not** matter — `www.` and `api.` subdomains are already distinct origins.

Under SOP the browser will still **send** a cross-origin request, but it **hides the response** from JavaScript unless the server returns the right CORS headers.

## How it works

**Simple requests** (GET/POST/HEAD, only safelisted headers, standard content-types) go straight through; the browser checks the response for `Access-Control-Allow-Origin` and blocks the JS from reading it if absent/mismatched.

**Preflight** — anything else (custom headers like `Authorization`, methods like `PUT`/`DELETE`, `Content-Type: application/json`) triggers an `OPTIONS` preflight *before* the real request:

```
Browser → OPTIONS /api/orders
          Origin: https://myapp.com
          Access-Control-Request-Method: PUT
          Access-Control-Request-Headers: authorization, content-type

Server  → 204
          Access-Control-Allow-Origin: https://myapp.com
          Access-Control-Allow-Methods: GET, PUT, DELETE
          Access-Control-Allow-Headers: authorization, content-type
          Access-Control-Max-Age: 600        (cache the preflight)

Browser → PUT /api/orders   (only now does the real request go)
```

## The credentialed-request rule (where most CORS bugs live)

When the request carries **credentials** (cookies, HTTP auth, client certs), two constraints kick in:

1. `Access-Control-Allow-Credentials: true` must be present, **and**
2. `Access-Control-Allow-Origin` **cannot be `*`** — the server must echo the *specific* origin.

So the server reads the request's `Origin` header, checks it against an allowlist, and reflects it:

```
Origin: https://myapp.com                          (request; browser-set, JS cannot forge)
↓
Access-Control-Allow-Origin: https://myapp.com     (response; echoed, not "*")
Access-Control-Allow-Credentials: true
Vary: Origin                                        (so caches don't cross origins)
```

Note: a `Bearer` token in an `Authorization` header is **not** a "credential" in the CORS sense — you don't need `Allow-Credentials` for it, just allow the `Authorization` header.

## Where the headers get configured

CORS headers are HTTP **response** headers, set by whichever layer sends the response to the browser:

- **Application code** (most common): `cors` middleware (Express), `@CrossOrigin` / `CorsConfigurationSource` (Spring), `CORSMiddleware` (FastAPI).
- **Edge** — `[[reverse-proxy]]` (Nginx `add_header`) or API gateway (`[[aws-api-gateway]]` CORS config), so backends never think about CORS.

## The architectural alternative: avoid CORS entirely

Instead of splitting `myapp.com` (frontend) and `api.myapp.com` (backend), put a `[[reverse-proxy]]` in front and route by path so everything is **same-origin**:

```
myapp.com/          → frontend (static / CDN)
myapp.com/api/*     → backend service(s)
```

Now there is **zero CORS config** and cookies are **first-party** (increasingly important as third-party cookies are deprecated).

| Approach | CORS needed? | Cookies | Notes |
|---|---|---|---|
| Separate `api.` subdomain | Yes, must configure | Cross-site (needs `SameSite`/domain care) | Clean separation, independent scaling/CDN |
| Same-origin via reverse proxy / gateway | None | First-party, simple | One more hop; couples FE/BE routing |

An `[[api-gateway-microservices-pattern]]` extends this: one public origin, fan-out by sub-path to many services, with auth/rate-limiting applied once at the edge. Killing CORS and applying auth happen to live in the same box but are **independent** wins.

## When to use / when not to

- **Use CORS config** when the browser genuinely must call a different origin you control (or a partner's API) — e.g. a public API consumed by third-party web apps, or an SPA that must hit `api.` directly.
- **Prefer the same-origin proxy** when you own both frontend and backend and want to avoid CORS + keep cookies first-party.

## Trade-offs & pitfalls

- **CORS is not a security boundary.** It is enforced *only in browsers*. `curl`, Postman, and other backends ignore it and get the response regardless. It protects the *user*, not the *server*. Real access control still needs `[[oauth-oidc]]` / `[[service-to-service-auth]]`.
- **CORS is not a CSRF defense** — arguably the opposite (it *loosens* SOP). The attack SOP/`SameSite` actually defend against is `[[csrf]]`.
- **`Allow-Origin: *` + credentials** is the classic misconfiguration — silently blocked by browsers.
- **Reflecting `Origin` without an allowlist** is an over-permissive footgun (effectively "allow all, with credentials").
- Forgetting `Vary: Origin` causes caches/CDNs to serve one origin's ACAO to another.

## Real-world examples

- SPA at `myapp.com` calling `api.myapp.com` — must configure CORS with echoed origin + `Allow-Credentials` if using cookies.
- Same team collapsing both behind `[[aws-api-gateway]]` or Nginx at `myapp.com/api/*` — CORS disappears.

## See also

- [[reverse-proxy]] — the primitive that makes same-origin routing possible
- [[api-gateway-microservices-pattern]] — one origin, fan-out to many services + edge policy
- [[aws-api-gateway]] — managed edge where CORS can be configured
- [[oauth-oidc]] — the actual authorization layer (CORS is not one)
- [[service-to-service-auth]] — real security boundaries behind the gateway
- [[csrf]] — the attack SOP/SameSite defend against (CORS does not)
