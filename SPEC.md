# GoGovSG Service Specification

This is a comprehensive, language-agnostic functional specification of **GoGovSG** — the official Singapore government link shortener. The document captures product behavior in enough detail that an implementer (human or LLM) can recreate the system without reading the existing source. Visual styling is explicitly out of scope; behavior, data, and contracts are not.

The document uses RFC 2119 keywords (`MUST`, `SHOULD`, `MAY`, `MUST NOT`) when specifying conformance requirements. Defaults, field schemas, and error categories are given as concrete values. Implementation-defined choices are flagged.

---

## Table of Contents

1. Problem Statement
2. Goals and Non-Goals
3. System Overview
4. Core Domain Model
5. Deployment Branding
6. Configuration Specification
7. Identifiers, Validation, and Constants
8. Authentication and Session Management
9. Short URL Lifecycle
10. Redirect Service
11. File Hosting
12. Threat Detection
13. Bulk Operations and Async Jobs
14. QR Code Generation
15. Statistics and Analytics
16. Audit Trail (Link History)
17. Public Directory Search
18. Client Application
19. External REST API (v1)
20. Email Delivery
21. Persistence and Caching
22. Observability, Security, and Operations
23. Failure Model and Recovery
24. Reference Algorithms
25. Test and Validation Matrix
26. Implementation Checklist
27. Backward Compatibility (External Surfaces Only)

Appendix A. Serverless Functions
Appendix B. Validation Rules Reference
Appendix C. Metrics Reference
Appendix D. Locale Schema
Appendix E. Public Surface Map

---

## 1. Problem Statement

GoGovSG solves four operational problems for a government link-shortening service:

1. **Trust.** Citizens receiving links over SMS, email, or social media cannot easily distinguish official government URLs from phishing. A government-controlled origin (e.g., `go.gov.sg`) provides a recognizable, official prefix.
2. **Length.** Government URLs are frequently too long to fit into SMS, social-media posts, or printed materials. They must be shortened while preserving the recognizability that commercial shorteners cannot.
3. **Spam filtering.** Commercial shorteners are often blocked or rate-limited by mail providers. A government-controlled origin avoids this.
4. **Officer self-service.** Public officers from authorized email domains MUST be able to create, manage, transfer, and analyze short links themselves without contacting an administrator.

The system also serves as a controlled file-hosting endpoint, so officers can publish documents through a short link without provisioning their own hosting.

---

## 2. Goals and Non-Goals

### 2.1 Goals

- Permit authenticated users from whitelisted email domains to create, edit, transfer, deactivate, and inspect short URLs and short links to uploaded files.
- Provide a fast, cached redirect path that detects malicious destinations at redirect time and refuses to send users to known threats.
- Provide bulk creation of short URLs from a CSV file, optionally accompanied by asynchronously generated QR codes.
- Provide single-click QR code generation in SVG, PNG, and JPEG formats, with a configurable brand logo and accent color.
- Provide per-link click statistics broken down by date, weekday-hour, and device class.
- Provide a public, searchable directory of short links so citizens can verify a link's authenticity before clicking.
- Provide a programmatic REST API for officers who wish to integrate the shortener with their own systems, plus a privileged admin API for provisioning links on behalf of other officers.
- Provide a complete, append-only audit trail of every change to every link.
- Run as a single deployment, with the public name, short-URL hostname, file-hosting hostname, allowed email domain, and copy supplied through runtime configuration.

### 2.2 Non-Goals

- The system does not perform link analytics beyond click counts, device class, day, and weekday-hour. It does not record per-click geolocation or referrer.
- The system does not implement two-factor authentication beyond the OTP itself. Sessions are renewed by re-authenticating with OTP.
- The system does not support arbitrary file MIME types. Allowed file extensions are a fixed allowlist.
- The system does not store unhashed OTPs or API keys. Compromised secrets can only be rotated, not recovered.
- The system does not throttle redirects. The redirect path is intended to be the highest-throughput surface.
- The system does not implement custom internationalization. Only English is supported.

---

## 3. System Overview

### 3.1 Components

A conforming implementation consists of the following logical components. Concrete technology choices for each are implementation-defined; the names below are roles, not products.

1. **HTTP server** that serves three classes of route: redirect (`GET /:shortUrl`), API (`/api/*`), and static assets (the client bundle, locales, transition page, error page).
2. **Primary durable store** holding users, URLs, tags, jobs, click aggregates, and an append-only URL history. Any storage technology that supports transactions, secondary indexes, and full-text search over weighted text fields is acceptable.
3. **Read replica** (optional) for read-heavy paths (redirect lookups, statistics) when the corresponding feature flag is enabled.
4. **Five logical caches** with independent eviction and TTL policies: OTP cache, session store, redirect cache, statistics cache, URL threat-scan cache. They MAY share an underlying storage technology or be separate; only their key spaces and TTLs are normative.
5. **Object store** for hosted files, fronted by a public hostname (§11).
6. **Async job pipeline** for QR-code bundle generation: a durable queue that fans out batches to one or more background workers, which write artifacts back to the object store and notify the server through a callback.
7. **Email transport** for OTP delivery and job-completion notifications.
8. **External integrations** (all optional): URL threat-scanning service, antivirus service, web analytics, observability platform.
9. **Single-page client** comprising five subapplications (home, login, user dashboard, public directory, API integration).
10. **Out-of-band utilities** for link ownership migration and email-delivery event capture (Appendix A).

### 3.2 Layering

The server is organized in three logical layers, regardless of how the source tree is arranged:

- **API layer** — request routing, schema validation, authentication/authorization middleware.
- **Module layer** — controllers and services implementing business rules. The functional modules are: `auth`, `user`, `bulk`, `job`, `qr`, `redirect`, `threat`, `audit`, `analytics`, `statistics`, `directory`, `display`, `api`.
- **Persistence layer** — repositories, model definitions, and caching policies.

Cross-cutting concerns (configuration, logging, metrics, dependency wiring) are implementation-defined.

### 3.3 Process Model

The server runs as one or more long-lived processes that service every HTTP route. There are no in-process background workers other than fire-and-forget side effects (statistics increments, cache warming). Long-running work (bulk QR generation, link migration, email-event capture) MUST run out-of-process and SHOULD be horizontally scalable independent of the HTTP server.

---

## 4. Core Domain Model

The domain consists of seven first-class entities, each with a stable identifier and a defined lifecycle. The schema below is conceptual; concrete representation (table layout, document shape, key encoding, index types) is implementation-defined. What MUST be preserved is the set of attributes, their constraints, and the relationships between entities.

### 4.1 User

Represents one authenticated officer.

| Attribute | Logical type | Constraints |
|-----------|--------------|-------------|
| `id` | opaque identifier | Server-assigned, stable for the lifetime of the user. |
| `email` | string, unique, lowercase | MUST satisfy the validation rules of §7.3. Normalized to lowercase on write. |
| `apiKeyHash` | string, unique, nullable | Stored verifier for an API key (§8.2). |
| `createdAt`, `updatedAt` | timestamp | Server-managed. |

A user is created lazily the first time their email successfully verifies an OTP, or the first time an admin provisions a link on their behalf.

### 4.2 Url

Represents one short link, identified by its short URL slug.

| Attribute | Logical type | Constraints |
|-----------|--------------|-------------|
| `shortUrl` | string identifier | Primary key. MUST match `/^[a-zA-Z0-9-]+$/`. Treated case-insensitively at lookup (§10, §27.2). |
| `longUrl` | string, non-null | MUST validate per §7.2. For file links, holds the public file URL (§11). |
| `state` | enum (`ACTIVE`, `INACTIVE`) | Default `ACTIVE`. Inactive URLs MUST resolve to 404 on redirect. |
| `isFile` | boolean | `true` ⇔ `longUrl` is a file-hosting URL. |
| `contactEmail` | string, nullable | MUST be lowercased and pass the email check of §7.3 if present. |
| `description` | string | Max 200 printable ASCII characters. Default empty. |
| `source` | enum (`BULK`, `API`, `CONSOLE`) | Records origin of creation. |
| `tags` | set of tag references | At most 3 tags per link (§4.3). |
| `tagStrings` | string | Display denormalization of `tags` (e.g., semicolon-separated), used for ordering and search. Implementation-defined whether this is stored or derived. |
| `safeBrowsingExpiry` | timestamp, nullable | Expiry of the last clean URL threat scan (§12.1). |
| `userId` | reference to User, nullable | Owner. |
| `createdAt`, `updatedAt` | timestamp | |

Lookup by `shortUrl` MUST be O(1)-equivalent (hash-indexed or cached). Directory search (§17) requires a ranking mechanism that weights matches in `shortUrl`, `longUrl`, and `description` with progressively decreasing relevance.

### 4.3 Tag

Represents one normalized tag.

| Attribute | Logical type | Constraints |
|-----------|--------------|-------------|
| `id` | opaque identifier | Server-assigned. |
| `tagString` | string, unique | Display form. Matches `/^[A-Za-z0-9_-]+$/`, ≤ 25 chars. |
| `tagKey` | string | Lowercased form of `tagString` used for case-insensitive search. |
| `createdAt`, `updatedAt` | timestamp | |

`Url` ⇄ `Tag` is many-to-many. A link MUST NOT carry more than `MAX_NUM_TAGS_PER_LINK = 3` tags. The same `Tag` MAY be reused across multiple links; the association MUST be reference-counted (implementation-defined) or at least non-destructive on unlink.

### 4.4 UrlHistory

Append-only history of every `Url` mutation. Every create, update, or ownership transfer of a `Url` MUST produce one `UrlHistory` record automatically; bulk mutations that bypass this invariant are forbidden.

| Attribute | Logical type | Notes |
|-----------|--------------|-------|
| `id` | opaque identifier | |
| `urlShortUrl` | reference to Url | Indexed for history retrieval. |
| `userId` | reference to User | Acting user. |
| `longUrl`, `state`, `isFile`, `contactEmail`, `description`, `source`, `tagStrings` | same types as `Url` | Denormalized snapshot at the moment of change. |
| `createdAt`, `updatedAt` | timestamp | |

The audit endpoint (§16) reads history in reverse chronological order and computes change sets pairwise.

### 4.5 Click Statistics

Click counts are tracked in four logical aggregates, all keyed by `shortUrl`. They MAY be stored in a single record, four records, or any other layout that supports atomic increment.

- **Total clicks** — single integer counter per short URL.
- **Device-class totals** — four counters per short URL: `mobile`, `tablet`, `desktop`, `others`.
- **Daily totals** — one counter per (`shortUrl`, `date`) pair.
- **Weekday/hour heatmap** — one counter per (`shortUrl`, weekday ∈ [0,7), hour ∈ [0,24)) triple.

All time-dimensioned counters MUST be keyed in **Asia/Singapore** local time. A total-clicks entry MUST exist for every `Url` (created together with it). The total-clicks aggregate MUST support efficient descending ordering for popularity sorting.

### 4.6 Job and JobItem

Async-job tracking for bulk QR generation.

**Job**:

| Attribute | Logical type | Notes |
|-----------|--------------|-------|
| `id` | opaque identifier | |
| `uuid` | UUID, unique | External identifier exposed to the client. |
| `userId` | reference to User | |
| `status` | enum (`IN_PROGRESS`, `SUCCESS`, `FAILURE`) | Default `IN_PROGRESS`. Computed aggregate over items. |
| `createdAt`, `updatedAt` | timestamp | |

**JobItem**:

| Attribute | Logical type | Notes |
|-----------|--------------|-------|
| `id` | opaque identifier | |
| `jobItemId` | string, unique | Format: `${job.uuid}/${batchIndex}`. Used as the artifact key in the object store and as the worker-callback handle. |
| `jobId` | reference to Job | |
| `status` | enum (`IN_PROGRESS`, `SUCCESS`, `FAILURE`) | Default `IN_PROGRESS`. |
| `message` | string | Free-form failure detail. Default empty. |
| `params` | structured value | Payload supplied to the background worker. |
| `createdAt`, `updatedAt` | timestamp | |

Job aggregate status is computed from items per §13.3.

### 4.7 OTP Record

A short-lived authentication record. Key: `${email}:${ip}`. Value: `{ hashedOtp, retries }`. TTL: `OTP_EXPIRY` seconds (default 300). MUST be evicted automatically on TTL expiry; explicit deletion on successful verification is REQUIRED. Implementation MUST use a store that supports atomic decrement of the retry counter under contention.

---

## 5. Deployment Branding

The system runs as a single deployment. Its public identity (name, hostnames, allowed email domain, on-page copy, QR-code styling) is supplied through runtime configuration rather than being hardwired in source.

The configurable identity surface consists of:

- **`OG_URL`** — the canonical origin URL (e.g., `https://go.gov.sg`). Used for circular-redirect prevention, trusted-referrer detection, and as the basis for constructing the full short link in QR codes.
- **`VALID_EMAIL_GLOB_EXPRESSION`** — the email-domain allowlist (e.g., `*.gov.sg`).
- **`AWS_S3_BUCKET`** — the file-hosting bucket name. In production this is also the hostname of the file-serving domain (e.g., `file.go.gov.sg`), making the canonical file URL `https://${AWS_S3_BUCKET}/${shortUrl}.${ext}`. See §11.
- **Display name** — the human-readable service name (e.g., "Go.gov.sg") returned in API responses and shown in templated HTML (transition page, 404 page).
- **Locale strings** — the single English copy bundle loaded by the client (§18.9, Appendix D).
- **QR-code brand color and logo** — the dark color used when rendering QR codes (§14) and the centered logo overlay.

A reimplementation that needs only one deployment MAY hard-code these values as long as the External REST API contract (§27.4) and the file URL shape (§27.3) remain configurable through deployment, since they affect URLs already in the wild.

---

## 6. Configuration Specification

Configuration is supplied via environment variables. The application MUST validate required variables at startup and exit with status 1 if any are missing.

### 6.1 Required Variables (all deployments)

| Variable | Description |
|----------|-------------|
| `DB_URI` | Primary durable-store connection string. |
| `REPLICA_URI` | Read-replica connection string. |
| `OG_URL` | Origin URL of the service (e.g., `https://go.gov.sg`); used for circular-redirect prevention and trusted-referrer detection. |
| `REDIS_OTP_URI` | Connection string for the OTP cache. |
| `REDIS_SESSION_URI` | Connection string for the session store. |
| `REDIS_REDIRECT_URI` | Connection string for the redirect cache. |
| `REDIS_STAT_URI` | Connection string for the statistics cache. |
| `REDIS_SAFE_BROWSING_URI` | Connection string for the URL threat-scan cache. |
| `SESSION_SECRET` | Secret for session-token signing and the visit-tracking cookie. |
| `VALID_EMAIL_GLOB_EXPRESSION` | Glob pattern (extended-glob, globstar, brace, and negation are disabled). |
| `AWS_S3_BUCKET` | Bucket name for file uploads. |
| `API_KEY_SALT` | Salt used by the password-hashing function for API key suffixes. |
| Display name | Human-readable service name shown in templated HTML and returned in some API responses. Implementation-defined (env var or build-time constant). |

### 6.2 Production-only Required Variables

`SES_HOST`, `SES_PORT`, `SES_USER`, `SES_PASS` MUST be set when `NODE_ENV !== 'development'`.

### 6.3 Optional Variables and Defaults

| Variable | Default | Purpose |
|----------|---------|---------|
| `NODE_ENV` | `production` | Controls cookie security, log level, OTP rate-limit enforcement. |
| `SALT_ROUNDS` | 10 | Password-hash work factor (bcrypt-equivalent cost) for OTPs and API keys. |
| `OTP_EXPIRY` | 300 (s) | OTP TTL. |
| `REDIRECT_EXPIRY` | 300 (s) | Redirect-cache TTL. |
| `COOKIE_MAX_AGE` | 86_400_000 (ms = 24 h) | Session cookie lifetime. |
| `BULK_UPLOAD_MAX_NUM` | 1000 | Max URLs per CSV. |
| `BULK_UPLOAD_RANDOM_STR_LENGTH` | 8 | Generated short-URL length for bulk. |
| `API_LINK_RANDOM_STR_LENGTH` | 8 | Generated short-URL length for API. |
| `BULK_QR_CODE_BATCH_SIZE` | 1000 | URLs per background-worker batch. |
| `BULK_QR_CODE_BUCKET_URL` | empty | Base URL used when constructing JobItem download URLs. |
| `ACTIVATE_BULK_QR_CODE_GENERATION` | `false` | Master switch for the bulk QR generation pipeline. |
| Async job queue endpoint | implementation-defined | Address (URL, host:port, etc.) at which the background-worker queue accepts new job-batch messages. |
| Async job queue enqueue timeout | 10_000 (ms) | Timeout when enqueueing a job-batch message. |
| `JOB_POLL_INTERVAL` | 5000 (ms) | Server-side long-poll interval. |
| `JOB_POLL_ATTEMPTS` | 12 | Server-side long-poll attempts. After exhaustion respond 408. |
| `FF_EXTERNAL_API` | `false` | Gates `/api/v1/*` and `/api/v1/admin/*`. |
| `FF_USE_REPLICA_FOR_REDIRECTS` | `false` | Use replica DB for redirect lookups. |
| `API_KEY_VERSION` | `v1` | Component of API key string. |
| `ADMIN_API_EMAILS` | empty | Comma-separated emails allowed to call admin API. |
| `SAFE_BROWSING_KEY` | undefined | Credential for the URL threat-scanning service. Disabled when unset. |
| `SAFE_BROWSING_LOG_ONLY` | `false` | When `true`, threats are logged but not blocked. |
| `CLOUDMERSIVE_KEY` | undefined | Antivirus key. When unset, virus scan is skipped. |
| `CSP_REPORT_URI` | undefined | CSP violation reporting endpoint. |
| `CSP_ONLY_REPORT_VIOLATIONS` | `false` | Run CSP in report-only mode. |
| `GA_TRACKING_ID` | undefined | Web-analytics property identifier. |
| `LOGIN_MESSAGE` | undefined | Banner on login page. |
| `USER_MESSAGE` | undefined | Banner on user dashboard. |
| `ANNOUNCEMENT_TITLE`, `ANNOUNCEMENT_SUBTITLE`, `ANNOUNCEMENT_MESSAGE`, `ANNOUNCEMENT_URL`, `ANNOUNCEMENT_IMAGE`, `ANNOUNCEMENT_BUTTON_TEXT` | undefined | Optional logged-in modal. All-or-none semantics: client renders the modal only if `ANNOUNCEMENT_MESSAGE` is truthy. |
| `ROTATED_LINKS` | undefined | Comma-separated short URLs to rotate on the landing page. |
| `USER_COUNT`, `CLICK_COUNT`, `LINK_COUNT` | 77288, 666_820_545, 28_151_439 | Static counters for landing page. |
| `DB_POOL_SIZE` | 40 | Database connection-pool size. |
| `BUCKET_ENDPOINT` | implementation-defined | Object-store endpoint override (used to point at a local emulator in development). |
| `ACCESS_ENDPOINT` | implementation-defined | Object-store access endpoint used to construct browser-facing file URLs in development. |
| `OTP_RATE_LIMIT` | implementation-defined; 0 in development | OTP requests per IP per minute. |
| `DD_SERVICE`, `DD_ENV`, `DD_API_KEY` | undefined | Observability-platform identification (service name, environment, credential). The names are conventional for one popular platform; equivalents in any other observability platform are acceptable. |
| `POSTMAN_API_URL`, `POSTMAN_API_KEY`, `ACTIVATE_POSTMAN_FALLBACK` | undefined / `false` | Optional Postman email fallback. |

### 6.4 Cookie Configuration

Session cookies MUST be named `gogovsg` and set with:

```
httpOnly: true
sameSite: 'strict'
secure: NODE_ENV !== 'development'
maxAge: COOKIE_MAX_AGE
```

A separate cookie (conventionally `visits`) MUST track per-visitor short URL history for transition-page suppression, with `maxAge` = 7 days and signed/encrypted using `SESSION_SECRET`. Its serialized array MUST be capped at `COOKIE_SESSION_MAX_SIZE_BYTES` (default 2000) by LRU eviction.

---

## 7. Identifiers, Validation, and Constants

### 7.1 Short URL

- Pattern: `/^[a-zA-Z0-9-]+$/`.
- Used as the primary identifier of a `Url` entity and as the redirect-cache key (lowercased internally).
- For auto-generation: draw characters uniformly at random from the alphabet `0123456789abcdefghijklmnopqrstuvwxyz` using a cryptographically strong random source. Length is `BULK_UPLOAD_RANDOM_STR_LENGTH` (default 8) for bulk-created links or `API_LINK_RANDOM_STR_LENGTH` (default 8) for API-created links. On collision, retry.

### 7.2 Long URL

Validation is centralized in `src/shared/util/validation.ts` and MUST apply on both client and server.

- MUST be a fully-qualified URL with `https:` scheme. Plain hostnames are rejected.
- MUST have a valid TLD; IP addresses are rejected.
- Validation rejects: non-`https:` schemes, schemeless URLs, hostless URLs, URLs with underscores in the hostname, URLs with a trailing dot, URLs lacking a valid TLD. URLs containing userinfo (`https://user:pass@host/...`) MAY be permitted.
- MUST NOT be circular: the URL's hostname MUST NOT resolve to the service origin (`OG_URL` hostname).
- MUST NOT match the blacklist (substring blocklist sourced from `src/server/resources/blacklist`).

### 7.3 Email

- MUST pass `validator.isEmail()` with `{ allow_utf8_local_part: false }`.
- MUST be lowercased and trimmed before storage and before pattern matching.
- MUST satisfy a glob match against `VALID_EMAIL_GLOB_EXPRESSION` where extended-glob, globstar, brace, and negation expansions are all disabled.

### 7.4 Tag

- Pattern: `/^[A-Za-z0-9_-]+$/`, length ≤ 25.
- At most 3 tags per link, no duplicates within a link.
- `tagKey` is the lowercased `tagString`.

### 7.5 Description

- Length ≤ 200, printable ASCII only (`/^[\x20-\x7F]*$/`).

### 7.6 File

- Max single upload size: 20 MiB.
- Max CSV bulk size: 5 MiB.
- Allowed extensions (case-insensitive): `avi, bmp, csv, docx, dwf, dwg, dxf, gif, jpeg, jpg, mpeg, mpg, ods, pdf, png, pptx, rtf, tif, tiff, txt, xlsx, zip`.
- MIME type MUST be detected by inspecting the file's byte content (i.e., magic-number sniffing) rather than trusting client-supplied MIME headers. If sniffing yields no result, fall back to the filename extension. The following extensions MUST be mapped manually because magic-number sniffing does not yield a useful MIME for them: `csv → text/csv`, `dwf → application/x-dwf`, `dxf → application/dxf`.

### 7.7 Constants

| Constant | Value |
|----------|-------|
| `MAX_CSV_UPLOAD_SIZE` | 5 MiB |
| `MAX_FILE_UPLOAD_SIZE` | 20 MiB |
| `LINK_DESCRIPTION_MAX_LENGTH` | 200 |
| `BULK_UPLOAD_HEADER` | `"Original links to be shortened"` |
| `TAG_SEPARATOR` | `;` |
| `MAX_NUM_TAGS_PER_LINK` | 3 |
| `MIN_TAG_SEARCH_LENGTH` | 3 |
| `DEFAULT_URL_SCAN_RESULT_EXPIRY_SECONDS` | 86_400 (24 h) |

---

## 8. Authentication and Session Management

### 8.1 OTP Login Flow

Authentication is one-factor via email-delivered one-time password. Three endpoints participate:

**`POST /api/login/otp`** — generates and sends an OTP.

Request body:

```
{ email: string (required, lowercase, matches email glob) }
```

Server behavior:

1. Apply an IP-keyed rate limit: window 60 s, max `OTP_RATE_LIMIT`. Return 429 on overflow.
2. Generate a 6-digit numeric OTP using cryptographic randomness.
3. Hash the OTP with a slow, salted password-hashing function (e.g. bcrypt, scrypt, Argon2) using a deployment-wide work factor — see `SALT_ROUNDS` for the bcrypt-equivalent setting.
4. Store `{ hashedOtp, retries: 3 }` in the OTP cache at key `${email}:${ip}` with TTL `OTP_EXPIRY`.
5. Send the unhashed OTP to the supplied email via the configured email transport (§20). The email body MUST include the OTP, the requester's IP, and the deployment's display name.
6. On success, return `200 { message: "OTP generated and sent." }` and increment `OTP_GENERATE_SUCCESS`. On transport failure, increment `OTP_GENERATE_FAILURE` and return 500.

**`POST /api/login/verify`** — verifies an OTP.

Request body:

```
{ email: string (required), otp: string (required) }
```

Server behavior:

1. Fetch `{ hashedOtp, retries }` from the OTP cache at `${email}:${ip}`. If absent, return 401 (expired/not found).
2. Compare the submitted OTP against `hashedOtp` using the password-hashing function's constant-time verify.
3. **Mismatch**: decrement `retries`. If `retries > 0`, rewrite the record and respond 401 `"OTP hash verification failed, ${retries} attempt(s) remaining."`. If `retries == 0`, delete the record and respond 401 (locked out for this email+IP until expiry).
4. **Match**: find-or-create the user by email, establish a session for `{ id, email }`, asynchronously delete the OTP record, return `200 { message: "OTP hash verification ok.", user }`. Emit `OTP_VERIFY_SUCCESS`. Emit `USER_NEW` on first-time creation.

**`GET /api/login/isLoggedIn`** — returns `200 { user }` when an authenticated session exists, `404` otherwise.

**`GET /api/login/emaildomains`** returns the configured email glob to the client; **`GET /api/login/message`** returns `LOGIN_MESSAGE`.

### 8.2 API Key Authentication

API keys authenticate the external REST API. Key structure: `${apiEnv}_${apiKeyVersion}_${randomSuffix}`.

- `apiEnv` distinguishes production keys from non-production keys (conventionally `"live"` vs `"test"`).
- `apiKeyVersion` defaults to `"v1"` and exists so that the hashing scheme MAY be upgraded under a new version without invalidating older keys.
- `randomSuffix` is 32 cryptographically random bytes, base64 encoded.

The server stores `${apiEnv}_${apiKeyVersion}_${hash(randomSuffix, API_KEY_SALT)}` in `User.apiKeyHash`, where `hash` is a slow, salted password-hashing function. The full key is shown to the user exactly once at generation.

**`POST /api/user/apiKey`** (session-authenticated) generates a new key, overwriting any prior key. The response body MUST contain the unhashed key. Emit `API_KEY_GENERATE` with tag `isnew = true|false`.

**`GET /api/user/hasApiKey`** (session-authenticated) returns whether `apiKeyHash` is set.

API key verification:

1. Parse `Authorization: Bearer <key>`. Return 401 if missing or malformed.
2. Split the key on `_`; hash the suffix; look up the user by the rebuilt `${env}_${version}_${hash}`.
3. On hit, associate the request with `user.id`. On miss, return 401.

Admin-only routes additionally require the authenticated user's email to be present in `ADMIN_API_EMAILS`; failure returns 401.

### 8.3 Session Management

Authenticated sessions associate a session token (carried as a cookie) with the principal `{ user: { id, email } }`. The session store technology is implementation-defined; it MUST:

- Persist sessions across process restarts.
- Allow per-session TTL of `COOKIE_MAX_AGE`.
- Support explicit destruction on logout.

Cookie attributes are listed in §6.4. The session cookie name is implementation-defined (the existing implementation uses `gogovsg`).

**`GET /api/logout`** destroys the current session and returns `200 { message: "Logged out" }`.

A session guard MUST reject requests lacking an authenticated session with 401, and on success MUST make the authenticated `userId` available to downstream handlers, so handlers can treat session and API key auth uniformly.

---

## 9. Short URL Lifecycle

### 9.1 Creation (`POST /api/user/url`)

Authentication: session.

Multipart accepted (file upload) or JSON. Validation:

```
shortUrl: string (required, §7.1)
longUrl?: string  // XOR with file
file?: UploadedFile
tags?: string[]   // each per §7.4
description?: string (per §7.5)
contactEmail?: string (per §7.3)
```

Exactly one of `longUrl` or `file` MUST be supplied. Multiple files MUST return 422.

Middleware chain:

1. Multipart upload acceptance with `MAX_FILE_UPLOAD_SIZE` limit.
2. Form-data preprocessing: place the file under a known request field; parse the `tags` JSON if it is a string.
3. Single-file check: exactly one file may be present (or none).
4. File extension/MIME validation (§7.6).
5. File antivirus scan (§12.2).
6. URL threat-scan (§12.1).
7. `userController.createUrl`.

Service behavior (`UrlManagementService.createUrl`):

1. Verify user exists.
2. `urlRepository.isShortUrlAvailable(shortUrl)` — return 400 `AlreadyExistsError` if taken.
3. If file, upload to the object store at key `${shortUrl}.${ext}` and store `longUrl = ${file domain}/${shortUrl}.${ext}`; set `isFile = true`.
4. Insert `urls` row (in a transaction, so `afterCreate` writes `url_clicks` row and `url_history` row).
5. Associate tags (upserting `tags` rows via §13.4 logic, attaching through `url_tag`).
6. `safeBrowsingExpiry = now + DEFAULT_URL_SCAN_RESULT_EXPIRY_SECONDS` if the URL was scanned clean.
7. Emit `SHORTLINK_CREATE` with tags `source` and `isfile`.
8. Return the persisted `StorableUrl`.

### 9.2 Update (`PATCH /api/user/url`)

Authentication: session. Validation:

```
shortUrl: string (required)
longUrl?: string
state?: 'ACTIVE' | 'INACTIVE'
description?: string
contactEmail?: string
tags?: string[]
file?: UploadedFile  // XOR with longUrl
```

Service behavior:

1. Verify user owns the link (`userRepository.findOneUrlForUser`). If not, return 403/404.
2. Reject any attempt to replace a file URL's `longUrl`, or to attach a file to a non-file URL.
3. Apply only the supplied fields. State transitions: any → any.
4. If `longUrl` changed, re-run the URL threat-scan on the new URL.
5. If `file` supplied for an existing file URL, upload the new file and overwrite the existing object at the same key.
6. If tags changed, replace associations atomically.
7. Invalidate the redirect cache entry for this `shortUrl`.
8. Persist update (in transaction → `afterUpdate` hook writes history).
9. Return updated `StorableUrl`.

### 9.3 Ownership Transfer (`PATCH /api/user/url/ownership`)

```
shortUrl: string (required)
newUserEmail: string (required, per §7.3)
```

1. Verify current user owns `shortUrl`. Return 400 `AlreadyOwnLinkError` if `newUserEmail` equals the current user's email.
2. `findOrCreateWithEmail(newUserEmail)`.
3. Update `urls.userId` (transaction, history hook fires).
4. Send notification email to the new owner.

### 9.4 Deactivation

There is no explicit user-facing delete. Setting `state = INACTIVE` removes the link from redirects. The system MAY also auto-deactivate via §12.1.5 (malicious-link detection at redirect time).

### 9.5 Listing (`GET /api/user/url`)

This endpoint is consumed only by the SPA (§18.5). It is **not** part of the External REST API and MAY be redesigned during a rewrite (see §27.6). The shape below describes the current implementation.

Parameters are read from the query string.

```
limit?:         int 0..1000 (default 1000)
offset?:        int (default 0)
orderBy?:       'createdAt' | 'clicks' (default 'createdAt')
sortDirection?: 'asc' | 'desc' (default 'desc')
state?:         'ACTIVE' | 'INACTIVE'
isFile?:        boolean
searchText?:    string   // XOR with tags; default ''
tags?:          string   // semicolon-separated; default ''
```

Returns `{ urls: StorableUrl[], count: number }`. Search uses case-insensitive substring match on `shortUrl` and `longUrl`; tags filter uses case-insensitive wildcard matching against the comma-/semicolon-separated `tagStrings` representation.

### 9.6 Tag Autocomplete (`GET /api/user/tag`)

```
searchText: string (required, length ≥ MIN_TAG_SEARCH_LENGTH, valid tag form)
limit: int (required)
```

Returns `string[]` — tagStrings whose `tagKey` matches `${searchText}%` on the user's own URLs.

### 9.7 Tag Upsert (Transactional)

When a write supplies tags, the server MUST:

1. For each tag string, attempt `findOrCreate` on `tags` keyed by `tagKey`. On unique-constraint races, refetch.
2. Replace the link's `url_tag` associations with the resolved set.
3. Recompute `urls.tagStrings` as the semicolon-joined `tagString`s of the new set.

---

## 10. Redirect Service

### 10.1 Request Path

Route: `GET /:shortUrl`. The endpoint accepts a short URL slug matching `[a-zA-Z0-9-]+`.

Two behaviors of this endpoint are **externally binding** because they affect resolution of links already published in the wild (see §27.2):

- **Trailing-character tolerance**: the captured slug MAY be followed by a single trailing non-slug character (e.g., a `.` appended by an SMS or email client). `/foo.` MUST resolve to the same short URL as `/foo`. The path-matching mechanism is implementation-defined; only the resolution behavior is normative.
- **Case-insensitive lookup**: `/Foo`, `/FOO`, and `/foo` MUST resolve identically. The existing implementation lowercases the captured slug before cache and database lookup. Stored `shortUrl` values in the canonical deployment are effectively lowercase.

Everything else in this section describes the current implementation; rewrites may diverge.

The redirect handler renders the transition page from a server-side template. The current implementation loads its in-page JavaScript from a separate route — `GET /assets/transition-page/js/redirect.js` (a server-rendered `text/javascript` response parameterized with the web-analytics tracking ID and the event category/action names used for the `loaded` and `proceeded` events). This indirection exists because the tracking ID is a runtime configuration value. A reimplementation that inlines the script, or uses a different transition page entirely, is acceptable; this auxiliary route is not part of the external contract.

The current implementation uses a signed visit-tracking cookie (conventionally named `visits`) to suppress the transition page on repeat visits. The cookie name and the mechanism are implementation details; a rewrite may use a different scheme or no mechanism at all.

### 10.2 Resolution Algorithm

```
function redirectFor(shortUrl, pastVisits, userAgent, referrer):
    if not matches /^[a-zA-Z0-9-]+$/: throw NotFoundError
    dest = redirectCache.get(shortUrl) ?? db.lookup(shortUrl)
    if dest is null or dest.state == INACTIVE: throw NotFoundError
    cacheAsync(shortUrl, dest)  # TTL = REDIRECT_EXPIRY

    if not dest.isFile and (dest.safeBrowsingExpiry is null or expired):
        if isThreat(dest.longUrl):
            deactivateMaliciousShortUrl(shortUrl)
            emailOwner(shortUrl)
            throw NotFoundError  # 404 to user, not 451
        urls.update(shortUrl, safeBrowsingExpiry = now + 24h)

    visits = writeShortlinkToCookie(pastVisits, shortUrl)
    type = (isCrawler(userAgent) or fromTrustedReferrer(referrer)
            or hasVisited(pastVisits, shortUrl))
           ? Direct
           : TransitionPage

    return { longUrl: dest.longUrl, visits, type }
```

DB lookup MUST use the replica when `FF_USE_REPLICA_FOR_REDIRECTS=true`, falling back to primary on replica error.

### 10.3 Response

- `RedirectType.Direct`: HTTP 302 with `Location: longUrl`.
- `RedirectType.TransitionPage`: HTTP 200 rendering `transition-page.ejs` with `escapedLongUrl`, `rootDomain` of the destination, and `gaTrackingId`.

Both response paths MUST update `visits` cookie and trigger the side effects in §10.5 and §10.6.

### 10.4 Crawler and Referrer Heuristics

`isCrawler(userAgent)`: parse the user-agent string; if any of the standard fields (browser name, rendering-engine name, OS name) is missing → crawler. Any user agent whose name matches the bot regex `/bot|facebookexternalhit|Facebot|Slackbot|TelegramBot|WhatsApp|Twitterbot|Pinterest|Postman|url|Google-PageRenderer/` MUST be classified as device `'others'` for statistics purposes.

`fromTrustedReferrer(referrer)`: parse the referrer; trusted iff its origin equals `OG_URL`'s origin. Parse failures are treated as untrusted.

### 10.5 Click Statistics Update

After producing the redirect response (but possibly before returning to the client), the server MUST fire-and-forget a click-recording operation that, atomically and idempotently under concurrency, performs the following four updates for the resolved `shortUrl`:

1. Increment the total-clicks counter by 1.
2. Increment the appropriate device-class counter by 1 (`mobile` / `tablet` / `desktop` / `others`), where the class is derived from the user agent (§10.4).
3. Increment the (shortUrl, today's date) counter in the daily aggregate by 1.
4. Increment the (shortUrl, weekday, hour) counter in the weekday/hour aggregate by 1.

All four updates MUST be observed in **Asia/Singapore** local time. The implementation MAY achieve atomicity through a stored procedure with locking, a single transaction with upserts, a single record with multiple counters, or any other mechanism. Errors MUST be caught and logged but MUST NOT block the redirect response.

### 10.6 Web Analytics Pageview

When `GA_TRACKING_ID` (or the equivalent web-analytics property identifier) is configured, the server SHOULD record a pageview event for each non-crawler redirect via the configured web-analytics service. The event SHOULD carry the short URL, long URL, and a stable visitor identifier propagated through a cookie. Crawlers MUST be excluded. The web-analytics integration is optional; absence of it MUST NOT change redirect behavior.

### 10.7 Cookie Eviction

The `visits` cookie holds an array of recent short URLs. Visiting a known short URL MUST move it to the end; unknown short URLs MUST be appended. When the serialized cookie size exceeds `COOKIE_SESSION_MAX_SIZE_BYTES` (default 2000), entries MUST be removed from the head (LRU).

---

## 11. File Hosting

Files attached to short URLs are stored in the object store under the bucket/container identified by `AWS_S3_BUCKET`:

- Object key: `${shortUrl}.${ext}`.
- ACL: public-read when `state = ACTIVE`, private when `state = INACTIVE`. Cache-Control on uploaded objects is `no-cache`.
- `longUrl` for a file URL MUST be constructed as `${fileURLPrefix}${AWS_S3_BUCKET}/${key}`.
  - In production, `fileURLPrefix = 'https://'`, so the URL is `https://${AWS_S3_BUCKET}/${shortUrl}.${ext}`. The bucket name is conventionally also the public hostname (e.g., `file.go.gov.sg`), making the canonical file URL `https://${FILE_HOSTNAME}/${shortUrl}.${ext}`. **External integrators and stored history depend on this URL shape; see §27.3.**
  - In development, `fileURLPrefix` is the local object-store emulator's `ACCESS_ENDPOINT` followed by `/`.
- The reverse derivation `getKeyFromLongUrl(longUrl)` MUST extract the key as the final path segment.
- Files MUST pass the extension/MIME and antivirus checks of §12.2 before upload.
- Replacing a file (on edit) MUST overwrite the same key. The system MUST NOT permit changing whether a URL is a file URL after creation, nor changing its `longUrl` independently of the underlying object.

For development, a local object-store emulator MAY be exposed at `BUCKET_ENDPOINT`.

---

## 12. Threat Detection

### 12.1 URL Threat Scanning

The server integrates with an external **URL threat-scanning service** that, given a URL, returns whether the URL is known to host malware, social-engineering content, unwanted software, or related threats. The choice of provider is implementation-defined; the configuration uses `SAFE_BROWSING_KEY` as a generic credential. The integration MUST cover the conceptual threat classes:

- malware
- social engineering / phishing
- unwanted software
- extended-coverage social engineering (lower-confidence phishing detection)

#### 12.1.1 Cache

Threat-scan results MUST be cached by URL with a short TTL (default 300 s). Cache hits return the cached threat verdict; cache misses query the external service. The cache is logically separate from the redirect cache and other caches.

Additionally, the *clean* verdict MUST be cached on the `Url` itself via `safeBrowsingExpiry` (default 24 h) so that the redirect path (§10.2) avoids re-scanning on every hit.

#### 12.1.2 When the service is unconfigured

If no threat-scanning credential is configured, the server MUST log a warning at startup and treat all URLs as clean. Threat scanning is OPTIONAL infrastructure.

#### 12.1.3 Log-only mode

When `SAFE_BROWSING_LOG_ONLY=true`, a detected threat MUST be logged and the `MALICIOUS_ACTIVITY_LINK` counter incremented, but the verdict returned to callers MUST be "clean". This mode permits dry-run deployment of the scanner.

#### 12.1.4 Bulk scan

A bulk scan over an array of URLs MUST evaluate each URL (concurrency permitted) and return true if any URL is a threat. Implementations MAY short-circuit on the first positive.

#### 12.1.5 At redirect

§10.2 specifies that an expired `safeBrowsingExpiry` triggers a re-scan and that a threat-positive scan MUST:

- Deactivate the link (`state = INACTIVE`).
- Email the owner.
- Return 404 to the caller (parity with not-found, avoiding leakage).

### 12.2 File Threat Scanning

The server integrates with an external **antivirus service** that, given a file's bytes, returns whether the file contains a virus or is password-protected. The choice of provider is implementation-defined; the configuration uses `CLOUDMERSIVE_KEY` as a generic credential. The scanner SHOULD be configured to refuse executables, scripts, and structurally invalid files.

File scan pipeline:

1. If no antivirus credential is configured, skip the scan.
2. Call the antivirus service. On error, emit `SCAN_FAILED_FILE` and return 500 `"Your file could not be scanned at this moment…"`.
3. If the file is password-protected, return 400 `"Cannot upload password-protected files."`.
4. If the file contains a virus, emit `MALICIOUS_ACTIVITY_FILE` and return 400 `"File is likely to be malicious."`.

### 12.3 File Extension/MIME Validation

Before invoking the antivirus service, the server MUST:

1. Detect the file's MIME type from its byte content (not from the upload's declared header); fall back to the filename extension; apply the manual overrides in §7.6.
2. Reject the upload with 415 `"File type disallowed."` if the resolved extension is empty or not in the allowlist (default §7.6).
3. Stamp the resolved MIME on the in-flight upload record so downstream consumers (object-store upload, antivirus call) see the canonical type.

---

## 13. Bulk Operations and Async Jobs

### 13.1 Bulk Upload (`POST /api/user/url/bulk`)

Multipart upload. File size ≤ `MAX_CSV_UPLOAD_SIZE` (5 MiB). Optional `tags` JSON.

Pipeline:

1. Multipart upload acceptance with `MAX_CSV_UPLOAD_SIZE` limit.
2. File extension/MIME validation restricted to `csv`.
3. File antivirus scan.
4. Parse and validate the CSV per §13.1.1–§13.1.2.
5. URL threat-scan on every parsed URL (bulk variant; §12.1.4).
6. Generate short URLs and persist the bulk URL mappings.
7. If `ACTIVATE_BULK_QR_CODE_GENERATION === 'true'`, dispatch the async-job pipeline (§13.2).

#### 13.1.1 CSV format

- Header row MUST be exactly `BULK_UPLOAD_HEADER = "Original links to be shortened"`.
- One column per row.
- Empty rows skipped.
- Rows ≤ `BULK_UPLOAD_MAX_NUM` (default 1000).

#### 13.1.2 Row-level validation

Apply in order. Any failure aborts the upload with HTTP 400 and `MessageType.FileUploadError`. Each failure emits `BULK_VALIDATION_ERROR` with the corresponding tag:

| Check | Tag | Message |
|-------|-----|---------|
| `rows ≤ BULK_UPLOAD_MAX_NUM` | `acceptableLinkCount` | `"File exceeded {N} original URLs to shorten"` |
| `header == BULK_UPLOAD_HEADER` | `validHeader` | `"Row 1: bulk upload header is invalid"` |
| Exactly 1 column | `onlyOneColumn` | `"Row {N}: {row} contains more than one column of data"` |
| Non-empty | `isNotEmpty` | `"Row {N} is empty"` |
| `isValidUrl(row)` | `isValidUrl` | `"Row {N}: {url} is not valid"` |
| `not isBlacklisted(row)` | `isNotBlacklisted` | `"Row {N}: {url} is blacklisted"` |
| `not isCircularRedirects(row, OG_URL.hostname)` | `isNotCircularRedirect` | `"Row {N}: {url} redirects back to {host}"` |
| No CSV-parser error | `noParsingError` | `"Parsing error"` |

#### 13.1.3 Short URL generation

For each accepted long URL, call `generateShortUrl(BULK_UPLOAD_RANDOM_STR_LENGTH)`. Collision detection runs inside `UrlManagementService.bulkCreate`.

#### 13.1.4 Persistence

`UrlManagementService.bulkCreate({ userId, urlMappings, tags })`:

- Wrap in transaction.
- Insert `urls` rows with `source = BULK`, `state = ACTIVE`, `safeBrowsingExpiry = now + 24h`.
- Apply the supplied tags to all rows.
- The `afterBulkCreate` hook MUST write `url_history` rows and create `url_clicks` rows.

#### 13.1.5 Response

```
200 { count: number, job?: Job }
```

`job` is present iff QR generation was enabled.

### 13.2 Job lifecycle

When the bulk pipeline produces a job:

1. Create a `Job` record with status `IN_PROGRESS` and a fresh UUID.
2. Chunk `urlMappings` into batches of size `BULK_QR_CODE_BATCH_SIZE` (default 1000).
3. For each batch with index `i`:
   - Create a `JobItem` with `jobItemId = "${job.uuid}/${i}"`, status `IN_PROGRESS`, empty `message`, and `params = { jobItemId, mappings }`.
   - Enqueue the same `params` payload onto the async job queue (§3.1). The queue MUST deliver the message to a background worker at least once; duplicate delivery MUST be tolerated by the worker (the artifact key is deterministic on `jobItemId`).

Emit `JOB_START_SUCCESS` or `JOB_START_FAILURE` per item.

### 13.3 Job status aggregation

```
function computeJobStatus(items):
    if any item.status == FAILURE: return FAILURE
    if any item.status == IN_PROGRESS: return IN_PROGRESS
    return SUCCESS
```

When a job transitions out of `IN_PROGRESS`, the server MUST send a completion email to the owner. Emit `JOB_EMAIL_SUCCESS`/`JOB_EMAIL_FAILURE`.

### 13.4 Job callback (`POST /api/callback/qr`)

Admin API-key authenticated.

Request body:

```
{ userId: number, jobItemId: string, status: { isSuccess: boolean, errorMessage?: string } }
```

Server:

1. `JobManagementService.updateJobItemStatus(jobItemId, status)`. Emit `JOB_ITEM_UPDATE_*`.
2. `updateJobStatus(jobId)` — recompute parent status and persist. Emit `JOB_UPDATE_*`.
3. If parent transitioned to a terminal status, send the completion email and include the per-item download URLs `${BULK_QR_CODE_BUCKET_URL}/${jobItemId}`.

### 13.5 Job status polling

**`GET /api/user/job/status?jobId={id}`** (session) — long poll:

```
for attempt in 1..JOB_POLL_ATTEMPTS:
    job = repository.findJobForUser(userId, jobId)
    if job is null: return 404
    if job.status != IN_PROGRESS: return 200 { job, jobItemUrls }
    sleep JOB_POLL_INTERVAL ms
return 408
```

`jobItemUrls = job.items.map(i => `${BULK_QR_CODE_BUCKET_URL}/${i.jobItemId}`)`.

**`GET /api/user/job/latest`** returns the latest job for the user without long-polling.

---

## 14. QR Code Generation

### 14.1 Single-URL Endpoint (`GET /api/qrcode`)

Query parameters:

```
url:    string (required, valid short URL)
format: 'image/svg+xml' | 'image/png' | 'image/jpeg' (required)
```

Behavior:

1. Look up the short URL. If absent, return 400 `"Short link does not exist"`.
2. Construct the full URL `${OG_URL}/${shortUrl}`.
3. Render with the `qrcode` library:
   - SVG output, error correction level `H`, margin 0.
   - Dark color from the deployment's brand color (§14.3).
4. Compose onto a 1000-pixel-wide canvas:
   - 85 px top margin.
   - 800×800 QR centered.
   - A configurable brand logo SVG overlaid at the center of the QR.
   - 85 px between QR and text.
   - Text: the human-readable short link (e.g., `go.gov.sg/foo`) in IBM Plex Sans 32 px, line height 1.35, anchor middle, wrapped every 36 characters.
   - 85 px bottom margin after final line.
5. If `format` is `image/png` or `image/jpeg`, rasterize the rendered canvas to the target format.
6. Respond with the correct `Content-Type` and header `Filename: ${OG_URL_HOST}/${shortUrl}`. Body is the binary buffer.

### 14.2 Bulk QR Pipeline

Enabled iff `ACTIVATE_BULK_QR_CODE_GENERATION === 'true'`. See §13.2 for the dispatch protocol and Appendix A for the background-worker contract.

For each `jobItemId = "${job.uuid}/${i}"`, the worker produces three artifacts in the object store under the prefix `${jobItemId}/`:

- `${jobItemId}/generated.csv` — header `"Short URL,Original URL"`, one row per mapping.
- `${jobItemId}/generated_svg.zip` — `${shortUrl}.svg` per mapping.
- `${jobItemId}/generated_png.zip` — `${shortUrl}.png` per mapping.

### 14.3 Color and Logo

QR codes MUST be rendered with a single brand dark color and a single brand logo, both implementation-defined. The chosen color and logo are deployment-wide constants. Both the synchronous QR-code endpoint (§14.1) and the bulk-generation worker (§14.2, Appendix A.3) MUST use the same color and logo so that a single QR rendered ad hoc is visually indistinguishable from one rendered as part of a bulk job.

---

## 15. Statistics and Analytics

### 15.1 Global Statistics (`GET /api/stats`)

Public, unauthenticated. Returns static counters from environment:

```
{ userCount: USER_COUNT, clickCount: CLICK_COUNT, linkCount: LINK_COUNT }
```

These values do not refresh dynamically; they are updated by redeploy.

### 15.2 Per-Link Statistics (`GET /api/link-stats`)

Session-authenticated. Query:

```
url:    string (required)
offset?: int (default 6) -- number of days back to include in daily breakdown
```

Server:

1. Authorize: user MUST own the link.
2. Read (preferably from a read replica or other read-optimised path):
   - The total-clicks counter.
   - The device-class counters.
   - The daily-clicks aggregates between `today − offset` and `today` (Asia/Singapore).
   - All 24×7 weekday/hour buckets.
3. If all device counters are zero AND no other aggregates exist, return null and 404.
4. Return:

```
{
  totalClicks: number,
  deviceClicks: { desktop, tablet, mobile, others },
  dailyClicks: [{ date, clicks }],
  weekdayClicks: [{ weekday, hours, clicks }]
}
```

### 15.3 Click Aggregation

Per §10.5, every successful redirect MUST invoke the click-recording operation. That operation MUST be idempotent and safe under concurrent invocation for the same short URL.

### 15.4 Web Analytics Hits

§10.6. When web analytics is configured, the server MAY assign a stable visitor identifier via a cookie and propagate it across redirects; absence of the cookie SHOULD cause a fresh identifier to be issued.

---

## 16. Audit Trail (Link History)

### 16.1 Endpoint (`GET /api/link-audit`)

Session-authenticated. Query:

```
url:    string (required, short URL)
offset?: int (default 0)
limit?:  int (default 10)
```

The user MUST own the link; otherwise return 404 `"User does not own this short url"`.

### 16.2 Algorithm

1. Fetch the last `limit + 1 + offset` `url_history` rows ordered by `updatedAt DESC`.
2. Compute change sets pairwise:
   - For each adjacent (newer, older) pair, emit one change set per differing field among `state`, `userEmail`, `longUrl`, `tagStrings` with `{ type: 'update', key, prevValue: older[key], currValue: newer[key], updatedAt: newer.updatedAt }`.
3. If the oldest fetched row is the creation row (`offset + limit ≥ totalCount`), emit `{ type: 'create', key: 'longUrl', prevValue: null, currValue: creationRow.longUrl, updatedAt: creationRow.createdAt }`.
4. Return:

```
{ changes: LinkChangeSet[], limit, offset, totalCount }
```

Change sets MUST be ordered newest first.

---

## 17. Public Directory Search

### 17.1 Endpoint (`GET /api/directory/search`)

Public; **does** require a session in the current implementation. Query:

```
query:   string (required)
order:   'RECENCY' | 'POPULARITY' (required)
limit?:  int (capped at 100)
offset?: int (default 0)
state?:  'ACTIVE' | 'INACTIVE'
isFile?: 'true' | 'false' | '' (empty == any)
isEmail: 'true' | 'false' (required)
```

### 17.2 Query Modes

- **Email mode** (`isEmail === 'true'`): the `query` is interpreted as an email or email substring. The query MUST be reduced to the part after `@` if present. The server performs a case-insensitive substring match against `User.email`. Emit `DIRECTORY_SEARCH_EMAIL`.
- **Text mode** (`isEmail === 'false'`): `@` MUST be stripped from the query to avoid leaking email-domain enumeration through this path. The server performs a full-text search against `Url` fields using English-language tokenization, with weighted relevance: `shortUrl` matches rank highest, `longUrl` matches medium, `description` matches lowest. The exact weight values are implementation-defined; what matters is the *ordering*. Emit `DIRECTORY_SEARCH_DOMAIN`.

### 17.3 Sort

- `POPULARITY` → results ordered by total-clicks aggregate (descending).
- `RECENCY` → results ordered by `Url.createdAt` (descending).

### 17.4 Response

```
{
  urls: [{ shortUrl: string, email: string, state: 'ACTIVE'|'INACTIVE', isFile: boolean }],
  count: number
}
```

---

## 18. Client Application

The browser client is a single-page application served as a static bundle. Framework choice (React, Vue, Svelte, etc.), state-management approach, routing strategy (hash, history, path-based), and UI toolkit are implementation-defined. All client behaviors described here MUST be reproducible regardless of visual styling. State-shape descriptions in this section are derived from the current implementation; they document the *information* the client holds, not a required data structure.

### 18.1 Routes

| Path | Subapp | Access | Behavior |
|------|--------|--------|----------|
| `/` | home | anonymous | If logged in, redirect to `/user`. Otherwise render landing. |
| `/login` | login | anonymous | If logged in, redirect to `/user`. |
| `/user` | user dashboard | authenticated | Private route; if anonymous, redirect to `/login` and remember the originally-requested location. |
| `/directory` | directory | public | Public search. |
| `/apiintegration` | API integration | authenticated | Private route. |
| `/404/:shortUrl` | 404 | public | "Link not found" page. |

Private routes MUST be guarded so anonymous users are redirected to `/login`. After a successful OTP verification, the login subapp MUST navigate back to the originally-requested location if one was remembered, else `/user`.

### 18.2 Global Layout

Across all subapps, the client MUST:

- On every page mount, fire a web-analytics page-view event when web analytics is configured. Page titles are implementation-defined; the existing app uses labels like `HOME PAGE`, `EMAIL LOGIN PAGE`, `OTP LOGIN PAGE`, `USER PAGE`, `CREATE LINK PAGE`, `DIRECTORY PAGE`, `API INTEGRATION`.
- Render a global notification surface (toast/snackbar) capable of displaying transient error, success, and info messages, with a programmatic API to enqueue and dismiss them.
- Send all API requests with credentials included and same-origin mode. The HTTP client implementation is unspecified.
- Request a `LOGIN_MESSAGE` banner on the login page and a `USER_MESSAGE` banner on the dashboard, displaying each when non-empty.

### 18.3 Home Subapp

Information held: optional rotating-links list and `{ userCount, linkCount, clickCount }`.

On mount:

1. `GET /api/login/isLoggedIn` — if 200, navigate to `/user`.
2. `GET /api/links` — server returns rotating-link payload (string sourced from `ROTATED_LINKS`).
3. `GET /api/stats` — populate the statistics counters on the landing page.

### 18.4 Login Subapp

Information held: the email being entered, an email validator function, the currently authenticated user (if any), and a form-state value drawn from the set:

```
EMAIL_READY | EMAIL_PENDING | OTP_READY | OTP_PENDING | RESEND_OTP_DISABLED
```

State machine:

```
EMAIL_READY --submit-> EMAIL_PENDING --resp.ok-> OTP_READY
                                  --resp.err-> EMAIL_READY (error toast)
OTP_READY   --submit-> OTP_PENDING   --resp.ok-> logged-in (navigate to /user)
                                  --resp.err-> OTP_READY (error toast)
OTP_READY   --resend-> EMAIL_PENDING -> OTP_READY -> RESEND_OTP_DISABLED (20 s) -> OTP_READY
```

Bootstrap fetches:

1. `GET /api/login/isLoggedIn`. If 200, mark the session authenticated; if 404, mark it anonymous.
2. `GET /api/login/emaildomains` — server returns the glob string. The client constructs an email validator that combines a glob match against this string with a structural email check (per §7.3).

Validation:

- Email input lowercases on every keystroke.
- Email error message: `"This doesn't look like a valid ${domain} email."` shown only when the field has a value and fails the validator.
- OTP input is not validated client-side beyond non-emptiness.

When real-user observability is configured, the client SHOULD identify the authenticated user to it on both bootstrap (existing session) and after successful OTP verification.

### 18.5 User Dashboard Subapp

Information held by the dashboard:

- Whether initial data has been loaded; whether a fetch is in flight.
- The current page of URLs (`StorableUrl[]`) and the total count.
- Form inputs for creating a new short URL: short URL, long URL, tags.
- Whether the create-URL modal is open.
- A table configuration value with the following dimensions:

```
isTag:           boolean        // is the active filter a tag filter?
numberOfRows:    number         // page size; default 10
pageNumber:      number         // 0-indexed
searchText:      string         // user search input
searchInput:     string         // debounced 500 ms version of searchText
tags:            string         // semicolon-separated tag filter
filter:          { isFile?: boolean, state?: 'ACTIVE'|'INACTIVE' }
orderBy:         string         // default 'createdAt'
sortDirection:   'asc' | 'desc'
```

- The user-message banner and announcement-modal contents.
- Upload-success and upload-error flags for URL/file creation.
- Whether a file upload is in flight.
- The most recently created short URL (for surfacing in the UI).
- The current page of link-history change sets and its total count.
- A status-bar message with a header, body, severity (`SUCCESS`, `ERROR`, `INFO`), and a list of related download URLs.

#### 18.5.1 Initial Load

On mount, in order:

1. Fetch `GET /api/user/url?{tableConfig}` and store `{ urls, count }`.
2. If the email validator is not cached, fetch `GET /api/login/emaildomains` and build it.
3. Fetch `GET /api/user/message` to populate the banner.
4. Fetch `GET /api/user/announcement` to populate the announcement modal.
5. Fetch `GET /api/user/job/latest` to determine whether a status bar should be displayed.

If the total URL count is zero and no filters are active, render an empty state with a "Create link" CTA. Otherwise render the URL table.

#### 18.5.2 Create Link Modal

Three tabs:

- **URL**: fields `shortUrl`, `longUrl`, optional `tags`. On submit: `POST /api/user/url` JSON.
- **File**: fields `shortUrl`, file picker, optional `tags`. On submit: `POST /api/user/url` multipart.
- **Bulk**: file picker (CSV ≤ 5 MiB), optional `tags`. On submit: `POST /api/user/url/bulk` multipart.

The modal MUST:

- Strip leading `https://` from the long URL field for display and prepend it back before sending.
- Validate `shortUrl` against `/^[a-z0-9-]+$/`.
- Validate tags via `isValidTags`.
- Show error toasts categorized by `MessageType` returned in the response body.
- On bulk success, dispatch `getLatestJob()` so the status bar reflects the new job.

#### 18.5.3 Link Drawer

Opens by row click. Local context state:

```
controlPanelIsOpen:    boolean
relevantShortLink:     string | null
qrCodeModalIsOpen:     boolean
uploadFileError:       string | null
linkHistoryIsActive:   boolean
```

Drawer features (each maps to an existing API):

| Feature | Backend |
|---------|---------|
| Edit long URL | `PATCH /api/user/url { shortUrl, longUrl }` |
| Edit description + contact email | `PATCH /api/user/url { shortUrl, description, contactEmail }` |
| Edit tags | `PATCH /api/user/url { shortUrl, tags }` |
| Replace file | `PATCH /api/user/url` multipart |
| Toggle ACTIVE/INACTIVE | `PATCH /api/user/url { shortUrl, state }` (optimistic via `TOGGLE_URL_STATE_SUCCESS`) |
| Transfer ownership | `PATCH /api/user/url/ownership { shortUrl, newUserEmail }` |
| View link history | `GET /api/link-audit?url=&offset=&limit=` |
| View statistics | `GET /api/link-stats?url=&offset=` |
| Generate QR code | `GET /api/qrcode?url=&format=` |

#### 18.5.4 Bulk QR Status Bar

When a bulk job exists, the status bar polls `GET /api/user/job/status?jobId=` with exponential backoff. Rendering rules:

- `IN_PROGRESS` → INFO variant: `"QR codes generation in progress. Please wait to download your QR codes"`.
- `SUCCESS` → SUCCESS variant: `"QR codes successfully generated. Please download your QR codes here or via email"`. `callbacks` contains the per-format download URLs constructed from `BULK_QR_CODE_BUCKET_URL`.
- `FAILURE` → ERROR variant: `"QR codes failed to generate. Please try again"`.

#### 18.5.5 Announcement Modal

If `state.user.announcement` is truthy, the client MUST render a modal at first dashboard mount displaying the configured `title`, `subtitle`, `message`, `image`, and a CTA button labeled `buttonText` linking to `url`. The modal MUST be dismissible.

### 18.6 Directory Subapp

State (`state.directory`):

```
results:        UrlTypePublic[]
resultsCount:   number
queryForResult: string | null
```

Behavior:

- The query input is debounced 500 ms. Each commit updates the URL search string (`/directory?query=&order=&rowsPerPage=&currentPage=&state=&isFile=&isEmail=`).
- On any change to the URL search string, the client MUST refetch `GET /api/directory/search`.
- Query parameters MUST be transformed:
  - `query.toLowerCase().trim()`.
  - If `isEmail === 'true'`, retain only the substring after `@`.
  - Else strip `@` characters entirely.
- Filter "Reset" clears all filters but preserves the query and `isEmail` mode and resets pagination to page 0.
- Pagination changes (rows-per-page or page) re-issue the GET with updated parameters; rows-per-page change MUST reset page to 0.

### 18.7 API Integration Subapp

Information held: whether the current user has an API key, whether the key-display modal is open, and the most recently generated API key (transient — held only long enough to show it to the user).

On mount: `GET /api/user/hasApiKey` to populate the "has key" flag.

The "Generate API Key" action calls `POST /api/user/apiKey` (no body) and opens the key-display modal with the returned key. The full key is shown exactly once; the client MUST NOT persist it in long-lived storage.

### 18.8 HTTP Client

The client makes HTTP requests using any library or built-in mechanism. All requests MUST include credentials (so the session cookie is sent) and use same-origin mode. JSON requests MUST send `Content-Type: application/json`; multipart uploads MUST omit `Content-Type` so the runtime can set the boundary automatically.

### 18.9 Internationalization

The client loads a single English locale bundle at startup. The bundle's location and loading mechanism are implementation-defined (the existing implementation places it under `/locales/en/...` and loads it via an i18n library). HTML escaping MUST be performed by the rendering layer, not by the i18n library, so translation values may contain inline HTML safely. The locale schema is in Appendix D.

---

## 19. External REST API (v1)

Gated by `FF_EXTERNAL_API === 'true'`. Mounted under `/api/v1`. Authenticated by API key (§8.2).

### 19.1 User-scope

| Route | Body / Query |
|-------|--------------|
| `GET /api/v1/urls` | Same query as `/api/user/url` minus tag filtering. Returns `UrlsPaginated`. |
| `POST /api/v1/urls` | `{ longUrl (required, HTTPS), shortUrl? (auto-gen length `API_LINK_RANDOM_STR_LENGTH`) }`. Returns mapped `StorableUrl`. `source = API`. |
| `PATCH /api/v1/urls/:shortUrl` | `{ longUrl?, state? }`. File editing NOT allowed via API. |

API responses use a thin DTO mapping that omits internal fields (e.g., `safeBrowsingExpiry`, `userId`).

### 19.2 Admin-scope

Mounted under `/api/v1/admin/`. Additional middleware: `apiKeyAdminAuthMiddleware`.

| Route | Body |
|-------|------|
| `POST /api/v1/admin/urls` | `{ email (target user), longUrl, shortUrl? }`. Server: `findOrCreateWithEmail(email)`; create URL under the target user. If `userId !== targetUser.id`, transfer ownership immediately. |

### 19.3 QR-job callback

`POST /api/callback/qr` (admin-scope; §13.4).

---

## 20. Email Delivery

A single mailer abstraction sends:

- OTP emails (login). Subject and body include the OTP and requester IP.
- Job-completion emails (bulk QR). Body includes the per-item artifact URLs.
- Ownership-transfer notifications.
- Malicious-link deactivation notices.

The email transport is implementation-defined: any mechanism capable of delivering plain-text or HTML mail to arbitrary recipients is acceptable (transactional email provider, on-prem SMTP, third-party email API). The implementation MAY define a fallback transport that is attempted when the primary transport fails.

Bounce/complaint capture, if supported by the chosen transport, MAY be performed out-of-band by an auxiliary worker that listens for delivery-status notifications (Appendix A.4).

---

## 21. Persistence and Caching

### 21.1 Primary durable store

A primary durable store backs the entities of §4. Required behaviors:

- Transactional writes spanning multiple entities (used for URL creation + history + initial clicks-counter row).
- Secondary indexes sufficient to support the access patterns described in this document, including descending order by total clicks (popularity sort) and a weighted full-text index over (`shortUrl`, `longUrl`, `description`) for directory search.
- Asia/Singapore time handling for time-dimensioned aggregates.
- Bulk mutations on `Url` MUST be forbidden so every change passes through the URL-history write (§4.4). Implementations MAY enforce this in the data layer, in a service layer, or both.
- URL create/update/transfer MUST atomically write a `UrlHistory` record (§4.4). A `Url` create MUST atomically seed the total-clicks aggregate.

A read replica is OPTIONAL. When `FF_USE_REPLICA_FOR_REDIRECTS` is true, the redirect lookup MAY consult the replica; on replica error the primary MUST be queried. All reads outside the redirect path SHOULD prefer the primary unless eventual consistency is explicitly acceptable (directory search, statistics).

### 21.2 Caches

The five logically separate caches of §3.1 have the following key spaces and TTLs:

| Purpose | Key | Value | TTL |
|---------|-----|-------|-----|
| OTP | `${email}:${ip}` | `{ hashedOtp, retries }` | `OTP_EXPIRY` (default 300 s) |
| Session | session-token identifier | serialized session | `COOKIE_MAX_AGE` (touched on access) |
| Redirect | lowercased `shortUrl` | `{ longUrl, isFile, safeBrowsingExpiry }` | `REDIRECT_EXPIRY` (default 300 s) |
| Statistics | implementation-defined | implementation-defined | implementation-defined |
| URL threat-scan | full URL | `{ threatTypes, expireTime }` | 300 s |

The redirect cache MUST be invalidated on `Url` update. Implementations MAY use any cache technology (in-memory, key-value store, distributed cache). Caches MAY share an underlying storage technology so long as their key spaces and eviction policies remain independent.

### 21.3 Object store

- File-upload bucket: `AWS_S3_BUCKET`. Visibility toggled by `state` (§11). Object naming, ACL semantics, and storage-class choices are implementation-defined.
- QR bulk output bucket: `BULK_GENERATION_BUCKET`. Public-readable. The bucket key for each artifact MUST be `${jobItemId}/${filename}` so the server-side polling endpoint can construct download URLs deterministically.

---

## 22. Observability, Security, and Operations

### 22.1 Logging

Server logs MUST be structured (key-value or JSON) so that downstream log pipelines can index them. Log level SHOULD be configurable (e.g., `debug` in development, `info` in production). Choice of logging library is implementation-defined.

HTTP request logging SHOULD record at minimum: method, path, HTTP version, status code, response size, referrer, user agent, response time, and the following extracted fields:

- **Client IP**, derived from a forwarding header (e.g., `CF-Connecting-IP`, `X-Forwarded-For`) if present, otherwise the connection-level remote address.
- **Redirect URL**: the `Location` header on 3xx responses.
- **User ID**: the authenticated session's user id, or empty.

### 22.2 Tracing and APM

Application performance monitoring is OPTIONAL. When configured, the server SHOULD propagate trace context across HTTP boundaries, async-job dispatch, and the email transport, and SHOULD tag spans with the route name. Identification fields (`DD_SERVICE`, `DD_ENV`, or their analogues in the chosen platform) are implementation-defined.

### 22.3 Real-user observability (client)

Browser-side real-user monitoring is OPTIONAL. When configured, the client SHOULD identify the authenticated user to the platform on login.

### 22.4 Metrics

The server emits the metrics listed in Appendix C. The metrics MUST be named exactly as listed (the names form a stable observability contract). They MAY be emitted to any platform (StatsD, Prometheus, OpenTelemetry, vendor-specific). Tags on metrics MUST be preserved.

### 22.5 HTTP security headers

The server MUST emit a Content-Security-Policy header restricting the origins from which the client may load scripts, styles, fonts, images, and connect targets. A conforming default-deny policy is:

```
default-src 'self';
style-src   'self' 'unsafe-inline' <web-font CDN>;
font-src    'self' <web-font CDN>;
img-src     'self' data: <web-analytics origin> <file-hosting origin>;
script-src  'self' <web-analytics origin> <real-user-monitoring agent origin>;
worker-src  blob:;
connect-src 'self' <web-analytics origin> <real-user-monitoring intake origin> [+ CSP_REPORT_URI if set];
frame-ancestors 'self';
upgrade-insecure-requests;
```

Specific allow-listed origins depend on which external services (web analytics, real-user monitoring, file hosting, web-font provider) are configured.

When `CSP_ONLY_REPORT_VIOLATIONS=true` the header MUST be sent as `Content-Security-Policy-Report-Only` instead.

Additional headers (HSTS, X-Content-Type-Options, X-Frame-Options or equivalent) SHOULD be set per current web-security best practice; the existing implementation derives most of them from a helmet-style middleware.

All responses MUST set `Cache-Control: no-store`.

### 22.6 Rate Limiting

`POST /api/login/otp` MUST be rate-limited per client IP. Default window 60 s; default cap `OTP_RATE_LIMIT` requests per window (0 disables; default 0 in development). Overflowing requests MUST receive HTTP 429. Rate-limit overflow events SHOULD be logged at warning level.

No other endpoints are rate-limited.

### 22.7 Server timeouts

The server MUST configure HTTP keep-alive and headers timeouts that are compatible with whichever load balancer fronts it (e.g., an L7 load balancer with a 100-second idle timeout requires the server's keep-alive to be lower than that). The existing deployment uses `keepAliveTimeout = 65 s` and `headersTimeout = 66 s`; a rewrite MAY pick any values that satisfy the load-balancer constraint.

### 22.8 Error handler and not-found fallbacks

The server MUST translate the following error classes into HTTP responses:

- A schema-validation error from request input → HTTP 400 with the validation message in `{ message }`.
- A malformed JSON request body → HTTP 400 with `{ message: 'Bad Request. JSON is malformed' }`.
- Any other unhandled exception → HTTP 500 with an error page or JSON body. The response body format is implementation-defined.

There are three not-found surfaces. Only one is part of the external contract (§27):

1. **External REST API 404 (`/api/v1/...`)** — JSON body, e.g., `{ message: 'Resource not found.' }`. Integrators rely on the JSON content type; the body schema is implementation-defined.
2. **Short-URL 404 (`GET /:shortUrl`)** — HTTP 404 from the redirect endpoint. The current implementation renders an HTML "link not found" page; a rewrite may render any HTML or JSON 404, so long as the status code is 404.
3. **SPA / internal API 404** — any other path. The current implementation renders the same HTML template as surface 2; a rewrite may handle this however it likes.

The process MUST install a global handler for unhandled promise rejections (or the equivalent in the chosen runtime) and emit the `ERROR_UNHANDLED_REJECTION` metric when one occurs.

### 22.9 Health checks

The server SHOULD expose a way for the surrounding infrastructure to determine its liveness. The mechanism (dedicated HTTP endpoint, TCP probe, process exit code) is implementation-defined.

---

## 23. Failure Model and Recovery

### 23.1 Custom error classes

| Class | HTTP | Source |
|-------|------|--------|
| `NotFoundError` | 404 | Missing URL/user/job. |
| `AlreadyExistsError` | 400 (`MessageType.ShortUrlError`) | Short URL collision on create. |
| `AlreadyOwnLinkError` | 400 | Ownership transfer to current owner. |
| `InvalidOtpError` | 401 | OTP mismatch (carries retries-left). |
| `InvalidUrlUpdateError` | 400 | Disallowed update (e.g., file → URL). |

### 23.2 Standard response

```
{ ok?: boolean, message: string, type?: MessageType }
```

`MessageType` ∈ `{ 'ShortUrlError', 'LongUrlError', 'FileUploadError' }`.

### 23.3 Status codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 302 | Redirect |
| 400 | Validation failure, malicious content, conflict |
| 401 | Missing/invalid OTP, session, or API key |
| 403 | Owner mismatch |
| 404 | Resource not found |
| 408 | Job long-poll exhausted |
| 415 | File extension/MIME disallowed |
| 422 | Multiple files in single-file endpoint |
| 429 | OTP rate limit |
| 500 | Mailer/db/scan/service failure |

### 23.4 Partial-state recovery

- Bulk creates run inside a single transaction; failures roll back fully. The CSV is not partially applied.
- Job items are independent: a failed item MUST mark its parent job FAILURE on the next aggregation, but other items may have already produced object-store artifacts. The completion email MUST report the partial state.
- A redirect-time Safe Browsing failure (not threat-positive — the scan itself errored) MUST NOT block the redirect when `SAFE_BROWSING_LOG_ONLY=true`. Otherwise it MAY fail closed; implementation-defined.

---

## 24. Reference Algorithms

### 24.1 Login OTP

```
on POST /api/login/otp:
    rate_limit(ip)
    validate(email)
    otp = random6digit()
    hash = bcrypt(otp, SALT_ROUNDS)
    redis_otp.set(email+':'+ip, { hashedOtp: hash, retries: 3 }, ex=OTP_EXPIRY)
    mailer.send_otp(email, otp, ip)
    metric(OTP_GENERATE_SUCCESS)
    return 200

on POST /api/login/verify:
    record = redis_otp.get(email+':'+ip)
    if record is None: return 401
    if bcrypt.compare(otp, record.hashedOtp):
        user = users.find_or_create(email)
        session.user = user
        redis_otp.delete(email+':'+ip)
        metric(OTP_VERIFY_SUCCESS)
        return 200 { user }
    record.retries -= 1
    if record.retries > 0:
        redis_otp.set(email+':'+ip, record, ex=remaining_ttl)
        return 401 message="… ${record.retries} attempt(s) remaining."
    redis_otp.delete(email+':'+ip)
    return 401
```

### 24.2 Redirect

```
on GET /:shortUrl:
    if not /^[a-zA-Z0-9-]+$/: return 404
    dest = redis_redirect.get(shortUrl) or db_lookup(shortUrl)
    if dest is None or dest.state == INACTIVE: return 404
    fire_and_forget redis_redirect.set(shortUrl, dest, ex=REDIRECT_EXPIRY)

    if not dest.isFile and (dest.safeBrowsingExpiry is None or expired):
        if threat(dest.longUrl):
            mark_inactive(shortUrl); email_owner(shortUrl); return 404
        urls.update(shortUrl, safeBrowsingExpiry = now + 24h)

    visits = update_cookie(req.cookies.visits, shortUrl)
    direct = isCrawler(ua) or trustedReferrer(ref) or hasVisited(req.cookies.visits, shortUrl)

    fire_and_forget recordClick(shortUrl, deviceClass(ua))
    fire_and_forget ga_pageview(shortUrl, dest.longUrl, ua, ref) unless crawler

    if direct: 302 Location=dest.longUrl
    else:       render transition_page(escape(dest.longUrl), rootDomain, GA_TRACKING_ID)
```

### 24.3 Bulk Create

```
on POST /api/user/url/bulk:
    csv = parse(csvBuffer, headerMustEqual=BULK_UPLOAD_HEADER)
    if csv.errors: 400
    for i, row in enumerate(csv.rows, 1):
        if not validate_row(row, i, OG_URL.host): 400  # per §13.1.2
    longUrls = csv.rows
    if bulk_is_threat(longUrls): 400
    mappings = [(generate_short_url(), u) for u in longUrls]
    db.transaction(): bulk_insert_urls(mappings, tags, userId, source=BULK)
    if ACTIVATE_BULK_QR_CODE_GENERATION:
        job = jobs.create(userId)
        for i, batch in enumerate(chunk(mappings, BULK_QR_CODE_BATCH_SIZE)):
            jobItem = job_items.create(jobId=job.id, jobItemId=f"{job.uuid}/{i}", params={...})
            jobQueue.enqueue(jobItem.params)
        return 200 { count: len(mappings), job }
    return 200 { count: len(mappings) }
```

### 24.4 Job Callback

```
on POST /api/callback/qr (admin API key):
    item = job_items.find(jobItemId)
    item.status = SUCCESS if status.isSuccess else FAILURE
    item.message = status.errorMessage or ''
    job = jobs.find(item.jobId)
    job.status = compute_status(job.items)
    if job.status != IN_PROGRESS:
        mailer.send_job_completion(job)
    return 200
```

---

## 25. Test and Validation Matrix

A conforming implementation MUST cover the following test categories. Choice of test framework, runner, and harness is implementation-defined.

### 25.1 Unit tests

- All request-schema validators reject malformed inputs along the boundaries described in this document.
- Pure validation helpers (URL, short-URL, tag, printable-ASCII, circular-redirect, blacklist) MUST be exercised against canonical positive and negative samples.
- Pure aggregation helpers (`computeJobStatus`, `computeChangeSets`, cookie eviction, device-class classification) MUST be exercised against the spec examples.
- Mappers between domain entities and DTOs MUST be covered.

### 25.2 Integration tests

Runs against a live stack containing the primary durable store, the caches, the object store, and the email transport (or stand-ins). MUST cover:

- Full OTP login round-trip.
- URL create / update / transfer / list.
- Bulk upload with the async-job pipeline mocked or stubbed.
- Redirect path including the URL threat-scan cache.
- Audit endpoint pagination.

### 25.3 End-to-end tests

Runs against a headless browser hitting the deployed (or locally-orchestrated) stack. MUST cover the user stories of §3: login, create URL, edit, transfer, directory filter, transition page, link audit, API integration.

### 25.4 Continuous Integration

Lint + dependency audit + unit + integration + e2e MUST run on every push and pull request. Production deploys are triggered by an explicit release action.

---

## 26. Implementation Checklist

### 26.1 Required for conformance

- [ ] All entities of §4 modelled with the documented attributes, constraints, and relationships.
- [ ] All required env vars of §6.1 validated at startup with fail-fast.
- [ ] Email allowlist enforced as the **intersection** of a glob match against `VALID_EMAIL_GLOB_EXPRESSION` and a structural email check.
- [ ] OTP flow with per-IP rate limit, salted-hash storage, 3-retry semantics, and TTL eviction.
- [ ] Session mechanism with strict same-site cookies.
- [ ] API key flow with version-prefixed key strings and salted-hash suffix storage.
- [ ] Content-Security-Policy header per §22.5 and `Cache-Control: no-store` everywhere.
- [ ] Short URL CRUD with full URL/file/tag validation, automatic history writes, and redirect-cache invalidation.
- [ ] Redirect path with cache, optional replica lookup, URL threat re-scan, transition page, fire-and-forget click recording, web-analytics hit.
- [ ] Bulk upload pipeline per §13 including all eight row-level validators and the BULK_VALIDATION_ERROR metric tags.
- [ ] QR endpoint with the layout and color rules of §14.
- [ ] Link statistics endpoint with read-optimised access (replica reads when configured).
- [ ] Link audit endpoint with the change-set algorithm of §16.2.
- [ ] Directory search with weighted ranking and the `isEmail` mode rules of §17.2.
- [ ] External and admin v1 API gated by `FF_EXTERNAL_API`.
- [ ] All five error classes and the response shape of §23.2.

### 26.2 Recommended extensions

- [ ] Tracing, real-user observability, and metrics wired to the chosen observability platform.
- [ ] One-command local development environment that stands up the durable store, caches, object store, email transport, and async-job queue.
- [ ] Out-of-process workers for link migration, email-event capture, and bulk QR generation per Appendix A.

### 26.3 Operational validation

- [ ] OTP delivery verified end-to-end against a real transport in staging.
- [ ] Replica failover behavior measured.
- [ ] Bulk QR job processes a 1000-row CSV within the worker's wall-clock budget.
- [ ] URL threat-scan outage soft-fails (or fails closed) per the configured `SAFE_BROWSING_LOG_ONLY` policy.

---

## 27. Backward Compatibility (External Surfaces Only)

### 27.1 Scope

GoGovSG is a long-lived public service, but only a small subset of its HTTP surface is consumed by third parties. A reimplementation MAY freely change everything *except* the surfaces that exist outside the system's own client. This section enumerates the externally-binding surfaces; **anything not listed here is implementation-defined and may be changed without notice.**

The externally-binding surfaces are:

1. The **redirect endpoint** at `GET /:shortUrl` — invoked by every link in the wild (SMS, email, posters, QR codes, partner sites, search engines).
2. The **file URL shape** at `https://{file-hostname}/{shortUrl}.{ext}` — appears as `urls.longUrl` for file-backed short links, republished in user-facing materials.
3. The **External REST API** at `/api/v1/*` and `/api/v1/admin/*` — designed and documented for third-party integration; gated by `FF_EXTERNAL_API`.
4. The **API key format and authentication** for the External REST API — integrators have generated keys and store them in their own systems.

Everything else — the SPA-facing `/api/*` routes (login, user, qrcode, link-stats, link-audit, directory, callback), cookie names, session storage layout, response envelopes, HTML 404 templates, asset paths, log format, internal headers, the dual query/body parameter source on `GET /api/user/url`, the `hasApiKey` stringly-typed response — is **internal**. A rewrite SHOULD reach functional parity with these surfaces (so the SPA still works), but is free to change paths, methods, request shapes, response shapes, and status semantics. The SPA is part of the rewrite and may be updated in lockstep.

### 27.2 Redirect endpoint (external)

`GET /{shortUrl}` MUST continue to be served at the deployment's production origin (configured via `OG_URL`, e.g. `https://go.gov.sg`).

Required behaviors:

- **Path grammar**: `shortUrl ∈ [a-zA-Z0-9-]+`. A reimplementation MUST resolve every previously-issued short URL whose slug satisfies this grammar.
- **Trailing-character tolerance**: requests of the form `/{shortUrl}.` (or with one trailing non-slug character) MUST resolve to the same short URL. SMSes and email clients commonly append sentence terminators to the URL.
- **Case-insensitive lookup**: `/Foo`, `/FOO`, and `/foo` MUST resolve identically. The slug grammar accepts uppercase, but no two short URLs ever differed only by case in the canonical deployment.
- **Successful resolution**: respond with either:
  - HTTP 302 with `Location: <longUrl>` (direct redirect), or
  - HTTP 200 with an HTML page that navigates the user to `longUrl` after a short delay (the "transition page").
  - The choice between the two is implementation-defined; both have been used historically. Crawlers MUST receive 302.
- **Not-found resolution**: when `shortUrl` does not exist or has been deactivated, respond with HTTP 404. The body MAY be HTML or JSON; deployed link previews tolerate either.
- **Threat-deactivation parity**: a previously-resolvable short URL MAY become 404 if its destination is detected as malicious. This is a documented and acceptable outcome from a citizen's perspective.

Behaviors that are **not** part of the external contract and may change:

- The exact HTML rendered on the transition page or 404 page.
- The Google Analytics events emitted during a redirect.
- The internal mechanism (cache, replica, transition-page suppression cookie) used to compute the redirect.
- Whether the transition page is shown at all, and the criteria for showing it.

### 27.3 File-hosting URLs (external)

When a short link points to a hosted file, its `longUrl` MUST take the form:

```
https://{FILE_HOSTNAME}/{shortUrl}.{ext}
```

where `FILE_HOSTNAME` is the public file-serving hostname of the deployment (e.g., `file.go.gov.sg`). This URL is what citizens see, what gets shared in messages, and what is stored in `urls.longUrl` for every file-backed short link issued to date.

Required behaviors:

- The hostname and key format MUST resolve every file URL ever issued. A reimplementation that wants to move to a different storage backend MUST either keep the hostname pointing at the new backend with key-compatible paths, or maintain a redirect from the legacy URLs.
- The `{ext}` portion MUST match the actual file's extension and be a single segment.

Behaviors that may change:

- The storage backend itself (any object store: S3, GCS, on-prem, custom, etc.).
- The `Content-Type`, `Cache-Control`, and ACL semantics of the underlying object — as long as the URL remains fetchable by a browser.
- The internal key format inside the storage backend, as long as the public URL is preserved.

### 27.4 External REST API v1 (external)

The `/api/v1/*` and `/api/v1/admin/*` namespaces are the documented integration surface for third parties. A reimplementation MUST preserve:

- **Paths and HTTP methods** exactly:
  - `GET /api/v1/urls`
  - `POST /api/v1/urls`
  - `PATCH /api/v1/urls/:shortUrl`
  - `POST /api/v1/admin/urls`
- **Authentication scheme**: `Authorization: Bearer <apiKey>` (§8.2 / §27.5).
- **Feature-flag gating**: when `FF_EXTERNAL_API` is disabled, these paths MUST return HTTP 404 (not 401). Integrators detect "API disabled" by 404.
- **Request schema** for each route, as defined in §19. New optional fields MAY be added; required fields MUST NOT be added; existing required fields MUST NOT be removed.
- **Response schema** for each route. The mapped `StorableUrl` returned by these endpoints is a stable, versioned DTO — it omits internal fields (`safeBrowsingExpiry`, `userId`) by design. Adding fields is safe; removing, renaming, or retyping fields is not.
- **Status codes**: 200 on success, 400 on validation failure, 401 on missing/invalid API key, 404 when the feature is disabled or the resource is absent. Status semantics MUST NOT shift between these classes.

Implementation details that MAY change:

- The internal handler that backs each route.
- Whether `:shortUrl` is lifted into the request body before validation (the existing implementation does this via a middleware; a rewrite need not).
- The on-disk representation of the URL record.
- The presence of additional response fields beyond the documented schema.

### 27.5 API key authentication (external)

API keys issued by the current deployment MUST continue to authenticate after a rewrite. A key is the opaque string `${env}_${version}_${random}` returned once at generation time. To preserve this:

- The hash stored at `users.apiKeyHash` MUST remain verifiable. The existing format is `${env}_${version}_${bcrypt(random, API_KEY_SALT)}`. A reimplementation MAY use a different verification scheme **only if** it migrates existing hashes to the new scheme during deployment, or rejects existing keys and forces all integrators to re-generate.
- A rewrite that wishes to preserve existing keys MUST therefore preserve the bcrypt verification path and the `API_KEY_SALT` environment variable.
- A rewrite that explicitly does *not* wish to preserve existing keys MUST publish a key-rotation deadline before deployment.

`API_KEY_VERSION` exists precisely to bracket this concern. Bumping the version (`v1` → `v2`) is a clean way to introduce a new format while continuing to verify old keys against the old format.

### 27.6 Out of scope for backward compatibility

The following are part of the rewrite's own design surface and MAY change freely:

| Surface | Why it's internal |
|---------|-------------------|
| All `/api/*` routes outside `/api/v1/*` | Consumed only by the SPA, which is rewritten together with the server. |
| `/api/login/*`, `/api/logout`, `/api/user/*`, `/api/qrcode`, `/api/link-stats`, `/api/link-audit`, `/api/directory/search`, `/api/callback/qr`, `/api/stats`, `/api/links`, `/api/ga` | Same reason. |
| Cookie names (`gogovsg`, `visits`) | Session cookies are reissued on next login; the visits cookie at worst causes one extra transition-page display. |
| Session-store technology (`connect-redis` layout) | A rewrite may use any session backend. |
| `GET /assets/transition-page/js/redirect.js` | An implementation detail of the transition page; if the rewrite renders the transition page differently, this route can disappear. |
| Locale loader path (`/locales/...`) | Tied to the client's i18n configuration. |
| The `hasApiKey` stringly-typed `'true'/'false'` response | SPA-only; safe to replace with a boolean. |
| The Joi-on-body validator over `GET /api/user/url` that reads from `req.query` | SPA-only quirk. |
| Internal headers like `Cache-Control: no-store` | A rewrite may apply different caching policies. |
| The morgan log format and StatsD metric names | Internal observability — a rewrite may emit different telemetry. |
| HTML 404, 500 templates | Visual surface, free to redesign. |

### 27.7 Removal policy for the External REST API

Because §27.4 is the only HTTP surface bound by an external versioning contract, removal of an `/api/v1/*` field follows this policy:

1. Introduce a new versioned namespace (`/api/v2/*`).
2. Continue serving `/api/v1/*` for at least one quarter after the new version is announced.
3. Emit a `Deprecation` HTTP response header on every `/api/v1/*` response during the deprecation window.

Adding fields to existing `/api/v1/*` responses is non-breaking and requires no version bump.

---

## Appendix A. Out-of-process Workers

Four logical workers complement the long-running server. Each describes a contract — input shape, side effects, output — rather than a concrete deployment artifact. Implementations MAY realise them as cloud functions, container jobs, in-process scheduled tasks, or any other mechanism, provided the contract is honored.

### A.1 Migrate URL to user

Input:

```
{ shortUrl: string, toUserEmail: string }
```

Behavior: reassign ownership of `shortUrl` to the user with the given email, creating the user if necessary. The operation MUST atomically update the URL's owner and emit a `UrlHistory` record. Returns `{ Status: "URL successfully migrated." }` on success; on error, fails with `"URL migration failed. ${error}"`.

### A.2 Migrate user's links

Input:

```
{ fromUserEmail: string, toUserEmail: string }
```

Behavior: reassign every URL owned by `fromUserEmail` to `toUserEmail`, creating the target user if necessary. The operation MUST atomically update ownership for every affected URL and emit a `UrlHistory` record per URL. Returns `{ Status: "URL successfully migrated. ${rowCount} rows affected" }` on success; fails with `"User links migration failed. ${error}"`.

### A.3 Bulk QR code generation

Triggered by an async-job-queue message of the form `{ jobItemId, mappings: [{shortUrl, longUrl}, …] }` (§13.2).

Environment / configuration: a domain string used to build the human-readable short link encoded in each QR; the bulk-output object-store bucket; the callback endpoint URL and shared-secret bearer token.

Steps:

1. Build a CSV with header `Short URL,Original URL` and upload to `${jobItemId}/generated.csv`.
2. For each `shortUrl`, render a branded QR per §14 as SVG, then collect all SVGs into a zip and upload to `${jobItemId}/generated_svg.zip`.
3. Repeat as PNG → `${jobItemId}/generated_png.zip`.
4. Notify the server via the configured callback with `Authorization: Bearer <shared secret>` and body `{ jobItemId, status: { isSuccess, errorMessage } }`.

On any step failure, the callback MUST be sent with `isSuccess=false` and the error message. The worker MUST be safe under at-least-once delivery: re-running for the same `jobItemId` MUST overwrite the same artifact keys deterministically.

### A.4 Email-delivery event capture

If the chosen email transport publishes bounce, complaint, or delivery events, an auxiliary worker SHOULD subscribe to those events, identify the affected recipient address, mark it as undeliverable, and suppress future sends. The event format, subscription mechanism, and storage of suppression state are all implementation-defined.

---

## Appendix B. Validation Rules Reference

Quick reference for the rules in §7 and §12:

| Field | Pattern / Rule | Max length |
|-------|----------------|------------|
| `shortUrl` | `^[a-zA-Z0-9-]+$` | implementation-defined; auto-gen length 8 |
| `longUrl` | HTTPS, valid TLD, not IP, not circular, not blacklisted | none |
| `email` | `validator.isEmail` AND `minimatch(VALID_EMAIL_GLOB_EXPRESSION)` | none |
| `description` | printable ASCII | 200 |
| `tag` (string) | `^[A-Za-z0-9_-]+$` | 25 |
| tags per link | unique | 3 |
| file extension | allowlist (§7.6) | — |
| file size | ≤ 20 MiB | — |
| CSV size | ≤ 5 MiB | — |
| CSV rows | ≤ `BULK_UPLOAD_MAX_NUM` | — |

---

## Appendix C. Metrics Reference

All metrics MUST be prefixed `go.`. The metrics MAY be emitted as counters, gauges, histograms, or any other primitive supported by the observability platform; their names and tags are the stable contract.

| Metric | Tags | Source |
|--------|------|--------|
| `apikey.generate` | `isnew:bool` | API key creation. |
| `otp.generate.success` / `.failure` | — | `/api/login/otp`. |
| `otp.verify.success` / `.failure` | — | `/api/login/verify`. |
| `malicious_activity.file` | — | Antivirus positive. |
| `malicious_activity.link` | — | URL threat-scan positive. |
| `scan.file.failure` | — | Antivirus error. |
| `scan.link.failure` | — | URL threat-scan error. |
| `shortlink.clicks` | — | Redirect served. |
| `shortlink.create` | `source:CONSOLE/API/BULK`, `isfile:bool` | URL creation. |
| `user.new` | — | First-time user create. |
| `bulk.validation.error` | `acceptableLinkCount`, `validHeader`, `onlyOneColumn`, `isNotEmpty`, `isValidUrl`, `isNotBlacklisted`, `isNotCircularRedirect`, `noParsingError` | Per-row failure. |
| `bulk.hash.success` / `.failure` | — | Bulk persistence. |
| `job.start.success` / `.failure` | — | Per-item job dispatch. |
| `job.update.success` / `.failure` | — | Aggregate status recompute. |
| `job.email.success` / `.failure` | — | Job completion email. |
| `directory.search.domain` / `.email` | — | Directory queries. |
| `error.unhandled_rejection` | — | Process-level handler. |

---

## Appendix D. Locale Schema

The deployment ships exactly one English locale file. Its location and loading mechanism are implementation-defined. The key schema MUST be:

```
general:
  emailDomain          string  // e.g. "gov.sg"
  shortUrlPrefix       string  // e.g. "go.gov.sg/"
  appTitle             string  // e.g. "Go.gov.sg"
  officerType          string  // e.g. "public officers"
  appCatchphrase:      { styled: string, noStyle: string }
  appDescription:      { subtitle: string }
  linkAdmins           string
  appSignInPrompt      string
  copyright            string
  copyrightTag         string
  builtBy              string
  links:
    contribute, directory, apiintegration, dashboard,
    feedback, faq, privacy, terms, contact,
    builtBy, linkedin, facebook, apiDoc  // all strings (URLs or routes)
homePage:
  targetUsersPhrase    string
  features:
    antiPhishing       string
    customised         string
    analytics          string
    fileSharing:       { description: string }
  trustedBy:           { '1': string, '2': string, '3': string, '4': string, '5': string }
login:
  whitelistPhrase      string
  referrals:
    '1': { officerPhrase: string, emailDomain: string, link: string }
    '2': { officerPhrase: string, emailDomain: string, link: string }
  placeholders:
    email              string
```

The locale loader is initialized for English only (no other languages are supported). HTML escaping of interpolated values MUST be performed by the rendering layer, not by the locale-loading library.

---

## Appendix E. Public Surface Map

A single-page index of every public HTTP route.

Columns:
- **Auth**: `S` = session required, `K` = API key required, `A` = admin role required, `–` = public.
- **BC**: backward-compatibility class. **`E`** = externally-binding (third-party integrators, published links, or stored URLs in the wild depend on this route — see §27); **`I`** = internal (consumed only by the GoGovSG SPA or other rewritable surfaces; may be redesigned).

| Method | Route | Auth | BC | Section |
|--------|-------|------|----|---------|
| `GET` | `/:shortUrl` | – | **E** | §10, §27.2 |
| `GET` | `/api/v1/urls` | K | **E** | §19.1, §27.4 |
| `POST` | `/api/v1/urls` | K | **E** | §19.1, §27.4 |
| `PATCH` | `/api/v1/urls/:shortUrl` | K | **E** | §19.1, §27.4 |
| `POST` | `/api/v1/admin/urls` | K + A | **E** | §19.2, §27.4 |
| `GET` | `/api/ga` | – | I | §15.4 |
| `GET` | `/api/stats` | – | I | §15.1 |
| `GET` | `/api/links` | – | I | §5, §18.3 |
| `GET` | `/api/login/message` | – | I | §8.1 |
| `GET` | `/api/login/emaildomains` | – | I | §8.1 |
| `POST` | `/api/login/otp` | – (IP rate-limited) | I | §8.1 |
| `POST` | `/api/login/verify` | – | I | §8.1 |
| `GET` | `/api/login/isLoggedIn` | – | I | §8.1 |
| `GET` | `/api/logout` | – | I | §8.3 |
| `GET` | `/api/user/url` | S | I | §9.5 |
| `POST` | `/api/user/url` | S | I | §9.1 |
| `PATCH` | `/api/user/url` | S | I | §9.2 |
| `PATCH` | `/api/user/url/ownership` | S | I | §9.3 |
| `POST` | `/api/user/url/bulk` | S | I | §13.1 |
| `GET` | `/api/user/tag` | S | I | §9.6 |
| `POST` | `/api/user/apiKey` | S | I | §8.2 |
| `GET` | `/api/user/hasApiKey` | S | I | §8.2 |
| `GET` | `/api/user/message` | S | I | §6.3 |
| `GET` | `/api/user/announcement` | S | I | §6.3 |
| `GET` | `/api/user/job/latest` | S | I | §13.5 |
| `GET` | `/api/user/job/status` | S | I | §13.5 |
| `GET` | `/api/qrcode` | S | I | §14.1 |
| `GET` | `/api/link-stats` | S | I | §15.2 |
| `GET` | `/api/link-audit` | S | I | §16 |
| `GET` | `/api/directory/search` | S | I | §17 |
| `POST` | `/api/callback/qr` | K + A | I | §13.4 |
| `GET` | `/assets/transition-page/js/redirect.js` | – | I | §10.1 |
| `GET` | `/locales/en/translation.json` (or implementation-defined path) | – | I | §18.9 |

A rewrite that wishes to remain compatible with deployed links and existing API integrations need only preserve the **E**-class rows of this table (plus the file URL shape of §27.3 and the API key verification rules of §27.5). All **I**-class rows may be redesigned, removed, or replaced; the SPA must be updated in lockstep.

---

*End of specification.*
