# WordPress Plugin Data Schema

The WordPress plugin stores only integration settings, indexing state, and frontend session metadata. The backend stores chat history, embeddings, usage, and long-term personalization records.

## Options

Use one option array to reduce option sprawl:

```php
ai_sunny_settings = [
    'enabled' => true,
    'api_base_url' => 'https://api.example.com',
    'api_key' => 'ais_live_xxxxx',
    'api_key_prefix' => 'ais_live',
    'widget_enabled' => true,
    'widget_position' => 'bottom_right',
    'widget_theme' => 'light',
    'indexing_enabled' => true,
    'index_business_listings' => true,
    'index_event_listings' => true,
    'index_reviews' => true,
    'index_weekend_picks' => true,
    'request_timeout' => 20,
    'debug_logging' => false,
    'last_successful_sync_at' => '',
    'last_diagnostics' => [],
];
```

Recommended additional options:

```php
ai_sunny_schema_version = '1'
ai_sunny_reindex_state = [
    'running' => false,
    'cursor' => 0,
    'total' => 0,
    'processed' => 0,
    'failed' => 0,
    'started_at' => '',
    'finished_at' => '',
]
```

Secrets:

- `api_key` must not be autoloaded if possible.
- `api_key` must never be returned by frontend REST endpoints.
- Provisioning key should come from a server constant/env, not a browser-visible setting.

## Post Meta

For every indexed source post:

```text
_ai_sunny_indexed_at
_ai_sunny_index_status
_ai_sunny_index_error
_ai_sunny_content_hash
_ai_sunny_backend_content_id
_ai_sunny_last_payload
```

Status values:

```text
pending
indexed
unchanged
deleted
failed
skipped
```

`_ai_sunny_last_payload` is optional and should be disabled or capped in production to avoid storing large duplicated content.

## User Meta

Reserved for future personalization:

```text
_ai_sunny_external_user_id
_ai_sunny_preferred_location
_ai_sunny_children
_ai_sunny_interests
_ai_sunny_budget_preference
_ai_sunny_travel_distance_miles
_ai_sunny_favorite_source_ids
```

For v1, avoid duplicating full conversation history in WordPress. Store only the mapping needed to resume a backend conversation:

```text
_ai_sunny_recent_conversation_ids
```

## Transients

Use transients for short-lived operational state:

```text
ai_sunny_rate_limit_{session_hash}
ai_sunny_diagnostics_cache
ai_sunny_reindex_lock
ai_sunny_chat_session_{session_hash}
```

Suggested TTLs:

- Rate limit: 1 to 15 minutes.
- Diagnostics cache: 5 minutes.
- Reindex lock: 15 minutes.
- Anonymous chat session: 30 days.

## Directorist Mapping

Business listings:

```php
[
    'source_type' => 'business_listing',
    'source_id' => (string) $post_id,
    'source_url' => get_permalink($post_id),
    'title' => $listing_title,
    'summary' => $tagline_or_excerpt,
    'body' => $content,
    'status' => $post_status === 'publish' ? 'active' : 'deleted',
    'directory_type' => $directory_type_name,
    'categories' => $category_names,
    'locations' => $location_names,
    'tags' => $tag_names,
    'amenities' => $amenity_values,
    'age_min' => $age_min,
    'age_max' => $age_max,
    'price_level' => $price_level,
    'is_featured' => $featured,
    'is_sponsored' => $sponsored,
    'raw_payload' => $raw_meta,
]
```

Event listings:

```php
[
    'source_type' => 'event_listing',
    'source_id' => (string) $post_id,
    'starts_at' => $event_start_iso,
    'ends_at' => $event_end_iso,
    'locations' => $location_names,
    'age_min' => $age_min,
    'age_max' => $age_max,
]
```

Weekend Picks:

```php
[
    'source_type' => 'weekend_pick',
    'source_id' => (string) $post_id,
    'title' => get_the_title($post_id),
    'body' => wp_strip_all_tags($post->post_content),
    'source_url' => get_permalink($post_id),
]
```

Reviews:

```php
[
    'source_type' => 'review',
    'source_id' => (string) $comment_id,
    'source_url' => get_permalink($listing_id) . '#comment-' . $comment_id,
    'title' => 'Review for ' . get_the_title($listing_id),
    'body' => $comment_text,
    'raw_payload' => [
        'rating' => $rating,
        'listing_id' => $listing_id,
    ],
]
```

## Browser Local Storage

The widget may store only non-secret UI/session state:

```text
ai_sunny_anonymous_session_id
ai_sunny_recent_conversation_id
ai_sunny_widget_open
```

Do not store backend API keys, OpenAI keys, raw admin settings, or private user profile data in browser storage.

