# Server App Architecture

## Purpose

The Ask Sunny server receives trusted server-side requests from WordPress, and later from a mobile app, then performs conversational RAG over the site's indexed content.

The server is responsible for:

- Authenticating WordPress and mobile API calls.
- Indexing Directorist listings, separate approved listing reviews, and WordPress post-type content selected and sent by the WordPress plugin.
- Generating and storing embeddings.
- Running structured and semantic retrieval.
- Orchestrating conversational flows with LangGraph.
- Calling OpenAI Responses API for model reasoning, tool use, and complete-response generation.
- Persisting conversations, messages, citations, tool calls, user profiles, favorites, and usage events.
- Returning grounded answers with direct links.

## Runtime Stack

- Runtime: Bun.
- Language: JavaScript, following the backend service's Bun/Hono runtime pattern.
- HTTP framework: Hono.
- Agent framework: LangGraph.js.
- Model API: OpenAI Responses API.
- Embeddings: OpenAI embeddings, model configured by environment.
- Database: PostgreSQL with pgvector.
- Cache: Redis optional.
- Deployment shape: stateless API process behind HTTPS reverse proxy, with PostgreSQL and optional Redis as shared state.

## Environment Contract

```dotenv
NODE_ENV=production
HOST=127.0.0.1
PORT=3100
LOG_LEVEL=info
REQUEST_BODY_LIMIT=2mb

ASK_SUNNY_INSTALLATION_PROVISIONING_KEY=replace-with-long-random-secret
ASK_SUNNY_ADMIN_EMAIL=admin@example.com
ASK_SUNNY_ADMIN_PASSWORD=replace-with-strong-password
ASK_SUNNY_ADMIN_SESSION_TTL_SECONDS=86400

DATABASE_URL=postgres://ask_sunny:strong-password@127.0.0.1:5432/ask_sunny
PG_POOL_MAX=10

OPENAI_API_KEY=replace-with-openai-api-key
OPENAI_RESPONSES_URL=https://api.openai.com/v1/responses
OPENAI_EMBEDDINGS_URL=https://api.openai.com/v1/embeddings
ASK_SUNNY_CHAT_MODEL=gpt-5.1
ASK_SUNNY_EMBEDDING_MODEL=text-embedding-3-small
ASK_SUNNY_EMBEDDING_DIMENSIONS=1536
OPENAI_REQUEST_TIMEOUT_MS=45000

REDIS_ENABLED=false
REDIS_URL=redis://127.0.0.1:6379
REDIS_KEY_PREFIX=ask_sunny

MAX_CHAT_INPUT_CHARS=4000
MAX_RETRIEVAL_RESULTS=12
MAX_TOOL_ITERATIONS=6
DEFAULT_TIMEZONE=UTC
```

Because chat is returned as one complete response, the WordPress proxy timeout must be greater than `OPENAI_REQUEST_TIMEOUT_MS`; a 60-second WordPress timeout provides application overhead around the 45-second model timeout.

Model names are deployment configuration, not hardcoded constants. Before production launch, verify the current OpenAI recommended model and Responses API behavior from official OpenAI docs.

## High-Level Server Flow

```mermaid
flowchart TD
  Request[Incoming HTTP request] --> BodyLimit[Body limit and JSON parsing]
  BodyLimit --> Auth{Public route?}
  Auth -- yes --> PublicRoute[Health or provisioning]
  Auth -- no --> ApiKey[Bearer API key validation]
  ApiKey --> Route[Hono route handler]
  Route --> Service[Application service]
  Service --> DB[(PostgreSQL)]
  Service --> Cache[(Optional Redis)]
  Service --> OpenAI[OpenAI API]
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
- `retrieve_content`: dispatch searches to the listing, review, and WordPress-content repositories represented by selected keys, apply kind-specific structured constraints, then merge their scored results. Selected keys must be a subset of the backend's stored `allowed_data_source_keys`. Event questions select an Event Directory listing source and apply its metadata fields; review or rating questions may also select that directory's companion review source.
- `rank_and_filter`: merge semantic, structured, featured, configured promotion-metadata, and personalization signals.
- `generate_answer`: call OpenAI Responses API with tool outputs and citation candidates.
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

## OpenAI Responses API Usage

Use Responses API for:

- Agentic tool calls within a chat turn.
- Structured final answer generation.
- Complete structured response generation for the widget.
- Multi-turn continuity through server-side conversation context.

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

## Indexing Architecture

WordPress owns the source registry, enable/disable controls, and indexing filters. It sends only eligible listings, reviews, and posts. The backend dispatches by `source_kind` into separate persistence boundaries: Directorist listings, Directorist reviews, and WordPress content. Each kind has its own table, normalizer, repository, and embedding table. Listing payloads separate Directorist core preset values into canonical columns and put every non-core value into one flat `listing_metadata` map. Review records resolve `parent_data_source_key` and `parent_source_id` to a foreign key in `directorist_listings` so ranking can aggregate review evidence.

WordPress owns the source settings UI and computes the complete allowlist. After provisioning and whenever source enablement changes, WordPress atomically synchronizes `allowed_data_source_keys` to backend installation configuration. The backend stores and enforces that list across SQL filtering, vector retrieval, detail lookup, and model tools. Disabling a source updates the allowlist without deleting indexed content. Deletion occurs only when WordPress sends an explicit per-item deletion or bulk delete-by-data-source request initiated by an administrator or maintenance workflow.

If the allowlist is empty or unavailable, retrieval fails closed and returns no candidates. Chat input cannot override the stored list.

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
  Embed --> Upsert[Upsert matching content and embedding tables]
  Upsert --> Analyze[Record indexing event]
  Analyze --> Done[Return status]
  Validate --> Delete{Delete/unpublish?}
  Delete -- yes --> Tombstone[Mark inactive and remove from retrieval]
```

## Security Principles

- OpenAI keys live only on the backend server.
- Installation provisioning secret is used only by trusted WordPress server-side code.
- Generated installation API keys are hashed at rest on the backend and stored server-side in WordPress options.
- Browser requests use WordPress nonces or anonymous session tokens, never backend bearer keys.
- Mobile app access should use a separate public-client auth path, not the WordPress installation API key.
- Store user preference and conversation data with deletion/export paths planned from the beginning.

## Error Handling

- Invalid JSON: `400 validation_error`.
- Missing or invalid API key: `401 authentication_error`.
- Authenticated but wrong key scope: `403 forbidden`.
- Missing content or conversation: `404 not_found`.
- OpenAI failure in chat: return a friendly fallback answer and persist an error usage event.
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
  participant DB as PostgreSQL
  participant OAI as OpenAI Responses

  Browser->>WP: POST /chat
  WP->>API: POST /chat with installation key
  API->>DB: Load conversation/profile
  API->>DB: Load persisted allowed source keys
  API->>Graph: Start graph run
  Graph->>DB: Retrieve candidates
  Graph->>OAI: Create response
  OAI-->>Graph: Complete model response and tool results
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
