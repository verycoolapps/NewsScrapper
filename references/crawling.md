# Crawling Engine — NEWS OMNIVERSE

Worldwide collection infrastructure. Scope: RSS/Atom feeds, licensed/public
APIs, sitemaps, and permitted crawling — never paywall/auth/CAPTCHA bypass
(SKILL §02, §03).

## 1. The mandatory 10-step worker pipeline

Every worker — regardless of source type — implements all ten steps, in order
(SKILL §03):

| # | Step | Responsibility |
|---|------|----------------|
| 1 | Source identification | Resolve the `news_sources`/`source_endpoints` record; bind collection method (RSS, API, sitemap, crawl). |
| 2 | Permission verification | Check `source_permissions`: robots.txt status, ToS status, license info, method allowlist. No permission → no fetch. |
| 3 | Request scheduling | Dequeue/claim due jobs from `crawl_jobs` honoring priority + adaptive interval. |
| 4 | Rate limiting | Acquire per-domain token; respect per-domain concurrency cap and request budget. |
| 5 | Content retrieval | HTTPX fetch (Scrapy for HTML crawling). Conditional headers when available. |
| 6 | Article extraction | Feedparser for feeds; Selectolax/BeautifulSoup for permitted HTML; structured metadata for APIs. |
| 7 | Metadata normalization | Map to extraction targets (§04): title, original headline, summary, publisher, author, pub date, updated, original URL, canonical URL, content-if-licensed, image, language, country, region, topic, keywords, entities. Missing → `null`. Never fabricate. |
| 8 | Duplicate detection | Run six-level dedupe (§7 below). Record links, do not destroy source records. |
| 9 | Storage | Persist article + `crawl_attempts`/`crawl_logs` rows; publish normalized event to queue for AI enrichment. |
| 10 | Error reporting | Classify failure, update `source_health_logs`, `crawl_attempts`, circuit-breaker state; emit metrics/traces. |

## 2. Pipeline definition (reference pseudocode)

```python
async def crawl_worker(job: CrawlJob) -> AttemptResult:
    source = await sources.get(job.source_id)            # 1
    perm = await permissions.check(source)                # 2
    if not perm.fetch_allowed:
        return AttemptResult.SKIPPED_PERMISSION

    await scheduler.claim(job)                           # 3
    await rate_limiter.acquire(source.domain)            # 4
    try:
        resp = await httpx.get(                          # 5
            source.endpoint, headers=conditional_headers(source),
            timeout=per_domain_timeout(source))
        if resp.status_code == 304:
            return AttemptResult.NOT_MODIFIED
        items = extract_items(resp, source.method)       # 6
        for item in items:
            norm = normalize(item)                       # 7  -> null-safe
            await dedupe.run(norm)                       # 8  -> L1..L6
            await articles.store(norm, source)           # 9
            await queue.publish("articles.normalized", norm)
    except FetchError as e:
        await errors.report(job, source, e)              # 10
        return AttemptResult.FAILED(e)
```

## 3. Priority model

| Priority | Applies to | Scheduling behavior |
|----------|-----------|---------------------|
| **HIGH** | Breaking news, rapidly changing events | Shortest adaptive interval; may preempt lower-priority due jobs within per-domain caps. |
| **MEDIUM** | Frequently updated sources | Default interval; balanced share of worker pool. |
| **LOW** | Static or infrequently updated publications | Longest interval; backfill/batch lanes. |

Priority is assigned from source volatility signals (update frequency observed
in `source_health_logs`, event velocity) and can be overridden per-source in
the Admin Control Center (§19: configure intervals, per-domain rate limits).

## 4. Adaptive intervals

- Each endpoint carries `adaptive_interval_seconds`, seeded from observed
  `Last-Modified`/ETag change frequency.
- Interval shrinks when changes are detected frequently; grows (up to a
  configured max) when content is stable.
- Never re-request unchanged content: conditional requests below are
  mandatory, not optional (§03).

## 5. Conditional HTTP requests

| Validator | Request header | Stored from |
|-----------|---------------|-------------|
| ETag | `If-None-Match` | previous `ETag` response header |
| Last-Modified | `If-Modified-Since` | previous `Last-Modified` response header |

On `304 Not Modified`: update `last_checked_at` only; no extraction, no
storage. Validators are per-endpoint state in `source_endpoints` /
`crawl_attempts`. Both validators should be sent when available; honor
`Cache-Control`/`Vary` where present.

## 6. Resilience mechanics

### 6.1 Exponential backoff
`delay = min(base * 2**attempt + jitter, max_delay)` with full jitter. Applied
to 5xx, 429, and network errors. `Retry-After` header takes precedence over
computed backoff.

### 6.2 Retry limits
- Max attempts per job (configurable, e.g. 5) tracked in `crawl_attempts`.
- After exhaustion → dead-letter queue (§6.6). Retry count never resets
  silently; admin "retry failed jobs" (§19) creates a new attempt lineage.

### 6.3 Per-domain concurrency
Hard cap on simultaneous in-flight requests per registrable domain (default
1–4 depending on measured tolerance). Enforced via Redis token bucket keyed
`crawl:domain:<domain>`; token acquisition happens at pipeline step 4.

### 6.4 Request budgets
Per-source and per-domain byte/time budgets per window (hour/day). Budget
exhaustion → job deferred to next window, recorded in `crawl_logs`. Budgets
protect both our infra and publisher goodwill.

### 6.5 Circuit breakers
Per-domain breaker: CLOSED → OPEN after N consecutive failures or error-rate
threshold → HALF_OPEN probe after cooldown. State persisted so restarts don't
reset it. OPEN domain skips scheduling; `source_health_logs` records
transitions. One failing publisher must never interrupt collection
infrastructure (§03 isolation invariant).

### 6.6 Dead-letter queues
Exhausted or poison jobs land in DLQ with full context (source, endpoint,
last error, attempt history). DLQ is an admin workflow (§19 failed job
management): inspect, fix, replay, or discard. Nothing silently drops.

## 7. Failure isolation

- Workers are stateless processes; jobs carry all context.
- Per-source timeouts; a hung publisher cannot stall the fleet (async
  concurrency + per-job deadline).
- Extraction failures are isolated per article: one malformed item does not
  fail the whole feed pull — the item goes to DLQ/`crawl_logs` with reason.
- Scrapy/HTTPX/Playwright run in separate worker pools so a Playwright crash
  cannot affect feed polling.

## 8. Permission & robots model (hard rules)

Source record carries `robots_txt_status`, `tos_status`, `license_information`,
`collection_method` (Global Source Registry, §02). Rules:

1. **NEVER bypass** paywalls, authentication, CAPTCHAs, access restrictions,
   or anti-bot controls (§02 hard rule, §15.3).
2. Respect `robots.txt` — a `Disallow` for our UA/path means no fetch; honor
   `Crawl-delay` as a floor on interval.
3. Respect publisher ToS and licensing: RSS/Atom, official APIs, sitemaps,
   and permitted crawling only (§02 hard rule).
4. Playwright is used **only where permitted and necessary** (§13) — e.g.
   JS-rendered pages on sources whose ToS permits automated access; never to
   defeat bot detection.
5. `source_permissions` is the single gate consulted at pipeline step 2.
   Permission state changes propagate without redeploy.

## 9. Extraction tooling assignment

| Method | Tool | When |
|--------|------|------|
| RSS/Atom | Feedparser | Primary path for feed endpoints |
| Public/licensed API | HTTPX + Pydantic parsers | Official APIs, press-release feeds |
| Sitemap | HTTPX + XML parser | Article URL discovery |
| Permitted HTML | Scrapy (crawl) + Selectolax / BeautifulSoup (parse) | Sources without feeds, ToS-permitted |
| JS-rendered | Playwright (gated worker pool) | Only where permitted & necessary |

## 10. Health & observability per source

`source_health_logs` records: last successful collection, last article
discovered, failure counts, latency, HTTP status classes, breaker state.
`source_endpoints` stores validators (ETag/Last-Modified). Coverage analytics
(§17) distinguishes **source discovery** from **successful article
collection** — an endpoint can be discovered yet inaccessible; never conflate.

## 11. Crawl job lifecycle

```mermaid
stateDiagram-v2
    [*] --> SCHEDULED
    SCHEDULED --> PERMISSION_CHECK
    PERMISSION_CHECK --> DENIED: no permission
    PERMISSION_CHECK --> FETCHING
    FETCHING --> EXTRACTING: 200
    FETCHING --> BACKOFF: 5xx/429/net err
    BACKOFF --> FETCHING: attempts left
    BACKOFF --> DLQ: attempts exhausted
    EXTRACTING --> NORMALIZED
    NORMALIZED --> STORED --> ENQUEUED
    EXTRACTING --> DLQ: poison item
    ENQUEUED --> [*]
    DENIED --> [*]
```

## 12. Testing hooks (§22 critical scenarios)

- publisher unavailable → breaker opens, others unaffected
- malformed XML feed → per-item failure, feed survives
- changed HTML structure → extraction failure logged, source flagged
- article appearing in multiple feeds → dedupe links, both source records kept
- source behavior violation (robots/ToS) → permission revoked, fetch stops

## 13. Source onboarding checklist

When a source is approved (§19 "approve discovered sources"), the registry
entry must have: verified official domain, country/region/city resolved,
primary + supported languages, editorial category, chosen collection method,
endpoint rows with validators, a `source_permissions` row carrying robots/ToS
status and license info, an initial priority, and a baseline interval. A
source without a permission row is never scheduled — step 2 denies it.

Discovered-but-unverified sources stay in `operational_status =
'discovered'` and are excluded from collection until approved; the coverage
matrix reports them separately from active publishers (§17: discovery ≠
collection).
