# Explainer for the JavaScript API for Device Bound Session Credentials

This proposal is an early design sketch by Google to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Google

## Participate

- <https://github.com/explainers-by-googlers/dbsc-js-api/issues>

## Introduction

[Device Bound Session Credentials](https://w3c.github.io/webappsec-dbsc/) (DBSC) protects sessions against off-device cookie theft by binding them to a cryptographic key on the user's device. Today, a server starts session setup through an HTTP response header, and the browser generates keys and registers the session in the background.

Because registration runs in the background, scripts on the page cannot tell when a session finishes registering, whether setup failed, or which bound sessions already exist. Identity Providers and security-sensitive applications often need that information before redirecting a user or sending authenticated requests.

This explainer proposes a JavaScript API that lets websites wait for pending DBSC registrations and inspect active sessions.

## Non-goals

- Deprecating header-based flows: This API complements the HTTP header mechanism and does not replace it.
- Insecure contexts: DBSC only works in [secure contexts](https://www.w3.org/TR/secure-contexts/), so exposing this API elsewhere is out of scope.
- Server-side telemetry: Passive error logging for background refreshes and session terminations belongs in the main DBSC specification (for example using the Reporting API).

## Use cases

### SSO handoffs

In an SSO flow, an Identity Provider (IdP) completing sign-in redirects the user back to a Relying Party (RP). Without coordination, that redirect often fires before the IdP's own DBSC session finishes registering on the device.

With this proposal, an IdP can show a loading state, wait for registration to finish (or handle an error), and then redirect. On later visits, the IdP can also check whether a bound session already exists. That lets the RP rely on an established IdP key for a [federated DBSC session](https://w3c.github.io/webappsec-dbsc/#federated-sessions) or [DBSC for SSO](https://github.com/WICG/dbsc-sso).

### Coordinating bound cookie issuance

During sign-in, the server starts DBSC registration, and the registration endpoint sets the bound authentication cookie once key setup completes. Because authentication cookies are typically `HttpOnly`, page scripts cannot check `document.cookie` to see when they arrive.

This proposal lets the page wait until the bound session is ready before sending sensitive requests, or fall back if registration fails. Sites no longer need temporary unbound cookies during setup.

### Developer interest

External developers have asked for client-side observability on the main DBSC repository (w3c/webappsec-dbsc#117, w3c/webappsec-dbsc#176, w3c/webappsec-dbsc#223). Teams building client-heavy applications report race conditions when HTTP headers start session setup in the background. They asked for capability detection and a way to wait for registrations before sending authenticated requests.

## Potential Solution

We propose adding `navigator.deviceBoundSessions` on `Window` with two Promise-based methods:

- `waitForRegistrations(options)` waits for pending registrations on the current `Document` and resolves with the registration outcomes recorded on that `Document`.
- `getSessions()` resolves with the active DBSC sessions visible to the calling script.

Websites can check `navigator.deviceBoundSessions` to detect browser support:

```js
if (!navigator.deviceBoundSessions) return;
```

### Waiting for registrations

Servers start DBSC registrations through the [`Secure-Session-Registration`](https://w3c.github.io/webappsec-dbsc/#header-secure-session-registration) HTTP response header. That header can arrive on a navigation response (including redirects leading to the page) or on a subresource request (such as `fetch()`).

`waitForRegistrations()` waits until in-flight registrations for the `Document` finish. It then resolves with an array of registration outcomes recorded on that `Document`, sorted by their `endTime`. When no registrations are in flight, it resolves right away with any buffered outcomes on the `Document`, or `[]` if none happened.

With an inline `fetch()`, scripts await the response first so the registration has already started when `waitForRegistrations()` runs:

```js
const response = await fetch('/start-dbsc');
if (!response.ok) return;

const attempts = await navigator.deviceBoundSessions.waitForRegistrations();
if (attempts.at(-1)?.success) {
  // Proceed with authenticated requests or SSO redirect
} else {
  // Handle registration failure or missing registration header
}
```

The browser records the registration as in-flight when processing response headers, before the `fetch()` promise resolves. This ensures that `waitForRegistrations()` either sees a finished registration or waits for it. For simplicity, the browser ignores headers processed after a `waitForRegistrations()` call, even if they occur before its promise resolution.

On a navigation landing page (for example `POST /login` redirecting to `/handoff`), scripts call `waitForRegistrations()` directly on load. The browser buffers outcomes from the navigation request and its redirects on the resulting `Document`, so scripts do not miss fast registrations that finished before JavaScript ran.

In a Single Page Application, earlier registrations may already sit in the `Document` buffer. Scripts can record `Date.now()` before `fetch()` and filter for outcomes with a newer `endTime`.

Below are examples of successful and failed registration outcomes:

```json
{
  "registrationUrl": "https://id.example.com/register",
  "endTime": 1780493529912,
  "success": true,
  "session": {
    "sessionOrigin": "https://id.example.com",
    "sessionId": "foo",
    "refreshUrl": "https://id.example.com/refreshFoo",
    "creationTime": 1780493529912
  }
}
```

```json
{
  "registrationUrl": "https://id.example.com/register",
  "endTime": 1780493529912,
  "success": false,
  "reason": "Persistent server error",
  "code": 500
}
```

DBSC also supports [federated sessions](https://w3c.github.io/webappsec-dbsc/#federated-sessions) that share an existing provider key. For those attempts, the outcome object (successful or not) includes `providerUrl` and `providerSessionId` from the registration header.

All timestamps use milliseconds since the Unix epoch, and all URLs are absolute.

### Timing out

Registration involves key operations and a network request, so callers will often bound how long they wait. `waitForRegistrations()` accepts an `AbortSignal` and rejects if the signal aborts:

```js
try {
  const attempts = await navigator.deviceBoundSessions.waitForRegistrations({
    signal: AbortSignal.timeout(3000),
  });
  if (attempts.at(-1)?.success === false) {
    // Handle fast registration failure
  }
} catch (err) {
  // Timed out or aborted via AbortController
}
```

First-class `AbortSignal` support lets callers link timeouts with user cancellation or custom `AbortController` logic. This signal only cancels the wait, not any underlying registrations.

### Querying active sessions

Each established DBSC session has a unique "identifier" on its [registrable domain](https://url.spec.whatwg.org/#host-registrable-domain) (the session's "origin"). On later pages, websites can call `getSessions()` to list active sessions visible to the script (`[]` if none exist):

```js
const sessions = await navigator.deviceBoundSessions.getSessions();
const match = sessions.find(
  (s) => s.sessionId === 'target-session'
);
if (match) {
  // Inspect match.creationTime, match.refreshUrl, etc.
}
```

Each session entry has the following shape:

```json
{
  "sessionOrigin": "https://example.com",
  "sessionId": "foo",
  "refreshUrl": "https://example.com/refreshFoo",
  "creationTime": 1780493529912
}
```

## Detailed design discussion

### Cross-page registration visibility

The browser retains registration outcomes only on the `Document` where they happened (plus redirects leading to that `Document`). In practice, a page that triggers registration waits for the result before navigating away, and later pages can check established sessions through `getSessions()`. If a multi-page flow needs to pass failure details across navigations, scripts can already coordinate through `sessionStorage`. Keeping the built-in buffer per-`Document` avoids persistent storage management, and we could add an opt-in cross-page option later if needed.

### Filtering options

`waitForRegistrations()` waits on all in-flight registrations for the `Document`, since pages rarely run more than one DBSC registration at once. We can add optional filters (such as `registrationUrl` or a `since` timestamp) later without breaking existing code.

### Web Workers

We have not found clear use cases combining DBSC with Web Workers, so we propose exposing `navigator.deviceBoundSessions` on `Window` only.

## Considered alternatives

### Server-side HTTP polling

Today, websites without a client API must poll an HTTP endpoint so the server can check whether bound cookies have arrived. We rejected this approach for three reasons:

- Implementation complexity: Requires custom state tracking on both client and server.
- Performance overhead: Repeated polling wastes network requests, battery, and server capacity.
- Lack of standardization: Forces each site to build its own coordination protocol.

### Observer-based API

We considered a dedicated observer class (modeled on `PerformanceObserver` or `ReportingObserver`) that delivers session and registration records through a callback. We chose Promise-based methods instead for two reasons:

- DBSC registrations happen rarely, typically once during sign-in. Callback observers suit continuous event streams, whereas Promises fit a low-volume, one-shot wait.
- Waiting once during sign-in or listing active sessions required extra boilerplate to disconnect the observer and wire up timeouts.

### Reporting API (`ReportingObserver`)

We also considered surfacing registration events through `ReportingObserver`. However, the JavaScript side of the Reporting API exists for analytics and monitoring scripts that cannot set HTTP reporting headers, not for driving live sign-in flows. `ReportingObserver` also lacks `AbortSignal` support, cannot list established sessions on later pages, and silently ignores unknown report types on browsers without DBSC support.

### Integration with Credential Management API

`navigator.credentials` handles WebAuthn, FedCM, and password managers. Session binding is separate from user identity, though: many sessions have no user identity attached (such as pre-login flows, shopping carts, or guest checkouts). Overloading `navigator.credentials` would complicate both surfaces, so a separate `navigator.deviceBoundSessions` object fits better.

### Synchronous getter (`getExistingSessions()`)

We considered a synchronous `navigator.deviceBoundSessions.getExistingSessions()` method. Browsers typically store DBSC state in a separate network process or on disk, so reading it synchronously from the main thread would require blocking cross-process calls or replicating state into every renderer. An asynchronous `getSessions()` method avoids blocking the main thread.

Sites checking whether a user is authenticated should still verify bound cookies on the server rather than relying on client-side checks. See [Malicious browser extensions](#malicious-browser-extensions).

### Registration trigger (`start()`)

We considered a method such as `navigator.deviceBoundSessions.start()` to start registrations directly from JavaScript without HTTP headers, and decided against it:

- Redundancy: Combining `fetch()` with `waitForRegistrations()` already covers script-initiated flows.
- Security risks: Passing the cryptographic challenge through JavaScript lets a malicious extension intercept the challenge and register its own exportable key instead. Requiring a separate browser-fetched challenge endpoint (see w3c/webappsec-dbsc#176) would undo most of the simplicity benefit.

### Exposing all session details

Exposing fields like a session's public key could tempt sites to verify challenges against client-reported keys, which would open sessions to key-substitution attacks. We limit session objects to non-cryptographic metadata.

## Security and Privacy Considerations

### Secure contexts

`navigator.deviceBoundSessions` is only available in [Secure Contexts](https://www.w3.org/TR/secure-contexts/), like the rest of DBSC.

### Visibility and scoping

The API enforces two boundaries so scripts only see session state that matches cookie access:

#### Active sessions

DBSC defines how to [identify if a URL is in scope](https://w3c.github.io/webappsec-dbsc/#algo-url-in-scope). `getSessions()` (and the `session` field on a registration outcome) includes a session only when the script's URL is in scope or matches the session's refresh URL.

Cross-site frames with partitioned storage must call the Storage Access API before reading unpartitioned DBSC session data. Scope exclusion rules (such as `{ "type": "exclude", "domain": "untrusted.example.com", "path": "/" }`) also hide sessions from excluded subdomains and paths.

#### Registration attempts

The browser keeps registration outcomes in memory on the `Document` that triggered them (including redirects leading to that `Document`), and only when the `Document` is `same-origin` with `registrationUrl`. This way, cross-site pages and subdomains cannot observe or time another origin's registrations.

When the browser blocks third-party cookies, unprivileged third-party iframes cannot establish an unpartitioned DBSC session in the first place, but they are allowed to see their own failed attempt.

On success, the `session` field appears only when the new session passes the active-session check above. This is just an ergonomic consideration, as subsequent `getSessions()` calls would not return out-of-scope sessions.

> [!NOTE]
> Performing a scope check before populating a successful registration attempt's `session` complicates the following use case: When `login.example.com` registers several sessions for `app.example.com`, the registering origin cannot differentiate the resulting sessions.

Browsers should cap the size of this per-`Document` list and discard it on unload.

### Tracking vectors

This API adds no cross-site tracking surfaces and does not bypass third-party cookie restrictions. SSO flows still rely on standard browser mechanisms such as OAuth redirects or FedCM.

DBSC [respects cookie policies](https://w3c.github.io/webappsec-dbsc/#privacy-cookies), and browsers [must delete sessions and keys](https://w3c.github.io/webappsec-dbsc/#privacy-considerations) when users clear site data. Error reasons stay coarse-grained, and browsers monotonically coarsen `creationTime` and `endTime` to prevent hardware timing side-channels across security chips.

### Malicious browser extensions

A malicious browser extension can tamper with JavaScript objects and fake API return values, which can cause a client-side Denial of Service (DoS). Protections such as `HttpOnly` bound cookies still help the server verify browser-originated session proofs. Sites that cannot accept JavaScript tampering can stick to the header-only DBSC protocol, although extensions can disrupt network flows in other ways too.

## Stakeholder Feedback / Opposition

Implementors and other stakeholders did not publicly state positions on this work yet.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Amr Aboelkher
- Arnar Birgisson [@arnar](https://github.com/arnar)
- Daniel Margolis
- Daniel Rubery [@drubery](https://github.com/drubery)
- Lucas Santos [@lucasrsant](https://github.com/lucasrsant)
- Martin Thomson [@martinthomson](https://github.com/martinthomson)
- Nina Satragno [@nsatragno](https://github.com/nsatragno)
