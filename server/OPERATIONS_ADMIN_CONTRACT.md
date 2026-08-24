# Operations And Admin Contract

## 1. Scope

This contract is normative for SV-US-012. It fixes admin authentication, diagnostics, reindex
coordination, operational metrics, and alert guidance. It does not make
the backend an editorial source of truth: WordPress still re-sends content for every reindex.

## 2. Admin Authentication

Admin routes accept an active `api_keys.key_type=admin` bearer key whose server-owned metadata
contains the required `admin:read` or `admin:write` scope.

The launch provisioning route creates only `website` keys with the
server-defined installation scopes. A valid website key on an admin route returns `403 forbidden`.
Malformed, unknown, hash-mismatched, and revoked credentials return the generic
`401 authentication_error`. Only successful authorization updates key last-use.

`POST /auth/admin` is the admin-key provisioning boundary. Its strict body contains `username` and
`password`; it validates against `ASK_SUNNY_ADMIN_USERNAME` and `ASK_SUNNY_ADMIN_PASSWORD` using
constant-time comparisons, then creates an `api_keys.key_type=admin` credential owned by the
normalized username with fixed `admin:read` and `admin:write` scopes. The plaintext API key is
returned once, and only its prefix and SHA-256 digest are persisted. Login failures never reveal
which field differed and never log credentials. There are no admin-user or admin-session tables.

Read routes require `admin:read`; reindex creation requires `admin:write`.

WordPress installation operations are a separate boundary. Active `website` keys receive
`operations:read`; this scope authorizes the website projection from `GET /system/diagnostics` and
provider configuration through `POST /system/provider`. It does not authorize any `/admin/*` route.

## 3. Diagnostics

`GET /system/diagnostics` with an `admin:read` key returns safe current operational state:

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

## 4. Website Diagnostics Projection

`GET /system/diagnostics` with a website key reuses the operational probes but explicitly projects only service
version and dependency status; selected AI and embedding configuration; ParadeDB, vector, BM25, and
requested/effective hybrid state; retrieval-configuration version/update time; content counts; and
the latest safe indexing time/outcome. It excludes credentials, URLs, pool internals, package paths,
raw errors, deployment secrets, visitor/conversation data, and administrative controls.

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
`404 reindex_job_not_found`. The admin projection of `GET /system/diagnostics` exposes the latest record. No backend worker
claims to rebuild WordPress content. The administrator/plugin uses the record as coordination,
causes WordPress to re-send eligible source-of-truth payloads through normal idempotent content
routes, and verifies indexing/usage state. Completion mutation is deferred until a WordPress-owned
reporting contract exists; launch status therefore remains honestly `awaiting_wordpress`.

## 6. Metrics And Correlation

The runtime exposes safe metrics through diagnostics and structured logs:

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
