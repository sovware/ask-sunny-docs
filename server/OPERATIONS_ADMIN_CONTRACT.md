# Operations And Admin Contract

## 1. Scope

This contract is normative for SV-US-012. It fixes admin authentication, diagnostics, privacy-safe
usage aggregation, reindex coordination, operational metrics, and alert guidance. It does not make
the backend an editorial source of truth: WordPress still re-sends content for every reindex.

## 2. Admin Authentication

Admin routes accept either:

- an active `api_keys.key_type=admin` bearer key whose server-owned metadata contains the required
  `admin:read` or `admin:write` scope; or
- an unexpired opaque admin session created by `POST /admin/sessions` and sent as a bearer token.

The launch provisioning route continues to create only `wordpress_installation` keys with exactly
the four installation scopes. A valid installation key on an admin route returns the same generic
`403 forbidden` as any authenticated wrong-scope key and does not fall back to session parsing.
Malformed, unknown, hash-mismatched, expired, disabled-user, and revoked credentials return the
generic `401 authentication_error`. Only successful authorization updates key/session last-use.

`POST /admin/sessions` is the bootstrap login boundary. Its strict body contains `email` and
`password`; it validates against `ASK_SUNNY_ADMIN_EMAIL` and `ASK_SUNNY_ADMIN_PASSWORD` using
constant-time comparisons, upserts the configured admin user with a one-way password hash, and
returns a new opaque token once with `expires_at`. Only the token digest is stored. Login failures
never reveal which field differed and never log credentials. Sessions expire after
`ASK_SUNNY_ADMIN_SESSION_TTL_SECONDS`; creating a session deletes expired sessions for that user.

The session token format is `ask_admin_session_<43 base64url characters>`. Admin API-key creation
and rotation remain an operator-controlled secret-management operation outside the public HTTP API.

Read routes require `admin:read`; reindex creation requires `admin:write`.

## 3. Diagnostics

`GET /admin/diagnostics` returns safe current operational state:

- deployment mode and service version;
- database, Redis, and pool total/idle/waiting state;
- running PostgreSQL major, `pg_search`/vector extension versions, preload state, required BM25
  indexes, latest migration, direct smoke result, and requested/effective hybrid state/reason;
- the exact configured package compatibility evidence and verified/mismatch status;
- selected generation provider/model and independent embedding provider/model/dimensions, never
  credentials or URLs;
- authoritative allowlist keys/version/update time;
- active content counts by stored data source key/kind, last indexed time, and latest safe indexing
  outcome;
- latest reindex coordination record.

Diagnostics are read-only and bounded. Dependency probe failures return the same schema with safe
`error`/`unavailable` states and do not expose SQL or exception text.

## 4. Usage

`GET /admin/usage` requires RFC3339 `from` and `to`, with `from < to`, a maximum inclusive range of
92 days, and optional `event_type` from the stored server event taxonomy. Unknown parameters or
event types return `400 validation_error`.

The response contains `from`, `to`, optional filter, totals, and UTC daily buckets. Totals include:

- chat turns, indexing/mutation events, and retrieval events;
- successes, errors, average and p95 latency;
- input/output tokens and retrieval result count;
- vector, BM25, and fused candidate counts;
- vector-only fallback count and error counts by stable `error_code`.

Daily buckets contain only date, event counts, errors, average latency, tokens, retrieval count, and
fallback count. Aggregation reads the existing safe `usage_events` columns/metadata. It never
selects or returns conversation message content, tool arguments/results, visitor identities,
source/result identities, query/filter text, raw metadata, provider name/model, or provider state.
Provider identity remains runtime diagnostic/ephemeral metric context and is not added to usage rows.

## 5. Reindex Coordination

Migration SV-US-012 adds `reindex_jobs` with server UUID, requested source keys, force flag,
`status=awaiting_wordpress`, safe correlation ID, requester kind, timestamps, and bounded safe
metadata. It has no content payload, credential, provider field, query, SQL, or embedding.

`POST /admin/reindex` accepts a strict body containing canonical `data_source_keys` and `force`.
Keys must be a non-empty subset of currently stored data-source descriptors and are bounded by
`MAX_ALLOWED_DATA_SOURCE_KEYS`; they do not have to be currently enabled because reindex can repair
a retained source. The route inserts one record and returns `202` with `ok`, `job_id`,
`status=awaiting_wordpress`, requested keys, and `created_at`.

`GET /admin/reindex/:job_id` requires `admin:read` and returns that same bounded record or a generic
`404 reindex_job_not_found`. `GET /admin/diagnostics` exposes the latest record. No backend worker
claims to rebuild WordPress content. The administrator/plugin uses the record as coordination,
causes WordPress to re-send eligible source-of-truth payloads through normal idempotent content
routes, and verifies indexing/usage state. Completion mutation is deferred until a WordPress-owned
reporting contract exists; launch status therefore remains honestly `awaiting_wordpress`.

## 6. Metrics And Correlation

The runtime exposes safe metrics through diagnostics/usage and structured logs:

- request correlation, route, status, latency, and stable error code;
- job ID/correlation and requested source count;
- chat turn count/latency/tokens/outcome;
- indexing outcome, embedding latency/attempts, and source kind;
- retrieval mode, degradation, vector/BM25/fused counts and latency;
- database pool total/idle/waiting counts;
- selected generation and embedding provider/model only in ephemeral diagnostics/log metric context.

Never log messages, queries, filters, tool arguments/results, visitor identity, source/result identity,
credentials, provider bodies, or database URLs. An observability write failure must not change the
underlying chat, retrieval, indexing, or reindex result.

## 7. Alert Baseline

Operations guidance defines launch alerts for:

- `/health` not ready for 2 minutes;
- p95 chat latency above the WordPress proxy budget or chat errors/timeouts above 5% for 5 minutes;
- provider or embedding rate-limit/error spikes above 5% for 5 minutes;
- `bm25_runtime_error` or hybrid requested-but-ineffective on any production probe;
- database pool waiting count above zero for 5 minutes or utilization above 80%;
- latest successful content indexing older than the installation's agreed freshness window;
- reindex jobs remaining `awaiting_wordpress` beyond 30 minutes.

Threshold changes are versioned operations decisions. Alerts use correlation/job IDs and safe
aggregate state, not private payloads.

## 8. Verification

SV-US-012 requires unit, HTTP, and live PostgreSQL evidence for session hashing/expiry, admin API
key and session authorization, wrong-scope installation denial, diagnostics success/degradation,
bounded usage date/event validation, exact aggregates/daily buckets, privacy projections, reindex
validation/persistence/status lookup, safe correlation, pool metrics, provider-neutral rows, and
restart durability. Normal formatting, type, contract, security, Docker, and pinned database gates
remain mandatory.
