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
  installation_name TEXT NOT NULL DEFAULT 'Palm Beach Mama Club',
  primary_domain TEXT NOT NULL,
  wordpress_site_url TEXT NOT NULL,
  timezone TEXT NOT NULL DEFAULT 'America/New_York',
  settings JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

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

## Content Index

```sql
CREATE TABLE content_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_type TEXT NOT NULL CHECK (
    source_type IN ('business_listing', 'event_listing', 'review', 'weekend_pick', 'blog_post', 'faq', 'promotion')
  ),
  source_id TEXT NOT NULL,
  source_url TEXT NOT NULL,
  title TEXT NOT NULL,
  summary TEXT NOT NULL DEFAULT '',
  body TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'draft', 'expired', 'deleted')),
  directory_type TEXT NULL,
  categories TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  locations TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  tags TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  amenities TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
  age_min INTEGER NULL,
  age_max INTEGER NULL,
  price_level TEXT NULL,
  starts_at TIMESTAMPTZ NULL,
  ends_at TIMESTAMPTZ NULL,
  latitude NUMERIC(10, 7) NULL,
  longitude NUMERIC(10, 7) NULL,
  is_featured BOOLEAN NOT NULL DEFAULT false,
  is_sponsored BOOLEAN NOT NULL DEFAULT false,
  sponsor_label TEXT NULL,
  business_member_level TEXT NULL,
  source_updated_at TIMESTAMPTZ NULL,
  content_hash TEXT NOT NULL,
  raw_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  normalized_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (source_type, source_id)
);

CREATE INDEX content_items_source_type_idx ON content_items (source_type);
CREATE INDEX content_items_status_idx ON content_items (status);
CREATE INDEX content_items_event_dates_idx ON content_items (starts_at, ends_at);
CREATE INDEX content_items_featured_idx ON content_items (is_featured, is_sponsored);
CREATE INDEX content_items_categories_gin_idx ON content_items USING GIN (categories);
CREATE INDEX content_items_locations_gin_idx ON content_items USING GIN (locations);
CREATE INDEX content_items_payload_gin_idx ON content_items USING GIN (normalized_payload);
```

## Embeddings

```sql
CREATE TABLE content_embeddings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content_item_id UUID NOT NULL REFERENCES content_items(id) ON DELETE CASCADE,
  embedding_model TEXT NOT NULL,
  embedding_dimensions INTEGER NOT NULL,
  embedding_text TEXT NOT NULL,
  embedding vector(1536) NOT NULL,
  content_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (content_item_id, embedding_model, embedding_dimensions)
);

CREATE INDEX content_embeddings_vector_idx
ON content_embeddings
USING ivfflat (embedding vector_cosine_ops);
```

If the embedding dimension changes, create a new embedding column/table migration rather than mutating the existing `vector(1536)` column in place.

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
  children JSONB NOT NULL DEFAULT '[]'::jsonb,
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
  content_item_id UUID NULL REFERENCES content_items(id) ON DELETE SET NULL,
  source_type TEXT NOT NULL,
  source_id TEXT NOT NULL,
  favorite_type TEXT NOT NULL DEFAULT 'saved',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (external_user_id, source_type, source_id, favorite_type)
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

