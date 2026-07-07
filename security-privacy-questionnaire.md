# Self-Review Questionnaire: Security and Privacy

1. What information does this feature expose to the origin?

    This API exposes the status of active and pending Device Bound Session Credentials (DBSC) sessions. This includes:

    * Unique session identifiers (`sessionId`).
    * Session registration health status (created/failed timestamps, generic reason strings), Document-scoped.
    * Session configuration metadata (refresh URLs, session scope, and remaining lifetime), origin(s)-scoped.

    It **does not** expose private key material, hardware serial numbers, or stable device identifiers.

2. Do the features in this specification expose any personally identifiable information (PII)?

    No. DBSC sessions are bound to pseudonymous, site-specific cryptographic session tokens. No direct user-identifiable details (such as names, emails, or device owner identities) are ever exposed by the API.

3. Do the features in this specification expose any temporary identifiers?

    Yes. The API exposes `sessionId` and `refreshUrl` associated with active sessions. However, these are temporary session identifiers that are already known to the registering/refreshing server. They are strictly scoped to the origin and cannot be used to link identities across different sites.

4. Do the features in this specification distinguish between behavior in first-party and third-party contexts?

    Yes. In third-party contexts (such as cross-origin iframes), the JS API is blocked by default. It can only be queried if the top-level document explicitly delegates permission via Permissions Policy. Furthermore, because DBSC relies on the network stack setting in-scope cookies, third-party registrations will automatically fail in user agents that block third-party cookies.

5. Do the features in this specification persist data?

    Yes, indirectly. The underlying DBSC specification persists cryptographic key pairs and session registration states across browser restarts. However, this JS API does not introduce any new storage vectors; it merely queries the status of those existing sessions. The main DBSC spec also prevents "zombie cookie" tracking by mandating that all persisted session metadata and keys must be permanently deleted whenever the user clears site data or cookies for that origin.

6. Do the features in this specification expose any device/hardware state or configuration?

    No. While the underlying cryptographic operations may occur on hardware (such as a TPM), the JS API hides granular hardware capabilities. To prevent hardware configuration leakage, the API returns generic browser exceptions instead of detailed hardware-level failure codes. It provides no additional mechanism to query specific supported cryptographic chips or algorithms.

7. Do the features in this specification allow the origin to detect sensor readings?

    No.

8. Do the features in this specification allow the origin to detect the physical location of the device?

    No.

9. Do the features in this specification allow the origin to access local files?

    No.

10. Do the features in this specification allow the origin to access other devices on the local network?

    No.

11. Do the features in this specification allow the origin to access any system-level capabilities?

    Yes, indirectly. If a point-in-time signature capability like the `fetch()` extension we discuss, the API requests the browser's cryptographic layer to sign payloads using the device's hardware-backed key. This operation is fully managed by the browser's native network service and does not expose direct, low-level access to the system keychain or cryptographic hardware.

12. Do the features in this specification allow the origin to access any peer-to-peer or side-channel communication mechanisms?

    No.

13. Do the features in this specification allow the origin to execute arbitrary code?

    No.

14. Do the features in this specification expose any timing or performance measurements?

    Yes, potentially. Hardware-backed signature generation speeds vary depending on the device's security chip. To eliminate this timing side-channel, the proposal recommends latency fuzzing or clamping (e.g., adding synthetic random delays or rounding execution times to standard intervals) on synchronous operations that involve hardware-key cryptography.

15. Do the features in this specification require secure contexts (HTTPS)?

    Yes, like the underlying DBSC protocol.

16. Do the features in this specification define any permission-gated capabilities?

    Yes. Access to the API in cross-origin frames is guarded by Permissions Policy.

17. Do the features in this specification introduce any privacy or tracking vectors (e.g., cross-site tracking)?

    This specification facilitates federated identity protection. One of the primary use cases is streamlining Single Sign-On (SSO) handoffs between Identity Providers (IdPs) and Relying Parties (RPs). However, the API does not introduce new cross-site tracking vectors. It cannot be used to silently link user identities across unassociated sites without standard web platform authentication flows.

18. Do the features in this specification allow the origin to identify the user's social network accounts or other online accounts?

    No.

19. Do the features in this specification allow the origin to bypass user controls or preferences?

    No. The API respects user privacy actions (such as clearing site data, blocking 3rd party cookies, browsing in Incognito mode). When cookies are cleared, the associated keys and session states are permanently deleted, ensuring they cannot be queried to resurrect previous session states.

20. Do the features in this specification allow the origin to access data in private/Incognito tabs?

    No. DBSC sessions established in normal browsing mode are completely isolated from Private/Incognito windows, and vice versa. Ephemeral sessions established within Private Browsing must use ephemeral keys that are destroyed upon closing the private session.

21. Do the features in this specification expose any data about the user's browsing history or bookmarks?

    No.

22. How does this specification mitigate risks related to security, privacy, and tracking?

    The specification enforces multi-layered mitigations:

    * SOP or Opt-In scoping: Visibly limits session status visibility to the origin of the DBSC session (and its refreshing origin if different) by default. Other eTLD+1 origins may be specified if the DBSC session itself already covers them.
    * Fingerprinting Deterrents: Uses sanitized DOMExceptions for errors and mandates latency fuzzing to block hardware profiling.
    * Cookie-Deletion Sync: Follows DBSC’s deletion of cryptographic keys and session records when site data is cleared.

    Explicitly not mitigated:

    * Malicious extensions may tamper with JavaScript objects and alter the API output, potentially masking enrollment successes and causing DoS.
