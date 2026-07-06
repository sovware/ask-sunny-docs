# Server App Architecture

## Purpose

The AI Sunny server is the backend for the Ask Sunny concierge. It receives trusted server-side requests from WordPress, and later from a mobile app, then performs conversational RAG over Palm Beach Mama Club content.

The server is responsible for:

- Authenticating WordPress and mobile API calls.
- Normalizing and indexing Directorist business, event, review, newsletter, and promotion content.
- Generating and storing embeddings.
- Running structured and semantic retrieval.
- Orchestrating conversational flows with LangGraph.
- Calling OpenAI Responses API for model reasoning, tool use, and streaming.
- Persisting conversations, messages, citations, tool calls, user profiles, favorites, and usage events.
- Returning grounded answers with direct links.

## Runtime Stack

- Runtime: Bun.
- Language: JavaScript, following the existing AI Search server convention.
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

AI_SUNNY_INSTALLATION_PROVISIONING_KEY=replace-with-long-random-secret
AI_SUNNY_ADMIN_EMAIL=admin@example.com
AI_SUNNY_ADMIN_PASSWORD=replace-with-strong-password
AI_SUNNY_ADMIN_SESSION_TTL_SECONDS=86400

DATABASE_URL=postgres://ai_sunny:strong-password@127.0.0.1:5432/ai_sunny
PG_POOL_MAX=10

OPENAI_API_KEY=replace-with-openai-api-key
OPENAI_RESPONSES_URL=https://api.openai.com/v1/responses
OPENAI_EMBEDDINGS_URL=https://api.openai.com/v1/embeddings
AI_SUNNY_CHAT_MODEL=gpt-5.1
AI_SUNNY_EMBEDDING_MODEL=text-embedding-3-small
AI_SUNNY_EMBEDDING_DIMENSIONS=1536
OPENAI_REQUEST_TIMEOUT_MS=45000

REDIS_ENABLED=false
REDIS_URL=redis://127.0.0.1:6379
REDIS_KEY_PREFIX=ai_sunny

MAX_CHAT_INPUT_CHARS=4000
MAX_RETRIEVAL_RESULTS=12
MAX_TOOL_ITERATIONS=6
DEFAULT_TIMEZONE=America/New_York
```

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
  Service --> Response[JSON or SSE response]
```

## LangGraph Chat Architecture

LangGraph owns the orchestration of a chat turn. Each graph run receives a durable `conversation_id`, the new user message, optional visitor/user context, and request metadata. Graph state contains the active user request, conversation summary, retrieved candidates, tool results, citations, response draft, and moderation/status metadata.

Recommended graph nodes:

- `load_context`: load conversation, recent messages, user profile, favorites, and site settings.
- `classify_intent`: classify whether the user needs recommendations, event search, business search, clarification, or general help.
- `extract_constraints`: identify location, date, ages, budget, indoor/outdoor preference, category, amenities, and accessibility needs.
- `decide_tools`: choose retrieval tools and whether a clarifying question is required.
- `retrieve_businesses`: search business directory content.
- `retrieve_events`: search event content by date and family suitability.
- `retrieve_editorial`: search Weekend Picks, FAQs, blog posts, promotions, and sponsored content.
- `rank_and_filter`: merge semantic, structured, featured, sponsored, and personalization signals.
- `generate_answer`: call OpenAI Responses API with tool outputs and citation candidates.
- `persist_turn`: write messages, tool calls, citations, usage, and graph status.

```mermaid
flowchart TD
  Start([Chat request]) --> Load[Load conversation and profile]
  Load --> Classify[Classify intent]
  Classify --> Extract[Extract constraints]
  Extract --> Clarify{Need clarification?}
  Clarify -- yes --> GenerateClarify[Generate follow-up question]
  Clarify -- no --> ToolPlan[Plan retrieval tools]
  ToolPlan --> Business[Business retrieval]
  ToolPlan --> Events[Event retrieval]
  ToolPlan --> Editorial[Editorial retrieval]
  Business --> Rank[Rank and filter]
  Events --> Rank
  Editorial --> Rank
  Rank --> Generate[Responses API answer generation]
  GenerateClarify --> Persist[Persist turn]
  Generate --> Persist
  Persist --> End([JSON or SSE output])
```

LangGraph persistence should use a PostgreSQL-backed checkpointer when implementation begins. Durable application records still live in the schema described in `SERVER_DATABASE_SCHEMA.md`; checkpoints are for graph recovery and short-term orchestration, not the only audit log.

## OpenAI Responses API Usage

Use Responses API for:

- Agentic tool calls within a chat turn.
- Structured final answer generation.
- Streaming text deltas for the widget.
- Multi-turn continuity through server-side conversation context.

The server should provide custom tools to the model through the application layer, not expose database credentials or raw SQL. Tool implementations run in server code and return compact result objects.

Recommended tools:

- `search_businesses`
- `search_events`
- `search_weekend_picks`
- `get_listing_detail`
- `get_user_preferences`
- `save_user_preference`

The final answer should include:

- `answer`: user-facing text.
- `citations`: direct links and source labels.
- `recommendations`: structured cards for the WordPress widget.
- `follow_up_questions`: optional next-step prompts.
- `conversation_id`: durable ID for continuity.

## Indexing Architecture

WordPress sends content payloads whenever Directorist content changes or an admin triggers reindexing. The backend normalizes each payload into a canonical content record, computes a content hash, skips unchanged records, generates embeddings when needed, and stores structured fields for filtering.

```mermaid
flowchart TD
  WP[WordPress indexing request] --> Auth[Validate installation API key]
  Auth --> Validate[Validate payload]
  Validate --> Normalize[Normalize source fields]
  Normalize --> Hash[Compute content hash]
  Hash --> Changed{Changed?}
  Changed -- no --> Skip[Return unchanged]
  Changed -- yes --> Embed[Generate embedding]
  Embed --> Upsert[Upsert content and embedding]
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
- Streaming failure: emit an SSE `error` event and close the stream.

## Server Flow Charts

### Streaming Chat Flow

```mermaid
sequenceDiagram
  participant Browser
  participant WP as WordPress REST
  participant API as AI Sunny Server
  participant Graph as LangGraph
  participant DB as PostgreSQL
  participant OAI as OpenAI Responses

  Browser->>WP: POST /chat/stream
  WP->>API: POST /chat/stream with installation key
  API->>DB: Load conversation/profile
  API->>Graph: Start graph run
  Graph->>DB: Retrieve candidates
  Graph->>OAI: Create streaming response
  OAI-->>Graph: Text/tool deltas
  Graph-->>API: Stream events
  API-->>WP: SSE events
  WP-->>Browser: SSE events
  Graph->>DB: Persist messages, citations, usage
```

### Future Mobile Flow

```mermaid
flowchart TD
  Mobile[Mobile app] --> MobileAuth[Mobile auth token]
  MobileAuth --> Server[AI Sunny backend]
  Server --> Profile[User profile and favorites]
  Server --> Graph[LangGraph chat flow]
  Graph --> Content[Business/event retrieval]
  Graph --> Response[Personalized answer]
```

