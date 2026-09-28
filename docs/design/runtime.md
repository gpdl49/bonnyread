# Runtime Platform, HA and Events

Status: **design proposal** (2026-09-28). Scope: how the Go core is structured and how it runs: module layout, process lifecycle, the durable job queue, leases, the scheduler, events and cross-node fan-out, notifications, webhooks, scripts and alerts, metrics, logging and tracing, health checks, housekeeping, backup and restore, configuration, shutdown and crash safety, storage, and the WASM plug-in host.

Domain behaviour (parsing, decisions, import, requests, subtitles), auth, the REST API, clients and deployment packaging belong to other design documents. This one only defines the hooks they plug into.

Binding inputs: [decisions.md](../decisions.md), especially D7 (own webhook and script format, first-class metrics and alerts), D12 (headless core with REST, SSE and `/metrics`), D13 (ent with Atlas, on Postgres, MySQL and SQLite), D14 (active-active HA on Postgres, leases, SKIP LOCKED, LISTEN/NOTIFY), D15 (wazero plug-ins), D16 (secrets in the database), D18 (the range from Kubernetes HA to a desktop service) and D19 (redesign, don't replicate).

Paths below are relative to `submodules/` unless they start with `docs/`.

---

## 1. Outcomes and invariants to preserve

These are the behaviours the reference apps get right, or that users depend on, restated as requirements. The implementation is re-derived in §3.

| # | Invariant | Reference evidence |
|---|---|---|
| I1 | **Background work survives restarts.** A queued or running task is not silently lost on a crash or restart. It is re-run or reported as failed. | Sonarr persists commands and orphans `Started` ones on boot (`sonarr/src/NzbDrone.Core/Messaging/Commands/CommandQueueManager.cs:276-281`, `CommandRepository.cs:32-35`). Profilarr resets `running` jobs to `queued` (`profilarr/docs/backend/jobs.md` "Startup Recovery"). autobrr has no persisted queue and loses in-flight delayed grabs ([autobrr.md](../analysis/autobrr.md) l.454): the counter-example. |
| I2 | **No duplicate work for the same subject.** A second "refresh series 5" while one is pending collapses into the first. Saving settings repeatedly does not pile up jobs. | Sonarr compares command bodies by reflection (`CommandEqualityComparer.cs:15-60`, used at `CommandQueueManager.cs:70,114`). Profilarr uses a unique `dedupe_key` (`jobs.md` "Queue Deduplication"). |
| I3 | **User-initiated work jumps the queue.** Interactive search and manual import are not stuck behind a library refresh. | `CommandPriority` plus ordering by priority, then queue time (`CommandQueue.cs:187-188`). `RefreshMonitoredDownloads` is `High` (`Jobs/TaskManager.cs:69-74`). |
| I4 | **No two file operations race on the same files.** Import, rename, move and delete never run concurrently on overlapping paths. | Sonarr's global `RequiresDiskAccess` mutex (`CommandQueue.cs:163-172`) on import, rename, move and delete commands (`Download/ProcessMonitoredDownloadsCommand.cs`, `MediaFiles/Commands/*`). Exclusive commands (`HousekeepingCommand`, `ApplicationUpdateCommand`) wait for an empty queue (`CommandQueue.cs:197-208`). |
| I5 | **Periodic work keeps its cadence across restarts, without catch-up storms.** After downtime, a missed task runs once, not N times. Intervals are user-tunable. | `ScheduledTasks` rows with `LastExecution + Interval < now` (`TaskManager.cs:46-51`). APScheduler `coalesce=True` (`bazarr/bazarr/app/scheduler.py:254`). Profilarr's "not due yet" guard (`jobs.md` "Fixed Interval"). |
| I6 | **Periodic work doesn't stampede.** Many schedules with the same interval (for example per-indexer RSS) are spread out. | Bazarr randomises interval jobs to 75–100% of the interval at startup (`bazarr/bazarr/app/scheduler.py:111`). autobrr uses `SkipIfStillRunning` ([autobrr.md](../analysis/autobrr.md) l.438). |
| I7 | **Live UI.** Every change to a resource the UI shows (queue, items, commands, health, tasks) reaches open browsers within about a second, as `{resource, action}` invalidations plus command progress. | Radarr SignalR `{action: updated\|deleted\|sync, resource}` (`radarr/src/Radarr.Http/REST/RestControllerWithSignalR.cs`, [radarr.md](../analysis/radarr.md) l.346). Sonarr ProgressMessaging (`sonarr/src/NzbDrone.Core/ProgressMessaging/ProgressMessageTarget.cs`). Profilarr SSE for jobs. |
| I8 | **One failing subscriber doesn't break the others.** A broken notification target doesn't block imports or other targets, and failing providers back off. | The EventAggregator catches exceptions per handler (`sonarr/src/NzbDrone.Core/Messaging/Events/EventAggregator.cs` l.100-145). `EscalationBackOff` runs 0 s, 1 m, 5 m, 15 m and up to 12 h+ (`sonarr/src/NzbDrone.Core/ThingiProvider/Status/EscalationBackOff.cs`). |
| I9 | **Notifications are routed to the right people.** Requesters hear about *their* requests. Admins hear about admin events, but not about their own actions. System targets get system events. | Seerr `shouldSendAdminNotification` (`seerr/server/lib/notifications/index.ts:67-85`), the per-user bitmask `hasNotificationType` (`:22`). comparison §2.2: "one notification system with user-scoped routing". |
| I10 | **Health issues are machine-actionable, re-evaluated when relevant, and alerted on transitions.** Checks are re-run shortly after a relevant event (debounced), notifications fire on *failed* and *restored*, and nobody has to parse message text. | `[CheckOn(typeof(Event))]` on 78 check sites plus a 5 s debounce (`sonarr/src/NzbDrone.Core/HealthCheck/HealthCheckService.cs:54`). `HealthCheckFailedEvent`/`RestoredEvent` feed notifications (`Notifications/NotificationService.cs:28-29`). Anti-pattern: Profilarr parses health *text* to delete media ([comparison.md](../analysis/comparison.md) §1.2). |
| I11 | **Backups are consistent and restorable, and restore is staged.** A backup never copies a live SQLite file byte by byte. Restore is applied before the DB opens. Migrations are preceded by an automatic backup. | SQLite online backup (`sonarr/src/NzbDrone.Core/Backup/MakeDatabaseBackup.cs:22-46`). Profilarr stages restore for the next startup (`profilarr/src/lib/server/utils/backup/applyPending.ts`). autobrr runs integrity check, backup and prune before migrating ([autobrr.md](../analysis/autobrr.md) l.163, `autobrr/internal/database/sqlite.go:163-330`). Gap: no Arr and no Bazarr backs up Postgres ([sonarr.md](../analysis/sonarr.md) l.70, [bazarr.md](../analysis/bazarr.md) l.180). |
| I12 | **Secrets can come from files.** Container secrets can be mounted as files. | autobrr `AUTOBRR__<KEY>_FILE` (`autobrr/internal/config/config.go:455-457`). |
| I13 | **Metrics are scrapeable and cardinality-safe.** | autobrr collectors (`autobrr/internal/metrics/collector/*.go`), Unpackerr `/metrics` with auth (`unpackerr/pkg/unpackerr/metrics.go:108-148`). D7 makes this first-class. |
| I14 | **Housekeeping keeps the DB bounded.** Command history, logs and orphans are trimmed, and SQLite is compacted. | 31 Sonarr housekeepers ([sonarr.md](../analysis/sonarr.md) l.178-183), `CleanupCommandQueue`, `TrimLogDatabase`, Prowlarr VACUUM ([prowlarr.md](../analysis/prowlarr.md) l.78). |
| I15 | **Untrusted extension code can't take over the process.** | Anti-pattern: Lidarr loads unsigned .NET assemblies in-process ([comparison.md](../analysis/comparison.md) §3.4). D15. |

New invariants that the monolith adds, because it runs HA (D14):

- **N1. Work runs at least once, and its side effects are fenced.** A worker that lost ownership of a job or lease (because of a pause, a partition or a slow disk) can't commit results.
- **N2. Clock-independent coordination.** All lease and claim comparisons use database time (`now()`), not node clocks. Sonarr needs `FixFuture*Times` housekeepers to repair clock skew ([sonarr.md](../analysis/sonarr.md) l.183); we avoid the problem instead.
- **N3. Identical behaviour on one node and on N nodes.** SQLite single-node and Postgres HA run the same code paths. Only the claim SQL, the doorbell and the lease backend change.

---

## 2. How the references do it today, and the accidental complexity

| Concern | Reference implementation | Accidental complexity (D19: don't copy) |
|---|---|---|
| **Command queue** | NzbDrone (7 forks): an in-memory `List<CommandModel>` guarded by `Monitor.Wait/PulseAll` (`sonarr/.../Messaging/Commands/CommandQueue.cs`), mirrored into a `Commands` table, 3 fixed threads (`CommandExecutor.cs:14` `THREAD_LIMIT = 3`), dispatch by `(dynamic)` to `IExecute<T>` resolved from DryIoc (`CommandExecutor.cs:42,72`). | Two sources of truth (memory and DB). Dedupe by reflection over every property (`CommandEqualityComparer.cs`), so adding a field silently changes the dedupe semantics. Exclusivity is three global booleans (`IsExclusive`, `IsLongRunning`, `RequiresDiskAccess`, `Command.cs:27-29`), so *any* disk command blocks *every* other disk command, for all series. It is single-process only. |
| | Profilarr: a SQLite `job_queue` with `dedupe_key`, a single-threaded timeout dispatcher, and a conditional-UPDATE claim (`profilarr/docs/backend/jobs.md`, `src/lib/server/jobs/dispatcher.ts`). | Strictly serial. It needs a *second* lock (`sync_status` columns) for per-instance exclusivity. No auto-retry. |
| | Bazarr: an in-memory `jobs_queue` deque with N threads (`bazarr/bazarr/app/jobs_queue.py`). Kapowarr: one FIFO `TaskHandler` thread (`kapowarr/backend/features/tasks.py`). Cleanuparr: Quartz `[DisallowConcurrentExecution]` jobs (`cleanuparr/code/backend/Cleanuparr.Api/Jobs/GenericJob.cs:16`) plus MassTransit **in-memory** queues (`DependencyInjection/MainDI.cs:32`). Seerr: `node-schedule` callbacks (`seerr/server/job/schedule.ts`), with side effects in ORM subscribers and no retry (`seerr/server/subscriber/MediaRequestSubscriber.ts`, [seerr.md](../analysis/seerr.md) l.233). | Four different non-durable queues. A message bus library (MassTransit) is used as an in-process function call. Side effects fire from ORM hooks, so failures turn into manual "retry" buttons. |
| **Scheduler** | A `ScheduledTasks` table with minute intervals and a 30 s timer that pushes commands (`sonarr/.../Jobs/Scheduler.cs`, `TaskManager.cs:65-135`). APScheduler with "manual" and "never" implemented as a cron **in a century** (`bazarr/bazarr/app/scheduler.py:58,231`). robfig/cron (autobrr). croner plus queue rows (Profilarr). | A hard-coded default task list that has to be reconciled with DB rows at boot. Per-entity schedules (per-indexer RSS, per-list sync) don't exist. The global RSS interval hits every indexer at the same moment. |
| **Event aggregator** | Sync `IHandle<T>` plus fire-and-forget `IHandleAsync<T>` on `TaskFactory` (`EventAggregator.cs:111-145`). autobrr: a typed `Topic[T]` over `maniartech/signals` with per-listener panic recovery (`autobrr/internal/events/eventbus.go`). | NzbDrone's async handlers are lost on crash. Core flow runs through events (import triggers notifications, which trigger refresh), so control flow can only be seen at runtime. |
| **Cross-process events** | SignalR hub `/signalr/messages` in every fork. Bazarr's three SignalR client classes, plus webhooks, plus an hourly full sync ([comparison.md](../analysis/comparison.md) §4). | All of it exists because the apps are separate processes. Gone in a monolith, except UI push. |
| **Notifications** | NzbDrone: `NotificationService` implements a dozen `IHandle<…Event>` handlers and dispatches to about 30 providers (`sonarr/src/NzbDrone.Core/Notifications/NotificationService.cs:20-31`). Seerr: per-user bitmasks and agents (`seerr/server/lib/notifications/`). Bazarr: Apprise. Profilarr, autobrr, Cleanuparr and Unpackerr each have their own. | Seven notification systems with overlapping target lists and differing payloads. Sending happens synchronously inside the event handler, with no durable retry. |
| **Health** | 20–34 checks per fork, event-triggered and debounced (`HealthCheckService.cs`), plus `ServerSideNotificationService` pulling messages from Servarr ([sonarr.md](../analysis/sonarr.md) l.85). | Checks such as `IndexerJackettAllCheck` in six forks only exist because of multi-app setups ([verification/interactions.md](../verification/interactions.md) #34). Messages are free text, and other apps parse them. |
| **Housekeeping** | 31–38 daily housekeepers, most of them `CleanupOrphaned*` ([radarr.md](../analysis/radarr.md) l.177-179). | Orphan cleanup exists because the schema has no foreign keys with `ON DELETE CASCADE`. |
| **Backup** | Zip of `config.xml` plus an SQLite online backup, with retention. No Postgres (Arrs, Bazarr). Profilarr: tar.gz with secrets **stripped** (`profilarr/src/lib/server/utils/backup/sanitize.ts`), so a restore loses the keys. Kapowarr: import, reverted after 60 s unless confirmed ([kapowarr.md](../analysis/kapowarr.md) l.95). | Each app has its own format. None handles Postgres. |
| **Config** | NzbDrone: `config.xml` plus Options classes (`sonarr/src/NzbDrone.Common/Options/*.cs`) plus a DB `Config` key/value table. Seerr: a JSON settings file with migrations ([seerr.md](../analysis/seerr.md) l.356). Bazarr: dynaconf YAML. autobrr: TOML plus `AUTOBRR__` env plus `_FILE`. | Three layers with overlapping keys (`config.xml` vs DB) and settings-file migrations. |
| **HA** | None. autobrr: "process-local singletons, only one instance can run" ([autobrr.md](../analysis/autobrr.md) l.486). Recyclarr: no locking between concurrent crons ([recyclarr.md](../analysis/recyclarr.md) l.442). | n/a |
| **Plug-ins** | Lidarr downloads unsigned assemblies into the process ([comparison.md](../analysis/comparison.md) §3.4). Bazarr vendors 22k lines of provider scrapers in-tree ([bazarr.md](../analysis/bazarr.md) l.44). | No isolation, no capability model. |

**Summary:** every app reinvents the same five primitives: a durable job table, a schedule table, an in-process event dispatch, a push channel and a provider-status backoff. They only differ in how much durability they skip. The monolith needs each primitive once, built on the database it already has.

---

## 3. Proposed design

### 3.1 Module structure

One Go module, `github.com/gpdl49/bonnyread`, and one binary. Wiring is explicit in a composition root, following autobrr's hand-wired `main.go` ([autobrr.md](../analysis/autobrr.md) l.30). There is no DI container, no `init()` registration and no reflection dispatch.

```
cmd/bonnyread/                  main: flags → config.Load → app.Run(ctx); subcommands (migrate, backup, restore, db copy, plugin)
internal/
  platform/                     runtime kernel; imports NO domain package
    config/                     bootstrap config (flags/env/file/_FILE) + typed DB settings registry
    db/                         ent client(s), dialect capabilities, Atlas migrations, tx helpers, fencing
    store/                      ent schema (ONE package, required by ent) + generated code (store/ent)
      schema/                   files prefixed by owner: platform_job.go, platform_lease.go, media_item.go, …
    lifecycle/                  ordered service group, readiness, shutdown
    jobs/                       durable queue: enqueue, claim, workers, retries, reaper
    lease/                      named leases + fencing tokens; node registry
    sched/                      schedule rows → jobs
    events/                     typed in-process bus, after-commit hooks, cluster fan-out, SSE hub
    notify/                     event catalog, routing, targets registry, templating, delivery
    health/                     check registry, issue store, transitions
    housekeeping/               housekeeper registry
    backup/                     archive, VACUUM INTO, staged restore, logical export
    obs/                        slog setup, metrics registry, OTel tracing
    httpx/                      shared outbound HTTP client: rate budgets, SSRF guard, tracing, proxy
    plugin/                     wazero host, manifests, capability host functions
    registry/                   the Module interface and Registrar (see below)
  domain/…                      pipeline packages (other design docs): release, indexer, download, import, profile, request, subtitle, …
  kinds/{tv,movie,music,book,adult,comic}/   per-media-kind modules
  api/                          REST + SSE handlers (other design doc)
  app/                          composition root: builds platform, registers modules, starts lifecycle
```

**Dependency rules.** These are enforced by a test that walks `go list -deps` and fails on violations; `go-arch-lint` is an option.
1. `platform/*` imports only the standard library, `platform/*` and `store/ent`. It never imports `domain`, `kinds` or `api`.
2. `domain/*` imports `platform/*` and other `domain/*` packages in an acyclic order. Where a cycle would appear, the dependency is inverted through an interface owned by the lower package, or the code reacts to an event (autobrr breaks import cycles this way, [autobrr.md](../analysis/autobrr.md) l.35). Events are *not* used for core flow that must be atomic; that stays a direct call.
3. `kinds/*` import `domain/*` and `platform/*`, never each other.
4. `api` imports `domain`, `kinds` (through interfaces) and `platform`. Nothing imports `api` or `app`.

**ent constraint.** ent needs every schema in one Go package, so `store/schema` is shared. Ownership is by file prefix and mixins. Platform tables are `job`, `job_dedupe`, `job_running_key`, `schedule`, `lease`, `node`, `setting`, `health_issue`, `notification_target`, `notification_subscription`, `notification_delivery`, `plugin` and `backup`.

**Registration.** Each media kind, and each group of built-in providers, is a `registry.Module`:

```go
type Module interface {
    Name() string
    Register(r *registry.Registrar) error
}

// Registrar collects everything a module contributes; frozen before lifecycle start.
func (r *Registrar) Job(kind string, h jobs.Handler, opts jobs.KindOpts)          // handler + default queue/timeout/retry/dedupe policy
func (r *Registrar) Schedule(key string, def sched.Default)                       // system schedule with default spec and bounds
func (r *Registrar) Event(t events.Type, meta events.Meta)                        // notifiable? audience rule, schema for docs
func (r *Registrar) Settings(ns string, s any)                                    // typed struct; tags give defaults, bounds, secret flag
func (r *Registrar) Health(c health.Check)
func (r *Registrar) Housekeeper(name string, fn housekeeping.Func)
func (r *Registrar) Provider(kind provider.Kind, impl string, f provider.Factory) // indexer, download client, notification target, …
func (r *Registrar) Metrics(c prometheus.Collector)
```

`app` builds the list explicitly (`[]registry.Module{tv.Module{…}, movie.Module{…}, notifytargets.Module{…}, …}`), and phase-3 kinds can be left out with a build tag. Providers carry a settings struct whose tags generate the JSON Schema that the UI renders as forms. This replaces ThingiProvider's `FieldDefinition` attributes. WASM plug-ins (§3.12) register into the **same** provider registry through an adapter, so the rest of the code can't tell a built-in provider from a plug-in.

### 3.2 Process lifecycle

One binary with roles (`--roles=api,worker`, default both). Every node is equal: there is no "primary" role (D14 active-active). Startup order, with each step blocking the next:

1. **Config**: parse flags, env and file, resolve `_FILE`, validate (§3.10). Derive `node_id` (stable per pod: `BONNYREAD_NODE_ID`, else hostname plus a random suffix persisted in the node-local state dir).
2. **Storage guard**: on SQLite, take an exclusive `flock` on `<data>/bonnyread.lock` and refuse to start if it is held. This prevents two processes, or a second "HA" replica on shared storage, from opening one SQLite file.
3. **DB open**: see §3.3 for pools and pragmas. On Postgres, check `server_version ≥ 15`.
4. **Migrate**: take the migration lock (Postgres `pg_advisory_lock(<const>)` on a dedicated connection; MySQL `GET_LOCK`; SQLite is already exclusive). If migrations are pending: on SQLite, run a **pre-migration backup** (`VACUUM INTO`) and `PRAGMA quick_check` first; on Postgres, log a warning that the operator's backup is the safety net. Apply the embedded Atlas migrations for the dialect. **Rolling upgrades:** migrations must be expand/contract (additive in release N, destructive in N+1), so N-1 nodes keep working during a rollout. A node whose schema version is *newer* than its binary refuses to start. The node table records versions, and a health check warns about mixed versions.
5. **Build**: construct platform services, register modules, freeze the registry, load settings into a snapshot, compile enabled plug-ins.
6. **Start** (the `lifecycle.Group`, in dependency order): obs → node heartbeat → cluster listener (Postgres LISTEN) → event bus → job workers → lease manager (long-lived singletons) → scheduler loop → HTTP server. `/healthz/ready` turns true only after the HTTP listener is up and the listener connection is subscribed.
7. **Shutdown** on SIGTERM/SIGINT, the service manager or `POST /system/shutdown`: see §3.13.

### 3.3 Database access pattern

- **Postgres**: one `pgxpool.Pool` shared by ent (`stdlib.OpenDBFromPool`) and by the LISTEN connection, which is acquired and held separately. ent runs with the `sql/lock`, `sql/upsert` and `sql/execquery` features: `ForUpdate(sql.WithLockAction(sql.SkipLocked))` exists ([entgo.io/docs/feature-flags](https://entgo.io/docs/feature-flags)), and `execquery` gives raw SQL for the few hot statements.
- **SQLite** (`modernc.org/sqlite`, CGO-free, so cross-compiling for Tauri targets works; autobrr uses it too): **two ent clients over two pools**. `RW` has `MaxOpenConns=1`, `_txlock=immediate`, WAL, `busy_timeout=5000`, `foreign_keys=ON` and `synchronous=NORMAL`. `RO` has N connections with `query_only=1`. On Postgres and MySQL, `RO` is the same client as `RW`. This avoids `SQLITE_BUSY` storms and keeps long reads from blocking the writer.
- **Time**: every coordination column is written with the database's `now()` (N2).
- **No read replicas in v1.** SSE invalidations rely on read-after-commit consistency (§4.2).

### 3.4 Durable job queue

#### Library evaluation

Checked 2026-09-28.

| Option | Databases | Fit |
|---|---|---|
| **River** (`riverqueue/river`, MPL-2.0) | Postgres (`riverpgxv5`, or `riverdatabasesql` in poll-only mode because `database/sql` has no LISTEN) and **SQLite (experimental; "may still have a few tweaks made to its schema")**. **No MySQL.** ([database-drivers](https://riverqueue.com/docs/database-drivers), [sqlite](https://riverqueue.com/docs/sqlite)) | Excellent semantics: unique jobs `ByArgs`/`ByPeriod`/`ByState` ([unique-jobs](https://riverqueue.com/docs/unique-jobs)), snooze, cancellation, internal leader election with a 5 s TTL ([leader-election](https://riverqueue.com/docs/leader-election)). But: (a) no MySQL, and D13 requires it; (b) its own migration line (`rivermigrate`) outside ent/Atlas, which breaks D13's "one schema, Atlas versioned migrations"; (c) the leader is internal and isn't exposed for our own singletons; (d) periodic jobs can be skipped during leader failover, and *durable* periodic jobs are River Pro, a commercial product ([periodic-jobs](https://riverqueue.com/docs/periodic-jobs), [pro/durable-periodic-jobs](https://riverqueue.com/docs/pro/durable-periodic-jobs)); (e) a transactional insert with an ent transaction needs a `*sql.Tx`, which is plumbing to spike. |
| **goqite** (`maragudk/goqite`, MIT) | SQLite and Postgres | A message queue with visibility timeouts. No priorities, dedupe or concurrency keys: we would rebuild most of it anyway. |
| neoq, gue, backlite, asynq | Postgres-only, SQLite-only or Redis | Each covers one tier. asynq adds Redis, which is ruled out (no new infrastructure). |
| **Own ent-based queue** | Postgres, MySQL 8+/MariaDB 10.6+ (both support `SKIP LOCKED`: [MySQL](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html), [MariaDB](https://mariadb.com/kb/en/select/#skip-locked)), SQLite | Tables live in the ent schema and Atlas migrations. It runs in the caller's ent transaction (a free transactional outbox). About 1.5–2k lines plus tests. We borrow River's semantics, not its code. |

**Decision: build a small ent-based queue**, designed so River could replace the Postgres backend later if MySQL is dropped (see §6). The deciding factors are D13 (MySQL, a single Atlas schema) and the need for per-entity concurrency keys, which River doesn't offer out of the box.

#### Schema (portable: no partial indexes, since MySQL lacks them)

```
job                (id PK bigint, kind, args JSON, queue, priority smallint, state, run_at, attempt, max_attempts,
                    timeout_s, dedupe_key NULL, concurrency_key NULL, node_affinity NULL, parent_id NULL,
                    created_by_user NULL, trigger (manual|schedule|event|system), traceparent,
                    claimed_by NULL, heartbeat_at NULL, cancel_requested bool, progress JSON, error text,
                    created_at, started_at, finished_at)
  index (queue, state, priority DESC, run_at)    -- claim path
job_dedupe         (key PK, job_id FK→job ON DELETE CASCADE)            -- collapses duplicates (I2)
job_running_key    (key PK, job_id FK→job ON DELETE CASCADE, claimed_by) -- ≤1 running job per key cluster-wide (I4)
```

States: `available → running → completed | failed | cancelled`. A retry goes back to `available` with a future `run_at`. There is no separate "scheduled" state: it is `available` with `run_at > now`.

#### Semantics

- **Enqueue** takes the caller's `*ent.Tx`: `jobs.Enqueue(ctx, tx, SearchItem{ItemID: 42, Slot: "4k"}, jobs.Opts{…})`. The job commits atomically with the business write (this *is* the outbox). Kinds are Go types with a `Kind() string`, and args are JSON (schema-versioned per kind).
- **Dedupe (I2)** replaces reflection equality with an explicit key per kind, for example `refresh:item:42`, `rss:indexer:7` or `search:item:42:4k`. Inserting a `job_dedupe` row that conflicts means "already pending". The policy per kind is:
  - `WhilePending` (the default): the dedupe row is deleted when the job starts, so one running and one pending job may coexist. Use it for refreshes, where data may have changed since the run started.
  - `WhilePendingOrRunning`: the row is deleted at finish. Use it for RSS and polls; this is autobrr's `SkipIfStillRunning`.
  - On conflict the existing job may be **promoted**: raise its priority and pull `run_at` earlier. This covers a user clicking "search" while a low-priority backlog search for the same item is waiting.
- **Priorities (I3)**: `interactive` (100), `high` (75, for example download monitoring), `normal` (50) and `background` (10, backlog, bulk refresh). The claim order is `priority DESC, run_at, id`.
- **Queues** are separate worker pools per node, sized in settings. `default` (8) is network and CPU work. `io` (2) is file import, move, rename and extraction, to keep disk contention bounded ([unpackerr.md](../analysis/unpackerr.md) l.369). `metadata` (4) is rate-limited source calls. `plugin` (4). `script` (2). One slow class can't starve another.
- **Concurrency keys (I4)** replace NzbDrone's global `RequiresDiskAccess` and `IsExclusive`. A job declares `concurrency_key`, typically the top-level library entity (`lib:series:5`) for any file operation on that entity, or `dlclient:3`. The claim inserts into `job_running_key` in the same transaction; a conflict (`ON CONFLICT DO NOTHING`, or `INSERT IGNORE` with affected rows = 0) puts that job back to `available` inside the transaction. This gives cluster-wide mutual exclusion **per entity** instead of one global disk lock, so importing series A no longer waits for renaming series B. Domain documents choose the keys; the platform guarantees the mechanism. Global exclusivity (`HousekeepingCommand`, updater) is dropped: housekeeping is a set of ordinary jobs (§3.9), and the self-updater is deferred (D16).
- **Claim** (Postgres):
  ```sql
  WITH c AS (
    SELECT id FROM job
     WHERE queue = $1 AND state = 'available' AND run_at <= now()
       AND (node_affinity IS NULL OR node_affinity = $node)
     ORDER BY priority DESC, run_at, id
     LIMIT $n FOR UPDATE SKIP LOCKED)
  UPDATE job j SET state='running', attempt=attempt+1, claimed_by=$node,
         started_at=now(), heartbeat_at=now()
    FROM c WHERE j.id = c.id RETURNING j.*;
  ```
  MySQL/MariaDB: the same `SELECT … FOR UPDATE SKIP LOCKED` in a transaction, then an `UPDATE … WHERE id IN (…)`, because MySQL has no `UPDATE … RETURNING`. SQLite: `UPDATE … WHERE id IN (SELECT … LIMIT n) RETURNING *` inside `BEGIN IMMEDIATE`; the single writer makes it atomic. Each dialect has its own file (`claim_pg.go`, `claim_mysql.go`, `claim_sqlite.go`) behind one interface, plus a shared conformance test suite that runs against all three in CI.
- **Wake-up**: after the enqueuing transaction commits, an in-process channel poke fires. On Postgres, a `NOTIFY bonnyread_jobs, '<queue>'` doorbell also goes out. Idle workers poll as a fallback (every 2 s, jittered), so a lost doorbell only adds latency.
- **Heartbeat and reaper**: running jobs heartbeat every 10 s (one batched `UPDATE … WHERE id IN (…) AND claimed_by=$node` per node), and the returned `cancel_requested` flags feed back. **Every** node runs the reaper every 15 s. It is idempotent: `UPDATE job SET state='available', run_at=now() WHERE state='running' AND heartbeat_at < now() - interval '45 s'`, which also deletes the job's `job_running_key`. It needs no leader.
- **Fencing (N1)**: ownership is `(job.id, attempt)`. Completion is `UPDATE job SET state='completed' … WHERE id=$id AND attempt=$a AND state='running'`; 0 rows means ownership was lost, so the result is discarded and logged. Handlers that write domain state do it through `jobs.FencedTx(ctx, fn)`, which begins a transaction, runs `SELECT 1 FROM job WHERE id=$id AND attempt=$a AND state='running' FOR UPDATE` (on SQLite, a plain select, since there's a single writer), then runs `fn`. A stale worker can't commit. **Consequence: execution is at least once.** Handlers must be idempotent with respect to external systems (see §4.1).
- **Retries**: per-kind policy. The default is exponential backoff with full jitter, `min(15s·2^attempt, 6h)`, with `max_attempts` 5 for background jobs and 1 for interactive ones (the user sees the failure at once and can retry). Handlers return typed outcomes: `jobs.Permanent(err)` (no retry), `jobs.Snooze(d)` (re-run later without consuming an attempt: rate-limited, waiting for a download client, and so on), or a plain error (retry). A panic is recovered and treated as a retryable error with its stack logged.
- **Timeouts**: per kind (for example RSS 2 m, search 5 m, import 30 m, backup 2 h). They are enforced through `context.WithTimeout`. Go can't kill a goroutine that ignores its context: after `timeout + 30 s`, the worker marks the job failed (fenced), frees the slot, increments `bonnyread_job_abandoned_total`, and raises a health issue. Leaked goroutines must be bugs, not policy.
- **Cancellation**: an `available` job is set to `cancelled` directly. A `running` one gets `cancel_requested=true` plus a `NOTIFY` so the owning node cancels its context within about 100 ms, instead of at the next heartbeat. The handler returns `ctx.Err()`, and the state becomes `cancelled`.
- **Progress** (I7): `jobs.Progress(ctx, msg, pct)` is throttled to 1/s per job, written to `job.progress` and fanned out as an event (§3.6). A per-job run log (last 200 lines, capped) is stored with the job; it answers "why did this import fail" without shipping logs.
- **Parent and child jobs**: bulk actions ("search all missing") create a parent job with children. The parent's progress aggregates the children's, and cancelling the parent cascades.
- **Retention**: completed jobs are kept 2 days, failed 14 days and cancelled 2 days, all configurable. Housekeeping deletes them in batches.

### 3.5 Leases and singleton work

**Most "singleton" work doesn't need a leader.** D14 lists RSS, download-client polling, schedules and git pulls as singleton work. In this design they are all **schedule rows turned into dedupe-keyed jobs** (§3.5.1), which any node may run exactly because of `SKIP LOCKED`, dedupe keys and concurrency keys. Only work that holds a **long-lived connection or exclusive external session** needs a lease:

- IRC announce connections, one lease per network (`irc:<network_id>`);
- push subscriptions to external systems, if any (for example a media-server websocket);
- filesystem watchers on import folders, if enabled (`watch:<path_id>`). Scheduled scans are the default.

**Mechanism: database leases, not advisory locks.**

```
lease (name PK, holder_node, token bigint, acquired_at, renewed_at, expires_at, meta JSON)
node  (id PK, hostname, version, schema_version, roles, started_at, heartbeat_at)
```

- **Acquire**: `UPDATE lease SET holder_node=$me, token=token+1, acquired_at=now(), expires_at=now()+$ttl WHERE name=$n AND (expires_at < now() OR holder_node=$me) RETURNING token`. The row is created with an insert-or-ignore first. **Renew** every `ttl/3` with a `WHERE holder_node=$me AND token=$t` guard.
- **Self-demotion**: the holder stops the singleton's work (cancels its context) when no renewal has succeeded for `2/3·ttl`, measured on its **monotonic** clock. Others can only take over after `ttl` of DB time, so under bounded clock *rate* drift the old holder has stopped before a new one starts.
- **Fencing token**: `token` is monotonic per lease. Any write caused by the singleton (for example enqueuing a grab from an IRC announce) runs `lease.Fence(tx, name, token)`, a guarded `SELECT … FOR SHARE`, so a paused former holder can't commit.
- **Defaults**: `ttl = 15 s`, renew every 5 s, and followers poll free leases every 2 s. A `NOTIFY bonnyread_lease` on release lets followers react at once. **Failover time**: a graceful shutdown takes under a second; a crash or partition takes 15–20 s.
- **Balancing**: a node only takes a free lease if it holds at most `ceil(total_leases / live_nodes)`. Live nodes are those with `node.heartbeat_at` within 30 s. This spreads IRC networks across replicas without a coordinator.
- **Single-node / SQLite**: an in-memory lease backend that is always granted (the process already holds the data-dir `flock`). It has the same interface.

Why not advisory locks? Postgres session-level advisory locks are bound to one connection. That means they don't mix with a pooled `pgxpool`, they break under PgBouncer transaction pooling, and they vanish silently when the connection drops, so the app learns late. They also carry no fencing token, can't be seen from the UI, and have no MySQL/SQLite equivalent with the same semantics (MySQL `GET_LOCK` is also session-scoped). Advisory locks are used **only** for the short startup migration lock.

#### 3.5.1 Scheduler

Schedules are **rows**, following Profilarr's "schedule = queue row" idea but kept separate from jobs:

```
schedule (id PK, key UNIQUE, job_kind, args JSON, spec_kind (interval|cron|manual), spec, tz,
          jitter_pct, enabled, next_run_at, last_enqueued_at, last_job_id, owner_type, owner_id,
          overlap (skip|queue_one), bounds JSON)
```

- **Sources**:
  1. **System schedules** that modules declare in code, with defaults and bounds (housekeeping daily, health 6 h, backup weekly, metadata refresh 12 h). They are upserted at boot *without* overwriting user overrides or moving an existing `next_run_at` (Profilarr's "rebuild preserves queued run time").
  2. **Per-entity schedules** that domain services create, update or delete **in the same transaction** as their owner: one RSS schedule per indexer, one sync per import list, one poll per download client, one pull per profile git database. Deleting an indexer cascades to its schedule, so there are no orphans.
- **Loop**: on **every** node there is no leader. Each loop sleeps until `min(next_run_at)`, capped at 30 s and woken by `NOTIFY bonnyread_sched` when schedules change. Then it runs:
  ```sql
  SELECT … FROM schedule WHERE enabled AND next_run_at <= now() ORDER BY next_run_at LIMIT 50 FOR UPDATE SKIP LOCKED
  ```
  and for each row, in one transaction: enqueue `job_kind(args)` with dedupe key `sched:<id>` (the `overlap=skip` policy means `WhilePendingOrRunning`), then set `next_run_at = next(spec, now()) + jitter`. The next run is computed from **now**, not from the missed slot, so downtime coalesces into one run (I5). On SQLite the same statement runs without `SKIP LOCKED`.
- **Jitter (I6)** is deterministic per schedule: `hash(schedule.id) mod (jitter_pct · interval)`, applied once so the phase is stable. Sixty indexers on a 15-minute RSS interval land spread across the window instead of in one burst. Cron schedules get no jitter unless asked.
- **Cron**: only a **parser** is needed to compute the next time. Use `netresearch/go-cron`, the maintained, API-compatible fork of the unmaintained `robfig/cron/v3`, with DST fixes ([pkg.go.dev](https://pkg.go.dev/github.com/netresearch/go-cron)). A timezone is set per schedule, defaulting to the server's timezone setting.
- **"Run now"** enqueues the same job with `interactive` priority. The schedule's `next_run_at` isn't moved unless the kind asks (`RescheduleOnManualRun`). "Manual only" is `spec_kind = manual`, not a cron in a century (`bazarr/bazarr/app/scheduler.py:58`).
- **Minimum intervals** come from the registered bounds (for example RSS ≥ 10 m unless the indexer allows less), validated at write time.

### 3.6 Events, outbox and cross-node fan-out

There are three distinct mechanisms, deliberately not one bus:

1. **Typed in-process events** (`events.Topic[T]`, generic, similar to autobrr's): `Publish(ctx, tx, ev)` records the event on the transaction, and delivery depends on the subscriber's class:
   - **After-commit listeners** (in-process, best-effort): cache invalidation, metrics counters, the SSE hub. They run after the commit on the node that committed, with panic recovery per listener. They are skipped if the transaction rolls back.
   - **Durable subscribers**: at `Publish`, the bus **enqueues one job per durable subscriber in the same transaction** (`kind = "event:<subscriber>"`, args = event envelope, dedupe key = `event_id:subscriber`). Examples: notification routing, webhook dispatch, the subtitle search after an import, the media-server refresh. Each retries independently (I8), survives crashes (I1), and can run on any node. **The transactional outbox is the job queue**; there is no second relay table to poll.
   - There are **no synchronous in-transaction handlers.** If something must be atomic with the write, it is a direct function call. This keeps core control flow visible in code (D19).
2. **Cluster change notices** (Postgres only): after commit, the committing node sends **one** `NOTIFY bonnyread_events, '<json>'` per transaction, carrying compact `ChangeNotice`s: `{origin_node, seq, items:[{topic, action, ids, audience}]}`. Payloads are kept well under Postgres's 8000-byte limit; if they'd be larger, a `{topic, action:"resync"}` goes out instead. A listener goroutine on each node (a dedicated pgx connection with `WaitForNotification`) feeds the local SSE hub and local cache invalidation, and ignores its own `origin_node`. **Delivery is at most once:** when the LISTEN connection reconnects, the node broadcasts `resync` to its SSE clients and flushes its caches. NOTIFY is used only as a doorbell and invalidation channel, never as the reliable path. Two reasons: notifications are lost while disconnected, and NOTIFY takes a global lock at commit ([Recall.ai 2025](https://www.recall.ai/blog/postgres-listen-notify-does-not-scale); [counterpoint, DBOS](https://www.dbos.dev/blog/postgres-listen-notify-scalability)). Our volume is small (tens per second, batched per transaction, sent after the business commit rather than inside it), so the global lock doesn't matter at this scale. Channels are `bonnyread_events`, `bonnyread_jobs`, `bonnyread_sched`, `bonnyread_lease` and `bonnyread_cancel`.
   The LISTEN connection must bypass PgBouncer transaction pooling (a direct or session-pooled DSN, `BONNYREAD_DB_LISTEN_DSN`, which defaults to the main DSN).
3. **SSE to browsers** (D12): a per-node hub. Each connection has an authenticated principal, and each notice has an `audience` (a permission, or explicit user ids), so a request-only user never receives admin-only resources. Messages are **invalidation hints plus small payloads**: `{resource, action, ids}` and job progress, like I7. The UI invalidates its query cache. Event ids are `<node_epoch>:<seq>`. When a client reconnects with a `Last-Event-ID` whose epoch isn't this node's current epoch (it was on another node, or this node restarted), the server sends `resync` first. So there is **no event store** for replay; refetching is cheaper and always correct. A keep-alive comment goes out every 15 s, and there's a per-user connection cap.

On MySQL and SQLite (single-node), mechanism 2 is a no-op and after-commit listeners feed SSE directly.

### 3.7 Notifications, webhooks, scripts and alert sinks

One pipeline serves user notifications, webhooks, custom scripts and alerting, as D7 asks.

**Event catalog.** Modules register notifiable event types with id, version, JSON schema, audience rule and default template. The union from [comparison.md](../analysis/comparison.md) §2.2 includes `release.grabbed`, `media.imported`, `media.upgraded`, `media.deleted`, `request.*`, `issue.*`, `subtitle.downloaded`, `profile.sync_failed`, `irc.disconnected`, `download.extraction_failed`, `download.manual_interaction_required`, `health.failed`, `health.restored`, `job.failed` (opt-in) and `plugin.disabled`. The catalog is published in the OpenAPI document and the docs (it is the webhook contract, D7).

**Envelope** (the own format, D7; versioned):
```json
{ "specversion": "bonnyread.event/1", "id": "01J…", "type": "media.imported", "time": "2026-09-28T12:00:00Z",
  "instance": {"name": "home", "url": "https://…"}, "actor": {"type": "user|system|schedule", "id": "…"},
  "subject": {"kind": "episode", "id": 1234, "slot": "4k"}, "data": { … type-specific … } }
```
The field names are close to CloudEvents 1.0, so a CloudEvents-aware receiver can map them. We don't claim to be a CloudEvents binding.

**Tables:**
```
notification_target       (id, owner_user_id NULL = system, impl, name, settings JSON (secrets flagged), enabled,
                           template_overrides JSON, status: failures, disabled_until, last_error)
notification_subscription (id, target_id, event_pattern ('media.*'), filter JSON (kinds, tags, slots, severity ≥, own_only))
notification_delivery     (id, target_id, event_id, attempt, status, http_status, error, sent_at)   -- last N per target
```

**Flow:**
1. A domain event is published with the durable subscriber `notify.route` (§3.6).
2. `notify.route(event)` resolves recipients:
   - **system targets** with a matching subscription;
   - **user targets** whose owner is in the event's audience, re-checking permissions *now* (D17 RBAC). This follows Seerr's semantics: requesters get their request's events, users with the manage permission get admin events except for their own actions (I9), and `own_only` filters apply.
   Then it enqueues one `notify.deliver{target, event}` job per target (dedupe `event_id:target_id`).
3. `notify.deliver` renders and sends through the target implementation, records a `notification_delivery` row, and applies **provider backoff** (the `EscalationBackOff` schedule, I8). While a target is backed off, deliveries **snooze** instead of failing, up to a maximum age (default 6 h), and then drop with a health issue. So a Discord outage doesn't lose the evening's imports.

**Targets** are providers in the registry: built-in ones (Discord, Slack, Telegram, email/SMTP, ntfy, Gotify, Pushover, Apprise API, media servers for library refresh (these are really domain actions, but reuse the same targets), Kavita, webhook, script, Alertmanager), plus WASM targets (D15).

- **Webhook**: `POST` of the envelope, `Content-Type: application/json`, and `X-Bonnyread-Event`, `X-Bonnyread-Delivery` and `X-Bonnyread-Signature: t=<unix>,v1=<hex HMAC-SHA256(secret, t.body)>` headers. Timeout 10 s; 2xx counts as success, 410 disables the target, other failures retry.
- **Custom script**: runs `path` with args from the target settings, in the `script` queue, with a timeout (60 s default), a sanitised environment plus `BONNYREAD_EVENT_TYPE`, `BONNYREAD_EVENT_ID` and `BONNYREAD_EVENT_FILE` (the path to the envelope JSON), and the envelope on stdin. This replaces the dozens of `Sonarr_*` variables with one documented schema. Output is captured into the delivery row, capped at 64 KB. In HA the script must exist at the same path on every node; a node-scoped health check verifies it (§3.8). Scripts can be disabled deployment-wide (`BONNYREAD_ALLOW_SCRIPTS=false`) for locked-down Kubernetes.
- **Alertmanager** (alert sink): implements the Alertmanager v2 API, `POST /api/v2/alerts` with labels `{alertname, severity, check, subject, instance}` and annotations `{summary, description, docs_url}` ([alerts API](https://prometheus.io/docs/alerting/latest/alerts_api/)). The API expects clients to **re-send firing alerts periodically** and to re-send resolved ones for a few minutes. So this target also subscribes to a platform-internal `alerts.tick` schedule (every 1 m) that re-sends all active `health_issue` rows at or above the configured severity with `endsAt = now + 4 m`, and sends `endsAt = now` for issues resolved in the last 5 m. Generic targets (ntfy and others) get `health.failed`/`health.restored` as ordinary events.

**Test sends** go through the same `deliver` path, synchronously, with an `X-Bonnyread-Test: true` header.

### 3.8 Health checks

```go
type Check interface {
    ID() string                      // "indexer.unavailable"
    Scope() health.Scope             // Cluster | Node
    Triggers() []events.Type         // re-evaluate on these events (I10)
    Interval() time.Duration         // plus periodic
    Run(ctx context.Context) ([]health.Issue, error)
}
type Issue struct { Code string; Severity Severity; Subject Ref; Args map[string]any; DocsURL string }
```

- Results go into `health_issue (check_id, subject_key, node_id NULL for cluster scope, severity, code, args, first_seen, last_seen, resolved_at)`. A **transition** (new, severity change, resolved) publishes `health.failed` or `health.restored` as a durable event, which feeds notifications and Alertmanager.
- **Triggering**: an event in `Triggers()` enqueues `health.run:<check>` with `run_at = now + 5 s` and dedupe `WhilePending`, which gives Sonarr's debounce cluster-wide for free. A schedule covers the periodic run. **Node-scoped** checks (root folders mounted and writable *on this node*, script targets present, free space in temp, plug-in cache writable, clock skew against DB `now()` > 2 s, LISTEN connection healthy) run through jobs with `node_affinity = node_id`, enqueued by a per-node schedule that each node creates for itself at start and removes on clean shutdown.
- Issues carry **codes and structured args**; the UI localises them. No code path reads message text.
- `/healthz/live` (process responsive) and `/healthz/ready` (DB reachable, schema version compatible, listener subscribed, not draining) are **separate** from domain health. An indexer outage must never take a pod out of the load balancer.

### 3.9 Housekeeping

- Housekeepers are registered functions. The daily `housekeeping` schedule enqueues a parent job with **one child job per housekeeper**, so each has its own failure, timeout and metrics. There's no global exclusivity.
- Most of NzbDrone's 31 orphan cleanups are **designed out**: the ent schema uses foreign keys with `ON DELETE CASCADE` or `SET NULL`. A schema review rule: every edge declares its delete behaviour.
- The platform housekeepers:
  - job, notification delivery and resolved-health retention;
  - stale `node` rows (heartbeat older than 1 day);
  - expired leases (cosmetic);
  - temp and scratch dirs on each node (node-affine);
  - the plug-in compilation cache (evict unused hashes);
  - backup retention;
  - SQLite `PRAGMA optimize` and `wal_checkpoint(TRUNCATE)`, plus `VACUUM` weekly when the free-page ratio exceeds 20%;
  - Postgres: nothing (autovacuum).

  Domain housekeepers (the recycle bin, the image cache, the metadata cache TTLs from D3) are registered by their modules.

### 3.10 Configuration (D16)

There are two layers with no overlap:

1. **Bootstrap config**: whatever is needed before the DB is open, or differs per node. Sources in precedence order: flags, then `BONNYREAD_<KEY>` env, then an optional YAML file (`--config`), then defaults. **Every key accepts `<KEY>_FILE`** (I12). The file is read at startup, with trailing newlines trimmed and an error if both forms are set. The keys:
   - `DATA_DIR` (shared; D14), `STATE_DIR` (node-local: locks, compilation cache, temp);
   - `DB_DSN`, `DB_LISTEN_DSN`, `DB_MAX_CONNS`;
   - `HTTP_ADDR`, `BASE_URL`, `TRUSTED_PROXIES`, `METRICS_ADDR` (optional separate listener);
   - `LOG_LEVEL`, `LOG_FORMAT` (json|text), `LOG_FILE` (optional rotating file);
   - `NODE_ID`, `ROLES`;
   - `PLUGINS_ALLOW`, `ALLOW_SCRIPTS`;
   - the standard `OTEL_*` variables.
2. **Runtime settings in the DB**: typed structs registered per namespace (§3.1). Tags give the defaults, validation, bounds, `secret` (write-only through the API, redacted in logs and exports) and a description for the generated UI. They are stored in `setting (namespace, key, value JSON, version, updated_by, updated_at)` with optimistic concurrency. Each node holds an immutable snapshot behind `atomic.Pointer`; a change publishes `settings.changed`, and every node reloads (through NOTIFY). **No setting requires a restart**: subsystems read the snapshot per operation or subscribe to changes, for example resizing worker pools.
   - **Pinned settings** (for GitOps and Kubernetes): `BONNYREAD_SET__<NAMESPACE>__<KEY>=<json>` (and `_FILE`) overrides the DB value. The API reports such settings as `managed: env` and read-only. This covers "config as code" for settings. Provisioning of *entities* (indexers, clients) is an open question (§6).
   - Secrets stay in the DB (D16). Entity settings (indexer keys, target tokens) live on their entities and are marked secret the same way.

### 3.11 Observability

**Logging**: `log/slog` from the standard library, JSON in containers and text on a TTY. Every record carries `node`, and where relevant `job_id`, `job_kind`, `attempt`, `request_id`, `user_id`, and `trace_id`/`span_id` when tracing is on. Output goes to stdout, plus an optional size-rotated file for desktop installs. Secrets are redacted by a `slog.Handler` wrapper that recognises `secret`-tagged values (through a `LogValuer` on the secret type). The UI log viewer shows: (a) the per-job run log (§3.4), which is persisted and cluster-wide, and (b) a **per-node in-memory ring buffer** (the last 10k records at or above the configured level) for the node serving the request. That is complete on a single node; in HA the page says which node it shows, and platform log aggregation (Loki and similar) is the real answer. Logs are **not** written to the main database: Sonarr's `Logs` DB and `TrimLogDatabase` aren't carried over.

**Metrics** (`prometheus/client_golang`, I13):
- **`/metrics`, per node**: Go runtime and process metrics; `http_requests_total` and duration histograms by route template; `bonnyread_jobs_executed_total{kind,queue,outcome}`; `bonnyread_job_duration_seconds{kind}`; `bonnyread_job_abandoned_total`; worker pool utilisation; SSE connections; `bonnyread_lease_held{name}`; LISTEN reconnects; outbound HTTP by `{target_type, target}` (indexer and metadata source names, which are bounded and user-configured) with status class and latency; plug-in calls, traps and duration; `bonnyread_irc_connected{network}`; extraction bytes and durations (as Unpackerr does).
- **`/metrics/cluster`, DB-derived and identical from any node**, computed on demand and cached 30 s:
  - queue depth by `{queue, state}` and the oldest `available` age;
  - `bonnyread_health_issues{check,severity}`;
  - library counts by `{kind, status}` (monitored, missing, cutoff unmet);
  - download queue by `{client, state}`;
  - requests pending;
  - indexer and client disabled states;
  - `bonnyread_nodes_live`.

  In Kubernetes, scrape `/metrics` per pod (a PodMonitor) and `/metrics/cluster` once through the Service (a ServiceMonitor). This avoids N-fold duplicate series without a leader.
- **Cardinality rules**: no per-item, per-release or per-user labels. Labels come from bounded, admin-configured sets only.
- **Auth**: a scoped API key or bearer token with the `metrics:read` permission (D17), or an unauthenticated separate `METRICS_ADDR` listener meant for in-cluster scraping.

**Tracing**: OpenTelemetry Go (traces and metrics stable; logs at release candidate, [opentelemetry.io/docs/languages/go](https://opentelemetry.io/docs/languages/go/)). It is **off by default** (a noop provider) and switched on by standard `OTEL_EXPORTER_OTLP_ENDPOINT` and related env vars. Spans cover HTTP handlers, outbound `httpx` (the `otelhttp` transport), job execution, ent queries (through an ent driver wrapper, with sampling) and plug-in calls. **Across the queue**, the enqueuing span's W3C `traceparent` is stored in `job.traceparent`, and the job span *links* to it (rather than being a child), because execution may be minutes later on another node. Metrics stay on Prometheus; OTel metrics export isn't needed in v1.

### 3.12 WASM plug-in host (D15)

**Runtime**: `tetratelabs/wazero` v1.12 ([pkg.go.dev](https://pkg.go.dev/github.com/tetratelabs/wazero)). One `wazero.Runtime` per process with:
- `WithCloseOnContextDone(true)`, so a context deadline or cancel terminates guest execution. The docs note a small overhead; it is required for timeouts.
- `WithMemoryLimitPages`, set per plug-in from its manifest, capped by the admin (default 256 pages = 16 MiB, maximum 4096).
- `WithCompilationCache(NewCompilationCacheWithDir(STATE_DIR/wasm-cache))`, keyed by module hash.
- WASI preview 1, with **no filesystem** (the default `FSConfig` denies all), no env, no args, and only walltime and random.

**ABI**: spike **Extism** (`extism/go-sdk`, BSD-3, built on wazero, with plug-in development kits for Go, Rust, JS and others) against a thin custom ABI. Extism gives authors a mature development kit and a memory and serialisation convention. We would **not** use its built-in HTTP; our own host function replaces it. If Extism hides needed wazero controls (per-call timeout, memory cap, instance pooling), fall back to a custom ABI: JSON or MessagePack over linear memory, `alloc`/`dealloc` exports, and one `bonny_call(fn_id, ptr, len)` entry point. Either way the **host ABI is versioned** (`abi: 1`) and plug-ins declare it.

**Package**: a `.bnyplug` zip containing `manifest.json` (id, version, abi, provides (`subtitle_provider` | `indexer` | `ddl_host` | `notification_target`), a settings JSON Schema, the capability request, and `min_host_version`), `plugin.wasm` and an optional `signature`. Signing: minisign/ed25519 keys configured as trusted by the admin, with an explicit "allow unsigned" toggle. The trust root is an open question (§6).

**Storage and HA**: the plug-in row stores the manifest, the sha256, the granted capabilities, the enabled flag and the **wasm bytes in the DB** (a few MB each). Every node sees the same plug-ins with no shared-filesystem dependency. A node compiles on first use, and the result is cached per node by hash.

**Capabilities** are host functions granted per plug-in. The admin approves them at install; a new request on upgrade needs re-approval.

| Host function | Grant | Enforcement |
|---|---|---|
| `http.do(req) → resp` | `http: {hosts: ["*.opensubtitles.com"], private_networks: false}` | Goes through core `httpx`: host allow-list, SSRF guard (RFC 1918, loopback and link-local are blocked unless granted, and the check is repeated after DNS resolution), per-plug-in rate limit and indexer-style budgets, proxy settings, 10 MB response cap, per-request timeout, tracing and metrics. Cookie jars are per plug-in configuration (for scraper logins). |
| `kv.get/set/delete` | `kv: {quota_kb: 1024}` | Namespaced per plug-in configuration, in the DB (so shared in HA). |
| `cache.get/set(ttl)` | default | Per node, in memory, bounded LRU. |
| `settings.get` | default | Only the plug-in configuration's own settings, including the secrets it declared. |
| `log(level,msg)` | default | Rate-limited, tagged `plugin=<id>`. |
| `regex.match/replace` | default | Host-side regexp2 with a timeout (D1), so plug-ins get .NET-compatible patterns consistent with the core. |
| `html.select(doc, css)` | default | Host-side HTML parsing and selection (goquery). This saves every scraper from shipping a parser, and keeps parsing fast. |
| `parser.parse_release(title)` | default | The core release parser (D1c reuse). |

No sockets, no filesystem and no process are available. **Notification-target plug-ins** that post to LAN hosts need `private_networks: true`, which is flagged in the approval UI.

**Instance lifecycle**: compiled module → **pool of instances** per plug-in configuration, sized by its concurrency limit (default 4). Module instances are used by one goroutine at a time. Each call borrows an instance and gets a context with a per-capability timeout (search 30 s, download 60 s, parse 2 s). On a trap, timeout or memory exhaustion, the instance is **discarded**, not reused, and the failure feeds that provider's status backoff. Five traps within 10 minutes auto-disable the plug-in, raise a health issue and emit `plugin.disabled`. **Upgrade and hot reload**: a `plugin.changed` event makes every node compile the new version, route new calls to it, drain old instances once their in-flight calls return, and close the old module. Disable closes the pools.

### 3.13 Graceful shutdown and crash safety

On SIGTERM, with a default grace of 25 s so that it fits within Kubernetes' default 30 s `terminationGracePeriodSeconds` (the deployment document sets probes and the preStop hook):

1. `/healthz/ready` returns 503, and the node waits `READINESS_DRAIN` (default 5 s in Kubernetes, 0 on a single node) so the load balancer stops routing.
2. Stop the scheduler loop and **release all leases** (`expires_at = now()`, then `NOTIFY bonnyread_lease`): IRC fails over in under a second.
3. Workers stop claiming. Running jobs get the remaining grace period to finish. When grace expires, their contexts are cancelled, and each interrupted job is **released**: a fenced `state='available', run_at=now(), attempt=attempt-1` (shutdown doesn't consume an attempt), followed by `NOTIFY bonnyread_jobs`, so another node picks it up at once.
4. Close SSE streams with a `retry: 1000` hint, then shut down the HTTP server (in-flight requests finish, bounded by grace).
5. Remove the node's schedules and node-affine jobs that are still `available`. Mark the `node` row stopped.
6. On SQLite: `PRAGMA optimize`, `wal_checkpoint(TRUNCATE)`, remove the unclean-shutdown marker, close, release the `flock`.

**Crash (SIGKILL, OOM, power loss)**: Postgres or SQLite WAL guarantees committed state. Running jobs are requeued by the reaper after 45 s (on SQLite, at the next startup immediately, since the lock proves no other process). Leases expire after the TTL. SSE clients reconnect elsewhere and resync. Outbox semantics mean no event or notification committed with its business write is lost. An SQLite startup after an unclean shutdown (the marker file is present) runs `PRAGMA quick_check` and raises a health issue if it fails.

### 3.14 Backup and restore

- **SQLite (single node)**:
  - Backups are made with `VACUUM INTO '<tmp>'`: a consistent, compacted copy taken without stopping writers for long. The copy is packaged as `bonnyread-<version>-<ts>.tar.zst` with `manifest.json` (app version, schema version, dialect, created_at, node) and user-installed plug-in bytes. They are already in the DB, so nothing else is needed.
  - Kinds are scheduled (weekly, configurable), manual and pre-migration, with a retention count and a backup directory under `DATA_DIR/backups` by default.
  - **Optional encryption with a passphrase** (age, scrypt recipient), because the archive contains every secret (D16: "backups must be protected the same way").
  - **Restore is staged**: upload or select an archive, then validate the manifest (dialect, and schema version ≤ the binary's), extract it to `DATA_DIR/restore-pending/`, run `PRAGMA integrity_check` on the extracted DB, and restart. The next startup swaps the files *before opening the DB* and keeps the replaced DB as `pre-restore-<ts>.db` (as Profilarr's `applyPending.ts` does). Newer migrations are applied normally after that.
- **Postgres and MySQL**: the app **does not** take physical backups. Point-in-time recovery, `pg_dump` and Kubernetes operators (CloudNativePG and similar) are the operator's job, and the deployment document gives recipes. A health check warns when no external-backup acknowledgement setting is set.
- **Logical export and import (all dialects)**: `bonnyread db export` / `db import`, also available as a job and API, streams all ent entities in dependency order to NDJSON inside the same archive format, dialect-agnostic. This is the **SQLite → Postgres migration path** when a user outgrows one node (`bonnyread db copy --from sqlite:… --to postgres:…`), and a portable backup for MySQL and Postgres users who want one. A **sanitised** variant (secrets stripped, as Profilarr's `sanitize.ts` does) exists for sharing configurations or bug reports, clearly labelled as not restorable without re-entering secrets.

### 3.15 Storage assumptions

- `DATA_DIR` is **shared** in HA (D14) and holds backups, profile-database git clones (D9) and the image cache. Media roots and download paths are mounted at identical paths on every node.
- `STATE_DIR` is **node-local**: the lock file, the wasm compilation cache, temp and extraction scratch space (unless a download's extraction target is on shared storage) and log files.
- **Mount verification**: each root folder gets a marker file `.bonnyread-root` containing the root's UUID, written on creation. A node-scoped health check reads it on every node, which catches a root that is mounted differently or missing on one replica before an import on that node fails. Import jobs re-check it before writing.
- **Git clones** on shared storage: profile-database pulls use the concurrency key `pcd:<db_id>`, so two nodes never run git in the same working tree at once.
- **SQLite must not live on network filesystems.** This is documented, and enforced as far as possible by the `flock` (which NFS may not honour; this is the documented risk).

---

## 4. Walkthroughs

### 4.1 A node dies mid-RSS-cycle

Setup: three nodes A, B and C on Postgres. A holds the lease `irc:3` (token 41) and is running `rss:indexer:7` (job 9001, attempt 1). RSS was enqueued from the schedule row `rss:indexer:7` with dedupe `WhilePendingOrRunning`.

1. **t0**: A has fetched and parsed the feed and evaluated 50 releases. In `FencedTx` it committed a grab decision for episode E (the `grab` job 9002, dedupe `grab:item:E:slot:1080p`, together with a pending-grab row). Then A is killed (OOM).
2. **t0 to t0+15 s**: lease `irc:3` expires. B's lease poller (≤ 2 s) sees it free; B holds 1 lease, which is under `ceil(4/3) = 2`. It acquires the lease with token 42 and connects to the IRC network. Announces during the gap (about 15–20 s) are missed, and the next RSS run of that indexer covers them, since announce and RSS overlap by design.
3. **The grab job 9002** was committed, so it sits `available`. C claims it within the poll interval or the doorbell and sends it to the download client. The download agent's idempotency rule applies: the client-side label or tag carries `grab_id=9002`, so a retry first checks for an existing entry, and a torrent is also idempotent by infohash.
4. **t0+45 s**: some node's reaper sees job 9001 with `heartbeat_at < now()-45 s` and makes it `available`. B claims attempt 2 and refetches the feed. RSS is idempotent: the decision engine sees the pending grab for E (a "queue/pending" rejection) and the dedupe key blocks a second `grab` job, so nothing is grabbed twice. Releases A never reached are evaluated now.
5. **The schedule**: A's death doesn't affect it. Any node's scheduler loop enqueues the next RSS run at `next_run_at`, and meanwhile the dedupe key blocks a double enqueue.
6. **What if A was only paused** (a 60 s stall or a partition) and then resumes? A's lease renewer finds no successful renewal for over 10 s and has already cancelled the IRC context (self-demotion). Any announce-derived enqueue A attempts runs `lease.Fence(tx, "irc:3", 41)` → fails. A's RSS worker tries to complete job 9001 with `attempt=1` → 0 rows → the result is discarded; its `FencedTx` writes also fail. **No stale writes.**

**Failover time**: IRC 15–20 s, an in-flight job 45–60 s, new work immediately. On a *graceful* stop of A: IRC in under 1 s, and the in-flight RSS job is released at the end of the grace period and claimed at once.

### 4.2 A job enqueued on node A, executed on node B, with its event reaching a browser on node C

1. The user's browser holds an SSE stream to **C** (the load balancer picked it). The user clicks "Search 4K" for movie 42; that `POST /api/v1/…/search` lands on **A**.
2. A: `tx { jobs.Enqueue(SearchItem{42,"4k"}, priority=interactive, dedupe="search:item:42:4k", created_by=u) }` commits. After commit: `NOTIFY bonnyread_jobs,'default'` and `NOTIFY bonnyread_events,{job:created, id, audience:[u, perm:view_activity]}`. A returns `201 {job_id}`.
3. The doorbell reaches every node. **B** has an idle `default` worker and wins the `FOR UPDATE SKIP LOCKED` claim (A and C skip the locked row or find nothing).
4. B runs the search and reports progress (`Searching 6 indexers…`, `3/6`) at most once per second. Each report is an `UPDATE job.progress` plus `NOTIFY bonnyread_events,{job:progress,…}`.
5. **C**'s listener receives each notice, filters by audience (u is allowed), and writes an SSE message `id: c7:1182`, `event: job`, data `{id, progress}` to u's stream. The browser updates the progress in the command toast.
6. B finishes: `FencedTx { decision rows; enqueue grab job; events.Publish(release.grabbed) → enqueues notify.route in the same tx; complete job 9005 }` commits. After commit it sends one NOTIFY with `[job:completed, movie:updated 42, queue:updated]`.
7. C forwards the invalidations. The browser refetches `GET /movie/42` and the queue; the load balancer may send those to A. Postgres delivers notifications **only after commit**, and all nodes read the primary, so A's read sees B's commit.
8. **Failure cases**: if C's LISTEN connection blipped during step 6, then on reconnect C sends `resync` to all its SSE clients, and the browser refetches its active queries. If the browser reconnects to A with `Last-Event-ID: c7:1182`, A sees a foreign epoch and sends `resync`.

### 4.3 SQLite single node, from startup to shutdown (desktop service or small container)

1. The service starts `bonnyread` (D12 OS service). Config: `DATA_DIR=~/…/bonnyread`, `DB_DSN` unset, so the default is `sqlite://$DATA_DIR/bonnyread.db`.
2. Take the `flock` on `bonnyread.lock`. If it is held, exit with "another instance is running".
3. Open the `RW` (1 conn, immediate transactions) and `RO` pools with WAL, busy timeout and foreign keys. The unclean-shutdown marker is present from a crash, so run `PRAGMA quick_check` → ok.
4. Pending migration 0042: `VACUUM INTO backups/pre-migrate-0041-<ts>.db`, then apply Atlas SQLite migrations 0042 inside a transaction.
5. Recovery: `UPDATE job SET state='available' … WHERE state='running'`. The lock proves these belonged to the dead process, so there's no 45 s wait and the attempt is not decremented, since it was a crash.
6. Build: the lease backend is in-memory and always leader, the cluster notifier is a noop, and the event bus delivers after-commit to the local SSE hub. Load the settings snapshot and compile enabled plug-ins (the compilation cache makes this fast after the first run).
7. Start: node heartbeat (still written, so the UI "system" page is uniform), workers (`default` 4, `io` 1: smaller defaults on a single node, from `runtime.NumCPU`), IRC connections (the lease is granted at once), the scheduler loop (the same SQL without `SKIP LOCKED`; overdue schedules coalesce and fire once, with jitter), and HTTP. Then create the marker and set ready = true.
8. Steady state: an enqueue is followed by an in-process poke, so a worker picks the job up within milliseconds. There's no NOTIFY and no polling other than the 2 s idle fallback, which is cheap on SQLite (an index-only query).
9. The user quits from the tray, and the service gets SIGTERM. Readiness goes false (no drain delay on a single node), IRC disconnects, workers get 15 s of grace, interrupted jobs are released (`available`, attempt restored), SSE closes with `retry`, and the HTTP server shuts down. Then `PRAGMA optimize`, `wal_checkpoint(TRUNCATE)`, remove the marker, close the DB, release the `flock`, exit 0.

---

## 5. Simplifications compared with the references

| Reference mechanism | Replaced by | Why it's simpler or more correct |
|---|---|---|
| NzbDrone in-memory `CommandQueue` mirrored to the `Commands` table, 3 threads, `(dynamic)` dispatch | One `job` table is the only source of truth; typed handlers registered at build time; per-queue pools | No dual state. It works across nodes. Dispatch can be checked at compile time. |
| `CommandEqualityComparer` reflection dedupe | An explicit dedupe key per kind with a pending or running policy | Deterministic. Adding a field can't change the semantics. |
| Global `IsExclusive`, `IsLongRunning` and `RequiresDiskAccess` | Per-entity `concurrency_key` plus queue pools | More parallelism (series A and B import concurrently) with the same safety per entity (I4), cluster-wide. |
| Scheduled-task table plus a hard-coded default list plus one global RSS interval | Schedule rows owned by modules and entities, jitter, cron parser | Per-indexer cadence, no stampede, no reconciliation code, schedules cascade-deleted with their owner. |
| APScheduler "cron in a century" for manual; Quartz, node-schedule, croner, robfig: 5 schedulers | `spec_kind = manual`; one loop | One implementation, and it is HA-safe without a leader. |
| Leader-elected singleton for RSS, polling and git pulls (as D14 literally reads) | Leaderless: schedule rows and SKIP LOCKED plus dedupe keys; leases only for long-lived connections | Fewer failover paths. Work spreads over all nodes. A dead leader doesn't stall periodic work. |
| Fire-and-forget async event handlers (`IHandleAsync`), MassTransit in-memory, ORM-subscriber side effects (Seerr) | After-commit listeners (best-effort UI and cache) versus durable subscribers as outbox jobs in the same transaction | Side effects survive crashes and retry independently. Core flow stays as direct calls. |
| SignalR hubs, Bazarr's three SignalR clients, webhooks and hourly sync | In-process events plus one NOTIFY channel plus SSE invalidations with a resync fallback | No event store, no replay protocol, and correct after any gap. |
| Seven notification systems | One catalog, one envelope, route jobs and deliver jobs, backoff per target | Per-user routing everywhere, durable retries, one documented webhook and script contract (D7). |
| `Sonarr_*` env vars for custom scripts | The envelope on stdin or in a file, plus 3 env vars | One schema for webhooks and scripts. |
| Health text parsed by other apps; `ServerSideNotificationService` | Coded issues with subjects; transitions become events; Alertmanager sink | Machine-actionable, and alertable (D7). |
| 31–38 housekeepers, mostly orphan cleanup | FK cascades plus about 8 retention and maintenance jobs | Most of the code disappears. |
| Per-app backup formats; no Postgres story | One archive format; `VACUUM INTO`; staged restore; logical export and `db copy` | Consistent everywhere, and there's a migration path from SQLite to Postgres. |
| `config.xml` plus Options plus the DB `Config` table; Seerr settings-file migrations | Bootstrap env and file (with `_FILE`) plus typed DB settings with an env pin | No overlapping keys, no restarts to apply settings. |
| Logs DB plus `TrimLogDatabase` | stdout, an optional file, a node ring buffer and per-job run logs | No log writes to the main database (important on SQLite). |
| Lidarr's in-process .NET plug-ins | wazero sandbox, capability grants, instance pools, auto-disable | I15. |

---

## 6. Issues with decisions, and open questions

### Issues with decisions

1. **D13 (MySQL/MariaDB support) against D14 and the runtime.**
   - D14 makes Postgres mandatory for HA, so MySQL can only be a single-node tier, where SQLite already serves.
   - MySQL costs a third claim dialect (no `UPDATE … RETURNING`), no partial indexes (the dedupe tables above exist partly for portability), no LISTEN/NOTIFY, a third Atlas migration tree, and a third CI matrix leg for every platform conformance test.
   - It also rules out River as a future backend (River has Postgres and experimental SQLite only).

   **Recommendation:** reconsider MySQL, or declare it "single-node, best effort". The queue design above supports it, but at a real ongoing cost. *We have not deviated.*
2. **D14 wording, "singleton work (… download-client polling, RSS and other schedules, profile-database git pulls) runs under leader election".** This design meets the outcome (at most one execution per due time or subject) **without** leader election for those items, using schedule rows with `SKIP LOCKED` and dedupe or concurrency keys. Leases are kept for long-lived connections (IRC). Please confirm this reading. It is a refinement rather than a deviation, but it changes what the "leader" term means in other documents.
3. **D14 "cross-node events via Postgres LISTEN/NOTIFY".** This is adopted, but **only as a best-effort doorbell and invalidation channel**. Reliable delivery (notifications, webhooks, follow-up work) must go through the transactional job queue, because NOTIFY loses messages while a listener is disconnected and takes a global lock at commit. Other documents shouldn't design anything that relies on NOTIFY being delivered. Deployment consequence: the listener needs a direct or session-pooled connection (no PgBouncer transaction mode).
4. **D16 (secrets in the DB) and backups.** SQLite backups and logical exports contain every secret in plaintext. D16 says backups must be protected "the same way" as the DB, but a backup file is routinely copied off the disk. This design adds **optional archive encryption** (age passphrase) and a clearly labelled sanitised export. Confirm that this is in the spirit of D16 and doesn't count as the "encryption at rest" it declined.
5. **D12 "`/metrics`" on the core.** With N active-active replicas, DB-derived gauges would be duplicated N times. This design splits them into per-node `/metrics` and `/metrics/cluster`. That extends D12's single endpoint; please confirm.
6. **D15 "notification targets" as plug-ins.** Targets often post to LAN hosts, which the default SSRF guard blocks. Plug-in targets therefore need an explicit `private_networks` grant, which weakens the sandbox for that plug-in. Acceptable, but it should be a visible admin decision.
7. **D13 "one schema, Atlas versioned migrations" and rolling upgrades (D14/D18).** Active-active rolling deploys require expand/contract migrations and N-1 compatibility. This is a process rule the decisions don't state; it constrains every domain document's schema changes.

### Open questions

1. **Extism or a custom ABI** for plug-ins (a spike is needed: per-call timeouts, memory caps and instance pooling through Extism's wrapper), and the **signing trust root** (a project key, per-publisher keys, or admin-managed keys only).
2. **Declarative provisioning** of entities (indexers, clients, targets) for GitOps, in the style of Grafana provisioning. Settings pinning (§3.10) covers settings only.
3. **The cluster-wide log viewer.** Is the per-node ring buffer plus per-job logs enough, or should WARN+ records go to a bounded DB table (costly on SQLite)?
4. **Cluster-wide rate budgets** for indexers and metadata sources. The `httpx` budget store needs a design shared with the indexer document: DB-backed "next allowed at" tokens are simple and correct at our request rates, but each call costs a DB round trip.
5. **Job history exposure.** How long should completed jobs stay visible in the UI's "tasks" and "queue" pages, and is a separate summarised `job_run` history table (as Profilarr has) worth it?
6. **Cost of `WithCloseOnContextDone`** for hot plug-in paths (for example subtitle scoring). Measure it in the spike; if it's high, use coarser-grained calls.
7. **The minimum Postgres version** (15 is proposed for `MERGE` and newer features) and whether PgBouncer transaction pooling is supported at all, beyond the listener exception.
8. **`db copy` from SQLite to Postgres as a supported, tested upgrade path.** This is recommended, since D18 expects users to move from a desktop install to HA.

Sources consulted for library and protocol facts (2026-09-28): [River database drivers](https://riverqueue.com/docs/database-drivers), [River SQLite](https://riverqueue.com/docs/sqlite), [River leader election](https://riverqueue.com/docs/leader-election), [River unique jobs](https://riverqueue.com/docs/unique-jobs), [River periodic jobs](https://riverqueue.com/docs/periodic-jobs), [River repo (MPL-2.0)](https://github.com/riverqueue/river), [goqite](https://github.com/maragudk/goqite), [ent feature flags](https://entgo.io/docs/feature-flags), [wazero API](https://pkg.go.dev/github.com/tetratelabs/wazero), [Extism Go SDK](https://github.com/extism/go-sdk), [netresearch/go-cron](https://pkg.go.dev/github.com/netresearch/go-cron), [OpenTelemetry Go status](https://opentelemetry.io/docs/languages/go/), [Alertmanager alerts API](https://prometheus.io/docs/alerting/latest/alerts_api/), [Postgres LISTEN/NOTIFY does not scale (Recall.ai)](https://www.recall.ai/blog/postgres-listen-notify-does-not-scale), [LISTEN/NOTIFY actually scales (DBOS)](https://www.dbos.dev/blog/postgres-listen-notify-scalability), [MySQL locking reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html), [MariaDB SELECT … SKIP LOCKED](https://mariadb.com/kb/en/select/#skip-locked).
