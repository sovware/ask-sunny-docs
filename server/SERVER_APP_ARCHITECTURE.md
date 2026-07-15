# Server App Architecture

## Purpose

The Ask Sunny server receives trusted server-side requests from WordPress, and later from a mobile app, then performs conversational RAG over the site's indexed content.

The server is responsible for:

- Authenticating WordPress and mobile API calls.
- Indexing Directorist listings, separate approved listing reviews, and WordPress post-type content selected and sent by the WordPress plugin.
- Generating and storing embeddings.
- Running structured, ParadeDB BM25, and pgvector semantic retrieval with hybrid fusion.
- Orchestrating conversational flows with LangGraph.
- Calling the configured OpenAI or Groq Responses API for model reasoning, tool use, and complete-response generation.
- Persisting conversations, messages, citations, tool calls, user profiles, favorites, and usage events.
- Returning grounded answers with direct links.

## Runtime Stack

- Runtime: Bun.
- Language: JavaScript, following the backend service's Bun/Hono runtime pattern.
- HTTP framework: Hono.
- Agent framework: LangGraph.js.
- Model API: provider-neutral generation interface with adapters registered by name and selected at runtime from `AI_PROVIDER`.
- Embeddings: independently configured embedding provider; OpenAI is the launch default.
- Database: ParadeDB's PostgreSQL distribution with `pg_search` and pgvector.
- Search: hybrid BM25 keyword matching plus dense vector similarity.
- Cache: Redis optional.
- Deployment shape: either native Bun + ParadeDB PostgreSQL services or an optional Docker Compose stack, in both cases behind an HTTPS reverse proxy.

## Environment Contract

```dotenv
NODE_ENV=production
HOST=0.0.0.0
PORT=3100
LOG_LEVEL=info
REQUEST_BODY_LIMIT=2mb

ASK_SUNNY_INSTALLATION_PROVISIONING_KEY=replace-with-long-random-secret
ASK_SUNNY_ADMIN_EMAIL=admin@example.com
ASK_SUNNY_ADMIN_PASSWORD=replace-with-strong-password
ASK_SUNNY_ADMIN_SESSION_TTL_SECONDS=86400

DATABASE_URL=postgres://ask_sunny:strong-password@127.0.0.1:5432/ask_sunny
PG_POOL_MAX=10

AI_PROVIDER=openai
AI_REQUEST_TIMEOUT_MS=45000

OPENAI_API_KEY=replace-with-openai-api-key
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_CHAT_MODEL=replace-with-supported-openai-model

GROQ_API_KEY=replace-with-groq-api-key
GROQ_BASE_URL=https://api.groq.com/openai/v1
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
MAX_ALLOWED_SEARCH_IDS=1000
MAX_ALLOWED_DATA_SOURCE_KEYS=1000
MAX_CONTENT_ITEM_BYTES=524288
MAX_METADATA_FIELDS=100
MAX_METADATA_KEY_CHARS=64
MAX_METADATA_LABEL_CHARS=120
MAX_METADATA_STRING_CHARS=2000
MAX_METADATA_ARRAY_ITEMS=50
MAX_METADATA_NESTING_DEPTH=4

REDIS_ENABLED=false
REDIS_URL=redis://127.0.0.1:6379
REDIS_KEY_PREFIX=ask_sunny

MAX_CHAT_INPUT_CHARS=4000
MAX_RETRIEVAL_RESULTS=12
MAX_TOOL_ITERATIONS=6
DEFAULT_TIMEZONE=UTC
```

`AI_PROVIDER` is the only switch for chat generation. A provider registry resolves that value to an adapter implementing the provider-neutral generation interface; orchestration, persistence, routes, and domain services must not branch on provider names. The selected adapter's API key, base URL, and model must be valid at startup; credentials for an inactive provider may be omitted. Embeddings are configured independently because generation and embedding providers do not have identical capabilities. Changing `AI_PROVIDER` does not change stored vector dimensions or trigger re-embedding, and provider identity is not stored in application database tables.

Because chat is returned as one complete response, the WordPress proxy timeout must be greater than `AI_REQUEST_TIMEOUT_MS`; a 60-second WordPress timeout provides application overhead around the 45-second provider timeout.

Model names are deployment configuration, not hardcoded constants. Verify the selected provider's current model, Responses API, structured-output, and tool-use support before production launch. The provider adapter must not send parameters unsupported by the active provider.

The example connection URLs target native services. Docker Compose overrides their hosts with Compose service names such as `paradedb` and `redis`; application code and all other configuration remain identical.

`HYBRID_SEARCH_ENABLED=false` is the safe installation and upgrade value. Hybrid is the intended normal production mode, but an operator changes the value to `true` only after the running PostgreSQL major version, execution OS, CPU architecture, and installed `pg_search` package/image are an exact supported match and every extension, migration, index, direct-query, and application verification passes. See [`HYBRID_SEARCH_PLAN.md`](HYBRID_SEARCH_PLAN.md).

## High-Level Server Flow

```mermaid
flowchart TD
  Request[Incoming HTTP request] --> BodyLimit[Body limit and JSON parsing]
  BodyLimit --> Auth{Public route?}
  Auth -- yes --> PublicRoute[Health or provisioning]
  Auth -- no --> ApiKey[Bearer API key validation]
  ApiKey --> Route[Hono route handler]
  Route --> Service[Application service]
  Service --> DB[(ParadeDB PostgreSQL)]
  Service --> Cache[(Optional Redis)]
  Service --> Provider[Selected AI provider]
  Service --> Response[Complete JSON response]
```

## LangGraph Chat Architecture

LangGraph owns the orchestration of a chat turn. Each graph run receives a durable `conversation_id`, the new user message, optional visitor/user context, and request metadata. Graph state contains the active user request, conversation summary, retrieved candidates, tool results, citations, response draft, and moderation/status metadata.

Recommended graph nodes:

- `load_context`: load conversation, recent messages, user profile, favorites, and persisted `allowed_data_source_keys` from backend installation configuration.
- `classify_intent`: classify whether the user needs recommendations, source search, clarification, or general help.
- `extract_constraints`: identify relevant dates, locations, categories, budget, amenities, accessibility needs, core preset fields, and generic listing metadata.
- `decide_tools`: choose retrieval tools and whether a clarifying question is required.
- `select_data_sources`: choose relevant concrete sources from labels, descriptions, and context metadata.
- `retrieve_content`: dispatch BM25 and vector searches to the listing, review, and WordPress-content repositories represented by selected keys, apply kind-specific structured constraints, then fuse their scored results. Selected keys must be a subset of the backend's stored `allowed_data_source_keys`. Event questions select an Event Directory listing source and apply its metadata fields; review or rating questions may also select classified review keys when the global Listing Reviews family is enabled.
- `rank_and_filter`: merge semantic, structured, featured, configured promotion-metadata, and personalization signals.
- `generate_answer`: call the selected provider adapter with tool outputs and citation candidates.
- `persist_turn`: write messages, tool calls, citations, usage, and graph status.

```mermaid
flowchart TD
  Start([Chat request]) --> Load[Load conversation and profile]
  Load --> Classify[Classify intent]
  Classify --> Extract[Extract constraints]
  Extract --> Clarify{Need clarification?}
  Clarify -- yes --> GenerateClarify[Generate follow-up question]
  Clarify -- no --> SourcePlan[Select data sources from context metadata]
  SourcePlan --> Retrieve[Retrieve within selected data source keys]
  Retrieve --> Rank[Rank and filter]
  Rank --> Generate[Responses API answer generation]
  GenerateClarify --> Persist[Persist turn]
  Generate --> Persist
  Persist --> End([Complete JSON output])
```

LangGraph persistence should use a PostgreSQL-backed checkpointer when implementation begins. Durable application records still live in the schema described in [`SERVER_DATABASE_SCHEMA.md`](SERVER_DATABASE_SCHEMA.md); checkpoints are for graph recovery and short-term orchestration, not the only audit log.

## AI Provider Adapter

Use the configured provider's Responses API for:

- Agentic tool calls within a chat turn.
- Structured final answer generation.
- Complete structured response generation for the widget.
- Multi-turn continuity through server-side conversation context.

The launch adapter registry includes `openai` and `groq`. `AI_PROVIDER=openai` resolves the OpenAI adapter and `AI_PROVIDER=groq` resolves the Groq adapter. Adding a future provider requires registering another implementation, not editing orchestration or persistence code. Each adapter owns request construction, supported parameters, structured-output validation, tool-call normalization, usage normalization, timeout handling, and error mapping.

Do not depend on provider-hosted conversation state. The application loads and persists provider-neutral conversation history and LangGraph state, then supplies the required context on every turn. Database tables do not store the active provider or provider-specific conversation IDs. This keeps provider switching deterministic and avoids coupling to provider-specific response-storage features.

The server should provide custom tools to the model through the application layer, not expose database credentials or raw SQL. Tool implementations run in server code and return compact result objects.

Recommended tools:

- `list_data_sources`
- `search_content`
- `get_content_detail`
- `get_user_preferences`
- `save_user_preference`

The final answer should include:

- `answer`: user-facing text.
- `citations`: direct links and source labels.
- `recommendations`: structured cards for the WordPress widget.
- `follow_up_questions`: optional next-step prompts.
- `conversation_id`: durable ID for continuity.

## ParadeDB Hybrid Retrieval

ParadeDB runs either as a native PostgreSQL-compatible service or as the database service in the optional Docker stack. Enable both `vector` and ParadeDB's `pg_search` extension. The `listings` table keeps normalized listing state and its pgvector embedding together and receives a ParadeDB BM25 index over the same searchable representation. Review and WordPress-content repositories retain their source-specific vector tables and BM25 indexes.

Retrieval order:

1. Load the persisted `allowed_data_source_keys`; fail closed if it is empty or unavailable.
2. Apply source-kind, status, structured metadata, and allowed-key filters before ranking.
3. Generate BM25 candidates through `pg_search` and dense candidates through pgvector.
4. Bound both candidate sets using the configured multiplier and maximum limit.
5. Fuse results with reciprocal-rank fusion and the configured BM25/vector weights.
6. Apply application ranking signals, deduplicate, and attach citations.

Hybrid search is the normal mode only after deployment verification and then executes BM25 plus vector retrieval for every eligible query. Before installing or loading `pg_search`, deployment must prove that the artifact matches the running PostgreSQL major version, execution OS/release, and CPU architecture. Native deployments inspect the PostgreSQL host; Docker deployments inspect the database container and record its pinned image digest. Startup/readiness must also verify both extensions, migrations, BM25 indexes, and direct BM25 smoke queries. If any compatibility fact is missing or mismatched, or another verification fails, hybrid must remain ineffective and `HYBRID_SEARCH_ENABLED=false`; diagnostics report the precise degraded reason and the service may use vector retrieval without pretending BM25 is active. Native and Docker deployment paths follow the same backup, verification, and rollback safety principles in [`HYBRID_SEARCH_PLAN.md`](HYBRID_SEARCH_PLAN.md) and [`SETUP_AND_OPERATIONS.md`](../shared/SETUP_AND_OPERATIONS.md).

## Indexing Architecture

WordPress owns the source registry, enable/disable controls, and indexing filters. It sends only eligible listings, reviews, and posts. The backend dispatches by `source_kind` into separate persistence boundaries: Directorist listings, Directorist reviews, and WordPress content. Each kind has its own table, normalizer, and repository. Raw metadata, normalized metadata, deterministic embedding text, vector, content hash, indexing timestamps, and soft-delete state live atomically in `listings`. Review and WordPress content use their own content and vector tables. Review records resolve `parent_data_source_key` and `parent_source_id` to the composite listing key so ranking can aggregate review evidence.

Validation and persistence preparation are separate boundaries. Source-kind validation produces a
bounded public payload and a retrieval-only source descriptor. A read-model repository accepts only
a source-specific prepared storage record; for review and WordPress records that includes the
`content_hash` and `search_document` fields required by their tables. SV-US-006 owns deterministic
serialization, hash/embedding-text production, embedding calls, and vector decisions. SV-US-005
must not invent placeholder hashes or persist incomplete active review/WordPress rows.

WordPress owns the source settings UI and computes the complete allowlist. After provisioning and whenever source enablement changes, WordPress atomically synchronizes `allowed_data_source_keys` to backend installation configuration. The backend stores and enforces that list across structured filtering, BM25 retrieval, vector retrieval, detail lookup, and model tools. Disabling a source updates the allowlist without deleting indexed content. Deletion occurs only when WordPress sends an explicit per-item deletion or bulk delete-by-data-source request initiated by an administrator or maintenance workflow.

If the allowlist is empty or unavailable, retrieval fails closed and returns no candidates. Chat input cannot override the stored list.

One application policy accessor owns allowlist loading and intersection. Structured, BM25, vector,
detail, and model-tool boundaries accept the accessor's scoped result rather than trusting raw caller
or model keys. The installation-authenticated retrieval-configuration read route exposes the current
canonical keys, version, and update time for WordPress reconciliation and diagnostics; the broader
admin diagnostics route remains admin-authenticated.

```mermaid
flowchart TD
  WP[WordPress indexing request] --> Auth[Validate installation API key]
  Auth --> Validate[Validate content and source metadata]
  Validate --> Registry[Upsert retrieval-only source metadata]
  Registry --> Dispatch[Dispatch by source kind and storage table]
  Dispatch --> Normalize[Run source-specific normalizer]
  Normalize --> Hash[Compute content hash]
  Hash --> Changed{Changed?}
  Changed -- no --> Skip[Return unchanged]
  Changed -- yes --> Embed[Generate embedding]
  Embed --> Upsert[Atomically upsert source-specific content and vector state]
  Upsert --> Analyze[Record indexing event]
  Analyze --> Done[Return status]
  Validate --> Delete{Delete/unpublish?}
  Delete -- yes --> Tombstone[Mark inactive and remove from retrieval]
```

## Security Principles

- OpenAI, Groq, and embedding-provider keys live only in the backend environment.
- Installation provisioning secret is used only by trusted WordPress server-side code.
- Generated installation API keys are hashed at rest on the backend and stored server-side in WordPress options.
- Installation keys use unique non-secret prefixes for lookup, high-entropy secret segments, fixed server-defined route scopes, and immediate atomic revocation of older installation keys on rotation.
- Malformed, unknown, hash-mismatched, and revoked bearer keys share one `401 authentication_error`; wrong-scope credentials receive a non-descriptive `403 forbidden`.
- Browser requests use WordPress nonces or anonymous session tokens, never backend bearer keys.
- Mobile app access should use a separate public-client auth path, not the WordPress installation API key.
- Store user preference and conversation data with deletion/export paths planned from the beginning.

## Error Handling

- Invalid JSON: `400 validation_error`.
- Missing or invalid API key: `401 authentication_error`.
- Authenticated but wrong key scope: `403 forbidden`.
- Missing content or conversation: `404 not_found`.
- Selected AI-provider failure in chat: return a friendly fallback answer, emit provider context only to ephemeral logs/metrics, and persist a provider-neutral error usage event.
- Retrieval failure: continue with available sources when possible, but disclose limited results.
- Chat timeout or generation failure: return one JSON error response with a stable code and friendly message, then persist the error usage event.

## Server Flow Charts

### Complete Chat Response Flow

```mermaid
sequenceDiagram
  participant Browser
  participant WP as WordPress REST
  participant API as Ask Sunny Server
  participant Graph as LangGraph
  participant DB as ParadeDB PostgreSQL
  participant AI as Selected AI Provider

  Browser->>WP: POST /chat
  WP->>API: POST /chat with installation key
  API->>DB: Load conversation/profile
  API->>DB: Load persisted allowed source keys
  API->>Graph: Start graph run
  Graph->>DB: BM25 + vector hybrid retrieval
  Graph->>AI: Create response
  AI-->>Graph: Complete model response and tool results
  Graph->>DB: Persist messages, citations, usage
  Graph-->>API: Complete answer payload
  API-->>WP: JSON response
  WP-->>Browser: Sanitized JSON response
```

### Future Mobile Flow

```mermaid
flowchart TD
  Mobile[Mobile app] --> MobileAuth[Mobile auth token]
  MobileAuth --> Server[Ask Sunny backend]
  Server --> Profile[User profile and favorites]
  Server --> Graph[LangGraph chat flow]
  Graph --> Content[Indexed-content retrieval]
  Graph --> Response[Personalized answer]
```
