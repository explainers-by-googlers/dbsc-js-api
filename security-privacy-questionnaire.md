# Self-Review Questionnaire: Security and Privacy

1. What information does this feature expose, and for what purposes?

    This API exposes the status of active and pending Device Bound Session Credentials (DBSC) sessions. This includes:

    * Unique session identifiers.
    * Session registration status: created/failed timestamps, generic reason strings.
    * Session configuration metadata: refresh URLs, session scope (domain patterns and paths), session credentials (cookies), and duration until next refresh.

    It does not expose public or private key material, hardware serial numbers, or stable device identifiers.

    Its purpose is to give sites visibility into the inherently asynchronous process of header-based DBSC regsitration.

2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

    Yes, a session's details, such as its registration endpoint or credentials, are important for sites to correctly identify the session.

3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

    No. DBSC sessions are bound to pseudonymous, site-specific credentials. No direct user-identifiable details are exposed by the API, unless the credentials themselves (e.g. cookie names) reveal such information.

4. How do the features in your specification deal with sensitive information?

    A session's credentials list may include the names of cookies that are `HttpOnly` or `SameSite=Strict`. The specification mandates that cookies are redacted from the list if they would otherwise not be available to JS (e.g. via `document.cookie`).

5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?

    Not beyond session configuration data (see question 1), including session identifiers (see question 13), and error messages (see question 20).

6. Do the features in your specification introduce state that persists across browsing sessions?

    Yes, indirectly. The underlying DBSC specification persists session configuration across browser restarts. However, this JS API does not introduce any new stored data, it queries the status of existing sessions. The main DBSC spec also prevents "zombie cookie" tracking by mandating that all persisted session metadata and keys must be permanently deleted whenever the user clears site data or cookies for that origin.

7. Do the features in your specification expose information about the underlying platform to origins?

    No. While DBSC operations may occur on hardware (such as a TPM), the JS API hides granular hardware capabilities. To prevent hardware configuration leakage, the API surfaces generic reason strings instead of detailed hardware-level failure codes. It provides no additional mechanism to query specific supported cryptographic chips or algorithms.

8. Does this specification allow an origin to send data to the underlying platform?

    No.

9. Do features in this specification enable access to device sensors?

    No.

10. Do features in this specification enable new script execution/loading mechanisms?

    No.

11. Do features in this specification allow an origin to access other devices?

    No.

12. Do features in this specification allow an origin some measure of control over a user agent’s native UI?

    No.

13. What temporary identifiers do the features in this specification create or expose to the web?

    The API exposes `sessionId` associated with active sessions. However, these are temporary session identifiers that are already known to the server. They are scoped to the DBSC session's origin and that of its refresh endpoint, and cannot be used to link identities across different sites.

14. How does this specification distinguish between behavior in first-party and third-party contexts?

    In third-party contexts, the JS API is only allowed to read inherently unpartitioned DBSC session data if allowed by policies (e.g. via Storage Access API). Because DBSC relies on the network stack setting in-scope cookies, third-party registrations will automatically fail in user agents that block third-party cookies. A corresponding error for the failed registration attempt is all the API will surface.

15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

    DBSC sessions established in normal browsing mode are completely isolated from Private/Incognito windows, and vice versa. The API does not grant visibility across these boundaries.

16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

    Not yet. The explainer consolidates both into one section. The specification could separate the two.

17. Do features in your specification enable origins to downgrade default security protections?

    No.

18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

    Cached Observers stop listening for events. Upon reuse, existing Observers resume active listening. Buffered Observers replay events that occurred while the document was cached. New buffered Observers replay the whole event buffer.

19. What happens when a document that uses your feature gets disconnected?

    Disconnected Observers stop listening for events.

20. Does your spec define when and how new kinds of errors should be raised?

    Yes, DBSC registrations can fail for a number of reasons. The failure report shows coarse-grained messages to block hardware profiling.

21. Does your feature allow sites to learn about the user’s use of assistive technology?

    No.

22. What should this questionnaire have asked?

    1. Does your feature expose any timing or performance measurements?

        Yes, potentially. Hardware-backed key minting and signature speeds vary depending on the device's security chip. The spec may recommend latency fuzzing or clamping on `registration.endTime` to eliminate this timing side-channel.

    2. How does your spec handle local compromise (Actor-in-the-Browser)?

        Malicious extensions may tamper with JavaScript objects and alter the API output, potentially masking enrollment successes and causing DoS. This is a risk sites must accept when using the DBSC JS API.
