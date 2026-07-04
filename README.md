# OAuth Protected Authorization

This is the working area for the individual Internet-Draft, "OAuth Protected Authorization".

* [Editor's Copy](https://dickhardt.github.io/oauth-protected-authorization/draft-hardt-oauth-protected-authorization.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-hardt-oauth-protected-authorization)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-hardt-oauth-protected-authorization)
* [Compare Editor's Copy to Individual Draft](https://dickhardt.github.io/oauth-protected-authorization/#go.draft-hardt-oauth-protected-authorization.diff)

This document replaces [draft-hardt-httpbis-redirect-headers](https://datatracker.ietf.org/doc/draft-hardt-httpbis-redirect-headers), refocused on the OAuth use case following feedback from the HTTPBIS working group at IETF 125. The prior draft and explainer are preserved in [archive/](archive/).

## Abstract

This document defines browser support for protecting OAuth 2.0 authorization requests and authorization responses during redirect-based authorization flows. A single Structured Field header field, OAuth-Authorization, is set by the OAuth client in the redirect response that sends the browser to the authorization server, and by the authorization server in the redirect response that returns the browser to the OAuth client. In both cases the browser augments the header with the attested origin of the redirecting party and delivers it to the redirect destination. The mechanism provides security for the authorization request: the authorization server receives a browser-attested, tamper-evident statement of which origin initiated the request. The mechanism provides security and privacy for the authorization response: the authorization code is delivered in the browser-protected header instead of the redirect URI, and never appears in a URL, eliminating its exposure through browser history, server logs, Referer headers, analytics systems, and URL sharing. The header is generated, validated, and delivered by the browser, and is inaccessible to scripts, service workers, and browser extensions. Existing OAuth deployments continue to function unchanged; the protections activate only when the OAuth client, browser, and authorization server all support them.

## Additional Resources

* [Explainer Document](explainer.md) - Detailed explanation, use cases, and examples

## Contributing

See the [guidelines for contributions](https://github.com/dickhardt/oauth-protected-authorization/blob/main/CONTRIBUTING.md).

Contributions can be made by creating pull requests.
The GitHub interface supports creating pull requests using the Edit (✏) button.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed. See [the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

## Authors

- Dick Hardt (Hellō)
- Sam Goto (Google)
