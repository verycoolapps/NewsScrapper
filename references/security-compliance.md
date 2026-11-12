# Security & Compliance Reference

Derived from SKILL.md §15 (Security & Compliance), §14 (Database), §16 (Evidence-Aware Intelligence), and the access restrictions in §02 (Source Discovery).

## 1. Security Foundation

The platform must implement, at minimum, every foundation item named in SKILL.md §15. Table below maps each requirement to the concrete controls expected.

### 1.1 Authentication

- Email + password with Argon2id (or bcrypt cost ≥ 12) hashing. Never store reversible passwords.
- Password policy: minimum 12 characters, breached-password screening, no forced periodic rotation (NIST 800-63B aligned).
- Session tokens: short-lived access JWT (≤ 15 min) + rotating refresh token stored server-side in `users`/`user_preferences` session tables or Redis with revocation.
- Account lifecycle: email verification on registration, password reset via single-use expiring tokens (≤ 1 h), session invalidation on password change.
- Organization membership: users belong to `organizations`; switching org context requires an explicit tenant claim in the token.

### 1.2 MFA (multi-factor authentication)

- TOTP (RFC 6238) and WebAuthn/passkeys supported. SMS OTP disallowed as sole factor.
- MFA enrollment mandatory for roles: `admin`, `org_owner`, `enterprise_admin` (org users with billing/admin rights); optional for free/pro readers.
- Recovery codes: 10 single-use codes, hashed at rest, shown once at enrollment.
- Admin actions (§3) require step-up MFA verification within the last 15 minutes.

### 1.3 RBAC

Roles:

| Role | Scope |
|------|-------|
| `reader` | Personal feed, search, saved articles, alerts |
| `pro_user` | + personalization, saved searches, history |
| `org_member` | Org-shared collections, org monitoring within subscription |
| `org_admin` | Manage org members, dashboards, exports |
| `support` | Read-only user/session diagnostics, no content mutation |
| `admin` | Full Admin Control Center (§19 modules) |
| `super_admin` | User/role management, retention policy changes, infra config |

- Permissions enforced server-side on every route; never rely on frontend hiding.
- Tenant isolation: every org-scoped query must filter by `organization_id`; cross-tenant IDOR is a release blocker.
- Least privilege for service accounts: crawler workers, AI workers, and API consumers each get distinct credentials.

### 1.4 API authentication

- API keys for programmatic access (§18): `no` prefix, shown once at creation, stored hashed (SHA-256 + pepper), prefix stored for lookup.
- Scopes: `news:read`, `search:read`, `events:read`, `alerts:write`, `collections:write`, `admin:read` (admin-only).
- IP allowlists optional per key; rotation and expiry supported; revocation is immediate.
- Rate limits keyed per API key *and* per user session; see §1.6.

### 1.5 Input validation & output sanitization

- All API inputs validated with Pydantic schemas; strict types, bounded strings, allowlisted enums for filters (country, language, category, sort).
- All query parameters for `/api/v1/search` and filters parsed through typed models — no string concatenation into SQL or OpenSearch queries.
- Stored XSS surface: article titles/summaries are publisher-controlled text. Sanitize on render (React default escaping + explicit `dangerouslySetInnerHTML` ban), sanitize on ingest for any field that may contain markup.
- URL fields (`original_url`, `canonical_url`) validated against `http(s)` scheme allowlist; reject `javascript:`, `data:`, protocol-relative edge cases.
- HTML from crawled pages is never served raw to clients; extraction produces structured text only.

### 1.6 Rate limiting

- Per-user and per-IP sliding-window limits on auth endpoints (login, register, reset) to blunt credential stuffing.
- Per-domain crawler rate limits are publisher-policy driven (§3.4), separate from API rate limits.
- API v1 defaults: e.g., 60 req/min per key for reads, 10 req/min for writes (`alerts`, `saved-searches`, `collections`); exact values configurable per subscription tier.
- Return `429` with `Retry-After` header; never silently drop.

### 1.7 Secrets management

- No secrets in the repository. `.env` files are gitignored; `.env.example` ships with names only.
- Production secrets via a secrets manager (Vault, AWS Secrets Manager, or platform-native equivalent) or injected env vars; crawler/AI provider keys rotate on schedule.
- Database credentials, S3 keys, AI provider keys, CAPTCHA-free feed credentials — all enumerated in the deployment config (§runbook).
- Rotate on suspicion of exposure; audit log every secret access where feasible.

### 1.8 Audit logging

- `audit_logs` table (SKILL.md §14) records: actor, action, target, timestamp, IP, user-agent, request id, before/after diff for mutations.
- Mandatory events: login/logout/MFA changes, role changes, source approval/disable, retention policy edits, takedown actions, API key creation/revocation, admin config changes (AI, translation, duplicate detection), data exports.
- Append-only: audit rows are never updated or deleted through application code; corrections are new rows.
- Retention: audit logs retained ≥ 12 months regardless of content retention tier.

### 1.9 Encryption

- TLS 1.2+ everywhere; HSTS enabled; no mixed content.
- At-rest encryption for Postgres volumes, S3 buckets, backups, and Redis snapshots where the platform supports it.
- Field-level encryption for sensitive columns if PII is stored (user emails hashed or encrypted at rest per privacy policy).
- Secrets in transit only via TLS; mTLS between internal services optional but recommended for worker → API calls.

### 1.10 Backup & recovery

- Postgres: daily `pg_dump`/continuous WAL archiving; restore tested quarterly (procedure in runbook).
- S3-compatible object storage: versioning enabled; cross-region replication for article payloads where licensed.
- Redis: persistence configured for queues that must survive restart, or accept loss with re-derivation from Postgres/Kafka.
- Recovery objectives to define per deployment: RPO ≤ 24 h for metadata, RPO ≤ 1 h for queue state; RTO ≤ 4 h.

### 1.11 Vulnerability monitoring

- Dependency scanning in CI (pip-audit / npm audit / osv-scanner), container image scanning, and scheduled `npm audit`/`pip audit` equivalents.
- SAST on auth, payment, and ingestion paths; DAST against staging before release.
- Patch SLA: critical CVEs patched within 72 h; high within 14 days.
- `system_metrics` + Sentry capture runtime anomalies; security events feed `audit_logs` and alerting.

---

## 2. Legal & Content Compliance

SKILL.md §15 legal requirements are binding, not aspirational. The sections below operationalize them.

### 2.1 Copyright

- We are an aggregator, not a republisher. Default storage posture: metadata, headline, short factual description, canonical link (§2.6).
- Full article body text is stored and displayed **only** when: (a) the publisher's license/API terms explicitly permit redistribution, (b) content is public domain / government release, or (c) a commercial license is in place (ENTERPRISE tier, SKILL.md §20).
- AI summaries and translations are derivative works: always attributed, always linked to the original, never presented as the publisher's text.
- DMCA-style takedown intake (§2.4) with a documented counter-notice path; repeat infringer policy defined per jurisdiction.

### 2.2 robots.txt

- Every crawl-enabled source must have a `source_permissions` row recording robots evaluation result and the user-agent policy applied.
- Respect `Disallow`, `Crawl-delay`, and per-path rules. Re-evaluate robots on change (hash of robots.txt stored; diff triggers re-review).
- A source whose robots.txt disallows our UA is automatically `crawl_enabled=false`; it may still be ingested via RSS/API where the feed itself is public.

### 2.3 Terms of Service

- `source_permissions.ToS_status` tracked per source: `permitted`, `restricted`, `prohibited`, `unknown`.
- `prohibited` sources are excluded from all collection methods, including RSS, unless the feed is explicitly offered as a public syndication channel.
- ToS changes detected (manual review queue in admin) can flip status and disable crawling pending review.

### 2.4 Takedown handling

SOP (details in runbook):

1. Intake via published contact + web form; log every request with timestamp.
2. Verify request authenticity (copyright holder or authorized agent).
3. Within SLA (target ≤ 72 h), either remove content, restrict to metadata-only, or reject with reason.
4. Record in `audit_logs` and the publisher takedown module.
5. Notify requester of outcome; keep a suppression list so removed content is not re-ingested from another feed without review.
6. Preserve a minimal tombstone record (source id, URL, removal date, reason) for audit — not the content itself.

### 2.5 Attribution

- Every displayed article shows publisher name, original URL (clickable, `rel="noopener"`), publication timestamp, and country/language.
- Publisher logos/names used nominatively; no implied endorsement.
- Feed-level attribution requirements honored (e.g., "via PublisherName").

### 2.6 Metadata vs. full-text storage rules

| Data class | Default | Requires license |
|------------|---------|------------------|
| Headline, author, pub date, URL, canonical URL | Store + display | No |
| Short factual summary / dek | Store + display | Generally no; keep within fair-dealing norms |
| Featured image | Store thumbnail/link only | Hotlink preferred; cache only if licensed |
| Full article body | Metadata-only by default | Yes — store body only when `license_info` permits |
| AI summary / translation | Store + display, clearly labeled | No (our generated content, but must cite original) |
| Entities, topics, embeddings | Store | Derived metadata, permitted |

- `articles` rows carry a `content_storage` flag: `metadata_only` | `excerpt` | `fulltext`.
- Publisher-specific policy overrides (§2.7) can tighten this further.

### 2.7 Publisher-specific content policies

- `source_permissions` stores per-publisher rules: allowed storage depth, required attribution string, image policy, AI-processing opt-out flag.
- Publishers may request `ai_processing=false`: metadata + link only, no summarization/translation/entity extraction beyond classification. Enforced in the processing pipeline before AI stages.
- Publishers may request `no_image_cache=true`: store image URL only.
- Policy changes propagate to already-ingested articles: a `content_storage` downgrade triggers body-text purge job.

### 2.8 Retention policies

- Configurable per data class (SKILL.md §14 requires retention policies): article metadata 24 months default, full text per license terms (often shorter), user data per account lifecycle + legal hold, search history 90 days default, audit logs ≥ 12 months.
- Deletion must cascade: removed articles leave `article_versions`, `embeddings`, `event_articles` consistent; vector index entries purged.
- User deletion: anonymize PII, retain aggregate metrics without identity.
- Retention changes are admin actions with audit logging and a preview of affected row counts.

### 2.9 Privacy regulations

- GDPR/CCPA basis documented: legitimate interest for aggregation, consent for personalization/alerts, contract for paid tiers.
- Data export and erasure endpoints for users (wired to §18 API where applicable).
- Do-not-sell / no ad tracking: the platform does not run third-party ad trackers; analytics is first-party and aggregate.

### 2.10 Hard prohibitions (never, regardless of config)

- Never bypass paywalls, authentication, CAPTCHAs, access restrictions, or anti-bot controls.
- Never scrape a source whose `ToS_status = prohibited`.
- Never present AI output as publisher content; always label `ORIGINAL CONTENT` / `MACHINE TRANSLATION` / `AI-GENERATED SUMMARY` (SKILL.md §05 sacred rule).
- Never fabricate verification badges or claim `CONFIRMED BY PRIMARY SOURCE` without a linked primary source (SKILL.md §16).
