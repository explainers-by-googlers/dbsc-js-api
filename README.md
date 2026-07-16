# Explainer for the JavaScript API for Device Bound Session Credentials

This proposal is an early design sketch by Google to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Google User Protection
- Chrome & Web Ecosystem
- Google Identity Platforms

## Participate

- <https://github.com/explainers-by-googlers/dbsc-js-api/issues>

## Table of Contents

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
  - [SSO handoffs](#sso-handoffs)
  - [Simplified federated binding](#simplified-federated-binding)
  - [Synchronous cookie issuance](#synchronous-cookie-issuance)
- [Potential Solution](#potential-solution)
  - [Querying existing sessions](#querying-existing-sessions)
  - [Pending registrations](#pending-registrations)
    - [Past events buffer](#past-events-buffer)
- [Detailed design discussion](#detailed-design-discussion)
  - [Open question: Web Workers](#open-question-web-workers)
  - [Open question: Proof generation (signed `fetch()`)](#open-question-proof-generation-signed-fetch)
  - [Open question: async / await registration query](#open-question-async--await-registration-query)
  - [Open question: Naming](#open-question-naming)
- [Considered alternatives](#considered-alternatives)
  - [Server-side HTTP polling](#server-side-http-polling)
  - [Integration with Credential Management API](#integration-with-credential-management-api)
  - [Client-initiated refresh (`refresh()`)](#client-initiated-refresh-refresh)
  - [Registration trigger (`start()`)](#registration-trigger-start)
  - [Alternative existing sessions visibility](#alternative-existing-sessions-visibility)
  - [Alternative registration attempts visibility](#alternative-registration-attempts-visibility)
  - [Exposing all session details](#exposing-all-session-details)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
  - [Secure contexts](#secure-contexts)
  - [Scoping](#scoping)
    - [Existing sessions visibility](#existing-sessions-visibility)
    - [Registration attempts visibility](#registration-attempts-visibility)
  - [Permissions policy](#permissions-policy)
  - [Tracking vectors](#tracking-vectors)
  - [Malicious browser extensions](#malicious-browser-extensions)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Device Bound Session Credentials (DBSC) provides cryptographic protection against off-device cookie theft by binding sessions to a user's device. Currently, DBSC relies on asynchronous HTTP headers for session initialization.

This asynchronous nature creates an observability gap for websites, especially Identity Providers (IdPs) that require deterministic guarantees of session establishment before granting access.

This explainer proposes a new JavaScript API that provides a synchronization mechanism for websites to wait for DBSC registrations.

## Goals

- Provide JavaScript capabilities to check the state of DBSC registration attempts and established sessions.
- Enable IdPs to know what and how cookies are bound before redirecting a user during SSO flows.
- Eliminate the vulnerability window where temporary unbound cookies are issued during session setup.

## Non-goals

- Deprecating header-based flows: The HTTP header mechanism will remain fully supported and is not being replaced by this API.
- Insecure contexts: DBSC is inherently designed for secure contexts. Exposing this API in non-HTTPS environments is explicitly out of scope.

## User research

External developers have actively requested these capabilities on the main W3C WebAppSec DBSC repository:

- Synchronous observability (w3c/webappsec-dbsc#117, w3c/webappsec-dbsc#176, w3c/webappsec-dbsc#223): Developers building client-heavy applications note that the asynchronous nature of the header-based API creates race conditions. They explicitly requested capability detection and a way to wait for registrations to complete before firing subsequent authentication requests.
- Proof-of-possession and JS bindings (Issues w3c/webappsec-dbsc#23, w3c/webappsec-dbsc#177): There is strong community interest in client-side request signing (aligning with [DPoP](https://datatracker.ietf.org/doc/html/rfc9449)) and direct JS bindings for key usage to simplify server-side architectures.
- Ecosystem adoption (Issue w3c/webappsec-dbsc#260): Active open-source projects (like `dbsc-toolkit`) highlight real-world prototyping and the need for developer-friendly primitives.

## Use cases

### SSO handoffs

In a federated SSO ecosystem, an IdP completing a login flow may redirect the user back to the Relying Party (RP). This often happens before the IdP's own DBSC session is fully registered on the device. An IdP can use this API to display a loading state, wait for the registration to confirm, and *then* execute the redirect, ensuring the federated trust chain is fully anchored to hardware.

See also: [DBSC for SSO](https://github.com/WICG/dbsc-sso).

### Simplified federated binding

For closely aligned domains, users authenticating on the primary domain can wait for synchronous cookie issuance. The known key can then immediately be used to protect cookies on the secondary domain, avoiding multiple asynchronous registrations and simplifying user journeys.

See also: [federated DBSC sessions](https://w3c.github.io/webappsec-dbsc/#federated-sessions).

### Synchronous cookie issuance

When a user authenticates, the server initiates DBSC registration but may initially issue less secure cookies. Using this API, the webpage awaits confirmation that the DBSC session is registered. Once confirmed, the page requests fully privileged, device-bound authentication cookies. This simplifies authentication and closes the window for infostealer malware.

## Potential Solution

We propose extending the user agent `Navigator` interface and introducing a new `Observer` object to allow state-querying capabilities.

### Querying existing sessions

An established DBSC session is uniquely identified by its `origin` and `session identifier`. Browsers store additional metadata about these sessions. Websites can ask the browser to instantly return a list of all active sessions they are authorized to see. This allows the page to verify if a user already has a DBSC session before deciding what to do next.

Here is an example of how a developer would query those sessions in code:

```js
// Ensure DBSC is supported
if (!('securesession' in navigator)) return;
// List all sessions visible to us
const sessions = navigator.securesession.getExistingSessions();
// Find a session of interest or confirm its absence
for (const session of sessions) { }
```

This synchronous call ignores pending registrations and immediately returns an array of currently existing sessions. Only sessions [visible from the current origin](#existing-sessions-visibility) are returned. For example:

```json
[
  {
    "origin": "https://example.com",
    "sessionId": "foo",
    "creationTime": 1780493529912,
    "sessionScope": { ... },
    "refreshUrl": "/refreshFoo",
    "expiresIn": 600000
  },
  {
    "origin": "https://example.com",
    "sessionId": "xyz",
    "creationTime": 1780493529912,
    "sessionScope": { ... },
    "refreshUrl": "https://id.example.com/refreshXyz",
    "expiresIn": 600000
  }
]
```

All time durations and timestamps are in milliseconds for consistency.

### Pending registrations

Registration attempts are triggered by HTTP headers. They may result from a local `fetch()` call, but also a navigation response, a subresource fetch, etc. This part of the API is a synchronization mechanism to learn about all registration attempts. We suggest using the `Observer` interface, specifically for its buffering capability described [in the next subsection](#past-events-buffer).

A simple use case with an inline fetch would look like this:

```js
const observer = new DbscRegistrationObserver((reports, observer) => {
  reports.forEach((report) => {
    if (report.registrationUrl === 'https://id.example.com/register') {
      if (report.success) {
        // Finish the authentication flow or proceed with SSO redirects
      } else {
        // Handle the registration failure
      }
    }
  });
});
observer.observe();
// Fetch a Secure-Session-Registration header
await fetch('https://id.example.com/start-dbsc');
```

Below are example reports of a typical success and failure:

```json
{
  "success": true,
  "endTime": 1780493529912,
  "registrationUrl": "https://id.example.com/register",
  "origin": "https://example.com",
  "sessionId": "foo"
}
```

```json
{
  "success": false,
  "endTime": 1780493529912,
  "registrationUrl": "https://id.example.com/register",
  "reason": "Persistent server error",
  "code": 500
}
```

Note that failed attempts do not have a `session identifier` or `origin`. Only successful attempts convert into a concrete DBSC session with those attributes. Atempts to [federate](https://w3c.github.io/webappsec-dbsc/#federated-sessions) do have more details, such as `provider url` and `provider session id`.

#### Past events buffer

A navigation response itself may contain DBSC registration headers. Scripts on the resulting page may miss such registration events. It's therefore useful to grant visibility into past events:

```js
const options = { buffered: true };
const observer = new DbscRegistrationObserver((reports, observer) => {
  // Handle reports, including past events.
}, options);
observer.observe();
```

Certain login flows also take place over multiple pages, which the buffer also covers. For background scripts (Web Workers) the buffer gives more limited visibility. Determining exactly which past events a script is allowed to see requires careful consideration. We explore how this works in a section about [past attempts visibility](#registration-attempts-visibility) and evaluate a few [alternative approaches](#alternative-registration-attempts-visibility). Additionally, we have highlighted specific challenges related to background scripts in the [open question regarding Web Workers](#open-question-web-workers).

Regardless of the scope of this buffer, browsers should put a reasonable limit on the number of stored events and their time-to-live. This will prevent privacy and memory issues, especially in long-lived Single Page Applications or background Web Workers

## Detailed design discussion

### Open question: Web Workers

Web Workers can call `getExistingSessions()` but they might also need access to registration attempts. However, most are decoupled from a top-level navigation context. They would therefore not "see" as much as an in-Document script. We could still give background scripts visibility into registrations triggered by their own fetches.

If there are valid Web Workers use cases requiring non-local registration visibility we could change their visibility to a broader per-origin scope, akin to `localStorage` (see [below](#alternative-registration-attempts-visibility)). We would then need to address a different set of concerns, such as Incognito mode leaks, cross-origin redirect complexities, opaque origin iframes, BFCache interactions, or memory management.

### Open question: Proof generation (signed `fetch()`)

We are exploring a point-in-time challenge mechanism, such as `fetch(url, { secureSessionId: 'session_id' })`. The resulting request would then contain a cryptographic proof of possession of the private key of `session_id` (assuming such a session exists, and the request is [in scope of that session](https://w3c.github.io/webappsec-dbsc/#algo-url-in-scope)).

The proof of possession could be a JWT ([RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)) or an HTTP Message Signature ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)).

### Open question: async / await registration query

The [first code example](#querying-existing-sessions) of this proposal would arguably be much more ergonomic if scripts could simply await a Promise. An initial draft proposed a Promise-returning `navigator.securesession.waitForRegistrations()` that would resolve once all pending registrations finish. It would also automatically include buffered events, reporting a status list of all past registrations. This is less in line with modern APIs and seems like a generally worse developer experience.

### Open question: Naming

The current proposal uses a mix of "Secure Session" and "DBSC" in different places. This could be unified, for example by replacing `navigator.securesession` with `navigator.dbsc`, or `SecureSessionObserver` with `DbscRegistrationObserver`.

## Considered alternatives

### Server-side HTTP polling

Today the only practical alternative to this API is for websites to implement HTTP polling. This ad-hoc check lets the server-side confirm the DBSC registration through cookies. This was rejected due to:

- Implementation complexity: Requires bespoke state management on both client and server.
- Performance overhead: Continuous polling consumes unnecessary network traffic, device battery, and server resources.
- Lack of standardization: Prevents a unified approach to session state management across the web.

### Integration with Credential Management API

`navigator.credentials` already handles WebAuthn, FedCM, and password managers. It might seem intuitive to add DBSC to this namespace. However, session authentication is fairly independent from user identity: many sessions don't have an identity associated (intermediate sessions during sign-in flows, carts in shopping, guest checkout flows).

Also, overloading arguments to `navigator.credentials.get` would make that API more complex, and `navigator.credentials.getExistingSessions` would be confusing. We argue that a separate `navigator` namespace fits the DBSC use cases better.

### Client-initiated refresh (`refresh()`)

We considered a `navigator.securesession.refresh()` method that would trigger a cookie refresh. However, the existing spec already encourages proactive background refreshes by the browser. Site-initiated refreshes might be counter-productive, especially if we introduce a [point-in-time challenge mechanism](#open-question-proof-generation-signed-fetch).

### Registration trigger (`start()`)

We considered an active registration API (e.g. `navigator.securesession.start()`) to initiate registrations purely via JS instead of HTTP headers. We're leaning towards abandoning this part of the API:

- Redundancy: The proposed read-only API combined with a `fetch()` call is functionally very similar to a dedicated `start()` method, making its added value minimal.
- Security risks: Exposing the cryptographic challenge directly to the JS environment introduces risks. A malicious browser extension could intercept the challenge, block the registration, and complete it with their own exportable private key (key-swapping).
  - We could mitigate this attack by mandating a challenge-issuing endpoint called by the browser itself. But that approach also has security caveats for IdPs and strengthens the redundancy argument above.

### Alternative existing sessions visibility

Instead of extending the list of origins from which an existing session is visible (see [below](#existing-sessions-visibility)), we could mandate a simpler, stricter Same-Origin Policy (SOP). The only origin allowed to query existing sessions is the session's origin. Subdomains that need DBSC information would then host an iframe to the main origin. The iframe calls `navigator.securesession.getExistingSessions()` (note the permissions policy for cross-origin frames, also below) and posts the status back to the parent window via `window.parent.postMessage()`. But coordinating events promotes bespoke solutions and increases the attack surface.

On the contrary, sites may want to include more domains from which existing sessions are visible. It could be interesting to add a new [session scope](https://w3c.github.io/webappsec-dbsc/#session-scope) field to expand the allowed origins. But we feel this is not necessary and too error prone.

### Alternative registration attempts visibility

Our [scope proposal below](#registration-attempts-visibility) ties registration events to the top-level navigation context. In other words, different pages in the same tab can observe the status of past registrations.

Registration attempts could have a broader origin scope, similar to that of `localStorage`. While it doesn't raise obvious privacy concerns this option introduces extra complexity. We discuss this more in [this open question section](#open-question-web-workers).

Alternatively, attempts may also have a more restricted, Document-like scope. Statuses would only be retained in the specific environment (like the current `WindowOrWorkerGlobalScope`) that triggered the registration request. This sidesteps most privacy and race condition concerns but also prevents multi-page use cases.

Another option is to use the synchronous `getExistingSessions()` to snapshot the state before registering an event listener or observer. However, [this method as proposed above](#querying-existing-sessions) only returns information about past successes. A site may want to react to failures and cannot afford to miss registration failure events. And adding past failures to the output of `getExistingSessions()` is similar to the proposed Observer buffer capability while making the API harder to understand.

### Exposing all session details

We think it's unnecessary and unwise to expose certain details about a session. For example, reporting a session's public key might prompt a site to rely on this capability for challenge verification, which would expose their sessions to a range of attacks.

## Security and Privacy Considerations

Since this API exposes cryptographic session states, we suggest enforcing strict boundaries.

### Secure contexts

The JS features are only available over HTTPS, like the rest of DBSC.

### Scoping

#### Existing sessions visibility

Existing sessions are visible to scripts executing on the session's [origin](https://w3c.github.io/webappsec-dbsc/#framework-scope) and the origin of its [refresh URL](https://w3c.github.io/webappsec-dbsc/#framework-session) if different.

Rationale for including the refresh origin by default: Large sites often have a hardened subdomain dedicated to authentication, with stronger XSS protections. This is a natural origin for hosting refresh endpoints. Allowing direct access to DBSC data from such subdomains encourages better security architectures. The browser also verified the relationship between these two origins via a .well-known check at registration.

#### Registration attempts visibility

Registration attempts do not have a session configuration. The resulting session's `origin` is only known if and when the attempt succeeds. An attempt is instead partitioned by its registration endpoint's origin and the top-level browsing context of the response that triggered it. This is similar to the per-tab visibility of `sessionStorage` (see [Session-Only Storage](https://html.spec.whatwg.org/multipage/webstorage.html)).

If no top-level browsing context exists, attempts are instead stored alongside the `WindowOrWorkerGlobalScope`. Such attempts have more limited visibility. We discuss this in the [open question regarding Web Workers](#open-question-web-workers).

### Permissions policy

Cross-origin frames cannot query registration states unless explicitly authorized via a permission policy (e.g. `allow="dbsc-session-credentials-get"`).

### Tracking vectors

A session's details must not be readable when the session or its cookies are not available. Sessions and keys [must be deleted](https://w3c.github.io/webappsec-dbsc/#privacy-considerations) alongside cleared site data, making them unavailable for lookup. This restriction includes Incognito modes or 3rd party contexts for user agents that block `SameSite=None` cookies.

This proposal helps with [SSO handoffs](#sso-handoffs) but does not bypass third-party cookie restrictions or cross-site tracking protections. The underlying identity federation still relies on standard platform capabilities (e.g. FedCM, OAuth redirects).

This API should not introduce additional surface for user tracking. Registration error statuses stay high level and simple to avoid new device fingerprinting avenues. Browsers should also add latency fuzzing or clamping to asynchronous methods involving cryptographic operations, to avoid timing side-channels.

### Malicious browser extensions

A malicious browser extension may tamper with JavaScript objects and fake all API return values. This could lead to a Denial of Service (DoS). A number of bespoke measures, such as using `HttpOnly` cookies, can help build confidence that the server is interacting with a browser-originated request. Sites that deem this unacceptable can fall back to the headers-based protocol, though we should note this is also susceptibile to other DoS attacks.

## Stakeholder Feedback / Opposition

Implementors and other stakeholders did not publicly state positions on this work yet.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Arnar Birgisson [@arnar](https://github.com/arnar)
- Daniel Rubery [@drubery](https://github.com/drubery)
- Lucas Santos [@lucasrsant](https://github.com/lucasrsant)
