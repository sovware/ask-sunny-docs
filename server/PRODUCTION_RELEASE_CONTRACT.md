# Production Release Contract

## 1. Scope And Release Evidence

This contract is normative for SV-US-013. It fixes the native and optional Docker release sequence,
backup/restore rehearsal, smoke and load baselines, resilience/security approval, compatibility
fallback, and final report. A release candidate is one immutable Git commit plus its migration set,
environment-key contract, deployment configuration, and, for Docker, immutable image digests.

Every automated action defaults to inspection or validation. A command that stops a service,
migrates, restores, replaces an environment file, or changes traffic requires an explicit deployment
mode/confirmation and operates only on paths/services passed by the operator. No release command
prints secret values, creates a production `.env` from placeholders, or silently overwrites an
existing environment.

The final report is a version-controlled, secret-free JSON artifact. It records release commit,
deployment mode, UTC rehearsal time, backup artifact checksums, migration versions, health/smoke
outcomes, load measurements, resilience/security results, residual risks, and the package evidence
and compatibility decision defined below. Test fixtures and CI reports are release evidence, not a
claim that an uninspected customer production host was deployed.

## 2. Environment, Backup, And Migration Boundary

The release validator compares `.env.example` keys with the secret-managed environment and reports
only missing/unknown setting names. It preserves every existing line/value and fails before install,
backup, stop, migration, or restart when configuration is invalid. Environment backups use mode
`0600`, are written outside the repository, and are verified by non-secret SHA-256 checksum. Restore
requires an explicit source, destination, and confirmation; it first preserves the current file and
never prints either file.

Before any extension/schema change, create a PostgreSQL custom-format dump with the deployment's
matching `pg_dump`, prove it is non-empty, validate it with `pg_restore --list`, calculate SHA-256,
and rehearse it into a disposable database or disposable Compose volume. A volume snapshot is only
an additional safeguard. Stop application code before an incompatible migration, apply immutable
checksum-tracked migrations once, and never improvise a down migration or mark a failed version
manually.

The native sequence is:

1. resolve the immutable release commit; install Bun dependencies with the frozen lockfile;
2. inspect/validate the existing environment without changing secrets;
3. create and verify database and environment backups;
4. retain `HYBRID_SEARCH_ENABLED=false` and collect compatibility evidence;
5. stop the supervised API before incompatible migration, migrate, and run checks;
6. restart/enable the supervised API, verify HTTPS health, and run release smokes;
7. enable hybrid only after every compatibility/readiness/direct-query gate passes; and
8. write the final report and preserve rollback coordinates.

The Docker sequence is the same safety boundary, using immutable API, ParadeDB, proxy, and optional
Redis image identities; private networks; persistent volumes; and TLS material mounted read-only.
Only the HTTPS proxy publishes a production port. The database container supplies `pg_dump`, restore,
PostgreSQL/OS/architecture, package, extension, and image-digest evidence. Compose/container startup
is not readiness; HTTPS health and smokes must pass.

## 3. Compatibility And Hybrid Gate

For native candidates, inspect the PostgreSQL execution host. For Docker candidates, inspect inside
the running database container and record the immutable database image digest. The final report
contains:

- running PostgreSQL major from `SHOW server_version_num`;
- execution OS ID and release/codename;
- execution CPU architecture;
- expected package PostgreSQL major, OS ID/release, architecture, and pinned package version;
- installed `pg_search` and vector extension versions;
- `shared_preload_libraries` state and required BM25 index/direct-smoke outcomes;
- immutable image digest for Docker; and
- `verified` or `blocked` plus the stable mismatch reason and effective hybrid mode.

Any missing, ambiguous, unsupported, or mismatched fact stops before package installation/force-load
and before extension/BM25 migrations. The report records the blocker,
`HYBRID_SEARCH_ENABLED=false`, and `effective_mode=vector_only`. The same fallback applies when
`pg_search`, any BM25 index, or a direct scored smoke query fails. Existing vector storage and
vector-only application smoke must remain usable; operators restore the verified backup instead of
forcing an incompatible artifact.

## 4. Recovery Rehearsal And Smokes

Recovery is rehearsed against an isolated production-like native database/host or pinned Compose
stack. It restores the database dump and environment backup, starts the selected dependency shape,
revalidates configuration and compatibility, then proves:

1. `/health` is usable in the reported effective mode;
2. the previously issued installation credential still authorizes and a wrong/old credential does
   not gain access;
3. admin authentication and safe diagnostics work;
4. known indexed content remains retrievable with direct BM25/vector and application retrieval
   checks as permitted by the effective mode;
5. a deterministic grounded `admin_test` chat returns a citation to restored evidence and persists
   a provider-neutral turn; and
6. a tracked reindex request remains coordination for WordPress resend.

Production provider-key rotation is rehearsed by validating the replacement secret, restarting,
running a smoke, and revoking the old secret through the provider/secret manager. CI uses explicit
non-secret deterministic provider fixtures and never stores production keys.

## 5. Launch Load Baseline

The minimum launch host remains 2 CPU cores and 2 GiB RAM with `PG_POOL_MAX=10`. The deterministic
release harness uses a production build, pinned PostgreSQL/ParadeDB, fixed provider/embedding
fixtures, warmed search indexes, and these exact workloads:

- 8 simultaneous complete grounded chat turns;
- 4 simultaneous maximum-default batches of 50 idempotent indexing items;
- 200 scoped direct BM25, vector, and application hybrid retrieval iterations; and
- an oversized request at the configured maximum boundary.

The baseline passes only when:

- unexpected/error responses are 0% and duplicate delivery produces no corrupt/duplicate state;
- grounded chat p95 is at most 2,000 ms with the deterministic provider fixture;
- a 50-item indexing batch p95 is at most 5,000 ms with deterministic embeddings;
- direct BM25/vector and application hybrid retrieval p95 are each at most 500 ms;
- database pool waiting is zero at completion and never exceeds 2 during the workload;
- process/container peak memory remains below 1 GiB and no dependency exceeds its configured CPU or
  memory limit when limits are applied; and
- an oversized payload is rejected with `413` before content, embedding, or database work.

Real-provider latency is evaluated separately against `AI_REQUEST_TIMEOUT_MS=45000` and the
60-second WordPress proxy budget; it does not weaken the deterministic application baseline. A
customer workload above these assumptions requires a new measured capacity decision rather than
silently increasing pool, payload, or concurrency limits.

## 6. Resilience Approval

Automated tests cover network refusal/timeout and bounded retries; OpenAI and Groq selected-adapter
failure without automatic cross-provider switching; embedding retry/permanent failure with prior
content preservation; PostgreSQL/readiness failure; missing/wrong `pg_search`, vector, migration, and
BM25 indexes; direct-smoke failure; request-local BM25 vector-only fallback; vector failure that never
becomes BM25-only semantic success; optional Redis failure with durable PostgreSQL continuation;
duplicate WordPress delivery; process restart with durable credentials, content, conversations,
checkpoints, and reindex records; and observability-write failure that cannot change durable results.

Every failure has a bounded timeout/retry or fail-closed behavior, a stable public error/degradation
code, safe correlation, and a no-corruption assertion. The final report records every tested failure
and any unavailable real external dependency as a residual risk rather than claiming it passed.

## 7. Security And Privacy Approval

Release approval requires `bun audit` with no critical/high finding and automated/manual evidence
for credential hashing/rotation/scope, admin-session expiry, no permissive CORS at the private API,
reverse-proxy request limiting, HTTPS/security headers, SSRF-safe no-fetch source URLs, parameterized
SQL/server-owned filter paths, prompt-injection isolation, strict body/metadata/batch/tool limits,
log redaction, non-root/read-only container surfaces where applicable, private database/Redis/API
ports, and secret-free images/repository/reports.

The review also proves conversation deletion, external-user anonymization, retention scheduling,
backup retention/access, and restore-time privacy behavior. No critical/high item may be waived.
Medium/lower findings need an owner, due date, mitigation, and final-report entry. Proxy rate limits
use a per-client launch baseline of 5 authentication attempts per minute and 600 general API
requests per minute, returning 429 without forwarding rejected request bodies.

## 8. Release Baseline Completion

SV-US-013 completes only when native and Docker procedures are executable and safely testable,
Docker recovery and compatibility/fallback are rehearsed, the launch load baseline passes, the
resilience and security matrices have no critical/high gap, all existing checks remain green, the
versioned final report/checklist contains no secrets, GitHub CI passes, and the story PR is
squash-merged. Native service-manager execution on the actual target and real-provider smokes remain
explicit deployment-time sign-off items when that infrastructure is not available to CI.
