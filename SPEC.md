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

A conforming implementation consists of the following components:

1. **Express HTTP server** that serves three classes of route: redirect (`GET /:shortUrl`), public API (`/api/*`), and static assets (the React bundle, locales, transition page, error page).
2. **PostgreSQL primary database** holding users, URLs, tags, jobs, click aggregates, and an append-only URL history.
3. **PostgreSQL replica database** used for read-heavy paths (redirect lookups, statistics) when a feature flag is enabled.
4. **Five logical Redis databases**: OTP cache, session store, redirect cache, statistics cache, Safe Browsing threat cache.
5. **S3-compatible object store** for hosted files.
6. **SQS queue** + AWS Lambda for asynchronous QR-code bundle generation.
7. **Email transport** (AWS SES in production, MailDev in development) for OTP delivery and job-completion notifications.
8. **External integrations**: Google Web Risk (Safe Browsing), Cloudmersive (antivirus), Google Analytics, Datadog APM/RUM/StatsD.
9. **React SPA** comprising five subapplications (home, login, user dashboard, public directory, API integration).
10. **Serverless helper Lambdas** for link ownership migration and SES event capture.

### 3.2 Layering

The server-side application is organized in three layers:

- **API layer** (`src/server/api/`) — Express routers, Joi schema validation, request middleware.
- **Module layer** (`src/server/modules/*`) — Controllers and services implementing business rules. Modules are: `auth`, `user`, `bulk`, `job`, `qr`, `redirect`, `threat`, `audit`, `analytics`, `statistics`, `directory`, `display`, `api`.
- **Persistence layer** (`src/server/repositories/`, `src/server/models/`) — Sequelize models, repositories, caching policies.

Cross-cutting concerns (DI, config, logging, metrics) live in `src/server/util/`. Inversion of control is provided by a DI container; bindings MUST be assembled at startup and SHOULD NOT be reassembled per request.

### 3.3 Process Model

A single Node.js process (Node 18) services every HTTP route. There are no in-process background workers other than asynchronous fire-and-forget side effects (statistics increments, cache warming). All long-running work (QR generation, link migration, SES event capture) MUST run in separate Lambda processes triggered by SQS or SNS.

---

## 4. Core Domain Model

The domain consists of seven first-class entities, each with a stable identifier and a defined lifecycle.

### 4.1 User

Represents one authenticated officer.

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | int (PK, autoincrement) | Server-assigned. |
| `email` | text, UNIQUE, lowercase | MUST match `VALID_EMAIL_GLOB_EXPRESSION` and `validator.isEmail()`. Setter MUST trim and lowercase. |
| `apiKeyHash` | text, UNIQUE, nullable | Bcrypt hash of API key suffix. See §8.2. |
| `createdAt`, `updatedAt` | timestamp | Server-managed. |

A user is created lazily the first time their email successfully verifies an OTP, or the first time an admin provisions a link on their behalf.

### 4.2 Url

Represents one short link, identified by its short URL slug.

| Field | Type | Constraints |
|-------|------|-------------|
| `shortUrl` | string (PK) | MUST match `/^[a-zA-Z0-9-]+$/`. Stored case-sensitively but treated case-insensitively when used in cache keys. |
| `longUrl` | text, NOT NULL | MUST validate per §7.2. For file links, holds the S3 public URL. |
| `state` | enum (`ACTIVE`, `INACTIVE`), default `ACTIVE` | Inactive URLs MUST resolve to 404. |
| `isFile` | boolean, NOT NULL | `true` ↔ `longUrl` is an S3 object URL. |
| `contactEmail` | text, nullable | Optional. MUST be lowercased and pass email glob check if present. |
| `description` | text, NOT NULL, default `''` | Max 200 printable ASCII characters. |
| `source` | enum (`BULK`, `API`, `CONSOLE`), NOT NULL | Records origin of creation. |
| `tagStrings` | text, NOT NULL, default `''` | Semicolon-separated denormalized tag display names. |
| `safeBrowsingExpiry` | timestamp, nullable | Expiry of last clean Safe Browsing scan. |
| `userId` | int (FK → users.id), nullable | Owner. |
| `createdAt`, `updatedAt` | timestamp | |

A GIN full-text index combining (`shortUrl` weight A, `longUrl` weight B, `description` weight C) MUST exist for directory search.

### 4.3 Tag

Represents one normalized tag. Many-to-many with `Url` through `url_tag`.

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | int (PK, autoincrement) | |
| `tagString` | string(255), UNIQUE, NOT NULL | Display form. Matches `/^[A-Za-z0-9_-]+$/`, ≤ 25 chars. |
| `tagKey` | string(255), NOT NULL, indexed | Lowercased form used for case-insensitive search. |
| `createdAt`, `updatedAt` | timestamp | |

A link MUST NOT carry more than `MAX_NUM_TAGS_PER_LINK = 3` tags.

### 4.4 UrlHistory

Append-only history of every URL mutation. Written automatically by Url `afterCreate` and `afterUpdate` hooks.

| Field | Type | Notes |
|-------|------|-------|
| `id` | int (PK) | |
| `urlShortUrl` | string, FK → urls.shortUrl, NOT NULL, indexed | |
| `userId` | int, FK → users.id, NOT NULL | Acting user. |
| `longUrl`, `state`, `isFile`, `contactEmail`, `description`, `source`, `tagStrings` | Same types as `urls` | Denormalized snapshot at moment of change. |
| `createdAt`, `updatedAt` | timestamp | |

A `beforeBulkUpdate` hook on `Url` MUST reject bulk updates so every change passes through `afterUpdate` and lands in history.

### 4.5 Click Statistics

Four tables aggregate clicks, all keyed by `shortUrl`.

- **`url_clicks`** — composite total. PK: `shortUrl`. Columns: `clicks` (int, default 0).
- **`devices_stats`** — device-class totals. PK: `shortUrl`. Columns: `mobile`, `tablet`, `desktop`, `others` (ints, default 0).
- **`daily_stats`** — daily totals. Composite PK: (`shortUrl`, `date`). Columns: `clicks`.
- **`weekday_stats`** — weekday-hour heatmap. Composite PK: (`shortUrl`, `weekday`, `hours`). `weekday` ∈ [0,7), `hours` ∈ [0,24).

All three time-dimensioned tables MUST be keyed in Asia/Singapore local time. A `url_clicks` row MUST be created automatically when a `Url` row is created. The `url_clicks.clicks` column MUST be indexed descending for popularity sorting.

### 4.6 Job and JobItem

Async-job tracking for bulk QR generation.

`Job`:

| Field | Type | Notes |
|-------|------|-------|
| `id` | int (PK) | |
| `uuid` | UUID v4, UNIQUE, NOT NULL | External identifier. |
| `userId` | int, FK → users.id | |
| `status` | enum (`IN_PROGRESS`, `SUCCESS`, `FAILURE`), default `IN_PROGRESS` | |
| `createdAt`, `updatedAt` | timestamp | |

`JobItem`:

| Field | Type | Notes |
|-------|------|-------|
| `id` | int (PK) | |
| `jobItemId` | string, UNIQUE, NOT NULL | Format: `${job.uuid}/${batchIndex}`. |
| `jobId` | int, FK → jobs.id, NOT NULL | |
| `status` | enum (`IN_PROGRESS`, `SUCCESS`, `FAILURE`), default `IN_PROGRESS` | |
| `message` | string, NOT NULL, default `''` | Free-form failure detail. |
| `params` | JSON, NOT NULL | Lambda invocation payload. |
| `createdAt`, `updatedAt` | timestamp | |

Job aggregate status is computed from items per §13.3.

### 4.7 OTP Record (Redis)

Not a SQL entity but a first-class object. Key: `${email}:${ip}`. Value: `{ hashedOtp, retries }`. TTL: `OTP_EXPIRY` seconds (default 300).

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
| `DB_URI` | Postgres primary connection string. |
| `REPLICA_URI` | Postgres read-replica connection string. |
| `OG_URL` | Origin URL of the service (e.g., `https://go.gov.sg`); used for circular-redirect prevention and trusted-referrer detection. |
| `REDIS_OTP_URI` | Redis URI for OTP cache. |
| `REDIS_SESSION_URI` | Redis URI for session store. |
| `REDIS_REDIRECT_URI` | Redis URI for redirect cache. |
| `REDIS_STAT_URI` | Redis URI for statistics cache. |
| `REDIS_SAFE_BROWSING_URI` | Redis URI for Safe Browsing threat cache. |
| `SESSION_SECRET` | Secret for both `express-session` and `cookie-session`. |
| `VALID_EMAIL_GLOB_EXPRESSION` | Glob pattern allowed by minimatch with `{ noext: false, noglobstar: true, nobrace: true, nonegate: true }`. |
| `AWS_S3_BUCKET` | Bucket name for file uploads. |
| `API_KEY_SALT` | Bcrypt salt used to hash API key suffixes. |
| Display name | Human-readable service name shown in templated HTML and returned in some API responses. Implementation-defined (env var or build-time constant). |

### 6.2 Production-only Required Variables

`SES_HOST`, `SES_PORT`, `SES_USER`, `SES_PASS` MUST be set when `NODE_ENV !== 'development'`.

### 6.3 Optional Variables and Defaults

| Variable | Default | Purpose |
|----------|---------|---------|
| `NODE_ENV` | `production` | Controls cookie security, log level, OTP rate-limit enforcement. |
| `SALT_ROUNDS` | 10 | Bcrypt cost for OTPs and API keys. |
| `OTP_EXPIRY` | 300 (s) | OTP TTL. |
| `REDIRECT_EXPIRY` | 300 (s) | Redirect-cache TTL. |
| `COOKIE_MAX_AGE` | 86_400_000 (ms = 24 h) | Session cookie lifetime. |
| `BULK_UPLOAD_MAX_NUM` | 1000 | Max URLs per CSV. |
| `BULK_UPLOAD_RANDOM_STR_LENGTH` | 8 | Generated short-URL length for bulk. |
| `API_LINK_RANDOM_STR_LENGTH` | 8 | Generated short-URL length for API. |
| `BULK_QR_CODE_BATCH_SIZE` | 1000 | QR batch size per Lambda invocation. |
| `BULK_QR_CODE_BUCKET_URL` | empty | Base URL used when constructing JobItem download URLs. |
| `ACTIVATE_BULK_QR_CODE_GENERATION` | `false` | Master switch for the QR Lambda pipeline. |
| `SQS_BULK_QRCODE_GENERATE_START_URL` | undefined | Queue URL. |
| `SQS_REGION` | empty | Queue region. |
| `SQS_TIMEOUT` | 10_000 (ms) | SQS send timeout. |
| `JOB_POLL_INTERVAL` | 5000 (ms) | Server-side long-poll interval. |
| `JOB_POLL_ATTEMPTS` | 12 | Server-side long-poll attempts. After exhaustion respond 408. |
| `FF_EXTERNAL_API` | `false` | Gates `/api/v1/*` and `/api/v1/admin/*`. |
| `FF_USE_REPLICA_FOR_REDIRECTS` | `false` | Use replica DB for redirect lookups. |
| `API_KEY_VERSION` | `v1` | Component of API key string. |
| `ADMIN_API_EMAILS` | empty | Comma-separated emails allowed to call admin API. |
| `SAFE_BROWSING_KEY` | undefined | Google Web Risk API key. Disabled when unset. |
| `SAFE_BROWSING_LOG_ONLY` | `false` | When `true`, threats are logged but not blocked. |
| `CLOUDMERSIVE_KEY` | undefined | Antivirus key. When unset, virus scan is skipped. |
| `CSP_REPORT_URI` | undefined | CSP violation reporting endpoint. |
| `CSP_ONLY_REPORT_VIOLATIONS` | `false` | Run CSP in report-only mode. |
| `GA_TRACKING_ID` | undefined | Google Analytics property. |
| `LOGIN_MESSAGE` | undefined | Banner on login page. |
| `USER_MESSAGE` | undefined | Banner on user dashboard. |
| `ANNOUNCEMENT_TITLE`, `ANNOUNCEMENT_SUBTITLE`, `ANNOUNCEMENT_MESSAGE`, `ANNOUNCEMENT_URL`, `ANNOUNCEMENT_IMAGE`, `ANNOUNCEMENT_BUTTON_TEXT` | undefined | Optional logged-in modal. All-or-none semantics: client renders the modal only if `ANNOUNCEMENT_MESSAGE` is truthy. |
| `ROTATED_LINKS` | undefined | Comma-separated short URLs to rotate on the landing page. |
| `USER_COUNT`, `CLICK_COUNT`, `LINK_COUNT` | 77288, 666_820_545, 28_151_439 | Static counters for landing page. |
| `DB_POOL_SIZE` | 40 | Sequelize pool size. |
| `BUCKET_ENDPOINT` | `http://localstack:4566` | S3 endpoint override (dev). |
| `ACCESS_ENDPOINT` | `http://localhost:4566` | S3 access endpoint (dev). |
| `OTP_RATE_LIMIT` | implementation-defined; 0 in development | OTP requests per IP per minute. |
| `DD_SERVICE`, `DD_ENV`, `DD_API_KEY` | undefined | Datadog identification. |
| `POSTMAN_API_URL`, `POSTMAN_API_KEY`, `ACTIVATE_POSTMAN_FALLBACK` | undefined / `false` | Optional Postman email fallback. |

### 6.4 Cookie Configuration

Session cookies MUST be named `gogovsg` and set with:

```
httpOnly: true
sameSite: 'strict'
secure: NODE_ENV !== 'development'
maxAge: COOKIE_MAX_AGE
```

A separate `visits` cookie (cookie-session middleware) MUST track per-visitor short URL history for transition-page suppression, with `maxAge` = 7 days and signed by `SESSION_SECRET`. Its serialized array MUST be capped at `COOKIE_SESSION_MAX_SIZE_BYTES` (default 2000) by LRU eviction.

---

## 7. Identifiers, Validation, and Constants

### 7.1 Short URL

- Pattern: `/^[a-zA-Z0-9-]+$/`.
- Used as PK of `urls` and as Redis cache key (lowercased internally).
- For auto-generation: `nanoid/async` with alphabet `0123456789abcdefghijklmnopqrstuvwxyz` and length `BULK_UPLOAD_RANDOM_STR_LENGTH` (default 8) or `API_LINK_RANDOM_STR_LENGTH` (default 8). On collision, retry.

### 7.2 Long URL

Validation is centralized in `src/shared/util/validation.ts` and MUST apply on both client and server.

- MUST be a fully-qualified URL with `https:` scheme. Plain hostnames are rejected.
- MUST have a valid TLD; IP addresses are rejected.
- Validator options: `{ protocols: ['https'], require_tld: true, require_protocol: true, require_host: true, allow_underscores: false, allow_trailing_dot: false, disallow_auth: false }`.
- MUST NOT be circular: the URL's hostname MUST NOT resolve to the service origin (`OG_URL` hostname).
- MUST NOT match the blacklist (substring blocklist sourced from `src/server/resources/blacklist`).

### 7.3 Email

- MUST pass `validator.isEmail()` with `{ allow_utf8_local_part: false }`.
- MUST be lowercased and trimmed before storage and before pattern matching.
- MUST satisfy `minimatch(email, VALID_EMAIL_GLOB_EXPRESSION, { noext: false, noglobstar: true, nobrace: true, nonegate: true })`.

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
- MIME type MUST be detected from buffer (via `file-type`), with manual mapping for `csv → text/csv`, `dwf → application/x-dwf`, `dxf → application/dxf`.

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

Request body (Joi):

```
{ email: string (required, lowercase, matches email glob) }
```

Server behavior:

1. Apply the OTP-generation rate limiter keyed by client IP: `windowMs = 60_000`, `max = OTP_RATE_LIMIT`. Return 429 on overflow.
2. Generate a 6-digit numeric OTP using cryptographic randomness.
3. Hash the OTP with bcrypt using `SALT_ROUNDS`.
4. Store `{ hashedOtp, retries: 3 }` in the OTP Redis at key `${email}:${ip}` with TTL `OTP_EXPIRY`.
5. Send the unhashed OTP to the supplied email through the configured mailer (SES in production, MailDev in development, Postman as optional fallback). The email body MUST include the OTP, the requester's IP, and the deployment's display name.
6. On success, return `200 { message: "OTP generated and sent." }` and increment `OTP_GENERATE_SUCCESS`. On mailer failure, increment `OTP_GENERATE_FAILURE` and return 500.

**`POST /api/login/verify`** — verifies an OTP.

Request body:

```
{ email: string (required), otp: string (required) }
```

Server behavior:

1. Fetch `{ hashedOtp, retries }` from Redis at `${email}:${ip}`. If absent, return 401 (expired/not found).
2. Compare with `bcrypt.compare(otp, hashedOtp)`.
3. **Mismatch**: decrement `retries`. If `retries > 0`, rewrite the record and respond 401 `"OTP hash verification failed, ${retries} attempt(s) remaining."`. If `retries == 0`, delete the record and respond 401 (locked out for this email+IP until expiry).
4. **Match**: `userRepository.findOrCreateWithEmail(email)`, set `req.session.user = { id, email }`, asynchronously delete the OTP record, return `200 { message: "OTP hash verification ok.", user }`. Emit `OTP_VERIFY_SUCCESS`. Emit `USER_NEW` on first-time creation.

**`GET /api/login/isLoggedIn`** — returns `200 { user }` when a session user exists, `404` otherwise.

**`GET /api/login/emaildomains`** returns the configured email glob to the client; **`GET /api/login/message`** returns `LOGIN_MESSAGE`.

### 8.2 API Key Authentication

API keys authenticate the external REST API. Key structure: `${apiEnv}_${apiKeyVersion}_${randomSuffix}`.

- `apiEnv` is `"live"` when `DD_ENV === 'production'`, otherwise `"test"`.
- `apiKeyVersion` defaults to `"v1"`.
- `randomSuffix` is 32 cryptographically random bytes, base64 encoded.

The server stores `${apiEnv}_${apiKeyVersion}_${bcryptHash(randomSuffix, API_KEY_SALT)}` in `users.apiKeyHash`. The full key is shown to the user exactly once at generation.

**`POST /api/user/apiKey`** (session-authenticated) generates a new key, overwriting any prior key. The response body MUST contain the unhashed key. Emit `API_KEY_GENERATE` with tag `isnew = true|false`.

**`GET /api/user/hasApiKey`** (session-authenticated) returns whether `apiKeyHash` is set.

The `apiKeyAuthMiddleware`:

1. Parse `Authorization: Bearer <key>`. Return 401 if missing or malformed.
2. Split the key on `_`; bcrypt-hash the suffix; query `users` by the rebuilt `${env}_${version}_${hash}`.
3. On hit, set `req.body.userId = user.id` and call `next()`. On miss, return 401.

The `apiKeyAdminAuthMiddleware` additionally verifies `user.email ∈ ADMIN_API_EMAILS` and returns 401 otherwise.

### 8.3 Session Management

Sessions are stored in Redis (`REDIS_SESSION_URI`) via `connect-redis`. The session object is `{ user: { id, email } }`. The session middleware MUST set `resave: false`, `saveUninitialized: false`. Cookie attributes are listed in §6.4.

**`GET /api/logout`** destroys the session and returns `200 { message: "Logged out" }`.

A session-guard middleware (`userGuard`) MUST reject requests lacking `req.session.user.id` with 401 and, on success, inject `req.body.userId = req.session.user.id` so handlers can treat session and API key auth uniformly.

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

1. `fileUploadMiddleware(MAX_FILE_UPLOAD_SIZE)`.
2. `preprocessFormData` — moves file under `req.body.files`, parses tags JSON.
3. `fileCheckController.singleFileCheck` — exactly one file (or none).
4. `fileCheckController.fileExtensionAndMimeTypeCheck()` — §7.6.
5. `fileCheckController.fileVirusCheck` — §12.2.
6. `urlCheckController.singleUrlCheck` — §12.1.
7. `userController.createUrl`.

Service behavior (`UrlManagementService.createUrl`):

1. Verify user exists.
2. `urlRepository.isShortUrlAvailable(shortUrl)` — return 400 `AlreadyExistsError` if taken.
3. If file, upload to S3 at key `${shortUrl}.${ext}` and store `longUrl = ${file domain}/${shortUrl}.${ext}`; set `isFile = true`.
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
4. If `longUrl` changed, re-run Safe Browsing on the new URL.
5. If `file` supplied for an existing file URL, upload the new file and overwrite the existing S3 object.
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

Parameters are read from the query string. (The existing implementation also declares a body-level Joi validator with default fallbacks; the controller extracts conditions from `req.query`. A reimplementation is free to drop the body schema and accept query parameters exclusively.)

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

Returns `{ urls: StorableUrl[], count: number }`. Search uses substring match on `shortUrl` and `longUrl`; tags filter uses ILIKE wildcards.

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

- **Trailing-character tolerance**: the captured slug MAY be followed by a single trailing non-slug character (e.g., a `.` appended by an SMS or email client). `/foo.` MUST resolve to the same short URL as `/foo`. The existing implementation expresses this as the Express path regex `/:shortUrl([a-zA-Z0-9-]+).?`; a rewrite may parse the path differently as long as the behavior is preserved.
- **Case-insensitive lookup**: `/Foo`, `/FOO`, and `/foo` MUST resolve identically. The existing implementation lowercases the captured slug before cache and database lookup. Stored `shortUrl` values in the canonical deployment are effectively lowercase.

Everything else in this section describes the current implementation; rewrites may diverge.

The redirect handler renders the transition page from a server-side template. The current implementation loads its in-page JavaScript from a separate route — `GET /assets/transition-page/js/redirect.js` (a server-rendered `text/javascript` response built from the `redirect.ejs` template, parameterized with `gaTrackingId`, `EventCategory.TRANSITION_PAGE`, `EventAction.LOADED`, and `EventAction.PROCEEDED`). This indirection exists because the GA tracking ID is a runtime environment variable. A reimplementation that inlines the script, or uses a different transition page entirely, is acceptable; this auxiliary route is not part of the external contract.

Middleware in the current implementation includes `cookieSession` for the `visits` cookie (used to suppress the transition page on repeat visits). The cookie name is an implementation detail; a rewrite may use a different mechanism or no mechanism at all.

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

`isCrawler(userAgent)`: parse with `ua-parser-js`; if `browser.name`, `engine.name`, or `os.name` is missing → crawler. The bot regex `/bot|facebookexternalhit|Facebot|Slackbot|TelegramBot|WhatsApp|Twitterbot|Pinterest|Postman|url|Google-PageRenderer/` MUST classify a UA as device `'others'` for statistics purposes.

`fromTrustedReferrer(referrer)`: parse the referrer; trusted iff its origin equals `OG_URL`'s origin. Parse failures are treated as untrusted.

### 10.5 Click Statistics Update

After producing the redirect response (but possibly before returning to the client), the server MUST fire-and-forget:

```
device = getDeviceType(userAgent)   # mobile|tablet|desktop|others
db.exec("SELECT update_link_statistics(:shortUrl, :device)")
```

`update_link_statistics(text, text)` is a stored procedure that, within a transaction guarded by `pg_advisory_xact_lock(2142616474639426746)`:

1. `UPDATE url_clicks SET clicks = clicks + 1 WHERE shortUrl = :shortUrl;`
2. `INSERT INTO devices_stats … ON CONFLICT (shortUrl) DO UPDATE SET <device> = <device> + 1;`
3. `INSERT INTO daily_stats(shortUrl, date, clicks) VALUES (:shortUrl, current_date_in_sgt, 1) ON CONFLICT DO UPDATE SET clicks = clicks + 1;`
4. `INSERT INTO weekday_stats(shortUrl, weekday, hours, clicks) VALUES (:shortUrl, dow_sgt, hour_sgt, 1) ON CONFLICT DO UPDATE SET clicks = clicks + 1;`

The procedure MUST hardcode Asia/Singapore for `date`, `dow`, and `hour` extraction. Errors MUST be caught and logged but MUST NOT block the response.

### 10.6 Google Analytics Pageview

After the response is decided but before/while it is sent, the server SHOULD POST a measurement-protocol pageview to `https://www.google-analytics.com/collect` using `GA_TRACKING_ID`, including a server-generated/ingested GA cookie. Crawlers MUST be excluded.

### 10.7 Cookie Eviction

The `visits` cookie holds an array of recent short URLs. Visiting a known short URL MUST move it to the end; unknown short URLs MUST be appended. When the serialized cookie size exceeds `COOKIE_SESSION_MAX_SIZE_BYTES` (default 2000), entries MUST be removed from the head (LRU).

---

## 11. File Hosting

Files attached to short URLs are stored on S3 in `AWS_S3_BUCKET`:

- Object key: `${shortUrl}.${ext}`.
- ACL: public-read when `state = ACTIVE`, private when `state = INACTIVE`. Cache-Control on uploaded objects is `no-cache`.
- `longUrl` for a file URL MUST be constructed as `${fileURLPrefix}${AWS_S3_BUCKET}/${key}`.
  - In production, `fileURLPrefix = 'https://'`, so the URL is `https://${AWS_S3_BUCKET}/${shortUrl}.${ext}`. The bucket name is conventionally also the public hostname (e.g., `file.go.gov.sg`), making the canonical file URL `https://${FILE_HOSTNAME}/${shortUrl}.${ext}`. **External integrators and stored history depend on this URL shape; see §27.3.**
  - In development, `fileURLPrefix` is the LocalStack `ACCESS_ENDPOINT` followed by `/`.
- The reverse derivation `getKeyFromLongUrl(longUrl)` MUST extract the key as the final path segment.
- Files MUST pass the extension/MIME and antivirus checks of §12.2 before upload.
- Replacing a file (on edit) MUST overwrite the same key. The system MUST NOT permit changing whether a URL is a file URL after creation, nor changing its `longUrl` independently of the underlying object.

For development, LocalStack provides an S3-compatible endpoint at `BUCKET_ENDPOINT`.

---

## 12. Threat Detection

### 12.1 URL Threat Scanning

**Service: Google Web Risk.** Endpoint: `https://webrisk.googleapis.com/v1/uris:search?key={SAFE_BROWSING_KEY}`. Checked threat types:

- `MALWARE`
- `SOCIAL_ENGINEERING`
- `UNWANTED_SOFTWARE`
- `SOCIAL_ENGINEERING_EXTENDED_COVERAGE`

#### 12.1.1 Cache

A dedicated Redis (`REDIS_SAFE_BROWSING_URI`) caches threat results keyed by full URL. Cache TTL: 300 seconds (`SafeBrowsingRepository.DEFAULT_CACHE_DURATION_IN_S`). Cache hit returns the cached `{ threatTypes, expireTime }`. Cache miss queries Web Risk.

A separate `urls.safeBrowsingExpiry` column caches the *clean* result for 24 hours; this column is checked on redirect (§10.2).

#### 12.1.2 When `SAFE_BROWSING_KEY` is absent

The threat service MUST log a warning at startup and treat all URLs as clean.

#### 12.1.3 Log-only mode

When `SAFE_BROWSING_LOG_ONLY=true`, a detected threat MUST be logged and metric `MALICIOUS_ACTIVITY_LINK` incremented but the call MUST return clean.

#### 12.1.4 Bulk scan

`isThreatBulk(longUrls[])` MUST call `isThreat` concurrently and return true on any positive.

#### 12.1.5 At redirect

§10.2 specifies that an expired `safeBrowsingExpiry` triggers a re-scan and that a threat-positive scan MUST:

- Deactivate the link (`state = INACTIVE`).
- Email the owner.
- Return 404 to the caller (parity with not-found, avoiding leakage).

### 12.2 File Threat Scanning

**Service: Cloudmersive.** Options: `{ allowExecutables: false, allowInvalidFiles: false, allowScripts: false }`.

Pipeline (`fileVirusCheck`):

1. If `CLOUDMERSIVE_KEY` is unset, skip.
2. Call Cloudmersive. On error, emit `SCAN_FAILED_FILE` and return 500 `"Your file could not be scanned at this moment…"`.
3. If `isPasswordProtected`: return 400 `"Cannot upload password-protected files."`.
4. If `hasVirus`: emit `MALICIOUS_ACTIVITY_FILE`, return 400 `"File is likely to be malicious."`.

### 12.3 File Extension/MIME Validation

`fileExtensionAndMimeTypeCheck(allowed?)` middleware:

1. `getExtensionAndMimeType(file)` — use `file-type` on the buffer; fall back to filename extension; apply the manual overrides in §7.6.
2. If the resolved extension is empty or not in `allowed` (default §7.6), return 415 `"File type disallowed."`.
3. Set `file.mimetype` for downstream consumers.

---

## 13. Bulk Operations and Async Jobs

### 13.1 Bulk Upload (`POST /api/user/url/bulk`)

Multipart upload. File size ≤ `MAX_CSV_UPLOAD_SIZE` (5 MiB). Optional `tags` JSON.

Pipeline:

1. `bulkCSVUploadMiddleware(MAX_CSV_UPLOAD_SIZE)`.
2. `fileExtensionAndMimeTypeCheck(['csv'])`.
3. `fileVirusCheck`.
4. `BulkController.validateAndParseCsv` (delegates to `BulkService.parseCsv`).
5. `urlCheckController.bulkUrlCheck` — Safe Browsing on every parsed URL.
6. `BulkController.bulkCreate` — generates short URLs and persists.
7. If `ACTIVATE_BULK_QR_CODE_GENERATION === 'true'`, `JobController.createAndStartJob`.

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
| No PapaParse error | `noParsingError` | `"Parsing error"` |

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

1. `JobManagementService.createJob(userId)` inserts a `jobs` row with status `IN_PROGRESS`.
2. Chunk `urlMappings` by `BULK_QR_CODE_BATCH_SIZE` (default 1000) using `lodash.chunk`.
3. For each batch with index `i`:
   - Insert `JobItem { jobItemId: "${job.uuid}/${i}", jobId, status: IN_PROGRESS, message: '', params: { jobItemId, mappings } }`.
   - SQS send to `SQS_BULK_QRCODE_GENERATE_START_URL` with timeout `SQS_TIMEOUT`. Body: `{ jobItemId, mappings }`.

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
5. If `format` is `image/png` or `image/jpeg`, rasterize with `sharp`.
6. Respond with the correct `Content-Type` and header `Filename: ${OG_URL_HOST}/${shortUrl}`. Body is the binary buffer.

### 14.2 Bulk QR Pipeline

Enabled iff `ACTIVATE_BULK_QR_CODE_GENERATION === 'true'`. See §13.2 for the dispatch protocol and Appendix A for the Lambda implementation.

For each `jobItemId = "${job.uuid}/${i}"`, the Lambda produces three S3 objects:

- `${jobItemId}/generated.csv` — header `"Short URL,Original URL"`, one row per mapping.
- `${jobItemId}/generated_svg.zip` — `${shortUrl}.svg` per mapping.
- `${jobItemId}/generated_png.zip` — `${shortUrl}.png` per mapping.

### 14.3 Color and Logo

QR codes MUST be rendered with a single brand dark color and a single brand logo, both implementation-defined. The chosen color and logo are deployment-wide constants. Both the synchronous QR-code endpoint (§14.1) and the bulk-generation Lambda (§14.2, Appendix A.3) MUST use the same color and logo so that a single QR rendered ad hoc is visually indistinguishable from one rendered as part of a bulk job.

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
2. Query the replica:
   - `url_clicks.clicks` (total).
   - `devices_stats` (one row).
   - `daily_stats` between `today − offset` and `today` (Asia/Singapore).
   - `weekday_stats` (all 24×7 buckets).
3. If all device counters are zero AND no other rows exist, return null and 404.
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

Per §10.5, every successful redirect MUST invoke `update_link_statistics(shortUrl, device)`. The function MUST be idempotent under retries because the advisory lock is per-transaction.

### 15.4 Google Analytics Hits

§10.6. The server MAY use a server-generated GA client cookie carried across redirects via the redirect-bound `_ga` cookie; it MUST regenerate one if absent.

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

- **Email mode** (`isEmail === 'true'`): the `query` is interpreted as an email or email substring. The query MUST be reduced to the part after `@` if present. SQL: `users.email ILIKE '%${q}%'`. Emit `DIRECTORY_SEARCH_EMAIL`.
- **Text mode** (`isEmail === 'false'`): `@` MUST be stripped from the query to avoid leaking email-domain enumeration through this path. SQL: `plainto_tsquery('english', :q) @@ (weighted tsvector)`. Emit `DIRECTORY_SEARCH_DOMAIN`.

The weighted tsvector is `setweight(to_tsvector('english', shortUrl), 'A') || setweight(to_tsvector('english', longUrl), 'B') || setweight(to_tsvector('english', description), 'C')`.

### 17.3 Sort

- `POPULARITY` → `ORDER BY url_clicks.clicks DESC`.
- `RECENCY` → `ORDER BY urls.createdAt DESC`.

### 17.4 Response

```
{
  urls: [{ shortUrl: string, email: string, state: 'ACTIVE'|'INACTIVE', isFile: boolean }],
  count: number
}
```

---

## 18. Client Application

The browser client is a single-page React application using Redux (with thunk) for state, hash-based routing (`createHashHistory`), and Material-UI for layout. All client behaviors described here MUST be reproducible regardless of visual styling.

### 18.1 Routes

| Path | Subapp | Access | Behavior |
|------|--------|--------|----------|
| `/` | home | anonymous | If logged in, redirect to `/user`. Otherwise render landing. |
| `/login` | login | anonymous | If logged in, redirect to `/user`. |
| `/user` | user dashboard | authenticated | Private route; if anonymous, redirect to `/login` and remember `location.state.previous`. |
| `/directory` | directory | public | Public search. |
| `/apiintegration` | API integration | authenticated | Private route. |
| `/404/:shortUrl` | 404 | public | "Link not found" page. |

Private routes MUST be guarded by a `PrivateRoute` wrapper that checks `state.login.isLoggedIn` and redirects when false. After a successful OTP, the login subapp MUST navigate back to `location.state.previous` if present, else `/user`.

### 18.2 Global Layout

Across all subapps, the client MUST:

- On every page mount, fire a Google Analytics page-view event (page title is implementation-defined; the existing app uses `HOME PAGE`, `EMAIL LOGIN PAGE`, `OTP LOGIN PAGE`, `USER PAGE`, `CREATE LINK PAGE`, `DIRECTORY PAGE`, `API INTEGRATION`).
- Render a global snackbar at the application root. Actions `SET_ERROR_MESSAGE`, `SET_SUCCESS_MESSAGE`, `SET_INFO_MESSAGE`, `CLOSE_SNACKBAR` mutate the visible message and variant.
- Send all API requests with `credentials: 'include'` and `mode: 'same-origin'`. Cross-fetch is acceptable.
- Request a `LOGIN_MESSAGE` banner on the login page and a `USER_MESSAGE` banner on the dashboard.

### 18.3 Home Subapp

State (`state.home`):

```
linksToRotate?: string[]
statistics:    { userCount, linkCount, clickCount }
```

On mount:

1. `GET /api/login/isLoggedIn` — if 200, navigate to `/user`.
2. `GET /api/links` — server returns rotating-link payload (string sourced from `ROTATED_LINKS`). Dispatch `SET_LINKS_TO_ROTATE`.
3. `GET /api/stats` — dispatch `LOAD_STATS`.

### 18.4 Login Subapp

State (`state.login`):

```
email:        string
emailValidator: (email) => boolean   // closes over the server-supplied glob
user:         { id?, email? }
isLoggedIn:   boolean
formVariant:  'EMAIL_READY' | 'EMAIL_PENDING' | 'OTP_READY' | 'OTP_PENDING' | 'RESEND_OTP_DISABLED'
```

State machine:

```
EMAIL_READY --submit-> EMAIL_PENDING --resp.ok-> OTP_READY
                                  --resp.err-> EMAIL_READY (error toast)
OTP_READY   --submit-> OTP_PENDING   --resp.ok-> IS_LOGGED_IN_SUCCESS (navigate to /user)
                                  --resp.err-> OTP_READY (error toast)
OTP_READY   --resend-> EMAIL_PENDING -> OTP_READY -> RESEND_OTP_DISABLED (20 s) -> OTP_READY
```

Bootstrap fetches:

1. `GET /api/login/isLoggedIn`. If 200, set Datadog RUM user and dispatch `IS_LOGGED_IN_SUCCESS`. If 404, dispatch `IS_LOGGED_OUT`.
2. `GET /api/login/emaildomains` — server returns the glob string. The client constructs `emailValidator(email)` using `minimatch` AND `validator.isEmail`.

Validation:

- Email input lowercases on every keystroke.
- Email error message: `"This doesn't look like a valid ${domain} email."` shown only when the field has a value and fails `emailValidator`.
- OTP input is not validated client-side beyond non-emptiness.

After verify success, the client MUST identify the Datadog RUM user with `{ id, email }`.

### 18.5 User Dashboard Subapp

State (`state.user`):

```
initialised:           boolean
isFetchingUrls:        boolean
urls:                  UrlType[]
shortUrl, longUrl:     string  // create form
tags:                  string[]
createUrlModal:        boolean
tableConfig:           UrlTableConfig
urlCount:              number
message:               string | null
announcement:          AnnouncementType | null
uploadState:           { urlUpload: bool, fileUpload: bool }
isUploading:           boolean
createShortLinkError:  string | null
uploadFileError:       string | null
lastCreatedLink?:      string
linkHistory:           LinkChangeSet[]
linkHistoryCount:      number
statusBarMessage:      { header, body, variant: SUCCESS|ERROR|INFO, callbacks: string[] }
```

`UrlTableConfig`:

```
isTag:           boolean
numberOfRows:    number       // default 10
pageNumber:      number       // 0-indexed
searchText:      string
searchInput:     string       // debounced 500 ms
tags:            string       // semicolon-separated
filter:          { isFile?: boolean, state?: 'ACTIVE'|'INACTIVE' }
orderBy:         string       // default 'createdAt'
sortDirection:   'asc' | 'desc'
```

#### 18.5.1 Initial Load

On mount, dispatch in order:

1. `getUrlsForUser()` — `GET /api/user/url?{tableConfig}`. Dispatch `IS_FETCHING_URLS` true/false and `GET_URLS_FOR_USER_SUCCESS` with `{ urls, count }`.
2. If no cached emailValidator, fetch `/api/login/emaildomains`.
3. `GET /api/user/message` → `SET_USER_MESSAGE`.
4. `GET /api/user/announcement` → `SET_USER_ANNOUNCEMENT`.
5. `GET /api/user/job/latest` to determine whether a status bar should be displayed.

If `urlCount === 0` and no filters are active, render an empty state with a "Create link" CTA. Otherwise render `UserLinkTable`.

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

State (`state.api`):

```
hasApiKey:     boolean
apiKeyModal:   boolean
apiKey:        string
```

On mount: `GET /api/user/hasApiKey` → `USER_HAS_API_KEY` or `USER_HAS_NO_API_KEY`.

The "Generate API Key" action calls `POST /api/user/apiKey` (no body), dispatches `GENERATE_API_KEY_SUCCESSFULLY` with the response body, and opens `ApiKeyModal`. The full key is shown exactly once; the client MUST NOT persist it.

### 18.8 Request Utility

A single module wraps `cross-fetch`:

- `get(url, options?)`
- `postJson(url, data, options?)` — sets `Content-Type: application/json`.
- `postFormData(url, data, options?)` — no Content-Type; browser sets multipart boundary.
- `patch(url, data, options?)`
- `patchFormData(url, data, options?)`

All requests MUST include `credentials: 'include'` and `mode: 'same-origin'`.

### 18.9 Internationalization

The client loads a single English locale bundle via i18next at startup. The loader path is implementation-defined (the existing implementation uses `/locales/en/{{ns}}.json`). Default and fallback `lng` is `en`, `whitelist` is `['en']`, `interpolation.escapeValue` MUST be `false` (React handles XSS escaping). The locale schema is in Appendix D.

---

## 19. External REST API (v1)

Gated by `FF_EXTERNAL_API === 'true'`. Mounted under `/api/v1`. Authenticated by API key (§8.2).

### 19.1 User-scope

| Route | Body / Query |
|-------|--------------|
| `GET /api/v1/urls` | Same query as `/api/user/url` minus tag filtering. Returns `UrlsPaginated`. |
| `POST /api/v1/urls` | `{ longUrl (required, HTTPS), shortUrl? (auto-gen length `API_LINK_RANDOM_STR_LENGTH`) }`. Returns mapped `StorableUrl`. `source = API`. |
| `PATCH /api/v1/urls/:shortUrl` | `{ longUrl?, state? }`. File editing NOT allowed via API. |

API responses use a thin DTO mapper (`UrlV1Mapper`) that omits internal fields (e.g., `safeBrowsingExpiry`, `userId`).

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
- Job-completion emails (bulk QR). Body includes per-item S3 URLs.
- Ownership-transfer notifications.
- Malicious-link deactivation notices.

Production transport: Nodemailer over AWS SES (`pool: true`, `maxMessages: 100`, `maxConnections: 20`). Development: Nodemailer to MailDev (`ignoreTLS: true`). When `ACTIVATE_POSTMAN_FALLBACK=true`, on SES failure the server SHOULD attempt the configured Postman API.

Bounce/complaint capture is performed out-of-band by the `capture-ses-events` Lambda (Appendix A.4).

---

## 21. Persistence and Caching

### 21.1 PostgreSQL

- One Sequelize instance with `replication.write = DB_URI`, `replication.read = [REPLICA_URI]`.
- Timezone: `+08:00`.
- Pool size: `DB_POOL_SIZE` (default 40).
- All models declare `defaultScope = { useMaster: true }` so writes and reads are master-consistent by default. A `useReplica` scope is provided for analytics and search.
- A `beforeBulkUpdate` hook on `Url` MUST reject bulk updates to preserve the history invariant (§4.4).
- `afterCreate`, `afterUpdate`, `afterBulkCreate` hooks MUST write to `url_history` and (for create) seed `url_clicks`.
- A GIN full-text index over the weighted (shortUrl, longUrl, description) tsvector MUST exist.
- Index `idx_url_clicks_clicks_desc` MUST exist for popularity sort.

### 21.2 Redis databases

Five logically separate Redis databases (or instances):

| Purpose | Key | Value | TTL |
|---------|-----|-------|-----|
| OTP | `${email}:${ip}` | `{ hashedOtp, retries }` | `OTP_EXPIRY` (default 300 s) |
| Session | `sess:${sid}` | serialized session | `COOKIE_MAX_AGE` (touch on access) |
| Redirect | `${shortUrl}` (lowercased) | `{ longUrl, isFile, safeBrowsingExpiry }` | `REDIRECT_EXPIRY` (default 300 s) |
| Statistics | implementation-defined | implementation-defined | implementation-defined |
| Safe Browsing | `${url}` | `{ threatTypes, expireTime }` | 300 s |

The redirect cache MUST be invalidated on `Url` update.

### 21.3 S3

- File-upload bucket: `AWS_S3_BUCKET`. ACL toggled by `state` (§11).
- QR bulk output bucket: `BULK_GENERATION_BUCKET` (Lambda environment). Public-read.

---

## 22. Observability, Security, and Operations

### 22.1 Logging

- Winston logger. Level: `debug` in development, `info` in production. JSON output. Console transport with colorization.
- Morgan HTTP middleware with custom tokens:
  - `client-ip` — `req.headers['cf-connecting-ip']` → `req.ip` → `req.connection.remoteAddress`.
  - `redirectUrl` — value of `Location` header on 302 responses.
  - `userId` — `req.session?.user?.id ?? ''`.
- Format: `:client-ip - ":method :url HTTP/:http-version" :status ":redirectUrl" ":userId" :res[content-length] ":referrer" ":user-agent" :response-time ms`.

### 22.2 Datadog APM

- `dd-trace` initialized with `profiling: true, logInjection: true, runtimeMetrics: true`.
- HTTP integration sets `resource.name = "${method} ${host}${path}"`.
- Identified by `DD_SERVICE` (deployment-specific service name) and `DD_ENV`.

### 22.3 Datadog RUM (client)

- `@datadog/browser-rum` instrumented for the React SPA when configured.

### 22.4 StatsD metrics

All metrics MUST be prefixed `go.`. A complete list is in Appendix C. Metrics are emitted from request handlers and middleware via `dogstatsd.increment(metric, 1, 1, [tags])`.

### 22.5 Helmet / CSP

```
Content-Security-Policy:
  default-src 'self';
  style-src   'self' 'unsafe-inline' https://fonts.googleapis.com/;
  font-src    'self' https://fonts.gstatic.com/;
  img-src     'self' data: https://www.google-analytics.com/
              https://www.googletagmanager.com/
              https://${AWS_S3_BUCKET}/;
  script-src  'self' https://www.google-analytics.com/
              https://ssl.google-analytics.com/
              https://www.googletagmanager.com/
              https://*.browser-intake-datadoghq.com/
              https://www.datadoghq-browser-agent.com/;
  worker-src  blob:;
  connect-src 'self' https://www.google-analytics.com/
              https://stats.g.doubleclick.net/
              https://*.browser-intake-datadoghq.com/
              [+ CSP_REPORT_URI if set];
  frame-ancestors 'self';
  upgrade-insecure-requests;
```

If `CSP_ONLY_REPORT_VIOLATIONS=true` the header MUST be `Content-Security-Policy-Report-Only`.

All responses MUST set `Cache-Control: no-store`.

### 22.6 Rate Limiting

`/api/login/otp` is rate-limited per IP via `express-rate-limit`:

- `windowMs = 60_000`
- `max = OTP_RATE_LIMIT` (0 disables; default 0 in dev, implementation-defined positive in prod)
- `keyGenerator(req) = getIp(req)`
- `onLimitReached` MUST log a warning

No other endpoints are rate-limited.

### 22.7 Server timeouts

- `keepAliveTimeout = 65_000 ms` (must be < ALB idle timeout, set to 100 s by `.ebextensions/00-elb-timeout.config`).
- `headersTimeout = 66_000 ms` (must be > `keepAliveTimeout`).

### 22.8 Error handler and not-found fallbacks

```
errorHandler(err, req, res, next):
    if err.error?.isJoi:        res.badRequest(err.error.toString()); return
    if err.statusCode == 400 and err.type == 'entity.parse.failed':
                                 res.badRequest('Bad Request. JSON is malformed'); return
    res.status(500).render('500.error.ejs')
```

There are three not-found surfaces. Only one is part of the external contract (§27):

1. **External REST API 404 (`/api/v1/...`)** — JSON body, e.g., `{ message: 'Resource not found.' }`. Integrators rely on the JSON content type; the body schema is implementation-defined.
2. **Short-URL 404 (`GET /:shortUrl`)** — HTTP 404 from the redirect endpoint. The current implementation renders an HTML "link not found" page; a rewrite may render any HTML or JSON 404, so long as the status code is 404.
3. **SPA / internal API 404** — any other path. The current implementation renders the same HTML template as surface 2; a rewrite may handle this however it likes.

The process MUST listen for `unhandledRejection` and increment `ERROR_UNHANDLED_REJECTION`.

### 22.9 Health checks

There is no dedicated health endpoint. ALB performs TCP health checks against port 8080.

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
- Job items are independent: a failed item MUST mark its parent job FAILURE on the next aggregation, but other items may have already produced S3 artifacts. The completion email MUST report the partial state.
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

    fire_and_forget update_link_statistics(shortUrl, deviceClass(ua))
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
            sqs.send(SQS_BULK_QRCODE_GENERATE_START_URL, body=jobItem.params)
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

A conforming implementation MUST cover the following test categories.

### 25.1 Unit (`npm run test`, Jest)

- All Joi schemas reject malformed inputs.
- `isValidUrl`, `isValidShortUrl`, `isValidTag(s)`, `isPrintableAscii`, `isCircularRedirects`, `isBlacklisted` MUST be exercised against canonical positive and negative samples.
- `computeJobStatus`, `computeChangeSets`, `cookieEvict`, `getDeviceType` MUST be exercised against the spec examples.
- Mappers MUST be covered.
- Coverage threshold (statements) ≥ 24%.

### 25.2 Integration (`npm run test:integration`, Jest)

Runs against a live docker-compose stack (Postgres, Redis, LocalStack, MailDev). MUST cover:

- Full OTP login round-trip.
- URL create/update/transfer/list.
- Bulk upload with mocked SQS.
- Redirect path including Safe Browsing cache.
- Audit endpoint pagination.

### 25.3 End-to-end (`npm run test:e2e-headless`, TestCafe)

Chrome headless against `npm run dev`. MUST cover the user stories of §3 (login, create URL, edit, transfer, directory filter, transition page, link audit, API integration).

### 25.4 Continuous Integration

Lint + lockfile audit + unit + e2e + integration MUST run on every push and pull request. Production deploys are triggered by GitHub Release.

---

## 26. Implementation Checklist

### 26.1 Required for conformance

- [ ] All entities and indexes of §4 created on startup migration.
- [ ] All required env vars of §6.1 validated at startup with fail-fast.
- [ ] Email allowlist enforced as the **intersection** of `validator.isEmail` and minimatch with the documented flags.
- [ ] OTP flow with per-IP rate limit, bcrypt-hashed storage, 3-retry semantics, and Redis TTL.
- [ ] Session store with strict same-site cookies (cookie name is implementation-defined; see §27.6).
- [ ] API key flow with version-prefixed key strings and bcrypt-hashed suffix storage.
- [ ] Helmet CSP per §22.5, `Cache-Control: no-store` everywhere.
- [ ] Short URL CRUD with full URL/file/tag validation, history hooks, and redirect-cache invalidation.
- [ ] Redirect path with cache, replica fallback (gated), Safe Browsing recheck, transition page, click-stats fire-and-forget, GA hit.
- [ ] Bulk upload pipeline per §13 including all eight row-level validators and the BULK_VALIDATION_ERROR metric tags.
- [ ] QR endpoint with the layout and color rules of §14.
- [ ] Link statistics endpoint with replica reads.
- [ ] Link audit endpoint with the change-set algorithm of §16.2.
- [ ] Directory search with weighted tsvector and the `isEmail` mode rules of §17.2.
- [ ] External and admin v1 API gated by `FF_EXTERNAL_API`.
- [ ] All five error classes and the response shape of §23.2.

### 26.2 Recommended extensions

- [ ] Datadog APM, RUM, and StatsD wired up.
- [ ] LocalStack-based local development via `npm run dev`.
- [ ] Serverless deploy of the four Lambdas in Appendix A.

### 26.3 Operational validation

- [ ] OTP delivery verified end-to-end against a real SES sandbox in staging.
- [ ] Replica failover behavior measured.
- [ ] Bulk QR job processes 1000-row CSV within Lambda timeout (120 s).
- [ ] Safe Browsing API outage soft-fails (or fails closed) per the configured `SAFE_BROWSING_LOG_ONLY` policy.

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

- The storage backend itself (S3, GCS, on-prem, etc.).
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
| Locale loader path (`/locales/...`) | Tied to the SPA's i18next configuration. |
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

## Appendix A. Serverless Functions

Four Lambda functions complement the long-running server.

### A.1 `migrate-url-to-user`

Handler: `src/server/serverless/lambda-migrate-url-to-user/index.handler`. Memory: 128 MB.

Event:

```
{ shortUrl: string, toUserEmail: string }
```

Behavior: open a connection with `DATABASE_URL`, run `SELECT migrate_url_to_user($1, $2)` (target user MUST exist or be created by the procedure), return `{ Status: "URL successfully migrated." }`. On error, throw `"URL migration failed. ${error}"`.

### A.2 `migrate-user-links`

Handler: `src/server/serverless/lambda-migrate-user-links/index.handler`. Memory: 128 MB.

Event:

```
{ fromUserEmail: string, toUserEmail: string }
```

Behavior: `SELECT migrate_user_links($1, $2)`, return `{ Status: "URL successfully migrated. ${rowCount} rows affected" }`.

### A.3 `bulk-qrcode-generation`

Handler: `src/server/serverless/bulk-qrcode-generation/index.handler`. Memory: 2048 MB. Timeout: 120 s.

Triggered by SQS messages of the form `{ jobItemId, mappings: [{shortUrl, longUrl}, …] }`.

Environment: `DOMAIN` (used to build the human-readable short link encoded in each QR), `BULK_GENERATION_BUCKET`, `EB_CALLBACK_ENDPOINT`, `EB_CALLBACK_SECRET`.

Steps:

1. Build a CSV (header `Short URL,Original URL`) and upload to `${jobItemId}/generated.csv`.
2. For each `shortUrl`, render a branded QR per §14 and write `${tmp}/svg/${shortUrl}.svg`.
3. Zip the directory and stream to `${jobItemId}/generated_svg.zip`.
4. Repeat for PNG → `${jobItemId}/generated_png.zip`.
5. Remove `${tmp}/${jobItemId}` recursively.
6. POST to `EB_CALLBACK_ENDPOINT` with header `Authorization: Bearer ${EB_CALLBACK_SECRET}` and body `{ jobItemId, status: { isSuccess, errorMessage } }`.

On any step failure, the callback MUST be sent with `isSuccess=false` and the error message.

### A.4 `capture-ses-events`

Handler: `src/server/serverless/capture-ses-events/index.handler`. Memory: 128 MB.

Subscribed to an SNS topic carrying SES bounce/complaint/delivery events. Current behavior logs the SNS message; conforming implementations SHOULD parse the SES event, mark the relevant email as undeliverable, and suppress future sends.

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

All metrics MUST be prefixed `go.`.

| Metric | Tags | Source |
|--------|------|--------|
| `apikey.generate` | `isnew:bool` | API key creation. |
| `otp.generate.success` / `.failure` | — | `/api/login/otp`. |
| `otp.verify.success` / `.failure` | — | `/api/login/verify`. |
| `malicious_activity.file` | — | Cloudmersive positive. |
| `malicious_activity.link` | — | Safe Browsing positive. |
| `scan.file.failure` | — | Cloudmersive error. |
| `scan.link.failure` | — | Safe Browsing error. |
| `shortlink.clicks` | — | Redirect served. |
| `shortlink.create` | `source:CONSOLE/API/BULK`, `isfile:bool` | URL creation. |
| `user.new` | — | First-time user create. |
| `bulk.validation.error` | `acceptableLinkCount`, `validHeader`, `onlyOneColumn`, `isNotEmpty`, `isValidUrl`, `isNotBlacklisted`, `isNotCircularRedirect`, `noParsingError` | Per-row failure. |
| `bulk.hash.success` / `.failure` | — | Bulk persistence. |
| `job.start.success` / `.failure` | — | Per-item SQS dispatch. |
| `job.update.success` / `.failure` | — | Aggregate status recompute. |
| `job.email.success` / `.failure` | — | Job completion email. |
| `directory.search.domain` / `.email` | — | Directory queries. |
| `error.unhandled_rejection` | — | Process-level handler. |

---

## Appendix D. Locale Schema

The deployment ships exactly one English locale file. Its location is implementation-defined (the existing implementation places it under `public/locales/en/translation.json` and loads it via i18next). The key schema MUST be:

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

i18next is initialized with `lng: 'en'`, `fallbackLng: 'en'`, `whitelist: ['en']`, `interpolation.escapeValue: false` (React escapes separately).

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
