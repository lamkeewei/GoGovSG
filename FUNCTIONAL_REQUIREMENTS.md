# Functional Requirements

This document enumerates what the system MUST do from a user and business perspective. It contains no implementation details (no APIs, schemas, status codes, technologies, or storage choices). Each requirement is independently testable against a running system.

Requirement IDs are stable within this document and may be cited by other artifacts. Normative keywords (`MUST`, `MUST NOT`, `SHOULD`, `MAY`) follow RFC 2119.

---

## 1. Roles and Access

- **FR-1.1** The system MUST distinguish three classes of user: **citizens** (anonymous visitors), **officers** (authenticated users), and **administrators** (officers with elevated privileges on the integration API).
- **FR-1.2** Administrator status MUST be determined by an explicit, deployment-configured list of email addresses. It MUST NOT be assignable through the user interface.
- **FR-1.3** Officers MUST be able to act only on resources they own, except where this document explicitly permits otherwise (administrator provisioning, ownership transfer acceptance).
- **FR-1.4** Citizens MUST be able to use the redirect surface and the public directory without any account.

---

## 2. Authentication

- **FR-2.1** The system MUST allow an officer to sign in by entering their work email and receiving a one-time password (OTP) at that address.
- **FR-2.2** The system MUST accept only email addresses whose domain matches the deployment's allowlist pattern.
- **FR-2.3** OTPs MUST be single-use and MUST expire after a short, configurable validity window (default 5 minutes).
- **FR-2.4** The system MUST limit the rate at which OTPs can be requested for a single network address.
- **FR-2.5** A submitted OTP MUST be accepted at most a small number of times (default 3) before the OTP is invalidated and the officer must request a new one.
- **FR-2.6** A user record MUST be created automatically the first time an OTP is successfully verified for an email address that does not yet have one.
- **FR-2.7** The system MUST maintain an authenticated session for a configurable duration (default 1 day) and MUST allow the officer to explicitly sign out.
- **FR-2.8** The system MUST treat email addresses case-insensitively for sign-in and ownership purposes.
- **FR-2.9** The system MUST allow an officer to mint a personal API key for programmatic access. Each minting replaces any prior key for that officer.
- **FR-2.10** The system MUST display a newly minted API key to the officer exactly once and MUST NOT make it retrievable thereafter.
- **FR-2.11** The system MUST allow an officer to check whether they currently have an active API key without exposing the key itself.
- **FR-2.12** Existing API keys MUST continue to authenticate after deployment upgrades; key rotation MUST be an explicit operation, not a side effect of a routine change.

---

## 3. Short Link Management

### 3.1 Creation

- **FR-3.1.1** An officer MUST be able to create a short link mapping a custom slug to an HTTPS destination URL.
- **FR-3.1.2** An officer MAY supply an optional description (subject to length and printable-ASCII constraints) and an optional contact email at creation time.
- **FR-3.1.3** An officer MAY assign up to three tags to a link at creation time.
- **FR-3.1.4** The system MUST reject a slug that is already in use, with an error that distinguishes "already taken" from other validation failures.
- **FR-3.1.5** The system MUST reject a destination URL that:
  - is not HTTPS;
  - is not a structurally valid URL;
  - resolves to a literal IP address;
  - points back to the service's own origin (would create a redirect loop);
  - matches the deployment's blacklist of disallowed destinations.
- **FR-3.1.6** New links MUST be created in the **active** state.

### 3.2 Update

- **FR-3.2.1** The owner of a link MUST be able to edit its destination URL, description, contact email, tags, and active state.
- **FR-3.2.2** The system MUST reject any attempt to edit the resource of a link the requester does not own.
- **FR-3.2.3** A link's "is a file" character is fixed at creation time and MUST NOT be modifiable thereafter.
- **FR-3.2.4** When a link's destination URL is changed, the new destination MUST be threat-scanned (FR-12.1) before the change is accepted.

### 3.3 Ownership Transfer

- **FR-3.3.1** The owner of a link MUST be able to transfer ownership to another officer by specifying that officer's email.
- **FR-3.3.2** If the target email does not correspond to an existing user, a user record MUST be created for them.
- **FR-3.3.3** The new owner MUST be notified of the transfer by email.
- **FR-3.3.4** Transferring a link to oneself MUST be rejected.

### 3.4 Deactivation

- **FR-3.4.1** The owner of a link MUST be able to set its state to **inactive**.
- **FR-3.4.2** An inactive link MUST resolve as "not found" when visited.
- **FR-3.4.3** The system MAY auto-deactivate a link if its destination is detected as malicious after publication (see FR-12.5).

### 3.5 Listing and Search

- **FR-3.5.1** An officer MUST be able to view their own links in a paginated, sortable list.
- **FR-3.5.2** The officer's link list MUST be filterable by active/inactive state, file/non-file character, and substring of the slug or destination.
- **FR-3.5.3** The officer's link list MUST be filterable by tag, with multiple tags treated as an OR query.
- **FR-3.5.4** Substring search and tag filter are mutually exclusive within a single request.
- **FR-3.5.5** The list MUST be sortable by creation date and by click count, ascending or descending.

### 3.6 Tags

- **FR-3.6.1** A tag is a short alphanumeric label (with hyphens and underscores allowed, maximum 25 characters).
- **FR-3.6.2** No link may carry more than three tags; a tag may not appear twice on the same link.
- **FR-3.6.3** The same tag may be reused across many links; deleting or unlinking a tag from one link MUST NOT affect its association with others.
- **FR-3.6.4** Tag matching is case-insensitive; the original display casing is preserved.
- **FR-3.6.5** An officer MUST be able to autocomplete from the set of tags they have previously used, by prefix, with a configurable minimum query length.

---

## 4. File Hosting

- **FR-4.1** An officer MUST be able to create a short link whose destination is a file the officer uploads, instead of an external URL.
- **FR-4.2** The system MUST accept only files whose detected content type matches an allowlisted set of safe office, document, image, archive, and CAD formats. The list is fixed at the deployment level and is NOT extensible by individual users.
- **FR-4.3** Individual file uploads MUST be limited to a configurable maximum size (default 20 MiB).
- **FR-4.4** The system MUST scan every uploaded file with an antivirus service before accepting it (when antivirus is configured). Password-protected archives and virus-positive files MUST be rejected.
- **FR-4.5** The public URL of a hosted file MUST be derived from the short link's slug and the file's extension. The owner MUST be able to replace the underlying file without changing this URL.
- **FR-4.6** When a file-backed link is deactivated, the underlying file MUST become non-public.
- **FR-4.7** The system MUST NOT allow a file-backed link to be converted to a URL-backed link, or vice versa, after creation.

---

## 5. Redirect Behavior (Citizen Surface)

- **FR-5.1** Visiting the service origin followed by a slug MUST navigate the visitor to the slug's destination URL or, if the slug is unknown or inactive, present a clear "link not found" page.
- **FR-5.2** Slug resolution MUST be case-insensitive.
- **FR-5.3** The system MUST tolerate a trailing punctuation character appended to the slug (such as a sentence-terminating dot pasted from a message) and resolve to the same link.
- **FR-5.4** A first-time human visitor to a link SHOULD be shown a brief interstitial **transition page** that displays the destination and gives the visitor a chance to verify the address bar before proceeding. This defends against phishing impersonation.
- **FR-5.5** A repeat human visitor to the same link MUST NOT be shown the transition page again on subsequent visits within a reasonable window.
- **FR-5.6** A visitor arriving from the service's own origin (e.g., from the directory) MUST be redirected without seeing the transition page.
- **FR-5.7** A crawler or automated agent MUST be redirected directly, with no transition page.
- **FR-5.8** Every successful resolution to an active link MUST record one click for that link, attributed to the device class derived from the user agent. Click recording MUST NOT block the user-visible redirect.

---

## 6. Statistics and Analytics

- **FR-6.1** An officer MUST be able to view per-link statistics for any link they own: total click count, breakdown by device class (mobile / tablet / desktop / other), daily click history (configurable window, default 7 days), and a weekday-by-hour click heatmap.
- **FR-6.2** Time dimensions in per-link statistics MUST be expressed in the deployment's home time zone (Asia/Singapore by default).
- **FR-6.3** Citizens visiting the landing page MUST see top-line counters (total users, total links, total clicks). The exact values are deployment-configured.
- **FR-6.4** When external web analytics is configured, every non-crawler redirect SHOULD be reported to it.

---

## 7. Audit Trail

- **FR-7.1** Every create, update, ownership transfer, and state change to a link MUST be recorded immutably.
- **FR-7.2** The owner of a link MUST be able to view its complete change history in reverse chronological order, with pagination.
- **FR-7.3** The history MUST identify, for each change: the field that changed, the previous and new values, the time of change, and the acting officer's email.
- **FR-7.4** The initial creation event MUST be visible in the history as a distinct entry, not just an "update".
- **FR-7.5** History entries MUST NOT be editable or deletable by any user, including administrators.

---

## 8. Bulk Operations

- **FR-8.1** An officer MUST be able to create many short links in one operation by uploading a CSV file containing destination URLs.
- **FR-8.2** The CSV file MUST be limited in size (default 5 MiB) and in number of rows (default 1000 URLs).
- **FR-8.3** The CSV MUST have a single column with a fixed header text. Files with a missing or incorrect header MUST be rejected before any link is created.
- **FR-8.4** Every row MUST be validated against the same rules as single link creation (FR-3.1.5). The bulk upload MUST be rejected atomically if any row fails — partial creation MUST NOT occur.
- **FR-8.5** Slugs for bulk-created links MUST be generated by the system at random; the user MUST NOT supply them.
- **FR-8.6** Optionally, the same set of tags MAY be applied to every link in a bulk upload.
- **FR-8.7** When bulk QR-code generation is enabled at the deployment level, a successful bulk upload MUST also produce QR-code artifacts asynchronously (FR-9.4).

---

## 9. QR Code Generation

- **FR-9.1** An officer MUST be able to download a branded QR code for any of their links on demand.
- **FR-9.2** QR codes MUST be available in SVG, PNG, and JPEG formats. The image MUST encode the full short URL.
- **FR-9.3** Every QR code MUST be rendered with the deployment's brand color and a centered brand logo overlay. The styling MUST be identical whether the QR is produced on demand or as part of a bulk job.
- **FR-9.4** Bulk QR code generation MUST produce three artifacts per batch: a CSV mapping short URLs to long URLs, a zipped bundle of SVG QR codes, and a zipped bundle of PNG QR codes.
- **FR-9.5** The officer MUST be able to poll for completion of a bulk-QR job and receive download links to the artifacts when finished.
- **FR-9.6** The officer MUST be notified by email when a bulk-QR job terminates (either successfully or unsuccessfully).

---

## 10. Public Directory

- **FR-10.1** Anyone (including citizens) MUST be able to search the directory of published links to verify a link's authenticity before clicking.
- **FR-10.2** Directory search MUST rank results so that matches in the slug rank highest, matches in the destination URL rank next, and matches in the description rank lowest.
- **FR-10.3** Directory search MUST be filterable by:
  - active/inactive state,
  - file/non-file character,
  - search-by-text vs. search-by-owner-email mode.
- **FR-10.4** Directory search MUST support sorting by popularity (click count) or by recency (creation date).
- **FR-10.5** Search results MUST disclose, for each match: the slug, the owner's email, the link's state, and whether it is file-backed. The destination URL MUST NOT be disclosed in search results.
- **FR-10.6** When the user is in "by-text" mode, the search query MUST be normalised to prevent it from enumerating email domains.
- **FR-10.7** Search results MUST be paginated, with a reasonable maximum page size.

---

## 11. External Integration API

- **FR-11.1** When the deployment has enabled the External Integration API, officers MUST be able to use a personal API key (FR-2.9) to:
  - list their own short links,
  - create a new short link (custom or auto-generated slug),
  - update an existing short link's destination or state.
- **FR-11.2** Administrators MUST additionally be able, via the External Integration API, to:
  - create a short link on behalf of another officer, identified by email.
- **FR-11.3** The External Integration API MUST NOT permit file uploads, file replacement, or operations on links the requester does not own (except administrator provisioning).
- **FR-11.4** When the External Integration API is disabled at the deployment level, requests to it MUST be indistinguishable from requests to a non-existent endpoint.
- **FR-11.5** The External Integration API's contract — request and response shapes, paths, and authentication — MUST be versioned and MUST remain stable within a major version.

---

## 12. Threat Protection

- **FR-12.1** Every destination URL submitted at link creation, link update, or bulk upload MUST be checked against a URL threat-scanning service (when one is configured). Detected threats MUST be rejected.
- **FR-12.2** Every uploaded file MUST be checked against an antivirus service (when one is configured). Detected viruses and password-protected archives MUST be rejected.
- **FR-12.3** Threat-scan and antivirus verdicts MUST be cached so that repeated scans of the same destination or file within a short window do not require a fresh external call.
- **FR-12.4** A clean URL-threat verdict has a finite shelf life (default 24 hours). Once expired, the verdict MUST be refreshed the next time the link is resolved.
- **FR-12.5** If a previously-published link's destination becomes malicious after the fact (detected during refresh per FR-12.4), the system MUST:
  - deactivate the link automatically,
  - email the link's owner to inform them of the action,
  - present visitors with a "link not found" response indistinguishable from the response for a non-existent link.
- **FR-12.6** The deployment MAY enable a "log only" threat mode in which detections are recorded but not enforced. This mode is for staging and rollout validation; production MUST default to enforcing detections.
- **FR-12.7** When neither threat-scan nor antivirus service is configured, the system MUST still operate, with the corresponding check skipped. Absence of these services MUST NOT prevent link creation or file upload.

---

## 13. Notifications and User Messaging

- **FR-13.1** The system MUST be able to send the following emails to officers:
  - OTP delivery on sign-in.
  - Bulk-QR job completion (success or failure).
  - Notification when a link they own is auto-deactivated due to threat detection.
  - Notification when a link is transferred to them.
- **FR-13.2** The system MUST allow the deployment to display a banner on the sign-in page (configurable text).
- **FR-13.3** The system MUST allow the deployment to display a banner on the signed-in dashboard (configurable text).
- **FR-13.4** The system MUST support a one-time announcement modal that appears on first sign-in to the dashboard. The modal carries configurable title, subtitle, body, image, call-to-action label, and call-to-action URL. The modal renders only when configured.
- **FR-13.5** The landing page MUST display a configurable list of featured short links for rotation.

---

## 14. Deployment Branding

- **FR-14.1** A deployment's public name, short-URL origin, file-hosting hostname, allowed-email-domain pattern, and brand color/logo for QR codes MUST be configurable per deployment.
- **FR-14.2** The English copy used by the client (page titles, headings, prompts, link labels) MUST be loaded from a single localisation bundle for the deployment, not hard-coded.
- **FR-14.3** Only English is supported as a UI language.

---

## 15. Citizen-facing Behavioral Guarantees

These requirements describe behaviors visible to citizens (and to integrators with deployed links) that MUST NOT regress without an explicit migration.

- **FR-15.1** Short-URL paths in the form `/<slug>` MUST continue to resolve at the deployment's production origin.
- **FR-15.2** Resolution MUST remain case-insensitive and trailing-character tolerant (FR-5.2, FR-5.3) so that links shared in older messages keep working.
- **FR-15.3** File URLs in the form `https://<file-hosting hostname>/<slug>.<ext>` MUST continue to resolve to the corresponding file as long as the link is active.
- **FR-15.4** A successful redirect MUST take the visitor to the destination URL either directly or after a transition page; in either case the destination is reached.
- **FR-15.5** When a link is unknown or inactive, the citizen MUST see a clear "not found" experience, not an internal-error page.

---

## 16. Failure Behavior

- **FR-16.1** A failure of an optional integration (threat-scan service, antivirus, web analytics, observability) MUST NOT take down primary user flows.
- **FR-16.2** A failure to record a click MUST NOT delay or block the user-visible redirect.
- **FR-16.3** A failure during a bulk upload MUST be reported to the user with row-level diagnostics (which row failed, why); no partial state MUST result.
- **FR-16.4** Bulk-QR jobs MUST surface terminal status (success or failure) to the requesting officer through both the dashboard and email notification.

---

## 17. Out of Scope

The following are explicitly NOT functional requirements of this system.

- **FR-17.1** Multi-factor authentication beyond the OTP itself.
- **FR-17.2** Per-click geolocation, referrer tracking, or visitor-identity profiling.
- **FR-17.3** Internationalisation of the UI beyond English.
- **FR-17.4** Acceptance of arbitrary file MIME types beyond the allowlist of FR-4.2.
- **FR-17.5** Click-rate throttling on the redirect surface.
- **FR-17.6** Recovery of forgotten or compromised API keys (only rotation is supported).

---

*End of functional requirements.*
