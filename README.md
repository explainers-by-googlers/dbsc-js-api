# Explainer for the JavaScript API for Device Bound Session Credentials

This proposal is an early design sketch by Google to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Google

## Participate

- <https://github.com/explainers-by-googlers/dbsc-js-api/issues>

## Introduction

Device Bound Session Credentials (DBSC) provides cryptographic protection against off-device cookie theft by binding sessions to a user's device. Currently, DBSC relies on asynchronous HTTP headers for session initialization.

This asynchronous nature creates an observability gap for websites, especially Identity Providers that require deterministic guarantees of session establishment before granting access.

This explainer proposes a new JavaScript API that provides a synchronization mechanism for websites to wait for DBSC registrations.

## Non-goals

- Deprecating header-based flows: The HTTP header mechanism will remain fully supported and is not being replaced by this API.
- Insecure contexts: DBSC is inherently designed for secure contexts. Exposing this API in non-[secure contexts](https://www.w3.org/TR/secure-contexts/) is explicitly out of scope.

## Use cases

### SSO handoffs

In an SSO ecosystem, the Identity Provider (IdP) completing a login flow typically redirects the user back to a Relying Party (RP). This often happens before the IdP's own DBSC session gets registered on the device. An IdP can use this API to display a loading state, wait for the registration to complete, and *then* execute the redirect.

This allows the RP to rely on the existence of a trusted IdP key to establish a [federated DBSC session](https://w3c.github.io/webappsec-dbsc/#federated-sessions) or leverage the upcoming [DBSC for SSO](https://github.com/WICG/dbsc-sso).

### Synchronous cookie issuance

When a user authenticates, the server initiates DBSC registration but may initially issue less secure cookies. Using this API, the webpage awaits confirmation that the DBSC session is registered. Once confirmed, the page requests fully privileged, device-bound authentication cookies. This simplifies authentication and closes the gap where cookies can be exfiltrated by local malware.

### Developer interest

External developers have actively requested synchronous observability on the main W3C WebAppSec DBSC repository (w3c/webappsec-dbsc#117, w3c/webappsec-dbsc#176, w3c/webappsec-dbsc#223): Developers building client-heavy applications note that the asynchronous nature of the header-based API creates race conditions. They explicitly requested capability detection and a way to wait for registrations to complete before firing subsequent authentication requests.

## Potential Solution

The API provides a synchronization mechanism to learn about registration attempts and existing sessions.

### Past events visibility

A navigation response may contain DBSC registration headers. Scripts on the resulting page might miss such events if our API relies on a strict push-based model, like `EventTarget`. We must therefore grant visibility into past events, and propose to use a new Observer class for this.

This functionality is inspired by the [report buffer of `ReportingObserver`](https://www.w3.org/TR/reporting-1/#observers). We borrow the term "buffer" for consistency.

We expect virtually all use cases to want this buffering capability. To avoid mistakes, the constructor's `options` has an `unbuffered` boolean. This makes the buffered behavior the default.

### Querying sessions

An established DBSC session is identified by a unique string (`session identifier`) on a [registrable domain](https://url.spec.whatwg.org/#host-registrable-domain). Browsers store additional metadata about these sessions. Websites must be able to list active sessions. This allows the page to learn more details about a session.

Instead of a separate synchronous method, and given the observer's buffer capability, we can consolidate this feature into the observer itself. Unless created with `{ unbuffered: true }`, the observer will report existing sessions.

Here is an example of how to query existing sessions:

```js
if (typeof DeviceBoundSessionsObserver === "undefined") return;

const callback = (reports, observer) => {
  reports.forEach((report) => {
    if (
      report.type === 'session' &&
      report.refreshUrl.endsWith('/refreshFoo')
    ) {
      // Inspect report.sessionId etc.
    }
  });
}

new DeviceBoundSessionsObserver(callback).observe();
```

This code will match a session whose refresh URL is `/refreshFoo`. If sessions exist, the code runs immediately. It runs again when a registration successfully creates a new session or overwrites one.

This is an example `session` report:

```json
{
  "type": "session",
  "sessionOrigin": "https://example.com",
  "sessionId": "foo",
  "refreshUrl": "/refreshFoo",
  "creationTime": 1780493529912,
  "sessionScope": [
    { "type": "exclude", "domain": "*.example.com", "path": "/static" }
  ],
  "credentials": [{
    "type": "cookie",
    "name": "auth_cookie",
    "attributes": "Domain=example.com; Path=/; Secure; SameSite=None"
  }],
  "refreshDueIn": 600000
}
```

All time durations and timestamps are in milliseconds for consistency.

### Registration attempts

DBSC registration attempts are triggered by HTTP headers. They may result from a local `fetch()` call, but also a navigation response, a subresource fetch, etc.

A simple use case with an inline fetch would look like this:

```js
const callback = (reports, observer) => {
  reports.forEach((report) => {
    if (
      report.type === 'registration' &&
      report.registrationUrl === 'https://id.example.com/register'
    ) {
      observer.disconnect();
      if (report.success) {
        // Finish the authentication flow or proceed with SSO redirects
      } else {
        // Handle the registration failure
      }
    }
  });
};
// Ignore past attempts
const options = {unbuffered: true};

new DeviceBoundSessionsObserver(callback, options).observe();
// Fetch a Secure-Session-Registration header
await fetch('https://id.example.com/start-dbsc');
```

Below are examples of typical successful and failed `registration` reports. Success reports include the resulting `session` report, which is not reported separately to that observer instance. Failure reports contain more details about the reason.

```json
{
  "type": "registration",
  "registrationUrl": "https://id.example.com/register",
  "endTime": 1780493529912,
  "success": true,
  "session": { }
}
```

```json
{
  "type": "registration",
  "registrationUrl": "https://id.example.com/register",
  "endTime": 1780493529912,
  "success": false,
  "reason": "Persistent server error",
  "code": 500
}
```

Certain DBSC registration headers cause an attempt to [federate](https://w3c.github.io/webappsec-dbsc/#federated-sessions). The corresponding registration report, successful or not, includes the `provider URL` and `provider session id` from the header.

### Timing out

Due to the complex and networked nature of DBSC, placing an upper bound on the observing time will be extremely commonplace. Instead of forcing developers to use a separate mechanism, we propose to make `AbortSignal` a first-class citizen of this API. This would be an additional option passed to the constructor:

```js
const dbscObserver = new DeviceBoundSessionsObserver(
  (reports, observer) => { /* ... */ },
  { signal: AbortSignal.timeout(3000) },
)
```

This allows developers to use an `AbortController` and link not only timeouts but also user cancellation, navigation, etc.

## Detailed design discussion

### Open question: Session termination report

The observer could report session termination events and their reason. This may be useful, although some of this information is already available to websites through DBSC [debug headers](https://w3c.github.io/webappsec-dbsc/#header-secure-session-skipped).

For example:

```json
{
  "type": "termination",
  "sessionOrigin": "https://example.com",
  "sessionId": "foo",
  "refreshUrl": "/refreshFoo",
  "creationTime": 1780493529912,
  "terminationTime": 1780493529912,
  "reason": "unreachable"
}
```

Such reports would have the same visibility as the now-terminated session.

### Web Workers

We could not find convincing use cases involving both DBSC and Web Workers. We therefore propose not allowing `DeviceBoundSessionsObserver` in Web Workers at all.

## Considered alternatives

### Server-side HTTP polling

Today the only practical alternative to this API is for websites to implement HTTP polling. This ad-hoc check lets the server-side confirm the DBSC registration through cookies. This was rejected due to:

- Implementation complexity: Requires bespoke state management on both client and server.
- Performance overhead: Continuous polling consumes unnecessary network traffic, device battery, and server resources.
- Lack of standardization: Prevents a unified approach to session state management across the web.

### Integration with Credential Management API

`navigator.credentials` already handles WebAuthn, FedCM, and password managers. It might seem intuitive to add DBSC to this namespace. However, session authentication is fairly independent from user identity: many sessions don't have an identity associated (e.g. intermediate sessions during sign-in flows, carts in shopping, or guest checkout flows).

Also, overloading arguments to `navigator.credentials.get` would make that API more complex, and `navigator.credentials.newDbscObserver` would be confusing. We argue that a global-scope constructor fits the DBSC use cases better.

### Synchronous getter (`getExistingSessions()`)

Using a synchronous getter and event listener is a common JS pattern. For example:

```js
if (document.readyState === 'complete') {
  doSomething();
} else {
  window.addEventListener('load', () => doSomething());
}
```

Instead of using the observer for all operations we could extend the user agent `Navigator` interface. For DBSC, the pattern above would become:

```js
// Synchronously list all visible sessions
const sessions = navigator.deviceBoundSessions.getExistingSessions();
if (hasSessionOfInterest(sessions)) {
  // Proceed
} else {
  new DeviceBoundSessionsObserver(/* ... */).observe();
}
```

We feel this dual interface is not needed for our primary use case of synchronizing actions with registration events. In addition, this synchronous getter may encourage developers to rely on JavaScript to detect DBSC sessions in other scenarios. In such cases a server-side decision based on bound cookies is often better, as it's much more robust and secure. See also [this section about malicious browser extensions](#malicious-browser-extensions).

Developers who truly need to only list existing sessions can still use the observer, although it's a more cumbersome setup.

### async / await registration query

The [first code example](#registration-attempts) of this proposal would arguably be more ergonomic if scripts could simply await a Promise. An initial draft proposed a Promise-returning `navigator.deviceBoundSessions.waitForRegistrations()` that would resolve once all pending registrations finish. It would also automatically include buffered events, reporting a status list of all past registrations. This is less in line with modern APIs and seems like a generally worse developer experience.

### Registration trigger (`start()`)

We considered an active registration API (e.g. `navigator.deviceBoundSessions.start()`) to initiate registrations purely via JS instead of HTTP headers. We think this part of the API is not needed:

- Redundancy: The proposed read-only API combined with a `fetch()` call is functionally very similar to a dedicated `start()` method, making its added value minimal.
- Security risks: Exposing the cryptographic challenge directly to the JS environment introduces risks. A malicious browser extension could intercept the challenge, block the registration, and complete it with its own exportable private key.
  - We could mitigate this attack by mandating a challenge-issuing endpoint called by the browser itself (see w3c/webappsec-dbsc#176). But that approach does not make the serving setup significantly simpler, comes with security caveats, and strengthens the redundancy argument above.

### Alternative registration attempts visibility

Our [scope proposal below](#buffer-scope) ties registration events to `localStorage`. Attempts could instead have a more restricted, Document-like scope. Statuses would only be retained in the Environment Settings Object that triggered the registration request. This sidesteps most race condition and privacy concerns but also prevents multi-page use cases requiring details about past registration attempts.

### Exposing all session details

We think it's unnecessary and unwise to expose certain details about a session. For example, reporting a session's public key might prompt a site to rely on this capability for challenge verification, which would expose their sessions to a range of attacks.

## Security and Privacy Considerations

Since this API exposes cryptographic session states, we suggest enforcing strict boundaries.

### Secure contexts

The observer is only available over [Secure Contexts](https://www.w3.org/TR/secure-contexts/), like the rest of DBSC.

### `HttpOnly` cookies

A `session` report contains a list of bound [credentials](https://w3c.github.io/webappsec-dbsc/#format-session-credentials). Some of these may be `HttpOnly` cookies. While the API does not return the value of cookies, it would reveal their names.

To keep `HttpOnly` cookies truly off-JavaScript, the API removes or redacts such cookies from a `session` entry. See also whatwg/cookiestore#37, loosely related to that topic.

### Tracking vectors

This API does not introduce additional surfaces for user tracking. This proposal helps with SSO handoffs but does not bypass third-party cookie restrictions or cross-site tracking protections. The underlying identity federation still relies on standard platform capabilities (e.g. FedCM, OAuth redirects).

DBSC itself also [respects cookie policies](https://w3c.github.io/webappsec-dbsc/#privacy-cookies). For example, a registration cannot succeed when it is not allowed to set any of the cookies referenced by the new DBSC session. This API similarly blocks access to DBSC information in contexts where the DBSC cookies do not apply. An existing session's details must likewise not be readable by scripts when its cookies would not be included in a request.

Sessions and keys [must be deleted](https://w3c.github.io/webappsec-dbsc/#privacy-considerations) alongside cleared site data, making them unavailable for lookup past their lifetime.

Registration error statuses stay high-level and simple, to avoid new device fingerprinting avenues.

### Buffer scope

As stated in the previous section, the API does not introduce additional tracking avenues. The reports buffer is no exception, and does not add cross-page or cross-site communication. In other words, the buffer doesn't reveal information that scripts cannot already share using other storage APIs, such as `localStorage` or `IndexedDB`.

Browsers should also put a reasonable limit on the number of attempts stored in each buffer and their time-to-live. This prevents privacy and memory issues, especially in long-lived Single Page Applications or background tabs.

[This document](buffer-visibility.md) goes into more technical details about how we might achieve such a guarantee.

### Malicious browser extensions

A malicious browser extension may tamper with JavaScript objects and fake all API return values. This could lead to a Denial of Service (DoS). A number of bespoke measures, such as using `HttpOnly` cookies, can help build confidence that the server is interacting with a browser-originated request. Sites that deem this unacceptable can fall back to the headers-based protocol, though we should note this is also susceptible to other DoS attacks.

## Stakeholder Feedback / Opposition

Implementors and other stakeholders did not publicly state positions on this work yet.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Amr Aboelkher
- Arnar Birgisson [@arnar](https://github.com/arnar)
- Daniel Margolis
- Daniel Rubery [@drubery](https://github.com/drubery)
- Lucas Santos [@lucasrsant](https://github.com/lucasrsant)
- Nina Satragno [@nsatragno](https://github.com/nsatragno)
