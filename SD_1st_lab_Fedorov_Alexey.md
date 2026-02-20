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
9. **A9**: There is an administrator role that can manage users and content.
10. **A10**: The video processing queue is a message broker (e.g., RabbitMQ or Kafka).

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

### DFD 3: Video Search & Streaming

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

### DFD 4: Commenting on a Video

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
| T2 | AS1 – User Credentials | Information Disclosure | Credentials intercepted in transit. | Login endpoint does not enforce HTTPS, or internal DB connections are unencrypted (see A8). | 8.1 (High) | Enforce TLS/HTTPS for all client-facing endpoints. Use TLS for internal connections to the database. |
| T3 | AS1 – User Credentials | Spoofing | Attacker gains access to a user's account via brute-force attack. | No rate limiting or account lockout on the login endpoint. | 7.5 (High) | Implement rate limiting, account lockout after N failed attempts, and support multi-factor authentication (MFA). |
| T4 | AS1 – User Credentials | Spoofing | Attacker performs credential stuffing using leaked credentials from other services. | No detection of automated login attempts; no MFA. | 7.5 (High) | Implement CAPTCHA on login, anomaly detection for login patterns, and MFA. |
| T5 | AS2 – User Profile Data | Tampering | Attacker modifies another user's profile data. | Broken access control — profile update endpoint does not properly verify that the requester owns the profile (IDOR). | 6.5 (Medium) | Enforce server-side authorization checks; validate that the authenticated user ID matches the profile being modified. |
| T6 | AS2 – User Profile Data | Information Disclosure | Private profile fields (e.g., email) exposed to unauthorized users. | API returns full user object including private fields to any requester. | 5.3 (Medium) | Implement field-level access control; return only public fields to non-owners. |
| T7 | AS3 – User View History | Information Disclosure | An attacker accesses another user's view history. | IDOR vulnerability on the view history endpoint — user ID parameter is not validated against the session. | 6.5 (Medium) | Enforce that the view history endpoint only returns data for the authenticated user; do not accept user ID as a client parameter. |
| T8 | AS3 – User View History | Repudiation | A user denies having viewed specific content. | View history records can be deleted by the user or are not logged immutably. | 3.5 (Low) | Implement immutable audit logs for view history; separate user-facing "clear history" from internal audit trail. |
| T9 | AS4 – User Upload History | Tampering | Attacker manipulates upload history to hide or attribute uploads to another user. | Insufficient authorization on upload history records; no integrity checks. | 5.3 (Medium) | Upload history should be derived from video metadata ownership and protected by the same access controls. Use audit logging. |
| T10 | AS5 – Video Metadata | Tampering | Attacker modifies video metadata (title, description, visibility) of another user's video. | IDOR on video management endpoint — no ownership validation. | 7.1 (High) | Server-side ownership checks on all video management operations. |
| T11 | AS5 – Video Metadata | Elevation of Privilege | A regular user changes a private/hidden video to public to expose another user's content, or vice versa. | Authorization logic does not distinguish between video owner and other authenticated users. | 7.1 (High) | Strictly enforce that only the video owner (TL3) or admin (TL4) can change visibility settings. |
| T12 | AS5 – Video Metadata | Information Disclosure | Hidden or private video metadata appears in search results or is accessible via direct API call. | Search query does not properly filter by visibility; direct access to video metadata endpoint lacks authorization checks. | 5.3 (Medium) | Apply visibility filters at the database query level; enforce access control on direct video metadata access endpoints. |
| T13 | AS6 – Video Objects | Information Disclosure | Attacker accesses private video files directly from the object store by guessing or enumerating URLs. | Video object store URLs are predictable (e.g., sequential IDs) and not protected by authentication. | 7.5 (High) | Use signed, time-limited URLs for video access. Enforce authentication and authorization before generating download/stream URLs. Use non-guessable identifiers (UUIDs). |
| T14 | AS6 – Video Objects | Tampering | Attacker replaces a video file in the object store with malicious content. | Object store write access is not properly restricted; no integrity verification. | 7.5 (High) | Restrict write access to the object store to Video Upload Servers only. Implement checksum verification. Use IAM policies to control access. |
| T15 | AS6 – Video Objects | Denial of Service | Attacker uploads excessively large or numerous video files to exhaust storage and processing resources. | No file size limit, no upload rate limiting, no per-user quota. | 7.5 (High) | Enforce maximum file size limits, per-user upload quotas, and rate limiting on the upload endpoint. Validate file type before processing. |
| T16 | AS7 – Comments | Tampering | Attacker modifies or deletes another user's comments. | IDOR on comment deletion/edit endpoint — no ownership check. | 5.3 (Medium) | Enforce that only the comment author (or video owner / admin) can modify or delete a comment. |
| T17 | AS7 – Comments | Spoofing | Attacker posts comments impersonating another user. | Comment creation endpoint accepts a user ID parameter from the client instead of deriving it from the session. | 6.5 (Medium) | Always derive the comment author from the server-side session/token; never trust client-supplied user IDs. |

## Conclusion

This threat model identifies 35 threats across 12 assets of the YouTube-like video platform. The most critical threats (CVSS ≥ 9.0) involve SQL injection (T27, T28), JWT forgery (T21), plaintext password storage (T1), insecure admin access (T30), and privilege escalation (T31). These should be addressed as highest priority during the design and implementation phase.

Key recommendations for the development team:

- **Secure authentication**: Use bcrypt/Argon2 for password hashing, enforce MFA for all accounts (especially admin), use strong JWT signing, and implement rate limiting on login endpoints.
- **Authorization**: Apply strict server-side ownership and role checks on every endpoint. Never trust client-supplied user IDs or roles. Use IDOR-resistant patterns.
- **Input validation**: Use parameterized queries exclusively to prevent SQL injection. Sanitize all user-generated content (comments, video metadata) to prevent XSS.
- **Encryption in transit**: Enforce HTTPS for all client-facing endpoints. Enable TLS for all internal communication (database, message queue, object store).
- **Access control on storage**: Use signed, time-limited URLs for video object access. Restrict object store write permissions to upload servers only.
- **Rate limiting and quotas**: Apply rate limits on uploads, comments, search, and login. Enforce per-user storage quotas. Deploy DDoS protection.
- **Secure video processing**: Validate uploaded files before transcoding. Run transcoders in sandboxed containers with restricted privileges.
- **Monitoring and logging**: Implement audit logging for authentication events, content changes, and administrative actions. Monitor for anomalous patterns.

Addressing these issues at the design stage will significantly reduce the attack surface and improve the overall security posture of the platform.
