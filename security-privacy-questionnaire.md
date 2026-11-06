# Self-Review Questionnaire: Security and Privacy

1. What information does this feature expose, and for what purposes?

    This API exposes the status of active Device Bound Session Credentials (DBSC) sessions and per-document registration attempts:

    - Session metadata for visible active sessions: `sessionOrigin`, `sessionId`, `refreshUrl`, and `creationTime`.
    - Per-document registration outcomes: `registrationUrl`, `success`, `endTime`, coarse `reason` strings and HTTP status `code` on failure, `providerUrl` and `providerSessionId` for federated attempts, and the resulting `session` metadata when visible.

    It does not expose public or private key material, cookie names or values, hardware serial numbers, or stable device identifiers.

    Its purpose is to let sites synchronize with the background registration process started by DBSC HTTP headers and inspect active bound sessions.

2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

    Yes. Session identifiers, origins, and refresh URLs let sites identify active sessions without exposing cookie credentials or cryptographic keys. Registration outcomes stay limited to same-origin attempts on the current `Document`.

3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

    No. DBSC sessions bind to pseudonymous, site-specific credentials. The API exposes no user-identifiable details unless the site encodes such information in its own `sessionId` or URLs.

4. How do the features in your specification deal with sensitive information?

    The API never exposes cryptographic key material or cookie names and values (including `HttpOnly` cookies). Scripts see session metadata only when their URL is in scope of the session (or matches its `refresh_url`) and the environment has storage access, which matches the contexts where DBSC cookies apply.

5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?

    Not beyond session metadata (see question 1), session identifiers (see question 13), and coarse registration error reasons (see question 20).

6. Do the features in your specification introduce state that persists across browsing sessions?

    Not beyond what the core DBSC specification already stores. `getSessions()` reads existing DBSC session metadata, which browsers must delete whenever the user clears cookies or site data for that origin (avoiding "zombie cookie" tracking). `waitForRegistrations()` keeps registration outcomes only in ephemeral per-`Document` memory. Browsers discard that memory on unload and never persist it across navigations or restarts.

7. Do the features in your specification expose information about the underlying platform to origins?

    No. DBSC operations may run on hardware such as a TPM, so the JS API hides granular hardware capabilities. Failure outcomes use coarse reason categories instead of hardware-specific error codes, and the API provides no way to query supported cryptographic chips or algorithms.

8. Does this specification allow an origin to send data to the underlying platform?

    No.

9. Do features in this specification enable access to device sensors?

    No.

10. Do features in this specification enable new script execution/loading mechanisms?

    No.

11. Do features in this specification allow an origin to access other devices?

    No.

12. Do features in this specification allow an origin some measure of control over a user agent's native UI?

    No.

13. What temporary identifiers do the features in this specification create or expose to the web?

    The API exposes the `sessionId` of visible active sessions. The server chooses these identifiers at registration time. They apply only within the DBSC session's site (under its session scope and refresh URL) and cannot link identities across sites.

14. How does this specification distinguish between behavior in first-party and third-party contexts?

    In third-party contexts, scripts can read unpartitioned DBSC session metadata (`getSessions()` and the `session` field of a registration outcome) only when browser storage policies allow it, such as after a Storage Access API grant. `waitForRegistrations()` isolates registration outcomes to the `Document` that started them (when `same-origin` with `registrationUrl`). In browsers that block third-party cookies, third-party registrations fail because the network stack cannot set the session's cookies. In that case, `waitForRegistrations()` only surfaces the failure outcome for that `Document`'s own attempt.

15. How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?

    Browsers isolate DBSC sessions in normal windows from Private or Incognito windows in both directions. The API does not grant visibility across that boundary.

16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

    Not yet. The explainer combines both into one section, and the draft specification includes separate headings for future text.

17. Do features in your specification enable origins to downgrade default security protections?

    No.

18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

    `waitForRegistrations()` scopes registration outcomes to the `Document` itself, so registrations on other pages never enter a BFCached `Document`'s buffer. When a page returns from BFCache, its previously recorded outcomes remain on that `Document`, and `getSessions()` reflects the current DBSC session store.

19. What happens when a document that uses your feature gets disconnected?

    When a `Document` becomes inactive or is destroyed, the browser discards its registration buffer and scripts in that context cannot resolve new session or registration queries.

20. Does your spec define when and how new kinds of errors should be raised?

    `waitForRegistrations()` rejects with the `AbortSignal`'s abort reason when aborted (for example `TimeoutError`). Protocol-level DBSC registration failures resolve with `success: false` and coarse-grained `reason` categories to avoid hardware profiling.

21. Does your feature allow sites to learn about the user's use of assistive technology?

    No.

22. What should this questionnaire have asked?

    1. Does your feature expose any timing or performance measurements?

        Yes, potentially. Hardware-backed key generation and signing speeds vary across security chips. Coarsening `endTime` and `creationTime` timestamps (and optionally fuzzing or clamping completion timing) mitigates this side-channel.

    2. How does your spec handle local compromise (Actor-in-the-Browser)?

        Malicious extensions can tamper with JavaScript objects and fake API return values, which can mask enrollment successes and cause DoS. Sites must account for this risk when relying on client-side synchronization.
