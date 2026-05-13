# GoGovSG Service Specification

This is a comprehensive, language-agnostic functional specification of **GoGovSG** — the official Singapore government link shortener. The document captures product behavior in enough detail that an implementer (human or LLM) can recreate the system without reading the existing source. Visual styling is explicitly out of scope; behavior, data, and contracts are not.

The document uses RFC 2119 keywords (`MUST`, `SHOULD`, `MAY`, `MUST NOT`) when specifying conformance requirements. Defaults, field schemas, and error categories are given as concrete values. Implementation-defined choices are flagged.

---

## Table of Contents

1. Problem Statement
2. Goals and Non-Goals
3. Deployment Branding
4. Configuration
5. Identifiers, Validation, and Constants
6. Authentication and Session Management
7. Short URL Lifecycle
8. Redirect Service
9. File Hosting
10. Threat Detection
11. Bulk Operations and Async Jobs
12. QR Code Generation
13. Statistics and Analytics
14. Audit Trail (Link History)
15. Public Directory Search
16. Client Application
17. External REST API (v1)
18. Email Delivery
19. Security Headers and Rate Limiting
20. Failure Model and Recovery
21. Backward Compatibility (External Surfaces Only)

Appendix A. Validation Rules Reference
Appendix B. Locale Schema
Appendix C. Public Surface Map

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

## 3. Deployment Branding

The system runs as a single deployment. Its public identity (name, hostnames, allowed email domain, on-page copy, QR-code styling) is supplied through runtime configuration rather than being hardwired in source.

The configurable identity surface consists of:

- **`OG_URL`** — the canonical origin URL (e.g., `https://go.gov.sg`). Used for circular-redirect prevention, trusted-referrer detection, and as the basis for constructing the full short link in QR codes.
- **`VALID_EMAIL_GLOB_EXPRESSION`** — the email-domain allowlist (e.g., `*.gov.sg`).
- **`AWS_S3_BUCKET`** — the file-hosting bucket name. In production this is also the hostname of the file-serving domain (e.g., `file.go.gov.sg`), making the canonical file URL `https://${AWS_S3_BUCKET}/${shortUrl}.${ext}`. See §9.
- **Display name** — the human-readable service name (e.g., "Go.gov.sg") returned in API responses and shown in templated HTML (transition page, 404 page).
- **Locale strings** — the single English copy bundle loaded by the client (§16.8, Appendix B).
- **QR-code brand color and logo** — the dark color used when rendering QR codes (§12) and the centered logo overlay.

A reimplementation that needs only one deployment MAY hard-code these values as long as the External REST API contract (§21.4) and the file URL shape (§21.3) remain configurable through deployment, since they affect URLs already in the wild.

---

## 4. Configuration

The deployment MUST supply the following configuration. Specific environment-variable names appear in this document because they are referenced by name from other sections; the values are functional inputs, not a contract about how they are loaded.

**Identity**

- `OG_URL` — origin URL of the service (e.g., `https://go.gov.sg`). Used for circular-redirect prevention, trusted-referrer detection, and as the basis for building the full short link encoded in QR codes.
- `VALID_EMAIL_GLOB_EXPRESSION` — glob pattern that defines which email addresses may sign in. Extended-glob, globstar, brace, and negation expansions MUST be disabled.
- `AWS_S3_BUCKET` — public hostname (and storage bucket) for hosted files. See §9.
- Display name — human-readable service name returned in API responses and rendered in templated HTML.

**Secrets**

- `SESSION_SECRET` — used to sign session tokens and the visit-tracking cookie.
- `API_KEY_SALT` — salt for the password-hashing function used on API key suffixes.

**Functional defaults**

| Setting | Default | Purpose |
|---------|---------|---------|
| `OTP_EXPIRY` | 300 s | OTP validity window. |
| `OTP_RATE_LIMIT` | implementation-defined | OTP requests per IP per minute. May be 0 (disabled) for local development. |
| `REDIRECT_EXPIRY` | 300 s | Redirect-cache TTL. |
| `COOKIE_MAX_AGE` | 24 h | Session cookie lifetime. |
| `SALT_ROUNDS` | 10 | Password-hash work factor for OTPs and API keys. |
| `BULK_UPLOAD_MAX_NUM` | 1000 | Maximum URLs per bulk CSV. |
| `BULK_UPLOAD_RANDOM_STR_LENGTH` | 8 | Generated short-URL length for bulk-created links. |
| `API_LINK_RANDOM_STR_LENGTH` | 8 | Generated short-URL length for API-created links. |
| `BULK_QR_CODE_BATCH_SIZE` | 1000 | URLs per background-worker batch. |
| `JOB_POLL_INTERVAL` | 5000 ms | Server-side long-poll interval. |
| `JOB_POLL_ATTEMPTS` | 12 | Server-side long-poll attempts before responding 408. |
| `API_KEY_VERSION` | `v1` | Version component of API key string. |
| `USER_COUNT`, `CLICK_COUNT`, `LINK_COUNT` | static counters for the landing page. |

**Feature flags**

- `FF_EXTERNAL_API` (default `false`) — gates `/api/v1/*` and `/api/v1/admin/*`.
- `FF_USE_REPLICA_FOR_REDIRECTS` (default `false`) — read redirect lookups from a replica when configured.
- `ACTIVATE_BULK_QR_CODE_GENERATION` (default `false`) — master switch for the bulk QR pipeline.
- `SAFE_BROWSING_LOG_ONLY` (default `false`) — log URL-threat detections but do not block.

**Optional integrations**

- `SAFE_BROWSING_KEY` — credential for the URL threat-scanning service (§10.1).
- `CLOUDMERSIVE_KEY` — credential for the antivirus service (§10.2).
- `GA_TRACKING_ID` — web-analytics property identifier.
- `ADMIN_API_EMAILS` — comma-separated emails granted admin scope on the External REST API.

**User-facing copy**

- `LOGIN_MESSAGE` — banner on the login page.
- `USER_MESSAGE` — banner on the user dashboard.
- `ANNOUNCEMENT_TITLE`, `ANNOUNCEMENT_SUBTITLE`, `ANNOUNCEMENT_MESSAGE`, `ANNOUNCEMENT_URL`, `ANNOUNCEMENT_IMAGE`, `ANNOUNCEMENT_BUTTON_TEXT` — content for an optional logged-in announcement modal. The modal renders only when `ANNOUNCEMENT_MESSAGE` is truthy.
- `ROTATED_LINKS` — comma-separated short URLs featured on the landing page.

### 4.1 Cookies

Session cookies MUST be `httpOnly`, `sameSite: 'strict'`, `secure` in production, and live for `COOKIE_MAX_AGE`. A separate visit-tracking cookie (conventionally `visits`) MUST be `maxAge` = 7 days, signed using `SESSION_SECRET`, and capped in serialized size (default 2000 bytes) by LRU eviction so it does not grow unboundedly across many short-URL visits.

---

## 5. Identifiers, Validation, and Constants

### 5.1 Short URL

- Pattern: `/^[a-zA-Z0-9-]+$/`.
- Used as the primary identifier of a `Url` entity and as the redirect-cache key (lowercased internally).
- For auto-generation: draw characters uniformly at random from the alphabet `0123456789abcdefghijklmnopqrstuvwxyz` using a cryptographically strong random source. Length is `BULK_UPLOAD_RANDOM_STR_LENGTH` (default 8) for bulk-created links or `API_LINK_RANDOM_STR_LENGTH` (default 8) for API-created links. On collision, retry.

### 5.2 Long URL

Validation is centralized in `src/shared/util/validation.ts` and MUST apply on both client and server.

- MUST be a fully-qualified URL with `https:` scheme. Plain hostnames are rejected.
- MUST have a valid TLD; IP addresses are rejected.
- Validation rejects: non-`https:` schemes, schemeless URLs, hostless URLs, URLs with underscores in the hostname, URLs with a trailing dot, URLs lacking a valid TLD. URLs containing userinfo (`https://user:pass@host/...`) MAY be permitted.
- MUST NOT be circular: the URL's hostname MUST NOT resolve to the service origin (`OG_URL` hostname).
- MUST NOT match the blacklist (substring blocklist sourced from `src/server/resources/blacklist`).

### 5.3 Email

- MUST pass `validator.isEmail()` with `{ allow_utf8_local_part: false }`.
- MUST be lowercased and trimmed before storage and before pattern matching.
- MUST satisfy a glob match against `VALID_EMAIL_GLOB_EXPRESSION` where extended-glob, globstar, brace, and negation expansions are all disabled.

### 5.4 Tag

- Pattern: `/^[A-Za-z0-9_-]+$/`, length ≤ 25.
- At most 3 tags per link, no duplicates within a link.
- `tagKey` is the lowercased `tagString`.

### 5.5 Description

- Length ≤ 200, printable ASCII only (`/^[\x20-\x7F]*$/`).

### 5.6 File

- Max single upload size: 20 MiB.
- Max CSV bulk size: 5 MiB.
- Allowed extensions (case-insensitive): `avi, bmp, csv, docx, dwf, dwg, dxf, gif, jpeg, jpg, mpeg, mpg, ods, pdf, png, pptx, rtf, tif, tiff, txt, xlsx, zip`.
- MIME type MUST be detected by inspecting the file's byte content (i.e., magic-number sniffing) rather than trusting client-supplied MIME headers. If sniffing yields no result, fall back to the filename extension. The following extensions MUST be mapped manually because magic-number sniffing does not yield a useful MIME for them: `csv → text/csv`, `dwf → application/x-dwf`, `dxf → application/dxf`.

### 5.7 Constants

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

## 6. Authentication and Session Management

### 6.1 OTP Login Flow

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
5. Send the unhashed OTP to the supplied email via the configured email transport (§18). The email body MUST include the OTP, the requester's IP, and the deployment's display name.
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

### 6.2 API Key Authentication

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

### 6.3 Session Management

Authenticated sessions associate a session token (carried as a cookie) with the principal `{ user: { id, email } }`. The session store technology is implementation-defined; it MUST:

- Persist sessions across process restarts.
- Allow per-session TTL of `COOKIE_MAX_AGE`.
- Support explicit destruction on logout.

Cookie attributes are listed in §4.1. The session cookie name is implementation-defined (the existing implementation uses `gogovsg`).

**`GET /api/logout`** destroys the current session and returns `200 { message: "Logged out" }`.

A session guard MUST reject requests lacking an authenticated session with 401, and on success MUST make the authenticated `userId` available to downstream handlers, so handlers can treat session and API key auth uniformly.

---

## 7. Short URL Lifecycle

### 7.1 Creation (`POST /api/user/url`)

Authentication: session.

Multipart accepted (file upload) or JSON. Validation:

```
shortUrl: string (required, §5.1)
longUrl?: string  // XOR with file
file?: UploadedFile
tags?: string[]   // each per §5.4
description?: string (per §5.5)
contactEmail?: string (per §5.3)
```

Exactly one of `longUrl` or `file` MUST be supplied. Multiple files MUST return 422.

Middleware chain:

1. Multipart upload acceptance with `MAX_FILE_UPLOAD_SIZE` limit.
2. Form-data preprocessing: place the file under a known request field; parse the `tags` JSON if it is a string.
3. Single-file check: exactly one file may be present (or none).
4. File extension/MIME validation (§5.6).
5. File antivirus scan (§10.2).
6. URL threat-scan (§10.1).
7. `userController.createUrl`.

Service behavior (`UrlManagementService.createUrl`):

1. Verify user exists.
2. `urlRepository.isShortUrlAvailable(shortUrl)` — return 400 `AlreadyExistsError` if taken.
3. If file, upload to the object store at key `${shortUrl}.${ext}` and store `longUrl = ${file domain}/${shortUrl}.${ext}`; set `isFile = true`.
4. Insert `urls` row (in a transaction, so `afterCreate` writes `url_clicks` row and `url_history` row).
5. Associate tags (upserting tag entities per §7.7).
6. `safeBrowsingExpiry = now + DEFAULT_URL_SCAN_RESULT_EXPIRY_SECONDS` if the URL was scanned clean.
7. Emit `SHORTLINK_CREATE` with tags `source` and `isfile`.
8. Return the persisted `StorableUrl`.

### 7.2 Update (`PATCH /api/user/url`)

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

### 7.3 Ownership Transfer (`PATCH /api/user/url/ownership`)

```
shortUrl: string (required)
newUserEmail: string (required, per §5.3)
```

1. Verify current user owns `shortUrl`. Return 400 `AlreadyOwnLinkError` if `newUserEmail` equals the current user's email.
2. `findOrCreateWithEmail(newUserEmail)`.
3. Update `urls.userId` (transaction, history hook fires).
4. Send notification email to the new owner.

### 7.4 Deactivation

There is no explicit user-facing delete. Setting `state = INACTIVE` removes the link from redirects. The system MAY also auto-deactivate via §10.1.5 (malicious-link detection at redirect time).

### 7.5 Listing (`GET /api/user/url`)

This endpoint is consumed only by the SPA (§16.5). It is **not** part of the External REST API and MAY be redesigned during a rewrite (see §21.6). The shape below describes the current implementation.

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

### 7.6 Tag Autocomplete (`GET /api/user/tag`)

```
searchText: string (required, length ≥ MIN_TAG_SEARCH_LENGTH, valid tag form)
limit: int (required)
```

Returns `string[]` — tagStrings whose `tagKey` matches `${searchText}%` on the user's own URLs.

### 7.7 Tag Upsert (Transactional)

When a write supplies tags, the server MUST:

1. For each tag string, attempt `findOrCreate` on `tags` keyed by `tagKey`. On unique-constraint races, refetch.
2. Replace the link's `url_tag` associations with the resolved set.
3. Recompute `urls.tagStrings` as the semicolon-joined `tagString`s of the new set.

---

## 8. Redirect Service

### 8.1 Request Path

Route: `GET /:shortUrl`. The endpoint accepts a short URL slug matching `[a-zA-Z0-9-]+`.

Two behaviors of this endpoint are **externally binding** because they affect resolution of links already published in the wild (see §21.2):

- **Trailing-character tolerance**: the captured slug MAY be followed by a single trailing non-slug character (e.g., a `.` appended by an SMS or email client). `/foo.` MUST resolve to the same short URL as `/foo`. The path-matching mechanism is implementation-defined; only the resolution behavior is normative.
- **Case-insensitive lookup**: `/Foo`, `/FOO`, and `/foo` MUST resolve identically. The existing implementation lowercases the captured slug before cache and database lookup. Stored `shortUrl` values in the canonical deployment are effectively lowercase.

Everything else in this section describes the current implementation; rewrites may diverge.

The redirect handler renders the transition page from a server-side template. The current implementation loads its in-page JavaScript from a separate route — `GET /assets/transition-page/js/redirect.js` (a server-rendered `text/javascript` response parameterized with the web-analytics tracking ID and the event category/action names used for the `loaded` and `proceeded` events). This indirection exists because the tracking ID is a runtime configuration value. A reimplementation that inlines the script, or uses a different transition page entirely, is acceptable; this auxiliary route is not part of the external contract.

The current implementation uses a signed visit-tracking cookie (conventionally named `visits`) to suppress the transition page on repeat visits. The cookie name and the mechanism are implementation details; a rewrite may use a different scheme or no mechanism at all.

### 8.2 Resolution Algorithm

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

### 8.3 Response

- `RedirectType.Direct`: HTTP 302 with `Location: longUrl`.
- `RedirectType.TransitionPage`: HTTP 200 rendering `transition-page.ejs` with `escapedLongUrl`, `rootDomain` of the destination, and `gaTrackingId`.

Both response paths MUST update the visit-tracking cookie and trigger the side effects in §8.5.

### 8.4 Crawler and Referrer Heuristics

`isCrawler(userAgent)`: parse the user-agent string; if any of the standard fields (browser name, rendering-engine name, OS name) is missing → crawler. Any user agent whose name matches the bot regex `/bot|facebookexternalhit|Facebot|Slackbot|TelegramBot|WhatsApp|Twitterbot|Pinterest|Postman|url|Google-PageRenderer/` MUST be classified as device `'others'` for statistics purposes.

`fromTrustedReferrer(referrer)`: parse the referrer; trusted iff its origin equals `OG_URL`'s origin. Parse failures are treated as untrusted.

### 8.5 Side Effects

Each non-crawler redirect MUST record a click against the resolved short URL. The recording covers four aggregates — a total counter, a device-class counter (mobile / tablet / desktop / others, derived from the user agent), a per-day counter, and a per-(weekday, hour) counter. The day/weekday/hour dimensions MUST be expressed in **Asia/Singapore** local time. Click recording MUST NOT block the redirect response; failures to record MUST NOT cause user-visible errors.

When web analytics is configured, each non-crawler redirect SHOULD also be reported to the analytics service. Crawler redirects MUST be excluded from both click counts and analytics.

---

## 9. File Hosting

Files attached to short URLs are stored in the object store under the bucket/container identified by `AWS_S3_BUCKET`:

- Object key: `${shortUrl}.${ext}`.
- ACL: public-read when `state = ACTIVE`, private when `state = INACTIVE`. Cache-Control on uploaded objects is `no-cache`.
- `longUrl` for a file URL MUST be constructed as `${fileURLPrefix}${AWS_S3_BUCKET}/${key}`.
  - In production, `fileURLPrefix = 'https://'`, so the URL is `https://${AWS_S3_BUCKET}/${shortUrl}.${ext}`. The bucket name is conventionally also the public hostname (e.g., `file.go.gov.sg`), making the canonical file URL `https://${FILE_HOSTNAME}/${shortUrl}.${ext}`. **External integrators and stored history depend on this URL shape; see §21.3.**
  - In development, `fileURLPrefix` is the local object-store emulator's `ACCESS_ENDPOINT` followed by `/`.
- The reverse derivation `getKeyFromLongUrl(longUrl)` MUST extract the key as the final path segment.
- Files MUST pass the extension/MIME and antivirus checks of §10.2 before upload.
- Replacing a file (on edit) MUST overwrite the same key. The system MUST NOT permit changing whether a URL is a file URL after creation, nor changing its `longUrl` independently of the underlying object.

For development, a local object-store emulator MAY be exposed at `BUCKET_ENDPOINT`.

---

## 10. Threat Detection

### 10.1 URL Threat Scanning

The server integrates with an external **URL threat-scanning service** that, given a URL, returns whether the URL is known to host malware, social-engineering content, unwanted software, or related threats. The choice of provider is implementation-defined; the configuration uses `SAFE_BROWSING_KEY` as a generic credential. The integration MUST cover the conceptual threat classes:

- malware
- social engineering / phishing
- unwanted software
- extended-coverage social engineering (lower-confidence phishing detection)

#### 10.1.1 Cache

Threat-scan results MUST be cached by URL with a short TTL (default 300 s). Cache hits return the cached threat verdict; cache misses query the external service. The cache is logically separate from the redirect cache and other caches.

Additionally, the *clean* verdict MUST be cached on the `Url` itself via `safeBrowsingExpiry` (default 24 h) so that the redirect path (§8.2) avoids re-scanning on every hit.

#### 10.1.2 When the service is unconfigured

If no threat-scanning credential is configured, the server MUST log a warning at startup and treat all URLs as clean. Threat scanning is OPTIONAL infrastructure.

#### 10.1.3 Log-only mode

When `SAFE_BROWSING_LOG_ONLY=true`, a detected threat MUST be logged and the `MALICIOUS_ACTIVITY_LINK` counter incremented, but the verdict returned to callers MUST be "clean". This mode permits dry-run deployment of the scanner.

#### 10.1.4 Bulk scan

A bulk scan over an array of URLs MUST evaluate each URL (concurrency permitted) and return true if any URL is a threat. Implementations MAY short-circuit on the first positive.

#### 10.1.5 At redirect

§8.2 specifies that an expired `safeBrowsingExpiry` triggers a re-scan and that a threat-positive scan MUST:

- Deactivate the link (`state = INACTIVE`).
- Email the owner.
- Return 404 to the caller (parity with not-found, avoiding leakage).

### 10.2 File Threat Scanning

The server integrates with an external **antivirus service** that, given a file's bytes, returns whether the file contains a virus or is password-protected. The choice of provider is implementation-defined; the configuration uses `CLOUDMERSIVE_KEY` as a generic credential. The scanner SHOULD be configured to refuse executables, scripts, and structurally invalid files.

File scan pipeline:

1. If no antivirus credential is configured, skip the scan.
2. Call the antivirus service. On error, emit `SCAN_FAILED_FILE` and return 500 `"Your file could not be scanned at this moment…"`.
3. If the file is password-protected, return 400 `"Cannot upload password-protected files."`.
4. If the file contains a virus, emit `MALICIOUS_ACTIVITY_FILE` and return 400 `"File is likely to be malicious."`.

### 10.3 File Extension/MIME Validation

Before invoking the antivirus service, the server MUST:

1. Detect the file's MIME type from its byte content (not from the upload's declared header); fall back to the filename extension; apply the manual overrides in §5.6.
2. Reject the upload with 415 `"File type disallowed."` if the resolved extension is empty or not in the allowlist (default §5.6).
3. Stamp the resolved MIME on the in-flight upload record so downstream consumers (object-store upload, antivirus call) see the canonical type.

---

## 11. Bulk Operations and Async Jobs

### 11.1 Bulk Upload (`POST /api/user/url/bulk`)

Multipart upload. File size ≤ `MAX_CSV_UPLOAD_SIZE` (5 MiB). Optional `tags` JSON.

Pipeline:

1. Multipart upload acceptance with `MAX_CSV_UPLOAD_SIZE` limit.
2. File extension/MIME validation restricted to `csv`.
3. File antivirus scan.
4. Parse and validate the CSV per §11.1.1–§11.1.2.
5. URL threat-scan on every parsed URL (bulk variant; §10.1.4).
6. Generate short URLs and persist the bulk URL mappings.
7. If `ACTIVATE_BULK_QR_CODE_GENERATION === 'true'`, dispatch the async-job pipeline (§11.2).

#### 11.1.1 CSV format

- Header row MUST be exactly `BULK_UPLOAD_HEADER = "Original links to be shortened"`.
- One column per row.
- Empty rows skipped.
- Rows ≤ `BULK_UPLOAD_MAX_NUM` (default 1000).

#### 11.1.2 Row-level validation

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

#### 11.1.3 Short URL generation

For each accepted long URL, call `generateShortUrl(BULK_UPLOAD_RANDOM_STR_LENGTH)`. Collision detection runs inside `UrlManagementService.bulkCreate`.

#### 11.1.4 Persistence

`UrlManagementService.bulkCreate({ userId, urlMappings, tags })`:

- Wrap in transaction.
- Insert `urls` rows with `source = BULK`, `state = ACTIVE`, `safeBrowsingExpiry = now + 24h`.
- Apply the supplied tags to all rows.
- The `afterBulkCreate` hook MUST write `url_history` rows and create `url_clicks` rows.

#### 11.1.5 Response

```
200 { count: number, job?: Job }
```

`job` is present iff QR generation was enabled.

### 11.2 Job model

A bulk upload that requests QR generation produces a **job** with a stable UUID and a `status` of `IN_PROGRESS`, `SUCCESS`, or `FAILURE`. The job is decomposed into one or more **job items**, each carrying a chunk of mappings (default `BULK_QR_CODE_BATCH_SIZE = 1000`) and a stable `jobItemId` of the form `${job.uuid}/${batchIndex}`. The `jobItemId` is used as the artifact key in object storage, so re-running an item produces the same paths.

A job's aggregate status is derived from its items: `FAILURE` if any item failed, `IN_PROGRESS` if any item is still running, `SUCCESS` otherwise. When a job leaves `IN_PROGRESS`, the server MUST notify the owner by email.

### 11.3 Job status polling

**`GET /api/user/job/status?jobId={id}`** (session) — long poll. The server waits up to `JOB_POLL_ATTEMPTS × JOB_POLL_INTERVAL` for the job to leave `IN_PROGRESS`. On a terminal status it returns:

```
{ job: { id, uuid, status, ... }, jobItemUrls: string[] }
```

where `jobItemUrls` is one URL per item, derived from the artifact key. On poll exhaustion it responds 408.

**`GET /api/user/job/latest`** returns the latest job for the user without long-polling.

---

## 12. QR Code Generation

### 12.1 Single-URL Endpoint (`GET /api/qrcode`)

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
   - Dark color from the deployment's brand color (§12.3).
4. Compose onto a 1000-pixel-wide canvas:
   - 85 px top margin.
   - 800×800 QR centered.
   - A configurable brand logo SVG overlaid at the center of the QR.
   - 85 px between QR and text.
   - Text: the human-readable short link (e.g., `go.gov.sg/foo`) in IBM Plex Sans 32 px, line height 1.35, anchor middle, wrapped every 36 characters.
   - 85 px bottom margin after final line.
5. If `format` is `image/png` or `image/jpeg`, rasterize the rendered canvas to the target format.
6. Respond with the correct `Content-Type` and header `Filename: ${OG_URL_HOST}/${shortUrl}`. Body is the binary buffer.

### 12.2 Bulk QR Pipeline

Enabled iff `ACTIVATE_BULK_QR_CODE_GENERATION === 'true'`. See §11.2 for the dispatch protocol.

For each `jobItemId = "${job.uuid}/${i}"`, the worker produces three artifacts in the object store under the prefix `${jobItemId}/`:

- `${jobItemId}/generated.csv` — header `"Short URL,Original URL"`, one row per mapping.
- `${jobItemId}/generated_svg.zip` — `${shortUrl}.svg` per mapping.
- `${jobItemId}/generated_png.zip` — `${shortUrl}.png` per mapping.

### 12.3 Color and Logo

QR codes MUST be rendered with a single brand dark color and a single brand logo, both implementation-defined. The chosen color and logo are deployment-wide constants. Both the synchronous QR-code endpoint (§12.1) and the bulk-generation pipeline (§12.2) MUST use the same color and logo so that a single QR rendered ad hoc is visually indistinguishable from one rendered as part of a bulk job.

---

## 13. Statistics and Analytics

### 13.1 Global Statistics (`GET /api/stats`)

Public, unauthenticated. Returns static counters from environment:

```
{ userCount: USER_COUNT, clickCount: CLICK_COUNT, linkCount: LINK_COUNT }
```

These values do not refresh dynamically; they are updated by redeploy.

### 13.2 Per-Link Statistics (`GET /api/link-stats`)

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

---

## 14. Audit Trail (Link History)

### 14.1 Endpoint (`GET /api/link-audit`)

Session-authenticated. Query:

```
url:    string (required, short URL)
offset?: int (default 0)
limit?:  int (default 10)
```

The user MUST own the link; otherwise return 404 `"User does not own this short url"`.

### 14.2 Algorithm

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

## 15. Public Directory Search

### 15.1 Endpoint (`GET /api/directory/search`)

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

### 15.2 Query Modes

- **Email mode** (`isEmail === 'true'`): the `query` is interpreted as an email or email substring. The query MUST be reduced to the part after `@` if present. The server performs a case-insensitive substring match against `User.email`. Emit `DIRECTORY_SEARCH_EMAIL`.
- **Text mode** (`isEmail === 'false'`): `@` MUST be stripped from the query to avoid leaking email-domain enumeration through this path. The server performs a full-text search against `Url` fields using English-language tokenization, with weighted relevance: `shortUrl` matches rank highest, `longUrl` matches medium, `description` matches lowest. The exact weight values are implementation-defined; what matters is the *ordering*. Emit `DIRECTORY_SEARCH_DOMAIN`.

### 15.3 Sort

- `POPULARITY` → results ordered by total-clicks aggregate (descending).
- `RECENCY` → results ordered by `Url.createdAt` (descending).

### 15.4 Response

```
{
  urls: [{ shortUrl: string, email: string, state: 'ACTIVE'|'INACTIVE', isFile: boolean }],
  count: number
}
```

---

## 16. Client Application

The browser client is a single-page application. Internal framework choice, state management, routing strategy, and UI toolkit are implementation-defined. The behaviors below are the functional contract.

### 16.1 Routes

| Path | Subapp | Access | Behavior |
|------|--------|--------|----------|
| `/` | home | anonymous | If logged in, redirect to `/user`. Otherwise render landing. |
| `/login` | login | anonymous | If logged in, redirect to `/user`. |
| `/user` | user dashboard | authenticated | If anonymous, redirect to `/login` and remember the originally-requested location. |
| `/directory` | directory | public | Public search. |
| `/apiintegration` | API integration | authenticated | Private route. |
| `/404/:shortUrl` | 404 | public | "Link not found" page. |

After a successful OTP verification, the login subapp navigates back to the originally-requested location if one was remembered, else `/user`.

### 16.2 Cross-Cutting Behavior

- Every page mount fires a web-analytics page-view event when web analytics is configured.
- A global notification surface displays transient error, success, and info messages.
- All API requests include credentials and are sent same-origin.
- The login page displays `LOGIN_MESSAGE` when present; the dashboard displays `USER_MESSAGE` when present.

### 16.3 Home

On mount: check session via `GET /api/login/isLoggedIn` (navigate to `/user` if logged in), fetch rotating-link payload via `GET /api/links`, and fetch landing-page counters via `GET /api/stats`.

### 16.4 Login

The login form moves through five states: **email-ready**, **email-pending**, **OTP-ready**, **OTP-pending**, **resend-disabled**.

```
email-ready  --submit-> email-pending --ok-> OTP-ready
                                     --err-> email-ready (error toast)
OTP-ready    --submit-> OTP-pending   --ok-> logged-in (navigate to /user)
                                     --err-> OTP-ready  (error toast)
OTP-ready    --resend-> email-pending -> OTP-ready -> resend-disabled (20 s) -> OTP-ready
```

On mount the client fetches `GET /api/login/isLoggedIn` and `GET /api/login/emaildomains`, then constructs an email validator that combines a glob match against the returned pattern with a structural email check (§5.3).

The email input is lowercased on every keystroke. The error message `"This doesn't look like a valid ${domain} email."` is shown only when the field has a value that fails the validator. The OTP is not validated client-side beyond non-emptiness.

### 16.5 User Dashboard

The dashboard lists the signed-in user's links in a table with pagination, sort, filtering, and search. On mount the client fetches the user's URLs (with the current table configuration), the user-message banner, the announcement-modal content, and the latest async-job status.

If the user has no URLs and no filter is active, the client renders an empty state with a "Create link" CTA. Otherwise it renders the table.

**Table configuration.** The table exposes:

- Page size and current page.
- A search input (debounced before issuing the request).
- A tag filter (semicolon-separated) — mutually exclusive with the search input.
- Optional filters by `isFile` and `state`.
- Sort by `createdAt` (default) or `clicks`, ascending or descending.

The active configuration is reflected in the URL query string so the page is shareable.

**Create-link modal.** Three modes:

- **URL** — short URL + long URL + optional tags. Submits to `POST /api/user/url` as JSON.
- **File** — short URL + file + optional tags. Submits to `POST /api/user/url` as multipart.
- **Bulk** — CSV + optional tags. Submits to `POST /api/user/url/bulk`.

The long-URL input strips a leading `https://` for display and re-adds it before submission. The short URL is validated against `/^[a-z0-9-]+$/`. Tags are validated per §5.4. Errors are surfaced as toasts categorised by the `MessageType` in the response body.

On bulk success the status bar (below) starts tracking the new job.

**Link drawer.** Clicking a row opens a drawer with the following features, each mapped to one API call:

| Feature | API |
|---------|-----|
| Edit long URL | `PATCH /api/user/url` |
| Edit description + contact email | `PATCH /api/user/url` |
| Edit tags | `PATCH /api/user/url` |
| Replace file | `PATCH /api/user/url` (multipart) |
| Toggle Active/Inactive | `PATCH /api/user/url` |
| Transfer ownership | `PATCH /api/user/url/ownership` |
| View link history | `GET /api/link-audit` |
| View statistics | `GET /api/link-stats` |
| Download QR code | `GET /api/qrcode` |

**Bulk-QR status bar.** While a bulk job is running, the dashboard polls `GET /api/user/job/status`. Status messages:

- `IN_PROGRESS` (info): `"QR codes generation in progress. Please wait to download your QR codes"`.
- `SUCCESS`: `"QR codes successfully generated. Please download your QR codes here or via email"`, with per-format download links.
- `FAILURE`: `"QR codes failed to generate. Please try again"`.

**Announcement modal.** If announcement configuration is present (§4), the dashboard shows a dismissible modal with the configured title, subtitle, image, message, and CTA button on first mount.

### 16.6 Directory

Search input is debounced. Each commit updates the page's query string with `query`, `order`, `rowsPerPage`, `currentPage`, `state`, `isFile`, `isEmail`. Any change to those query parameters re-issues `GET /api/directory/search`.

Before sending, the query is lowercased and trimmed. If `isEmail` is true, only the substring after `@` is sent; otherwise `@` characters are stripped entirely (to prevent the text search from leaking email-domain enumeration).

A "Reset filters" action clears all filters but preserves the query and `isEmail` mode and resets pagination to page 0. Changing rows-per-page MUST also reset to page 0.

### 16.7 API Integration

On mount: `GET /api/user/hasApiKey` to determine whether the user already has a key.

"Generate API Key" calls `POST /api/user/apiKey` and displays the returned key in a modal. The full key is shown exactly once; the client MUST NOT persist it.

### 16.8 Internationalization

The client loads a single English locale bundle at startup. The location of the bundle and the loader library are implementation-defined. HTML escaping MUST be performed by the rendering layer so that translation values may safely contain inline HTML. The locale schema is in Appendix B.

---

## 17. External REST API (v1)

Gated by `FF_EXTERNAL_API === 'true'`. Mounted under `/api/v1`. Authenticated by API key (§6.2).

### 17.1 User-scope

| Route | Body / Query |
|-------|--------------|
| `GET /api/v1/urls` | Same query as `/api/user/url` minus tag filtering. Returns `UrlsPaginated`. |
| `POST /api/v1/urls` | `{ longUrl (required, HTTPS), shortUrl? (auto-gen length `API_LINK_RANDOM_STR_LENGTH`) }`. Returns mapped `StorableUrl`. `source = API`. |
| `PATCH /api/v1/urls/:shortUrl` | `{ longUrl?, state? }`. File editing NOT allowed via API. |

API responses use a thin DTO mapping that omits internal fields (e.g., `safeBrowsingExpiry`, `userId`).

### 17.2 Admin-scope

Mounted under `/api/v1/admin/`. Additional middleware: `apiKeyAdminAuthMiddleware`.

| Route | Body |
|-------|------|
| `POST /api/v1/admin/urls` | `{ email (target user), longUrl, shortUrl? }`. Server: `findOrCreateWithEmail(email)`; create URL under the target user. If `userId !== targetUser.id`, transfer ownership immediately. |

### 17.3 QR-job callback

A callback endpoint is exposed under admin scope to allow background workers to report job-item completion. Its exact path, request shape, and authentication are implementation-defined; only the abstract callback contract of §11.2 is normative.

---

## 18. Email Delivery

A single mailer abstraction sends:

- OTP emails (login). Subject and body include the OTP and requester IP.
- Job-completion emails (bulk QR). Body includes the per-item artifact URLs.
- Ownership-transfer notifications.
- Malicious-link deactivation notices.

The email transport is implementation-defined: any mechanism capable of delivering plain-text or HTML mail to arbitrary recipients is acceptable (transactional email provider, on-prem SMTP, third-party email API). The implementation MAY define a fallback transport that is attempted when the primary transport fails.

Bounce/complaint capture, if supported by the chosen transport, MAY be performed out-of-band by an auxiliary worker that listens for delivery-status notifications.

---

## 19. Security Headers and Rate Limiting

### 19.1 Content Security Policy

The server MUST emit a Content-Security-Policy header restricting the origins from which the client may load scripts, styles, fonts, images, and connect targets. A conforming default-deny policy is:

```
default-src 'self';
style-src   'self' 'unsafe-inline' <web-font origin>;
font-src    'self' <web-font origin>;
img-src     'self' data: <web-analytics origin> <file-hosting origin>;
script-src  'self' <web-analytics origin>;
worker-src  blob:;
connect-src 'self' <web-analytics origin> [+ CSP_REPORT_URI if set];
frame-ancestors 'self';
upgrade-insecure-requests;
```

Allow-listed origins MUST be expanded to cover whichever external services (web analytics, file hosting, web-font provider, etc.) are configured. When `CSP_ONLY_REPORT_VIOLATIONS=true` the header MUST be sent as `Content-Security-Policy-Report-Only` instead.

Additional baseline headers (HSTS, X-Content-Type-Options, X-Frame-Options, etc.) SHOULD be set per current web-security best practice.

All responses MUST set `Cache-Control: no-store`.

### 19.2 Rate Limiting

`POST /api/login/otp` MUST be rate-limited per client IP. Default window 60 s, default cap `OTP_RATE_LIMIT` requests per window (0 disables). Overflowing requests MUST receive HTTP 429. No other endpoints are rate-limited.

---

## 20. Failure Model and Recovery

### 20.1 Custom error classes

| Class | HTTP | Source |
|-------|------|--------|
| `NotFoundError` | 404 | Missing URL/user/job. |
| `AlreadyExistsError` | 400 (`MessageType.ShortUrlError`) | Short URL collision on create. |
| `AlreadyOwnLinkError` | 400 | Ownership transfer to current owner. |
| `InvalidOtpError` | 401 | OTP mismatch (carries retries-left). |
| `InvalidUrlUpdateError` | 400 | Disallowed update (e.g., file → URL). |

### 20.2 Standard response

```
{ ok?: boolean, message: string, type?: MessageType }
```

`MessageType` ∈ `{ 'ShortUrlError', 'LongUrlError', 'FileUploadError' }`.

### 20.3 Status codes

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

### 20.4 Partial-state recovery

- Bulk creates run inside a single transaction; failures roll back fully. The CSV is not partially applied.
- Job items are independent: a failed item MUST mark its parent job FAILURE on the next aggregation, but other items may have already produced object-store artifacts. The completion email MUST report the partial state.
- A redirect-time Safe Browsing failure (not threat-positive — the scan itself errored) MUST NOT block the redirect when `SAFE_BROWSING_LOG_ONLY=true`. Otherwise it MAY fail closed; implementation-defined.

---

## 21. Backward Compatibility (External Surfaces Only)

### 21.1 Scope

GoGovSG is a long-lived public service, but only a small subset of its HTTP surface is consumed by third parties. A reimplementation MAY freely change everything *except* the surfaces that exist outside the system's own client. This section enumerates the externally-binding surfaces; **anything not listed here is implementation-defined and may be changed without notice.**

The externally-binding surfaces are:

1. The **redirect endpoint** at `GET /:shortUrl` — invoked by every link in the wild (SMS, email, posters, QR codes, partner sites, search engines).
2. The **file URL shape** at `https://{file-hostname}/{shortUrl}.{ext}` — appears as `urls.longUrl` for file-backed short links, republished in user-facing materials.
3. The **External REST API** at `/api/v1/*` and `/api/v1/admin/*` — designed and documented for third-party integration; gated by `FF_EXTERNAL_API`.
4. The **API key format and authentication** for the External REST API — integrators have generated keys and store them in their own systems.

Everything else — the SPA-facing `/api/*` routes (login, user, qrcode, link-stats, link-audit, directory, callback), cookie names, session storage layout, response envelopes, HTML 404 templates, asset paths, log format, internal headers, the dual query/body parameter source on `GET /api/user/url`, the `hasApiKey` stringly-typed response — is **internal**. A rewrite SHOULD reach functional parity with these surfaces (so the SPA still works), but is free to change paths, methods, request shapes, response shapes, and status semantics. The SPA is part of the rewrite and may be updated in lockstep.

### 21.2 Redirect endpoint (external)

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

### 21.3 File-hosting URLs (external)

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

### 21.4 External REST API v1 (external)

The `/api/v1/*` and `/api/v1/admin/*` namespaces are the documented integration surface for third parties. A reimplementation MUST preserve:

- **Paths and HTTP methods** exactly:
  - `GET /api/v1/urls`
  - `POST /api/v1/urls`
  - `PATCH /api/v1/urls/:shortUrl`
  - `POST /api/v1/admin/urls`
- **Authentication scheme**: `Authorization: Bearer <apiKey>` (§6.2 / §21.5).
- **Feature-flag gating**: when `FF_EXTERNAL_API` is disabled, these paths MUST return HTTP 404 (not 401). Integrators detect "API disabled" by 404.
- **Request schema** for each route, as defined in §17. New optional fields MAY be added; required fields MUST NOT be added; existing required fields MUST NOT be removed.
- **Response schema** for each route. The mapped `StorableUrl` returned by these endpoints is a stable, versioned DTO — it omits internal fields (`safeBrowsingExpiry`, `userId`) by design. Adding fields is safe; removing, renaming, or retyping fields is not.
- **Status codes**: 200 on success, 400 on validation failure, 401 on missing/invalid API key, 404 when the feature is disabled or the resource is absent. Status semantics MUST NOT shift between these classes.

Implementation details that MAY change:

- The internal handler that backs each route.
- Whether `:shortUrl` is lifted into the request body before validation (the existing implementation does this via a middleware; a rewrite need not).
- The on-disk representation of the URL record.
- The presence of additional response fields beyond the documented schema.

### 21.5 API key authentication (external)

API keys issued by the current deployment MUST continue to authenticate after a rewrite. A key is the opaque string `${env}_${version}_${random}` returned once at generation time. To preserve this:

- The hash stored at `users.apiKeyHash` MUST remain verifiable. The existing format is `${env}_${version}_${bcrypt(random, API_KEY_SALT)}`. A reimplementation MAY use a different verification scheme **only if** it migrates existing hashes to the new scheme during deployment, or rejects existing keys and forces all integrators to re-generate.
- A rewrite that wishes to preserve existing keys MUST therefore preserve the bcrypt verification path and the `API_KEY_SALT` environment variable.
- A rewrite that explicitly does *not* wish to preserve existing keys MUST publish a key-rotation deadline before deployment.

`API_KEY_VERSION` exists precisely to bracket this concern. Bumping the version (`v1` → `v2`) is a clean way to introduce a new format while continuing to verify old keys against the old format.

### 21.6 Out of scope for backward compatibility

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

### 21.7 Removal policy for the External REST API

Because §21.4 is the only HTTP surface bound by an external versioning contract, removal of an `/api/v1/*` field follows this policy:

1. Introduce a new versioned namespace (`/api/v2/*`).
2. Continue serving `/api/v1/*` for at least one quarter after the new version is announced.
3. Emit a `Deprecation` HTTP response header on every `/api/v1/*` response during the deprecation window.

Adding fields to existing `/api/v1/*` responses is non-breaking and requires no version bump.

---

## Appendix A. Validation Rules Reference

Quick reference for the rules in §5 and §10:

| Field | Pattern / Rule | Max length |
|-------|----------------|------------|
| `shortUrl` | `^[a-zA-Z0-9-]+$` | implementation-defined; auto-gen length 8 |
| `longUrl` | HTTPS, valid TLD, not IP, not circular, not blacklisted | none |
| `email` | `validator.isEmail` AND `minimatch(VALID_EMAIL_GLOB_EXPRESSION)` | none |
| `description` | printable ASCII | 200 |
| `tag` (string) | `^[A-Za-z0-9_-]+$` | 25 |
| tags per link | unique | 3 |
| file extension | allowlist (§5.6) | — |
| file size | ≤ 20 MiB | — |
| CSV size | ≤ 5 MiB | — |
| CSV rows | ≤ `BULK_UPLOAD_MAX_NUM` | — |

---

## Appendix B. Locale Schema

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

## Appendix C. Public Surface Map

A single-page index of every public HTTP route.

Columns:
- **Auth**: `S` = session required, `K` = API key required, `A` = admin role required, `–` = public.
- **BC**: backward-compatibility class. **`E`** = externally-binding (third-party integrators, published links, or stored URLs in the wild depend on this route — see §21); **`I`** = internal (consumed only by the GoGovSG SPA or other rewritable surfaces; may be redesigned).

| Method | Route | Auth | BC | Section |
|--------|-------|------|----|---------|
| `GET` | `/:shortUrl` | – | **E** | §8, §21.2 |
| `GET` | `/api/v1/urls` | K | **E** | §17.1, §21.4 |
| `POST` | `/api/v1/urls` | K | **E** | §17.1, §21.4 |
| `PATCH` | `/api/v1/urls/:shortUrl` | K | **E** | §17.1, §21.4 |
| `POST` | `/api/v1/admin/urls` | K + A | **E** | §17.2, §21.4 |
| `GET` | `/api/ga` | – | I | §13 |
| `GET` | `/api/stats` | – | I | §13.1 |
| `GET` | `/api/links` | – | I | §3, §16.3 |
| `GET` | `/api/login/message` | – | I | §6.1 |
| `GET` | `/api/login/emaildomains` | – | I | §6.1 |
| `POST` | `/api/login/otp` | – (IP rate-limited) | I | §6.1 |
| `POST` | `/api/login/verify` | – | I | §6.1 |
| `GET` | `/api/login/isLoggedIn` | – | I | §6.1 |
| `GET` | `/api/logout` | – | I | §6.3 |
| `GET` | `/api/user/url` | S | I | §7.5 |
| `POST` | `/api/user/url` | S | I | §7.1 |
| `PATCH` | `/api/user/url` | S | I | §7.2 |
| `PATCH` | `/api/user/url/ownership` | S | I | §7.3 |
| `POST` | `/api/user/url/bulk` | S | I | §11.1 |
| `GET` | `/api/user/tag` | S | I | §7.6 |
| `POST` | `/api/user/apiKey` | S | I | §6.2 |
| `GET` | `/api/user/hasApiKey` | S | I | §6.2 |
| `GET` | `/api/user/message` | S | I | §4 |
| `GET` | `/api/user/announcement` | S | I | §4 |
| `GET` | `/api/user/job/latest` | S | I | §11.3 |
| `GET` | `/api/user/job/status` | S | I | §11.3 |
| `GET` | `/api/qrcode` | S | I | §12.1 |
| `GET` | `/api/link-stats` | S | I | §13.2 |
| `GET` | `/api/link-audit` | S | I | §14 |
| `GET` | `/api/directory/search` | S | I | §15 |
| `GET` | `/assets/transition-page/js/redirect.js` | – | I | §8.1 |
| `GET` | `/locales/en/translation.json` (or implementation-defined path) | – | I | §16.8 |

A rewrite that wishes to remain compatible with deployed links and existing API integrations need only preserve the **E**-class rows of this table (plus the file URL shape of §21.3 and the API key verification rules of §21.5). All **I**-class rows may be redesigned, removed, or replaced; the SPA must be updated in lockstep.

---

*End of specification.*
