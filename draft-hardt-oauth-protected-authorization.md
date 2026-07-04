%%%
title = "OAuth Protected Authorization"
abbrev = "Protected Authorization"
ipr = "trust200902"
area = "Security"
workgroup = "Web Authorization Protocol"
keyword = ["oauth", "authorization", "browser", "security", "privacy"]

[seriesInfo]
status = "standard"
name = "Internet-Draft"
value = "draft-hardt-oauth-protected-authorization-latest"
stream = "IETF"

date = 2026-07-03T00:00:00Z

[[author]]
initials = "D."
surname = "Hardt"
fullname = "Dick Hardt"
organization = "Hellō"
  [author.address]
  email = "dick.hardt@gmail.com"

[[author]]
initials = "S."
surname = "Goto"
fullname = "Sam Goto"
organization = "Google"
  [author.address]
  email = "goto@google.com"

%%%

<reference anchor="JARM" target="https://openid.net/specs/oauth-v2-jarm.html">
  <front>
    <title>JWT Secured Authorization Response Mode for OAuth 2.0 (JARM)</title>
    <author initials="T." surname="Lodderstedt" fullname="Torsten Lodderstedt">
      <organization>yes.com</organization>
    </author>
    <author initials="B." surname="Campbell" fullname="Brian Campbell">
      <organization>Ping Identity</organization>
    </author>
    <date year="2022" month="November"/>
  </front>
</reference>

<reference anchor="FETCH" target="https://fetch.spec.whatwg.org/">
  <front>
    <title>Fetch Living Standard</title>
    <author>
      <organization>WHATWG</organization>
    </author>
    <date year="2026"/>
  </front>
</reference>

.# Abstract

This document defines browser support for protecting OAuth 2.0 authorization requests and authorization responses during redirect-based authorization flows. A single Structured Field header field, OAuth-Authorization, is set by the OAuth client in the redirect response that sends the browser to the authorization server, and by the authorization server in the redirect response that returns the browser to the OAuth client. In both cases the browser augments the header with the attested origin of the redirecting party and delivers it to the redirect destination. The mechanism provides security for the authorization request: the authorization server receives a browser-attested, tamper-evident statement of which origin initiated the request. The mechanism provides security and privacy for the authorization response: the authorization code is delivered in the browser-protected header instead of the redirect URI, and never appears in a URL, eliminating its exposure through browser history, server logs, Referer headers, analytics systems, and URL sharing. The header is generated, validated, and delivered by the browser, and is inaccessible to scripts, service workers, and browser extensions. Existing OAuth deployments continue to function unchanged; the protections activate only when the OAuth client, browser, and authorization server all support them.

.# Discussion Venues

*Note: This section is to be removed before publishing as an RFC.*

Source for this draft and an issue tracker can be found at https://github.com/dickhardt/oauth-protected-authorization.

{mainmatter}

# Introduction

OAuth 2.0 [@!RFC6749] redirect-based authorization flows carry protocol parameters in URLs. The critical parameter is the **authorization code**: it is a credential, exchangeable for tokens. When the authorization server redirects back to the OAuth client with `?code=...&state=...` in the redirect URI, that credential is written into browser history, web server access logs, proxy and load balancer logs, and analytics systems; it leaks through Referer headers to third-party resources loaded by the callback page; and it can be disclosed through URL sharing, screenshots, and copy/paste. The OAuth 2.0 Security Best Current Practice [@!RFC9700] documents these leakage vectors and prescribes mitigations that limit the damage of a leaked code (PKCE [@RFC7636], one-time use, short lifetimes), but the code is still exposed.

A second, related weakness is that the authorization server has no reliable way to know which web origin sent the user. The Referer header may be trimmed, stripped, or rewritten, and everything else in the authorization request is claimed by the requester rather than attested by anyone.

This document defines browser behavior and a single Structured Field [@!RFC9651] header field, `OAuth-Authorization`, used on both legs of the flow:

- For the **authorization request**, the OAuth client sets the header in the redirect response that sends the browser to the authorization server. The browser delivers it with the navigation, adding a browser-attested `origin` (and optionally a browser-validated `path`) identifying where the navigation actually came from. Its presence also signals to the authorization server that the browser and OAuth client support protected authorization responses.

- For the **authorization response**, the authorization server sets the header in the redirect response that returns the browser to the OAuth client, placing the authorization response parameters in the header instead of the redirect URI. The browser delivers it, exactly once, to the redirect URI, adding a browser-attested `origin` identifying the authorization server.

The browser performs the same processing in both cases ((#browser-processing)); which OAuth message the header carries is determined by the endpoint that receives it.

The mechanism provides **security for the authorization request, and security and privacy for the authorization response**:

- The authorization request parameters remain in the request URI, unchanged, so existing authorization servers continue to work. The header adds origin attestation, tamper evidence, and a capability signal. This is security without privacy: the OAuth client cannot know whether a given user's browser supports this mechanism, so the parameters never leave the URL.

- The authorization response parameters, above all the authorization code, exist only in the protected header and never appear in any URL. The authorization server sends them this way only after the header's arrival with the authorization request has proven that the browser and OAuth client support it.

Existing deployments continue to function unchanged, and adoption is deliberately inexpensive: for most clients, support arrives as an update to the OAuth library they already use, with no new endpoints, no keys, and no change to how authorization parameters are constructed or parsed ((#deployment-considerations)).

## Goals

1. Provide an explicit browser signal that a navigation is an OAuth authorization request.
2. Provide a protected, browser-mediated delivery mechanism for OAuth authorization responses.
3. Preserve complete compatibility with existing OAuth deployments and parameter processing.

This mechanism complements PKCE [@RFC7636], PAR [@RFC9126], `iss` identification [@RFC9207], and JARM [@JARM]. It does not replace any of them.

## Non-Goals

- This is not a back-channel protocol. Protocols that move the authorization exchange out of the front channel entirely (such as PAR [@RFC9126] for the request, or the AAuth protocol [@?I-D.hardt-oauth-aauth-protocol] for the full authorization exchange) provide stronger confidentiality with different deployment costs. See (#rationale-back-channel).
- This is not a generic HTTP redirect mechanism. It is intentionally specific to OAuth authorization flows. See (#rationale-oauth-specific).
- This mechanism does not protect against a compromised or non-conforming browser, a malicious authorization server, or a malicious OAuth client.
- Native application flows [@RFC8252] are out of scope. See (#rationale-native).

## Browser Enforcement Precondition

The security properties defined in this document are provided by browser behavior, not by the header name or value. The header MUST be inaccessible to page JavaScript, to `fetch()` and `XMLHttpRequest`, to service workers, and to browser extensions ((#browser-processing)). **A browser that cannot enforce all of the protections in (#browser-processing) (for example, because its extension APIs expose all request headers) MUST NOT implement this mechanism.** In that case the flow degrades safely to standard OAuth: no header is generated, and the authorization server responds with parameters in the redirect URI as it does today.

# Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [@!RFC2119] [@!RFC8174] when, and only when, they appear in all capitals, as shown here.

This document uses the terms "authorization request", "authorization response", "authorization server" (AS), "client", and "redirect URI" as defined by OAuth 2.0 [@!RFC6749], and "origin" as defined by [@!RFC6454].

**Client**: in this document, "client" always means the OAuth client as defined by [@!RFC6749], the application requesting authorization. The HTTP client performing navigations is always referred to as "the browser".

**Browser**: a user agent performing top-level navigations on behalf of a user.

**Redirecting server / redirecting URL**: the server, and the URL, from which a redirect response was received. The browser attests the origin (and optionally a path prefix) of the redirecting URL, which is well defined even when the redirect is the first response in a navigation (e.g., a login URL that immediately returns a redirect).

**Authorization request and response** always refer to the OAuth messages [@!RFC6749], never to HTTP requests and responses. At each redirect hop, the `OAuth-Authorization` header field appears in two HTTP messages: the redirecting server sets it in the redirect *response*, and the browser delivers it as a header field of the subsequent HTTP *request* to the redirect destination. This document always states which HTTP message is meant.

# Protocol Overview

The flow is standard OAuth; the additions introduced by this specification are marked **(new)**:

1. The client constructs a normal authorization request URI and returns a redirect response (302 or 303) to the browser. **(new)** The redirect response also carries an `OAuth-Authorization` header whose `query` member is the same serialized query string that appears in the authorization request URI.
2. **(new)** The browser recognizes the header, adds the browser-attested `origin` member (and `path` if claimed and valid), and delivers it as a header field of the HTTP request to the authorization server. The request URI is unchanged.
3. The AS processes the authorization request from the URI as it does today. **(new)** If the `OAuth-Authorization` header is present, the AS verifies the `query` member matches the request URI query, and now knows (a) the browser-attested origin of the client and (b) that the browser and client support protected authorization responses.
4. The AS authenticates the user and obtains authorization, using any number of internal redirects or pages; the header plays no role in these intermediate steps.
5. The AS returns a redirect response to the registered redirect URI. **(new)** Instead of placing the authorization response parameters in the redirect URI query, the AS places them in the `query` member of an `OAuth-Authorization` header. The redirect URI carries no response parameters.
6. **(new)** The browser adds the browser-attested `origin` of the AS and delivers the header, exactly once, as a header field of the HTTP request to the redirect URI.
7. **(new)** The client reads the authorization response parameters from the header's `query` member, using the same parsing it uses for URL query strings today, and verifies the browser-attested `origin` is the expected AS.

If any party does not support the mechanism, the header is simply absent and the flow proceeds as standard OAuth.

# The OAuth-Authorization Header Field {#header-field}

`OAuth-Authorization` is a Structured Field Dictionary [@!RFC9651] with three members, all Strings: `query`, `origin`, and `path`. The redirecting server sets the header in a redirect response; the browser validates and augments it, and delivers it as a header field of the subsequent HTTP request to the redirect destination ((#browser-processing)).

A recipient determines which OAuth message the header carries from its own role: an authorization endpoint receives authorization requests; a redirect URI receives authorization responses. Delivery to the wrong context fails closed ((#security-cross-context)).

## query {#query-member}

Set by the redirecting server; relayed by the browser without inspection or modification.

The `query` member carries the OAuth parameters, using the same serialization as a URL query string, so recipients parse it with the code they already use for URLs today:

- In an **authorization request** ((#authorization-request)), the client sets `query` to the query component of the authorization request URI, byte-identical to it.
- In an **authorization response** ((#authorization-response)), the AS sets `query` to the serialized authorization response parameters that would otherwise appear in the redirect URI query.

## origin

Set only by the browser: the ASCII serialization [@!RFC6454] of the redirecting URL's origin. Servers MUST NOT set `origin`; the browser removes any server-set value before setting its own. A receiving party can therefore rely on `origin` unconditionally: it cannot be set, suppressed, or modified by web content or by any server.

## path

Claimed by the redirecting server; validated, and set or removed, by the browser.

The redirecting server MAY include a `path` member as a claim about the path prefix of the redirecting URL; the value MUST begin and end with `/`. The browser delivers the member only if the redirecting URL's path begins with the claimed value, and removes it otherwise ((#path-validation)). A receiving party can therefore rely on a delivered `path` unconditionally: the browser only delivers claims it has verified.

# Authorization Request {#authorization-request}

## Client Behavior

The client constructs the authorization request URI exactly as it does today; all authorization request parameters remain in the URI query for compatibility. In the redirect response (302 or 303) that sends the browser to the authorization server, the client additionally sets the header:

```
HTTP/1.1 303 See Other
Location: https://as.example/authorize?client_id=abc&response_type=code
    &redirect_uri=https%3A%2F%2Fapp.example%2Fportal%2Fcb&state=123
    &code_challenge=E9Melhoa...&code_challenge_method=S256
OAuth-Authorization: query="client_id=abc&response_type=code
    &redirect_uri=https%3A%2F%2Fapp.example%2Fportal%2Fcb&state=123
    &code_challenge=E9Melhoa...&code_challenge_method=S256",
    path="/portal/"
```

(Line breaks in the examples are for readability only.)

The `query` member MUST be byte-identical to the query component of the `Location` URI. The client MAY include a `path` claim ((#path-validation)).

The browser processes the header per (#browser-processing) and delivers it with the navigation:

```
GET /authorize?client_id=abc&response_type=code&redirect_uri=...
    &state=123&code_challenge=...&code_challenge_method=S256 HTTP/1.1
Host: as.example
OAuth-Authorization: query="client_id=abc&response_type=code
    &redirect_uri=...&state=123&code_challenge=...
    &code_challenge_method=S256",
    origin="https://app.example", path="/portal/"
```

Note that the browser does NOT compare the `query` member with the `Location` URI query. Dropping the header on a mismatch would allow an attacker able to tamper with the URI to silently downgrade the flow to unprotected behavior; instead the header is always delivered and the authorization server detects tampering ((#as-behavior-request)).

## Authorization Server Behavior {#as-behavior-request}

The AS processes the authorization request from the request URI query exactly as it does today. Deployments that do not implement this specification require no changes.

An AS that implements this specification and receives an `OAuth-Authorization` header with an authorization request:

1. MUST verify that the `query` member is byte-identical to the query component of the request URI, and reject the authorization request with an error if they differ. A mismatch indicates tampering or a defective client and MUST NOT be silently ignored.
2. SHOULD verify that the `origin` member is consistent with the client's registered `redirect_uri` values (e.g., that the registered redirect URIs for the presented `client_id` belong to the attested origin), and reject the request otherwise. Unlike the Referer header, `origin` is browser-attested: it cannot be set, suppressed, or modified by web content.
3. MAY use a validated `path` member to discriminate between multiple clients registered on the same origin under different path prefixes.
4. MUST return the authorization response using the `OAuth-Authorization` header as defined in (#authorization-response). The header's arrival with the authorization request is the capability signal: it can only have arrived via a supporting browser from a supporting client.

On this leg the header provides origin attestation and tamper evidence; it does not provide confidentiality. The authorization request parameters remain visible in the URL, as they are today.

# Authorization Response {#authorization-response}

## Authorization Server Behavior

An AS whose authorization request arrived bearing the `OAuth-Authorization` header MUST deliver the authorization response in the header, and MUST NOT include response parameters in the redirect URI:

```
HTTP/1.1 303 See Other
Location: https://app.example/portal/cb
OAuth-Authorization: query="code=SplxlOBeZQQYbYS6WxSbIA
    &state=123&iss=https%3A%2F%2Fas.example"
```

The AS delivers error responses through the same header, for consistency: once support is established, every authorization response, success or error, arrives through the same channel, giving the client a single processing path. (Keeping error details out of URLs and logs is a secondary benefit.)

```
HTTP/1.1 303 See Other
Location: https://app.example/portal/cb
OAuth-Authorization: query="error=access_denied&state=123
    &iss=https%3A%2F%2Fas.example"
```

The AS MAY include a `path` claim ((#path-validation)).

An AS MUST NOT include an `OAuth-Authorization` header in its response redirect unless the corresponding authorization request arrived bearing the header: without that signal there is no evidence the browser will deliver the header or that the client will read it.

The browser processes the header per (#browser-processing) and delivers it with the navigation, exactly once, to the redirect URI only:

```
GET /portal/cb HTTP/1.1
Host: app.example
OAuth-Authorization: query="code=SplxlOBeZQQYbYS6WxSbIA
    &state=123&iss=https%3A%2F%2Fas.example",
    origin="https://as.example"
```

## Client Behavior

A client that receives an `OAuth-Authorization` header at its redirect URI:

1. MUST obtain the authorization response parameters by parsing the `query` member as a URL query string, and MUST ignore any query parameters present in the request URI.
2. MUST verify that the `origin` member matches the origin of the authorization server it sent the corresponding authorization request to, and reject the response otherwise. This browser-attested check complements the `iss` parameter [@RFC9207] and is stronger: `iss` is asserted by whichever server sent the response.
3. Processes the parameters (including `state` verification and the token request with PKCE) exactly as it does for URL-delivered responses today.

A client that receives authorization response parameters in the URL (because the user's browser, or the AS, does not support this mechanism) processes them as it does today. A client can never require the header: URL delivery may simply mean an unsupporting browser (see (#security-downgrade)).

Because the header is delivered only once, clients MUST complete processing (or persist what they need) on first delivery; a reload of the redirect URI carries neither the header nor any parameters. Note that this also means an authorization code can no longer be replayed out of browser history: there is nothing in the URL or the history entry to replay.

# Path Validation {#path-validation}

Multiple clients may share an origin, separated by path prefix (e.g., `https://host.example/app1/` and `https://host.example/app2/`). The origin alone cannot discriminate between them. The `path` member allows the receiving party to know which path prefix within the origin the redirect actually came from.

The `path` member is a claim made by the redirecting server and validated by the browser:

1. The redirecting server includes `path` in its header with a value that MUST begin and end with `/` (e.g., `path="/app1/"`).
2. The browser checks whether the path component of the redirecting URL begins with the claimed value.
3. If it does, the browser includes the `path` member in the delivered header.
4. If it does not, the browser removes the `path` member and delivers the header without it.

The trailing `/` requirement ensures prefix matching occurs on path-segment boundaries (`/app1/` does not match `/app1evil/x`).

The redirecting server cannot lie about its path: the browser only delivers a path claim it has verified. A receiving party that requires path discrimination treats an absent `path` member accordingly (e.g., an AS that registered a client with a path-scoped redirect URI SHOULD reject a request whose validated `path` is absent or inconsistent with the registration).

# Browser Processing Model {#browser-processing}

This is fundamentally a browser behavior specification: the security properties of the `OAuth-Authorization` header field are created entirely by the browser processing rules in this section. The browser acts as a trusted protocol participant, a role it already plays for every OAuth flow today by enforcing TLS, cookie isolation, redirect handling, and origin boundaries. This section makes that role explicit.

**Recognition.** The browser processes the header when, and only when, it appears in a 302 or 303 response to a top-level navigation. No URL heuristics are used. The header is ignored (and stripped) in all other contexts: subresource requests, fetch/XHR responses, embedded frames, and non-redirect responses.

**Identical processing on both legs.** The browser applies the same rules whether the redirecting server is a client sending an authorization request or an AS returning an authorization response. The browser does not parse or interpret the `query` member; in particular, it does not compare it with the `Location` URI ((#rationale-mismatch)).

**Single hop, stateless.** The header is relayed exactly one hop: from the redirect response in which the redirecting server set it, to the HTTP request for the `Location` URI, and no further. The browser keeps no state about an OAuth flow across hops; if an intermediate destination redirects again, that response must itself set the header for the browser to relay it. Any transient state used to relay the header is destroyed after delivery.

**Browser-generated members.** The browser removes any `origin` member set by a server and sets it itself; the browser validates any server-claimed `path` member and delivers it only if valid. Receiving parties can therefore rely on `origin` and `path` unconditionally.

**Protection requirements.** The browser MUST ensure that:

- Web content cannot set this header field on any request: it MUST be treated as a forbidden header name in the sense of Fetch [@FETCH].
- JavaScript cannot read this header field from any request or response, including via `fetch()`, `XMLHttpRequest`, performance and reporting APIs, or any other API.
- Service workers cannot observe or modify this header field, on either the redirect responses that carry it or the requests that deliver it, including via navigation preload.
- Browser extensions cannot read, modify, inject, suppress, or replay this header field, regardless of the permissions the extension holds.
- The header field is processed only for top-level navigations, delivered only to the `Location` destination of the redirect that carried it, delivered at most once, and never persisted.

These protections are intentionally stronger than those of `Sec-`-prefixed header fields, which today remain readable by extensions and service workers. A browser that cannot enforce every requirement in this list MUST NOT implement this mechanism (see (#security-precondition)).

**Standards coordination.** The processing model in this section (the forbidden header name, service worker opacity, and navigation integration) requires a normative change to the WHATWG Fetch standard [@FETCH]. A Fetch pull request defining this processing model is a deliverable of this work; this document defines the OAuth protocol semantics that rely on it.

# Deployment Considerations

## Incremental Adoption

No coordination between parties is required. Each party adds support independently, in any order:

- **Clients** add the `OAuth-Authorization` header to the redirect responses they already send, keeping all parameters in the URL, and read authorization responses from the header when present, falling back to URL parameters when it is not. This is a library update.
- **Browsers** implement the processing model in (#browser-processing).
- **Authorization servers** verify and use the header when it arrives with an authorization request, and switch the authorization response to the header, whose arrival proves the whole path supports it.

Until all three parties support the mechanism, every flow proceeds exactly as standard OAuth. There is no flag day and no breakage for any non-supporting party.

## Dual Transmission of Request Parameters

Clients send the authorization request parameters in both the URL and the header on every request, indefinitely: a client cannot know whether any given user's browser supports this mechanism, and this specification deliberately defines no discovery mechanism. This is why the authorization request gains security but not privacy ((#privacy-request)), and why the authorization response, which is sent in the header only when support is proven, gains both.

## Single-Page Applications

The authorization response is delivered to the server at the redirect URI; it is, by design, not readable by JavaScript ((#rationale-spa)). SPAs that process authorization codes in front-end code continue to work exactly as they do today (they will not receive the header because they do not send it from a server-issued redirect response). SPAs that adopt a backend-for-frontend gain the protections of this specification.

## Header Sizes

Authorization code responses are small (typically well under 1 KB). Deployments that layer large response payloads into the `query` member, such as JARM [@JARM] response JWTs, should be aware of intermediary header size limits, commonly 8 to 16 KB.

# Security Considerations

## Dependence on Browser Enforcement {#security-precondition}

Everything in this specification depends on the browser protections in (#browser-processing). If extensions, service workers, or page script can observe or forge the header, the mechanism provides no security benefit over URL parameters and MUST NOT be implemented. This is the central deployment requirement of this specification and is intentionally stated bluntly: the security properties are defined by browser behavior, and exist only where the browser provides them.

This specification assumes an honest, conforming browser. It cannot protect against a compromised or malicious browser, browser bugs that fail to enforce the processing model, or debugging tools operating with the user's authority. Servers SHOULD treat anomalous `origin` values as potential indicators of a non-conforming implementation.

## Downgrade {#security-downgrade}

An attacker who prevents the mechanism from operating (for example, by interfering with a non-protected portion of the flow) obtains at most today's OAuth: parameters in URLs, with all [@!RFC9700] mitigations (PKCE, `state`, one-time short-lived codes, exact redirect URI matching) still in force. A client cannot distinguish "the user's browser does not support this" from "support was stripped", and therefore can never hard-require the header; this residual downgrade is accepted and is exactly the status quo. The browser is stateless across the flow ((#browser-processing)), so it cannot mark a callback as "should have been protected"; a future extension could revisit this if browsers ever maintain per-flow state.

Query tampering, by contrast, is not a downgrade vector: the browser always delivers the header with the authorization request, and the AS rejects on mismatch ((#as-behavior-request)).

## Cross-Context Delivery {#security-cross-context}

A single header field name means a header could in principle reach a party expecting the other OAuth message. Both directions fail closed:

- An authorization endpoint that receives a header whose `query` member is not byte-identical to the request URI query rejects the request ((#as-behavior-request)); a header carrying authorization response parameters never matches an authorization request URI.
- A client that receives a header at its redirect URI verifies the browser-attested `origin` is its expected AS ((#authorization-response)). A header minted by any other party, including an attacker's site issuing a redirect to the client's redirect URI, carries the attacker's origin and is rejected. Web content and extensions cannot forge the header at all ((#browser-processing)).

## What Is and Is Not Protected

Protected: the authorization code, and the other authorization response parameters, never appear in URLs, eliminating exposure via browser history, web server and proxy logs, Referer headers, analytics and crash reporting, URL sharing, screenshots, and copy/paste; and both parties receive a browser-attested origin for the other side of each redirect.

Not protected: parameters in transit (TLS remains REQUIRED for every HTTP request and response carrying the header); the authorization request parameters, which remain in the URL by design; the parties themselves (a malicious AS or client is out of scope); and injection of an attacker's *legitimate* authorization response at a client (authorization code injection), for which PKCE remains REQUIRED.

## Server-Side Handling

When carrying an authorization response, the `query` member contains the authorization code and MUST be treated with the confidentiality of an `Authorization` header: excluded or redacted in access logs, application logs, and telemetry. Because the same field name is used on both legs, the simple deployment rule is to treat `OAuth-Authorization` as sensitive everywhere. Moving parameters out of URLs removes them from *default* URL logging; header logging is a configuration choice that servers MUST make deliberately.

Receiving parties MUST parse the header as a Structured Field [@!RFC9651] and reject malformed values. An HTTP request or response carrying more than one instance of the header field is invalid; recipients MUST ignore the header field entirely in that case, and browsers MUST NOT relay duplicated instances.

## Relationship to Existing Mechanisms

This mechanism supplements and does not relax any existing requirement: redirect URI registration and exact matching, `state` (or equivalent CSRF protection), PKCE [@RFC7636], and the mitigations of [@!RFC9700] all continue to apply. The `origin` member provides a browser-attested complement to `iss` [@RFC9207] for mix-up defense, and browser-attested client origin strengthens the AS's ability to detect requests initiated from unexpected origins, supplementing rather than replacing redirect URI validation.

# Privacy Considerations

## Authorization Request Privacy {#privacy-request}

Authorization request parameters remain in the URL indefinitely ((#deployment-considerations)) and remain exposed exactly as they are today. This is explicitly a non-goal: authorization request parameters (`client_id`, `redirect_uri`, `state`, `code_challenge`) are not secrets. Deployments that need request confidentiality should use PAR [@RFC9126].

## Authorization Response Privacy

Authorization response parameters never appear in URLs, browser history, Referer headers, or default logs, and the header is invisible to page JavaScript, embedded third parties, service workers, and extensions. The header is delivered once and never persisted.

## No New Cross-Site Information Flow

This mechanism activates only on OAuth authorization navigations: flows that by design already convey the client's identity to the AS (`client_id`, `redirect_uri`) in the URL. The browser-attested `origin` gives the AS nothing it does not already receive; it makes an existing claim reliable rather than adding a new one. On the return leg, the AS origin delivered to the client identifies a party the client chose and already knows. The header is invisible to all third parties, so it cannot be used as a side channel between origins. What remains true, and is unchanged by this specification, is that front-channel OAuth inherently reveals to the AS that a user is authorizing a given client; only back-channel protocols such as [@?I-D.hardt-oauth-aauth-protocol] change that.

## Origin and Path Disclosure

The `origin` member is a reliable equivalent of a scheme-plus-host Referer, and a validated `path` member discloses a path prefix, but only to the party the redirect was addressed to, which in OAuth already knows the counterparty. Unlike Referer, these members cannot be stripped by the user without the flow degrading to URL parameters (which disclose strictly more). This trade-off, favoring reliable counterparty verification over origin hiding, is appropriate only where mutual knowledge of the parties is expected, as it is in OAuth; it is one of the reasons this mechanism is OAuth-specific rather than generic ((#rationale-oauth-specific)).

## User Transparency

Users cannot inspect or modify the header in-page (unlike URL parameters). Browsers SHOULD surface it in developer tools, read-only, while maintaining all protections in (#browser-processing).

# IANA Considerations

This document registers one header field in the "Hypertext Transfer Protocol (HTTP) Field Name Registry" defined in [@!RFC9110].

## OAuth-Authorization Header Field

Field name: OAuth-Authorization

Status: permanent

Structured Type: Dictionary

Reference: [this document]

# Implementation Status

**Note to RFC Editor: Please remove this section before publication.**

Specification status: Exploratory draft. This document replaces draft-hardt-httpbis-redirect-headers, refocused on the OAuth use case following IETF 125 feedback.

Browser support: not yet implemented; requires the Fetch processing model in (#browser-processing).

Server and client support: reference implementations needed; adoption is a library update for both.

# Acknowledgments

The authors would like to thank early reviewers for their valuable feedback and insights that helped shape this proposal: Jonas Primbs, Warren Parad. This document was refocused on the OAuth use case in response to feedback from the HTTPBIS working group at IETF 125, in particular from Martin Thomson, Justin Richer, David Waite, Mike Bishop, Li Ruochen, and Yaroslav Rosomakho.

{backmatter}

# Example: Authorization Code Flow Before and After

## Without This Mechanism (Current OAuth)

Client redirects to the AS:

```
HTTP/1.1 303 See Other
Location: https://as.example/authorize?client_id=abc&response_type=code
    &redirect_uri=https%3A%2F%2Fapp.example%2Fportal%2Fcb&state=123
    &code_challenge=E9Melhoa...&code_challenge_method=S256
```

The AS has only the unreliable Referer header to know where the user came from. After authorization, the AS redirects back:

```
HTTP/1.1 303 See Other
Location: https://app.example/portal/cb?code=SplxlOBeZQQYbYS6WxSbIA
    &state=123&iss=https%3A%2F%2Fas.example
```

The authorization code is now in the URL: recorded in browser history, in the client's access logs and any intermediary's logs, sent in the Referer header to any third-party resource the callback page loads, and available to be shared, screenshotted, or pasted.

## With This Mechanism

Client redirects to the AS; the URL is identical to today, plus the header:

```
HTTP/1.1 303 See Other
Location: https://as.example/authorize?client_id=abc&response_type=code
    &redirect_uri=https%3A%2F%2Fapp.example%2Fportal%2Fcb&state=123
    &code_challenge=E9Melhoa...&code_challenge_method=S256
OAuth-Authorization: query="client_id=abc&response_type=code
    &redirect_uri=https%3A%2F%2Fapp.example%2Fportal%2Fcb&state=123
    &code_challenge=E9Melhoa...&code_challenge_method=S256",
    path="/portal/"
```

The redirecting URL is `https://app.example/portal/login`, so the browser validates the `/portal/` path claim, attests the origin, and delivers:

```
GET /authorize?client_id=abc&response_type=code&redirect_uri=...
    &state=123&code_challenge=...&code_challenge_method=S256 HTTP/1.1
Host: as.example
OAuth-Authorization: query="client_id=abc&response_type=code
    &redirect_uri=...&state=123&code_challenge=...
    &code_challenge_method=S256",
    origin="https://app.example", path="/portal/"
```

The AS verifies the `query` member matches the URL, verifies `https://app.example` is consistent with the registered redirect URI for `client_id=abc`, and, knowing the browser and client support protected authorization responses, returns the response in the header with a clean redirect URI:

```
HTTP/1.1 303 See Other
Location: https://app.example/portal/cb
OAuth-Authorization: query="code=SplxlOBeZQQYbYS6WxSbIA
    &state=123&iss=https%3A%2F%2Fas.example"
```

The browser attests the AS origin and delivers, once:

```
GET /portal/cb HTTP/1.1
Host: app.example
OAuth-Authorization: query="code=SplxlOBeZQQYbYS6WxSbIA
    &state=123&iss=https%3A%2F%2Fas.example",
    origin="https://as.example"
```

The client verifies `origin` is its expected AS, parses the `query` member with its existing query-string parser, verifies `state`, and exchanges the code (with its PKCE verifier). The authorization code never appeared in any URL: nothing in browser history, nothing in URL-based logs, nothing in Referer headers, nothing to share or replay.

# Design Rationale

## Why OAuth-Specific Rather Than Generic Redirect Headers {#rationale-oauth-specific}

A generic redirect-header mechanism gives the browser no way to know when protections apply, no defined trust model, and a broad new surface for navigation tracking. Scoping to OAuth lets the browser recognize exactly one flow shape, attach exactly the metadata that flow needs, and constrain the mechanism to navigations that already carry cross-site identifiers by design.

## Why a Single Header Field {#rationale-single}

The browser's processing is identical on both legs of the flow, so one header field name means one processing rule, one forbidden header name in Fetch, and one IANA registration. Separate "request" and "response" names would also invite conflating the OAuth message direction with the HTTP message direction: the header appears in an HTTP response and then an HTTP request on *each* leg. The receiving endpoint's role already determines which OAuth message the header carries, and delivery to the wrong context fails closed ((#security-cross-context)).

## Why Not Cryptographic Protection of Parameters

The exposure this document addresses is at the endpoints, not on the channel: TLS already protects parameters in transit. An encrypted or signed response (JARM [@JARM]) placed in a URL is still a URL: still written to history, still in access logs, still sent in Referer headers, still shareable. Cryptographic protection also requires key distribution between every client and AS pair, and no cryptographic scheme can attest a client's *web origin*: only the user agent knows which origin actually initiated a navigation. The mechanisms compose: a JARM response can be carried in the header's `query` member, gaining URL-freedom on top of its integrity properties.

## Why Not Move to the Back Channel {#rationale-back-channel}

Not putting sensitive parameters in the front channel at all is the strongest protection, and it is a different protocol. PAR [@RFC9126] moves the authorization request to the back channel; the AAuth protocol [@?I-D.hardt-oauth-aauth-protocol] moves the entire authorization exchange to authenticated back-channel HTTP. Those approaches carry different deployment costs: new endpoints, client authentication, and protocol changes on both sides. This specification exists for the enormous installed base of redirect-based OAuth deployments, which can adopt it as a library update with no new endpoints, no keys, and no change to how authorization parameters are constructed or parsed.

## Why the Authorization Request Parameters Stay in the URL

Compatibility, permanently. The authorization request URI is processed by every existing AS with no changes, and the client can never know whether the user's browser supports this mechanism, so it always sends both ((#deployment-considerations)). The `query` member on this leg is not for confidentiality (authorization request parameters are not secrets) but provides a canonical copy bound to browser-attested origin metadata, a tamper check ((#as-behavior-request)), and the capability signal that unlocks the protected authorization response.

## Why a Structured Field Dictionary Wrapping a Query String

The outer Structured Field [@!RFC9651] layer gives the browser a well-defined, extensible place for the members it owns (`origin`, `path`) without colliding with the OAuth parameter namespace. The OAuth parameters themselves stay query-string encoded inside the `query` member, so every implementation parses them with the same code path it uses for URLs today: no new parameter encoding, and no dual-parser inconsistency bugs.

## Why Redirect Responses Only (Why Not form_post)

This mechanism applies exclusively to redirect responses. The `form_post` response mode places parameters in the document as form fields, where they are visible to page JavaScript and extensions no matter what the browser does with headers; there is nothing there for the browser to protect. Redirects are also what the overwhelming majority of deployments use, and extending them is a library change rather than an infrastructure change.

## Why the AS Rejects on Query Mismatch, Rather Than the Browser Dropping the Header {#rationale-mismatch}

If the browser dropped the header when the `query` member disagreed with the URL, an attacker able to modify the URL query could make the header vanish, silently downgrading the flow to unprotected URL parameters. Instead the browser always delivers the header, and the AS treats a mismatch as an error ((#as-behavior-request)). Tampering becomes a visible failure rather than a silent downgrade.

## Why Not a Sec-Prefixed Name

The `Sec-` prefix means only that web content cannot *set* a header; extensions and service workers can still read `Sec-` request headers today. The protections required here are strictly stronger ((#browser-processing)) and are defined by normative browser behavior plus forbidden-header-name registration in Fetch. A `Sec-` prefix would understate the guarantee being made, and would break the symmetry of the same header field flowing from a redirect response into the browser's subsequent HTTP request.

## Why Not a response_mode Value {#rationale-response-mode}

The protected authorization response is functionally a new response mode, but it is negotiated by demonstrated browser capability (the header's arrival with the authorization request) rather than requested by the client. A `response_mode` parameter would allow a client to request behavior the user's browser may not support, which cannot work. In-band capability signaling is the only reliable negotiation.

## Why the Authorization Response Is Not Readable by JavaScript {#rationale-spa}

Deliberate. [@!RFC9700] and current browser-based-app guidance steer authorization codes away from script-accessible surfaces, where they are exposed to XSS, injected third-party code, and extensions. Delivering the authorization response only to the server at the redirect URI enforces that guidance structurally.

## Why Native Application Flows Are Out of Scope {#rationale-native}

The header cannot survive the custom-scheme or app-link handoff from the system browser back to a native app: the operating system delivers a URL, not headers. The threat model also differs: dedicated system authentication sessions do not run page JavaScript or extensions in the same way. Native flows continue to use their existing mechanisms (with PKCE, per [@RFC8252]).

## Why Browser-Attested Origin When iss Exists

The `iss` parameter [@RFC9207] is asserted by whichever server sends the response; in the mix-up scenarios it targets, the client must trust the asserter. The `origin` member is attested by the browser and cannot be set or influenced by any server. The two compose: `iss` travels unchanged inside the `query` member, and the client additionally gets an unforgeable statement of which origin the response actually came from.
