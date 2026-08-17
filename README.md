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
  - [Registration attempts](#registration-attempts)
    - [Past attempts buffer](#past-attempts-buffer)
  - [Querying sessions](#querying-sessions)
  - [Timing out](#timing-out)
- [Detailed design discussion](#detailed-design-discussion)
  - [Open question: Web Workers](#open-question-web-workers)
  - [Open question: Proof generation (signed `fetch()`)](#open-question-proof-generation-signed-fetch)
  - [Open question: Session termination report](#open-question-session-termination-report)
- [Considered alternatives](#considered-alternatives)
  - [Server-side HTTP polling](#server-side-http-polling)
  - [Integration with Credential Management API](#integration-with-credential-management-api)
  - [Synchronous getter (`getExistingSessions()`)](#synchronous-getter-getexistingsessions)
  - [async / await registration query](#async--await-registration-query)
  - [Client-initiated refresh (`refresh()`)](#client-initiated-refresh-refresh)
  - [Registration trigger (`start()`)](#registration-trigger-start)
  - [Alternative existing sessions visibility](#alternative-existing-sessions-visibility)
  - [Alternative registration attempts visibility](#alternative-registration-attempts-visibility)
  - [Exposing all session details](#exposing-all-session-details)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
  - [Secure contexts](#secure-contexts)
  - [Scoping](#scoping)
    - [Existing sessions visibility](#existing-sessions-visibility)
    - [Cross-site restrictions](#cross-site-restrictions)
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
- Suggest and discuss future API surface for a compatible and coherent initial specification.

## Non-goals

- Deprecating header-based flows: The HTTP header mechanism will remain fully supported and is not being replaced by this API.
- Insecure contexts: DBSC is inherently designed for secure contexts. Exposing this API in non-[secure contexts](https://www.w3.org/TR/secure-contexts/) is explicitly out of scope.

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

The API provides a synchronization mechanism to learn about registration attempts and existing sessions.

We propose introducing a new Observer object to allow state-querying capabilities. There are two different report types, described below. We suggest using the Observer interface over the more common "event listener" pattern, specifically for the buffering capability described [in this subsection](#past-attempts-buffer).

### Registration attempts

Registration attempts are triggered by HTTP headers. They may result from a local `fetch()` call, but also a navigation response, a subresource fetch, etc.

A simple use case with an inline fetch would look like this:

```js
if (typeof DeviceBoundSessionsObserver === "undefined") return;

const observer = new DeviceBoundSessionsObserver((reports, observer) => {
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
});
observer.observe();
// Fetch a Secure-Session-Registration header
await fetch('https://id.example.com/start-dbsc');
```

Below are examples of typical successful and failed `registration` reports. Success reports typically come with a `session` report, see [further down](#querying-sessions).

```json
{
  "type": "registration",
  "success": true,
  "registrationUrl": "https://id.example.com/register",
  "endTime": 1780493529912
}
```

```json
{
  "type": "registration",
  "success": false,
  "endTime": 1780493529912,
  "registrationUrl": "https://id.example.com/register",
  "reason": "Persistent server error",
  "code": 500
}
```

All time durations and timestamps are in milliseconds for consistency.

Attempts to [federate](https://w3c.github.io/webappsec-dbsc/#federated-sessions) have more details, such as `provider URL` and `provider session id`. Note that even successful attempts do not have a `sessionId` or `sessionOrigin`. We propose a `session` report type below to convey that information.

#### Past attempts buffer

A navigation response itself may contain DBSC registration headers. Scripts on the resulting page may miss such registration attempts. It's therefore necessary to grant visibility into past registration attempts. The observer constructor takes a `buffered` option that causes it to immediately report past attempts. See the example code in the next section.

This functionality is inspired by the [report buffer of `ReportingObserver`](https://www.w3.org/TR/reporting-1/#observers). We borrow the term "buffer" for consistency. Buffered events are not "consumed" by an Observer instance, but stay available to other instances.

Defining partitioned buffer space requires some consideration. We explore how this works in a section about [attempts visibility](#registration-attempts-visibility) and evaluate a few [alternative approaches](#alternative-registration-attempts-visibility).

### Querying sessions

An established DBSC session is identified by a unique string (`session identifier`) on a [registrable domain](https://url.spec.whatwg.org/#host-registrable-domain). Browsers store additional metadata about these sessions. Websites must be able to list active sessions they are authorized to see. This allows the page to learn more details about the session.

With the observer's buffer capability we can consolidate this feature into the observer itself. When created with `{ buffered: true }`, the observer will also immediately report existing sessions. Only sessions [visible from the current origin](#existing-sessions-visibility) are included.

Here is an example of how to query existing sessions:

```js
new DeviceBoundSessionsObserver(
  (reports, observer) => {
    reports.forEach((report) => {
      if (
        report.type === 'session' &&
        report.refreshUrl.endsWith('/refreshFoo')
      ) {
        // Inspect report.sessionId etc.
      }
    });
  },
  {
    buffered: true,
  },
).observe();
```

This code will match a session whose refresh URL is `/refreshFoo`. If sessions exist the code runs immediately. It runs again when a registration successfully creates a new session or overwrites one.

This is an example `session` report:

```json
{
  "type": "session",
  "sessionOrigin": "https://example.com",
  "sessionId": "foo",
  "refreshUrl": "/refreshFoo",
  "creationTime": 1780493529912,
  "sessionScope": { ... },
  "refreshDueIn": 600000
}
```

### Timing out

Due to the complex and networked nature of DBSC, placing an upper bound on the observing time will be extremely common practice. Instead of forcing developers to use a separate mechanism, we propose to make `AbortSignal` a first-class citizen of this API. This would be an additional option passed to the constructor:

```js
const dbscObserver = new DeviceBoundSessionsObserver(
  (reports, observer) => { /* ... */ },
  { signal: AbortSignal.timeout(3000) },
)
```

This allows developers to use an `AbortController` and link not only timeouts but also user cancellation, page teardowns, etc.

## Detailed design discussion

### Open question: Web Workers

This discussion relates to the [scoping](#scoping) section further down.

Established sessions (`session` reports) are virtually stored in the User Agent's Storage Shed (similar to `IndexedDB`). This means Web Workers have access to them without needing special treatment.

Registration attempts (`registration` reports) on the other hand go into the Traversable Navigable's Storage Shed (similar to `sessionStorage`). When there's no Traversable Navigable, these reports are at least visible to their Environment Settings Object. This means the caller, Web Worker or not, of a registration-triggering `fetch()` will have visibility into the resulting report.

Since most Web Workers do not have a Traversable Navigable, they will only have visibility into their "own" registration attempts. They cannot see as much as page scripts.

**Feedback most welcome:** If this visibility constraint blocks certain Web Worker use cases, we could change `registration` reports visibility to the broader User Agent's Storage Shed, alongside `session` reports. This may have implications, for example on privacy or multi-threading, hence our current proposal to limit registration visibility.

### Open question: Proof generation (signed `fetch()`)

We are exploring a point-in-time challenge mechanism, such as `fetch(url, { deviceBoundSessionsId: 'session_id' })`. The resulting request would then contain a cryptographic proof of possession of the private key of `session_id` (assuming such a session exists, and the request is [in scope of that session](https://w3c.github.io/webappsec-dbsc/#algo-url-in-scope)).

The proof of possession could be a JWT ([RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)) or an HTTP Message Signature ([RFC 9421](https://datatracker.ietf.org/doc/html/rfc9421)).

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

Instead of using the observer for everything we could extend the user agent `Navigator` interface. For DBSC, the pattern above would become:

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

Developers who truly need to only list existing sessions can still use the observer, although it's a more cumbersome setup and cannot be synchronous.

### async / await registration query

The [first code example](#registration-attempts) of this proposal would arguably be more ergonomic if scripts could simply await a Promise. An initial draft proposed a Promise-returning `navigator.deviceBoundSessions.waitForRegistrations()` that would resolve once all pending registrations finish. It would also automatically include buffered events, reporting a status list of all past registrations. This is less in line with modern APIs and seems like a generally worse developer experience.

### Client-initiated refresh (`refresh()`)

We considered a `navigator.deviceBoundSessions.refresh()` method that would trigger a cookie refresh. However, the existing spec already encourages proactive background refreshes by the browser. Site-initiated refreshes might be counter-productive, especially if we introduce a [point-in-time challenge mechanism](#open-question-proof-generation-signed-fetch).

### Registration trigger (`start()`)

We considered an active registration API (e.g. `navigator.deviceBoundSessions.start()`) to initiate registrations purely via JS instead of HTTP headers. We're leaning towards abandoning this part of the API:

- Redundancy: The proposed read-only API combined with a `fetch()` call is functionally very similar to a dedicated `start()` method, making its added value minimal.
- Security risks: Exposing the cryptographic challenge directly to the JS environment introduces risks. A malicious browser extension could intercept the challenge, block the registration, and complete it with their own exportable private key (key-swapping).
  - We could mitigate this attack by mandating a challenge-issuing endpoint called by the browser itself. But that approach also has security caveats for IdPs and strengthens the redundancy argument above.

### Alternative existing sessions visibility

Instead of allowing the refresh origin to query existing session (see [below](#existing-sessions-visibility)), we could mandate a simpler, stricter Same-Origin Policy. The only origin allowed to query existing sessions is the session's origin. Subdomains that need DBSC information would then host an iframe to the main origin. The iframe calls the API (note the permissions policy for cross-origin frames, [also below](#permissions-policy)) and posts the status back to the parent window via `window.parent.postMessage()`. But coordinating events promotes bespoke solutions and increases the attack surface.

Moreover, when the session's and refresh endpoint's origin are different, the latter is often on a hardened, more secure subdomain. Giving it direct access to DBSC data encourages better security architectures.

On the contrary, sites may want to include more domains from which existing sessions are visible. It could be interesting to add a new [session scope](https://w3c.github.io/webappsec-dbsc/#session-scope) field to expand the allowed origins. But we feel this is not necessary and too error prone.

### Alternative registration attempts visibility

Our [scope proposal below](#registration-attempts-visibility) ties registration events to the Traversable Navigable. In other words, different pages in the same tab can observe the status of past registrations.

Registration attempts could have a broader origin scope, similar to that of `localStorage`. While this alternative doesn't raise obvious privacy concerns it introduces extra complexity. We discuss this more in [this open question regarding Web Workers](#open-question-web-workers).

Alternatively, attempts may also have a more restricted, Document-like scope. Statuses would only be retained in the specific environment (like the current `WindowOrWorkerGlobalScope`) that triggered the registration request. This sidesteps most privacy and race condition concerns but also prevents multi-page use cases requiring attempt details, especially about failures.

### Exposing all session details

We think it's unnecessary and unwise to expose certain details about a session. For example, reporting a session's public key might prompt a site to rely on this capability for challenge verification, which would expose their sessions to a range of attacks.

## Security and Privacy Considerations

Since this API exposes cryptographic session states, we suggest enforcing strict boundaries.

### Secure contexts

The JS features are only available over [Secure Contexts](https://www.w3.org/TR/secure-contexts/), like the rest of DBSC.

### Scoping

The proposal has two different report types: `registration` and `session`. This section explains what report types are visible to what scripts using abstractions from the [Storage spec](https://storage.spec.whatwg.org/), like Storage Shed, Shelf, and Key.

Borrowing Storage concepts hopefully makes DBSC events visibility clear, while inheriting several privacy-preserving properties like opaque origins handling or partitioned states. We do not, however, suggest implementing a new formal Storage Endpoint for DBSC metadata, mainly because of the [ESO fallback](#registration-attempts-visibility) feature we describe further down.

#### Existing sessions visibility

An established DBSC session has a registrable domain (called its [`origin`](https://w3c.github.io/webappsec-dbsc/#framework-scope)) and a [`refresh URL`](https://w3c.github.io/webappsec-dbsc/#framework-session). These must be same-site but can be on different origins [under certain conditions](https://w3c.github.io/webappsec-dbsc/#ref-for-json-session-scope-include_site). A session doesn't automatically cover its whole `origin`: cookie policies [also apply](https://w3c.github.io/webappsec-dbsc/#privacy-cookies) to DBSC.

DBSC sessions are inherently unpartitioned, keyed only by their registrable domain. They also explicitly [do not support CHIPS](https://w3c.github.io/webappsec-dbsc/#ref-for-json-session-credential%E2%91%A0).

In Storage terms, DBSC session metadata live in the User Agent's (`local`) Shed, in the Shelf addressed by the first-party key `[session's registrable domain, session's origin]`. We propose to mirror this to another Shelf, `[session's registrable domain, session's refresh URL's origin]`.

> [!NOTE]
> **Rationale for including the refresh origin by default:**
> Large sites often have a hardened subdomain dedicated to authentication, with stronger XSS protections. This is a natural origin for hosting refresh endpoints. Allowing direct access to DBSC data from such a subdomain encourages better security architectures. We discuss this further in [this considered alternative](#alternative-existing-sessions-visibility).

#### Cross-site restrictions

For clients with partitioned storage and cross-site restrictions, an iframe `I` on the top-level site `S` would have the Storage Key `[S,I]`. It needs to use the Storage Access API to elevate its access to `[I,I]` and gain access to the DBSC session data stored there. This also aligns nicely with how SAA grants access to third-party cookies, which DBSC has deep ties to.

If needed, we could add a new entry to the `types` argument of `requestStorageAccess` to grant access only to DBSC data, as opposed to the whole `localStorage`.

#### Registration attempts visibility

A registration's URL is derived from a response's URL. The DBSC spec [requires](https://w3c.github.io/webappsec-dbsc/#ref-for-concept-request-url%E2%91%A1) this to be same-site with the originating request's origin.

Successful registration attempts also spawn a new `session` report, whose visibility follows [its own rules](#existing-sessions-visibility).

In Storage terms, registration attempts live in a Traversable Navigable's (`session`) Shed, in the Shelf addressed by the Key `[originating request's site, registration endpoint's origin]`.

The Traversable Navigable can be looked up from the request's [`client`](https://fetch.spec.whatwg.org/#concept-request-client) (or deduced from its `reserved client` for navigation requests).

If no Traversable Navigable exists, for example in certain Web Workers, `registration` reports are instead stored alongside the Environment Settings Object (ESO) found in the request's `client`. Such attempts have more limited visibility. We discuss this in the [open question regarding Web Workers](#open-question-web-workers).

Registration events that lack a Traversable Navigable and have a `null` request `client` are simply not observable through the JS API.

Browsers should also put a reasonable limit on the number of attempts stored in each buffer and their time-to-live. This prevents privacy and memory issues, especially in long-lived Single Page Applications or background Web Workers.

### Permissions policy

Cross-origin frames cannot call the DBSC API unless explicitly authorized via a permission policy (e.g. `allow="dbsc-session-credentials-get"`). This mirrors other granular, storage-related permissions like [`private-aggregation`](https://patcg-individual-drafts.github.io/private-aggregation-api/#exposing).

### Tracking vectors

A session's details must not be readable when its cookies would not be included in a request. Sessions and keys [must be deleted](https://w3c.github.io/webappsec-dbsc/#privacy-considerations) alongside cleared site data, making them unavailable for lookup. This restriction includes Incognito modes or third-party contexts for user agents that block `SameSite=None` cookies.

This proposal helps with [SSO handoffs](#sso-handoffs) but does not bypass third-party cookie restrictions or cross-site tracking protections. The underlying identity federation still relies on standard platform capabilities (e.g. FedCM, OAuth redirects).

This API should not introduce additional surface for user tracking. Registration error statuses stay high level and simple to avoid new device fingerprinting avenues. Browsers should also add latency fuzzing or clamping to asynchronous methods involving cryptographic operations, to avoid timing side-channels.

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
