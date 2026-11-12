# Operational Runbook

Derived from SKILL.md §13 (Technical Architecture), §14 (Database), §15 (Security), §21 (Performance), §22 (Testing & QA — the 10 critical scenarios), §23 (Phases), and §25 (Definition of Done).

## 1. Deployment Topology

```mermaid
flowchart TB
  subgraph Edge
    CF[Cloudflare CDN/WAF]
  end
  subgraph App[Application Tier]
    WEB[Next.js Frontend]
    API[FastAPI Backend]
    ADM[Admin UI/API]
  end
  subgraph Data[Data Tier]
    PG[(PostgreSQL)]
    OS[(OpenSearch)]
    RD[(Redis)]
    KF[(Kafka/Redpanda)]
    S3[(S3-compatible storage)]
  end
  subgraph Workers[Worker Tier]
    CR[Crawler Workers]
    AI[AI/Translation Workers]
    PX[Processing Workers]
  end
  CF --> WEB --> API
  API --> PG & OS & RD
  API -->|enqueue| KF
  KF --> CR & AI & PX
  CR -->|fetch| EXT((Publisher sites))
  AI --> LLM((LLM providers))
  PX --> S3
  CR & AI & PX --> PG
```

- **Frontend**: Next.js + TypeScript + Tailwind + shadcn/ui, deployed as Node service or static export behind CDN.
- **Backend**: FastAPI (Python), SQLAlchemy, Pydantic; horizontal replicas behind LB.
- **Crawling**: Scrapy/HTTPX/Feedparser/BeautifulSoup/Selectolax workers; Playwright only where permitted (§13). Workers are stateless; all state in Postgres/Kafka/Redis.
- **Data**: PostgreSQL (system of record), OpenSearch (search/semantic), Redis (cache, rate limits, queues), Kafka/Redpanda (pipeline events), S3-compatible (licensed full-text payloads, exports, backups).
- **Infra**: Docker images; Kubernetes only when scale justifies (§13). Cloudflare in front. GitHub Actions CI. Prometheus + Grafana + OpenTelemetry + Sentry.
- **Scaling**: horizontal crawler/AI workers; partitioning on `articles`/`crawl_logs` once volume demands (§14).

## 2. Environment Configuration Template

`.env.example` — copy to `.env`; never commit real values. Names only, per §25 deliverable "environment example".

```bash
# ── Core ──────────────────────────────────────────────
APP_ENV=production            # development|staging|production
APP_BASE_URL=https://news.example.com
API_BASE_URL=https://api.news.example.com
SECRET_KEY=                   # session/JWT signing — generate: openssl rand -hex 32
JWT_ACCESS_TTL_SECONDS=900
JWT_REFRESH_TTL_SECONDS=1209600
FIELD_ENCRYPTION_KEY=         # for sensitive columns, if used

# ── Database ──────────────────────────────────────────
DATABASE_URL=postgresql+psycopg://user:pass@postgres:5432/newsomniverse
DATABASE_POOL_SIZE=20
DATABASE_MAX_OVERFLOW=10
POSTGRES_BACKUP_DIR=/backups

# ── Search ────────────────────────────────────────────
OPENSEARCH_URL=https://opensearch:9200
OPENSEARCH_USERNAME=
OPENSEARCH_PASSWORD=
EMBEDDING_PROVIDER=           # e.g. openai|cohere|local — abstraction required (§13)
EMBEDDING_MODEL=
EMBEDDING_API_KEY=
SEMANTIC_SEARCH_ENABLED=true

# ── Cache / Queue ─────────────────────────────────────
REDIS_URL=redis://redis:6379/0
KAFKA_BOOTSTRAP_SERVERS=kafka:9092        # or REDPANDA_BROKERS
QUEUE_CRAWL_TOPIC=crawl.jobs
QUEUE_PROCESS_TOPIC=process.jobs
QUEUE_DLQ_TOPIC=dead.letters

# ── Object storage ────────────────────────────────────
S3_ENDPOINT=                   # leave empty for AWS
S3_BUCKET=news-assets
S3_ACCESS_KEY=
S3_SECRET_KEY=
S3_REGION=auto

# ── AI providers (never single-provider, §13) ─────────
LLM_PRIMARY_PROVIDER=
LLM_PRIMARY_MODEL=
LLM_PRIMARY_API_KEY=
LLM_FALLBACK_PROVIDER=
LLM_FALLBACK_MODEL=
LLM_FALLBACK_API_KEY=
AI_DAILY_TOKEN_BUDGET=500000

# ── MT / translation ─────────────────────────────────
MT_PRIMARY_PROVIDER=
MT_PRIMARY_API_KEY=
MT_FALLBACK_PROVIDER=
MT_FALLBACK_API_KEY=
TRANSLATION_ENABLED=true

# ── Crawler ───────────────────────────────────────────
CRAWLER_USER_AGENT="NewsOmniverseBot/1.0 (+https://news.example.com/bot)"
CRAWLER_GLOBAL_CONCURRENCY=50
CRAWLER_PER_DOMAIN_CONCURRENCY=2
CRAWLER_DEFAULT_INTERVAL_SECONDS=3600
CRAWLER_MAX_RETRIES=3
CRAWLER_BACKOFF_BASE_SECONDS=30
CRAWLER_CIRCUIT_BREAKER_THRESHOLD=5
CRAWLER_CIRCUIT_BREAKER_RESET_SECONDS=300
RESPECT_ROBOTS_TXT=true         # must stay true in production
PLAYWRIGHT_ENABLED=false        # only where permitted & necessary (§13)

# ── Rate limits (API) ─────────────────────────────────
API_RATE_LIMIT_READ=60          # req/min per key
API_RATE_LIMIT_WRITE=10
API_RATE_LIMIT_SEARCH=30
API_KEY_PREFIX=nk_live_
API_KEY_PREFIX_TEST=nk_test_

# ── Retention ─────────────────────────────────────────
RETENTION_ARTICLE_METADATA_MONTHS=24
RETENTION_FULLTEXT_MONTHS=12    # per license; metadata-only sources unaffected
RETENTION_SEARCH_HISTORY_DAYS=90
RETENTION_AUDIT_LOGS_MONTHS=12
RETENTION_CRAWL_LOGS_DAYS=30

# ── Observability ─────────────────────────────────────
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_SERVICE_NAME=news-api
SENTRY_DSN=
PROMETHEUS_PORT=9090
GRAFANA_URL=http://grafana:3000

# ── Email / notifications ─────────────────────────────
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=
SMTP_FROM=no-reply@news.example.com

# ── Takedown / legal ──────────────────────────────────
TAKEDOWN_CONTACT_EMAIL=legal@news.example.com
TAKEDOWN_SLA_HOURS=72
SUPPORT_EMAIL=support@news.example.com

# ── Feature flags ─────────────────────────────────────
FEATURE_SEMANTIC_SEARCH=true
FEATURE_BRIEFINGS=true
FEATURE_WORLD_MAP=true
FEATURE_ADMIN_CENTER=true
FEATURE_API_V1=true
```

## 3. Crawler Operations

### 3.1 Schedule tuning

- Default interval 1 h; adaptive per §03: unchanged (304/ETag match) → interval ×1.5 up to 24 h; changed → interval ×0.75 down to source floor (5 min for HIGH priority).
- HIGH priority = breaking-news sources; promote manually during major events, demote after 48 h quiet.
- Per-domain limits: `CRAWLER_PER_DOMAIN_CONCURRENCY` (default 2) and per-source `crawl_interval` override in admin Module 4.
- Never crawl faster than `Crawl-delay` in robots.txt; admin UI warns when configured interval violates it.

### 3.2 Dead-letter triage (daily)

1. Open Admin → Failed Job Management; group by error class.
2. `timeout`/`5xx`: check publisher status page; bulk retry after backoff reset.
3. `parse_error`: inspect extraction diff in Processing Monitor; if site redesign, update extractor selector (versioned), then retry with override (audited).
4. `robots_disallowed`: verify robots change; if publisher now disallows, set `crawl_enabled=false` — do not retry.
5. `rate_limited`: raise interval or lower priority; check per-domain budget.
6. `circuit_open`: wait for reset window or reset manually after confirming publisher recovered.
7. `permission_revoked`: source suspended by takedown/ToS — do not retry; route to Module 17.

### 3.3 Circuit-breaker reset

- Breaker opens after `CRAWLER_CIRCUIT_BREAKER_THRESHOLD` consecutive failures; auto-resets after `CRAWLER_CIRCUIT_BREAKER_RESET_SECONDS` with a half-open probe.
- Manual reset (admin Module 3/5): only after verifying publisher reachable and `source_permissions` still valid. Reset is audited. Never reset a breaker whose cause is `robots_disallowed` or `permission_revoked`.

### 3.4 Good-citizen rules

- Identify honestly via `CRAWLER_USER_AGENT` with contact URL; honor opt-out emails.
- Respect `Crawl-delay`, per-domain concurrency, quiet windows.
- Conditional requests always (`ETag`, `If-Modified-Since`) — never re-fetch unchanged content (§03).

## 4. Ingestion-to-Display Smoke Test

Run after every deploy touching ingestion, processing, or API. Target < 15 min.

```mermaid
flowchart LR
  A[Pick test source] --> B[Enqueue crawl job]
  B --> C[Verify crawl_attempts row]
  C --> D[Verify articles row + metadata]
  D --> E[Verify dedup: re-run → no dup]
  E --> F[Verify language detect + label]
  F --> G[Verify translation row if non-EN]
  G --> H[Verify OpenSearch doc]
  H --> I[GET /api/v1/news shows item]
  I --> J[UI feed shows item w/ source link]
```

Steps:

1. Choose a stable, permission-clean RSS source (prefer a `licensed`/`rss_enabled` fixture source seeded in staging).
2. Admin → Crawl Scheduling → "crawl now"; note job id.
3. Crawler Monitoring: job reaches `succeeded`; `crawl_attempts` has HTTP 200, bytes, duration.
4. DB: `articles` row exists with `original_url`, `published_at`, `language`, `country`, `content_storage` per source policy; `article_metadata` normalized; no fabricated fields (§04 — nulls where unavailable).
5. Re-run same job: dedup levels 1–3 prevent a second article row (check `duplicate_detection` logs).
6. Processing Monitor: item passed LANGUAGE DETECTION; if non-English, `article_translations` row exists with `kind=machine_translation` and label renders correctly.
7. OpenSearch: `GET /api/v1/search?q=<title words>` returns the article; `GET /api/v1/news/{id}` returns full provenance payload.
8. UI: Live News Feed shows headline, publisher, country, time; clicking opens `original_url` on publisher domain (never our cached copy unless licensed).
9. Negative checks: invalid API key → 401; missing scope → 403; rate-limit headers present.

Pass criteria: all steps green; any fabricated metric or missing provenance = fail (§24).

## 5. Failure-Scenario Recovery (SKILL.md §22 — all 10)

| # | Scenario | Detection | Recovery path |
|---|----------|-----------|----------------|
| 1 | Publisher unavailable | `crawl_attempts` timeouts/5xx; health → `failing` | Exponential backoff; breaker opens at threshold; interval auto-extends; source marked `degraded`; after N days → `discontinued` candidate. No data loss — other sources unaffected (isolation invariant §03). |
| 2 | Malformed XML (RSS/Atom) | `parse_error` in processing monitor | Feedparser/defused-XML rejects safely; job → retry once; persistent → dead-letter + alert; source flagged for endpoint review; sibling feeds unaffected. |
| 3 | Changed HTML structure | `extraction_confidence` drop; `parse_error` spike per source | Versioned extractor selectors; admin reviews extraction diff; deploy selector fix; retry with override; if unfixable, downgrade source to metadata-only via RSS/sitemap path. |
| 4 | Article in multiple feeds | Dedup level 1–3 match | Keep single `articles` row; record all source/feed sightings in provenance; canonical URL wins; no duplicate event inflation. |
| 5 | Two articles, same event | Dedup level 6 clustering | Create/link `events` + `event_articles`; publisher articles stay separate with original timestamps (§07). Verify `independent_sources` count excludes syndication dupes. |
| 6 | Translation failure | `article_translations` error rows; MT provider 5xx | Fallback provider (§13); retry queue; if both fail → article ships with original language only, `translations=[]`, UI shows original + "translation unavailable" — never blocks delivery. |
| 7 | Invalid AI response | Schema validation fail on LLM output | Retry with stricter prompt/temperature; validation rejects → summary/briefing omitted for that item; event/article unaffected; alert if failure rate > threshold; provider fallback engaged. |
| 8 | Source behavior violation | Unexpected content type, auth wall, CAPTCHA, robots change | Immediate `crawl_enabled=false`; job cancelled; admin notified; legal/compliance review; never attempt bypass (§15 hard rule). |
| 9 | Takedown request | Module 17 case created | SOP §security-compliance §2.4: verify → action ≤ 72 h → tombstone + suppression list → confirm. Article returns 410; re-ingestion blocked. |
| 10 | Search infra outage | OpenSearch red/unreachable; search p95 breach | API degrades: keyword path → DB `tsvector` fallback or cached results with `X-Degraded` header; semantic off; dashboard shows banner; never fabricate results (§24). Restore: cluster health → reindex from Postgres if needed. |

## 6. Monitoring & Alerting Hooks

### 6.1 Prometheus metrics (exposition names)

```
news_articles_ingested_total{source,language,category}
news_crawl_jobs_total{source,status}
news_crawl_duration_seconds{source,quantile}
news_crawl_attempts_total{source,result}        # result: success|timeout|5xx|parse_error|robots|rate_limited
news_pipeline_stage_duration_seconds{stage}
news_pipeline_stage_errors_total{stage,error_class}
news_dedup_matches_total{level}
news_events_created_total
news_translations_total{provider,status}
news_ai_requests_total{provider,task,status}
news_ai_tokens_total{provider,task}
news_api_requests_total{endpoint,method,status,tier}
news_api_request_duration_seconds{endpoint,quantile}
news_api_rate_limited_total{key_class}
news_queue_depth{queue}
news_queue_consumer_lag{queue,consumer}
news_dlq_size{queue}
news_circuit_breaker_state{source}              # 0=closed 1=half_open 2=open
news_source_health{source,status}
news_open_search_cluster_health                  # 0=green 1=yellow 2=red
news_postgres_connections{state}
news_backup_last_success_timestamp
news_takedown_cases_open
news_takedown_sla_breaches_total
```

### 6.2 Grafana dashboards

1. **Global Overview** — ingestion rate, active sources, countries/languages, queue depth, error rates, p95 latencies.
2. **Crawler Health** — per-source success rate, breaker states, interval distribution, top failing sources.
3. **Pipeline & AI** — stage latency/errors, LLM/MT token usage & cost, dedup match rates.
4. **API & Users** — requests by endpoint/tier, rate-limit hits, latency heatmaps, active users.
5. **Coverage Analytics** — mirrors §17 metrics: source status distribution, coverage bands, failure rate, ingestion latency.
6. **Infra & Cost** — node resources, DB pool, OpenSearch health, S3 growth, AI spend vs budget.
7. **Compliance** — takedown SLA, audit-log anomalies, retention job runs.

### 6.3 OpenTelemetry spans

- `crawl.job` (attrs: source_id, job_id, priority) → `crawl.fetch` → `crawl.parse` → `article.extract`
- `pipeline.stage` per §04 stage (attrs: article_id, stage, duration, outcome)
- `ai.task` (attrs: provider, model, task_type, tokens, latency)
- `api.request` (attrs: route, method, status, tier) — baggage carries `request_id`, `user_id`/`org_id`, `api_key_id`.
- `db.query` slow-query threshold 250 ms; `search.query` attrs: mode, filters count, hits.

### 6.4 Sentry

- Projects: `web` (frontend), `api` (FastAPI), `workers` (crawler/AI), with release tags per deploy.
- Alert rules: new issue in `workers` → page; error-rate spike > 2% on `api` → page; crawler `parse_error` rate anomaly → ticket.
- PII scrubbing before send; article content fields excluded from breadcrumbs (§security-compliance §2.9).

### 6.5 Alert routing

- Page (PagerDuty/Opsgenie or equivalent): API down, DB down, queue consumer lag > 15 min, breaker-open count > 10% of active sources, takedown SLA breach.
- Ticket: degraded source count rising, AI fallback engaged, retention job failures, disk > 75%.
- Dashboard-only: coverage band changes, trending anomalies.

## 7. Backup & Restore

- **Postgres**: WAL archiving + daily base backup; `pg_basebackup` or managed snapshots. Restore test quarterly: spin up staging, restore latest, run smoke test §4.
- **OpenSearch**: snapshots to S3 repository (daily, min 7 retained); restore = snapshot restore + alias swap.
- **Redis**: AOF + RDB; queues backed by Kafka so Redis loss is tolerable — rebuild rate-limit counters, caches warm naturally.
- **S3**: versioning + lifecycle; cross-region replication for licensed full-text.
- **Secrets/config**: sealed in secrets manager; `.env` never backed up to repo.
- **Restore order**: Postgres → OpenSearch (reindex from PG if snapshot stale) → Kafka topics (replay from earliest retained offset if needed) → workers → API.
- Documented RPO/RTO targets in §security-compliance §1.10; verify against actuals each quarter.

## 8. Takedown SOP (summary — full rules in §security-compliance §2.4)

1. Intake: `legal@` mailbox + web form + admin Module 17 manual entry. Log immediately with timestamp and requester identity.
2. Verify: requester is rights holder or authorized agent; capture evidence link.
3. Action within SLA (72 h default, `TAKEDOWN_SLA_HOURS`): remove article → tombstone record (source id, URL, date, reason — no content) → add URL/domain to suppression list → purge derived data (translations, embeddings, summaries) per retention rules.
4. Confirm to requester; log in `audit_logs`.
5. Counter-notice path: publisher may contest; admin reviews; restoration only on valid counter-notice per jurisdiction.
6. Prevention: suppression list enforced at ingestion before storage; weekly report of open/closed cases to compliance owner.

## 9. Routine Operations Checklist

**Daily**: dead-letter triage §3.2; breaker states review; takedown SLA check; AI spend vs budget; failed-job aging report.
**Weekly**: coverage matrix review; source `discontinued` candidates; retention job verification; dependency CVE scan results; backup integrity check.
**Monthly**: restore drill (staging); load-test baseline refresh; secret rotation review; audit-log anomaly review; robots.txt re-crawl for top 100 sources.
**Per release**: smoke test §4; migration dry-run on staging copy; rollback plan documented; feature-flag defaults verified.
