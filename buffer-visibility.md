# Report buffer visibility

This document attempts to give a more formal scope to events reported by `DeviceBoundSessionsObserver`.

The proposal has two different report types: `registration` and `session`. Here we explain what report types are visible to what scripts.

## Existing sessions visibility

An established DBSC session has a given [scope](https://w3c.github.io/webappsec-dbsc/#framework-scope). The spec also has an algorithm to [identify if a URL is in scope of a session](https://w3c.github.io/webappsec-dbsc/#algo-url-in-scope).

We propose running that algorithm on a script's URL to decide whether that script can see a given session. Note that the algorithm explicitly excludes the refresh URL. This is desirable for DBSC, but not for our visibility question. Our algorithm would instead automatically include the refresh URL.

Since exclusion rules apply, sites can hide their sessions from entire subdomains. For example `{ "type": "exclude", "domain": "untrusted.example.com", "path": "/" }`. Path-based exclusion offers less privacy or security benefits, but they still make sense to apply for ergonomic reasons.

## Cross-site restrictions

DBSC sessions data is inherently unpartitioned, keyed only by the session's registrable domain. DBSC also explicitly [does not support CHIPS](https://w3c.github.io/webappsec-dbsc/#algo-create-session).

For clients with partitioned storage and cross-site restrictions, an iframe `I` on the top-level site `S` would be partitioned by `[S,I]`. A script on `I` needs to use the Storage Access API to elevate its access to `[I,I]` and gain access to the unpartitioned DBSC session data.

If needed, we could add a new entry to the `types` argument of `requestStorageAccess` to grant access only to DBSC data.

## Registration attempts visibility

A registration's URL is derived from a response's URL. The DBSC spec [requires](https://w3c.github.io/webappsec-dbsc/#algo-session-request) this to be same-site with the originating request's origin.

Registration attempts can therefore be partitioned by `[registration endpoint's site, registration endpoint's origin]`. In other words, a registration event is 1P data to the registration URL's origin. This information would be available to scripts on that origin as if stored in `localStorage`.

Successful registration attempts also contain a `session` report. This may extend that new session's visibility, but not past the eTLD+1. We think this is not a privacy or security issue.
