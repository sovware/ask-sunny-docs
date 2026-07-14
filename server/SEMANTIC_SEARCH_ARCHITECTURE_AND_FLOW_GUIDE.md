# Ask Sunny Semantic Search Architecture and Flow Guide

## 1. Purpose

Ask Sunny is a Bun/Hono conversational RAG server for a single WordPress installation. WordPress and Directorist remain the editorial source of truth. The server receives normalized source records from the WordPress plugin, creates searchable representations, performs structured and semantic retrieval, and supplies grounded evidence to the configured chat-generation provider.

This guide defines the semantic indexing, retrieval, chat, and failure flows. Detailed tables and HTTP payloads remain authoritative in [`SERVER_DATABASE_SCHEMA.md`](SERVER_DATABASE_SCHEMA.md) and [`SERVER_REST_API_CONTRACT.md`](SERVER_REST_API_CONTRACT.md).

## 2. Architecture Scope

### WordPress Plugin Owns

- Discovering Directorist directory types and eligible source content.
- Presenting Listings, Listing Reviews, and enabled WordPress post-type controls.
- Applying administrator-defined indexing filters.
- Cleaning and sending public content and source context to the backend.
- Synchronizing the complete allowed data-source key list.
- Retrying or surfacing failed indexing operations.
- Rendering the public widget and proxying browser chat calls.

### Ask Sunny Server Owns

- Authenticating installation, admin, and future mobile requests.
- Validating and normalizing received content.
- Producing deterministic BM25 `search_document` and embedding text.
- Hashing content and skipping unchanged embeddings.
- Persisting source-specific indexed records and embeddings.
- Enforcing the stored source allowlist on every retrieval.
- Running structured filters, BM25 retrieval, vector retrieval, and rank fusion.
- Orchestrating chat through LangGraph and the provider-neutral AI adapter.
- Returning complete grounded answers, citations, and recommendation cards.
- Persisting conversations, evidence, tool calls, usage, and diagnostics.

### PostgreSQL / ParadeDB Owns

- Durable indexed content and conversation state.
- `pg_search` BM25 indexes over deterministic public search text.
- pgvector indexes over source-specific embedding tables.
- Transactional constraints, tombstones, migration state, and query execution.

### Optional Redis Owns

- Bounded query-embedding and exact-result caching when enabled.
- No source-of-truth state. Redis failure must not corrupt or prevent durable indexing.

## 3. Runtime Stack And Configuration

- Runtime: Bun.
- Language: JavaScript.
- API: Hono.
- Orchestration: LangGraph.js.
- Database: PostgreSQL with ParadeDB `pg_search`, pgvector, and `pgcrypto`.
- Chat generation: provider-neutral adapter selected by `AI_PROVIDER=openai|groq`.
- Embeddings: independently selected with `EMBEDDING_PROVIDER` and embedding environment settings.
- Deployment: native services or optional Docker Compose.
- Response transport: one complete JSON response; no partial token streaming.

Relevant environment contract:

```dotenv
DATABASE_URL=postgres://ask_sunny:strong-password@127.0.0.1:5432/ask_sunny
PG_POOL_MAX=10

AI_PROVIDER=openai
OPENAI_API_KEY=replace-with-openai-api-key
OPENAI_CHAT_MODEL=replace-with-supported-openai-model
GROQ_API_KEY=replace-with-groq-api-key
GROQ_CHAT_MODEL=replace-with-supported-groq-model

EMBEDDING_PROVIDER=openai
OPENAI_EMBEDDINGS_URL=https://api.openai.com/v1/embeddings
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMENSIONS=1536

HYBRID_SEARCH_ENABLED=false
HYBRID_VECTOR_WEIGHT=0.65
HYBRID_BM25_WEIGHT=0.35
HYBRID_RRF_K=60
HYBRID_CANDIDATE_MULTIPLIER=3
HYBRID_MAX_CANDIDATE_LIMIT=100
```

The hybrid flag begins `false` for installation or upgrade. It changes to `true` only after the `pg_search` package is proven compatible with the running PostgreSQL major version, execution OS, and architecture and all verification checks pass. The exact gate is defined in [`HYBRID_SEARCH_PLAN.md`](HYBRID_SEARCH_PLAN.md).

`AI_PROVIDER` affects generation only. Provider identity and provider-specific conversation identifiers are not stored in application tables. Changing generation provider does not change embedding dimensions or reindex content.

## 4. High-Level Architecture

```mermaid
flowchart LR
  Browser[Visitor or admin browser] --> WPRest[WordPress REST API]
  WPRest --> Plugin[Ask Sunny WordPress plugin]
  Plugin --> API[Ask Sunny Bun/Hono API]
  Mobile[Future mobile client] --> API
  API --> Graph[LangGraph chat orchestration]
  Graph --> Retrieval[Retrieval service]
  Retrieval --> PG[(PostgreSQL + pg_search + pgvector)]
  Retrieval --> Redis[(Optional Redis)]
  Graph --> Adapter[Provider-neutral AI adapter]
  Adapter --> ActiveProvider[OpenAI or Groq]
  API --> Durable[Conversation and usage persistence]
  Durable --> PG
```

Browser code never receives the installation API key, provisioning secret, database credentials, AI keys, or embedding key. Browser requests terminate at WordPress; only trusted server-side WordPress code calls the backend.

## 5. Source And Storage Model

Ask Sunny uses a concrete source key plus a broad source kind:

- `directorist:<directory-slug>` for a mandatory listing source.
- `directorist:<directory-slug>:reviews` for a classified review source controlled by the one global Listing Reviews setting.
- `wordpress:<post-type>` for an optional enabled WordPress post type.

The source-kind repositories remain separate:

| Source kind | Content table | Embedding table |
|---|---|---|
| `directorist_listing` | `directorist_listings` | `directorist_listing_embeddings` |
| `directorist_review` | `directorist_reviews` | `directorist_review_embeddings` |
| `wordpress_post` | `wordpress_content` | `wordpress_content_embeddings` |

`data_sources` stores source labels and retrieval context. `installation_config.allowed_data_source_keys` stores the authoritative retrieval allowlist. Disabling an optional source removes its key from that list but does not delete indexed rows. An explicit delete operation tombstones content.

Every retrieval begins by loading the stored allowlist. Model-selected or request-derived keys are intersected with it; neither a chat caller nor a model can expand it. A missing or empty allowlist fails closed with no candidates.

## 6. Content Normalization And Embedding

Each source repository converts its public, content-bearing fields into two deterministic representations:

- `search_document`: concise public text for ParadeDB BM25.
- `embedding_text`: structured semantic text for the configured embedding model.

For listings, include the title, summary, tagline, body, directory label, categories, locations, tags, amenities, public contact/location fields, price, and every approved `listing_metadata` value. For reviews, include the parent listing context, rating, categories, locations, and approved review text. For WordPress content, include title, summary, body, public taxonomies, and approved post metadata.

Normalization rules:

- Sanitize HTML and exclude scripts, shortcodes that do not render public text, and private/admin fields.
- Preserve stable field keys, labels, types, providers when relevant, and typed values.
- Sort map-like metadata by stable key before serialization.
- Format dates and times consistently and retain the installation timezone context.
- Store public file fields as URLs, not file bytes.
- Exclude database IDs used only internally, synchronization timestamps, hashes, and raw debug payloads from embeddings.
- Compute `content_hash` over the canonical normalized payload and normalization version.
- Re-embed only when the semantic representation, embedding model, dimensions, or normalization version changes.

The same model and dimensions must be used for indexed documents and query embeddings. A dimension change requires new vector storage, a migration, and a complete reindex plan.

## 7. Indexing Flow

```mermaid
sequenceDiagram
  participant WP as WordPress Plugin
  participant API as Ask Sunny API
  participant Repo as Source Repository
  participant Embed as Embedding Service
  participant DB as PostgreSQL / ParadeDB

  WP->>API: POST /content/upsert or /content/bulk-upsert
  API->>API: Authenticate and validate source identity
  API->>Repo: Normalize source-specific public fields
  Repo->>Repo: Build search_document and embedding_text
  Repo->>Repo: Compute canonical content_hash
  Repo->>DB: Read current hash and tombstone state
  alt Semantic content unchanged
    Repo->>DB: Refresh source metadata/state without re-embedding
  else New or changed content
    Repo->>Embed: Generate embedding
    Embed-->>Repo: Vector with configured dimensions
    Repo->>DB: Transactional content and embedding upsert
  end
  Repo->>DB: Invalidate/version affected cache identity
  API-->>WP: Indexed, unchanged, or per-item error result
```

Indexing requirements:

- Validate that the source key resolves to the matching repository before writing.
- A review must resolve its classified review source and parent listing.
- Reject private fields, oversized payloads, invalid URLs, and dimension mismatches.
- Treat repeated deliveries idempotently through source identity and content hash.
- Tombstone deleted, unpublished, or newly ineligible content; never retrieve tombstoned rows.
- Return non-2xx or a per-item error for failures so WordPress can retry or report them.
- Rebuild or analyze BM25/vector indexes through controlled operations, not inside a public chat request.

## 8. Semantic And Hybrid Retrieval Flow

```mermaid
flowchart TD
  Message[User message] --> Load[Load conversation and persisted source allowlist]
  Load --> Intent[Classify intent and extract constraints]
  Intent --> Select[Select relevant source keys]
  Select --> Intersect[Intersect with persisted allowed keys]
  Intersect --> Empty{Any allowed keys?}
  Empty -- no --> NoEvidence[Return no-evidence result]
  Empty -- yes --> Filters[Build validated source-specific filters]
  Filters --> QueryText[Normalize semantic and exact keyword query]
  QueryText --> Embed[Generate or load query embedding]
  Embed --> Vector[Scoped pgvector candidates]
  QueryText --> BM25[Scoped ParadeDB BM25 candidates]
  Vector --> Fusion[Weighted RRF]
  BM25 --> Fusion
  Fusion --> Policy[Final relevance and policy ranking]
  Policy --> Evidence[Load compact evidence and direct URLs]
```

Structured predicates for allowed source keys, status, date, location, category, directory type, and approved metadata must apply consistently to both BM25 and vector branches. Vector similarity thresholds apply before fusion. Each branch is bounded by configured candidate limits, uses stable tie-breaking, and returns diagnostics without leaking raw private content.

RRF is used because BM25 scores and cosine similarity have different scales:

```text
fused_score = vector_weight / (rrf_k + vector_rank)
            + bm25_weight / (rrf_k + bm25_rank)
```

After fusion, application ranking may consider exact structured matches, date and distance fit, configured metadata, source freshness, review evidence, user preferences, and eligible featured/promotion signals. Promotion never makes an irrelevant item relevant.

## 9. Chat Orchestration Flow

LangGraph receives a durable `conversation_id`, the new message, channel context, and authenticated installation scope. Recommended nodes are:

- `load_context`
- `classify_intent`
- `extract_constraints`
- `decide_tools`
- `select_data_sources`
- `retrieve_content`
- `rank_and_filter`
- `generate_answer`
- `persist_turn`

```mermaid
sequenceDiagram
  participant WP as WordPress
  participant API as POST /chat
  participant Graph as LangGraph
  participant Search as Retrieval Service
  participant DB as PostgreSQL / ParadeDB
  participant AI as Selected AI Adapter

  WP->>API: Complete chat-turn request
  API->>API: Authenticate, validate, rate-limit
  API->>Graph: Message and conversation scope
  Graph->>DB: Load conversation, profile, favorites, allowlist
  Graph->>Search: Search selected allowed sources
  Search->>DB: Structured + BM25 + vector retrieval
  DB-->>Search: Ranked candidates and evidence
  Search-->>Graph: Compact grounded tool result
  Graph->>AI: Provider-neutral history, tools, and evidence
  AI-->>Graph: Normalized tool calls or final structured answer
  Graph->>DB: Persist messages, evidence, tools, usage, status
  Graph-->>API: Complete answer, citations, recommendations
  API-->>WP: One JSON response
```

The model receives compact server-owned tool results, never database credentials or arbitrary SQL access. The server validates tool arguments, intersects source keys again, caps tool iterations, validates final output, and persists provider-neutral conversation history. If evidence is insufficient, the answer should ask a useful clarification or state that the site content does not support a confident answer.

## 10. API Boundaries

Primary semantic-search routes are:

- `PUT /retrieval/allowed-data-sources`: atomically replace the persisted source allowlist with optimistic version checking.
- `POST /content/upsert`: normalize and index one source record.
- `POST /content/bulk-upsert`: index a bounded mixed-source batch with per-item results.
- `POST /content/delete`: tombstone one source record.
- `POST /content/delete-by-data-source`: explicit administrative tombstone operation, not a disable-source action.
- `POST /chat`: execute one complete grounded chat turn.
- `GET /admin/diagnostics`: report search capability and indexing state.

`POST /chat` never accepts an AI provider, model, or caller-supplied source allowlist. It accepts conversational context only; server environment and persisted installation configuration control generation and retrieval.

## 11. Caching

Redis is optional and must be treated as an optimization.

- Query-embedding cache identity includes normalized query, embedding provider/model, dimensions, and normalization version.
- Exact retrieval-result cache identity includes normalized query, filters, allowed-source version, ranking version, candidate limits, and effective search mode.
- Do not reuse a final hybrid ranking across different exact keyword queries merely because their embeddings are similar.
- Source upsert, deletion, restore, allowlist change, embedding-model change, or ranking-version change must invalidate or advance the affected cache identity.
- Redis timeouts are short and bounded. On failure, log the event and continue through PostgreSQL without cache.

## 12. Hybrid Readiness And Package Safety

The server must never infer `pg_search` compatibility from the presence of a package alone. Before hybrid enablement, verify:

1. The running PostgreSQL major from `SHOW server_version_num`.
2. The execution OS ID/release or codename.
3. The execution CPU architecture.
4. The installed package/image target matches all three.
5. `pg_search` is preloaded when required and PostgreSQL has restarted.
6. `vector` and `pg_search` exist with recorded versions.
7. All expected migrations and source-kind BM25/vector indexes exist.
8. Direct BM25 queries and application hybrid checks pass.

For native PostgreSQL, inspect the host. For Docker, inspect inside the pinned database container and record its image digest. A mismatch or unverifiable artifact means effective hybrid mode is `false`; the API may serve vector-only search but must report the degraded reason. See [`HYBRID_SEARCH_PLAN.md`](HYBRID_SEARCH_PLAN.md) and [`SETUP_AND_OPERATIONS.md`](../shared/SETUP_AND_OPERATIONS.md).

## 13. Error Handling

### Validation Or Authentication Failure

- Invalid content/chat payload: `400` with a stable error code.
- Missing/invalid key: `401`.
- Valid key without required scope: `403`.
- Stale allowlist version: `409`.
- Rate limit exceeded: `429`.

### Indexing Failure

- Do not commit a content row with a missing or invalid embedding when the operation requires a new embedding.
- Return a retryable or permanent error classification.
- Preserve the previous usable indexed version when an update fails before transaction commit.

### Retrieval Failure

- A `pg_search` compatibility/readiness failure prevents hybrid mode; it is not silently ignored.
- A runtime BM25 failure uses the documented vector-only degraded path and labels diagnostics accurately.
- A vector/embedding failure must not be disguised as a successful semantic result.
- An empty allowlist or no relevant evidence yields no candidates, not cross-source fallback.

### Generation Failure

- Map provider errors through the active adapter to stable application errors.
- Persist a failed turn status without storing secrets or raw sensitive payloads.
- Return a safe retry message or evidence-only fallback defined by the product contract.
- Do not switch AI providers automatically unless a separate, explicit failover policy is designed and approved.

## 14. Security And Privacy

- Store only hashed API keys; return plaintext only at provisioning/rotation time.
- Keep provisioning, AI, embedding, database, and admin secrets in environment/secret storage.
- Parameterize SQL and validate metadata filter keys against an allowlist.
- Sanitize source HTML and URLs; never fetch arbitrary user-supplied URLs during chat.
- Enforce body, batch, query, candidate, tool-iteration, and timeout limits.
- Treat retrieved WordPress content as untrusted data and defend model prompts against embedded instructions.
- Never log secrets, full embeddings, raw private metadata, or unnecessary message content.
- Enforce retention and deletion rules for conversations, analytics, profiles, and tombstoned content.

## 15. Observability And Diagnostics

Capture per request or job:

- Correlation ID, route, status, duration, and error code.
- Effective search mode and degradation reason.
- Allowed and selected source-key counts without private content.
- BM25, vector, and fused candidate counts and latencies by source kind.
- Cache hit/miss/error state.
- Indexing outcome, skipped hash count, and embedding latency.
- Active provider adapter, model, latency, normalized usage, and error category.

Diagnostics must report requested/effective hybrid mode, PostgreSQL version, `pg_search` and pgvector versions, preload state, required indexes, latest migration, package compatibility verification status, and last direct smoke-test result. Provider information belongs in runtime diagnostics and usage telemetry, not in core domain tables.

## 16. Verification Checklist

- Content normalization is deterministic for all three source kinds.
- Repeated unchanged upserts do not regenerate embeddings.
- Tombstoned and disallowed sources cannot appear in retrieval.
- Structured filters constrain BM25 and vector paths identically.
- RRF is deterministic and respects vector thresholds.
- Review evidence remains linked to its parent listing.
- Citations contain valid direct URLs and claims trace to retrieved evidence.
- Multi-turn context and clarification behavior work without provider-hosted conversation state.
- Switching `AI_PROVIDER` needs no database migration and does not change retrieval.
- Native and Docker deployments both pass the `pg_search` package compatibility gate.
- A mismatch or missing extension keeps hybrid disabled and vector-only diagnostics honest.
- Backup, migration, restore, reindex, and cache invalidation procedures are rehearsed.

