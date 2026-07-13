# Server REST API Contract

All non-public server routes require:

```http
Authorization: Bearer {installation-or-admin-api-key}
Content-Type: application/json
Accept: application/json
```

Error responses use:

```json
{
  "error": {
    "code": "validation_error",
    "message": "Human-readable error message",
    "field": "optional_field_name"
  }
}
```

## Public Routes

### `GET /health`

Returns server health.

```json
{
  "ok": true,
  "service": "ask-sunny-api",
  "version": "1.0.0",
  "database": "ok",
  "redis": "disabled"
}
```

### `POST /auth/provision-installation`

Creates or rotates the WordPress installation API key. Uses the provisioning secret, not an existing installation key.

Request:

```json
{
  "provisioning_key": "long-shared-secret",
  "domain": "example.com",
  "wordpress_site_url": "https://example.com",
  "installation_name": "Example WordPress Site"
}
```

Response:

```json
{
  "ok": true,
  "api_key": "ask_live_xxxxx",
  "key_prefix": "ask_live",
  "installation": {
    "domain": "example.com",
    "timezone": "UTC"
  }
}
```

The backend stores only a hash of `api_key`.

## Retrieval Configuration Routes

### `PUT /retrieval/allowed-data-sources`

Persists the complete allowlist computed by the WordPress plugin. This replaces the previous list atomically; it does not delete content. The installation key may update the allowlist for its installation.

Request:

```json
{
  "allowed_data_source_keys": [
    "directorist:businesses",
    "directorist:businesses:reviews",
    "directorist:events",
    "directorist:events:reviews",
    "wordpress:post"
  ],
  "expected_version": 4
}
```

Response:

```json
{
  "ok": true,
  "allowed_data_source_keys": [
    "directorist:businesses",
    "directorist:businesses:reviews",
    "directorist:events",
    "directorist:events:reviews",
    "wordpress:post"
  ],
  "version": 5,
  "updated_at": "2026-07-13T10:30:00Z"
}
```

Duplicate keys are removed and the stored order is canonicalized. `expected_version` prevents an older admin request from overwriting newer configuration; a mismatch returns `409 retrieval_config_conflict`. An empty list is stored as an empty list and makes retrieval fail closed with no candidates.

The list uses concrete `data_source_key` classifications rather than broad `source_kind` values. For example, `directorist:events` and `directorist:events:reviews` can be allowed independently even though both are Directorist data.

## Content Routes

WordPress owns the source settings UI and indexing filters. It synchronizes the resulting allowlist through the retrieval configuration route above. A content upsert carries its source identity and context; the backend creates or refreshes retrieval metadata for that key as part of the upsert regardless of whether that key is currently allowed for RAG.

### `POST /content/upsert`

Indexes one source record. `source_kind` dispatches the payload to `directorist_listings`, `directorist_reviews`, or `wordpress_content`; there is no shared content-items table.

Request:

```json
{
  "source_kind": "directorist_listing",
  "data_source_key": "directorist:businesses",
  "data_source_label": "Business Directory",
  "source_context": {"content_kind": "business"},
  "source_id": "1514",
  "source_url": "https://example.com/directory/example-provider",
  "title": "Example Service Provider",
  "summary": "A short description of the services offered.",
  "tagline": "Trusted help when you need it.",
  "body": "Full cleaned content.",
  "status": "active",
  "wordpress_post_type": "at_biz_dir",
  "directory_type_id": "7",
  "directory_type_slug": "businesses",
  "directory_type_label": "Business Directory",
  "categories": ["Professional Services"],
  "locations": ["Downtown"],
  "tags": ["appointment", "accessible"],
  "amenities": ["wheelchair access", "parking"],
  "price_amount": 125.00,
  "price_currency": "USD",
  "price_display": "$125",
  "view_count": 320,
  "phone": "+1-555-0100",
  "zip_postal_code": "10001",
  "public_email": "hello@example.com",
  "fax": null,
  "website_url": "https://provider.example.com",
  "social_links": [{"network": "facebook", "url": "https://facebook.com/example"}],
  "address": "100 Main Street, Downtown",
  "latitude": 40.7128000,
  "longitude": -74.0060000,
  "featured_image_url": "https://example.com/uploads/provider.jpg",
  "image_urls": ["https://example.com/uploads/provider.jpg"],
  "is_featured": true,
  "listing_metadata": {
    "availability": {
      "label": "Availability",
      "field_type": "select",
      "value": "weekdays"
    }
  },
  "raw_payload": {}
}
```

Response:

```json
{
  "ok": true,
  "status": "indexed",
  "content_record_id": "uuid",
  "changed": true
}
```

A Directorist review is a separate content record:

```json
{
  "source_kind": "directorist_review",
  "data_source_key": "directorist:events:reviews",
  "data_source_label": "Event Directory Reviews",
  "source_context": {"content_kind": "review", "reviewed_content_kind": "event"},
  "source_id": "845",
  "source_url": "https://example.com/events/community-workshop#comment-845",
  "title": "Review for Community Workshop",
  "body": "Cleaned approved review text.",
  "status": "active",
  "wordpress_post_type": "at_biz_dir",
  "directory_type_id": "42",
  "directory_type_slug": "events",
  "directory_type_label": "Event Directory",
  "parent_data_source_key": "directorist:events",
  "parent_source_id": "2001",
  "categories": ["Workshops", "Education"],
  "locations": ["Downtown"],
  "rating_value": 5,
  "listing_metadata": {},
  "raw_payload": {}
}
```

Review upserts use the same response shape as listing and WordPress-post upserts.

An enabled WordPress post-type record is stored in `wordpress_content`:

```json
{
  "source_kind": "wordpress_post",
  "data_source_key": "wordpress:post",
  "data_source_label": "Blog",
  "source_context": {"content_kind": "article", "audience": "public"},
  "source_id": "501",
  "wordpress_post_type": "post",
  "source_url": "https://example.com/blog/visitor-guide",
  "title": "Visitor Guide",
  "summary": "A short guide for visitors.",
  "body": "Full cleaned article content.",
  "status": "active",
  "taxonomies": {"category": ["Guides"], "post_tag": ["planning"]},
  "post_metadata": {"audience": "new-visitors"},
  "raw_payload": {}
}
```

### `POST /content/bulk-upsert`

Indexes multiple records. Maximum batch size should default to `50`. A batch may contain multiple source kinds; the service validates and upserts each item into its matching table and embedding table.

Request:

```json
{
  "items": [
    {
      "source_kind": "directorist_listing",
      "data_source_key": "directorist:events",
      "data_source_label": "Event Directory",
      "source_context": {"content_kind": "event"},
      "source_id": "2001",
      "source_url": "https://example.com/events/community-workshop",
      "title": "Community Workshop",
      "status": "active",
      "wordpress_post_type": "at_biz_dir"
    }
  ]
}
```

Response:

```json
{
  "ok": true,
  "indexed": 1,
  "unchanged": 0,
  "failed": 0,
  "errors": []
}
```

### `POST /content/delete`

Removes content from retrieval. `data_source_key` resolves the source kind and matching table. Prefer tombstoning over hard deletion.

Request:

```json
{
  "data_source_key": "directorist:businesses",
  "source_id": "1514"
}
```

Response:

```json
{
  "ok": true,
  "status": "deleted"
}
```

### `POST /content/delete-by-data-source`

Tombstones every active record for a key in its source-kind table. WordPress calls this only for an explicit admin **Delete all indexed data** action or an equivalent deliberate maintenance operation. Disabling an optional WordPress source must not call this route.

Request:

```json
{
  "data_source_key": "wordpress:post"
}
```

Response:

```json
{
  "ok": true,
  "status": "deleted",
  "deleted_items": 35
}
```

## Chat Routes

### `POST /chat`

Runs one complete chat turn. The route returns one JSON response after retrieval and answer generation finish; it does not emit partial tokens or events.

Request:

```json
{
  "conversation_id": "optional-uuid",
  "anonymous_session_id": "browser-session-id",
  "external_user_id": "optional-wordpress-user-id",
  "message": "What workshops are available this Saturday?",
  "context": {
    "timezone": "UTC",
    "page_url": "https://example.com"
  }
}
```

The chat caller does not provide `allowed_data_source_keys`. The backend loads its persisted allowlist for every chat turn. Retrieval and tool calls may narrow that list for the user's context but must never search outside it.

Response:

```json
{
  "conversation_id": "uuid",
  "message_id": "uuid",
  "answer": "Here are a few good options...",
  "recommendations": [
    {
      "source_kind": "directorist_listing",
      "data_source_key": "directorist:events",
      "source_id": "2001",
      "title": "Community Workshop",
      "url": "https://example.com/events/community-workshop",
      "reason": "Matches the requested date and location.",
      "is_featured": false
    }
  ],
  "citations": [
    {
      "title": "Community Workshop",
      "url": "https://example.com/events/community-workshop",
      "source_kind": "directorist_listing",
      "data_source_key": "directorist:events"
    }
  ],
  "follow_up_questions": []
}
```

### `GET /conversations/:id`

Returns conversation history for an authenticated caller.

Response:

```json
{
  "conversation": {
    "id": "uuid",
    "title": "Saturday plans",
    "status": "active"
  },
  "messages": [
    {
      "id": "uuid",
      "role": "user",
      "content": "What can we do this Saturday?",
      "created_at": "2026-07-06T12:00:00Z"
    }
  ]
}
```

## Admin Routes

Admin routes require an admin API key or admin session.

### `POST /admin/reindex`

Requests a backend reindex job from already-known WordPress payloads or asks WordPress to trigger reindexing through the plugin.

Request:

```json
{
  "data_source_keys": ["directorist:businesses", "directorist:businesses:reviews", "directorist:events", "directorist:events:reviews", "wordpress:post"],
  "force": false
}
```

Response:

```json
{
  "ok": true,
  "job_id": "uuid",
  "status": "queued"
}
```

### `GET /admin/usage`

Returns usage and latency metrics.

Query parameters:

- `from`
- `to`
- `event_type`

Response:

```json
{
  "totals": {
    "chat_turns": 120,
    "indexing_events": 30,
    "errors": 2
  },
  "daily": []
}
```

### `GET /admin/diagnostics`

Returns operational state.

```json
{
  "database": "ok",
  "redis": "disabled",
  "openai": "configured",
  "retrieval_config": {
    "allowed_data_source_keys": [
      "directorist:businesses",
      "directorist:businesses:reviews",
      "directorist:events",
      "directorist:events:reviews"
    ],
    "version": 6,
    "updated_at": "2026-07-13T10:30:00Z"
  },
  "content_counts": {
    "directorist:businesses": 250,
    "directorist:businesses:reviews": 410,
    "directorist:events": 80,
    "directorist:events:reviews": 125,
    "wordpress:post": 35
  },
  "last_indexed_at": "2026-07-06T12:00:00Z"
}
```

## Validation Rules

- `message` is required for chat and capped by `MAX_CHAT_INPUT_CHARS`.
- Chat requests must reject or ignore any caller-supplied `allowed_data_source_keys`; only the persisted backend list is authoritative.
- `source_kind` must be `directorist_listing`, `directorist_review`, or `wordpress_post`.
- The backend must route each source kind only to its matching table and embedding table; it must never store unlike kinds in a shared generic content table.
- `data_source_key`, `data_source_label`, and `source_context` must be valid and internally consistent. The backend upserts source metadata from the received content rather than requiring a pre-registered or enabled source.
- `source_id`, `source_url`, and `title` are required for active content.
- Core Directorist preset fields use their canonical top-level payload fields and database columns.
- Every non-core field uses the flat `listing_metadata` map, whether it is a Directorist custom field or an extension/third-party preset field. Do not create separate namespaces.
- Listings whose directory type supports extension-provided event fields include those typed values under `listing_metadata`; they remain `directorist_listing` records.
- Each `listing_metadata` entry uses its stable field key and includes a label, field type, and sanitized typed value. Provider identity is optional metadata when available.
- Apply configured limits to metadata field count, key/label length, array size, and serialized payload size. Reject executable values, unsafe URLs, and unexpected nested objects.
- Active `directorist_review` records require `parent_data_source_key` and `parent_source_id`; the parent key must identify the listing source for the same directory type.
- Review upsert resolves its parent to `directorist_listings.id`; if the listing is not indexed yet, return `409 parent_listing_missing` so WordPress can index the listing first and retry the review.
- Only approved public reviews are active. Unapproved, spammed, trashed, or deleted reviews are tombstoned.
- Deleted content requires only `data_source_key` and `source_id`.
- WordPress applies indexing filters before sending content and synchronizes source allowance separately. Every backend candidate query, vector search, detail lookup used by RAG, and model tool call must constrain results to the stored allowlist.
- Chat routes must never accept raw SQL, arbitrary tool names, or model overrides from clients.
