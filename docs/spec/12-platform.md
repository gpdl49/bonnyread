# 12. Platform

This file specifies the runtime kernel under `internal/platform/`: the database layer, configuration, process lifecycle, the durable job queue, the scheduler, leases and row claims, the event bus and change log, notifications, webhooks, custom scripts and alert sinks, health checks, metrics, logging and tracing, housekeeping, backup and restore, the outbound HTTP client with proxies and challenge solvers, the shared rate limiter, the regex wrapper, and the WASM plug-in host. Domain areas plug into these through `registry.Module` (`01-architecture.md` §4) and choose their own job kinds, keys, events and settings; this file defines only the mechanisms and their guarantees. Applies from P1, except the plug-in host, which is required no later than P2 (its first consumer is subtitles).

## Concepts

| Term | Definition |
|---|---|
| **Job** | A row in `job`: a typed unit of background work with a kind, JSON args, queue, priority and state. |
| **Job kind** | A registered Go type with `Kind() string`, a handler and a `KindOpts` policy. |
| **Queue** | A named worker pool per node (`default`, `io`, `metadata`, `plugin`, `script`). |
| **Dedupe key** | A string that collapses duplicate pending (or pending-or-running) jobs for the same subject. |
| **Concurrency key** | A string that allows at most one *running* job per key, cluster-wide. |
| **Attempt** | One claim of a job; `(job.id, attempt)` is the ownership token used for fencing. |
| **Snooze** | Rescheduling a job without consuming an attempt. |
| **Schedule row** | A row in `schedule` that enqueues a job when due. |
| **Lease** | A row in `lease` giving one node ownership of a long-lived connection, with a monotonic fencing token. |
| **Row claim** | A short-lived cross-node critical section (seconds) held as a conditional-update claim on the shared row it protects, for single-flight operations (PLT-062). |
| **After-commit listener** | An in-process, best-effort event subscriber run after the publishing transaction commits. |
| **Durable subscriber** | An event subscriber delivered as a job enqueued in the publishing transaction. |
| **Change log** | The `event_log` table of UI-visible change notices, retained for SSE replay. |
| **Doorbell** | A best-effort Postgres `NOTIFY` that wakes other nodes; carries nothing that must be delivered. |
| **Rate key** | A named limiter subject (`acq:indexer:<id>`, `source:musicbrainz`, …) with spacing, quotas, lanes and cooldown. |
| **Lane** | A traffic class within a rate key (for example `interactive`, `rss`, `backlog`) with reserved or capped share. |
| **Egress** | A configured outbound route: direct, HTTP CONNECT proxy or SOCKS5 proxy. |
| **Solver** | A FlareSolverr-compatible challenge-solving service (FlareSolverr, Byparr). |
| **Plug-in** | A signed or admin-approved WebAssembly module with a manifest, run by the host under capability grants. |
| **Plug-in configuration** | One configured instance of a plug-in (for example one subtitle provider entry with its credentials). |

Aliases: the API (`11-api-and-clients.md`) shows job states as `queued|running|succeeded|failed|cancelled`; internally `available` = `queued` and `completed` = `succeeded`. `design/runtime.md` names health endpoints `/healthz/live` and `/healthz/ready`; this specification uses `/livez` and `/readyz` (the API design's names).

## Requirements

### 1. Database layer

**PLT-001** (MUST) The data layer **MUST** use ent with Atlas versioned migrations generated per dialect from one ent schema. Supported dialects are PostgreSQL ≥ 15 and SQLite only. _Source: D13 as amended by D20; open-issues B5._

**PLT-002** (MUST) The SQLite driver **MUST** be `modernc.org/sqlite` (pure Go), embedded in every build, so that builds stay `CGO_ENABLED=0`. The same library serves the PCD decoder (`07-profiles.md`). _Source: open-issues A18._

**PLT-003** (MUST) On PostgreSQL the core **MUST** use one `pgxpool.Pool` shared by ent (via `stdlib.OpenDBFromPool`), plus one separately held session connection for `LISTEN` (from `DB_LISTEN_DSN`, default `DB_DSN`). At startup the core **MUST** verify `server_version_num ≥ 150000` and refuse to start otherwise. _Source: design/runtime.md §3.3; D20._

**PLT-004** (MUST) ent **MUST** be built with the `sql/lock`, `sql/upsert` and `sql/execquery` features. Hot statements (claim, heartbeat, reaper, slot reservation) **MAY** be raw SQL through `execquery`, one file per dialect behind one Go interface. _Source: design/runtime.md §3.3–3.4._

**PLT-005** (MUST) On SQLite the core **MUST** open two ent clients: `RW` with one connection, `_txlock=immediate`, `journal_mode=WAL`, `busy_timeout=5000`, `foreign_keys=ON`, `synchronous=NORMAL`; and `RO` with N connections and `query_only=1`. On PostgreSQL `RO` and `RW` are the same client. _Source: design/runtime.md §3.3._

**PLT-006** (MUST) Every coordination timestamp (claims, heartbeats, lease expiry, schedule due times, rate slots, cooldowns) **MUST** be written and compared with the database's `now()`. _Source: design/runtime.md N2._

**PLT-007** (MUST) Every ent edge **MUST** declare its delete behaviour (`ON DELETE CASCADE`, `SET NULL` or `RESTRICT`); schema review **MUST** reject an edge without one. Orphan-cleanup housekeepers **MUST NOT** be used where a foreign key can express the rule. _Source: design/runtime.md §3.9._

**PLT-008** (MUST) Migrations **MUST** run under a migration lock: `pg_advisory_lock(<constant>)` on a dedicated connection on PostgreSQL; the data-directory lock on SQLite. Advisory locks **MUST NOT** be used for any other purpose (short single-flight sections use row claims, PLT-062). _Source: design/runtime.md §3.2, §3.5._

**PLT-009** (MUST) The singleton `schema_meta` row **MUST** record `schema_version` (last applied migration) and `min_binary_version`. A contract (destructive) migration **MUST** raise `min_binary_version` to the first release that no longer uses the removed shape. A binary whose version is below `min_binary_version` **MUST** refuse to start and report unready; a binary whose embedded migrations are behind `schema_version` **MUST** start (N-1 compatibility, `01-architecture.md` §8). _Source: open-issues A7; design/runtime.md §3.2 (amended for N-1)._

**PLT-010** (MUST) CI **MUST** check that each release's migrations are expand-only unless they are marked as contract migrations that raise `min_binary_version`, and **MUST** run the full platform conformance suite (claim, reaper, leases, schedules, rate limiter, change log) against PostgreSQL 15 and SQLite. _Source: open-issues A7; design/runtime.md §3.4._

**PLT-011** (MUST) Whether migrations run at node startup **MUST** be controlled by the bootstrap key `MIGRATE` (`auto` by default, or `off`; `13-deployment.md` DEP-061). With `off`, a node with pending migrations **MUST** wait unready until they are applied externally (`bonnyread migrate up`, for example a Helm hook job, `13-deployment.md`). _Source: design/runtime.md §3.2; design/identity-api-deployment.md §3.10._

**PLT-012** (MUST) On SQLite, before applying pending migrations, the core **MUST** run `PRAGMA quick_check` and take a pre-migration backup with `VACUUM INTO`. On PostgreSQL it **MUST** log a warning that the operator's backup is the safety net. _Source: design/runtime.md §3.2; I11._

**PLT-013** (MUST NOT) Read replicas **MUST NOT** be used in v1. _Source: design/runtime.md §3.3._

### 2. Configuration

Two layers with no overlapping keys: bootstrap configuration (needed before the database opens, or per node) and typed runtime settings in the database.

**PLT-014** (MUST) Bootstrap configuration **MUST** be read from, in precedence order: command-line flags, `BONNYREAD_<KEY>` environment variables, the bootstrap file, built-in defaults. The file is TOML at `<DATA_DIR>/config.toml` unless `--config` names another path [OPN-135]. _Source: open-issues A8; D20; design/identity-api-deployment.md §3.10; design/runtime.md §3.10._

**PLT-015** (MUST) Every bootstrap key **MUST** also accept `BONNYREAD_<KEY>_FILE`, naming a file whose content (trailing newlines trimmed) is the value. Setting both forms **MUST** be a startup error. _Source: design/runtime.md I12, §3.10._

**PLT-016** (MUST) The bootstrap keys **MUST** include at least:

| Key | Default | Meaning |
|---|---|---|
| `DATA_DIR` | OS-specific (`/data` in containers) | durable per-installation directory (SQLite DB, `config.toml`, TLS files, backups) |
| `STATE_DIR` | `<DATA_DIR>/state` | node-local disposable state: lock file, WASM compilation cache, temp, extraction scratch |
| `DB_DSN` | `sqlite://<DATA_DIR>/bonnyread.db` | database DSN (secret) |
| `DB_LISTEN_DSN` | `DB_DSN` | session-level DSN for `LISTEN` (PostgreSQL) |
| `DB_MAX_CONNS` | 20 (PostgreSQL) | pool size |
| `MIGRATE` | `auto` | `auto` runs migrations at startup; `off` never migrates (§1) |
| `HTTP_ADDR` | `127.0.0.1:7979` desktop, `:7979` container | main listener |
| `ADMIN_ADDR` | off | optional admin listener (`/metrics`, `/metrics/cluster`, `/livez`, `/readyz`, and `/debug/pprof` when `PPROF=true`) |
| `PPROF` | `false` | serve `/debug/pprof` on the admin listener (requires `system:manage`) |
| `BASE_URL` | none | external URL; its path is the sub-path the core is served under |
| `TRUSTED_PROXIES` | empty | CIDRs whose `X-Forwarded-*` headers are honoured |
| `TLS_MODE`, `TLS_CERT_FILE`, `TLS_KEY_FILE` | `off` | `off` \| `self-signed` \| `files` |
| `LOG_LEVEL`, `LOG_FORMAT`, `LOG_FILE` | `info`, `json` (non-TTY) / `text` (TTY), none | logging (§13) |
| `NODE_ID` | hostname + persisted random suffix | stable node identity |
| `ROLES` | `api,worker` | node roles |
| `READINESS_DRAIN`, `SHUTDOWN_GRACE` | `0s`/`25s` single node; `5s`/`25s` Kubernetes | shutdown timing (§3) |
| `ALLOW_SCRIPTS` | `true` | deployment-wide switch for script targets |
| `PLUGINS_ALLOW` | `signed` | ceiling for third-party plug-ins: `none` \| `signed` \| `any` |
| `OTEL_*` | unset | standard OpenTelemetry variables (§14) |

Identity bootstrap keys (bootstrap owner, OIDC seed, argon2 parameters) are specified in `10-identity-and-access.md`. _Source: design/runtime.md §3.10; design/identity-api-deployment.md §3.10._

**PLT-017** (MUST NOT) A value that can live in the database **MUST NOT** be read from bootstrap configuration, except through settings pinning (below). _Source: D16; open-issues A8._

**PLT-018** (MUST) Runtime settings **MUST** be typed Go structs registered per namespace through `Registrar.Settings`. Struct tags **MUST** supply the default, validation bounds, a `secret` flag and a description; the UI form and the settings JSON Schema **MUST** be generated from them. _Source: design/runtime.md §3.10._

**PLT-019** (MUST) Settings **MUST** be stored in `setting(namespace, key, value JSON, version, updated_by, updated_at)` with optimistic concurrency on `version`. _Source: design/runtime.md §3.10._

**PLT-020** (MUST) Each node **MUST** hold an immutable settings snapshot behind an atomic pointer. A change **MUST** publish `settings.changed`; every node **MUST** reload its snapshot on the doorbell and, independently, at least every 30 s. A setting **MUST NOT** require a restart: subsystems read the snapshot per operation or subscribe to changes (for example resizing worker pools). _Source: design/runtime.md §3.10; open-issues A4 (poll backstop)._

**PLT-021** (MUST) A setting **MAY** be pinned by `BONNYREAD_SET__<NAMESPACE>__<KEY>=<json>` (and `_FILE`). A pinned setting overrides the database value, **MUST** be reported by the API as `managed: env` and **MUST** be read-only through the API. _Source: design/runtime.md §3.10._

**PLT-022** (MUST) Secret settings and secret entity fields **MUST** be write-only through the API (`11-api-and-clients.md`), redacted in logs and errors, and stripped from sanitised exports. _Source: D16; design/runtime.md §3.10._

### 3. Registration and process lifecycle

**PLT-023** (MUST) `registry.Registrar` **MUST** provide:

| Method | Registers |
|---|---|
| `Job(kind, handler, KindOpts)` | handler plus default queue, priority, timeout, retry policy, dedupe policy, required tools |
| `Schedule(key, sched.Default)` | a system schedule with default spec and bounds |
| `Event(type, events.Meta)` | an event type: notifiable flag, audience rule, JSON Schema, default template |
| `Settings(namespace, struct)` | a typed settings namespace |
| `Health(check)` | a health check |
| `Housekeeper(name, func)` | a housekeeping function |
| `Provider(kind, impl, factory)` | an indexer, download client, notification target, metadata source, subtitle provider, … |
| `Metrics(collector)` | a Prometheus collector |

Duplicate names **MUST** fail startup. _Source: design/runtime.md §3.1._

**PLT-024** (MUST) Startup **MUST** run these steps in order, each blocking the next:
1. **Config:** parse and validate bootstrap configuration; derive `node_id`.
2. **Storage guard:** on SQLite, take an exclusive `flock` on `<DATA_DIR>/bonnyread.lock`; if held, exit with "another instance is running".
3. **DB open:** pools as in §1; version check on PostgreSQL.
4. **Migrate or verify:** §1.
5. **Recover (SQLite only):** return every `running` job to `available` immediately, without decrementing its attempt; if the unclean-shutdown marker exists, run `PRAGMA quick_check` and raise a health issue on failure.
6. **Build:** construct platform services, register modules, freeze the registry, load the settings snapshot, detect native tools, compile enabled plug-ins.
7. **Start** (dependency order): observability → node heartbeat → cluster listener (PostgreSQL) → event bus and change-log tailer → job workers → lease manager → scheduler loop → HTTP.
8. **Ready:** create the unclean-shutdown marker; `/readyz` turns true only after the HTTP listener is up and, on PostgreSQL, the listener connection is subscribed.

_Source: design/runtime.md §3.2, §4.3._

**PLT-025** (MUST) Each node **MUST** upsert its `node` row (hostname, version, schema version, roles, advertised tools, `started_at`) and heartbeat it every 10 s. A node is **live** when `heartbeat_at` is within 30 s of `now()`. _Source: design/runtime.md §3.5; open-issues A6._

**PLT-026** (MUST) On SIGTERM, SIGINT, a service-manager stop or `POST /api/v1/system/shutdown`, the node **MUST**:
1. make `/readyz` return 503 and wait `READINESS_DRAIN`;
2. stop the scheduler loop and release all leases (`expires_at = now()`, then `NOTIFY bonnyread_lease`);
3. stop claiming; give running jobs the rest of `SHUTDOWN_GRACE`; then cancel their contexts and release each interrupted job with a fenced update to `state='available', run_at=now(), attempt=attempt-1`, followed by `NOTIFY bonnyread_jobs`;
4. close SSE streams with a `retry` hint, then shut down HTTP (in-flight requests bounded by the grace period);
5. delete its per-node schedules and its node-affine jobs that are still `available`; mark the `node` row stopped;
6. on SQLite: `PRAGMA optimize`, `wal_checkpoint(TRUNCATE)`, remove the unclean-shutdown marker, close, release the `flock`.

_Source: design/runtime.md §3.13._

### 4. Durable job queue

**PLT-027** (MUST) The job queue **MUST** be implemented in-repo on ent (tables in the ent schema and Atlas migrations), not by an external queue library with its own migrations. _Source: design/runtime.md §3.4._

**PLT-028** (MUST) `jobs.Enqueue(ctx, tx, args, opts)` **MUST** insert the job within the caller's ent transaction, so the job commits atomically with the business write. After commit the enqueuer **MUST** poke local workers in-process and, on PostgreSQL, send `NOTIFY bonnyread_jobs, '<queue>'`. Job args **MUST** be JSON with a schema version per kind. _Source: design/runtime.md §3.4._

**PLT-029** (MUST) Job states **MUST** be `available → running → completed | failed | cancelled`. A retry or snooze returns the job to `available` with a future `run_at`; there is no separate scheduled state. _Source: design/runtime.md §3.4._

**PLT-030** (MUST) Each kind **MUST** declare a dedupe policy; when `dedupe_key` is set, enqueue inserts `job_dedupe(key, job_id)` and a conflict means "already pending":

| Policy | Dedupe row removed | Use |
|---|---|---|
| `WhilePending` (default) | when the job starts | refreshes (data may have changed since a run began) |
| `WhilePendingOrRunning` | when the job finishes | RSS, polls, scheduled work (`overlap=skip`) |

On conflict the caller **MAY** **promote** the existing job: raise its priority and pull `run_at` earlier. _Source: design/runtime.md §3.4; I2._

**PLT-031** (MUST) Priorities **MUST** be `interactive` (100), `high` (75), `normal` (50) and `background` (10). Claims **MUST** order by `priority DESC, run_at, id`. _Source: design/runtime.md §3.4; I3._

**PLT-032** (MUST) Each node **MUST** run one worker pool per queue, sized by settings. Defaults: `default` 8, `io` 2, `metadata` 4, `plugin` 4, `script` 2 on a server; on a single desktop node `default` and `io` **MUST** default from `runtime.NumCPU` (at least 4 and 1). _Source: design/runtime.md §3.4, §4.3._

**PLT-033** (MUST) A job **MAY** carry a `concurrency_key`. The claim **MUST** insert `job_running_key(key, job_id, claimed_by)` in the claiming transaction; on conflict the job **MUST** stay `available` and the claim move on. This gives cluster-wide mutual exclusion per key. Domain specifications choose the keys (for example `lib:title:<id>` for file operations on a title, `dlclient:<id>`, `pcd:<db_id>`). There **MUST NOT** be a global exclusive-job mechanism. _Source: design/runtime.md §3.4; I4; open-issues A1._

**PLT-034** (MUST) The claim **MUST** select `available` jobs of the worker's queue with `run_at <= now()`, matching `node_affinity` (null or this node), and of a kind that this node has registered and whose required tools it advertises; lock them with `FOR UPDATE SKIP LOCKED` (PostgreSQL) or inside `BEGIN IMMEDIATE` (SQLite); and set `state='running', attempt=attempt+1, claimed_by, started_at, heartbeat_at`. _Source: design/runtime.md §3.4; open-issues A6, A7._

**PLT-035** (MUST) Idle workers **MUST** poll every 2 s with jitter as a fallback to the doorbell. _Source: design/runtime.md §3.4; open-issues A4._

**PLT-036** (MUST) Each node **MUST** heartbeat its running jobs every 10 s in one batched update guarded by `claimed_by`, reading back `cancel_requested`. Every node **MUST** run a reaper every 15 s that returns jobs with `state='running'` and `heartbeat_at < now() - 45 s` to `available` (attempt not decremented) and deletes their `job_running_key` rows. The reaper **MUST** be idempotent and need no leader. _Source: design/runtime.md §3.4._

**PLT-037** (MUST) Ownership of a job attempt is `(job.id, attempt)`. Completion, failure and release updates **MUST** be guarded by `id`, `attempt` and `state='running'`; zero affected rows means ownership was lost, and the result **MUST** be discarded and logged. Handlers that write domain state **MUST** do so through `jobs.FencedTx(ctx, fn)`, which locks the job row with that guard (`SELECT … FOR UPDATE` on PostgreSQL) before running `fn`. _Source: design/runtime.md §3.4; N1._

**PLT-038** (MUST) Handlers **MUST** return one of: success; `jobs.Permanent(err)` (fail, no retry); `jobs.Snooze(d)` (re-run after `d` without consuming an attempt); or a plain error (retry). A panic **MUST** be recovered and treated as a retryable error with its stack logged. _Source: design/runtime.md §3.4._

**PLT-039** (MUST) The default retry policy **MUST** be exponential backoff with full jitter, `min(15 s · 2^attempt, 6 h)`, with `max_attempts` 5 for `background`/`normal`/`high` jobs and 1 for `interactive` jobs. Kinds **MAY** override. _Source: design/runtime.md §3.4._

**PLT-040** (MUST) Each kind **MUST** declare a timeout, enforced with `context.WithTimeout`. If a handler has not returned `timeout + 30 s` after start, the worker **MUST** mark the job failed (fenced), free its slot, increment `bonnyread_job_abandoned_total` and raise a health issue. _Source: design/runtime.md §3.4._

**PLT-041** (MUST) Cancelling an `available` job **MUST** set it `cancelled` directly. Cancelling a `running` job **MUST** set `cancel_requested=true` and send `NOTIFY bonnyread_cancel`, so the owning node cancels the handler's context within about 100 ms (else at the next heartbeat); the job ends `cancelled` when the handler returns `ctx.Err()`. Cancelling a parent job **MUST** cascade to its children. _Source: design/runtime.md §3.4._

**PLT-042** (MUST) `jobs.Progress(ctx, message, pct)` **MUST** be throttled to one write per second per job, stored in `job.progress`, and published as an ephemeral change (§7). Each job **MUST** keep a run log of its last 200 lines, stored with the job. _Source: design/runtime.md §3.4._

**PLT-043** (MUST) Bulk actions **MUST** be expressible as a parent job with child jobs; the parent's progress aggregates its children's. _Source: design/runtime.md §3.4._

**PLT-044** (MUST) Finished jobs **MUST** be retained, configurably: completed 7 days, cancelled 7 days, failed 14 days [OPN-069]; housekeeping deletes them in batches. _Source: design/runtime.md §3.4; open-issues Group C (runtime)._

**PLT-045** (MUST) Each job **MUST** record `trigger` (`manual|schedule|event|system`), `created_by_user` when manual, and the W3C `traceparent` of the enqueuing context. _Source: design/runtime.md §3.4, §3.11._

### 5. Scheduler

**PLT-046** (MUST) Schedules **MUST** be rows in `schedule`, from two sources: (1) **system schedules** declared by modules, upserted at boot without overwriting user overrides or moving an existing `next_run_at`; (2) **per-entity schedules** created, updated and deleted by domain services in the same transaction as their owner entity, and deleted with it. _Source: design/runtime.md §3.5.1; open-issues A1._

**PLT-047** (MUST) Every `worker` node **MUST** run the scheduler loop. The loop sleeps until the earliest `next_run_at`, capped at 30 s and woken early by `NOTIFY bonnyread_sched`; then claims up to 50 due rows (`enabled AND next_run_at <= now()`, ordered by `next_run_at`, `FOR UPDATE SKIP LOCKED` on PostgreSQL) and, per row in one transaction, enqueues `job_kind(args)` with dedupe key `sched:<schedule id>` and sets `next_run_at = next(spec, now()) + jitter`, `last_enqueued_at`, `last_job_id`. _Source: design/runtime.md §3.5.1; D20._

**PLT-048** (MUST) The next run **MUST** be computed from `now()`, not from the missed slot, so downtime coalesces into one run. _Source: design/runtime.md I5._

**PLT-049** (MUST) `overlap=skip` **MUST** use dedupe policy `WhilePendingOrRunning`; `overlap=queue_one` **MUST** use `WhilePending`. _Source: design/runtime.md §3.5.1._

**PLT-050** (MUST) Interval schedules **MUST** apply deterministic jitter: `hash(schedule.id) mod (jitter_pct · interval)`, applied once so that the phase is stable. Cron schedules get no jitter unless configured. _Source: design/runtime.md §3.5.1; I6._

**PLT-051** (MUST) `spec_kind` **MUST** be `interval`, `cron` or `manual`. Cron expressions **MUST** be parsed with `netresearch/go-cron` and evaluated in the schedule's time zone (default: the server time-zone setting). A `manual` schedule never becomes due. _Source: design/runtime.md §3.5.1._

**PLT-052** (MUST) "Run now" **MUST** enqueue the schedule's job with `interactive` priority and **MUST NOT** move `next_run_at` unless the kind sets `RescheduleOnManualRun`. _Source: design/runtime.md §3.5.1._

**PLT-053** (MUST) Schedule specs **MUST** be validated at write time against the bounds registered for the kind (for example a minimum RSS interval). _Source: design/runtime.md §3.5.1._

**PLT-054** (MUST) Node-scoped periodic work (node-scoped health checks, node temp cleanup) **MUST** use a per-node schedule that the node creates at start and deletes on clean shutdown, enqueuing jobs with `node_affinity = node_id`. Housekeeping **MUST** delete per-node schedules of nodes not live for 1 day. _Source: design/runtime.md §3.8, §3.13._

### 6. Leases and row claims

**PLT-055** (MUST) Leases **MUST** be used only for long-lived connections (`01-architecture.md` §6): IRC networks (`irc:<network_id>`), file watchers (`watch:<root_id>`) and future long-lived external connections. _Source: open-issues A1; D20._

**PLT-056** (MUST) Acquire **MUST** be `UPDATE lease SET holder_node=$me, token=token+1, acquired_at=now(), renewed_at=now(), expires_at=now()+$ttl WHERE name=$n AND (expires_at < now() OR holder_node=$me) RETURNING token`, after an insert-or-ignore of the row. Renew **MUST** run every `ttl/3`, guarded by `holder_node` and `token`. _Source: design/runtime.md §3.5._

**PLT-057** (MUST) A holder **MUST** stop the leased work (cancel its context) when no renewal has succeeded for `2/3 · ttl`, measured on its monotonic clock. _Source: design/runtime.md §3.5._

**PLT-058** (MUST) Any database write caused by leased work (for example enqueuing an evaluation from an IRC announce) **MUST** run `lease.Fence(tx, name, token)`, which fails the transaction if the lease's token differs. _Source: design/runtime.md §3.5; N1._

**PLT-059** (MUST) Defaults: `ttl` 15 s, renew every 5 s, followers poll free leases every 2 s, and a release sends `NOTIFY bonnyread_lease`. _Source: design/runtime.md §3.5._

**PLT-060** (MUST) A node **MUST** take a free lease only while it holds at most `ceil(total_leases / live_nodes)` leases, so leases spread across nodes without a coordinator. _Source: design/runtime.md §3.5._

**PLT-061** (MUST) On SQLite the lease backend **MUST** be in-memory and always granted, behind the same interface. _Source: design/runtime.md §3.5._

**PLT-062** (MUST) The platform **MUST** provide a row-claim single-flight primitive, `claim.Do(ctx, row, maxHold, fn)`, for short critical sections across nodes that guard one shared row: tracker login on `IndexerSession(indexer_id)` (`03-acquisition.md` ACQ-120) and challenge solving on `solver_clearance(host, egress_id)` (ACQ-124, PLT-123). After an insert-or-ignore of the row, a claim **MUST** be one conditional update `SET lock_holder = <node_id>:<nonce>, lock_until = now() + maxHold WHERE <key> AND (lock_holder IS NULL OR lock_until < now())` with `maxHold` defaulting to 60 s; zero affected rows means another node holds the claim, and the caller **MUST** wait, re-trying with backoff, until the claim is released or expires or its own deadline ends. The holder **MUST** re-read the row after claiming and skip the work if another node already completed it, and **MUST** release with an update guarded by `lock_holder`. The same SQL runs on SQLite (single writer). Database advisory locks (reserved for the migration lock, PLT-008) and leases (PLT-055) **MUST NOT** be used for this. [OPN-103] _Source: design/acquisition.md §3.10, §3.15; design/runtime.md §3.5 (advisory locks only for the migration lock); OPN-103 resolved in favour of ACQ-120/ACQ-124._

### 7. Event bus and change log

**PLT-063** (MUST) Events **MUST** be typed (`events.Topic[T]`) and published with `Publish(ctx, tx, ev)`, recording the event on the transaction. Delivery depends on the subscriber class:

| Class | Delivery | Examples |
|---|---|---|
| After-commit listener | in-process on the committing node, after commit; skipped on rollback; panic-isolated per listener | cache invalidation, metrics counters |
| Durable subscriber | one job per subscriber, enqueued in the publishing transaction: kind `event:<subscriber>`, args = envelope, dedupe `event:<event_id>:<subscriber>` | notification routing, webhook dispatch, subtitle search after import, media-server refresh |
| Change log | an `event_log` row in the publishing transaction (below) | SSE invalidations |

_Source: design/runtime.md §3.6._

**PLT-064** (MUST NOT) There **MUST NOT** be synchronous subscribers that run inside the publishing transaction. Work that must be atomic with the write is a direct call. _Source: design/runtime.md §3.6; D19._

**PLT-065** (MUST) One failing subscriber **MUST NOT** affect others: durable subscribers retry independently, and after-commit listener failures are logged and counted. _Source: design/runtime.md I8._

**PLT-066** (MUST) The job queue **MUST** be the transactional outbox; there **MUST NOT** be a second relay table polled for delivery. _Source: design/runtime.md §3.6; open-issues A4._

**PLT-067** (MUST) Every UI-visible change **MUST** append an `event_log` row in the publishing transaction: `{topic, action (created|updated|deleted|resync), ids, version, audience, origin_node}`, where `audience` is `{permission, kind?, library?, owner_ids?}` as defined by `10-identity-and-access.md`. Payloads **MUST** be ids and versions only (notify, then fetch). _Source: design/identity-api-deployment.md §3.7 (replay, ACL metadata); open-issues A4._

**PLT-068** (MUST) The change log **MUST** be delivered in a total order that is safe against out-of-order commits. On PostgreSQL each row **MUST** store `txid = pg_current_xact_id()` and a sequence number; nodes **MUST** read only rows with `txid < pg_snapshot_xmin(pg_current_snapshot())`, ordered by `(txid, seq)`. On SQLite rows are ordered by `seq` alone (single writer). The SSE event id is an opaque encoding of this cursor and is valid on every node. _Source: design/identity-api-deployment.md §3.7 ("sequence ids are cluster-global"); derived mechanism._

**PLT-069** (MUST) Each node **MUST** run one change-log tailer that reads new rows when woken by `NOTIFY bonnyread_events` and, independently, every 1 s, and feeds the local SSE hub and cache invalidation. The metric `bonnyread_event_log_lag_seconds` **MUST** report the age of the oldest undelivered row, and a health issue **MUST** be raised when it exceeds 30 s (a long-open write transaction holds the watermark back). _Source: open-issues A4; derived from the ordering rule above._

**PLT-070** (MUST) The change log **MUST** be retained so that a reconnecting client can resume: a row **MAY** be deleted only when it is older than 10 minutes **and** at least 50,000 newer rows exist [OPN-026]. A resume from a cursor that is no longer retained **MUST** produce `reset` (framing in `11-api-and-clients.md`). _Source: design/identity-api-deployment.md §3.7; open-issues Group C (identity)._

**PLT-071** (MUST) Ephemeral changes (job progress, download activity progress) **MUST NOT** be written to `event_log`. They are delivered in-process and, on PostgreSQL, as the `NOTIFY bonnyread_events` payload (kept under 8,000 bytes; larger batches send a `resync` notice for the topic instead). Loss of an ephemeral change is acceptable; clients refetch current state on reconnect. _Source: design/runtime.md §3.6; design/identity-api-deployment.md §3.7._

**PLT-072** (MUST) The SSE hub **MUST** filter every change against the connection principal's effective permissions through the single authoriser (`10-identity-and-access.md`) before sending, and **MUST** close a stream on a `principal` change for its user (`principal.changed`). _Source: design/identity-api-deployment.md §3.7; I7._

**PLT-073** (MUST) NOTIFY channels **MUST** be `bonnyread_events`, `bonnyread_jobs`, `bonnyread_sched`, `bonnyread_lease` and `bonnyread_cancel`. When the LISTEN connection reconnects, the node **MUST** flush node-local caches that rely on doorbell invalidation. On SQLite doorbells are no-ops. _Source: design/runtime.md §3.6._

### 8. Notifications

**PLT-074** (MUST) Modules **MUST** register notifiable event types with id, version, JSON Schema, audience rule and default template. The catalog **MUST** be published in the OpenAPI document and the documentation; it is the webhook contract. The catalog **MUST** include at least: `download.grabbed` (`05-downloads.md` DLD-122), `file.imported`, `file.upgraded`, `file.deleted` (`06-import.md` IMP-095), `request.*`, `issue.*`, `subtitle.downloaded`, `profile.update_blocked`, `profile.publish_failed`, `profile.conflict_pending` (`07-profiles.md` PRF-109), `irc.disconnected`, `download.extraction_failed`, `download.manual_interaction_required`, `health.failed`, `health.restored`, `job.failed` (opt-in) and `plugin.disabled`. Areas add their own types in their specifications [OPN-208]. _Source: design/runtime.md §3.7; D7._

**PLT-075** (MUST) Every notification, webhook and script payload **MUST** use the envelope:

```json
{ "specversion": "bonnyread.event/1", "id": "<ULID>", "type": "file.imported",
  "time": "2026-09-28T12:00:00Z", "instance": {"name": "home", "url": "https://…"},
  "actor": {"type": "user|system|schedule", "id": "…"},
  "subject": {"kind": "episode", "id": "…", "slot": "4k"}, "data": { } }
```

Changes within `bonnyread.event/1` **MUST** be additive. _Source: design/runtime.md §3.7; D7._

**PLT-076** (MUST) Routing **MUST** be a durable subscriber `notify.route` that resolves recipients at delivery time: system targets with a matching subscription, and user targets whose owner is in the event's audience with permissions re-checked now. Users with manage permissions receive admin events except for their own actions; requesters receive their own requests' events; `own_only` filters apply. It then enqueues one `notify.deliver{target, event}` job per target with dedupe `event_id:target_id`. _Source: design/runtime.md §3.7; I9; D17._

**PLT-077** (MUST) Subscriptions **MUST** match on an event pattern (`media.*`) plus a filter (media kinds, tags, slots, minimum severity, `own_only`). _Source: design/runtime.md §3.7._

**PLT-078** (MUST) `notify.deliver` **MUST** render the event with the target's template (default or override), send it, record a `notification_delivery` row, and apply per-target backoff along `0 s, 1 m, 5 m, 15 m, 30 m, 1 h, 3 h, 6 h, 12 h, 24 h`. While a target is backed off, deliveries **MUST** snooze rather than fail, up to a maximum age (default 6 h), after which they are dropped and a health issue is raised [OPN-136]. _Source: design/runtime.md §3.7; I8._

**PLT-079** (MUST) Built-in targets **MUST** include Discord, Slack, Telegram, email (SMTP), ntfy, Gotify, Pushover, Apprise API, Kavita, webhook, script and Alertmanager. Media-server library refresh (Plex, Jellyfin, Emby) reuses the target registry but is triggered by the import area. WASM plug-ins **MAY** provide further targets (§20). _Source: design/runtime.md §3.7; D15._

**PLT-080** (MUST) A test send **MUST** go through the same deliver path synchronously and mark the request as a test (webhooks: `X-Bonnyread-Test: true`). _Source: design/runtime.md §3.7._

**PLT-081** (MUST) The last 50 deliveries per target **MUST** be retained for inspection. _Source: design/runtime.md §3.7 ("last N per target"; N chosen here)._

### 9. Webhooks and custom scripts

**PLT-082** (MUST) A webhook delivery **MUST** be `POST <url>` with body = envelope, `Content-Type: application/json`, and headers `X-Bonnyread-Event: <type>`, `X-Bonnyread-Delivery: <delivery id>` and `X-Bonnyread-Signature: t=<unix seconds>,v1=<hex HMAC-SHA256(secret, "<t>.<body>")>`. The secret is generated per target (32 random bytes) and is write-only. _Source: design/runtime.md §3.7; D7._

**PLT-083** (MUST) Webhook timeout **MUST** be 10 s. A 2xx response is success; 410 disables the target and raises a health issue; any other outcome is retried under the target backoff. _Source: design/runtime.md §3.7._

**PLT-084** (SHOULD) The webhook documentation **SHOULD** tell receivers to verify the signature and reject timestamps older than 5 minutes. _Source: derived from the signature scheme._

**PLT-085** (MUST) A script target **MUST** run the configured executable path with configured arguments in the `script` queue, with a timeout (default 60 s), the envelope on stdin, and a sanitised environment containing only `PATH`, `HOME`, `TZ`, `BONNYREAD_EVENT_TYPE`, `BONNYREAD_EVENT_ID` and `BONNYREAD_EVENT_FILE` (path of a temp file holding the envelope). Arguments **MUST NOT** be interpreted by a shell. Combined stdout and stderr **MUST** be captured into the delivery row, capped at 64 KB. A non-zero exit is a failed delivery. _Source: design/runtime.md §3.7; D7._

**PLT-086** (MUST) With `ALLOW_SCRIPTS=false`, script targets **MUST** be rejected on create and skipped (with a health issue) on delivery. _Source: design/runtime.md §3.7._

**PLT-087** (MUST) In HA, a node-scoped health check **MUST** verify that each enabled script exists and is executable on every node. _Source: design/runtime.md §3.7._

### 10. Alert sinks

**PLT-088** (MUST) The Alertmanager target **MUST** implement the Alertmanager v2 API: `POST /api/v2/alerts` to each configured Alertmanager URL, with labels `{alertname, severity, check, subject, instance}` and annotations `{summary, description, docs_url}`. _Source: design/runtime.md §3.7; D7._

**PLT-089** (MUST) An `alerts.tick` system schedule (every 1 m) **MUST** re-send every active `health_issue` at or above the target's severity threshold with `endsAt = now + 4 m`, and send `endsAt = now` for issues resolved in the last 5 m. _Source: design/runtime.md §3.7._

**PLT-090** (MUST) Other targets **MUST** receive health transitions as ordinary `health.failed` and `health.restored` events. _Source: design/runtime.md §3.7._

### 11. Health checks

**PLT-091** (MUST) A health check **MUST** implement `ID()`, `Scope()` (`Cluster` or `Node`), `Triggers()` (event types), `Interval()` and `Run(ctx) ([]Issue, error)`. An issue carries a stable `code`, `severity` (`info|warning|error`), a subject reference, structured `args` and a `docs_url`. Code **MUST NOT** parse issue message text. _Source: design/runtime.md §3.8; I10._

**PLT-092** (MUST) Results **MUST** be stored in `health_issue`. A transition (new issue, severity change, resolution) **MUST** publish `health.failed` or `health.restored` as a durable event. _Source: design/runtime.md §3.8._

**PLT-093** (MUST) An event listed in a check's `Triggers()` **MUST** enqueue `health.run:<check>` with `run_at = now + 5 s` and dedupe `WhilePending` (cluster-wide debounce). A system schedule **MUST** run each check at its interval. Node-scoped checks **MUST** run as node-affine jobs on every node. _Source: design/runtime.md §3.8._

**PLT-094** (MUST) The platform **MUST** register at least these checks:

| Code | Scope | Condition |
|---|---|---|
| `platform.mixed_versions` | cluster | more than one binary version among live nodes |
| `platform.no_external_backup` | cluster | PostgreSQL and the external-backup acknowledgement setting is unset |
| `platform.backup_failed` | cluster | last scheduled backup failed (SQLite) |
| `platform.clock_skew` | node | node clock differs from DB `now()` by > 2 s |
| `platform.listener_down` | node | LISTEN connection not subscribed (PostgreSQL) |
| `platform.event_log_lag` | cluster | change-log lag > 30 s (§7) |
| `platform.job_abandoned` | cluster | a job exceeded timeout + 30 s |
| `platform.tool_missing` | cluster | jobs waiting > 5 min for a tool no live node has (`01-architecture.md` §9) |
| `platform.root_unavailable` | node | root folder marker missing, mismatched or not writable |
| `platform.colocation` | node | staging or download path not on a root's filesystem where hardlinks are expected (`01-architecture.md` §10) |
| `platform.temp_space` | node | free space in `STATE_DIR` below the configured minimum |
| `platform.script_missing` | node | an enabled script target is absent or not executable |
| `platform.plugin_cache` | node | the WASM compilation cache is not writable |
| `platform.sqlite_integrity` | node | `quick_check` failed after an unclean shutdown |
| `platform.target_backoff` | cluster | a notification target dropped deliveries after maximum age |
| `platform.plugin_disabled` | cluster | a plug-in was auto-disabled |

_Source: design/runtime.md §3.2, §3.8, §3.14–3.15; open-issues A5, A6._

**PLT-095** (MUST) `/livez` (process responsive) and `/readyz` (DB reachable, schema compatible, listener subscribed, not draining) **MUST** be independent of domain health. Both are unauthenticated and served on the main and admin listeners (`13-deployment.md` DEP-012) [OPN-138]. _Source: design/runtime.md §3.8._

### 12. Metrics

**PLT-096** (MUST) Each node **MUST** serve `/metrics` (Prometheus exposition, `prometheus/client_golang`) with process metrics: Go runtime and process; HTTP requests and latency by route template; `bonnyread_jobs_executed_total{kind,queue,outcome}`; `bonnyread_job_duration_seconds{kind}`; `bonnyread_job_abandoned_total`; worker pool utilisation per queue; SSE connections; `bonnyread_lease_held{name}`; LISTEN reconnects; `bonnyread_event_log_lag_seconds`; outbound HTTP by `{target_type, target, status_class}` with latency; rate-limit waits and denials by rate key; regex timeouts by source; plug-in calls, traps and duration; area metrics registered by modules. _Source: design/runtime.md §3.11; D7; open-issues A9._

**PLT-097** (MUST) Every node **MUST** serve `/metrics/cluster` with database-derived gauges, identical from any node, computed on demand and cached 30 s: queue depth by `{queue, state}` and oldest `available` age; `bonnyread_health_issues{check,severity}`; `bonnyread_nodes_live`; and gauges registered by areas (library counts, download queue, pending requests, disabled indexers and clients). _Source: open-issues A9; design/runtime.md §3.11._

**PLT-098** (MUST) Metric names **MUST** be prefixed `bonnyread_`. Labels **MUST** come only from bounded, admin-configured sets; per-item, per-release and per-user labels **MUST NOT** be used. _Source: design/runtime.md §3.11; I13._

**PLT-099** (MUST) `/metrics` and `/metrics/cluster` **MUST** require a principal with `metrics:read` on both the main and the admin listener; there is no unauthenticated metrics option [OPN-137]. _Source: design/runtime.md §3.11; design/identity-api-deployment.md §3.10._

### 13. Logging

**PLT-100** (MUST) Logging **MUST** use `log/slog`, JSON when not on a TTY and text on a TTY, to stdout, plus an optional size-rotated `LOG_FILE` for desktop installs. Records **MUST** carry `node` and, where relevant, `job_id`, `job_kind`, `attempt`, `request_id`, `user_id`, `trace_id` and `span_id`. _Source: design/runtime.md §3.11._

**PLT-101** (MUST) A redacting `slog.Handler` **MUST** replace values of the secret type (and `secret`-tagged settings) with a fixed marker. _Source: design/runtime.md §3.11; D16._

**PLT-102** (MUST) The UI log viewer **MUST** be served from the per-job run logs (cluster-wide, persisted) and a per-node in-memory ring buffer of the last 10,000 records at or above the configured level, labelled with the node it came from [OPN-068]. _Source: design/runtime.md §3.11; open-issues Group C (runtime)._

**PLT-103** (MUST NOT) Application logs **MUST NOT** be written to the main database. _Source: design/runtime.md §3.11._

### 14. Tracing

**PLT-104** (MUST) OpenTelemetry tracing **MUST** be off by default (noop provider) and enabled by the standard `OTEL_EXPORTER_OTLP_*` variables. Spans **MUST** cover HTTP handlers, outbound `httpx` calls, job executions, sampled database queries and plug-in calls. _Source: design/runtime.md §3.11._

**PLT-105** (MUST) A job's span **MUST** link to (not be a child of) the span that enqueued it, using the stored `traceparent`. _Source: design/runtime.md §3.11._

**PLT-106** (MUST NOT) The core **MUST NOT** export OTel metrics in v1; metrics are Prometheus only. _Source: design/runtime.md §3.11._

### 15. Housekeeping

**PLT-107** (MUST) Housekeepers **MUST** be registered functions. A daily `housekeeping` system schedule **MUST** enqueue a parent job with one child job per housekeeper, each with its own timeout, failure and metrics; there is no global exclusivity. _Source: design/runtime.md §3.9; I14._

**PLT-108** (MUST) Platform housekeepers **MUST** cover: job retention (§4), notification-delivery and resolved-health retention, change-log retention (§7), stale `node` rows and their per-node schedules (heartbeat older than 1 day), expired lease rows, per-node temp and scratch directories (node-affine), eviction of unused WASM compilation-cache entries, rate-limit usage buckets past their longest window, backup retention, and on SQLite `PRAGMA optimize`, `wal_checkpoint(TRUNCATE)` and a weekly `VACUUM` when the free-page ratio exceeds 20%. PostgreSQL relies on autovacuum. _Source: design/runtime.md §3.9._

### 16. Backup and restore

**PLT-109** (MUST) On SQLite the core **MUST** take backups with `VACUUM INTO` a temp file and package them as `bonnyread-<version>-<timestamp>.tar.zst` containing the database and `manifest.json` (app version, schema version, dialect, created_at, node, encrypted flag). Everything else durable (plug-ins, fetched definitions, PCD data) is in the database and needs no separate copy. _Source: design/runtime.md §3.14; open-issues A2._

**PLT-110** (MUST) Backup kinds **MUST** be scheduled (weekly by default), manual and pre-migration, with a retention count (default 7 scheduled backups; pre-migration backups kept until 3 newer ones exist) and a directory defaulting to `<DATA_DIR>/backups`. _Source: design/runtime.md §3.14 (retention counts chosen here)._

**PLT-111** (MUST) Backup archives and logical exports **MUST** support optional passphrase encryption with age (scrypt recipient). An encrypted archive's manifest **MUST** be readable only after decryption; the outer file is the age ciphertext. _Source: D20; open-issues B10._

**PLT-112** (MUST) Restore **MUST** be staged: validate the archive (decrypt if needed; dialect match; manifest schema version ≤ the binary's), extract to `<DATA_DIR>/restore-pending/`, run `PRAGMA integrity_check` on the extracted database, then restart. The next startup **MUST** swap the files before opening the database and keep the replaced database as `pre-restore-<timestamp>.db`; pending migrations then apply normally. _Source: design/runtime.md §3.14; I11._

**PLT-113** (MUST NOT) On PostgreSQL the core **MUST NOT** take physical backups; point-in-time recovery, `pg_dump` and operators are the operator's responsibility (`13-deployment.md` gives recipes), and `platform.no_external_backup` applies until acknowledged. _Source: design/runtime.md §3.14._

**PLT-114** (MUST) `bonnyread db export` and `db import` (also available as a job and through the API) **MUST** stream all entities in dependency order to NDJSON inside the same archive format, independent of dialect. _Source: design/runtime.md §3.14._

**PLT-115** (MUST) A **sanitised export** **MUST** exist: a logical export with every secret removed, a manifest flag `sanitised: true`, and a file name and UI label stating it cannot be restored without re-entering secrets. `db import` of a sanitised export **MUST** require explicit confirmation and leave secret fields unset. _Source: D20; open-issues B10._

**PLT-116** (MUST) `bonnyread db copy --from sqlite:<path> --to postgres:<dsn>` **MUST** copy an installation from SQLite to an empty, migrated PostgreSQL database, verify row counts per table, and be covered by an automated test [OPN-072]. _Source: design/runtime.md §3.14; open-issues Group C (runtime)._

### 17. Outbound HTTP (`httpx`), proxies and solvers

**PLT-117** (MUST) All outbound HTTP (indexers, download clients, metadata sources, notification targets, feeds, plug-ins) **MUST** go through clients built by `platform/httpx`. Areas **MAY** add middleware (for example the tracker session and classifier chain in `03-acquisition.md`) but **MUST NOT** construct bare `http.Client`s. _Source: design/runtime.md §3.1; design/acquisition.md §3.1, §3.10._

**PLT-118** (MUST) Each `httpx` client **MUST** be built from a policy: egress, rate key and lane, TLS trust, SSRF class, redirect policy (follow within policy, or manual), response size cap, per-request timeout, retry policy, User-Agent, redaction set and metric labels. _Source: design/runtime.md §3.12; design/acquisition.md §3.10._

**PLT-119** (MUST) TLS verification **MUST** always be on. Per-connection trust **MAY** add a custom CA or pin a certificate (SPKI SHA-256); there **MUST NOT** be an option to skip verification. _Source: design/identity-api-deployment.md I4._

**PLT-120** (MUST) The SSRF guard **MUST** classify each client:

| Class | Private, loopback, link-local destinations |
|---|---|
| `admin_configured` (download clients, indexers, media servers and targets configured by an admin) | allowed |
| `plugin` | denied unless the plug-in holds `private_networks: true` |
| `untrusted` (URLs from external data or non-admin input, for example image URLs from metadata) | denied |

The check **MUST** run on the resolved IP at dial time and on every redirect hop. _Source: design/runtime.md §3.12 (classes derived)._

**PLT-121** (MUST) Egress **MUST** be configurable as named proxies (`http` CONNECT or `socks5`, with optional credentials), plus a default egress setting; each provider **MAY** select an egress. _Source: design/acquisition.md §3.10._

**PLT-122** (MUST) The default User-Agent **MUST** be `bonnyread/<version> (+<project URL>)`. Areas **MAY** override it per provider where a source requires a browser-like agent. _Source: design/metadata-and-subtitles.md §A9; design/acquisition.md §3.10._

**PLT-123** (MUST) The platform **MUST** support FlareSolverr-compatible solvers (FlareSolverr, Byparr) configured as entities with URL, `max_sessions` (default 2) and timeout. On a challenge detected by the caller's classifier, `httpx` **MUST** claim the `solver_clearance(host, egress_id)` row (PLT-062), re-read it and reuse a still-valid clearance, otherwise send the solve request through the **same egress** as the original request, store the resulting `cf_clearance` cookies and User-Agent in `solver_clearance(host, egress_id)` (shared by all nodes, secret), and replay the original request. A failed solve is a transient error. _Source: design/acquisition.md §3.10; design/metadata-and-subtitles.md §B10._

**PLT-124** (MUST) The retry middleware default **MUST** be two retries with exponential backoff and jitter on connect errors, 408 and 5xx, honouring `Retry-After`; areas **MAY** override. _Source: design/acquisition.md §3.10._

**PLT-125** (MUST) `httpx` **MUST** record the metrics and spans of §12 and §14 and scrub redaction-set values (keys, tokens, cookies) from logs, errors and stored responses. _Source: design/acquisition.md §3.10; design/runtime.md §3.11._

### 18. Shared rate limiter

**PLT-126** (MUST) `platform/ratelimit` **MUST** provide, per rate key: (a) **spacing** — a minimum interval between requests; (b) **quotas** — named windows (per minute, hour, day, or a fixed daily reset time) counted in usage buckets; (c) **lanes** with reserved and capped shares of each quota; (d) a shared **cooldown**. It is used by indexers, metadata sources, subtitle providers and plug-ins. _Source: open-issues A3; D20._

**PLT-127** (MUST) A rate key's scope **MUST** be `global` (one limit for the whole cluster, stored in the database) or `per_node` (limits that apply per egress IP, kept in memory on each node). _Source: design/metadata-and-subtitles.md §A9, §A16._

**PLT-128** (MUST) Global spacing **MUST** be a slot reservation: `UPDATE rate_limit_key SET next_slot_at = greatest(next_slot_at, now()) + $interval WHERE key = $k RETURNING next_slot_at - $interval`; the caller waits until the returned slot (or gives up if it is beyond the caller's deadline). _Source: design/acquisition.md §3.9(a); open-issues A3._

**PLT-129** (MUST) Quota usage **MUST** be counted by upsert-increment into `rate_limit_usage(key, quota, bucket_start, lane)`. Availability **MUST** be computed from the window's bucket sum, which **MAY** be cached for up to 30 s; the resulting over-run is bounded by the number of nodes and is accepted. A denial **MUST** return `ErrBudget{RetryAt}` computed from the bucket that expires first. _Source: design/acquisition.md §3.9(b)._

**PLT-130** (MUST) Each rate key **MUST** support lanes defined by the calling area, each with a `reserve` (share of the quota that lower lanes may not consume) and a `cap` (maximum share of what remains). The `interactive` lane **MUST** always have a reserve, so background work cannot starve user actions. Lane names and shares are specified by the areas (`03-acquisition.md`, `08-metadata.md`, `09-subtitles.md`). _Source: open-issues A3; design/acquisition.md §3.9(b); design/metadata-and-subtitles.md §A9._

**PLT-131** (MUST) A 429 or 503 response, a `Retry-After` header or an area-classified "limit reached" **MUST** set `cooldown_until` on the rate key, honoured by every node. _Source: design/metadata-and-subtitles.md §A9; design/acquisition.md §3.9._

**PLT-132** (MUST) On SQLite spacing and cooldown **MUST** be in memory and quota usage persisted in `rate_limit_usage`, behind the same interface. _Source: design/acquisition.md §3.15; design/metadata-and-subtitles.md §A16._

### 19. Regex wrapper

**PLT-133** (MUST) All regular expressions from datasets and definitions (custom-format patterns, Cardigann `regexp`/`re_replace` filters and templates, Arr parser patterns kept on .NET syntax, Whisparr product codes) **MUST** be compiled through `platform/regex` on `github.com/dlclark/regexp2` with .NET semantics. Patterns written by the project **MAY** use Go `regexp` (RE2) where a rewrite is possible and passes the ported test corpora, allowing for RE2's ASCII-only `\d`, `\w` and `\b`. _Source: D1, D1a–D1c._

**PLT-134** (MUST) Every regexp2 match **MUST** have a timeout: default 100 ms per match, configurable per call site. A timeout **MUST** return a typed `ErrRegexTimeout` and increment `bonnyread_regex_timeouts_total{source}`; the caller's specification defines what a timeout means for its result. _Source: D1._

**PLT-135** (MUST) The wrapper **MUST** offer `MatchContext(ctx, …)` that refuses to start a match after the context's deadline, so callers can impose per-release evaluation deadlines between matches (`04-evaluation.md`). _Source: open-issues A20._

**PLT-136** (MUST) Before compilation, a normaliser **MUST** rewrite .NET named blocks `\p{IsBlock}` and `\P{IsBlock}` (inside and outside character classes) to code-point ranges from a table of all .NET named blocks. An unknown block name **MUST** fail compilation with code `regex_unknown_block`. _Source: D1, D1b._

**PLT-137** (MUST) CI **MUST** compile every pattern of the Dictionarry and TRaSH datasets and of the current Cardigann definition sets through the wrapper and fail on any error. _Source: D1._

**PLT-138** (MUST) Compiled patterns **MUST** be cached in a bounded LRU keyed by `(engine, options, pattern)`. _Source: design/runtime.md §3.12 (host-side regex); derived._

### 20. WASM plug-in host

**PLT-139** (MUST) Plug-ins **MUST** run in-process on `tetratelabs/wazero` with one runtime per process configured with `WithCloseOnContextDone(true)`, a per-plug-in memory limit, and a compilation cache in `<STATE_DIR>/wasm-cache` keyed by module hash. The cost of `WithCloseOnContextDone` on hot paths **MUST** be measured before P2; if it is high, the ABI **MUST** use coarser calls [OPN-070]. _Source: D15; design/runtime.md §3.12._

**PLT-140** (MUST) Guests **MUST** get WASI preview 1 with no preopened directories, no environment variables, no arguments and no sockets; only wall clock, monotonic clock and random. There **MUST** be no filesystem, network or process access except through host functions. _Source: D15; design/runtime.md §3.12; design/metadata-and-subtitles.md §B10._

**PLT-141** (MUST) The host ABI **MUST** be versioned (`abi: 1`) and shared by every plug-in kind; only message schemas differ per kind. The v1 ABI **MUST** be: guest exports `bonny_abi() → u32`, `bonny_alloc(len) → ptr`, `bonny_free(ptr, len)`, `bonny_describe() → u64` (manifest), `bonny_configure(ptr, len) → u64`, one export per kind operation (for example `bonny_search`, `bonny_download`), optional `bonny_test`; buffers are JSON in guest memory, and a `u64` return packs `ptr << 32 | len`. Host functions live in import module `bonny`. An Extism-based ABI **MUST** be evaluated in a spike against this ABI before P2 and **MAY** replace it only if it preserves per-call timeouts, memory caps and instance pooling [OPN-065] [OPN-055]. _Source: design/metadata-and-subtitles.md §B10; design/runtime.md §3.12; open-issues Group C (runtime, metadata)._

**PLT-142** (MUST) Plug-in kinds **MUST** be `subtitle_provider`, `indexer`, `ddl_host`, `notification_target` and `announce_parser`. IRC connections **MUST NOT** be pluggable. _Source: D15; design/acquisition.md §3.16; design/runtime.md §3.12._

**PLT-143** (MUST) A plug-in package **MUST** be a `.bnyplug` zip containing `manifest.json`, `plugin.wasm` and an optional `signature`. The manifest **MUST** declare: `id`, `version`, `abi`, `kind`, `min_host_version`, `license`, a settings JSON Schema, declared credentials (with `project_default` where the host holds a project key), requested capabilities and hosts, requested memory, rate limits and quotas, and kind-specific fields (for subtitle providers, as in `09-subtitles.md`). _Source: design/runtime.md §3.12; design/metadata-and-subtitles.md §B10._

**PLT-144** (MUST) Installed plug-ins **MUST** be stored in the database (`plugin` row: manifest, sha256, wasm bytes, signature, signer, granted capabilities, enabled). First-party plug-ins built from `plugins/` **MUST** be embedded in the binary and versioned with the core. Nodes **MUST** compile on first use and cache by hash. _Source: open-issues A2; design/runtime.md §3.12; design/metadata-and-subtitles.md §B10._

**PLT-145** (MUST) Third-party plug-ins **MUST** be verified against ed25519 (minisign) trust keys managed by an admin (`plugin_trust_key`). Unsigned or untrusted plug-ins **MAY** be installed only when `PLUGINS_ALLOW=any` and the admin enables "allow unsigned" for that install. With `PLUGINS_ALLOW=none`, only first-party plug-ins run. First-party embedded plug-ins are trusted as part of the binary [OPN-066]. _Source: D15; design/runtime.md §3.12; open-issues Group C (runtime)._

**PLT-146** (MUST) The admin **MUST** approve requested capabilities and hosts at install. An upgrade that requests any new capability, host or larger limit **MUST** be held until re-approved; the old version keeps running meanwhile. `private_networks: true` **MUST** be shown as a distinct warning in the approval UI. _Source: design/runtime.md §3.12, §6 item 6._

**PLT-147** (MUST) Host functions and their enforcement **MUST** be:

| Host function | Grant | Enforcement |
|---|---|---|
| `http_do(req) → resp` | `http: {hosts: [...], private_networks: false}` | through `httpx` with class `plugin`: host allow-list (also on redirects), SSRF guard, the plug-in's rate key and quotas, egress, TLS always verified, per-configuration cookie jar persisted in `kv`, response cap (default 10 MiB, manifest may request up to 20 MiB), tracing, metrics. For `indexer` plug-ins the call goes through the indexer's tracker client (`03-acquisition.md`) |
| solver path in `http_do` | `cloudflare: true` | §17 solver flow, only if a solver is configured |
| `captcha_solve(kind, site_key, page_url)` | `captcha: true` | only with a configured captcha-solving service [OPN-202] |
| `kv_get / kv_put(key, value, ttl) / kv_delete` | `kv: {quota_kb}` (default 1024) | namespaced per plug-in configuration, stored in the DB |
| `cache_get / cache_set(ttl)` | default | per node, in memory, bounded LRU |
| `settings_get` | default | only this configuration's settings and declared credentials |
| `session_get / session_set` | `indexer` plug-ins only | the indexer's shared tracker session (`03-acquisition.md` ACQ-024, ACQ-118) |
| `log(level, msg)` | default | rate-limited, tagged `plugin=<id>` |
| `regex_match / regex_replace` | default | host-side `platform/regex` (regexp2, timeout) |
| `html_select(doc, css)` | default | host-side HTML parsing (goquery) |
| `parse_release(title)` | default | the core release parser |
| `archive_list / archive_read` | `archives: true` | host-side extraction (zip; RAR and 7z with pure-Go readers) with limits on total uncompressed size and entry count, admin settings `plugins.archive_max_bytes` and `plugins.archive_max_entries` [OPN-202] |
| `now_ms()` | default | wall clock |

Every call **MUST** be checked against the manifest and grants. _Source: design/runtime.md §3.12; design/metadata-and-subtitles.md §B10; design/acquisition.md §3.16._

**PLT-148** (MUST) Credentials **MUST** be delivered to the guest at `bonny_configure`, never compiled into modules. Project default keys are held by the host (D3b) and are user-overridable. _Source: design/metadata-and-subtitles.md §B10; D3b._

**PLT-149** (MUST) Memory limits **MUST** default to 16 MiB (256 pages) when the manifest requests none, **MUST** be capped by an admin setting, and **MUST NOT** exceed 256 MiB (4,096 pages). _Source: design/runtime.md §3.12; design/metadata-and-subtitles.md §B10 (64 MiB for subtitle providers is a manifest request within this range)._

**PLT-150** (MUST) Each plug-in configuration **MUST** have a pool of instances sized by its concurrency limit (default 4); an instance serves one call at a time. Each call **MUST** carry a deadline set by the calling area (for example subtitle search 20 s, download 60 s, parse 2 s). Plug-ins **MUST NOT** rely on in-instance state surviving between calls; the host **MAY** reuse or replace instances at any time. _Source: design/runtime.md §3.12; design/metadata-and-subtitles.md §B10._

**PLT-151** (MUST) On a trap, deadline or memory exhaustion the instance **MUST** be discarded and the failure fed into the provider's backoff. Five traps within 10 minutes **MUST** auto-disable the plug-in, raise `platform.plugin_disabled` and emit `plugin.disabled`. _Source: design/runtime.md §3.12._

**PLT-152** (MUST) Enabling, upgrading or disabling a plug-in **MUST** publish `plugin.changed`. Every node **MUST** then compile the new version, route new calls to it, drain old instances as their calls return and close the old module; disable closes the pools. _Source: design/runtime.md §3.12._

**PLT-153** (MUST) Each first-party plug-in **MUST** ship a conformance kit (recorded HTTP fixtures and expected outputs) that runs in CI. _Source: design/metadata-and-subtitles.md §B10._

## Data model

Platform-owned tables (`internal/platform/store/schema/platform_*.go`). Types are ent-style; `time` is DB time.

```
schema_meta            id=1 PK, schema_version string, min_binary_version string, dialect enum(postgres|sqlite)

node                   id string PK, hostname, version, schema_version, roles []string, tools JSON {name: version},
                       started_at, heartbeat_at, stopped_at NULL
  index (heartbeat_at)

job                    id bigint PK, kind, args JSON, args_version int, queue, priority int16, state enum,
                       run_at, attempt int, max_attempts int, timeout_s int, dedupe_key NULL, concurrency_key NULL,
                       node_affinity NULL → node.id (SET NULL), parent_id NULL → job.id (CASCADE),
                       trigger enum(manual|schedule|event|system), created_by_user NULL, traceparent,
                       claimed_by NULL, heartbeat_at NULL, cancel_requested bool, progress JSON, run_log text,
                       error text, created_at, started_at, finished_at
  index (queue, state, priority DESC, run_at)            -- claim path
  index (state, heartbeat_at)                            -- reaper
  index (state, finished_at)                             -- retention
job_dedupe             key PK, job_id → job (CASCADE)
job_running_key        key PK, job_id → job (CASCADE), claimed_by

schedule               id PK, key UNIQUE, job_kind, args JSON, spec_kind enum(interval|cron|manual), spec, tz,
                       jitter_pct, enabled, next_run_at, last_enqueued_at, last_job_id NULL,
                       owner_type NULL, owner_id NULL, node_id NULL (per-node schedules), overlap enum(skip|queue_one),
                       bounds JSON
  index (enabled, next_run_at)

lease                  name PK, holder_node NULL, token bigint, acquired_at, renewed_at, expires_at, meta JSON

setting                (namespace, key) PK, value JSON, version int, updated_by NULL, updated_at

event_log              seq bigint PK (identity), txid xid8 NULL (PostgreSQL), topic, action, ids JSON,
                       version NULL, audience JSON, origin_node, created_at
  index (txid, seq)

health_issue           id PK, check_id, subject_key, node_id NULL (cluster scope), severity, code, args JSON,
                       docs_url, first_seen, last_seen, resolved_at NULL
  unique (check_id, subject_key, node_id) where resolved_at IS NULL (partial index on both dialects)

notification_target        id PK, owner_user_id NULL (system), impl, name, settings JSON (secret fields flagged),
                           enabled, template_overrides JSON, failures int, disabled_until NULL, last_error
notification_subscription  id PK, target_id → target (CASCADE), event_pattern, filter JSON
notification_delivery      id PK, target_id → target (CASCADE), event_id, attempt, status, http_status NULL,
                           error, output text (≤ 64 KB), sent_at

plugin                 id PK, plugin_id, version, abi, kind, manifest JSON, sha256, wasm bytes, signature NULL,
                       signer_key_id NULL → plugin_trust_key (SET NULL), source enum(builtin|installed),
                       granted JSON, pending_grants JSON NULL, enabled, installed_by, installed_at
  unique (plugin_id, version)
plugin_trust_key       id PK, name, public_key, added_by, added_at
plugin_kv              (owner_type, owner_id, key) PK, value bytes, expires_at NULL

backup                 id PK, kind enum(scheduled|manual|pre_migration), path, size, encrypted bool,
                       sanitised bool, app_version, schema_version, status, error, created_at

rate_limit_key         key PK, scope enum(global|per_node), interval_ms, next_slot_at, cooldown_until NULL,
                       cooldown_reason NULL, quotas JSON, lanes JSON
rate_limit_usage       (key → rate_limit_key (CASCADE), quota, bucket_start, lane) PK, used int

egress_proxy           id PK, name UNIQUE, kind enum(http|socks5), url, username NULL, password secret NULL
solver                 id PK, name, kind enum(flaresolverr|byparr), url, max_sessions, timeout_s, egress_id NULL
solver_clearance       (host, egress_id) PK, cookies secret, user_agent, expires_at, lock_holder NULL, lock_until NULL   -- PLT-062 row claim
```

Runtime-fetched content tables (definitions, PCD data) are owned by `03-acquisition.md` and `07-profiles.md`; sessions, API keys, device credentials and idempotency keys by `10-identity-and-access.md` and `11-api-and-clients.md`.

## Flows

### F1. Claim with a concurrency key

1. Worker on node B (queue `io`) opens a transaction and selects up to `n` claimable jobs `FOR UPDATE SKIP LOCKED`, filtered to kinds B has registered and tools B advertises.
2. For each selected job with a `concurrency_key`, B inserts `job_running_key`; on conflict (another import on the same title is running) that job is left `available` and skipped.
3. B marks the remaining jobs `running`, `attempt+1`, `claimed_by=B`, commits, and starts handlers.
4. The handler does its file I/O, then commits results in `FencedTx`; completion deletes the `job_running_key` row.
5. **Failure path:** B crashes. After 45 s the reaper on any node returns the job to `available` and deletes the key row; the next claim gets attempt 2. If B was only paused, its late `FencedTx` finds `attempt` changed and aborts.

### F2. Scheduled RSS poll

1. Indexer 7 is created; in the same transaction the acquisition area creates `schedule(key='rss:indexer:7', interval 15 m, jitter 20%, overlap=skip)`.
2. When due, the scheduler on node C claims the row, enqueues `rss.poll{indexer:7}` with dedupe `sched:<id>` (`WhilePendingOrRunning`), and sets `next_run_at = now + 15 m + jitter`.
3. Any node runs the job. Its requests take slots from rate key `acq:indexer:7` (`03-acquisition.md` ACQ-064) in the `rss` lane.
4. **Failure path:** the cluster was down for 2 hours; on restart the schedule fires once (next run from `now`), not eight times. If the previous poll is still running, the dedupe key blocks a second one.

### F3. Durable notification after import

1. The import handler's `FencedTx` writes the media file and publishes `file.imported`. In the same transaction the bus enqueues `event:notify.route` and appends change-log rows.
2. After commit: doorbells ring; the change-log tailers on all nodes deliver invalidations to SSE clients.
3. `notify.route` runs on any node, resolves the requester's Discord target and the system webhook target, and enqueues two `notify.deliver` jobs.
4. The webhook responds 200: delivered. Discord responds 503: the target backs off 1 m, the delivery snoozes.
5. **Failure path:** Discord stays down for 7 hours; after the 6 h maximum age the delivery is dropped, `platform.target_backoff` is raised and Alertmanager receives it on the next `alerts.tick`.

### F4. SQLite backup and staged restore

1. The weekly `backup` schedule enqueues a backup job; it runs `VACUUM INTO` a temp file, writes the manifest, packs `tar.zst`, encrypts with age if a passphrase is set, records a `backup` row and applies retention.
2. Later the admin selects that archive and restores: the core decrypts, validates dialect and schema version, extracts to `restore-pending/`, runs `integrity_check`, and restarts.
3. Startup sees `restore-pending/`, swaps the database files before opening the database, keeps `pre-restore-<ts>.db`, then migrates to the binary's schema.
4. **Failure path:** `integrity_check` fails or the manifest's schema version is newer than the binary's: restore is refused before restart, and the live database is untouched.

### F5. Plug-in upgrade requesting a new host

1. An admin installs version 1.3.0 of a subtitle plug-in. Its signature verifies against a trust key; the manifest adds host `dl.example.org`.
2. The row is stored with `pending_grants`; version 1.2.0 stays enabled; the admin sees the new host and approves.
3. `plugin.changed` is published; every node compiles 1.3.0 (cached by hash), routes new calls to it and drains 1.2.0 instances.
4. **Failure path:** 1.3.0 traps five times within 10 minutes; it is auto-disabled, `plugin.disabled` is emitted and the provider stops receiving calls until an admin acts.

## Non-goals

| Non-goal | Reason |
|---|---|
| River, goqite, asynq or another queue library | Own migration line outside ent/Atlas, missing per-entity concurrency keys; asynq needs Redis (design/runtime.md §3.4). |
| MySQL/MariaDB claim dialect | MySQL dropped (D20, B5). |
| Leader election for schedules | Every node runs the scheduler (A1). |
| Kubernetes `Lease` objects | Leases are DB rows; the chart needs no RBAC. |
| NOTIFY as a delivery channel | Doorbell only (A4). |
| Synchronous in-transaction event handlers | Core flow stays as direct calls (D19). |
| Logs in the database, Sonarr-style `TrimLogDatabase` | Costly on SQLite; logs go to stdout, file, ring buffer, job run logs. |
| Physical PostgreSQL backups by the app | Operator's responsibility. |
| Sonarr `Sonarr_*` script variables, Arr webhook payloads | D7: own format only. |
| Declarative provisioning of entities (indexers, clients, targets) | Later; settings pinning covers settings only [OPN-067]. |
| Cluster-wide log store | Platform log aggregation (Loki and similar) is the HA answer. |
| OTel metrics export | Prometheus is the metrics path in v1. |
| Native or unsigned-by-default plug-ins, plug-in filesystem or socket access | D15. |
| Self-updater | Deferred (D16). |

## Traceability

| Reference | Preserved | Deliberately changed (D19) |
|---|---|---|
| Sonarr `CommandQueueManager` / `Commands` table | Work survives restart; orphaned running work is recovered | One DB table, no in-memory mirror; reaper and fencing for multiple nodes |
| Sonarr `CommandEqualityComparer`; Profilarr `dedupe_key` | Duplicate work collapses | Explicit dedupe key per kind with pending or pending-or-running policy |
| Sonarr `CommandPriority` | User work jumps the queue | Four numeric priorities, per-queue pools |
| Sonarr `RequiresDiskAccess`, `IsExclusive` | No races on the same files | Per-entity concurrency keys; no global exclusivity |
| Sonarr `ScheduledTasks`; Bazarr APScheduler `coalesce`, random start | Cadence survives restarts without catch-up storms; no stampede | Schedule rows owned by entities; deterministic jitter; `manual` spec instead of "cron in a century" |
| Sonarr `EventAggregator`, `EscalationBackOff` | Subscriber isolation; provider backoff | Durable subscribers as outbox jobs; after-commit listeners |
| Radarr/Sonarr SignalR `{action, resource}` | Live UI via invalidations | SSE from a change log with cursor resume |
| Seerr per-user notification routing | Requesters and admins get the right events | One catalog and envelope for every target |
| Sonarr `[CheckOn]` health + 5 s debounce | Event-triggered, debounced checks; failed/restored notifications | Coded issues, cluster-wide debounce via dedupe, Alertmanager sink |
| Sonarr SQLite online backup; Profilarr staged restore; autobrr pre-migration backup | Consistent, staged, pre-migration backups | `VACUUM INTO`; optional age encryption; logical export and `db copy` to PostgreSQL |
| Profilarr sanitised backup | Shareable configuration without secrets | Labelled as not restorable; full backups keep secrets (optionally encrypted) |
| autobrr `_FILE` secrets | Container secrets from files | For every bootstrap key; typed DB settings for the rest |
| Profilarr/Whisparr regex timeouts | Timeout on every backtracking match | One wrapper with `\p{IsBlock}` normaliser and CI compile check of every dataset |
| Lidarr plug-in loader | Extensibility | WASM sandbox, capability grants, signing, auto-disable |
