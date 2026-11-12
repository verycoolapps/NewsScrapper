# API Ecosystem Reference

Derived from SKILL.md §18 (API Ecosystem), §09 (Search Engine), §11 (Personalization), §12 (Notification Intelligence), §17 (Coverage Analytics), §20 (Monetization), and §21 (Performance targets).

## 1. Conventions

### 1.1 Versioning

- All endpoints live under `/api/v1`. Breaking changes require a new major version (`/api/v2`) with at least 6 months of overlap.
- Deprecation announced via `Deprecation` and `Sunset` response headers plus docs; 90-day minimum notice.

### 1.2 Authentication

| Method | Use |
|--------|-----|
| Session cookie (web app) | Browser users; HttpOnly, Secure, SameSite=Lax |
| `Authorization: Bearer <jwt>` | SPA/mobile clients |
| `X-API-Key: nk_live_…` | Programmatic consumers (§18) |

- API keys carry scopes (§security-compliance §1.4). Missing scope → `403`; missing/invalid credential → `401`; malformed request → `400`; not found → `404`; conflict → `409`; rate limited → `429` with `Retry-After`.

### 1.3 Pagination

- Cursor-based for all list endpoints: `?cursor=<opaque>&limit=<n>`.
- `limit` default 25, max 100 (articles), 50 (events/sources).
- Response envelope:

```json
{
  "data": [ … ],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIz…",
    "has_more": true,
    "limit": 25
  }
}
```

- Offset pagination not offered (unstable under continuous ingestion).

### 1.4 Filtering & sorting

- Filters are typed query params, e.g. `?country=JP&language=ja&category=technology&from=2026-11-01&to=2026-11-12`.
- Search operators (`AND`, `OR`, `NOT`, `"exact phrase"`, `domain:`, `country:`, `language:`) per SKILL.md §09 are parsed server-side into the query DSL — never interpolated raw.
- Sort: `?sort=published_at|importance|relevance` with `order=asc|desc`; relevance only valid with `q`.
- Unknown filter values return `400` with the allowed enum in the error body.

### 1.5 Rate limiting

- Per-key sliding window; read endpoints 60 req/min, write endpoints 10 req/min, search 30 req/min (defaults; adjustable per subscription tier).
- Headers on every response: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
- Exceeding → `429` + `Retry-After`; sustained abuse → key throttling, then revocation review.

### 1.6 Response metadata & errors

- Every article payload preserves provenance fields: `publisher`, `original_url`, `published_at`, `country`, `language`, `verification_label`, `content_storage` (§security-compliance §2.6).
- Error shape:

```json
{ "error": { "code": "SOURCE_NOT_FOUND", "message": "…", "request_id": "…" } }
```

- `request_id` is returned on every response (`X-Request-Id`) and correlates to logs/OTel traces.

### 1.7 Usage analytics

- Every API call emits a `system_metrics`-adjacent usage record: key id, endpoint, status, latency, timestamp, tier.
- Power the admin "API usage" module (§19) and per-tier quota dashboards.
- Analytics are aggregate-only; no content of requests is logged beyond query metadata needed for abuse review.

---

## 2. Endpoint Reference

### 2.1 News

#### `GET /api/v1/news`

List articles. Query params: `country`, `region`, `city`, `language`, `publisher` (source id or domain), `category`, `topic`, `from`, `to`, `importance_min`, `verification`, `format` (`article|brief`), `q` (full-text), `sort`, `order`, `cursor`, `limit`.

```json
{
  "data": [
    {
      "id": "art_01J…",
      "title": "…",
      "summary": "…",
      "publisher": { "id": "src_…", "name": "…", "domain": "…" },
      "original_url": "https://…",
      "canonical_url": "https://…",
      "published_at": "2026-11-12T08:14:00Z",
      "country": "JP", "language": "ja",
      "category": "technology",
      "topics": ["semiconductors"],
      "entities": [{"name": "…", "type": "organization"}],
      "verification_label": "REPORTED BY ONE SOURCE",
      "content_storage": "metadata_only",
      "importance_score": 0.82,
      "event_id": "evt_…"
    }
  ],
  "pagination": { "next_cursor": "…", "has_more": true }
}
```

#### `GET /api/v1/news/{id}`

Single article. Returns the same shape plus `body` only when `content_storage = fulltext`; otherwise `body: null` and `excerpt` when available. Includes `translations[]` (each with `language`, `kind: machine_translation`, `text`, `translated_at`) and `versions[]` (timestamped metadata revisions). 404 for unknown id; 410 for takedown-removed articles (tombstone per §security-compliance §2.4).

### 2.2 Search

#### `GET /api/v1/search`

Modes: `keyword` (default), `phrase`, `semantic`, `nl` (natural language), `entity`, `event`, `publisher`, `country`, `region` — selected via `mode` param or inferred. Required features per SKILL.md §09: autocomplete (`/api/v1/search/suggest` — see §2.9), typo tolerance, relevance ranking, saved searches, alerts.

Params: `q` (required), `mode`, all news filters, `event_id`, `entity`, `publisher`, `country`, plus operator syntax in `q`. Response adds `matched_events[]` and `total_hits` (capped estimate when > 10k).

```json
{
  "data": [ …articles… ],
  "matched_events": [{"id": "evt_…", "title": "…", "article_count": 14}],
  "pagination": { … }
}
```

### 2.3 Events

#### `GET /api/v1/events`

List clustered events. Params: `status` (`developing|developed|breaking`), `importance_min`, `country`, `topic`, `from`, `to`, `sort=first_seen|importance|velocity`, `cursor`, `limit`.

```json
{
  "data": [{
    "id": "evt_01J…",
    "title": "…",
    "summary": "…",
    "status": "developing",
    "importance_score": 0.91,
    "first_seen_at": "…", "last_update_at": "…",
    "article_count": 42,
    "independent_sources": 11,
    "countries": ["JP","US","GB"],
    "verification_label": "CORROBORATED BY INDEPENDENT SOURCES",
    "timeline_url": "/api/v1/events/evt_01J…/timeline"
  }]
}
```

#### `GET /api/v1/events/{id}`

Full event: article list (publisher-level, never merged — SKILL.md §07), `event_entities`, `event_timelines`, `conflicting_reports[]`, `verification` evidence list with links to supporting articles. Also `GET /api/v1/events/{id}/timeline` for the chronological reconstruction (publication timestamps distinguished from event time).

### 2.4 Topics & trends

#### `GET /api/v1/topics`

List topics: `id`, `name`, `description`, `article_count_24h`, `trend` (`emerging|growing|declining|stable`). Params: `trend`, `category`, `country`, `cursor`, `limit`.

#### `GET /api/v1/topics/{id}`

Topic detail: related topics (`topic_relationships`), trend series (hourly/daily), top articles, top entities.

#### `GET /api/v1/trending`

Fastest-growing topics and breaking events. Params: `window` (`1h|6h|24h|7d`), `country`, `category`, `limit`. Response separates `breaking_events[]` (volume spike detection, SKILL.md §06.6) from `trending_topics[]`. Cached per SKILL.md §21.

### 2.5 Sources

#### `GET /api/v1/sources`

Public-safe source registry listing. Params: `country`, `language`, `category`, `status` (`active|degraded|disabled`), `collection_method`, `cursor`, `limit`. Returns publisher name, domain, country, languages, category, `coverage_score`, `operational_status` — never internal notes or permission details beyond public status.

#### `GET /api/v1/sources/{id}`

Source detail: endpoints (RSS/sitemap/API, public-facing), coverage stats (articles indexed, last successful collection, failure rate), languages supported. Internal fields (`ToS_status` detail, rate limits) are admin-only.

### 2.6 Geography & coverage

#### `GET /api/v1/countries`

Country list with per-country stats: article count (windowed), active sources, languages, coverage band (`strong|moderate|weak|unavailable` — World Coverage Matrix bands per SKILL.md §17).

#### `GET /api/v1/languages`

Supported languages with article counts and detection confidence stats.

#### `GET /api/v1/coverage`

The §17 coverage intelligence payload: totals (discovered/active/indexed/inaccessible/licensed/RSS/API/crawl-enabled sources), failure rate, avg ingestion latency, and the `world_coverage_matrix` (rows = countries, columns = languages/categories/publisher types, cells = coverage band). Always computed from `news_sources`, `source_health_logs`, `crawl_jobs` — never hardcoded (§24 anti-slop).

### 2.7 Briefings

#### `GET /api/v1/briefings`

AI-generated briefings (SKILL.md §06.1, §08.8). Params: `period` (`daily|weekly`), `scope` (`global|country|topic|custom`), `country`, `topic_ids`, `format` (`executive|standard|detailed`).

```json
{
  "data": [{
    "id": "brf_…",
    "period": "daily",
    "generated_at": "…",
    "scope": { "type": "global" },
    "sections": [
      { "heading": "Top Developments", "items": [
        { "event_id": "evt_…", "text": "…", "citations": ["art_…","art_…"] }
      ]}
    ],
    "model": "provider/model@version",
    "sources_used": 87
  }]
}
```

Every briefing item must carry citations; uncited claims are a defect (§16, §24).

### 2.8 User resources (authenticated)

#### `POST /api/v1/alerts`

Create an alert. Body:

```json
{
  "type": "breaking|new_articles|event|topic|country|org_mention|industry|saved_search",
  "name": "Japan semiconductor watch",
  "filters": { "country": "JP", "topic_ids": ["…"], "q": "…" },
  "frequency": "instant|hourly|daily|weekly",
  "importance_threshold": 0.7,
  "quiet_hours": { "start": "22:00", "end": "07:00", "tz": "Asia/Tokyo" },
  "channels": ["in_app","email"]
}
```

`GET /api/v1/alerts`, `PATCH /api/v1/alerts/{id}`, `DELETE /api/v1/alerts/{id}` for management. Notification controls per SKILL.md §12 (frequency, quiet hours, importance threshold — anti-spam).

#### `POST /api/v1/saved-searches`

```json
{ "name": "…", "query": "…", "mode": "semantic", "filters": { … }, "alert": true }
```

`GET/PATCH/DELETE /api/v1/saved-searches/{id}`.

#### `POST /api/v1/collections`

```json
{ "name": "…", "description": "…", "article_ids": ["art_…"], "visibility": "private|org" }
```

`GET /api/v1/collections`, `PATCH/DELETE /api/v1/collections/{id}`, `POST /api/v1/collections/{id}/articles` to add/remove members.

### 2.9 Supporting endpoints (implied by §09/§11)

- `GET /api/v1/search/suggest?q=` — autocomplete.
- `GET /api/v1/me`, `GET /api/v1/me/preferences`, `PATCH /api/v1/me/preferences` — personalization state (favorite countries/publishers/topics).
- `GET /api/v1/me/saved-articles`, `POST /api/v1/me/saved-articles`, `DELETE …/{id}` — saved articles & reading history.
- `GET /api/v1/me/notifications` — notification center feed.
- `POST /api/v1/feedback` — report incorrect classification, takedown-adjacent issues, or article problems.

---

## 3. Developer Documentation Requirements

Per §18 and §25, ship:

1. **OpenAPI 3.1 spec** auto-generated from FastAPI/Pydantic models, served at `/api/docs` (Swagger) and `/api/redoc`; spec file versioned in repo.
2. **Quickstart**: register → create API key → first `/api/v1/news` call, with copy-paste `curl`.
3. **Authentication guide**: key creation, scopes, rotation, revocation, IP allowlists.
4. **Rate limits & errors**: headers, codes, `Retry-After`, support contact.
5. **Pagination/filtering guide** with examples for every filter enum.
6. **Webhook spec** (optional v1.1): alert delivery contract — HMAC-signed POSTs, retry policy, event types matching §12 alert types.
7. **Changelog** per API version; deprecation notices ≥ 90 days.
8. **Sandbox/test mode**: `nk_test_…` keys with seeded fixture data and no real publisher content.
9. **Terms for API consumers**: attribution requirements, no-resale-of-fulltext clause unless licensed (§security-compliance §2.6), rate-limit compliance.

## 4. Performance & degradation

- Targets (§21): dashboard/search APIs p95 < 500 ms; trending cached; pagination efficient.
- Semantic search degrades gracefully: if embedding service is down, keyword/BM25 path serves results with a `X-Degraded: semantic` header rather than failing.
- AI briefings are generated async; `/api/v1/briefings` returns `202`-style pending state via `generated_at: null, status: "generating"` until ready.
- All endpoints must return within gateway timeout (30 s); long operations are never synchronous.
