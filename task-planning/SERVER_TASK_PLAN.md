# Ask Sunny Backend Server User Story Plan

## Product Goal

Deliver a production-ready Bun/Hono backend that runs natively or through optional Docker Compose, securely accepts WordPress content, uses ParadeDB BM25 + pgvector as the normal production retrieval mode only after a mandatory package compatibility gate, runs grounded multi-turn conversations through LangGraph and a runtime-selected provider abstraction, and returns complete answers with citations and recommendation cards.

This plan is self-contained. It defines all backend contract, implementation, test, security, and operational work required for the server release.

## Story Standard

Every story uses the standard format:

> As a **persona**, I want **a capability**, so that **I receive a measurable benefit**.

Acceptance criteria use Given–When–Then statements. A story is complete only when all acceptance criteria pass and every required task is checked off.

## Definition of Done

- All story acceptance criteria pass in automated or documented acceptance tests.
- Code is reviewed, formatted, linted, statically analyzed, and covered by appropriate unit, integration, contract, and security tests.
- Database changes use versioned, repeatable migrations and include recovery notes.
- API behavior matches the server REST contract and returns stable error shapes.
- Secrets and private content do not appear in normal logs, API errors, embeddings, or client-visible payloads.
- Logs, metrics, health checks, and runbooks are updated for every production-facing capability.
- Relevant architecture, schema, API, setup, and troubleshooting documentation is current.

## Epic 1 — Service Foundation

### SV-US-001 — Run and diagnose the API service

**User story**

> As a **server operator**, I want the API service to start predictably and report its health, so that I can deploy and monitor it safely.

**Acceptance criteria**

1. **Given** a valid environment configuration, **when** the service starts, **then** it listens on the configured host and port and reports a ready state.
2. **Given** a missing or invalid required setting, **when** startup is attempted, **then** the process fails before accepting traffic and identifies the invalid setting without exposing secrets.
3. **Given** a health request, **when** PostgreSQL is available and Redis is either healthy or disabled, **then** `GET /health` returns the documented healthy response.
4. **Given** an API request, **when** it completes or fails, **then** structured logs contain a correlation ID, route, status, and latency without credentials or private content.
5. **Given** a termination signal, **when** the service shuts down, **then** it stops accepting requests and closes shared clients gracefully.
6. **Given** a native deployment, **when** the service runs behind the reverse proxy, **then** Bun, ParadeDB, and optional Redis bind only to private interfaces.
7. **Given** an optional Docker deployment, **when** the Compose stack runs, **then** only the reverse proxy is public and application dependencies remain on private Docker networks.
8. **Given** hybrid is requested but its package or readiness gate has not passed, **when** health is queried, **then** requested and effective modes differ, the stable degraded reason is returned, and BM25 is not reported as active.

**Tasks**

- [ ] Confirm supported Bun version and final API route prefix.
- [ ] Scaffold the Bun, JavaScript, and Hono application.
- [ ] Add validated configuration loading and an example environment file.
- [ ] Add JSON parsing, request-body limits, correlation IDs, structured logging, and global error mapping.
- [ ] Implement `GET /health` with database, optional Redis, requested/effective hybrid mode, and stable degraded-reason status.
- [ ] Add graceful startup and shutdown handling.
- [ ] Add formatting, linting, static analysis, unit-test, and development commands.
- [ ] Add service-foundation tests and an operator startup guide.
- [ ] Add native service-manager/reverse-proxy templates and startup checks.
- [ ] Add an optional multi-stage API Dockerfile, Compose services, private networks, persistent volumes, and health checks.

**Dependencies:** none  
**Priority:** Must have

### SV-US-002 — Manage the database schema safely

**User story**

> As a **server operator**, I want versioned database migrations and recovery procedures, so that schema changes can be deployed without losing indexed or conversational data.

**Acceptance criteria**

1. **Given** an empty supported ParadeDB PostgreSQL database, **when** migrations run, **then** all launch tables, constraints, `pg_search`/pgvector extensions, BM25 indexes, and vector indexes are created successfully.
2. **Given** an up-to-date database, **when** migrations run again, **then** no migration is applied twice and no data is changed unexpectedly.
3. **Given** a failed migration, **when** deployment stops, **then** the failure is visible and the documented recovery procedure can restore a usable state.
4. **Given** records of unlike source kinds, **when** data is written, **then** database constraints and repositories prevent them from sharing the wrong content or embedding table.
5. **Given** a changed embedding dimension, **when** a migration is planned, **then** it requires new vector storage and an explicit re-embedding procedure.
6. **Given** any missing `pg_search` extension, BM25 index, or direct BM25 smoke-query failure, **when** deployment validation runs, **then** hybrid search remains disabled and the degraded state is reported.
7. **Given** a native package or Docker database image, **when** its compatibility gate runs, **then** its PostgreSQL major target, execution OS/release, and CPU architecture exactly match the running database environment before extension or BM25 migrations continue.
8. **Given** missing, ambiguous, or mismatched compatibility evidence, **when** migration preparation runs, **then** BM25 migrations do not run, `HYBRID_SEARCH_ENABLED=false` is retained, and vector-only status plus the blocker are reported.

**Tasks**

- [ ] Add PostgreSQL pooling, transaction helpers, and migration tooling.
- [ ] Create and track the `schema_migrations` table.
- [ ] Enable `pg_search`, `vector`, and `pgcrypto` through migrations.
- [ ] Implement configuration, API-key, data-source, content, embedding, conversation, usage, admin, and checkpoint tables.
- [ ] Add foreign keys, uniqueness constraints, checks, GIN indexes, source-specific BM25 indexes, and vector indexes.
- [ ] Add deterministic `search_document` and stable `search_key` fields for ParadeDB BM25 retrieval.
- [ ] Add `ANALYZE`, direct `|||`/`pdb.score(search_key)` smoke checks, and hybrid enablement gates.
- [ ] Add native-host and database-container compatibility checks for PostgreSQL major, OS ID/release or codename, CPU architecture, installed package version, and pinned image digest.
- [ ] Prevent extension/BM25 migrations and effective hybrid mode when compatibility evidence is incomplete or mismatched.
- [ ] Add migration integration tests from an empty database and an existing-version fixture.
- [ ] Document backup, migration failure, restore, and embedding-dimension procedures.

**Dependencies:** SV-US-001  
**Priority:** Must have

## Epic 2 — Installation Security and Retrieval Policy

### SV-US-003 — Provision and authenticate the WordPress installation

**User story**

> As a **WordPress administrator**, I want to provision a scoped installation credential, so that my site can call the backend without exposing reusable secrets to visitors.

**Acceptance criteria**

1. **Given** a valid provisioning secret and installation identity, **when** provisioning is requested, **then** the server returns a new installation key and stores only its secure hash.
2. **Given** an invalid provisioning secret, **when** provisioning is requested, **then** the server returns the documented authentication error and creates no credential.
3. **Given** an active installation key, **when** an allowed protected route is called, **then** authentication succeeds and `last_used_at` is updated.
4. **Given** a revoked, malformed, or wrong-scope key, **when** a protected route is called, **then** access is denied without revealing which credential field failed.
5. **Given** a key rotation, **when** the new key is issued, **then** the rotation is auditable and the previous key follows the configured revocation policy.

**Tasks**

- [ ] Finalize provisioning and bearer-authentication request/response fixtures.
- [ ] Implement provisioning-secret comparison and installation validation.
- [ ] Implement cryptographically secure key generation, prefixing, hashing, rotation, and revocation.
- [ ] Add authentication and key-scope authorization middleware.
- [ ] Redact provisioning and installation secrets from logs and errors.
- [ ] Add provisioning, rotation, revoked-key, and wrong-scope contract tests.
- [ ] Document emergency credential rotation.

**Dependencies:** SV-US-002  
**Priority:** Must have

### SV-US-004 — Enforce the authoritative retrieval allowlist

**User story**

> As a **site administrator**, I want the server to persist and enforce my site's allowed data-source keys, so that chat never searches content outside the approved sources.

**Acceptance criteria**

1. **Given** an authenticated complete allowlist and matching expected version, **when** it is submitted, **then** keys are validated, deduplicated, canonicalized, and replaced atomically with a new version.
2. **Given** a stale expected version, **when** an update is submitted, **then** the server returns `409 retrieval_config_conflict` and preserves the newer configuration.
3. **Given** an empty or missing stored allowlist, **when** any retrieval path executes, **then** it fails closed with zero candidates.
4. **Given** caller- or model-supplied source keys, **when** retrieval executes, **then** the requested keys are intersected with the stored allowlist.
5. **Given** a disabled source with retained indexed rows, **when** chat, vector search, structured search, or detail lookup runs, **then** no row from that source is returned.

**Tasks**

- [ ] Finalize allowlist update and version-conflict fixtures.
- [ ] Implement atomic allowlist replacement and optimistic version checking.
- [ ] Add a single fail-closed policy accessor for repositories and model tools.
- [ ] Apply the policy to structured queries, vector queries, detail lookup, and tool execution.
- [ ] Add concurrency and policy-bypass tests.
- [ ] Expose allowlist version and update time through diagnostics.

**Dependencies:** SV-US-003  
**Priority:** Must have

## Epic 3 — Content Ingestion and Embeddings

### SV-US-005 — Store each content source in its correct read model

**User story**

> As a **content administrator**, I want listings, reviews, and WordPress posts validated and stored independently, so that each content type keeps the structure needed for accurate retrieval.

**Acceptance criteria**

1. **Given** a valid Directorist listing, review, or WordPress-post payload, **when** it is upserted, **then** it is written only to its matching source-kind table.
2. **Given** a new data-source key, **when** valid content is received, **then** retrieval metadata is created or refreshed without making that source allowed automatically.
3. **Given** an active review whose parent listing is missing, **when** it is upserted, **then** the server returns `409 parent_listing_missing` and writes no orphan review.
4. **Given** malformed, oversized, private, executable, cross-kind, or unsafe metadata, **when** validation runs, **then** the payload is rejected with a stable field-specific error.
5. **Given** the same source ID under different data-source keys, **when** records are stored, **then** source identity remains unique within each concrete data source.

**Tasks**

- [ ] Create canonical contract fixtures for all three source kinds and boundary cases.
- [ ] Implement source-key, kind, parent, URL, status, field, and metadata validation.
- [ ] Implement retrieval-only `data_sources` registration and parent-source linkage.
- [ ] Implement separate listing, review, and WordPress-content repositories.
- [ ] Enforce metadata field count, key, label, array, nesting, URL, and payload-size limits.
- [ ] Add validation and cross-kind isolation tests.

**Dependencies:** SV-US-003  
**Priority:** Must have

### SV-US-006 — Index changed content idempotently

**User story**

> As a **content administrator**, I want changed content embedded and unchanged content skipped, so that the search index stays current without unnecessary model cost.

**Acceptance criteria**

1. **Given** a valid new record, **when** it is indexed, **then** canonical normalized content, deterministic embedding text, a content hash, and a vector are stored in the matching tables.
2. **Given** an identical payload delivered repeatedly, **when** it is processed, **then** the server reports `unchanged` and does not request another embedding.
3. **Given** any public content-bearing field change, **when** it is processed, **then** the content hash changes and the vector is regenerated.
4. **Given** only operational or raw-debug values change, **when** the payload is processed, **then** the content hash and embedding remain unchanged.
5. **Given** a transient embedding failure, **when** processing ends, **then** the prior usable record is not corrupted and the error is observable and safely retryable.

**Tasks**

- [ ] Define stable canonical serialization for each source kind.
- [ ] Implement normalized payload, embedding-text, and hash builders.
- [ ] Integrate the configured embeddings API with timeout, retry, and response validation.
- [ ] Implement transactional content and source-specific vector upserts.
- [ ] Skip embeddings for matching content hashes.
- [ ] Record indexing latency, model, token/cost metadata when available, and errors.
- [ ] Add deterministic hashing, unchanged-content, changed-content, and failure tests.

**Dependencies:** SV-US-005  
**Priority:** Must have

### SV-US-007 — Synchronize content in single and bulk operations

**User story**

> As a **WordPress integration**, I want retry-safe upsert and deletion endpoints, so that large sites can keep the backend index synchronized after updates and interruptions.

**Acceptance criteria**

1. **Given** a valid single upsert, **when** it is submitted, **then** the response identifies whether the record was indexed or unchanged.
2. **Given** a mixed batch within the configured limit, **when** it is submitted, **then** each item is processed by its correct repository and partial failures are reported without hiding successful results.
3. **Given** a batch larger than the configured limit, **when** it is submitted, **then** the server rejects it before processing any item.
4. **Given** a single deletion, **when** it is submitted, **then** the record is tombstoned and excluded from retrieval.
5. **Given** an explicit source-wide deletion, **when** it is submitted, **then** all active rows for that key are tombstoned without changing the allowlist.
6. **Given** a repeated upsert or deletion, **when** it is retried with the same content, **then** the final state is correct and no duplicate record is created.

**Tasks**

- [ ] Implement `/content/upsert`, `/content/bulk-upsert`, `/content/delete`, and `/content/delete-by-data-source`.
- [ ] Enforce a configurable batch limit with a launch default of 50.
- [ ] Define correlation, idempotency, and retry behavior for every mutation.
- [ ] Implement source-specific tombstoning and retrieval exclusion.
- [ ] Persist per-item errors and aggregate indexing usage events.
- [ ] Add contract, partial-batch, repeated-delivery, and deletion tests.

**Dependencies:** SV-US-006  
**Priority:** Must have

## Epic 4 — Retrieval and Ranking

### SV-US-008 — Retrieve relevant content within user constraints

**User story**

> As a **site visitor**, I want search results to match my meaning and stated constraints, so that recommendations are relevant to my actual question.

**Acceptance criteria**

1. **Given** an allowed query with structured constraints, **when** retrieval runs, **then** it filters by source key and applicable date, location, category, amenity, price, rating, taxonomy, or approved metadata fields.
2. **Given** a keyword query, **when** ParadeDB retrieval runs, **then** BM25 results retain source identity, direct URLs, matched metadata, and keyword score.
3. **Given** a natural-language query, **when** semantic retrieval runs, **then** pgvector results retain source identity, direct URLs, and matched metadata.
4. **Given** BM25 and vector candidates from multiple source kinds, **when** results are fused, **then** bounded weighted reciprocal-rank fusion produces normalized results and removes duplicates.
5. **Given** review evidence while the global optional Listing Reviews family is enabled, **when** it contributes to a recommendation, **then** it remains linked to its parent listing and is not returned as a listing card.
6. **Given** inactive, deleted, disallowed, or URL-less content, **when** retrieval runs, **then** that content is not returned as a recommendation candidate.
7. **Given** hybrid search is disabled because BM25 verification failed, **when** retrieval runs, **then** vector retrieval remains available and diagnostics report the degraded mode.
8. **Given** a verified normal installation with no hybrid override, **when** retrieval runs, **then** `HYBRID_SEARCH_ENABLED=true` and BM25 plus pgvector execute for the same query before fusion.
9. **Given** `HYBRID_SEARCH_ENABLED=true` without a successful package and readiness gate, **when** the service starts or reports readiness, **then** it refuses effective hybrid execution and exposes the exact degraded reason while retaining the vector-only path.

**Tasks**

- [ ] Implement ParadeDB BM25 and pgvector retrieval for listings.
- [ ] Implement ParadeDB BM25 and pgvector retrieval for reviews with parent linkage.
- [ ] Implement ParadeDB BM25 and pgvector retrieval for WordPress content.
- [ ] Add typed metadata, date/timezone, and optional distance filtering.
- [ ] Bound candidate sets and implement configurable weighted reciprocal-rank fusion.
- [ ] Add hybrid feature flags, weights, candidate limits, and vector-only fallback diagnostics.
- [ ] Use `HYBRID_SEARCH_ENABLED=false` as the safe bootstrap value and promote verified deployments to the intended `true` production state only after every hybrid gate passes.
- [ ] Add detail lookup constrained by the stored allowlist.
- [ ] Build retrieval fixtures for exact, semantic, empty, deleted, and disallowed cases.

**Dependencies:** SV-US-004, SV-US-007  
**Priority:** Must have

### SV-US-009 — Rank and cite grounded recommendations

**User story**

> As a **site visitor**, I want recommendations ranked by relevance and linked to their sources, so that I can understand and verify the answer.

**Acceptance criteria**

1. **Given** candidates that differ in constraint match and promotion status, **when** ranking runs, **then** semantic and exact constraint relevance outrank featured or configured promotion signals.
2. **Given** a recommendation, **when** it is returned, **then** it contains a stable source key, direct public URL, title, and evidence-based match reason.
3. **Given** a factual statement supported by retrieved content, **when** the answer payload is assembled, **then** a citation maps it to an allowed stored source.
4. **Given** uncertain dates, availability, or operating details, **when** evidence is insufficient, **then** the result is labeled uncertain rather than invented.
5. **Given** duplicate candidates or invalid URLs, **when** ranking completes, **then** duplicates and invalid recommendation targets are removed.

**Tasks**

- [ ] Define and version the launch ranking policy.
- [ ] Implement exact-match, semantic, date, location, category, metadata, freshness, review, featured, and configured-promotion signals.
- [ ] Implement review-evidence aggregation to parent listings.
- [ ] Build citation and recommendation-card assemblers.
- [ ] Add uncertainty and configured disclosure metadata.
- [ ] Create a retrieval evaluation set and baseline relevance metrics.
- [ ] Add ordering, deduplication, URL, citation, and disclosure tests.

**Dependencies:** SV-US-008  
**Priority:** Must have

## Epic 5 — Conversation and Grounded Answers

### SV-US-010 — Preserve conversation context safely

**User story**

> As a **returning visitor**, I want follow-up questions to retain the current conversation context, so that I do not have to repeat my constraints.

**Acceptance criteria**

1. **Given** a new anonymous or logged-in chat request, **when** the first turn starts, **then** a durable conversation and LangGraph thread are created.
2. **Given** a valid conversation ID belonging to the same visitor identity, **when** a follow-up arrives, **then** recent messages and relevant summary context are loaded.
3. **Given** a conversation belonging to another identity, **when** access is attempted, **then** no history or existence details are disclosed.
4. **Given** a successful or failed turn, **when** processing ends, **then** messages, citations, tool calls, application correlation IDs, usage, latency, and status are auditable without provider-specific identifiers.
5. **Given** a process restart during later use, **when** the conversation resumes, **then** durable messages and PostgreSQL-backed graph state remain available.

**Tasks**

- [ ] Implement conversation creation, identity binding, ownership checks, and history loading.
- [ ] Implement message, citation, tool-call, usage, and error persistence.
- [ ] Configure PostgreSQL-backed LangGraph checkpoints separately from audit tables.
- [ ] Implement history retrieval with authorization.
- [ ] Add soft deletion, anonymization, and retention service boundaries.
- [ ] Add isolation, restart, history, and deletion tests.

**Dependencies:** SV-US-002  
**Priority:** Must have

### SV-US-011 — Receive a complete grounded chat response

**User story**

> As a **site visitor**, I want one complete answer grounded in the site's approved content, so that I receive trustworthy guidance without partial or unsupported output.

**Acceptance criteria**

1. **Given** a valid message, **when** the chat workflow runs, **then** it loads context, extracts intent and constraints, selects allowed sources, retrieves evidence, ranks results, generates the answer, and persists the turn.
2. **Given** a request missing a material constraint, **when** a reliable answer cannot be produced, **then** the response asks one useful clarification instead of inventing an assumption.
3. **Given** model-selected tools or source keys, **when** tools run, **then** only registered server-owned tools execute and every source key is constrained by the stored allowlist.
4. **Given** a successful turn, **when** `POST /chat` returns, **then** one non-streaming JSON payload contains the conversation ID, message ID, answer, citations, recommendations, and optional follow-up questions.
5. **Given** a retrieval, model, schema, or timeout failure, **when** the turn ends, **then** the server returns a stable friendly error/fallback, persists the failure, and does not present unsupported claims.
6. **Given** caller-supplied SQL, model overrides, raw tool names, or allowed-source settings, **when** validation runs, **then** those values cannot alter server policy or execution.
7. **Given** `AI_PROVIDER=openai`, **when** a turn runs, **then** the OpenAI adapter uses only OpenAI environment configuration and returns the common internal response shape.
8. **Given** `AI_PROVIDER=groq`, **when** a turn runs, **then** the Groq adapter uses only Groq environment configuration, excludes unsupported provider parameters, supplies server-owned conversation history, and returns the same internal response shape.
9. **Given** an invalid provider value or missing selected-provider configuration, **when** the service starts, **then** startup fails without exposing any API key.
10. **Given** any conversation, message, tool-call, or usage record, **when** it is persisted, **then** no AI-provider discriminator or provider-specific conversation state is written to the database.

**Tasks**

- [ ] Add `AI_PROVIDER=openai|groq` as the single chat-generation switch.
- [ ] Add environment-only OpenAI and Groq keys, base URLs, models, and shared provider timeout.
- [ ] Keep embedding provider, model, URL, and dimensions independently configured.
- [ ] Implement a common generation-provider interface with OpenAI and Groq Responses adapters.
- [ ] Resolve adapters through a runtime registry so orchestration, routes, domain services, and persistence contain no provider-name branches.
- [ ] Normalize provider-specific tool calls, structured output, usage, errors, and supported parameters.
- [ ] Verify the launch Responses API models and embedding model against current provider documentation.
- [ ] Define structured schemas for model tool inputs, tool outputs, and final responses.
- [ ] Implement LangGraph nodes for context, intent, constraints, clarification, source selection, retrieval, ranking, generation, and persistence.
- [ ] Implement server-owned `list_data_sources`, `search_content`, and `get_content_detail` tools.
- [ ] Add bounded tool iterations, input limits, model timeout, schema validation, and grounded fallback behavior.
- [ ] Implement `POST /chat` and `GET /conversations/:id`.
- [ ] Add multi-turn, clarification, injection, allowlist, timeout, model-failure, and schema contract tests.

**Dependencies:** SV-US-009, SV-US-010  
**Priority:** Must have

## Epic 6 — Operations and Release

### SV-US-012 — Observe and administer the backend

**User story**

> As a **server operator**, I want diagnostics, usage reporting, and controlled reindex coordination, so that I can detect failures and support the WordPress integration.

**Acceptance criteria**

1. **Given** an authorized diagnostics request, **when** it runs, **then** it reports native-service or Docker dependency health, ParadeDB extensions and BM25 indexes, hybrid mode, runtime generation/embedding configuration, allowlist version, source counts, and latest indexing state.
2. **Given** an authorized usage query with a date range, **when** it runs, **then** it returns chat, indexing, latency, token, BM25/vector/fused retrieval, fallback, and error aggregates without exposing private message content or persisting provider identity.
3. **Given** a reindex coordination request, **when** it is accepted, **then** it receives a tracked status while WordPress remains responsible for re-sending source-of-truth content.
4. **Given** a wrong-scope installation key, **when** an admin-only endpoint is called, **then** access is denied.
5. **Given** an operational failure, **when** thresholds are exceeded, **then** logs and metrics provide enough correlation to diagnose the affected request or job.

**Tasks**

- [ ] Implement `GET /admin/diagnostics` and `GET /admin/usage`.
- [ ] Implement `POST /admin/reindex` as a tracked coordination boundary.
- [ ] Add admin key/session scope enforcement.
- [ ] Instrument chat, BM25/vector/fused retrieval, indexing, database-pool, selected-provider, embedding-provider, and error metrics.
- [ ] Add health, latency, error-rate, rate-limit, and stale-index alert guidance.
- [ ] Add diagnostics, usage, authorization, and privacy tests.

**Dependencies:** SV-US-007, SV-US-011  
**Priority:** Must have

### SV-US-013 — Deploy and recover the production service

**User story**

> As a **server operator**, I want a secure, observable, and recoverable deployment process, so that the API can be released and restored with predictable risk.

**Acceptance criteria**

1. **Given** a production-like native host, **when** deployment runs, **then** Bun dependencies install, environment configuration validates without overwriting secrets, ParadeDB is backed up and migrated, the supervised service restarts, and health checks pass.
2. **Given** a production-like Docker host, **when** the optional Docker deployment runs, **then** pinned images build or pull, the same environment contract validates, ParadeDB is backed up and migrated, Compose services restart, and health checks pass.
3. **Given** a previous database backup and environment backup, **when** recovery is rehearsed, **then** the service, authentication, diagnostics, indexed content, and chat smoke test return to a usable state.
4. **Given** expected launch concurrency and indexing volume, **when** load tests run, **then** agreed latency, error, connection-pool, payload, and resource thresholds are met.
5. **Given** network, OpenAI, Groq, ParadeDB, `pg_search`, pgvector, or optional Redis degradation, **when** resilience tests run, **then** failures are bounded, observable, and do not corrupt content or conversation state.
6. **Given** a security and privacy review, **when** release is approved, **then** no critical/high finding remains and retention/deletion procedures are documented.
7. **Given** `pg_search`, a BM25 index, or its direct smoke query fails, **when** rollback guidance is applied, **then** `HYBRID_SEARCH_ENABLED=false` is retained, vector-only search remains available, and the failure is included in the final report.
8. **Given** a native or Docker deployment candidate, **when** its `pg_search` artifact is inspected, **then** the final report records the running PostgreSQL major, execution OS/release, CPU architecture, package/extension version, image digest when applicable, and the compatibility decision.
9. **Given** any package mismatch, **when** deployment reaches the hybrid gate, **then** the rollout does not install or force-load that package, does not run BM25 migrations, and does not continue with hybrid enabled.

**Tasks**

- [ ] Create native Bun/systemd, ParadeDB, Redis, reverse-proxy, HTTPS, environment, deployment, and log-management instructions.
- [ ] Create an optional production multi-stage Dockerfile, Docker Compose stack, pinned ParadeDB/API images, private networks, persistent volumes, reverse-proxy, HTTPS, and secret/environment templates.
- [ ] Automate backup-before-migration, health verification, smoke tests, and rollback guidance.
- [ ] Configure logs, metrics, dashboards, alerts, and slow-query monitoring.
- [ ] Load-test chat, indexing batches, ParadeDB BM25, pgvector, hybrid fusion, and database pooling.
- [ ] Test retries, duplicate delivery, process restarts, partial failures, timeouts, and dependency degradation.
- [ ] Review authentication, CORS, SSRF, SQL injection, prompt injection, metadata limits, logging, and rate limits.
- [ ] Rehearse volume/database restore, environment restore, provider-key rotation, rollback, and reindex coordination.
- [ ] Follow the referenced Codex deployment runbook's backup-first, preserve-secrets, extension verification, stop-before-incompatible-migration, BM25 smoke-query, fallback, and final-report rules in both native and Docker procedures.
- [ ] Implement and rehearse the exact PostgreSQL-major, execution-OS/release, and CPU-architecture package gate for native hosts and database containers.
- [ ] Record the package compatibility decision and effective hybrid mode in health, diagnostics, and the deployment report.
- [ ] Keep the Hybrid Search Plan and Semantic Search Architecture and Flow Guide current with the implemented migration, retrieval, fallback, and deployment behavior.
- [ ] Complete the production-readiness checklist and record the release baseline.

**Dependencies:** SV-US-012  
**Priority:** Must have

## Recommended Story Order

1. SV-US-001 → SV-US-004: service, database, authentication, and retrieval policy.
2. SV-US-005 → SV-US-007: complete content-ingestion path.
3. SV-US-008 → SV-US-009: retrieval, ranking, and citations.
4. SV-US-010 and SV-US-011: durable conversation and grounded chat.
5. SV-US-012 → SV-US-013: operations, security, resilience, and release.

## Related Specifications

- [`SERVER_APP_ARCHITECTURE.md`](../server/SERVER_APP_ARCHITECTURE.md)
- [`HYBRID_SEARCH_PLAN.md`](../server/HYBRID_SEARCH_PLAN.md)
- [`SEMANTIC_SEARCH_ARCHITECTURE_AND_FLOW_GUIDE.md`](../server/SEMANTIC_SEARCH_ARCHITECTURE_AND_FLOW_GUIDE.md)
- [`SERVER_DATABASE_SCHEMA.md`](../server/SERVER_DATABASE_SCHEMA.md)
- [`SERVER_REST_API_CONTRACT.md`](../server/SERVER_REST_API_CONTRACT.md)
- [`DATA_AND_RAG_DESIGN.md`](../shared/DATA_AND_RAG_DESIGN.md)
- [`SETUP_AND_OPERATIONS.md`](../shared/SETUP_AND_OPERATIONS.md)
