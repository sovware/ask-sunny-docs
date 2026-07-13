# Server Database Schema

The Ask Sunny server uses PostgreSQL as the durable system of record for indexed content, conversations, usage, and admin state. WordPress remains the editorial source of truth; backend content tables are a searchable read model.

Use `BIGSERIAL` or UUIDs consistently during implementation. This document uses UUID primary keys where records may be referenced by browser, mobile, or support tooling.

## Extensions

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

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
  directory_type_id TEXT NULL,
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

`data_source_key` is a stable machine identifier such as `directorist:events`, `directorist:events:reviews`, or `wordpress:post`. Labels may change without changing the key. `parent_data_source_id` links a review source to the listing source for the same directory type. Disabling a WordPress source does not change these rows or its content rows; the persisted backend allowlist excludes it from RAG. Rows are tombstoned only after an explicit deletion request.

## Directorist Listings

```sql
CREATE TABLE directorist_listings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  data_source_id UUID NOT NULL REFERENCES data_sources(id),
  source_id TEXT NOT NULL,
  source_url TEXT NOT NULL,
  title TEXT NOT NULL,
  summary TEXT NOT NULL DEFAULT '',
  tagline TEXT NOT NULL DEFAULT '',
  body TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'draft', 'expired', 'deleted')),
  wordpress_post_type TEXT NOT NULL,
  directory_type_id TEXT NOT NULL,
  directory_type_slug TEXT NOT NULL,
  directory_type_label TEXT NOT NULL,
  source_context JSONB NOT NULL DEFAULT '{}'::jsonb,
  categories TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  locations TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  tags TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  amenities TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  price_amount NUMERIC(20, 4) NULL,
  price_currency TEXT NULL,
  price_display TEXT NOT NULL DEFAULT '',
  view_count BIGINT NOT NULL DEFAULT 0,
  phone TEXT NULL,
  zip_postal_code TEXT NULL,
  public_email TEXT NULL,
  fax TEXT NULL,
  website_url TEXT NULL,
  social_links JSONB NOT NULL DEFAULT '[]'::jsonb,
  address TEXT NOT NULL DEFAULT '',
  latitude NUMERIC(10, 7) NULL,
  longitude NUMERIC(10, 7) NULL,
  featured_image_url TEXT NULL,
  image_urls TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  is_featured BOOLEAN NOT NULL DEFAULT false,
  listing_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  source_updated_at TIMESTAMPTZ NULL,
  content_hash TEXT NOT NULL,
  raw_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  normalized_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (data_source_id, source_id)
);

CREATE INDEX directorist_listings_data_source_idx ON directorist_listings (data_source_id);
CREATE INDEX directorist_listings_status_idx ON directorist_listings (status);
CREATE INDEX directorist_listings_featured_idx ON directorist_listings (is_featured);
CREATE INDEX directorist_listings_categories_gin_idx ON directorist_listings USING GIN (categories);
CREATE INDEX directorist_listings_locations_gin_idx ON directorist_listings USING GIN (locations);
CREATE INDEX directorist_listings_metadata_gin_idx ON directorist_listings USING GIN (listing_metadata);
CREATE INDEX directorist_listings_context_gin_idx ON directorist_listings USING GIN (source_context);
```

Core Directorist preset fields map to dedicated columns. The canonical core mapping includes title, description/body, tagline, excerpt/summary, price, categories, locations, tags, view count, phone, ZIP/postal code, email, fax, social information, website, address/map coordinates, images, amenities when provided by core, and the featured flag. Every non-core field uses the same generic listing metadata column, whether it was created as a custom field or registered as a preset by an extension.

`listing_metadata` is a flat map keyed by the stable Directorist field key:

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

Do not create separate custom-field and third-party-preset namespaces. `provider` is optional descriptive metadata and does not change storage location. Use stable field keys, preserve typed JSON values, and store public file-upload fields as sanitized URLs rather than file bytes. Admin-only/private fields are excluded from `listing_metadata`, `normalized_payload`, and embeddings.

`normalized_payload` is the canonical snapshot used to compute `content_hash` and `embedding_text`. It contains every public content-bearing column plus `listing_metadata`; it excludes `raw_payload` and operational columns. Dynamic metadata that needs frequent structured filtering may receive a migration-managed JSONB expression index without being promoted to a fixed column.

## Directorist Reviews

```sql
CREATE TABLE directorist_reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  data_source_id UUID NOT NULL REFERENCES data_sources(id),
  parent_listing_id UUID NOT NULL REFERENCES directorist_listings(id) ON DELETE CASCADE,
  source_id TEXT NOT NULL,
  source_url TEXT NOT NULL,
  title TEXT NOT NULL,
  body TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'deleted')),
  rating_value NUMERIC(3, 2) NULL,
  categories TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  locations TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  review_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  source_updated_at TIMESTAMPTZ NULL,
  content_hash TEXT NOT NULL,
  raw_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  normalized_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (data_source_id, source_id)
);

CREATE INDEX directorist_reviews_data_source_idx ON directorist_reviews (data_source_id);
CREATE INDEX directorist_reviews_parent_idx ON directorist_reviews (parent_listing_id);
CREATE INDEX directorist_reviews_status_idx ON directorist_reviews (status);
CREATE INDEX directorist_reviews_rating_idx ON directorist_reviews (rating_value);
```

## WordPress Content

Each enabled non-Directorist post type shares the `wordpress_content` table and remains distinguishable by `data_source_id` and `wordpress_post_type`.

```sql
CREATE TABLE wordpress_content (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
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
```

The application must validate that a `data_source_id` has the matching `source_kind` before writing: `directorist_listing` to `directorist_listings`, `directorist_review` to `directorist_reviews`, and `wordpress_post` to `wordpress_content`.

## Embeddings

Use a separate embedding table for each content table so every foreign key is enforceable and deletion cascades correctly.

```sql
CREATE TABLE directorist_listing_embeddings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  listing_id UUID NOT NULL REFERENCES directorist_listings(id) ON DELETE CASCADE,
  embedding_model TEXT NOT NULL,
  embedding_dimensions INTEGER NOT NULL,
  embedding_text TEXT NOT NULL,
  embedding vector(1536) NOT NULL,
  content_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (listing_id, embedding_model, embedding_dimensions)
);

CREATE INDEX directorist_listing_embeddings_vector_idx
ON directorist_listing_embeddings
USING ivfflat (embedding vector_cosine_ops);

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

RAG queries search only the tables represented by the persisted allowed data-source keys, apply source-specific structured filters, then merge and rank the result sets in the application layer. If the embedding dimension changes, migrate all three embedding tables and re-embed each source kind rather than mutating existing vector columns in place.

## Conversations

```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_user_id TEXT NULL,
  anonymous_session_id TEXT NULL,
  channel TEXT NOT NULL DEFAULT 'web' CHECK (channel IN ('web', 'mobile', 'admin_test')),
  title TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived', 'deleted')),
  openai_conversation_id TEXT NULL,
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
  response_id TEXT NULL,
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
