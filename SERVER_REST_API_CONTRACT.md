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
  "service": "ai-sunny-api",
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
  "domain": "palmbeachmamaclub.com",
  "wordpress_site_url": "https://palmbeachmamaclub.com",
  "installation_name": "Palm Beach Mama Club"
}
```

Response:

```json
{
  "ok": true,
  "api_key": "ais_live_xxxxx",
  "key_prefix": "ais_live",
  "installation": {
    "domain": "palmbeachmamaclub.com",
    "timezone": "America/New_York"
  }
}
```

The backend stores only a hash of `api_key`.

## Content Routes

### `POST /content/upsert`

Indexes one source record.

Request:

```json
{
  "source_type": "business_listing",
  "source_id": "1514",
  "source_url": "https://palmbeachmamaclub.com/listing/example",
  "title": "Sunny Play Cafe",
  "summary": "Indoor play space with coffee and toddler area.",
  "body": "Full cleaned content.",
  "status": "active",
  "categories": ["Indoor Activities"],
  "locations": ["Palm Beach Gardens"],
  "tags": ["toddlers", "rainy day"],
  "amenities": ["playground", "changing table"],
  "age_min": 1,
  "age_max": 8,
  "price_level": "$$",
  "starts_at": null,
  "ends_at": null,
  "is_featured": true,
  "is_sponsored": false,
  "raw_payload": {}
}
```

Response:

```json
{
  "ok": true,
  "status": "indexed",
  "content_item_id": "uuid",
  "changed": true
}
```

### `POST /content/bulk-upsert`

Indexes multiple records. Maximum batch size should default to `50`.

Request:

```json
{
  "items": [
    {
      "source_type": "event_listing",
      "source_id": "2001",
      "source_url": "https://example.com/events/story-time",
      "title": "Saturday Story Time",
      "status": "active"
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

Removes content from retrieval. Prefer tombstoning over hard deletion.

Request:

```json
{
  "source_type": "business_listing",
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

## Chat Routes

### `POST /chat`

Runs one non-streaming chat turn.

Request:

```json
{
  "conversation_id": "optional-uuid",
  "anonymous_session_id": "browser-session-id",
  "external_user_id": "optional-wordpress-user-id",
  "message": "I have a 3-year-old and a 7-year-old. What can we do this Saturday?",
  "context": {
    "timezone": "America/New_York",
    "page_url": "https://palmbeachmamaclub.com"
  }
}
```

Response:

```json
{
  "conversation_id": "uuid",
  "message_id": "uuid",
  "answer": "Here are a few good options...",
  "recommendations": [
    {
      "source_type": "event_listing",
      "source_id": "2001",
      "title": "Saturday Story Time",
      "url": "https://example.com/events/story-time",
      "reason": "Good fit for both ages and happens this Saturday.",
      "is_sponsored": false,
      "is_featured": false
    }
  ],
  "citations": [
    {
      "title": "Saturday Story Time",
      "url": "https://example.com/events/story-time",
      "source_type": "event_listing"
    }
  ],
  "follow_up_questions": []
}
```

### `POST /chat/stream`

Runs one streaming chat turn over Server-Sent Events.

Request matches `POST /chat`.

Events:

```text
event: conversation
data: {"conversation_id":"uuid"}

event: delta
data: {"text":"Here are"}

event: recommendation
data: {"source_id":"2001","title":"Saturday Story Time","url":"https://example.com/events/story-time"}

event: done
data: {"message_id":"uuid","citations":[...]}
```

On failure:

```text
event: error
data: {"code":"openai_error","message":"Ask Sunny had trouble finishing this response."}
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
  "source_types": ["business_listing", "event_listing"],
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
  "content_counts": {
    "business_listing": 250,
    "event_listing": 80
  },
  "last_indexed_at": "2026-07-06T12:00:00Z"
}
```

## Validation Rules

- `message` is required for chat and capped by `MAX_CHAT_INPUT_CHARS`.
- `source_type` must be one of the known content types.
- `source_id`, `source_url`, and `title` are required for active content.
- Event records should include `starts_at` where available.
- Deleted content requires only `source_type` and `source_id`.
- Chat routes must never accept raw SQL, arbitrary tool names, or model overrides from clients.

