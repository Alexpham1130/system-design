---
title: "XSS (Cross-Site Scripting)"
category: concept
tags: [security, web, browser, xss, injection]
sources: []
updated: 2026-07-12
---

# XSS (Cross-Site Scripting)

## What it is

XSS is an injection attack where the attacker gets **their JavaScript to run inside your page's origin**. Once that happens the attacker code executes with the full privileges of your page — it is *inside* your defenses, not working around them.

> **Position in the hierarchy:** XSS is strictly more powerful than `[[csrf]]`. CSRF makes the browser *send* a request; XSS makes the attacker *become the page*.

## Why it sits above CSRF

If attacker code runs in your page, it can **read** the CSRF token, **read** the double-submit cookie, and forge a perfectly valid request — defeating nearly every CSRF defense. The one thing it cannot read is an **`HttpOnly` cookie**, which is exactly why `HttpOnly` exists and exactly why the XSS↔CSRF tension noted in `[[csrf]]` is real:

- Put the auth token in a **JS-readable** place → XSS steals it.
- Put it in an **`HttpOnly` cookie** → XSS can't read it, but the cookie is now ambient → CSRF is back.

You need to defend **both**; neither substitutes for the other.

## The three types

| Type | Where the payload lives | Example |
|---|---|---|
| **Stored (persistent)** | Saved server-side, served to every viewer | `<script>` in a comment/profile field runs for everyone who views it. Most dangerous — worm-able. |
| **Reflected** | Bounced off the server from the request | `search?q=<script>…` echoed unescaped into the results page; delivered via a crafted link. |
| **DOM-based** | Pure client-side, never touches the server | JS reads `location.hash` and assigns it to `element.innerHTML`. Server logs show nothing. |

## Defenses (priority order)

### 1. Context-aware output encoding — the real fix
Escape data **for the context where it lands**. HTML body, HTML attribute, JS string, URL, and CSS each need *different* escaping (`<` → `&lt;` in an HTML body is not what you need inside a `<script>` or an `href`). Modern frameworks (React, Angular, Vue) auto-escape by default — which is why the escape hatches (`dangerouslySetInnerHTML`, `v-html`, `[innerHTML]`) are precisely where XSS creeps back in.

### 2. Content Security Policy (CSP) — defense-in-depth backstop
An HTTP header declaring which script sources may run (`script-src 'self'`). Even if a payload is injected, an inline `<script>` won't execute. **Trusted Types** extends this by forcing dangerous DOM sinks to accept only sanitized values.

### 3. Sanitization for rich HTML
When user HTML must be allowed (WYSIWYG editor), run it through an **allowlist** sanitizer (DOMPurify). Never regex; never blocklist.

### 4. `HttpOnly` cookies
Doesn't prevent XSS but limits blast radius — script can't steal the session cookie. This is the knob that trades against CSRF (see `[[csrf]]`, `[[session-vs-jwt-auth]]`).

### 5. Input validation
Useful hygiene, **not** the primary defense. The fix belongs at **output** (where data meets a context), because the same value can be safe in one context and dangerous in another.

## Mental model vs CSRF

- **CSRF** is fixed at the **boundary** — `SameSite` + tokens govern who may send a request.
- **XSS** is fixed at the **point of output** — encode for context, with CSP as the safety net.

## When it matters

Anywhere untrusted data (user input, URL params, third-party content, even DB values that originated as user input) is rendered into a page or fed to a DOM sink. Server-rendered apps, SPAs, and email/HTML rendering are all in scope.

## Trade-offs & pitfalls

- **Blocklist/regex sanitization always loses** — encoders/allowlists win.
- **Framework auto-escaping lulls teams** into thinking they're safe until someone reaches for `innerHTML`.
- **CSP is a backstop, not a primary fix** — a strict, nonce-based CSP is powerful but operationally fiddly (inline scripts, third-party widgets).
- Input validation at the edge does **not** replace output encoding.

## Real-world examples

- Stored XSS in a social feed comment → self-propagating worm (the classic MySpace "Samy").
- Reflected XSS via an unescaped search parameter delivered by phishing link.
- DOM XSS from `innerHTML = location.hash` in a client-side router.

## See also

- [[csrf]] — XSS defeats most CSRF defenses; the `HttpOnly`↔CSRF tension
- [[cors]] — a different browser mechanism; does not defend against XSS
- [[session-vs-jwt-auth]] — token storage (JS-readable vs `HttpOnly`) is the XSS/CSRF pivot
- [[oauth-oidc]] — stolen tokens via XSS undermine any authz scheme
