# WordPress Plugin Data Schema

The WordPress plugin stores only integration settings, indexing state, and frontend session metadata. The backend stores chat history, embeddings, usage, and long-term personalization records.

## Options

Use one option array to reduce option sprawl:

```php
ask_sunny_settings = [
    'enabled' => true,
    'api_base_url' => 'https://api.example.com',
    'api_key' => 'ask_live_xxxxx',
    'api_key_prefix' => 'ask_live',
    'widget_enabled' => true,
    'widget_position' => 'bottom_right',
    'widget_theme' => 'light',
    'indexing_enabled' => true,
    // Directorist listing and review sources are discovered and always indexed.
    'wordpress_post_type_sources' => [
        'post' => [
            'enabled' => true,
            'label' => 'Blog',
            'description' => 'Published articles and guides.',
            'context_metadata' => [
                'content_kind' => 'article',
                'audience' => 'public',
            ],
            'filters' => [
                'taxonomies' => [
                    'category' => [
                        'operator' => 'IN',
                        'term_ids' => [12, 18],
                    ],
                ],
                'meta' => [],
            ],
        ],
        'page' => [
            'enabled' => false,
            'label' => 'Pages',
            'description' => '',
            'context_metadata' => [],
            'filters' => [
                'taxonomies' => [],
                'meta' => [],
            ],
        ],
    ],
    'request_timeout' => 60,
    'debug_logging' => false,
    'allowed_data_sources_version' => 0,
    'allowed_data_sources_synced_at' => '',
    'last_successful_sync_at' => '',
    'last_diagnostics' => [],
];
```

Recommended additional options:

```php
ask_sunny_schema_version = '1'
ask_sunny_reindex_state = [
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

`wordpress_post_type_sources` stores configuration only for non-Directorist post types. The settings API must reject an attempt to add the Directorist listing post type here or disable a discovered Directorist directory type. Only public, readable post types should be offered. Internal types such as revisions, navigation items, attachment internals, and plugin storage records must not be indexable.

Taxonomy filters use term IDs for stability and are evaluated before queuing an item. `IN` means at least one selected term must match; `AND` means all selected terms must match. Post-meta filters are limited to keys and comparison operators explicitly allowlisted by the plugin. Changing a source filter must tombstone items that no longer match and enqueue newly eligible items.

## Post Meta

For every indexed source post:

```text
_ask_sunny_indexed_at
_ask_sunny_index_status
_ask_sunny_index_error
_ask_sunny_content_hash
_ask_sunny_backend_content_id
_ask_sunny_data_source_key
_ask_sunny_last_payload
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

The Data Sources item API computes `not_indexed` when no indexing meta exists and `ineligible` when an item does not match its current indexing filter. Source enablement does not overwrite index status: an item may remain `indexed` while its separate retrieval status is `excluded_source_disabled`. Retrieval status values are computed as `included`, `excluded_source_disabled`, or `excluded_by_filter` and do not need to be stored in post meta.

`_ask_sunny_last_payload` is optional and should be disabled or capped in production to avoid storing large duplicated content.

## Comment Meta

Store the same indexing fields on each Directorist review comment, using comment meta:

```text
_ask_sunny_indexed_at
_ask_sunny_index_status
_ask_sunny_index_error
_ask_sunny_content_hash
_ask_sunny_backend_content_id
_ask_sunny_data_source_key
```

Only approved public Directorist reviews are active index records. Pending, spammed, trashed, or deleted reviews must be tombstoned if they were indexed previously.

## User Meta

Reserved for future personalization:

```text
_ask_sunny_external_user_id
_ask_sunny_preferred_location
_ask_sunny_custom_preferences
_ask_sunny_interests
_ask_sunny_budget_preference
_ask_sunny_travel_distance_miles
_ask_sunny_favorite_source_ids
```

For v1, avoid duplicating full conversation history in WordPress. Store only the mapping needed to resume a backend conversation:

```text
_ask_sunny_recent_conversation_ids
```

## Transients

Use transients for short-lived operational state:

```text
ask_sunny_rate_limit_{session_hash}
ask_sunny_diagnostics_cache
ask_sunny_reindex_lock
ask_sunny_chat_session_{session_hash}
```

Suggested TTLs:

- Rate limit: 1 to 15 minutes.
- Diagnostics cache: 5 minutes.
- Reindex lock: 15 minutes.
- Anonymous chat session: 30 days.

## Data Source Registry

The plugin derives and stores the local registry whenever Directorist directory types or source settings change:

```php
[
    [
        'data_source_key' => 'directorist:events',
        'source_kind' => 'directorist_listing',
        'label' => 'Event Directory',
        'wp_post_type' => 'at_biz_dir',
        'directory_type_id' => 42,
        'directory_type_slug' => 'events',
        'required' => true,
        'enabled' => true,
        'context_metadata' => [
            'content_kind' => 'event',
            'supports_event_dates' => true,
        ],
        'filters' => [],
    ],
    [
        'data_source_key' => 'directorist:events:reviews',
        'source_kind' => 'directorist_review',
        'label' => 'Event Directory Reviews',
        'wp_post_type' => 'at_biz_dir',
        'directory_type_id' => 42,
        'directory_type_slug' => 'events',
        'parent_data_source_key' => 'directorist:events',
        'required' => true,
        'enabled' => true,
        'context_metadata' => [
            'content_kind' => 'review',
            'reviewed_content_kind' => 'event',
        ],
        'filters' => [
            'comment_approved' => '1',
        ],
    ],
    [
        'data_source_key' => 'wordpress:post',
        'source_kind' => 'wordpress_post',
        'label' => 'Blog',
        'wp_post_type' => 'post',
        'required' => false,
        'enabled' => true,
        'context_metadata' => [
            'content_kind' => 'article',
            'audience' => 'public',
        ],
        'filters' => [
            'taxonomies' => [
                'category' => [
                    'operator' => 'IN',
                    'term_ids' => [12, 18],
                ],
            ],
        ],
    ],
]
```

`data_source_key` is a stable machine key, not the editable label. If a Directorist directory type is renamed, its key should continue to use a stable directory type ID or immutable slug mapping. A directory type removed from Directorist should be marked inactive and its backend items tombstoned.

`required`, `enabled`, and `filters` remain WordPress-owned configuration. WordPress derives the complete `allowed_data_source_keys` array from these settings and persists that derived list in the backend through `PUT /retrieval/allowed-data-sources`. Content payloads include only the selected source's identity, label, and RAG context metadata. Disabling an optional source preserves its backend items, stops automatic indexing, synchronizes an allowlist without its key, and leaves the local index status unchanged. Enabling it adds the key back and triggers reconciliation for locally eligible items.

`allowed_data_sources_version` stores the last backend version used for optimistic concurrency. `allowed_data_sources_synced_at` supports diagnostics. A source-setting change must not report success until the backend allowlist update succeeds; if it fails, retain the previous local setting and show an actionable error.

Disabling a source never triggers backend deletion. The admin may explicitly delete one item or use **Delete all indexed data** for a source; those actions set the affected local index states to `deleted` without changing the source's `enabled` setting. Normal content lifecycle reconciliation may still tombstone an item that is deleted, unpublished, unapproved, or no longer matches its configured indexing filter.

## Content Mapping

Before building a listing payload, classify every active Directory Builder field into one of two storage paths:

- `core_preset`: a preset shipped by Directorist core. Map it to the canonical top-level payload field backed by a database column.
- `listing_metadata`: every non-core field, including Directorist custom fields and presets registered by extensions or third-party integrations. Store all of them in the same flat metadata map.

Use Directorist's core field registry and stable field key to identify core presets; do not infer core status from an editable label. Any field not recognized as a core preset goes into `listing_metadata`. Exclude admin-only/private fields. Preserve scalar, array, boolean, numeric, date/time, and URL types after sanitization.

Core preset columns cover title, description, tagline, excerpt/summary, price, categories, locations, tags, view count, phone, ZIP/postal code, email, fax, social information, website, address/map coordinates, images, amenities when supplied by core, and featured state.

Directory listings:

```php
[
    'source_kind' => 'directorist_listing',
    'data_source_key' => 'directorist:' . $directory_type_slug,
    'data_source_label' => $directory_type_name,
    'source_context' => $directory_type_context_metadata,
    'source_id' => (string) $post_id,
    'source_url' => get_permalink($post_id),
    'title' => $listing_title,
    'summary' => $tagline_or_excerpt,
    'tagline' => $tagline,
    'body' => $content,
    'status' => $post_status === 'publish' ? 'active' : 'deleted',
    'wordpress_post_type' => get_post_type($post_id),
    'directory_type_id' => (string) $directory_type_id,
    'directory_type_slug' => $directory_type_slug,
    'directory_type_label' => $directory_type_name,
    'categories' => $category_names,
    'locations' => $location_names,
    'tags' => $tag_names,
    'amenities' => $amenity_values,
    'price_amount' => $price_amount,
    'price_currency' => $price_currency,
    'price_display' => $price_display,
    'view_count' => $view_count,
    'phone' => $phone,
    'zip_postal_code' => $zip_postal_code,
    'public_email' => $public_email,
    'fax' => $fax,
    'website_url' => $website_url,
    'social_links' => $social_links,
    'address' => $address,
    'latitude' => $latitude,
    'longitude' => $longitude,
    'featured_image_url' => $featured_image_url,
    'image_urls' => $image_urls,
    'is_featured' => $featured,
    'listing_metadata' => $generic_listing_metadata,
    'raw_payload' => $raw_meta,
]
```

An Event Directory listing uses the same payload. Event fields registered by an extension use the same generic listing metadata map as custom fields:

```php
[
    'source_kind' => 'directorist_listing',
    'data_source_key' => 'directorist:events',
    'source_id' => (string) $post_id,
    'listing_metadata' => [
        'event_start' => [
            'label' => 'Event Start',
            'provider' => 'directorist-events',
            'field_type' => 'datetime',
            'value' => $event_start_iso,
        ],
        'event_end' => [
            'label' => 'Event End',
            'provider' => 'directorist-events',
            'field_type' => 'datetime',
            'value' => $event_end_iso,
        ],
    ],
]
```

Enabled WordPress post types:

```php
[
    'source_kind' => 'wordpress_post',
    'data_source_key' => 'wordpress:' . $post->post_type,
    'data_source_label' => $source_config['label'],
    'source_context' => $source_config['context_metadata'],
    'source_id' => (string) $post_id,
    'wordpress_post_type' => $post->post_type,
    'title' => get_the_title($post_id),
    'summary' => wp_strip_all_tags(get_the_excerpt($post_id)),
    'body' => wp_strip_all_tags($post->post_content),
    'source_url' => get_permalink($post_id),
    'status' => $post->post_status === 'publish' ? 'active' : 'deleted',
    'taxonomies' => $configured_public_taxonomy_terms,
    'post_metadata' => $configured_public_post_metadata,
]
```

Directorist listing reviews:

```php
[
    'source_kind' => 'directorist_review',
    'data_source_key' => 'directorist:' . $directory_type_slug . ':reviews',
    'data_source_label' => $directory_type_name . ' Reviews',
    'source_context' => [
        'content_kind' => 'review',
        'reviewed_content_kind' => $directory_content_kind,
    ],
    'source_id' => (string) $comment_id,
    'source_url' => get_permalink($listing_id) . '#comment-' . $comment_id,
    'title' => 'Review for ' . get_the_title($listing_id),
    'body' => wp_strip_all_tags($comment->comment_content),
    'status' => $comment->comment_approved === '1' ? 'active' : 'deleted',
    'wordpress_post_type' => get_post_type($listing_id),
    'directory_type_id' => (string) $directory_type_id,
    'directory_type_slug' => $directory_type_slug,
    'directory_type_label' => $directory_type_name,
    'parent_data_source_key' => 'directorist:' . $directory_type_slug,
    'parent_source_id' => (string) $listing_id,
    'categories' => $parent_category_names,
    'locations' => $parent_location_names,
    'rating_value' => $rating,
    'listing_metadata' => [],
]
```

Review content must remain separate from the listing body and embedding. The parent relationship allows ranking to aggregate review signals for a listing without duplicating review text into the listing record.

## Browser Local Storage

The widget may store only non-secret UI/session state:

```text
ask_sunny_anonymous_session_id
ask_sunny_recent_conversation_id
ask_sunny_widget_open
```

Do not store backend API keys, OpenAI keys, raw admin settings, or private user profile data in browser storage.
