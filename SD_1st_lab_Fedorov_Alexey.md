# SD Lab 1: Threat Modeling

## Completed by Fedorov Alexey (tg @ullibniss)

---

## Introduction

This report presents a threat model for a YouTube-like video sharing platform currently in the design phase. The analysis follows the OWASP Threat Modeling Process and applies the STRIDE methodology to systematically identify threats, vulnerabilities, and countermeasures. The goal is to provide the development team with actionable security guidance before implementation begins.

## Assumptions

The following assumptions were made about the system during the analysis:

1. **A1**: The application is a web application accessed via standard web browsers over HTTPS.
2. **A2**: Users must register and authenticate to upload videos, comment, and view their history. Browsing and searching public videos is available to anonymous users.
3. **A3**: Authentication is session-based (cookies) or token-based (JWT).
4. **A4**: The application servers expose a REST API consumed by a frontend (SPA or server-rendered pages).
5. **A5**: The database stores user data, video metadata, comments, upload history, and view history. It is a relational database (e.g., PostgreSQL).
6. **A6**: Video objects (the actual binary files) are stored in a dedicated object store (e.g., S3-compatible storage).
7. **A7**: When a user uploads a video, the application server sends a task to the video processing queue. Video upload servers pick up the task, transcode the video into multiple quality levels, and store results in the object store.
8. **A8**: Internal communication between application servers, database, queue, and upload servers happens over a private network, but is **not encrypted by default** (this is a common design gap we will address).

## 1. Application Decomposition

### Trust Levels

| ID | Trust Level | Description |
|----|-------------|-------------|
| TL1 | Anonymous User | A user who has not authenticated. Can browse and search public videos, view public profiles. |
| TL2 | Authenticated User | A registered and logged-in user. Can upload/delete own videos, comment, view own history, manage own profile. |
| TL3 | Video Owner | An authenticated user with ownership rights over a specific video. Can set video visibility (public, private, hidden), delete own video and its comments. |
| TL4 | Administrator | A privileged user who can manage all users, videos, and comments. Has access to admin panels and system configuration. |
| TL5 | Application Server Process | Internal server-side process that handles business logic, communicates with database, queue, and object store. |
| TL6 | Video Upload Server Process | Internal process that picks up transcoding tasks from the queue and writes processed video files to object storage. |
| TL7 | Database Process | The database management system process that stores and retrieves all application data. |

### Entry Points

| ID | Entry Point | Description | Trust Level(s) |
|----|-------------|-------------|----------------|
| EP1 | HTTPS Web Interface (Main Page) | The main web application frontend served to users' browsers. | TL1, TL2, TL3, TL4 |
| EP2 | Login / Registration Endpoint | API endpoint for user authentication and account creation. | TL1 |
| EP3 | User Profile Endpoint | API endpoint for viewing and editing user profile data. | TL1 (view public), TL2 (edit own) |
| EP4 | Video Upload Endpoint | API endpoint that accepts video file uploads from authenticated users. | TL2 |
| EP5 | Video Streaming Endpoint | API endpoint that serves video streams in the requested quality. | TL1, TL2 |
| EP6 | Video Search Endpoint | API endpoint for searching, filtering, and sorting videos. | TL1, TL2 |
| EP7 | Comment API Endpoint | API endpoint for creating, viewing, and deleting comments on videos. | TL1 (view), TL2 (create/delete own) |
| EP8 | Video Management Endpoint | API endpoint for deleting videos and changing visibility settings. | TL3, TL4 |
| EP9 | View History Endpoint | API endpoint for retrieving the authenticated user's viewing history. | TL2 |
| EP10 | Admin Panel Endpoint | API endpoint / web interface for administrative operations. | TL4 |
| EP11 | Video Processing Queue Interface | Internal interface where application servers enqueue transcoding tasks and upload servers dequeue them. | TL5, TL6 |
| EP12 | Database Connection | Internal interface for SQL / query access to the database. | TL5 |
| EP13 | Object Store API | Internal interface for reading/writing video files to the object store. | TL5, TL6 |

### Assets

| ID | Asset | Description | Trust Level(s) Required |
|----|-------|-------------|------------------------|
| AS1 | User Credentials | Usernames, email addresses, and password hashes. | TL4, TL5 |
| AS2 | User Profile Data | Display name, avatar, bio, account settings. | TL2 (own), TL4 |
| AS3 | User View History | Record of videos a user has watched. | TL2 (own), TL4 |
| AS4 | User Upload History | Record of videos a user has uploaded. | TL2 (own), TL4 |
| AS5 | Video Metadata | Title, description, visibility setting (public/private/hidden), upload date, view count, owner reference. | TL2 (own), TL4 |
| AS6 | Video Objects (Files) | The actual video binary files stored in the object store in various quality levels. | TL1 (public), TL3 (private), TL5, TL6 |
| AS7 | Comments | Text comments on videos with author and timestamp. | TL1 (read public), TL2 (create/delete own), TL4 |
| AS8 | Session / Auth Tokens | Session cookies or JWTs that grant authenticated access. | TL2, TL3, TL4 |
| AS9 | System Availability | The overall availability of the platform to serve video streams and API requests. | All |
| AS10 | Video Processing Queue Messages | Internal task messages containing video processing instructions. | TL5, TL6 |
| AS11 | Database Integrity | Correctness and consistency of all stored data. | TL5, TL7 |
| AS12 | Admin Credentials & Access | Administrator authentication tokens and elevated privileges. | TL4 |

## 2. Data Flow Diagrams

### 1: User Authentication & Profile Access

<img width="1914" height="958" alt="image" src="https://github.com/user-attachments/assets/c38cc176-51af-46be-abd6-693ffb00a4a7" />

**Flow Description:**

1. The user sends login credentials (username/email + password) over HTTPS to the App Server.
2. The App Server validates credentials against the Database (step 3-4) by querying the stored password hash.
3. On successful authentication, the App Server generates a session token / JWT and returns it to the user along with profile data.
4. For profile viewing, the authenticated user requests profile data; the App Server fetches it from the Database and returns it.

**Trust Boundaries Crossed:**
- Internet ↔ App Server (EP1, EP2, EP3): User input crosses from an untrusted zone into the application.
- App Server ↔ Database (EP12): Internal boundary between application logic and data store.

### 2: Video Upload & Processing

<img width="1668" height="1296" alt="image" src="https://github.com/user-attachments/assets/d131cf01-6e96-4613-9bc9-439e3650bd6e" />

**Flow Description:**

1. Authenticated user uploads a video file via HTTPS to the App Server (EP4).
2. App Server stores video metadata (title, description, visibility, owner) in the Database.
3. App Server enqueues a transcoding task in the Video Processing Queue with a reference to the raw video.
4. A Video Upload Server dequeues the task.
5. The Video Upload Server transcodes the video into multiple quality levels and stores the resulting files in the Video Object Store.

**Trust Boundaries Crossed:**
- Internet ↔ App Server: Uploaded file crosses from untrusted user space.
- App Server ↔ Queue / Database / Object Store: Internal boundaries between components.

### 3: Video Search & Streaming

<img width="1224" height="1164" alt="image" src="https://github.com/user-attachments/assets/a4e00bfd-b2c2-4719-8873-ba0aad564ce1" />

**Flow Description:**

1. User sends a search query (with optional filters/sorting) via HTTPS to the App Server (EP6).
2. App Server queries the Database for matching video metadata, applying visibility rules (excluding private/hidden videos for non-owners).
3. Search results (video metadata list) are returned to the user.
4. User selects a video and requests a stream at a desired quality level (EP5).
5. App Server fetches the corresponding video file from the Object Store (or routes through a CDN).
6. Video stream is delivered to the user's browser.

**Trust Boundaries Crossed:**
- Internet ↔ App Server: Search query and streaming request from untrusted user.
- App Server ↔ Database and App Server ↔ Object Store: Internal boundaries.

### 4: Commenting on a Video

<img width="1326" height="716" alt="image" src="https://github.com/user-attachments/assets/d1f05d00-ec77-45d5-89f8-a6946e207e3e" />

**Flow Description:**

1. Authenticated user sends a POST request with comment text to the App Server (EP7).
2. App Server validates the user's authentication, checks authorization (e.g., commenting is allowed on this video), sanitizes input, and stores the comment in the Database.
3. Confirmation and the updated comment list are returned to the user.

**Trust Boundaries Crossed:**
- Internet ↔ App Server: Comment text from untrusted user input.
- App Server ↔ Database: Internal boundary.

## 3. Threat Determination (STRIDE)

The STRIDE model categorizes threats as: **S**poofing, **T**ampering, **R**epudiation, **I**nformation Disclosure, **D**enial of Service, **E**levation of Privilege.

CVSS scores are estimated using CVSS v3.0 base metrics.

### Threats Summary Table

| # | Asset | STRIDE Category | Threat | Vulnerability | CVSS Score | Countermeasure |
|---|-------|----------------|--------|---------------|------------|----------------|
| T1 | AS1 – User Credentials | Information Disclosure | User credentials are exposed and obtained by an attacker. | Passwords stored in plaintext or with a weak hashing algorithm in the database. | 9.1 (Critical) | Use a strong, salted hashing algorithm (bcrypt, Argon2). Never store plaintext passwords. |
| T2 | AS1 – User Credentials | Information Disclosure | Credentials intercepted in transit. | Login endpoint does not enforce HTTPS, or internal DB connections are unencrypted (see A8). | 8.1 | Enforce TLS/HTTPS for all client-facing endpoints. Use TLS for internal connections to the database. |
| T3 | AS1 – User Credentials | Spoofing | Attacker gains access to a user's account via brute-force attack. | No rate limiting or account lockout on the login endpoint. | 7.5  | Implement rate limiting, account lockout after N failed attempts, and support multi-factor authentication (MFA). |
| T4 | AS1 – User Credentials | Spoofing | Attacker performs credential stuffing using leaked credentials from other services. | No detection of automated login attempts; no MFA. | 7.5 | Implement CAPTCHA on login, anomaly detection for login patterns, and MFA. |
| T5 | AS2 – User Profile Data | Tampering | Attacker modifies another user's profile data. | Broken access control — profile update endpoint does not properly verify that the requester owns the profile (IDOR). | 6.5 | Enforce server-side authorization checks; validate that the authenticated user ID matches the profile being modified. |
| T6 | AS2 – User Profile Data | Information Disclosure | Private profile fields (e.g., email) exposed to unauthorized users. | API returns full user object including private fields to any requester. | 5.3| Implement field-level access control; return only public fields to non-owners. |
| T7 | AS3 – User View History | Information Disclosure | An attacker accesses another user's view history. | IDOR vulnerability on the view history endpoint — user ID parameter is not validated against the session. | 6.5 | Enforce that the view history endpoint only returns data for the authenticated user; do not accept user ID as a client parameter. |
| T8 | AS3 – User View History | Repudiation | A user denies having viewed specific content. | View history records can be deleted by the user or are not logged immutably. | 3.5  | Implement immutable audit logs for view history; separate user-facing "clear history" from internal audit trail. |
| T9 | AS7 – Comments | Tampering (XSS) | Attacker injects malicious JavaScript via a comment, which executes in other users' browsers. | Comment text is not sanitized before being rendered in the frontend. | 6.1 | Sanitize and escape all user input on both server and client side. Implement Content Security Policy (CSP) headers. |
| T10 | AS8 – Session / Auth Tokens | Spoofing | Attacker steals a session token via XSS or network sniffing and impersonates the user. | Tokens transmitted over unencrypted channels; XSS vulnerabilities allow token theft. | 8.0 | Enforce HTTPS everywhere. Set HttpOnly and Secure flags on cookies. Implement CSP to mitigate XSS. Use short-lived tokens with refresh rotation. |
| T11 | AS8 – Session / Auth Tokens | Tampering | Attacker forges or modifies a JWT to escalate privileges. | JWT uses a weak signing algorithm (e.g., `none` or HMAC with a guessable secret). | 9.8 (Critical) | Use strong asymmetric signing algorithms (RS256/ES256). Validate token signatures server-side. Never allow `alg: none`. |
| T12 | AS8 – Session / Auth Tokens | Repudiation | Actions performed with a stolen token cannot be attributed to the real attacker. | Insufficient logging of authentication events and token usage. | 4.3 | Log all authentication events with IP addresses and user agents. Implement session audit trails. |
| T13 | AS9 – System Availability | Denial of Service | Application becomes unavailable due to DDoS attack on the web endpoints. | No rate limiting, no CDN, no DDoS protection in front of application servers. | 7.5  | Deploy a CDN / DDoS mitigation service (e.g., Cloudflare). Implement rate limiting per IP. Use auto-scaling for application servers. |
| T14 | AS9 – System Availability | Denial of Service | Video processing queue is overwhelmed by a flood of upload requests, blocking legitimate transcoding. | No prioritization or rate limiting on the queue; single queue for all users. | 6.5  | Implement per-user rate limits on uploads. Use queue prioritization. Set maximum queue depth with back-pressure mechanisms. |
| T15 | AS10 – Queue Messages | Tampering | Attacker injects or modifies messages in the video processing queue to trigger malicious transcoding jobs. | Queue interface is not authenticated; internal network is assumed trusted (A8). | 7.5  | Authenticate all queue producers and consumers. Encrypt queue traffic (TLS). Validate message integrity (e.g., HMAC signatures on messages). |
| T16 | AS10 – Queue Messages | Information Disclosure | Queue messages containing video references or internal paths are intercepted. | Queue communication is unencrypted on the internal network. | 5.3  | Enable TLS for all queue communication. Restrict network access to the queue using firewall rules. |
| T17 | AS11 – Database Integrity | Tampering | Attacker performs SQL injection to modify or delete data. | Application constructs SQL queries using string concatenation with user input. | 9.8 (Critical) | Use parameterized queries / prepared statements exclusively. Employ an ORM. Apply input validation. Run database with least-privilege credentials. |
| T18 | AS11 – Database Integrity | Information Disclosure | Attacker extracts full database contents via SQL injection. | Same as T27 — SQL injection vulnerability. | 9.8 (Critical) | Same as T27. Additionally, encrypt sensitive columns (e.g., emails). Implement database activity monitoring. |
| T19 | AS11 – Database Integrity | Denial of Service | Attacker causes database overload via expensive search queries. | Search endpoint allows complex queries without pagination limits or query timeout. | 6.5  | Enforce pagination on all list endpoints. Set query timeouts. Use database connection pooling. Limit filter/sort complexity. |
| T20 | AS12 – Admin Credentials | Spoofing | Attacker gains admin access through compromised admin credentials. | Admin accounts use weak passwords; no MFA; admin panel exposed on the same domain. | 9.1 (Critical) | Enforce strong passwords for admin accounts. Require MFA. Restrict admin panel access by IP whitelist or VPN. Separate admin domain. |
| T21 | AS12 – Admin Credentials | Elevation of Privilege | Regular user escalates to admin role by manipulating request parameters. | Role stored in client-side token or cookie without server-side verification; or role assignment endpoint is not protected. | 9.8 (Critical) | Store roles server-side. Validate roles on every request. Protect role management endpoints to admin-only access. |
| T22 | AS6 – Video Objects | Tampering | Malicious video file exploits vulnerability in transcoding software (e.g., FFmpeg) leading to remote code execution on upload servers. | Uploaded files are not validated before being passed to the transcoder. | 8.8 | Validate file headers and magic bytes before processing. Run transcoding in sandboxed environments (containers with restricted privileges). Keep transcoding software up to date. |

## Conclusion

This threat model identifies 17 threats across 12 assets of the YouTube-like video platform. The most critical threats (CVSS ≥ 9.0) involve SQL injection (T17, T18), JWT forgery (T11), plaintext password storage (T1), insecure admin access (T20), and privilege escalation (T21). These should be addressed as highest priority during the design and implementation phase.

Key recommendations for the development team:

- **Secure authentication**: Use bcrypt/Argon2 for password hashing, enforce MFA for all accounts (especially admin), use strong JWT signing, and implement rate limiting on login endpoints.
- **Authorization**: Apply strict server-side ownership and role checks on every endpoint. Never trust client-supplied user IDs or roles. Use IDOR-resistant patterns.
- **Input validation**: Use parameterized queries exclusively to prevent SQL injection. Sanitize all user-generated content (comments, video metadata) to prevent XSS.
- **Encryption in transit**: Enforce HTTPS for all client-facing endpoints. Enable TLS for all internal communication (database, message queue, object store).
- **Rate limiting and quotas**: Apply rate limits on uploads, comments, search, and login. Enforce per-user storage quotas. Deploy DDoS protection.
- **Secure video processing**: Validate uploaded files before transcoding. Run transcoders in sandboxed containers with restricted privileges.

Addressing these issues at the design stage will reduce the attack surface and improve the overall security posture of the platform.

# References

1) Lecture 0,1 from course "Secure Developent"
2) https://owasp.org/www-community/Threat_Modeling_Process
3) https://www.cloudflare.com/ru-ru/learning/security/glossary/what-is-threat-modeling/
4) https://julian-wieg.medium.com/cyber-security-incident-response-how-to-use-assumptions-and-believes-to-drive-your-investigation-6c7ae7b176ec
5) https://docs.oort.io/user-trust-level
6) https://en.wikipedia.org/wiki/Asset_(computer_security)



