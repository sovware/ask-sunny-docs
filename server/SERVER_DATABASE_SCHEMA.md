# Server Database Schema

The Ask Sunny server uses ParadeDB's PostgreSQL distribution as the durable system of record for indexed content, conversations, usage, and admin state. WordPress remains the editorial source of truth; backend content tables are a searchable read model. ParadeDB `pg_search` provides BM25 keyword retrieval and pgvector provides dense similarity retrieval.

Ask Sunny stores listings as a single-tenant indexed read model. `data_source_id` provides the listing namespace, and each listing row stores its searchable document state and vector state atomically.

Use UUIDs for application-owned records and preserve positive WordPress integer IDs for Directorist listings. The listing composite key remains stable across reindexing and can be referenced by reviews without an extra synthetic identifier.

## Extensions

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_search;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

These extension statements may run only after the deployment compatibility gate. Native installations and optional Docker deployments must prove that the installed ParadeDB/`pg_search` artifact matches the running PostgreSQL major version, execution OS/release, and CPU architecture. For Docker, inspect the database container rather than assuming the host OS or image tag is sufficient. `pg_search` must be present in `shared_preload_libraries` when required by that version. Hybrid search is the verified production default, but installation/upgrade starts with it disabled; a missing or mismatched package keeps it ineffective and the service explicitly enters documented vector-only degraded mode.

## Core Configuration

```sql
CREATE TABLE installation_config (
  id BOOLEAN PRIMARY KEY DEFAULT true CHECK (id = true),
  installation_name TEXT NOT NULL DEFAULT 'WordPress Site',
  primary_domain TEXT NOT NULL,
  wordpress_site_url TEXT NOT NULL,
  timezone TEXT NOT NULL DEFAULT 'UTC',
  allowed_data_source_keys TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  allowed_data_sources_version BIGINT NOT NULL DEFAULT 0,
  allowed_data_sources_updated_at TIMESTAMPTZ NULL,
  settings JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX installation_allowed_data_sources_gin_idx
ON installation_config USING GIN (allowed_data_source_keys);

CREATE TABLE api_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key_prefix TEXT NOT NULL UNIQUE,
  key_hash TEXT NOT NULL UNIQUE,
  key_type TEXT NOT NULL CHECK (key_type IN ('wordpress_installation', 'admin', 'mobile_service')),
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'revoked')),
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_used_at TIMESTAMPTZ NULL,
  revoked_at TIMESTAMPTZ NULL
);
```

Allowlist replacement uses one conditional statement that matches
`allowed_data_sources_version = expected_version`, writes the complete canonical array, increments
the version by exactly one, and sets `allowed_data_sources_updated_at` from the database clock. A
zero-row update is `409 retrieval_config_conflict`; it must not retry against a newer version or
partially alter the array. The initial version is `0`, including before the first allowlist sync.

Canonical source keys are lower-case and use one of
`directorist:<slug>`, `directorist:<slug>:reviews`, or `wordpress:<post-type>`, where `<slug>` and
`<post-type>` match `[a-z0-9][a-z0-9_-]{0,63}`. The service trims and lowercases input, deduplicates
after canonicalization, and stores keys in ascending bytewise order. It validates shape and the
configured count bound but does not require a matching `data_sources` row: initial allowlist
synchronization may precede the first content delivery. An empty array is valid and fails retrieval
closed.

For WordPress installation credentials, `key_prefix` is the unique
`ask_live_<16-lowercase-hex-key-id>` portion of the presented key and `key_hash` is the hex SHA-256
digest of the complete high-entropy API key. The digest is used only after the prefix selects a
candidate row and is compared in constant time. Plaintext keys are never persisted.

The `metadata` object for a WordPress installation key contains its fixed `scopes`, a
`rotation_id`, and either `rotated_from_key_ids` on the newly issued key or `revocation_reason` plus
`replaced_by_key_id` on keys revoked by rotation. Provisioning/rotation atomically upserts the
singleton installation identity, inserts the new key, and revokes every previously active
`wordpress_installation` key. A failed transaction must leave the prior credential active.

## Data Source Metadata

The backend stores the identity and retrieval context of data sources represented by received content. It does not reproduce the WordPress settings UI or indexing-filter configuration. WordPress computes the allowed keys from its local settings and synchronizes them into `installation_config.allowed_data_source_keys`; the backend enforces that persisted list for RAG.

```sql
CREATE TABLE data_sources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  data_source_key TEXT NOT NULL UNIQUE,
  source_kind TEXT NOT NULL CHECK (source_kind IN ('directorist_listing', 'directorist_review', 'wordpress_post')),
  label TEXT NOT NULL,
  description TEXT NOT NULL DEFAULT '',
  wordpress_post_type TEXT NOT NULL,
  directory_type_id BIGINT NULL,
  directory_type_slug TEXT NULL,
  parent_data_source_id UUID NULL REFERENCES data_sources(id),
  context_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  source_updated_at TIMESTAMPTZ NULL,
  last_received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (source_kind NOT IN ('directorist_listing', 'directorist_review') OR directory_type_id IS NOT NULL),
  CHECK (source_kind <> 'directorist_review' OR parent_data_source_id IS NOT NULL)
);

CREATE INDEX data_sources_kind_idx ON data_sources (source_kind);
CREATE INDEX data_sources_context_gin_idx ON data_sources USING GIN (context_metadata);
```

`data_source_key` is a stable machine identifier such as `directorist:events`, `directorist:events:reviews`, or `wordpress:post`. Labels may change without changing the key. `parent_data_source_id` links a review classification to the listing source for the same directory type. Disabling the global Listing Reviews family or an optional WordPress source does not change these rows or its content rows; the persisted backend allowlist excludes the affected keys from RAG. Rows are tombstoned only after an explicit deletion request.

## Directorist Listings

```sql
CREATE TABLE listings (
  id BIGINT NOT NULL,
  data_source_id UUID NOT NULL REFERENCES data_sources(id),
  search_key BIGINT GENERATED ALWAYS AS IDENTITY UNIQUE,
  source_url TEXT NULL,
  title TEXT NULL,
  description TEXT NULL,
  directory_type_id BIGINT NULL,
  directory_type_slug TEXT NULL,
  is_featured BOOLEAN NOT NULL DEFAULT false,
  semantic_terms TEXT NULL,
  raw_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  normalized_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  embedding_text TEXT NULL,
  embedding vector(1536) NULL,
  embedding_model TEXT NULL,
  embedding_dimensions INTEGER NULL CHECK (embedding_dimensions IS NULL OR embedding_dimensions = 1536),
  content_hash TEXT NULL,
  indexed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NULL,
  deleted_at TIMESTAMPTZ NULL,
  PRIMARY KEY (data_source_id, id)
);

CREATE INDEX listings_data_source_active_idx
ON listings (data_source_id)
WHERE deleted_at IS NULL;

CREATE INDEX listings_content_hash_idx
ON listings (data_source_id, content_hash)
WHERE deleted_at IS NULL;

CREATE INDEX listings_data_source_featured_active_idx
ON listings (data_source_id, is_featured)
WHERE deleted_at IS NULL;

CREATE INDEX listings_normalized_metadata_gin_idx
ON listings USING GIN (normalized_metadata);

CREATE INDEX listings_embedding_hnsw_idx
ON listings
USING hnsw (embedding vector_cosine_ops)
WHERE deleted_at IS NULL AND embedding IS NOT NULL;

CREATE INDEX listings_bm25_idx
ON listings
USING bm25 (
  search_key,
  data_source_id,
  id,
  is_featured,
  title,
  description,
  directory_type_slug,
  semantic_terms,
  embedding_text,
  normalized_metadata
)
WITH (key_field = 'search_key')
WHERE deleted_at IS NULL;
```

The composite primary key preserves Directorist identity inside a concrete directory-type data source. Because Ask Sunny is single-tenant, `data_source_id` is the storage namespace. `search_key` is an independent stable key for ParadeDB. Active upserts require URL, title, and directory classification in application validation; those columns remain nullable so a delete received before its corresponding upsert can still create a minimal tombstone.

`raw_metadata` stores the complete sanitized public metadata received from WordPress. `normalized_metadata` stores the canonical searchable representation produced by the Directorist normalizer, including categories, locations, tags, amenities, price, public contact and map data, images, source context, core preset values, and custom or extension-provided fields. Frequently ranked scalar values such as `is_featured` remain dedicated columns; other metadata is promoted to a column only through a measured, migration-managed indexing requirement.

Within `normalized_metadata`, custom and extension-provided values use a flat `listing_metadata` map keyed by the stable Directorist field key:

```json
{
  "experience_level": {
    "label": "Experience Level",
    "field_type": "select",
    "value": "beginner"
  },
  "event_start": {
    "label": "Event Start",
    "provider": "directorist-events",
    "field_type": "datetime",
    "value": "2026-07-11T10:00:00Z"
  }
}
```

Do not create separate custom-field and third-party-preset namespaces. `provider` is optional descriptive metadata and does not change storage location. Use stable field keys, preserve typed JSON values, and store public file-upload fields as sanitized URLs rather than file bytes. Admin-only/private fields are excluded from `raw_metadata`, `normalized_metadata`, and embeddings.

`embedding_text` is the deterministic public text used by both embedding generation and the listing BM25 surface. `content_hash` covers the data-source identity, listing ID, directory classification, embedding text, normalized metadata, semantic terms, and normalization version. Operational values and `raw_metadata` are excluded from the hash.

`embedding_model` and `embedding_dimensions` identify the inline listing vector. SV-US-006 adds these
nullable columns through a forward migration. Pre-existing vectors have unknown identity and are
regenerated on the next delivery; the migration does not guess or backfill a model name. Every new
listing vector write stores both values atomically with the vector.

The listing repository must implement this write behavior:

- Find current state by `(data_source_id, id)` before embedding.
- If the content hash is unchanged and a vector exists, skip the write and embedding request.
- If only non-semantic metadata or ranking state changed while `embedding_text` is unchanged, update the row and reuse the existing vector.
- Otherwise create the embedding and upsert the complete row in one statement; every successful upsert refreshes `indexed_at` and clears `deleted_at`.
- Delete by setting `deleted_at`; an unknown ID may be inserted as a minimal tombstone so deletion state is recorded even when the complete listing row never arrived.
- Invalidate listing result and vector-candidate caches only after a successful write.

The public SV-US-007 deletion contract does not insert unknown-item placeholders for any source kind,
including listings. A missing source/item is an idempotent zero-row success. This keeps behavior
uniform with review and WordPress tables, whose required public columns do not permit minimal
tombstones. Known listings set `deleted_at`; known reviews/WordPress content set `status=deleted`.
Source-wide deletion changes only active rows in the matching source-kind table and never mutates the
installation allowlist.

Repository JSONB parameters must be sent as JSONB values, not JSON-encoded strings. Include an idempotent repair migration for legacy string-valued JSONB rows.

## Directorist Reviews

```sql
CREATE TABLE directorist_reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  search_key BIGINT GENERATED ALWAYS AS IDENTITY UNIQUE,
  data_source_id UUID NOT NULL REFERENCES data_sources(id),
  parent_data_source_id UUID NOT NULL,
  parent_listing_id BIGINT NOT NULL,
  source_id TEXT NOT NULL,
  source_url TEXT NOT NULL,
  title TEXT NOT NULL,
  body TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'deleted')),
  rating_value NUMERIC(3, 2) NULL,
  categories TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  locations TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  review_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  search_document TEXT NOT NULL DEFAULT '',
  source_updated_at TIMESTAMPTZ NULL,
  content_hash TEXT NOT NULL,
  raw_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  normalized_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (data_source_id, source_id),
  FOREIGN KEY (parent_data_source_id, parent_listing_id)
    REFERENCES listings(data_source_id, id) ON DELETE CASCADE
);

CREATE INDEX directorist_reviews_data_source_idx ON directorist_reviews (data_source_id);
CREATE INDEX directorist_reviews_parent_idx ON directorist_reviews (parent_listing_id);
CREATE INDEX directorist_reviews_status_idx ON directorist_reviews (status);
CREATE INDEX directorist_reviews_rating_idx ON directorist_reviews (rating_value);

CREATE INDEX directorist_reviews_bm25_idx
ON directorist_reviews
USING bm25 (search_key, search_document)
WITH (key_field = 'search_key');
```

## WordPress Content

Each enabled non-Directorist post type shares the `wordpress_content` table and remains distinguishable by `data_source_id` and `wordpress_post_type`.

```sql
CREATE TABLE wordpress_content (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  search_key BIGINT GENERATED ALWAYS AS IDENTITY UNIQUE,
  data_source_id UUID NOT NULL REFERENCES data_sources(id),
  source_id TEXT NOT NULL,
  wordpress_post_type TEXT NOT NULL,
  source_url TEXT NOT NULL,
  title TEXT NOT NULL,
  summary TEXT NOT NULL DEFAULT '',
  body TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'draft', 'deleted')),
  taxonomies JSONB NOT NULL DEFAULT '{}'::jsonb,
  post_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  search_document TEXT NOT NULL DEFAULT '',
  source_updated_at TIMESTAMPTZ NULL,
  content_hash TEXT NOT NULL,
  raw_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  normalized_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (data_source_id, source_id)
);

CREATE INDEX wordpress_content_data_source_idx ON wordpress_content (data_source_id);
CREATE INDEX wordpress_content_post_type_idx ON wordpress_content (wordpress_post_type);
CREATE INDEX wordpress_content_status_idx ON wordpress_content (status);
CREATE INDEX wordpress_content_taxonomies_gin_idx ON wordpress_content USING GIN (taxonomies);
CREATE INDEX wordpress_content_metadata_gin_idx ON wordpress_content USING GIN (post_metadata);

CREATE INDEX wordpress_content_bm25_idx
ON wordpress_content
USING bm25 (search_key, search_document)
WITH (key_field = 'search_key');
```

The application must validate that a `data_source_id` has the matching `source_kind` before writing: `directorist_listing` to `listings`, `directorist_review` to `directorist_reviews`, and `wordpress_post` to `wordpress_content`.

Read-model repository inputs are source-specific prepared records. Listing preparation may leave
nullable `embedding_text`, `embedding`, and `content_hash` empty before indexing, while review and
WordPress prepared records must include their non-null `search_document` and `content_hash` fields.
SV-US-006 creates those deterministic values; SV-US-005 defines and enforces the table boundary and
does not store placeholder hashes. All `data_sources` refresh plus matching content writes run in one
transaction. Review writes resolve both the classified parent source and the composite parent listing
before inserting, returning `409 parent_listing_missing` with no orphan content row when absent.

Registering or refreshing `data_sources` never modifies `installation_config.allowed_data_source_keys`.
The same `source_id` remains unique only within its concrete `data_source_id`; it may exist under a
different source key without collision.

## Embeddings And Hybrid Search

Listing vectors live on the `listings` row so normalized state, hash, embedding text, vector, and soft-delete state are updated atomically. Reviews and WordPress content retain source-specific embedding tables because they have independent lifecycles and repository contracts.

```sql
CREATE TABLE directorist_review_embeddings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  review_id UUID NOT NULL REFERENCES directorist_reviews(id) ON DELETE CASCADE,
  embedding_model TEXT NOT NULL,
  embedding_dimensions INTEGER NOT NULL,
  embedding_text TEXT NOT NULL,
  embedding vector(1536) NOT NULL,
  content_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (review_id, embedding_model, embedding_dimensions)
);

CREATE INDEX directorist_review_embeddings_vector_idx
ON directorist_review_embeddings
USING ivfflat (embedding vector_cosine_ops);

CREATE TABLE wordpress_content_embeddings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wordpress_content_id UUID NOT NULL REFERENCES wordpress_content(id) ON DELETE CASCADE,
  embedding_model TEXT NOT NULL,
  embedding_dimensions INTEGER NOT NULL,
  embedding_text TEXT NOT NULL,
  embedding vector(1536) NOT NULL,
  content_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (wordpress_content_id, embedding_model, embedding_dimensions)
);

CREATE INDEX wordpress_content_embeddings_vector_idx
ON wordpress_content_embeddings
USING ivfflat (embedding vector_cosine_ops);
```

RAG queries search only the tables represented by the persisted allowed data-source keys and apply source-specific structured filters before ranking. Listing retrieval obtains BM25 and vector candidates from `listings`; the other kinds obtain BM25 candidates from their content table and vector candidates from their source-specific embedding table. The application bounds both sets, fuses their ranks with the configured reciprocal-rank and weight settings, then performs final application ranking.

Final ranking does not add a persistence table in SV-US-009. The repository projects only safe
ranking fields from the existing dedicated listing/review/content columns and configured listing
metadata. Policy version and scores are carried in the internal result and later message/tool audit
payloads; evaluation fixtures and baselines are version-controlled artifacts. See
[`RANKING_AND_CITATION_CONTRACT.md`](RANKING_AND_CITATION_CONTRACT.md).

The shared predicate builder resolves only server-owned table/JSON surfaces. Generic metadata/date
keys must appear in `data_sources.context_metadata.retrieval_filter_keys`, are passed as query
parameters, and never become SQL identifiers. BM25 and vector statements receive the same allowed
data-source IDs and predicates. Review candidates join an active, URL-bearing parent listing whose
source key is also allowed. Detail lookup repeats the same active, URL, allowlist, and parent checks.

Do not run the BM25 migration until the `pg_search` package compatibility gate and extension checks pass. The migration must create all three indexes and run `ANALYZE` after bulk indexing. Direct smoke queries using the `|||` operator and `pdb.score(search_key)` must succeed for every populated source kind before `HYBRID_SEARCH_ENABLED=true`. If the package match cannot be proven, the extension or any required index is unavailable, or a smoke query fails, keep hybrid search disabled and report the degraded state. If the embedding dimension changes, add new vector storage and explicitly re-embed each source kind; do not silently alter an existing vector column in place.

## Conversations

Provider selection is runtime configuration and is deliberately absent from the database schema. Conversation, message, tool, and usage records use provider-neutral identifiers and payloads. The application may emit the active provider to ephemeral logs/metrics, but it must not persist an AI-provider discriminator or provider-specific conversation state in application tables.

```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_user_id TEXT NULL,
  anonymous_session_id TEXT NULL,
  channel TEXT NOT NULL DEFAULT 'web' CHECK (channel IN ('web', 'mobile', 'admin_test')),
  title TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived', 'deleted')),
  langgraph_thread_id TEXT NOT NULL,
  summary TEXT NOT NULL DEFAULT '',
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX conversations_user_idx ON conversations (external_user_id);
CREATE INDEX conversations_session_idx ON conversations (anonymous_session_id);

CREATE TABLE conversation_messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  role TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system', 'tool')),
  content TEXT NOT NULL,
  token_input_count INTEGER NULL,
  token_output_count INTEGER NULL,
  citations JSONB NOT NULL DEFAULT '[]'::jsonb,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX conversation_messages_conversation_idx
ON conversation_messages (conversation_id, created_at);

CREATE TABLE conversation_tool_calls (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  message_id UUID NULL REFERENCES conversation_messages(id) ON DELETE SET NULL,
  tool_name TEXT NOT NULL,
  arguments JSONB NOT NULL DEFAULT '{}'::jsonb,
  result JSONB NOT NULL DEFAULT '{}'::jsonb,
  status TEXT NOT NULL CHECK (status IN ('success', 'error', 'skipped')),
  latency_ms INTEGER NULL,
  error_code TEXT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Personalization

```sql
CREATE TABLE user_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_user_id TEXT NOT NULL UNIQUE,
  display_name TEXT NOT NULL DEFAULT '',
  email_hash TEXT NULL,
  home_location TEXT NULL,
  latitude NUMERIC(10, 7) NULL,
  longitude NUMERIC(10, 7) NULL,
  custom_preferences JSONB NOT NULL DEFAULT '{}'::jsonb,
  interests TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  budget_preference TEXT NULL,
  travel_distance_miles INTEGER NULL,
  preferences JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_favorites (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_user_id TEXT NOT NULL,
  data_source_key TEXT NOT NULL,
  source_kind TEXT NOT NULL CHECK (source_kind IN ('directorist_listing', 'directorist_review', 'wordpress_post')),
  source_id TEXT NOT NULL,
  favorite_type TEXT NOT NULL DEFAULT 'saved',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (external_user_id, data_source_key, source_id, favorite_type)
);
```

## Analytics And Admin

```sql
CREATE TABLE usage_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type TEXT NOT NULL,
  route TEXT NULL,
  method TEXT NULL,
  status_code INTEGER NULL,
  success BOOLEAN NOT NULL DEFAULT true,
  conversation_id UUID NULL REFERENCES conversations(id) ON DELETE SET NULL,
  external_user_id TEXT NULL,
  anonymous_session_id TEXT NULL,
  latency_ms INTEGER NULL,
  input_tokens INTEGER NULL,
  output_tokens INTEGER NULL,
  retrieval_count INTEGER NULL,
  error_code TEXT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX usage_events_created_idx ON usage_events (created_at);
CREATE INDEX usage_events_type_idx ON usage_events (event_type);

-- SV-US-006 records one content_index event per indexing attempt. Metadata is restricted to safe
-- source kind, outcome, embedding model/dimensions, normalization version, attempt count, and
-- embedding latency. Full content, raw payloads, vectors, provider bodies, credentials, and guessed
-- cost are forbidden. Observability failure does not roll back a committed content transaction.
-- SV-US-007 adds safe content_bulk_index, content_delete, and content_delete_source events. Bulk
-- metadata contains only item/outcome counts; delete metadata contains source kind and deleted count.
-- SV-US-008 adds content_retrieval. It uses retrieval_count and metadata limited to mode/degradation,
-- allowlist version, safe limits, source-kind count, candidate counts, and branch latencies. Query
-- text, filters, result identities/content, raw scores, vectors, SQL, and provider identity are forbidden.

CREATE TABLE admin_users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  display_name TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'disabled')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE admin_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  admin_user_id UUID NOT NULL REFERENCES admin_users(id) ON DELETE CASCADE,
  session_hash TEXT NOT NULL UNIQUE,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE schema_migrations (
  version TEXT PRIMARY KEY,
  applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## LangGraph Checkpoints

Use the official LangGraph checkpoint storage package or a local PostgreSQL checkpoint table during implementation. Keep checkpoint rows separate from `conversation_messages`; checkpoints are replay/recovery state, while conversation tables are product/audit state.

Minimum custom table shape if needed:

```sql
CREATE TABLE langgraph_checkpoints (
  thread_id TEXT NOT NULL,
  checkpoint_id TEXT NOT NULL,
  parent_checkpoint_id TEXT NULL,
  checkpoint JSONB NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (thread_id, checkpoint_id)
);
```

## Retention And Privacy

- Conversations should support soft deletion through `conversations.status = 'deleted'`.
- User profile deletion should remove or anonymize `user_profiles`, `user_favorites`, and conversation user references.
- Usage analytics may be retained longer after removing direct user identifiers.
- Store email hashes instead of raw email when the backend only needs stable identity matching.
