# OAuth Protected Authorization: Explainer

_Last updated: 2026-07-03_

**Author:** Dick Hardt
**Email:** dick.hardt@gmail.com

## TL;DR

The browser becomes a trusted participant in OAuth redirect flows, using one new browser-protected header, **`OAuth-Authorization`**, set by the OAuth client when sending the authorization request, and by the authorization server when returning the authorization response. The browser's processing is identical on both legs: it attests the origin of the redirecting party and delivers the header to the redirect destination.

| Leg | Header set by | Browser adds | Provides |
|-----|---------------|--------------|----------|
| Authorization request | OAuth client | `origin` (+ validated `path`) | **Security, not privacy**: browser-attested client origin, tamper evidence, capability signal. Request parameters stay in the URL. |
| Authorization response | Authorization server | `origin` (+ validated `path`) | **Security and privacy**: the authorization code never appears in any URL. |

**Why:** The authorization code is a credential. When an authorization server redirects back to the client with `?code=...&state=...`, that credential lands in browser history, server and proxy logs, analytics, and Referer headers, and can be shared, screenshotted, or pasted. Moving the response into a browser-protected header eliminates that entire leakage class. And unlike the Referer header, the browser-attested `origin` gives each party a reliable statement of who actually sent the redirect.

**The critical requirement:** this only works because the browser shields the header from *everything*: page JavaScript, `fetch()`, service workers, and browser extensions. A browser that cannot enforce that must not implement the mechanism; the flow then simply behaves as standard OAuth.

**Scope:** web-server-based OAuth/OIDC redirect flows. SPAs handling codes in front-end JavaScript, `form_post`, and native app flows are out of scope (see [FAQ](#7-faq)).

---

## Table of Contents
1. [The Problem](#1-the-problem)
2. [How It Works](#2-how-it-works)
3. [The Header](#3-the-header)
4. [Path Validation](#4-path-validation)
5. [Browser Requirements](#5-browser-requirements)
6. [Incremental Deployment](#6-incremental-deployment)
7. [FAQ](#7-faq)
8. [Status](#8-status)

## 1. The Problem

OAuth 2.0 redirect flows carry protocol parameters in URLs. Two distinct weaknesses:

**The authorization code leaks.** The redirect back to the client contains the authorization code, a credential exchangeable for tokens, in the URL query. URLs are recorded in browser history, logged by web servers, proxies, and load balancers, captured by analytics and crash reporting, sent to third parties via the Referer header, and casually shared by users. RFC 9700 (OAuth Security BCP) documents these vectors and mandates mitigations (PKCE, one-time short-lived codes) that limit the damage of a leaked code, but the code is still exposed.

**Origins are unverified.** The authorization server cannot reliably tell which web origin sent the user: Referer is stripped and trimmed by browsers, privacy tools, and proxies, and everything else in the request is claimed by the requester, attested by no one.

## 2. How It Works

Standard OAuth, with the browser doing one new thing on each leg. The client's redirect to the AS gains the header (parameters stay in the URL, unchanged):

```
HTTP/1.1 303 See Other
Location: https://as.example/authorize?client_id=abc&response_type=code
    &redirect_uri=...&state=123&code_challenge=...
OAuth-Authorization: query="client_id=abc&response_type=code
    &redirect_uri=...&state=123&code_challenge=...", path="/portal/"
```

The browser validates the `path` claim against the redirecting URL, attests the origin, and delivers the header with the navigation:

```
GET /authorize?client_id=abc&... HTTP/1.1
Host: as.example
OAuth-Authorization: query="client_id=abc&...",
    origin="https://app.example", path="/portal/"
```

The AS processes the URL exactly as today. The header tells it two things: the browser-attested origin of the client, and, because the header can only have arrived via a supporting browser from a supporting client, that it can return a **protected response**:

```
HTTP/1.1 303 See Other
Location: https://app.example/portal/cb
OAuth-Authorization: query="code=SplxlOBe&state=123
    &iss=https%3A%2F%2Fas.example"
```

The browser attests the AS origin and delivers the header, exactly once, to the redirect URI:

```
GET /portal/cb HTTP/1.1
Host: app.example
OAuth-Authorization: query="code=SplxlOBe&state=123&iss=...",
    origin="https://as.example"
```

The client parses the `query` member with the same code it uses for URL query strings, verifies `origin` is its expected AS, and proceeds (state check, token request with PKCE) as today. **The authorization code never appeared in any URL**: nothing in history, nothing in logs, nothing in Referer headers, nothing to share or replay.

If any party doesn't support the mechanism, the header is simply absent and the flow is standard OAuth. Nothing breaks.

## 3. The Header

`OAuth-Authorization` is a Structured Field Dictionary (RFC 9651):

| Member | Type | Set by | Value |
|--------|------|--------|-------|
| `query` | String | server (client or AS) | Serialized OAuth parameters, same encoding as a URL query string |
| `origin` | String | **browser only** | Origin of the URL that issued the redirect. Any server-set value is stripped. |
| `path` | String | **browser only** (validating a server claim) | Path prefix of the redirecting URL, present only if validated |

Design points:

- **One name, both legs.** The browser's processing is identical whether the redirect came from the client or the AS, and the receiving endpoint's role determines which OAuth message the header carries: authorization endpoints receive authorization requests, redirect URIs receive authorization responses. Naming the header per-leg would also invite confusing the OAuth message direction with the HTTP message direction: the header appears in an HTTP response and then an HTTP request on *each* leg. Cross-context delivery fails closed (the query-match rule at the AS; the origin check at the client).
- **The `query` member reuses URL query-string encoding**, so every OAuth implementation parses it with code it already has. The Structured Field wrapper exists so the browser has a collision-free, extensible place for the members it owns.
- **With the authorization request, `query` must be byte-identical to the URL query.** The browser does *not* compare them, because dropping the header on mismatch would let an attacker who tampers with the URL silently downgrade the flow. Instead the browser always delivers the header, and the AS rejects on mismatch: tampering becomes a visible error.
- **With the authorization response, the header replaces the URL parameters entirely.** Error responses travel the same way: once the AS supports the mechanism, every authorization response arrives through the same channel, giving the client a single processing path (keeping error details out of logs is a side benefit, not the goal).
- **Single hop, stateless browser.** The header is relayed from the redirect response that set it to the `Location` destination, and no further. AS-internal redirects (login, consent) are unaffected; the AS keeps its own state and sets the header on its final redirect.
- **Delivered once.** With the authorization response, the header is never persisted and never re-sent on reload or back/forward navigation. A pleasant side effect: with nothing in the URL and nothing replayable in history, authorization codes can no longer be replayed out of browser history at all.

## 4. Path Validation

Multiple OAuth clients can share an origin, separated by path (`https://host.example/app1/`, `https://host.example/app2/`). Origin alone cannot discriminate between them, so a server may *claim* a path prefix, which the browser *validates*:

1. Server sets `path="/app1/"` (must begin and end with `/`).
2. Browser checks the redirecting URL's path starts with `/app1/`.
3. Valid → the delivered header includes `path="/app1/"`. Invalid → the member is removed.

The receiving party can trust a `path` it receives, because the browser only delivers verified claims. The trailing-`/` rule keeps matching on path-segment boundaries (`/app1/` never matches `/app1evil/x`).

## 5. Browser Requirements

This is not an ordinary header, and its security properties come entirely from browser behavior, not from its name. The browser must ensure:

- Web content **cannot set** it (a forbidden header name in Fetch).
- JavaScript **cannot read** it via `fetch()`, `XMLHttpRequest`, performance/reporting APIs, or anything else.
- Service workers **cannot observe or modify** it, including via navigation preload.
- Browser extensions **cannot read, modify, inject, suppress, or replay** it, regardless of permissions.
- It is processed **only for 302/303 redirects of top-level navigations**, delivered only to the redirect's destination, at most once, never persisted.

This is deliberately stronger than `Sec-` header semantics: extensions and service workers can read `Sec-` request headers today, which is exactly why this header doesn't use that prefix. The processing model lands as a PR to the WHATWG Fetch standard; that PR is a deliverable of this work.

**A browser that cannot enforce all of this must not implement the mechanism.** The flow then degrades safely to standard OAuth. Stated bluntly: the security properties of this proposal exist only where the browser provides them.

## 6. Incremental Deployment

No coordination, no flag day. Each party adopts independently, in any order:

- **Clients** (a library update): add the `OAuth-Authorization` header to redirects they already send; read authorization responses from the header when present, fall back to URL parameters when not.
- **Browsers**: implement the processing model.
- **Authorization servers**: when the header arrives with an authorization request, verify it and switch the authorization response to the header, whose arrival proves the whole path supports it.

Until all three line up, every flow is exactly standard OAuth.

**The dual-send is permanent.** A client can never know whether a given user's browser supports the mechanism, and there is deliberately no discovery mechanism. Authorization request parameters therefore stay in the URL forever, which is why the authorization request gains security but not privacy, while the authorization response (sent in the header only when support is proven) gains both.

## 7. FAQ

**Why not protect the parameters with cryptography (JARM, signed request objects)?**
The exposure is at the endpoints, not on the wire; TLS already covers transit. An encrypted response in a URL is still a URL: still in history, still in logs, still in Referer headers. Cryptography also requires key distribution per client/AS pair, and no signature can attest a client's *web origin*: only the browser knows which origin initiated a navigation. The mechanisms compose: a JARM JWT can ride in the header's `query` member.

**Why not keep sensitive data out of the front channel entirely?**
That's the strongest answer, and it's a different protocol. PAR (RFC 9126) back-channels the request; the [AAuth protocol](https://datatracker.ietf.org/doc/draft-hardt-oauth-aauth-protocol/) moves the entire authorization exchange to authenticated back-channel HTTP. Those come with new endpoints, client authentication, and protocol changes on both sides. This proposal is for the enormous installed base of redirect-based OAuth: a library update, no new endpoints, no keys.

**Why is this OAuth-specific instead of generic redirect headers?**
A generic mechanism gives the browser no way to know when protections apply, no trust model, and a wide navigation-tracking surface. Scoping to OAuth lets the browser recognize one flow shape and constrain the mechanism to navigations that already carry cross-site identifiers by design. (This was the HTTPBIS feedback at IETF 125, and it was right.)

**What about SPAs?**
The authorization response is delivered to the server at the redirect URI and is unreadable by JavaScript. This is deliberate, since RFC 9700 and browser-based-app guidance steer codes away from script-accessible surfaces. SPAs handling codes in front-end JS keep working exactly as today (they never trigger the mechanism); SPAs with a backend-for-frontend gain the protections.

**What about native apps?**
Out of scope. The OS hands the app a URL, not headers, so the header cannot survive the custom-scheme/app-link handoff, and system auth sessions don't face the same page-JS/extension threats. Native flows keep RFC 8252 + PKCE.

**Doesn't this add a tracking channel?**
No new cross-site information flows. The mechanism activates only on OAuth navigations, which already carry `client_id` and `redirect_uri` to the AS in the URL; the browser-attested origin makes an existing claim reliable rather than adding a new one. The header is invisible to all third parties, so it can't be used as a side channel.

**Is this a new `response_mode`?**
Functionally yes, but it's negotiated by demonstrated browser capability (the header arriving with the authorization request), not requested by the client. A `response_mode` parameter would let a client request behavior the user's browser can't deliver.

## 8. Status

- **Spec:** [draft-hardt-oauth-protected-authorization](https://github.com/dickhardt/oauth-protected-authorization) replaces draft-hardt-httpbis-redirect-headers (archived in [archive/](archive/)), refocused on OAuth following HTTPBIS feedback at IETF 125.
- **Venue:** OAuth working group (protocol), with a WHATWG Fetch PR for browser behavior.
- **Browser support:** not yet implemented (Chromium has expressed interest).
- **Related:** RFC 9700 (Security BCP) · PKCE (RFC 7636) · PAR (RFC 9126) · `iss` (RFC 9207) · JARM · [AAuth](https://datatracker.ietf.org/doc/draft-hardt-oauth-aauth-protocol/)

## Acknowledgments

Thanks to Jonas Primbs and Warren Parad for early review, and to the HTTPBIS working group at IETF 125, in particular Martin Thomson, Justin Richer, David Waite, Mike Bishop, Li Ruochen, and Yaroslav Rosomakho, whose feedback drove the refocus on OAuth.
