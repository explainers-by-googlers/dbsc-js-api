# Report buffer visibility

This document attempts to give a more formal scope to events reported by `DeviceBoundSessionsObserver`. The explainer in this repository focuses on *what* the visibility should be. This document explores *how* we can achieve it.

The proposal has two different report types: `registration` and `session`. Here we explain what report types are visible to what scripts, using abstractions from the [Storage spec](https://storage.spec.whatwg.org/), like Storage Shed, Shelf, and Key.

Borrowing Storage concepts hopefully makes DBSC events visibility clear, while inheriting several privacy-preserving properties like opaque origins handling or partitioned states. We do not, however, suggest implementing a new formal Storage Endpoint for DBSC metadata, mainly because of the [ESO fallback](#registration-attempts-visibility) feature we describe further down.

## Existing sessions visibility

An established DBSC session has a registrable domain (called its [`origin`](https://w3c.github.io/webappsec-dbsc/#framework-scope)) and a [`refresh URL`](https://w3c.github.io/webappsec-dbsc/#framework-session). These must be same-site but can be on different origins [under certain conditions](https://w3c.github.io/webappsec-dbsc/#ref-for-json-session-scope-include_site). A session doesn't automatically cover its whole `origin`: cookie policies [also apply](https://w3c.github.io/webappsec-dbsc/#privacy-cookies) to DBSC.

DBSC sessions are inherently unpartitioned, keyed only by their registrable domain. They also explicitly [do not support CHIPS](https://w3c.github.io/webappsec-dbsc/#algo-create-session).

In Storage terms, DBSC session metadata live in the User Agent's (`local`) Shed, in the Shelf addressed by the first-party key `[session's registrable domain, session's origin]`. We propose to mirror this to another Shelf, `[session's registrable domain, session's refresh URL's origin]`.

> [!NOTE]
> **Rationale for including the refresh origin by default:**
> Large sites often have a hardened subdomain dedicated to authentication, with stronger XSS protections. This is a natural origin for hosting refresh endpoints. Allowing direct access to DBSC data from such a subdomain encourages better security architectures.

## Cross-site restrictions

For clients with partitioned storage and cross-site restrictions, an iframe `I` on the top-level site `S` would have the Storage Key `[S,I]`. It needs to use the Storage Access API to elevate its access to `[I,I]` and gain access to the DBSC session data stored there. This also aligns nicely with how SAA grants access to third-party cookies, which DBSC has deep ties to.

If needed, we could add a new entry to the `types` argument of `requestStorageAccess` to grant access only to DBSC data, as opposed to the whole `localStorage`.

## Registration attempts visibility

A registration's URL is derived from a response's URL. The DBSC spec [requires](https://w3c.github.io/webappsec-dbsc/#algo-session-request) this to be same-site with the originating request's origin.

In Storage terms, registration attempts live in the User Agent's (`local`) Shed, in the Shelf addressed by the Key `[originating request's site, registration endpoint's origin]`.

Successful registration attempts also contain a `session` report, possibly extending that report's visibility if the registering origin is different from the session's origin and its refresh endpoint's. We think this is not a privacy or security issue.
