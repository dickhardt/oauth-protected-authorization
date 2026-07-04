# Redirect Headers: Explainer Draft

_Last updated: 2025-12-03_

**Author:** Dick Hardt
**Email:** dick.hardt@hello.coop

## TL;DR

Redirect Headers move sensitive parameters from URLs to HTTP headers during browser redirects:
- **Redirect-Query** - Redirect parameters (replaces URL query string)
- **Redirect-Origin** - Browser-supplied origin (enables mutual authentication)
- **Redirect-Path** - Optional path validation (prevents redirect_uri manipulation)

**Why:** Authorization codes, tokens, and other sensitive data leak through URLs (browser history, Referer header, logs, analytics). Redirect Headers keep this data in headers that browsers control and don't expose to JavaScript or third parties.

**Primary use case:** Web server based OAuth/OIDC flows, but works for any protocol using browser redirects (SAML, proprietary auth flows). 

SPA, inapp mobile browsers are out of scope of this proposal.

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Problems with Existing Redirect Mechanisms](#2-problems-with-existing-redirect-mechanisms)
3. [Redirect Headers](#3-redirect-headers)
   - [3.1 Redirect-Query](#31-redirect-query)
   - [3.2 Redirect-Origin](#32-redirect-origin)
   - [3.3 Redirect-Path](#33-redirect-path)
4. [Use Cases](#4-use-cases)
5. [OAuth Redirect Security Threats](#5-oauth-redirect-security-threats)
6. [OAuth Example: Before and After](#6-oauth-example-before-and-after)
7. [OAuth Incremental Deployment](#7-oauth-incremental-deployment)
8. [Security and Privacy Considerations](#8-security-and-privacy-considerations)
9. [Feature Discovery](#9-feature-discovery)
10. [Implementation Status](#10-implementation-status)
11. [Related Documents](#11-related-documents)
12. [Acknowledgments](#12-acknowledgments)

## 1. Introduction

Authentication and authorization protocols (OAuth, OpenID Connect, SAML) use browser redirects to navigate users between applications and authorization servers. These redirects must carry protocol parameters, which historically appear in URLs or POSTed forms.

**Problem:** URLs leak sensitive data through browser history, Referer headers, server logs, analytics, and JavaScript access.

**Solution:** Redirect Headers move parameters into browser-controlled HTTP headers that aren't exposed in URLs or the DOM.

## 2. Problems with Existing Redirect Mechanisms
### URL-based redirects
- Parameters appear in the URL  
- URLs enter browser history  
- URLs are visible to scripts and extensions  
- URLs leak via the `Referer` header  
- URLs are logged in servers, proxies, analytics, crash reporting  

### POST redirects
- Parameters appear in DOM form fields  
- Visible to JavaScript  
- POST bodies may be logged or inspected  
- Cross-site POST requires `SameSite=None` cookies  

### `Referer` is unreliable
- May be stripped, rewritten, or truncated  
- May be removed by privacy tools and enterprise proxies  

Redirect Headers solve these limitations.

## 3. Redirect Headers

Three headers work together during top-level 302/303 redirects:

| Header | Set by | Direction | Purpose |
|--------|--------|-----------|---------|
| Redirect-Query | Server (client or AS) | Both directions | Carry parameters without URLs |
| Redirect-Origin | Browser | Both directions | Mutual origin authentication |
| Redirect-Path | Client | Client → AS | Validate redirect_uri path |

**Browser behavior:** Only processes these headers during top-level redirects. Ignores them for normal requests or embedded resources.  

### 3.1 Redirect-Query

Carries redirect parameters using URL query string encoding.

```http
Redirect-Query: "code=SplxlOBe&state=123"
```

- Replaces URL query parameters
- Parsed using standard URL query string parsing
- Prevents exposure via browser history, Referer, logs, and analytics
- Set by server (client or AS)

### 3.2 Redirect-Origin

Browser-supplied origin of the page initiating the redirect.

```http
Redirect-Origin: "https://app.example"
```

**Properties:**
- Set ONLY by the browser (cannot be spoofed by scripts)
- Enables AS to verify which client initiated the request
- Enables client to verify response came from correct AS
- Provides cryptographic mutual authentication (browser-mediated)

### 3.3 Redirect-Path

Optional path prefix for additional redirect_uri validation.

**Client sets:**
```http
Redirect-Path: "/app1/"
```

**Browser validates:**
1. Checks current URL path begins with `/app1/`
2. If valid: includes Redirect-Path in request to AS
3. If invalid: omits header (client cannot lie)

**AS enforces:**
- redirect_uri MUST begin with `Redirect-Origin` + `Redirect-Path`

This prevents redirect_uri manipulation attacks within the same origin.

---

## 4. Use Cases

**Primary: OAuth and OpenID Connect**
- Authorization code flow
- Implicit flow (though deprecated)
- Hybrid flows

**Other authentication protocols:**
- SAML assertions
- Proprietary SSO flows
- Any protocol requiring browser-mediated parameter passing

**When NOT to use:**
- Public, non-sensitive parameters that can appear in URLs
- Server-to-server communication (no browser involved)
- Protocols that don't use browser redirects

---

## 5. OAuth Redirect Security Threats

**Scope:** Redirect Headers specifically addresses OAuth and OIDC web-based redirect flows between websites where sensitive parameters are passed via URL query strings. This proposal does NOT address form_post mechanisms where data appears in the DOM, as that attack vector requires different mitigations.

### Authorization Code Theft from URL Query Strings

The primary security weakness is in the **authorization server's response** containing the authorization code in the URL query string. When an AS redirects back to the client with `?code=...&state=...`, the authorization code is exposed through multiple vectors:

**Known attack vectors:**

1. **Browser history leakage** - Authorization codes stored in browser history can be retrieved by attackers with device access
   - Reference: [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics-29)

2. **Server log exposure** - Authorization codes visible in web server access logs, proxy logs, and load balancer logs
   - Codes can be extracted in real-time or from archived logs

3. **Referer header leakage** - When the callback page loads third-party resources (images, scripts, analytics), the authorization code leaks via the Referer header
   - Reference: [OAuth 2.0 authentication vulnerabilities](https://portswigger.net/web-security/oauth)

4. **Browser-swapping attacks** - Attackers exploit scenarios where authorization codes leak through shared URLs or when users switch browsers during the flow
   - Discussion: [OAuth-WG Browser-Swapping thread](http://www.mail-archive.com/oauth@ietf.org/msg25296.html)
   - The core issue: "no technology to reliably verify that the user controlling the client is the same user who logs in to the AS/IdP"

5. **URL sharing** - Users may inadvertently share URLs containing authorization codes after errors or confusion

6. **Analytics and crash reporting** - Authorization codes captured by analytics systems, error tracking, and monitoring tools

**What Redirect Headers mitigates:**

By moving the authorization code from the URL query string to the `Redirect-Query` header, **all of these attack vectors are eliminated**. The authorization code never appears in:
- URLs (no browser history exposure)
- Referer headers (no leakage to third parties)
- Server logs (when servers are configured to not log sensitive headers)
- User-visible locations (no accidental sharing)

**Important clarification:**

The security concern is specifically the **authorization server's response** with the authorization code. The **client's authorization request** to the AS (containing `client_id`, `redirect_uri`, `state`) does not have known security concerns from being in the URL, as these parameters are not sensitive credentials. However, moving them to headers provides consistency and reduces URL clutter.

---

## 6. OAuth Example: Before and After

### Without Redirect Headers (current OAuth)

**Client Website returns to Browser:**
```http
HTTP/1.1 302 Found
Location: https://as.example/authorize?client_id=abc&state=123&redirect_uri=...
```

**Browser navigates, sends to AS:**
```http
GET /authorize?client_id=abc&state=123&redirect_uri=...
Host: as.example
Referer: https://app.example/login  ← Unreliable, may be stripped
```

**AS returns code to Browser:**
```http
HTTP/1.1 302 Found
Location: https://app.example/cb?code=SplxlOBe&state=123  ← Leaked in URL
```

**Browser sends code to Client Website:**
```http
GET /cb?code=SplxlOBe&state=123  ← In browser history, logs, analytics
Host: app.example
Referer: https://as.example/consent  ← Third-party resources see code via Referer
```

**Problems:**
- Authorization code appears in URL (history, logs, Referer, extensions)
- No cryptographic origin verification (Referer is optional and unreliable)

---

### With Redirect Headers

**Client Website returns to Browser:**
```http
HTTP/1.1 302 Found
Location: https://as.example/authorize?client_id=abc&state=123
Redirect-Query: "client_id=abc&state=123"
Redirect-Path: "/app1/"
```

**Browser navigates, adds origin and forwards to AS:**
```http
GET /authorize?client_id=abc&state=123
Host: as.example
Redirect-Origin: "https://app.example"  ← Browser-supplied, cannot be spoofed
Redirect-Path: "/app1/"
Redirect-Query: "client_id=abc&state=123"
```

**AS validates and returns to Browser:**
```http
HTTP/1.1 302 Found
Location: https://app.example/cb  ← No parameters in URL!
Redirect-Query: "code=SplxlOBe&state=123"
```

**Browser forwards back to Client Website:**
```http
GET /cb  ← Clean URL
Host: app.example
Redirect-Origin: "https://as.example"  ← Client verifies this
Redirect-Query: "code=SplxlOBe&state=123"  ← Not in URL, history, or Referer
```

**Benefits:**
- Authorization code never appears in URLs
- Mutual origin authentication (browser-verified)
- Backward compatible (browsers/servers without support fall back to URL parameters)

**Requirements:**
- If Redirect-Query received in request: AS MUST use Redirect-Query for response
- Client MUST verify Redirect-Origin matches expected AS
- AS MUST verify Redirect-Origin matches expected client
- When Redirect-Query is present, client MUST ignore URL parameters and use only header parameters

---

## 7. OAuth Incremental Deployment

Redirect Headers is designed for **incremental adoption** - each party (client, browser, authorization server) can independently add support, with functionality emerging when all parties support it.

### How It Works

**Clients** can start sending parameters in both locations:
- URL query string (for backward compatibility)
- `Redirect-Query` header (signaling support)

**Browsers** forward `Redirect-` headers when present:
- No special detection needed
- Simply forward the headers during redirects

**Authorization Servers** detect support and respond accordingly:
- If request includes `Redirect-Query` → AS knows both client and browser support it
- AS can then send response using **only** `Redirect-Query` header (no URL parameters)

### Adoption Path

Each party adds support independently, in any order:

```
Clients add support
  ├─ Send params in URL + Redirect-Query
  └─ Look for response in header, fall back to URL

Browsers add support
  ├─ Forward Redirect-* headers when present
  └─ No configuration needed

Authorization Servers add support
  ├─ Detect support when Redirect-Query received
  ├─ Confirm: client + browser both support it
  └─ Send response ONLY in Redirect-Query (omit URL params)

Result: Once all three support it → authorization code sent in header, not URL
```

**No coordination required** - each party adds support independently, and the system naturally converges to the secure behavior once all three support it. The client can immediately start sending both, browsers simply forward headers, and authorization servers detect support from incoming requests.

---

## 8. Security and Privacy Considerations

**Security:**
- Redirect-Query contains sensitive data - servers MUST treat as confidential
- Browsers MUST prevent JavaScript from reading or setting redirect headers
- Redirect-Path supplements but does NOT replace redirect_uri validation
- Network middleboxes (proxies, load balancers) can still observe headers
- Defense assumes honest browser (cannot protect against browser compromise)
- Transition period: URLs still leak until clients stop sending URL parameters

**Privacy:**
- Authorization codes and tokens removed from browser history
- No Referer leakage to third-party resources
- No exposure to JavaScript, browser extensions, or DOM inspection
- Reduced logging in analytics, crash reports, and server logs
- User-visible URLs are clean and don't reveal sensitive parameters

---

## 9. Feature Discovery

Some protocols may wish to explicitly discover browser support for Redirect Headers before relying on them. While not required for security (backward compatibility ensures graceful fallback), feature discovery can optimize behavior.

### Using Client Hints

Servers can advertise support and request browser capabilities using Client Hints:

**Server advertises support:**
```http
Accept-CH: Redirect-Supported
```

**Browser responds with capability:**
```http
Redirect-Supported: ?1
```

### Optimization for OAuth

Once both browser and AS support is confirmed through feature discovery, OAuth clients MAY send parameters **only** in `Redirect-Query`, omitting URL parameters entirely. This:
- Reduces URL length
- Prevents any possibility of parameter leakage via URLs
- Maintains backward compatibility (unsupported browsers fall back to URL parameters)

**Note:** Feature discovery is optional. The incremental deployment model ([Section 7](#7-oauth-incremental-deployment)) works without explicit discovery - authorization servers detect support by receiving `Redirect-Query` headers in requests.

---

## 10. Implementation Status

**Specification status:** Exploratory draft
**Browser support:** Not yet implemented (proposed specification)
**Server support:** Reference implementations needed

This specification requires:
- Browser vendors to implement header handling
- Authorization servers to support Redirect-Query
- Client applications to adopt the pattern

Deployment strategy: Backward compatible - clients send both URL and headers during transition.

---

## 11. Related Documents
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics-29)
- [OAuth 2.0 Threat Model and Security Considerations (RFC 6819)](https://tools.ietf.org/rfc/rfc6819.html)
- [OAuth-WG Browser-Swapping Discussion](http://www.mail-archive.com/oauth@ietf.org/msg25296.html)
- [IETF 121 OAuth Meeting Minutes](https://datatracker.ietf.org/doc/minutes-121-oauth-202411040930/)
- [OAuth 2.0 authentication vulnerabilities](https://portswigger.net/web-security/oauth)
- [Fetch Metadata Request Headers](https://w3c.github.io/webappsec-fetch-metadata/)
- [W3C Privacy Principles](https://w3ctag.github.io/privacy-principles/)
- [Client Hints Infrastructure](https://tools.ietf.org/html/rfc8942)

---

## 12. Acknowledgments

The author would like to thank early reviewers for their valuable feedback and insights that helped shape this proposal: Jonas Primbs,  Sam Goto, Warren Parad.
