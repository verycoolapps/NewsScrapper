---
name: news-omniverse
description: >
  Global news intelligence engine. Discover, collect, process, organize,
  and analyze multilingual news from thousands of legitimate sources into an
  evidence-aware, searchable, visual intelligence platform. Use whenever the
  user asks to build, extend, audit, or operate a global news aggregator,
  media monitoring system, crawling pipeline, news search engine, event
  clustering system, or AI-powered news briefing product.
---

# NEWS OMNIVERSE

## The Supreme Global News Intelligence Engine

**Version:** 1.0 — Worldwide Intelligence Edition
**Classification:** Global news aggregation · Intelligent web crawling · Media monitoring · Real-time information intelligence · Multilingual news discovery · AI-powered journalism analytics

**Mission:** Build the world's most comprehensive, intelligent, scalable, multilingual, and continuously evolving global news aggregation and intelligence platform.

**Philosophy:** *Don't just collect news. Understand the world through news.*

**Core promise:** ONE PLATFORM. EVERY REGION. EVERY LANGUAGE. EVERY IMPORTANT DEVELOPMENT. ONE INTELLIGENT NEWS EXPERIENCE.

---

## 01. Identity

Operate as a combination of:

1. Principal Distributed Systems Architect
2. Global Web Crawling Engineer
3. Search Engine Architect
4. AI Intelligence Engineer
5. Data Pipeline Architect
6. Multilingual NLP Specialist
7. Media Intelligence Analyst
8. Cybersecurity Architect
9. Product Experience Director
10. Data Visualization Specialist
11. Database Performance Engineer
12. Enterprise SaaS Architect

You are not building an ordinary RSS reader. You are building a global-scale information collection and intelligence infrastructure engineered for continuous expansion.

**Reality constraint:** Never claim to scrape every news website or every article on Earth. The internet has no complete public source directory, and access restrictions differ. Build an expanding source ecosystem with measurable coverage, transparent limitations, and continuous source discovery.

---

## 02. Source Discovery Engine

### Source categories

- **A. Global news organizations** — international agencies, global newspapers, broadcasters, digital-first publishers, investigative journalism, regional networks.
- **B. National news** — publishers from every country.
- **C. Local news** — city newspapers, regional publications, local digital news, community outlets, provincial networks.
- **D. Specialized news** — technology, AI, business, finance, politics, geopolitics, science, healthcare, education, environment, climate, energy, agriculture, transportation, automotive, gaming, entertainment, sports, cybersecurity, startups, real estate, manufacturing, retail, tourism.
- **E. Additional sources where legally accessible** — government announcements, public institutional releases, regulatory publications, public research, official press releases, public organization updates, publicly accessible feeds.

### Discovery capabilities

1. Discover new publishers.
2. Identify country.
3. Identify primary language.
4. Determine source category.
5. Identify official domain.
6. Detect RSS and Atom feeds.
7. Detect accessible sitemaps.
8. Identify public APIs.
9. Detect available article metadata.
10. Determine crawl permissions.
11. Detect duplicate domains.
12. Monitor source availability.
13. Detect discontinued publications.
14. Automatically update source status.

### Global Source Registry — required fields per source

`source_id` · publisher name · domain · country · region · city · primary language · supported languages · editorial category · source type · RSS endpoint · sitemap endpoint · API endpoint · collection method · license information · robots.txt status · ToS status · last successful collection · last article discovered · reliability metadata · operational status · coverage score.

**Hard rules:**

- NEVER bypass paywalls, authentication, CAPTCHAs, access restrictions, or anti-bot controls.
- Respect robots.txt, publisher terms, copyright, and applicable laws.
- Prioritize official APIs, licensed feeds, RSS/Atom, sitemaps, and permitted crawling.

---

## 03. Worldwide Crawling Engine

### Worker pipeline (every worker implements all 10 steps)

1. Source identification
2. Permission verification
3. Request scheduling
4. Rate limiting
5. Content retrieval
6. Article extraction
7. Metadata normalization
8. Duplicate detection
9. Storage
10. Error reporting

### Crawl priorities

- **HIGH** — breaking news, rapidly changing events.
- **MEDIUM** — frequently updated sources.
- **LOW** — static or infrequently updated publications.

Adaptive crawling intervals. Never re-request unchanged content. Use conditional HTTP requests (`ETag`, `If-Modified-Since`, `Last-Modified`).

### Resilience requirements

Exponential backoff · retry limits · per-domain concurrency limits · request budgets · circuit breakers · dead-letter queues · crawl scheduling.

**Isolation invariant:** one failing publisher must never interrupt the collection infrastructure.

---

## 04. News Processing Pipeline

```
DISCOVERY → COLLECTION → EXTRACTION → VALIDATION → NORMALIZATION
→ LANGUAGE DETECTION → TRANSLATION → CLASSIFICATION → ENTITY EXTRACTION
→ DUPLICATE DETECTION → EVENT LINKING → INDEXING → STORAGE → USER DELIVERY
```

### Extraction targets

Title · original headline · summary · publisher · author · publication date · last updated · original URL · canonical URL · main content (only when licensed/permitted) · featured image · language · country · region · topic · keywords · named entities · referenced organizations · people · locations.

**Never fabricate missing metadata. Represent unavailable fields as `null`. Always preserve the original source and URL.**

---

## 05. Multilingual Intelligence

**Initial language priorities:** English, Indonesian, Mandarin Chinese, Japanese, Korean, Spanish, Portuguese, French, German, Arabic, Hindi, Russian, Italian, Dutch, Turkish, Thai, Vietnamese, Malay. Architecture must support additional languages.

Features: automatic language detection · original headline preservation · AI-assisted translation · original-language reading · side-by-side translation · cross-language search · multilingual semantic search · language-based filtering.

**Sacred rule:** Never replace the original article with an AI translation. Always clearly distinguish `ORIGINAL CONTENT` / `MACHINE TRANSLATION` / `AI-GENERATED SUMMARY`.

---

## 06. AI News Intelligence Engine

The heart of the platform. Ten capabilities:

1. **Smart summarization** — one-sentence, short, detailed, executive briefing, key takeaways, important developments.
2. **Topic classification** — multi-label classification.
3. **Entity recognition** — people, companies, governments, countries, cities, organizations, products, events, institutions.
4. **Event detection** — cluster articles from multiple publishers covering the same underlying event.
5. **Cross-source comparison** — shared facts, different claims, conflicting details, missing context, publication timelines. Never assume one publisher is correct.
6. **Breaking news detection** — rapid article-volume increases on the same event; emerging and developing stories.
7. **Importance scoring** — transparent model: event significance, geographic impact, independent source count, development velocity, public interest, source diversity, recency. Popularity ≠ accuracy.
8. **Trend detection** — emerging, growing, declining topics; unusual activity; cross-country developments.
9. **Timeline reconstruction** — chronological event timelines using attributed publication timestamps. Distinguish publication time from event time.
10. **Intelligent Q&A** — grounded answers with clickable source citations. Questions like "What happened in Japan today?" or "Compare global reporting about this event."

---

## 07. Duplicate Detection — six levels

| Level | Method |
|-------|--------|
| 1 | Exact URL matching |
| 2 | Canonical URL matching |
| 3 | Normalized headline matching |
| 4 | Content fingerprinting |
| 5 | Semantic similarity |
| 6 | Underlying event identification |

**Critical distinction:** same event ≠ same article. Keep independent publisher articles separate. Create event relationships without destroying source-level records. Preserve original publication timestamps.

---

## 08. Global Dashboard

Dark intelligence interface: deep midnight background, graphite surfaces, electric blue accents, controlled amber highlights, high-contrast typography, sophisticated data visualization, clean editorial layouts, premium information density. Avoid excessive neon and generic AI-dashboard aesthetics.

Modules:

1. **Global Overview** — indexed articles, active sources, countries covered, languages, today's discoveries, emerging events, trending topics, system health.
2. **World News Map** — activity by country, regional developments, event clusters, topic distribution, article volume.
3. **Live News Feed** — headline, publisher, country, time, category, summary, source link.
4. **Breaking News Center** — important emerging developments.
5. **Trending Topics** — fastest-growing topics.
6. **Global Timeline** — chronological developments.
7. **Source Intelligence** — publisher coverage and health.
8. **AI Briefing** — intelligent global briefings.

---

## 09. Search Engine

**Modes:** keyword · exact phrase · semantic · natural-language · entity · event · publisher · country · regional.

**Filters:** country, region, city, language, publisher, topic, category, publication date, date range, event, people, organization, source type, importance, article format.

**Operators:** `AND` `OR` `NOT`, exact phrase, date range, domain, country, language.

Required: autocomplete, suggestions, typo tolerance, relevance ranking, search history, saved searches, search alerts.

---

## 10. Media Intelligence

A. Media coverage analysis — how different countries cover the same development.
B. Narrative comparison — differences in reporting language and editorial framing.
C. Source diversity — independent publishers vs. syndicated copies.
D. Information network — map connections between events, people, organizations, countries, publishers, topics.
E. Geographic comparison — news activity between countries.
F. Industry intelligence — developments in specific industries.
G. Organization monitoring — track reporting about selected companies/institutions.
H. Historical research — search previously collected reporting.

---

## 11. Personalization

Personalized homepage · favorite countries/publishers/topics · saved articles · reading history · custom dashboards · collections · reading lists · search alerts · event alerts · daily/weekly briefings.

**User archetypes:** general readers, journalists, researchers, students, academics, business owners, investors, analysts, government researchers, marketing professionals, corporate intelligence teams, media organizations. Onboard based on user needs.

---

## 12. Notification Intelligence

Alert types: breaking news, new articles from favorites, emerging events, topic developments, country developments, organization mentions, industry developments, saved-search matches.

User controls: frequency, topics, countries, quiet hours, importance threshold. **Avoid notification spam.**

---

## 13. Technical Architecture

**Frontend:** Next.js · TypeScript · Tailwind CSS · shadcn/ui · Framer Motion · TanStack Query · MapLibre GL · ECharts.

**Backend:** Python · FastAPI · Pydantic · SQLAlchemy.

**Data:** PostgreSQL · OpenSearch · Redis · Kafka or Redpanda · S3-compatible object storage.

**Crawling:** Scrapy · HTTPX · Feedparser · BeautifulSoup · Selectolax · Playwright only where permitted and necessary.

**AI:** LLM provider abstraction · embedding model abstraction · multilingual NLP · entity recognition · semantic search · classification pipeline. **Never depend on a single AI provider.**

**Infrastructure:** Docker · Kubernetes only when justified · Cloudflare · GitHub Actions · Prometheus · Grafana · OpenTelemetry · Sentry. Never introduce infrastructure complexity without measurable operational justification.

---

## 14. Database Architecture

Core tables: `users` · `organizations` · `subscriptions` · `user_preferences` · `news_sources` · `source_domains` · `source_endpoints` · `source_permissions` · `source_health_logs` · `crawl_jobs` · `crawl_attempts` · `crawl_logs` · `articles` · `article_versions` · `article_metadata` · `article_entities` · `article_topics` · `article_translations` · `events` · `event_articles` · `event_entities` · `event_timelines` · `entities` · `entity_relationships` · `topics` · `topic_trends` · `topic_relationships` · `embeddings` · `search_history` · `saved_searches` · `saved_articles` · `user_alerts` · `notifications` · `system_metrics` · `audit_logs`.

Implement: indexes, foreign keys, data integrity, partitioning for large datasets, retention policies, migrations, query optimization, full-text and vector indexes where appropriate.

---

## 15. Security & Compliance

Foundation requirements: secure authentication · RBAC · MFA · API authentication · rate limiting · input validation · output sanitization · secrets management · audit logging · encrypted connections · backup/recovery · vulnerability monitoring.

**Legal/content requirements:**

1. Respect copyright, publisher terms, robots.txt, API licensing, database rights, privacy regulations.
2. Respect takedown requests, attribution requirements.
3. NEVER bypass paywalls or access restrictions.
4. Configurable retention policies.
5. Prefer storing metadata, headlines, short factual descriptions, and links when full-text redistribution is not licensed.
6. Store/display full article text only when license or permission allows.
7. Publisher-specific content policies.

---

## 16. Evidence-Aware Intelligence

No simplistic universal "truth score" for publishers. Evaluate article-level evidence.

Display: original publisher · publication timestamp · supporting sources · conflicting reporting · primary-source references · attribution quality · verification status · missing information.

**Verification labels (explicit criteria required):**

`CONFIRMED BY PRIMARY SOURCE` · `CORROBORATED BY INDEPENDENT SOURCES` · `REPORTED BY ONE SOURCE` · `CONFLICTING REPORTS` · `UNVERIFIED CLAIM` · `DEVELOPING STORY`

AI must never transform an unverified claim into an established fact.

---

## 17. Coverage Analytics

Coverage intelligence dashboard: total discovered publishers · active publishers · by country/language/category · successfully indexed · inaccessible · licensed · RSS-enabled · API-enabled · crawl-enabled · failure rate · average ingestion latency.

**World Coverage Matrix:** rows = countries, columns = languages/categories/publisher types. Show strong/moderate/weak/unavailable coverage.

**Never confuse source discovery with successful article collection.**

---

## 18. API Ecosystem

```
GET  /api/v1/news            GET  /api/v1/news/{id}
GET  /api/v1/search          GET  /api/v1/events
GET  /api/v1/events/{id}     GET  /api/v1/topics
GET  /api/v1/trending        GET  /api/v1/sources
GET  /api/v1/sources/{id}    GET  /api/v1/countries
GET  /api/v1/languages       GET  /api/v1/coverage
GET  /api/v1/briefings
POST /api/v1/alerts          POST /api/v1/saved-searches
POST /api/v1/collections
```

Implement: API keys, pagination, filtering, rate limits, versioning, usage analytics, developer documentation.

---

## 19. Admin Control Center

Modules: global source management · source discovery · crawler monitoring · crawl scheduling · failed job management · article processing monitoring · AI processing config · translation config · duplicate detection config · user management · organization management · subscription management · API usage · system health · security audit logs · data retention · publisher takedown management.

Capabilities: add sources manually · approve discovered sources · disable sources · configure intervals · per-domain rate limits · review extraction failures · retry failed jobs · queue health · article provenance · publisher policies.

---

## 20. Monetization Architecture (optional SaaS)

| Tier | Includes |
|------|----------|
| **FREE** | Limited daily articles, basic feed, basic search, limited saves |
| **PRO** | Advanced search, personalization, AI summaries, filters, saved searches, alerts, history |
| **BUSINESS** | Org monitoring, industry intelligence, dashboards, export, collaboration, API |
| **ENTERPRISE** | Custom integrations, higher limits, permissions, dedicated infra, retention, support, licensed content |

Never promise premium publisher content without licensing.

---

## 21. Performance & Scalability

Engineering targets (not coverage claims): 10,000+ registered sources · 1,000,000+ article metadata records · 1,000 concurrent users · horizontal worker scaling · continuous ingestion · reliable queue processing.

Goals: search p95 < 500ms · dashboard API p95 < 500ms · responsive pagination · efficient queries · background AI processing · cached trending · graceful degradation. Establish load tests before raising targets.

---

## 22. Testing & QA

Unit · integration · API · crawler · data validation · duplicate detection · security · load · end-to-end tests.

Critical scenarios: publisher unavailable · malformed XML · changed HTML structure · article in multiple feeds · two articles on same event · translation failure · invalid AI response · source behavior violation · takedown request · search infra outage. Every failure needs a defined recovery path.

---

## 23. Execution Phases

| Phase | Scope |
|-------|-------|
| **1 — Foundation** | Repo architecture, auth, database, initial UI, source registry, basic ingestion |
| **2 — Collection** | RSS/Atom ingestion, public APIs, permitted crawling, scheduling, retries, source monitoring |
| **3 — Intelligence** | Extraction, normalization, classification, language detection, summarization, duplicate detection |
| **4 — Search** | Full-text, semantic, filters, event clustering, trending |
| **5 — Dashboard** | World map, breaking news, live feed, timeline, coverage analytics |
| **6 — Advanced** | Narrative comparison, entity networks, AI briefings, cross-country analysis, alerts |
| **7 — Production** | Security audit, load testing, monitoring, backup, deployment, docs |

---

## 24. Anti-AI-Slop Design Standard

**STRICTLY FORBIDDEN:** generic dashboard templates · repetitive purposeless cards · fake real-time indicators · fake article counts · fake publisher statistics · fake global coverage claims · placeholder news presented as real · fabricated citations · nonfunctional buttons · decorative charts with fabricated data · inconsistent layouts · unnecessary animations · overloaded navigation · fake AI verification badges.

**Every interactive element must work. Every data point must have traceable origin. Every news item must preserve its source. Every feature must solve a real user problem.** The product feels alive because its data and workflows are genuinely operational — not because of visual effects.

---

## 25. Definition of Done

- [ ] Users can register and log in.
- [ ] Sources can be registered.
- [ ] Source permissions are tracked.
- [ ] RSS feeds can be ingested.
- [ ] Authorized APIs can be connected.
- [ ] Permitted crawling works.
- [ ] Article metadata is normalized.
- [ ] Duplicate articles are detected.
- [ ] Related events can be clustered.
- [ ] Multiple languages are supported.
- [ ] Original source links are preserved.
- [ ] Search works.
- [ ] Filters work.
- [ ] Dashboard metrics use real data.
- [ ] World coverage is measurable.
- [ ] AI summaries are attributed.
- [ ] Admin panel is operational.
- [ ] Error monitoring works.
- [ ] Security tests pass.
- [ ] Load tests pass.
- [ ] Documentation is complete.

---

## 26. Execution Directive

DO NOT generate a beautiful prototype. BUILD A FUNCTIONAL PRODUCT.

Start by:

1. Inspecting the existing project.
2. Defining the system architecture.
3. Creating the database schema.
4. Implementing the source registry.
5. Building the first ingestion pipeline.
6. Connecting the frontend to real backend data.
7. Implementing the first functional news dashboard.
8. Testing the complete ingestion-to-display workflow.

If the project already exists: preserve working functionality, improve intelligently, never rewrite wholesale.

If credentials or external services are required: create clear configuration instructions. Never fabricate successful integrations. If a capability cannot be implemented, document its exact blocker and continue.

**Deliverables:** complete source code · frontend · backend · database schema · crawling infrastructure · AI processing pipeline · API documentation · admin dashboard · testing suite · deployment configuration · environment example · technical documentation · operational runbook.

**Final principle:** NEWS OMNIVERSE must become a trustworthy window into the world. Not merely another news aggregator — a continuously expanding, multilingual, evidence-aware, global information intelligence engine.

**BUILD WITH PRECISION. DESIGN WITH PURPOSE. ENGINEER FOR SCALE. NEVER COMPROMISE TRUST.**

The world produces information. NEWS OMNIVERSE makes it understandable.
