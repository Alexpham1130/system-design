---
title: "Session vs JWT Authentication"
category: trade-off
tags: [authentication, jwt, sessions, security, stateless, microservices, oauth, bff]
sources: []
updated: 2026-07-11
---

# Session vs JWT Authentication

The core question: where does the *state* that proves "who this request is" live? **Sessions keep it server-side** (the token is a reference); **JWT carries it inside the token** (the token is the data). Everything else — cost, scaling, revocation, where each fits — follows from that one difference.

## The two models

**Session tokens (stateful / server-side)**
- The token is an **opaque random string** (a session ID) — a *reference*, not data.
- Real session state lives in a **shared store** (DB, Redis, Memcached).
- Any server in the fleet can validate by looking up the store → works behind a load balancer with no sticky sessions.

**JWT (stateless / self-contained)**
- The token **is** the data: `header.payload.signature`, base64-encoded.
- Payload (claims: `user_id`, `roles`, `exp`) is **encoded, not encrypted** — anyone can read it. Never put secrets in it.
- The security work is **verifying the signature**, not "decoding." Signature proves the token was issued by you and is untampered.
- No store lookup required in the basic case.

## Per-request cost

Both push cost somewhere; they don't eliminate it.

| | Session | JWT |
|---|---|---|
| Per-request cost | Network round-trip to store (~0.5–2ms to Redis) | CPU for signature verify (HMAC ~µs; RSA/ECDSA ~0.1–1ms) |
| Where cost lands | I/O + a shared dependency | Local CPU only |
| Scaling pain | Store becomes a hot dependency / SPOF | None per-request |

HMAC-signed JWT verification is often **cheaper** than a Redis round-trip. That's the appeal: no shared state on the hot path.

## The trade-off that actually matters: revocation

The real axis is not decode-time vs query-time — it's **control**.

- **Session**: log out / ban / kill a stolen token → delete the row. Instant and precise, because you already hit the store every request.
- **JWT**: valid until it **expires**, by definition — nobody checks a store. You **can't** easily revoke a live JWT. Every workaround claws back statefulness:
  - Short expiry + refresh tokens (the refresh token is usually stateful, stored in DB).
  - A denylist of revoked JTIs → now you're doing a store lookup again, partly reinventing sessions.

> **JWT trades away easy revocation and control to gain statelessness. Sessions trade a shared-store dependency to gain instant, precise control.**

## Where each fits — by boundary, not by "microservices vs client-server"

The common "JWT for microservices, sessions for client-server" framing merges two separate boundaries. Split them:

| Boundary | Typical choice | Driving reason |
|---|---|---|
| Browser, server-rendered ↔ edge | Session cookie | Auto-managed, `HttpOnly`, easy revocation |
| SPA ↔ backend (browser) | `HttpOnly` cookie / BFF (trending) — *not* JWT in `localStorage` | XSS resistance |
| Mobile ↔ API | JWT (access) + refresh token | Secure native storage (Keychain/Keystore); no cookie model |
| Service ↔ service (internal) | JWT for **identity propagation** + mTLS for service authn | Avoid shared session-store bottleneck on every hop |

At the edge, the choice is driven by **client type**, not by whether the backend is microservices. A microservices backend serving a classic web app can use session cookies.

### Client type: mobile vs SPA diverge

- **Mobile** — JWT genuinely dominates and fits well: secure OS-level storage, no browser XSS surface, natural `Authorization: Bearer` usage.
- **SPA in a browser** — "SPA ⇒ JWT" is a **myth being actively corrected**. Storing a JWT in `localStorage` is an anti-pattern: any JS (a bad dependency, an XSS) can read and exfiltrate it. An `HttpOnly` cookie is unreadable by JS → strictly smaller blast radius. Popularity here lags best practice (a wave of ~2015–2018 tutorials taught `localStorage` JWT).

### The BFF (Backend-for-Frontend) pattern

Current mainstream recommendation for SPAs; dissolves the dilemma by using **both**:

1. SPA talks to its own small backend (the BFF) using a plain `HttpOnly` session cookie.
2. The BFF holds the real OAuth/JWT tokens **server-side** and attaches them to downstream API calls.
3. The browser **never sees a JWT**.

Browser gets cookie safety; internal services still get JWTs for identity propagation.

### Service-to-service: why JWT shines internally

- If every internal service hit a central session store per request, that store becomes a hot dependency on every hop (a 5-service chain = 5 lookups).
- Instead: the edge validates the user **once**, then propagates identity as a **signed, short-lived JWT**. Each service verifies the signature **locally** (CPU only, no I/O). No shared bottleneck.
- The revocation weakness is tolerable here: these JWTs are short-lived (seconds–minutes) and never leave the trust boundary.
- **Caveat:** service *authentication* (proving service A is really A) is usually **mTLS**, not JWT — see [[service-to-service-auth]] for the full mTLS + JWT layering. JWT here carries the *end-user's* identity through the chain, not the services' identity. See [[api-gateway-microservices-pattern]] — the gateway is where "validate once, propagate inward" happens.

## Hybrid is the norm

A common production stack uses **all of these at once**: short-lived stateless JWT access tokens (fast, no lookup) + stateful refresh tokens (revocable) + mTLS between services + `HttpOnly` cookies at the browser edge. Not either/or — different tools at different boundaries.

## See also
- [[oauth-oidc]] — OAuth is orthogonal to this axis; its access tokens can be opaque (session-like) or JWT
- [[service-to-service-auth]] — mTLS + propagated JWT for internal hops
- [[api-gateway-microservices-pattern]] — where edge auth and JWT propagation happen
- [[reverse-proxy]] — TLS termination and the edge where sessions/tokens are validated
- [[aws-api-gateway]] — managed front door with built-in auth/throttling
