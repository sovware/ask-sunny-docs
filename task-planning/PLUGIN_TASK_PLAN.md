# Ask Sunny WordPress Plugin User Story Plan

## Product Goal

Deliver a production-ready WpMVC WordPress plugin that presents aggregated Listings and Listing Reviews data-source tabs, supports globally optional reviews and optional WordPress content, securely synchronizes eligible content with the Ask Sunny backend, gives administrators indexing/widget controls and a Test Chat workspace, and provides visitors with an accessible multi-turn chat widget.

This plan is self-contained. It defines all plugin contract, implementation, test, security, accessibility, and release work required for the WordPress plugin release.

## Story Standard

Every story uses the standard format:

> As a **persona**, I want **a capability**, so that **I receive a measurable benefit**.

Acceptance criteria use Given–When–Then statements. A story is complete only when all acceptance criteria pass and every required task is checked off.

## Definition of Done

- All story acceptance criteria pass in automated or documented acceptance tests.
- PHP and JavaScript code is reviewed, formatted, linted, statically analyzed, and covered by appropriate unit, integration, REST, and browser tests.
- WordPress capabilities, nonces, sanitization, validation, and escaping are applied at every boundary.
- Backend, OpenAI, Groq, and embedding-provider credentials never appear in browser source, localized data, REST responses, or normal logs.
- Admin and frontend interfaces meet the supported browser, responsive-layout, keyboard, focus, semantic markup, contrast, and reduced-motion requirements.
- Plugin activation, deactivation, upgrade, and uninstall behavior is tested and documented.
- Relevant architecture, data schema, REST API, setup, and troubleshooting documentation is current.

## Epic 1 — Plugin Foundation and Setup

### WP-US-001 — Activate a compatible plugin safely

**User story**

> As a **WordPress administrator**, I want Ask Sunny to activate only in a supported environment, so that it does not destabilize my site or lose data during lifecycle changes.

**Acceptance criteria**

1. **Given** supported PHP, WordPress, WpMVC, and Directorist versions, **when** the plugin activates, **then** its providers, routes, migrations, and assets initialize once without warnings.
2. **Given** a missing or unsupported required dependency, **when** activation or boot occurs, **then** an actionable admin notice is displayed and incompatible features do not run.
3. **Given** the plugin is deactivated, **when** shutdown completes, **then** scheduled work is safely paused or cleared and user configuration/index status is preserved.
4. **Given** an explicit uninstall, **when** the documented data-retention choice is applied, **then** only the intended plugin-owned options, metadata, transients, or tables are removed.
5. **Given** an admin or public page unrelated to Ask Sunny, **when** it renders, **then** unnecessary plugin assets are not enqueued.

**Tasks**

- [ ] Confirm supported PHP, WordPress, WpMVC, Directorist, Node, and browser versions.
- [ ] Create the thin plugin bootstrap and WpMVC `config/app.php`.
- [ ] Scaffold providers, controllers, middleware, repositories, services, DTOs, routes, views, enqueues, and migrations.
- [ ] Add activation, deactivation, upgrade, and uninstall handlers.
- [ ] Add dependency/version checks and admin notices.
- [ ] Configure scoped admin and frontend asset builds.
- [ ] Add PHP and JavaScript formatting, linting, static analysis, unit-test, and build commands.
- [ ] Add lifecycle and compatibility tests.

**Dependencies:** none  
**Priority:** Must have

### WP-US-002 — Configure and provision the backend connection

**User story**

> As a **WordPress administrator**, I want to configure and provision the backend from WordPress, so that I can connect the site without placing secrets in browser code.

**Acceptance criteria**

1. **Given** an authorized administrator, **when** settings are saved, **then** validated values persist with documented defaults and secrets remain server-side.
2. **Given** a valid backend URL and provisioning credential, **when** provisioning succeeds, **then** the returned installation key is stored securely and connection health is displayed.
3. **Given** provisioning or rotation fails, **when** the error is returned, **then** the last known working key and settings remain intact and an actionable error is shown.
4. **Given** an unauthorized user or invalid nonce, **when** a settings or provisioning action is attempted, **then** the request is rejected and nothing changes.
5. **Given** browser source, localized data, network responses, or normal logs, **when** they are inspected, **then** neither provisioning nor installation credentials are present.

**Tasks**

- [ ] Implement documented settings options, defaults, validation, and sanitization.
- [ ] Implement a server-side API client with correlation IDs, timeouts, stable errors, and safe retry rules.
- [ ] Implement admin settings and provisioning REST controllers.
- [ ] Implement setup, provisioning, rotation, and connection-health views.
- [ ] Store a new installation key only after a successful response.
- [ ] Add capability, nonce, URL, response-sanitization, and secret-redaction controls.
- [ ] Add settings, provisioning, rotation, failure-preservation, and secret-exposure tests.

**Dependencies:** WP-US-001  
**Priority:** Must have

## Epic 2 — Data-Source Administration

### WP-US-003 — Discover and configure retrievable data sources

**User story**

> As a **WordPress administrator**, I want to see and configure every eligible content source, so that I control which optional content can participate in Ask Sunny answers.

**Acceptance criteria**

1. **Given** one or more Directorist directory types, **when** the registry refreshes, **then** each type has stable mandatory listing and classified review keys while the admin UI presents only aggregate Listings and Listing Reviews tabs.
2. **Given** a Directorist listing source, **when** the administrator views or edits it, **then** it cannot be disabled or excluded from launch indexing.
3. **Given** the global Listing Reviews setting changes, **when** synchronization succeeds, **then** every discovered review key is added to or removed from the backend allowlist atomically; no per-directory review control exists.
4. **Given** eligible public non-Directorist post types, **when** the registry refreshes, **then** each can be enabled with status, category, tag, and approved-meta indexing filters supported by its registered taxonomies.
5. **Given** a source label changes, **when** discovery runs again, **then** its stable machine key and existing status remain unchanged.
6. **Given** an optional source is enabled or disabled, **when** backend synchronization succeeds, **then** the displayed local state and stored retrieval-configuration version match the backend result.
7. **Given** synchronization fails or conflicts, **when** the action completes, **then** WordPress does not display an unsynchronized success state and provides a retry path.

**Tasks**

- [ ] Implement Directorist directory-type discovery.
- [ ] Generate immutable listing and review keys and context metadata for backend classification.
- [ ] Persist and validate one global `listing_reviews_enabled` setting; do not create per-directory review toggles.
- [ ] Add every review key to or remove every review key from the allowlist in one versioned update.
- [ ] Discover eligible public WordPress post types while excluding Directorist and unsafe types.
- [ ] Implement the local source registry, enabled state, descriptions, filters, counts, and version state.
- [ ] Implement status, category, tag, and controlled post-meta filter configuration for optional post types.
- [ ] Compute and synchronize the complete allowed-source list after provisioning and every configuration change.
- [ ] Implement optimistic-version conflict and retry handling.
- [ ] Add discovery, stable-key, mandatory-source, optional-source, filter, and synchronization tests.

**Dependencies:** WP-US-002  
**Priority:** Must have

## Epic 3 — Content Normalization

### WP-US-004 — Normalize Directorist listings

**User story**

> As a **site visitor**, I want all public listing details represented accurately in the search index, so that recommendations can match categories, locations, amenities, prices, dates, and custom requirements.

**Acceptance criteria**

1. **Given** an eligible published listing, **when** it is normalized, **then** all documented core Directorist preset values map to their canonical top-level fields.
2. **Given** a public custom or extension-provided field, **when** it is normalized, **then** it appears in the flat `listing_metadata` map with a stable key, label, field type, typed value, and optional provider.
3. **Given** HTML, media, coordinates, dates, arrays, URLs, or taxonomy terms, **when** they are normalized, **then** values use deterministic public-safe representations and the installation timezone policy.
4. **Given** a private, administrative, payment, executable, empty, or operational field, **when** normalization runs, **then** the field is excluded.
5. **Given** identical public content, **when** normalization runs repeatedly, **then** the serialized payload remains byte-for-byte stable.

**Tasks**

- [ ] Finalize canonical listing fixtures and field mapping.
- [ ] Implement core preset extraction.
- [ ] Implement generic custom and extension field extraction into `listing_metadata`.
- [ ] Implement typed value, URL, HTML/text, media, coordinate, taxonomy, and timezone normalization.
- [ ] Apply field visibility, privacy, nesting, count, and size limits.
- [ ] Add deterministic ordering and canonical payload serialization.
- [ ] Add core, extension, private-field, invalid-value, and deterministic fixture tests.

**Dependencies:** WP-US-003  
**Priority:** Must have

### WP-US-005 — Normalize approved Directorist reviews

**User story**

> As a **site visitor**, I want approved review evidence associated with the correct listing, so that recommendations can reflect relevant public experiences and ratings.

**Acceptance criteria**

1. **Given** an approved public review, **when** it is normalized, **then** it contains the review body, rating, direct URL, and its parent listing's directory source, categories, and locations.
2. **Given** global Listing Reviews is enabled and a review is normalized, **when** its source key is generated, **then** it uses the classified companion key for the parent listing's directory type.
3. **Given** a review whose parent has not been indexed, **when** synchronization is attempted, **then** the parent listing is queued first and the review is retried.
4. **Given** an enabled or previously indexed review becomes unapproved, spammed, trashed, or deleted, **when** the corresponding hook runs, **then** a tombstone is queued.
5. **Given** private reviewer data, **when** normalization runs, **then** it is excluded from the payload and logs.
6. **Given** global Listing Reviews is disabled, **when** a review changes, **then** no upsert is queued; already indexed reviews are excluded through the synchronized allowlist.

**Tasks**

- [ ] Finalize canonical review and tombstone fixtures.
- [ ] Implement approved-review and rating extraction.
- [ ] Gate review normalization and hooks on the global Listing Reviews setting.
- [ ] Resolve directory source, parent listing ID, inherited categories, locations, and direct URL.
- [ ] Implement listing-first dependency and missing-parent retry handling.
- [ ] Register approval, edit, spam, trash, unapproval, and deletion hooks.
- [ ] Exclude private reviewer identity and administrative fields.
- [ ] Add approval-transition, parent, source-classification, privacy, and retry tests.

**Dependencies:** WP-US-004  
**Priority:** Must have

### WP-US-006 — Normalize optional WordPress content

**User story**

> As a **WordPress administrator**, I want only eligible content from enabled post types synchronized, so that editorial answers respect my source and filter configuration.

**Acceptance criteria**

1. **Given** an enabled public post type and an eligible published post, **when** normalization runs, **then** title, excerpt, cleaned body, URL, taxonomies, approved metadata, status, and source identity are included.
2. **Given** a disabled post type, **when** a post changes, **then** no automatic upsert is sent for that source.
3. **Given** taxonomy or approved-meta inclusion filters, **when** eligibility is evaluated, **then** only matching posts are active in the synchronized set.
4. **Given** a filter change, **when** reconciliation runs, **then** newly eligible posts are queued for upsert and newly ineligible posts are queued for tombstone.
5. **Given** private content, revisions, autosaves, attachments, non-public metadata, or unsafe values, **when** extraction runs, **then** they are excluded.

**Tasks**

- [ ] Finalize canonical WordPress-content and eligibility fixtures.
- [ ] Implement enabled post-type eligibility evaluation.
- [ ] Implement taxonomy and controlled post-meta filters without arbitrary SQL or callbacks.
- [ ] Implement title, excerpt, body, URL, taxonomy, approved-meta, status, and timestamp normalization.
- [ ] Implement filter-change reconciliation.
- [ ] Register publish, update, status-transition, trash, restore, and deletion hooks.
- [ ] Add disabled-source, filter, reconciliation, revision, privacy, and payload tests.

**Dependencies:** WP-US-003  
**Priority:** Must have

## Epic 4 — Index Synchronization

### WP-US-007 — Keep the backend index synchronized

**User story**

> As a **content administrator**, I want content changes synchronized automatically and retryably, so that Ask Sunny reflects current site content without manual maintenance.

**Acceptance criteria**

1. **Given** an eligible content create or update event, **when** its hook runs, **then** one deduplicated indexing job is queued with a correlation ID.
2. **Given** a normal unpublish or deletion event, **when** its hook runs, **then** the matching backend record is tombstoned.
3. **Given** an initial or forced reindex, **when** it runs, **then** eligible content is processed in bounded batches with checkpoints, progress, and resumability.
4. **Given** a transient backend or network failure, **when** a job fails, **then** it retries with bounded backoff without duplicating records or hiding the last error.
5. **Given** a mixed batch result, **when** processing completes, **then** successful, unchanged, and failed records each receive accurate local status.
6. **Given** an optional source is disabled, **when** configuration changes, **then** automatic indexing stops and retrieval allowance is removed without deleting retained backend rows.
7. **Given** a review event while global Listing Reviews is disabled and no prior record exists, **when** hooks run, **then** no indexing job is queued.

**Tasks**

- [ ] Select and configure the WordPress background-job mechanism.
- [ ] Implement deduplicated single-item upsert and tombstone jobs.
- [ ] Implement bounded bulk jobs, checkpoints, cancellation, resume, and force-reindex behavior.
- [ ] Define correlation, idempotency, timeout, retryable error, and backoff behavior.
- [ ] Persist per-item index state, content ID, attempt count, timestamps, and sanitized error metadata.
- [ ] Implement parent-listing-first review retry.
- [ ] Implement explicit single-item and source-wide delete commands separately from disabling.
- [ ] Add interruption, retry, duplicate-event, partial-batch, disable, deletion, and status tests.

**Dependencies:** WP-US-004, WP-US-005, WP-US-006  
**Priority:** Must have

## Epic 5 — Administration Experience

### WP-US-008 — Manage plugin settings and indexing through REST

**User story**

> As a **WordPress administrator**, I want secure REST endpoints for plugin operations, so that the admin interface can manage settings, sources, indexing, and diagnostics consistently.

**Acceptance criteria**

1. **Given** a user with `manage_options` and a valid nonce, **when** an admin route is called with valid input, **then** the documented response is returned.
2. **Given** a user without permission or an invalid nonce, **when** any admin route is called, **then** the action is rejected without leaking protected state.
3. **Given** a source-items request, **when** pagination, search, or status filters are supplied, **then** eligible, ineligible, indexed, pending, deleted, and failed items are represented accurately.
4. **Given** a destructive single-item or source-wide delete, **when** confirmation is missing, **then** no deletion is sent.
5. **Given** any backend response or error, **when** it is returned through WordPress REST, **then** it is normalized and sanitized into the plugin's stable response shape.

**Tasks**

- [ ] Implement settings read/update routes.
- [ ] Implement aggregated Listings, Listing Reviews, optional post-type tab, source update, and tab-items routes.
- [ ] Implement provisioning, single index/delete, reindex, status, diagnostics, admin test-chat, and source-wide delete routes.
- [ ] Add admin middleware for capabilities and nonces.
- [ ] Add DTO validation, pagination, search, directory type, status, category, location, tag, index-status filters, stable errors, and destructive confirmation.
- [ ] Sanitize and minimize every backend response exposed to the admin application.
- [ ] Add permission, nonce, validation, pagination, confirmation, success, and error tests for every route.

**Dependencies:** WP-US-002, WP-US-003, WP-US-007  
**Priority:** Must have

### WP-US-009 — Operate Ask Sunny from an accessible dashboard

**User story**

> As a **WordPress administrator**, I want a clear dashboard for setup, sources, indexing, and diagnostics, so that I can operate Ask Sunny without command-line access.

**Acceptance criteria**

1. **Given** an unprovisioned installation, **when** the dashboard opens, **then** it guides the administrator through configuration, provisioning, source review, diagnostics, initial indexing, and widget enablement.
2. **Given** only Directorist sources are active, **when** Data Sources opens, **then** exactly two initial tabs appear: Listings and Listing Reviews.
3. **Given** the Listings tab, **when** items load, **then** the administrator can filter by directory type, status, category, and location while viewing counts, eligibility, index/retrieval status, timestamps, and errors.
4. **Given** the Listing Reviews tab, **when** items load, **then** the administrator can filter by directory type and see the one global reviews enabled state without per-directory toggles.
5. **Given** an optional post, page, or custom post type is enabled in settings, **when** Data Sources reloads, **then** that post type receives its own tab with status, category, and tag controls supported by its registered taxonomies.
6. **Given** an indexing operation, **when** it is pending, running, completed, interrupted, or failed, **then** accessible progress and appropriate retry/resume actions are shown.
7. **Given** a version conflict, network error, empty state, or destructive operation, **when** it occurs, **then** the dashboard presents a clear accessible state and never claims unsynchronized success.

**Tasks**

- [ ] Design Setup, General, Data Sources, Indexing, Diagnostics, and Test Chat submenus.
- [ ] Implement the setup/provisioning workflow.
- [ ] Implement the two initial Listings and Listing Reviews tabs.
- [ ] Implement Listings filters for directory type, status, category, and location.
- [ ] Implement Listing Reviews directory-type filtering and one global review enablement control.
- [ ] Add one tab per enabled optional post type with status, category, and tag filters when supported.
- [ ] Implement paginated item tables with search, indexing status, and tab-specific filters.
- [ ] Implement progress, retry, resume, conflict, empty, error, and confirmation states.
- [ ] Add keyboard, focus, screen-reader, contrast, responsive, and reduced-motion support.
- [ ] Add component, REST-integration, accessibility, and browser end-to-end tests.

**Dependencies:** WP-US-008

**Priority:** Must have

### WP-US-010 — Test the widget and backend integration

**User story**

> As a **WordPress administrator**, I want a Test Chat submenu that exercises the production widget and backend API path, so that I can verify configuration, retrieval, and responses before exposing the widget to visitors.

**Acceptance criteria**

1. **Given** an administrator with `manage_options`, **when** Test Chat opens, **then** it displays an isolated production-widget preview plus backend connection, allowlist-sync, and hybrid-search status.
2. **Given** a valid test message, **when** it is submitted, **then** WordPress calls an admin-only REST route that proxies the request with `channel = admin_test` through the same backend client and response validator used by public chat.
3. **Given** a successful response, **when** it renders, **then** the preview shows the answer, citations, recommendations, follow-up prompts, correlation ID, and latency.
4. **Given** a backend, authentication, retrieval, timeout, or schema failure, **when** the request finishes, **then** the submenu shows a sanitized actionable error and correlation ID without exposing credentials or private diagnostics.
5. **Given** caller-supplied provider, model, API key, tool, or allowlist override fields, **when** the test route validates input, **then** those fields are rejected or ignored.
6. **Given** a test conversation, **when** it is persisted by the backend, **then** it uses the `admin_test` product channel and remains separate from visitor sessions.

**Tasks**

- [ ] Add a capability-protected Test Chat submenu and admin asset entry point.
- [ ] Add `ChatTestController` and `POST /test-chat` behind admin middleware.
- [ ] Reuse the production chat proxy, DTO validation, response sanitization, and widget renderer.
- [ ] Add `admin_test` channel, correlation ID, latency, and integration-status output.
- [ ] Add reset/new-test-conversation behavior without storing secrets or full transcripts in browser storage.
- [ ] Add permission, nonce, success, error, timeout, injection, source-policy, and accessibility tests.

**Dependencies:** WP-US-008, WP-US-009, WP-US-012

**Priority:** Must have

## Epic 6 — Visitor Chat Experience

### WP-US-011 — Configure widget placement and appearance

**User story**

> As a **WordPress administrator**, I want to choose where the chat widget appears and customize its appearance, so that it fits the site's content strategy and visual identity.

**Acceptance criteria**

1. **Given** widget display mode is `all_pages`, **when** any public frontend page renders, **then** the global widget is eligible to appear.
2. **Given** display mode is `selected_pages` or `excluded_pages`, **when** a public page renders, **then** the widget appears only according to the saved unique published page IDs.
3. **Given** display mode is `shortcode_only`, **when** a page has no Ask Sunny shortcode, **then** global widget markup and assets are not rendered.
4. **Given** an administrator selects `light`, `dark`, `auto`, or `custom`, **when** appearance settings are saved, **then** the scheme and sanitized custom primary, surface, and text colors are previewed and persisted.
5. **Given** `bottom_left` or `bottom_right` is selected, **when** the widget renders, **then** it uses that position without covering required site controls at supported viewports.
6. **Given** a sanitized plain-text welcome message within the configured limit, **when** a visitor opens a new widget session, **then** the message appears before the first user turn.
7. **Given** invalid page IDs, modes, positions, colors, markup, scripts, or an inaccessible custom color combination, **when** settings are submitted, **then** validation rejects the invalid fields and preserves the previous valid configuration.

**Tasks**

- [ ] Add settings for enabled state, display mode, page IDs, position, color scheme, custom color tokens, and welcome message.
- [ ] Build page search/selection controls for selected and excluded display modes.
- [ ] Build color-scheme, custom-color, position, welcome-message, and live-preview controls.
- [ ] Implement strict enum, published-page, CSS-color, contrast, plain-text, and length validation.
- [ ] Implement current-page eligibility evaluation before global asset enqueue/render.
- [ ] Localize only sanitized public widget configuration on eligible pages.
- [ ] Keep shortcode rendering explicit and prevent duplicate global/shortcode initialization.
- [ ] Add settings REST, page-targeting, preview, sanitization, contrast, caching, and rendering tests.

**Dependencies:** WP-US-008

**Priority:** Must have

### WP-US-012 — Proxy visitor chat securely

**User story**

> As a **site visitor**, I want my chat request proxied through WordPress securely, so that I can use Ask Sunny without receiving backend credentials or bypassing site controls.

**Acceptance criteria**

1. **Given** the widget is enabled and a valid message is submitted, **when** the WordPress chat route receives it, **then** input and page context are sanitized and the server-side backend client sends the request with the installation credential.
2. **Given** a logged-in user, **when** chat is submitted, **then** the WordPress nonce and user identity are validated before proxying.
3. **Given** an anonymous user, **when** chat is submitted, **then** a privacy-conscious IP/session rate limit is enforced.
4. **Given** a caller-supplied model, tool, API key, or allowed-source setting, **when** input is validated, **then** the value cannot affect the backend request.
5. **Given** a complete backend response, **when** WordPress returns it, **then** only the documented answer, citation, recommendation, follow-up, conversation, and message fields are exposed.
6. **Given** a timeout, rate limit, invalid response, or backend failure, **when** the proxy completes, **then** it returns a stable friendly error without exposing credentials or internal details.

**Tasks**

- [ ] Implement widget-enabled middleware and chat input DTO validation.
- [ ] Implement logged-in nonce/user handling and anonymous session handling.
- [ ] Implement privacy-conscious anonymous rate limiting.
- [ ] Implement the server-side chat proxy with a timeout above the configured backend AI-provider timeout.
- [ ] Strip caller policy/model/tool fields and minimize page context.
- [ ] Validate and sanitize complete backend responses.
- [ ] Add anonymous, logged-in, abuse, oversized-input, timeout, malformed-response, and secret-exposure tests.

**Dependencies:** WP-US-002  
**Priority:** Must have

### WP-US-013 — Use an accessible multi-turn chat widget

**User story**

> As a **site visitor**, I want an accessible chat widget with citations and recommendation cards, so that I can ask follow-up questions and act on trustworthy results from any supported device.

**Acceptance criteria**

1. **Given** the current page is eligible under the saved display mode or contains a shortcode, **when** the page renders, **then** exactly one widget instance initializes per intended placement without duplicate handlers.
2. **Given** a visitor submits a message, **when** the complete response arrives, **then** the widget renders the answer, direct citations, recommendation cards, follow-up prompts, and retry state safely.
3. **Given** a follow-up question, **when** it is sent, **then** the existing anonymous session and conversation identifiers are reused.
4. **Given** a browser refresh, **when** the widget reopens, **then** only the documented continuity identifiers are restored from browser storage and no secrets or full private transcript are stored there.
5. **Given** keyboard-only, screen-reader, narrow-screen, zoomed, high-contrast, or reduced-motion use, **when** the widget is operated, **then** input, status, citations, cards, close/open controls, and errors remain perceivable and usable.
6. **Given** citation or recommendation content containing unsafe markup or URLs, **when** it renders, **then** it is escaped or rejected and cannot execute script.
7. **Given** saved widget appearance and welcome settings, **when** the widget initializes, **then** it applies the color scheme, custom tokens, position, and welcome message without exposing raw admin data.

**Tasks**

- [ ] Implement global and shortcode render providers with duplicate-initialization protection.
- [ ] Apply sanitized page-targeting and appearance settings through CSS custom properties and public widget configuration.
- [ ] Render the welcome message only for a new/reset conversation.
- [ ] Implement open/close, message list, composer, loading, completion, retry, and error states.
- [ ] Implement sanitized citation links, recommendation cards, reasons, disclosures, and follow-up prompts.
- [ ] Persist only anonymous session and conversation identifiers with reset behavior.
- [ ] Add live-region status, focus management, keyboard behavior, semantic markup, contrast, responsive layout, and reduced-motion styling.
- [ ] Test supported themes, cache/minification behavior, browsers, viewports, accessibility tools, unsafe output, and multi-turn continuity.

**Dependencies:** WP-US-011, WP-US-012

**Priority:** Must have

## Epic 7 — Plugin Release Readiness

### WP-US-014 — Diagnose and support the plugin in production

**User story**

> As a **WordPress administrator**, I want actionable diagnostics and privacy-safe logs, so that I can resolve connection, indexing, and widget problems without exposing visitor or credential data.

**Acceptance criteria**

1. **Given** an authorized diagnostics request, **when** it runs, **then** it reports plugin/dependency versions, backend reachability, authentication status, active AI and embedding providers, ParadeDB/hybrid-search state, source-version status, queue state, and sanitized recent failures.
2. **Given** an indexing or chat failure, **when** logs are inspected, **then** a correlation ID links WordPress and backend activity without recording credentials, private fields, or full visitor messages by default.
3. **Given** a stale or mismatched retrieval configuration, **when** diagnostics runs, **then** the mismatch is visible with a safe resynchronization action.
4. **Given** common widget, connection, source, or date/timezone problems, **when** support documentation is followed, **then** it provides a reproducible diagnostic path.
5. **Given** a data export, deletion, deactivation, or uninstall request, **when** it runs, **then** documented local privacy and retention behavior is followed.

**Tasks**

- [ ] Implement plugin, dependency, connection, authentication, source-version, queue, and recent-error diagnostics.
- [ ] Add correlation IDs and privacy-safe logging around provisioning, synchronization, indexing, and chat.
- [ ] Implement allowlist resynchronization and failed-job retry actions.
- [ ] Document widget, provisioning, irrelevant-result, date/timezone, and stale-index troubleshooting.
- [ ] Document and test local data export, deletion, retention, deactivation, and uninstall behavior.
- [ ] Add diagnostics authorization, redaction, mismatch, and recovery tests.

**Dependencies:** WP-US-007, WP-US-009, WP-US-010, WP-US-013

**Priority:** Must have

### WP-US-015 — Release a compatible and resilient plugin build

**User story**

> As a **WordPress site owner**, I want a tested and upgrade-safe plugin release, so that I can deploy Ask Sunny without breaking my content workflows or frontend experience.

**Acceptance criteria**

1. **Given** a clean supported WordPress installation with Directorist, **when** the release package is installed and configured, **then** provisioning, source discovery, initial indexing, diagnostics, and chat smoke tests pass.
2. **Given** an existing supported plugin version, **when** it is upgraded, **then** settings and index status remain valid and migrations run once.
3. **Given** duplicate hooks, rapid content changes, network failures, or interrupted background work, **when** resilience tests run, **then** jobs remain bounded, resumable, and idempotent.
4. **Given** the supported theme, browser, PHP, WordPress, Directorist, and caching matrix, **when** compatibility tests run, **then** no launch-blocking admin, indexing, REST, or widget issue remains.
5. **Given** a security, accessibility, performance, and privacy review, **when** release is approved, **then** no critical/high finding remains and the release checklist is complete.

**Tasks**

- [ ] Build automated PHP, REST, JavaScript, contract, accessibility, and browser CI suites.
- [ ] Test clean install, activation, upgrade, deactivation, uninstall, and rollback procedures.
- [ ] Test aggregated Listings and Listing Reviews tabs, global review enablement, enabled post-type tabs, all required filters, and high-volume batch indexing.
- [ ] Test the Test Chat submenu against successful, degraded, and failing backend integrations.
- [ ] Test duplicate hooks, rapid updates, queue interruption, retry, rate limits, and backend degradation.
- [ ] Test the supported WordPress, PHP, Directorist, theme, browser, viewport, and caching matrix.
- [ ] Review capabilities, nonces, SSRF, XSS, output escaping, secret storage, logging, metadata limits, and dependency vulnerabilities.
- [ ] Measure admin, indexing, REST-proxy, asset, and widget performance against launch thresholds.
- [ ] Build the release artifact and complete installation, upgrade, smoke-test, and support documentation.

**Dependencies:** WP-US-014

**Priority:** Must have

## Recommended Story Order

1. WP-US-001 → WP-US-003: plugin foundation, secure setup, and source administration.
2. WP-US-004 → WP-US-007: content normalization and synchronization.
3. WP-US-008 → WP-US-009: admin REST API and dashboard.
4. WP-US-011: widget placement and appearance configuration.
5. WP-US-012 → WP-US-013: secure chat proxy and visitor widget.
6. WP-US-010: Test Chat submenu and API integration verification.
7. WP-US-014 → WP-US-015: diagnostics, compatibility, resilience, and release.

## Related Specifications

- [`WP_PLUGIN_ARCHITECTURE.md`](../plugin/WP_PLUGIN_ARCHITECTURE.md)
- [`WP_PLUGIN_DATA_SCHEMA.md`](../plugin/WP_PLUGIN_DATA_SCHEMA.md)
- [`WP_PLUGIN_REST_API_CONTRACT.md`](../plugin/WP_PLUGIN_REST_API_CONTRACT.md)
- [`DATA_AND_RAG_DESIGN.md`](../shared/DATA_AND_RAG_DESIGN.md)
- [`SETUP_AND_OPERATIONS.md`](../shared/SETUP_AND_OPERATIONS.md)
