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
  "paradedb": "ok",
  "vector_search": "ok",
  "bm25_search": "ok",
  "hybrid_search": {
    "requested": true,
    "effective": true,
    "status": "enabled",
    "reason": null
  },
  "ai_provider": "openai",
  "redis": "disabled"
}
```

`hybrid_search.status` may report `disabled` or `degraded` when package compatibility is unproven, `pg_search`, a required BM25 index, or smoke verification is unavailable. Health must not report BM25 as enabled merely because the environment flag is set. A requested-but-ineffective hybrid configuration reports `requested: true`, `effective: false`, and a stable reason code.

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
  "api_key": "ask_live_7f3a91c24bd8e610_<random-secret>",
  "key_prefix": "ask_live_7f3a91c24bd8e610",
  "scopes": [
    "retrieval:write",
    "content:write",
    "chat:write",
    "conversations:read"
  ],
  "rotated_previous_key": false,
  "installation": {
    "domain": "example.com",
    "timezone": "UTC"
  }
}
```

`domain` is a lower-case hostname without a scheme, port, path, query, or fragment.
`wordpress_site_url` is an absolute HTTPS URL whose hostname exactly matches `domain`; subdirectory
installations may retain a path, while query and fragment components are rejected. The backend trims
the installation name, canonicalizes the identity, and creates the singleton installation record on
first provisioning. Later requests must match that stored domain and WordPress site URL or return
`409 installation_identity_conflict` without rotating a key.

The key format is `ask_live_<16-lowercase-hex-key-id>_<43-character-base64url-secret>`. The unique
`key_prefix` is the format through the key-id segment and may be logged for credential identification;
the full API key and secret segment must never be logged. The backend stores only the SHA-256 digest
of the complete high-entropy API key. A provisioned WordPress installation key receives exactly the
listed server-defined scopes; the caller cannot add scopes in the request.

Provisioning is also the rotation operation. The first successful request returns
`rotated_previous_key: false`. A later successful request for the same canonical installation creates
a new key, immediately revokes every prior active `wordpress_installation` key in the same database
transaction, returns `rotated_previous_key: true`, and writes cross-referenced rotation metadata on
the new and revoked rows. If any part of the transaction fails, the previous key remains active and
no new key is issued. The plaintext key is returned only in this response and cannot be recovered.

An invalid provisioning secret returns the same `401 authentication_error` used for invalid bearer
credentials and performs no installation or key write. Provisioning-secret comparison is
constant-time. Validation errors never echo the provisioning key, API key, or secret fragment.

For protected routes, malformed, unknown, hash-mismatched, and revoked bearer keys all return the
same `401 authentication_error`. An authenticated key missing a route's required scope returns
`403 forbidden` without naming the missing scope. Only a fully authorized request updates
`last_used_at`.

## Retrieval Configuration Routes

### `PUT /retrieval/allowed-data-sources`

Persists the complete allowlist computed by the WordPress plugin. This replaces the previous list atomically; it does not delete content. The installation key may update the allowlist for its installation.

Requires the `retrieval:write` installation-key scope.

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

Each input key is trimmed and lowercased, then must match one of
`directorist:<slug>`, `directorist:<slug>:reviews`, or `wordpress:<post-type>`. The `<slug>` and
`<post-type>` segments match `[a-z0-9][a-z0-9_-]{0,63}`. Duplicate canonical keys are removed and the
result is stored in ascending bytewise order. The array may contain at most
`MAX_ALLOWED_DATA_SOURCE_KEYS` entries, default `1000`. Shape validation does not require an existing
`data_sources` row because the initial allowlist may be synchronized before content arrives.

`expected_version` is a non-negative safe integer. A successful request, including a no-op list,
increments the stored version by exactly one and sets `updated_at` from the database clock. The
conditional replacement and increment occur in one statement. A stale or concurrently losing
request receives:

```json
{
  "error": {
    "code": "retrieval_config_conflict",
    "message": "The retrieval configuration changed; reload it before retrying."
  }
}
```

The conflict response does not include the current list. The caller reloads it through the
diagnostic route below; the service never retries a stale write automatically.

### `GET /retrieval/allowed-data-sources`

Returns the authoritative snapshot for WordPress reconciliation and diagnostics. Requires the
`retrieval:write` installation-key scope.

Response:

```json
{
  "ok": true,
  "allowed_data_source_keys": [
    "directorist:businesses",
    "directorist:businesses:reviews",
    "wordpress:post"
  ],
  "version": 5,
  "updated_at": "2026-07-13T10:30:00Z"
}
```

Before the first sync, the response contains an empty list, version `0`, and `updated_at: null`.
This route is the narrow installation-facing diagnostic surface; it does not weaken the separate
admin-authentication requirement for `GET /admin/diagnostics`.

The list uses concrete `data_source_key` classifications rather than broad `source_kind` values. For example, `directorist:events` and `directorist:events:reviews` can be allowed independently even though both are Directorist data.

## Content Routes

WordPress owns the source settings UI and indexing filters. It synchronizes the resulting allowlist through the retrieval configuration route above. A content upsert carries its source identity and context; the backend creates or refreshes retrieval metadata for that key as part of the upsert regardless of whether that key is currently allowed for RAG.

### `POST /content/upsert`

Indexes one source record. `source_kind` dispatches the payload to `listings`, `directorist_reviews`, or `wordpress_content`; there is no shared content-items table. Listing writes validate and normalize the payload, hash deterministic content, reuse an existing vector when embedding text is unchanged, otherwise embed, then atomically upsert the listing row and clear any tombstone.

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
  "data_source_key": "directorist:businesses",
  "source_id": "1514",
  "content_hash": "sha256-hex",
  "indexed_at": "2026-07-14T10:30:00Z",
  "changed": true,
  "embedding": {
    "reused": false
  }
}
```

`status` is `indexed` when a new vector is stored, `updated` when the row changes while its existing vector is reused, and `unchanged` when no database or embedding write is needed. The response uses public source identity rather than exposing an internal table key. Review and WordPress-post upserts use the same response shape.

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

Indexes multiple records. `MAX_CONTENT_BATCH_ITEMS` defaults to `50` and accepts 1 through 200. A
batch may contain multiple source kinds; the service validates and upserts each item through its
matching repository and vector-storage contract. The envelope must contain only a non-empty `items`
array within the limit. Empty/oversized/malformed envelopes return `422 validation_error` with
`field=items` before validating, hashing, embedding, or writing any item.

Valid batches process items independently and sequentially in request order. Each item keeps its own
transaction; one item failure never rolls back or hides another success. The HTTP response is `200`
with `ok: true` whenever the batch envelope was accepted, even if individual items fail. Unexpected
failures use the safe per-item code `indexing_error`; provider bodies, content, and vectors are never
included. Clients retry only failed items and may safely retry the whole batch because source identity
plus deterministic content hash is the idempotency boundary.

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
  "updated": 0,
  "unchanged": 0,
  "failed": 0,
  "results": [
    {
      "index": 0,
      "ok": true,
      "status": "indexed",
      "data_source_key": "directorist:events",
      "source_id": "2001",
      "content_hash": "sha256-hex",
      "changed": true,
      "embedding": {"reused": false}
    }
  ],
  "errors": []
}
```

Each `errors` member is also present in `results` at its request index and has
`{"index": 1, "ok": false, "status": "failed", "data_source_key": "...", "source_id":
"...", "error": {"code": "...", "message": "...", "field": "...", "retryable": true}}`.
`field` is omitted when not applicable. `retryable` is true only for transient dependency/server
failures, including `embedding_unavailable`.

### `POST /content/delete`

Removes content from retrieval. `data_source_key` resolves the source kind and matching table. Prefer
tombstoning over hard deletion. The request object contains only a canonical `data_source_key` and a
non-empty `source_id` of at most 128 characters; Directorist listing IDs must parse losslessly to a
positive safe integer. A missing source or item is an idempotent success with `deleted_items: 0` and
does not create a placeholder. A present active item is tombstoned with `deleted_items: 1`; repeating
the delete returns `0`. Listing rows set `deleted_at`; review and WordPress rows set `status=deleted`.
Vectors may remain stored but every retrieval path filters the owning tombstone/status first.

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
  "status": "deleted",
  "deleted_items": 1
}
```

### `POST /content/delete-by-data-source`

Tombstones every active record for a key in its source-kind table. WordPress calls this only for an
explicit admin **Delete all indexed data** action or an equivalent deliberate maintenance operation.
Disabling an optional WordPress source must not call this route. A missing key is an idempotent
success with zero items. The operation updates only the resolved content table in one transaction and
never inserts, deletes, or updates `installation_config.allowed_data_source_keys`.

Request:

```json
{
  "data_source_key": "wordpress:post"
}
```

All four content routes require an active installation credential with `content:write`. The existing
correlation middleware accepts or creates `X-Correlation-ID`, echoes it in the response header, and
associates logs with it; content bodies are never logged. No `Idempotency-Key` is required or stored:
single/bulk upserts use source identity plus content hash, and deletes use current tombstone state.
Malformed JSON/auth failures use the common error envelope. Accepted destructive requests do not
return `404`, which makes interrupted WordPress retries converge safely.

Each single upsert already records one `content_index` usage event. A bulk request additionally
records one non-durable `content_bulk_index` event with total latency and safe counts
`items/indexed/updated/unchanged/failed`; it contains no identities or content. Single/source deletion
records `content_delete` or `content_delete_source` with latency, success, safe source kind, and
`deleted_items`. Observability-write failure is logged safely and never changes the durable result.

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
  "channel": "web",
  "message": "What workshops are available this Saturday?",
  "context": {
    "timezone": "UTC",
    "page_url": "https://example.com"
  }
}
```

The chat caller does not provide `allowed_data_source_keys`. The backend loads its persisted allowlist for every chat turn. Retrieval and tool calls may narrow that list for the user's context but must never search outside it.

`channel` accepts `web`, `mobile`, or `admin_test`. WordPress sends `web` for the public widget and `admin_test` only from its capability-protected Test Chat route. Channel is product context, not an AI-provider selector.

The chat caller also cannot choose the AI provider or model. The server uses `AI_PROVIDER` and the selected provider's environment configuration for the entire turn.

SV-US-008 adds no public retrieval endpoint. `search_content` and `get_content_detail` are
server-owned application/tool boundaries used by the later chat workflow. Their validated filter
grammar, allowlist intersection, bounded candidate/result shape, review-parent handling, normalized
fusion score, detail exclusions, and vector-only fallback are normative in
[`HYBRID_SEARCH_PLAN.md`](HYBRID_SEARCH_PLAN.md). Neither boundary accepts raw SQL, a provider/model,
an embedding, or caller authority to expand the persisted allowlist.

SV-US-009 likewise adds no public endpoint. Its internal `rank_and_cite` boundary consumes only the
allowlist-scoped candidates above and follows
[`RANKING_AND_CITATION_CONTRACT.md`](RANKING_AND_CITATION_CONTRACT.md). The later chat route exposes
its recommendation, citation, disclosure, and uncertainty fields without accepting caller-supplied
source URLs or authority.

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
      "is_featured": false,
      "disclosures": [],
      "volatile_details": [],
      "uncertainties": [],
      "ranking": {
        "policy_version": "ask-sunny-ranking-v1",
        "relevance_score": 0.91,
        "quality_score": 0.4,
        "promotion_score": 0
      }
    }
  ],
  "citations": [
    {
      "title": "Community Workshop",
      "url": "https://example.com/events/community-workshop",
      "source_kind": "directorist_listing",
      "data_source_key": "directorist:events",
      "data_source_label": "Event Directory",
      "source_id": "2001",
      "citation_id": "citation-1",
      "claim_ids": ["claim-1"],
      "evidence_role": "primary"
    }
  ],
  "follow_up_questions": []
}
```

### `GET /conversations/:id`

Returns bounded recent history for the exact conversation owner. Requires an active installation
credential with `conversations:read` plus exactly one of
`X-Ask-Sunny-External-User-ID` or `X-Ask-Sunny-Anonymous-Session-ID`. Identity validation,
non-disclosing lookup, history bounds, and privacy behavior follow
[`CONVERSATION_CONTEXT_CONTRACT.md`](CONVERSATION_CONTEXT_CONTRACT.md).

Response:

```json
{
  "conversation": {
    "id": "uuid",
    "title": "Saturday plans",
    "status": "active",
    "summary": "The visitor wants a downtown activity this Saturday.",
    "created_at": "2026-07-06T12:00:00Z",
    "updated_at": "2026-07-06T12:05:00Z"
  },
  "messages": [
    {
      "id": "uuid",
      "role": "user",
      "content": "What can we do this Saturday?",
      "citations": [],
      "created_at": "2026-07-06T12:00:00Z"
    }
  ],
  "history_truncated": false
}
```

Missing, mismatched, deleted, and malformed conversation IDs share:

```json
{
  "error": {
    "code": "conversation_not_found",
    "message": "The conversation was not found."
  }
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
  "paradedb": {
    "pg_search": "ok",
    "vector": "ok",
    "bm25_indexes": "ok",
    "hybrid_search": {
      "requested": true,
      "effective": true,
      "status": "enabled",
      "reason": null
    },
    "package_compatibility": {
      "status": "verified",
      "postgresql_major": 18,
      "os_id": "ubuntu",
      "os_release": "noble",
      "architecture": "amd64",
      "package_version": "pinned-release",
      "extension_version": "installed-release",
      "image_digest": null,
      "verified_at": "2026-07-13T10:00:00Z"
    }
  },
  "redis": "disabled",
  "ai": {
    "provider": "openai",
    "configured": true,
    "model": "configured-model-name",
    "embedding_provider": "openai",
    "embedding_model": "text-embedding-3-small"
  },
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
- `channel` must be an allowed product channel. `admin_test` is accepted only from the trusted WordPress installation or an authorized server administrator.
- Chat requests must reject or ignore any caller-supplied `allowed_data_source_keys`; only the persisted backend list is authoritative.
- `source_kind` must be `directorist_listing`, `directorist_review`, or `wordpress_post`.
- The backend must route each source kind only to its matching persistence boundary; it must never store unlike kinds in a shared generic content table. Listing vectors live on `listings`, while review and WordPress-content vectors use their respective embedding tables.
- `data_source_key`, `data_source_label`, and `source_context` must be valid and internally consistent. The backend upserts source metadata from the received content rather than requiring a pre-registered or enabled source.
- `source_id`, `source_url`, and `title` are required for active content.
- A Directorist listing `source_id` must parse losslessly to a positive safe integer and is stored as `listings.id`; nonnumeric listing IDs are rejected with a field-specific validation error.
- Directorist `directory_type_id` and a review's `parent_source_id` must also parse losslessly to positive safe integers before database lookup or storage.
- Core Directorist preset fields are normalized into `normalized_metadata`; only measured, frequently ranked values such as `is_featured` receive dedicated listing columns.
- Every non-core field uses the flat `listing_metadata` map, whether it is a Directorist custom field or an extension/third-party preset field. Do not create separate namespaces.
- Listing `raw_payload` is sanitized and stored as `raw_metadata`; the canonical normalizer output, including the flat `listing_metadata` map, is stored as `normalized_metadata`.
- Listings whose directory type supports extension-provided event fields include those typed values under `listing_metadata`; they remain `directorist_listing` records.
- Each `listing_metadata` entry uses its stable field key and includes a label, field type, and sanitized typed value. Provider identity is optional metadata when available.
- Apply configured limits to metadata field count, key/label length, array size, and serialized payload size. Reject executable values, unsafe URLs, and unexpected nested objects.
- Active `directorist_review` records require `parent_data_source_key` and `parent_source_id`; the parent key must identify the listing source for the same directory type.
- Review upsert resolves its parent to the composite `listings(data_source_id, id)` key; if the listing is not indexed yet, return `409 parent_listing_missing` so WordPress can index the listing first and retry the review.
- Only approved public reviews are active. Unapproved, spammed, trashed, or deleted reviews are tombstoned.
- Deleted content requires only `data_source_key` and `source_id`.
- WordPress applies indexing filters before sending content and synchronizes source allowance separately. Every backend candidate query, vector search, detail lookup used by RAG, and model tool call must constrain results to the stored allowlist.
- Chat routes must never accept raw SQL, arbitrary tool names, or model overrides from clients.
- Chat routes must reject or ignore caller-supplied `ai_provider`, provider API keys, base URLs, and model names; only environment configuration is authoritative.
- Hybrid retrieval must constrain both BM25 and vector candidates to persisted allowed data-source keys and active records before fusion.

### Content And Metadata Safety Limits

The server applies these defaults before any source registration, content write, hash, or embedding
call. Deployments may lower them but must not exceed the documented environment-schema bounds.

| Setting | Default | Maximum accepted configuration | Applies to |
|---|---:|---:|---|
| `MAX_CONTENT_ITEM_BYTES` | 524288 | 2097152 | UTF-8 JSON serialization of one complete content item |
| `MAX_METADATA_FIELDS` | 100 | 500 | Entries in each metadata/taxonomy/context map |
| `MAX_METADATA_KEY_CHARS` | 64 | 128 | Stable metadata, taxonomy, and context keys |
| `MAX_METADATA_LABEL_CHARS` | 120 | 240 | Public metadata labels |
| `MAX_METADATA_STRING_CHARS` | 2000 | 10000 | One metadata/context/taxonomy string value |
| `MAX_METADATA_ARRAY_ITEMS` | 50 | 200 | One public metadata/context/taxonomy array |
| `MAX_METADATA_NESTING_DEPTH` | 4 | 8 | Sanitized `raw_payload`; typed metadata maps remain flatter |
| `MAX_CONTENT_BATCH_ITEMS` | 50 | 200 | Items accepted by one bulk-upsert envelope before any processing |

Stable metadata/context keys match `[a-z0-9][a-z0-9_-]*`. Keys beginning with `_` or the reserved
private prefixes `admin_`, `private_`, `secret_`, `token_`, `password_`, `payment_`, and `billing_`
are rejected. Objects use ordinary prototypes, own enumerable properties, and no executable values.
Strings containing script/iframe/object/embed markup, inline event-handler syntax, `javascript:`,
`vbscript:`, or executable `data:` URLs are rejected rather than sanitized silently.

Public URLs are absolute `http` or `https` URLs without embedded credentials. `url` and `file`
metadata field types contain one such URL or a bounded array of them. `listing_metadata` is a flat
map whose values contain only `label`, `field_type`, optional `provider`, and `value`; its `value` is
null, a finite number, a boolean, one bounded string, or a bounded array of those primitives.
`taxonomies`, `post_metadata`, `review_metadata`, and `source_context` are flat key maps with primitive
or primitive-array values. `raw_payload` may retain sanitized public JSON up to the configured depth;
unexpected deeper objects are rejected. Private fields are rejected even if the total payload is
otherwise valid.

`data_source_label`, source URLs, source IDs, title/status, directory identity, review parent fields,
and WordPress post type are validated before metadata. Active content requires an absolute safe
source URL and non-empty title. Directorist listing IDs/directory IDs and review parent listing IDs
must parse losslessly to positive JavaScript-safe integers.

The source-kind validator returns a bounded public payload and retrieval-only source descriptor. The
source-specific read-model repository accepts a prepared storage record. SV-US-006 supplies
deterministic `search_document`, `embedding_text`, `content_hash`, and vectors; SV-US-005 does not use
placeholder hashes to satisfy non-null columns.
