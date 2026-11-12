# AI Intelligence Engine — NEWS OMNIVERSE

The heart of the platform (SKILL §06). All inference runs in background workers
consuming normalized articles from the queue; results are attributed and
stored alongside the source article (§24).

## 1. Architecture rules

- **Provider-agnostic:** every LLM call and every embedding call goes through
  an abstraction layer; providers/models are configuration. Never depend on a
  single AI provider (§13).
- **Grounded outputs:** summaries, classifications, and Q&A answers must trace
  to stored article records — no fabricated citations (§24, §16).
- **Original preservation:** AI output never replaces the original article;
  labels `ORIGINAL CONTENT` / `MACHINE TRANSLATION` / `AI-GENERATED SUMMARY`
  are always explicit (§05 sacred rule).
- **Degradation:** provider outage → enrichment queues retry; articles remain
  fully readable without AI enrichment (see architecture.md §6).

## 2. The ten capabilities

### 2.1 Smart summarization — tiered lengths
| Tier | Output |
|------|--------|
| One-sentence | Single headline-grade sentence |
| Short | 2–4 sentences |
| Detailed | Multi-paragraph summary |
| Executive briefing | Decision-oriented condensation |
| Key takeaways | Bulleted extractable points |
| Important developments | Delta-focused: what changed |

All tiers generated per article where licensed content or sufficient
metadata exists; every summary carries `AI-GENERATED SUMMARY` attribution and
links to its source article(s).

### 2.2 Topic classification — multi-label
Each article receives zero-or-more labels from the topic taxonomy (§14
`topics`). Multi-label, not single-class: an article can be
`technology` + `cybersecurity` + `business`. Hierarchical categories mirror
the source-category model (§02: global/national/local/specialized/official).

### 2.3 Entity recognition
Extract and link: people, companies, governments, countries, cities,
organizations, products, events, institutions (§06.3). Entities resolve to
`entities` rows; article links via `article_entities`; cross-article
structure via `entity_relationships` and `event_entities`.

### 2.4 Event detection / clustering
Cluster articles from **multiple publishers** covering the same underlying
event into `events` rows joined by `event_articles`. Clustering inputs:
semantic embeddings, shared entities, temporal proximity, cross-source
metadata. Critical distinction (§07): same event ≠ same article — independent
publisher articles stay separate records; relationships are expressed through
the event, never by merging or deleting articles.

### 2.5 Cross-source comparison
For each event cluster, compute (§06.5, §10.A/B):
- shared facts across publishers
- different claims and conflicting details
- missing context relative to the cluster
- publication timelines

**Never assume one publisher is correct.** Output is structured comparison
data for the UI, not a verdict. Narrative comparison (§10.B) surfaces
differences in reporting language and editorial framing per country/language.

### 2.6 Breaking news detection
Signal: rapid article-volume increase on the same event plus emergence of new
events. Detectors consume ingestion counters and event-article growth rates;
outputs feed the Breaking News Center (§08.4). Emerging vs. developing stories
are distinguished by velocity trend, not a single threshold.

### 2.7 Importance scoring — transparent model
Score components (§06.7), each individually inspectable — no opaque scalar:

| Factor | Meaning |
|--------|---------|
| Event significance | gravity of the underlying event |
| Geographic impact | number/size of affected regions |
| Independent source count | distinct publishers, not syndicated copies |
| Development velocity | rate of new articles on the event |
| Public interest | engagement/search signals |
| Source diversity | spread across countries/languages/categories |
| Recency | time decay on freshness |

**Popularity ≠ accuracy** (§06.7). Scores are displayed with their factor
breakdown — users can see why something ranks high.

### 2.8 Trend detection
Track topics over time (`topic_trends`): emerging, growing, declining;
unusual activity spikes; cross-country developments (§06.8, §10.E). Trends are
computed from real ingestion volumes — no decorative charts (§24).

### 2.9 Timeline reconstruction
For each event, build a chronological timeline (`event_timelines`) from
attributed publication timestamps. **Distinguish publication time from event
time** (§06.9): a report published at T describes something that may have
occurred earlier; the timeline labels which timestamp it uses and never
presents publication time as occurrence time.

### 2.10 Intelligent Q&A — grounded answers
Natural-language questions ("What happened in Japan today?", "Compare global
reporting about this event.") answered **only** from stored articles/events,
with clickable source citations back to the exact publisher records (§06.10).
Retrieval: hybrid keyword + semantic search over OpenSearch; generation
constrained to retrieved context. Unanswerable → say so; never fabricate.

## 3. Duplicate detection — six levels

Ordered cheapest → most expensive; each level can short-circuit (§07):

| Level | Method | Implementation |
|-------|--------|----------------|
| 1 | Exact URL matching | Hash/lookup on `articles.original_url` unique index |
| 2 | Canonical URL matching | Normalize `canonical_url` (strip tracking params, resolve redirects recorded at fetch); match against canonical index |
| 3 | Normalized headline matching | Lowercase, strip punctuation/whitespace, collapse wire-service prefixes; fuzzy match on normalized headline |
| 4 | Content fingerprinting | SimHash/MinHash over extracted text shingles; near-duplicate threshold |
| 5 | Semantic similarity | Embedding cosine similarity over title+summary vectors in OpenSearch kNN |
| 6 | Underlying event identification | Cluster membership: article joins an existing `events` cluster via entity+semantic+temporal evidence |

Rules:
- Dedup runs at pipeline step 8 (crawling.md §1) and again at event-linking
  stage for cross-publisher clustering.
- L1–L4 produce *duplicate/near-duplicate article* links; L6 produces *event*
  relationships. Same event ≠ same article (§07).
- Preserve original publication timestamps on every record (§07).

## 4. Verification labels — explicit criteria

Every article/event displays one of six labels with criteria (SKILL §16):

| Label | Criteria |
|-------|----------|
| `CONFIRMED BY PRIMARY SOURCE` | Claim directly attributed to an official/primary document or on-record statement (government release, company filing, named spokesperson). |
| `CORROBORATED BY INDEPENDENT SOURCES` | ≥2 independent publishers (not syndication copies) report the same core claim. |
| `REPORTED BY ONE SOURCE` | Single publisher; no corroboration yet. |
| `CONFLICTING REPORTS` | Sources materially disagree on facts; both positions shown with attribution. |
| `UNVERIFIED CLAIM` | Assertion without attribution or corroboration; flagged as claim, not fact. |
| `DEVELOPING STORY` | Event still unfolding; label while velocity is high and facts are settling. |

Hard rules (§16):
- **No universal "truth score" for publishers** — evaluation is per-article,
  per-claim, evidence-based.
- AI must **never** transform an unverified claim into an established fact.
- Display alongside: original publisher, publication timestamp, supporting
  sources, conflicting reporting, primary-source references, attribution
  quality, missing information.

## 5. Multilingual rules

### 5.1 Priority languages (18)
English, Indonesian, Mandarin Chinese, Japanese, Korean, Spanish, Portuguese,
French, German, Arabic, Hindi, Russian, Italian, Dutch, Turkish, Thai,
Vietnamese, Malay (§05). Architecture supports additional languages — the set
is configuration, not a hardcoded enum.

### 5.2 Original-vs-translation distinction (sacred rule)
- Original headline is always preserved; never replaced (§05).
- Machine translation stored in `article_translations` keyed by
  (article, target language); original-language reading and side-by-side
  translation are UI features.
- Every translated/AI-generated surface is labeled; `ORIGINAL CONTENT` /
  `MACHINE TRANSLATION` / `AI-GENERATED SUMMARY` are distinct badges (§05).
- Cross-language search and multilingual semantic search operate over
  embeddings aligned across languages; language-based filtering is a first-
  class search filter (§05, §09).

### 5.3 Language detection
Automatic detection at normalization (step 7); detected language stored on
`articles.language` with confidence; feeds classification, translation
routing, and coverage analytics (§17 languages column).

## 6. Processing flow

```mermaid
flowchart LR
    N[Normalized article] --> LD[Language detection]
    LD --> TR[Translation queue]
    LD --> CL[Classification]
    CL --> EN[Entity extraction]
    EN --> DU[Duplicate detection L1-L5]
    DU --> EV[Event linking / clustering L6]
    EV --> EM[Embedding + indexing]
    TR --> EM
    EM --> SRV[Search / Dashboard / Q&A / Alerts]
```

AI outputs are written back to PostgreSQL (`article_metadata`,
`article_entities`, `article_topics`, `article_translations`,
`embeddings`) and projected into OpenSearch for retrieval. Background
processing is async — ingestion latency for raw articles is independent of
AI enrichment latency (§21).

## 7. Failure handling

| Failure | Behavior |
|---------|----------|
| LLM provider error/timeout | Retry with backoff via alternate provider if configured; else leave enrichment pending, alert. |
| Invalid AI response (schema violation) | Reject output, retry once, then quarantine to DLQ with payload for inspection (§22 scenario). |
| Translation failure | Article remains in original language; translation retried; UI shows original only. |
| Embedding failure | Article searchable via keyword path; vector index updated on retry. |
| Clustering error | Articles remain unclustered; event detection re-runs on schedule. |

## 8. Configuration surface (Admin, §19)

Admin Control Center exposes: AI processing config, translation config,
duplicate-detection config. Model selection, thresholds (dedupe similarity,
breaking-velocity, importance weights), language enablement, and provider
endpoints are runtime configuration — changes take effect without redeploy.
