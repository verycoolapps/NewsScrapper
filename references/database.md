# Database Architecture — NEWS OMNIVERSE

Relational core in PostgreSQL (SKILL §14). All ~35 core tables, grouped by
domain. OpenSearch holds the search/vector projection; Redis holds hot state;
PostgreSQL remains the system of record.

## 1. Domain groups

### 1.1 Users / organizations / subscriptions
| Table | Purpose | Key columns |
|-------|---------|-------------|
| `users` | Accounts | id, email, password_hash, mfa_secret, status, created_at |
| `organizations` | Tenants (BUSINESS/ENTERPRISE) | id, name, plan_tier, settings |
| `subscriptions` | Plan state per user/org | id, user_id/org_id, tier, status, period |
| `user_preferences` | Personalization: favorite countries/publishers/topics, reading prefs | user_id, prefs JSONB |

Auth: secure authentication, RBAC, MFA (§15). Roles map to user archetypes
(§11) via preferences + RBAC roles, not hardcoded types.

### 1.2 Sources
| Table | Purpose |
|-------|---------|
| `news_sources` | Global Source Registry: publisher name, domain, country, region, city, primary/supported languages, editorial category, source type, license info, reliability metadata, operational status, coverage score, collection method, RSS/sitemap/API endpoint refs, last successful collection, last article discovered, robots/ToS status |
| `source_domains` | Normalized registrable domains; dedupe of duplicate domains (§02.11) |
| `source_endpoints` | Per-endpoint: type (rss/atom/sitemap/api/crawl), URL, validators (etag, last_modified), adaptive_interval_seconds, priority (HIGH/MEDIUM/LOW), request budget state |
| `source_permissions` | Permission gate consulted at crawl step 2: robots status, ToS status, allowed methods, license, takedown flags |
| `source_health_logs` | Time-series health: success/failure, latency, HTTP classes, breaker state, articles discovered |

### 1.3 Crawling
| Table | Purpose |
|-------|---------|
| `crawl_jobs` | Scheduled work units: source/endpoint ref, priority, due_at, status, adaptive interval state |
| `crawl_attempts` | One row per fetch attempt: attempt_no, http_status, bytes, duration, error class, validator values |
| `crawl_logs` | Structured event log: decisions (scheduled/skipped/denied/backoff), reasons, worker id |

Indexes: `crawl_jobs(due_at, status)` partial index on pending; `crawl_attempts(job_id, attempt_no)`; `crawl_logs(source_id, created_at)`.

### 1.4 Articles
| Table | Purpose |
|-------|---------|
| `articles` | Core record: original_url (unique), canonical_url, title, original_headline, summary, publisher/source ref, author, published_at, updated_at, language, country, region, content (nullable — only when licensed), image_url, verification_label, importance_score + factor JSONB, content_fingerprint, normalized_headline |
| `article_versions` | Content revisions when source updates an article |
| `article_metadata` | Extensible extraction fields: keywords, referenced orgs/people/locations, raw provider payloads |
| `article_entities` | article ↔ entity links with role/confidence |
| `article_topics` | article ↔ topic multi-label links with score |
| `article_translations` | (article_id, target_lang) → translated title/body, model, status; original always preserved |

Partitioning: `articles` by `published_at` RANGE monthly (1M+ records target,
§21); `article_versions` follows parent. Retention: configurable per §15.4;
partition drops for aged partitions.

### 1.5 Events / entities / topics
| Table | Purpose |
|-------|---------|
| `events` | Event clusters: title, status (emerging/developing), importance breakdown, first/last seen |
| `event_articles` | event ↔ article links with cluster confidence, role (primary/corroborating/conflicting) |
| `event_entities` | entities prominent in an event |
| `event_timelines` | Chronological entries: article ref, publication timestamp, event-time estimate, attribution |
| `entities` | People, companies, governments, countries, cities, organizations, products, events, institutions; canonical name, type, wikidata-style external ids |
| `entity_relationships` | Typed edges: works_for, located_in, mentioned_with, subsidiary_of… |
| `topics` | Topic taxonomy nodes (hierarchical via parent_id) |
| `topic_trends` | Time-bucketed volumes per topic: emerging/growing/declining signals |
| `topic_relationships` | Topic hierarchy/relatedness |

### 1.6 Embeddings / search
| Table | Purpose |
|-------|---------|
| `embeddings` | Vector store reference: owner_type (article/event/entity), owner_id, model, dims, vector (pgvector) or OpenSearch doc id |
| `search_history` | Per-user queries for autocomplete/suggestions/history (§09) |
| `saved_searches` | Saved query definitions powering search alerts |

Vector strategy: embeddings stored in OpenSearch kNN (dense_vector) as the
serving index; PostgreSQL `pgvector` column optional for transactional
similarity needs. Index per embedding model — model upgrades re-embed via
background job, never in-place mutation.

### 1.7 Alerts / notifications
| Table | Purpose |
|-------|---------|
| `saved_articles` | User saves / reading lists / collections |
| `user_alerts` | Alert definitions: type (breaking, favorite-source, event, topic, country, org-mention, industry, saved-search), filters, frequency, quiet hours, importance threshold (§12) |
| `notifications` | Delivered notifications: user, alert ref, payload, read state, dedupe key |

Notification rules (§12): respect frequency, topics, countries, quiet hours,
importance threshold — anti-spam is enforced at dispatch, not just UI.

### 1.8 Metrics / audit
| Table | Purpose |
|-------|---------|
| `system_metrics` | Ingestion counters, latencies, queue depths, coverage aggregates feeding dashboards (§17, §21) |
| `audit_logs` | Immutable audit: auth events, admin mutations, permission changes, takedown actions (§15) |

## 2. DDL sketch (core tables)

```sql
CREATE TABLE news_sources (
    id            BIGSERIAL PRIMARY KEY,
    publisher_name TEXT NOT NULL,
    primary_domain TEXT NOT NULL REFERENCES source_domains(domain),
    country       CHAR(2), region TEXT, city TEXT,
    primary_language CHAR(5), supported_languages TEXT[],
    editorial_category TEXT, source_type TEXT NOT NULL,
    license_info  JSONB,
    robots_txt_status TEXT, tos_status TEXT,
    collection_method TEXT NOT NULL,          -- rss|api|sitemap|crawl
    reliability_metadata JSONB,
    operational_status TEXT NOT NULL DEFAULT 'discovered',
    coverage_score NUMERIC(5,2),
    last_successful_collection TIMESTAMPTZ,
    last_article_discovered TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE source_endpoints (
    id BIGSERIAL PRIMARY KEY,
    source_id BIGINT NOT NULL REFERENCES news_sources(id) ON DELETE CASCADE,
    endpoint_type TEXT NOT NULL,              -- rss|atom|sitemap|api|html
    url TEXT NOT NULL,
    etag TEXT, last_modified TEXT,
    adaptive_interval_seconds INT NOT NULL DEFAULT 3600,
    priority TEXT NOT NULL DEFAULT 'MEDIUM' CHECK (priority IN ('HIGH','MEDIUM','LOW')),
    request_budget_per_hour INT,
    UNIQUE (source_id, endpoint_type, url)
);

CREATE TABLE articles (
    id BIGSERIAL,
    source_id BIGINT NOT NULL REFERENCES news_sources(id),
    original_url TEXT NOT NULL,
    canonical_url TEXT,
    title TEXT NOT NULL,
    original_headline TEXT,
    summary TEXT,
    author TEXT,
    published_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ,
    language CHAR(5),
    country CHAR(2), region TEXT,
    content TEXT,                             -- NULL unless licensed
    image_url TEXT,
    verification_label TEXT CHECK (verification_label IN
        ('CONFIRMED_BY_PRIMARY_SOURCE','CORROBORATED_BY_INDEPENDENT_SOURCES',
         'REPORTED_BY_ONE_SOURCE','CONFLICTING_REPORTS',
         'UNVERIFIED_CLAIM','DEVELOPING_STORY')),
    importance_score NUMERIC(6,3),
    importance_factors JSONB,
    content_fingerprint TEXT,
    normalized_headline TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, published_at)
) PARTITION BY RANGE (published_at);

CREATE UNIQUE INDEX articles_original_url_key
    ON articles (original_url, published_at);
CREATE INDEX articles_canonical_idx ON articles (canonical_url);
CREATE INDEX articles_pubdate_idx  ON articles (published_at DESC);
CREATE INDEX articles_lang_country ON articles (language, country);
CREATE INDEX articles_normalized_headline_idx
    ON articles (normalized_headline);
```

## 3. Index strategy

| Access path | Index |
|-------------|-------|
| Article by URL / canonical | unique btree on `original_url`; btree on `canonical_url` |
| Dedupe L3 normalized headline | btree on `normalized_headline` (+ pg_trgm for fuzzy) |
| Dedupe L4 fingerprint | btree on `content_fingerprint` |
| Feed pagination | `(published_at DESC, id DESC)` composite per partition |
| Country/language filters | composite `(country, language, published_at)` |
| Event clustering lookups | `event_articles(article_id)`, `event_articles(event_id)` |
| Topic trend rollups | `topic_trends(topic_id, bucket_start)` |
| User library | `saved_articles(user_id, saved_at DESC)` |
| Alerts dispatch | `user_alerts(enabled, type)` partial; `notifications(user_id, created_at)` |
| Audit | `audit_logs(actor_id, created_at)`, `audit_logs(entity, entity_id)` |
| Crawl scheduling | partial `crawl_jobs(status, due_at) WHERE status='scheduled'` |

## 4. Full-text + vector strategy

- **Primary serving index: OpenSearch.** Documents per article carry title,
  summary, original headline, keywords, entity names, topic labels — analyzed
  per language. Filters (country, language, publisher, date range, category,
  importance) are keyword fields.
- **Full-text:** OpenSearch `match`/`multi_match` with language-specific
  analyzers; PostgreSQL `tsvector` generated column as fallback when search
  infra is degraded (architecture.md §6).
- **Vector:** `dense_vector` kNN fields for title+summary and event-cluster
  centroids; cosine similarity; model-versioned fields so re-embedding is
  additive.
- **Autocomplete/suggestions:** OpenSearch completion suggester + edge-ngram
  on publisher/topic names; `search_history` feeds personalization.

## 5. Partitioning, retention, migrations

- **Partitioning:** `articles`, `article_versions`, `crawl_attempts`,
  `crawl_logs`, `source_health_logs`, `system_metrics`, `audit_logs` by time
  (monthly for articles, daily for logs/metrics). Partition lifecycle managed
  by a scheduled job.
- **Retention (§15.4 configurable):** hot window in PG; aged partitions either
  archived to S3 (parquet/JSONL) then dropped, or retained per policy.
  Full-text retention may exceed metadata retention where licenses differ —
  `articles.content` is NULL unless licensed (§15.5–6).
- **Migrations:** Alembic (SQLAlchemy) with expand/contract pattern; every
  migration tested against partitioned tables; backfills run as background
  jobs, never inline in deploy.
- **Integrity:** FKs enforced; `ON DELETE` policy per domain (CASCADE only
  within source-owned children like endpoints/health logs; RESTRICT/SET NULL
  for user-facing content to preserve audit trail).

## 6. Query optimization notes

- API read paths: relational queries hit replicas; hot aggregates served from
  Redis (`system_metrics` rollups refreshed on a schedule — cached trending,
  §21).
- Avoid N+1 on article lists: fetch article + source + topic/entity labels in
  batched joins or via OpenSearch hydrated docs.
- `event_articles` cluster fetches use `event_id` index; timeline queries use
  `(event_id, published_at)`.
- Search latency target p95 < 500ms (§21): enforce `from+size` caps, use
  `search_after`/ PIT pagination, terminate-after for dashboard aggregates.
- Coverage matrix (§17) is a materialized rollup refreshed periodically —
  never computed per-request over raw crawl logs.
- Dashboard API p95 < 500ms: pre-aggregate Global Overview counters into
  Redis at ingestion commit boundaries.

## 7. Consistency model

- Crawl writes: PG commit → queue publish (transactional outbox pattern for
  `articles.normalized` events to avoid dual-write loss).
- AI enrichment: idempotent upserts keyed by (article_id, task_type, model);
  re-runs overwrite derived fields safely.
- OpenSearch projection is eventually consistent; PG is authoritative — a
  reindex job rebuilds the index from PG at any time.
