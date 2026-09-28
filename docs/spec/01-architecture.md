# 01. Architecture

This file specifies the overall shape of bonnyread: the system context, the client–server split, the three deployment topologies at an architectural level, the Go code layout and its dependency rules, the process model, the high-availability (HA) model, the consistency model, rolling upgrades and version skew, capability-routed jobs, storage assumptions and the security baseline. The platform primitives it relies on (job queue, scheduler, leases, events, rate limiter, plug-in host, configuration, backup) are specified in `12-platform.md`; packaging, installers, images and the Helm chart are in `13-deployment.md`; authentication and authorisation in `10-identity-and-access.md`; API conventions and SSE framing in `11-api-and-clients.md`. Applies to all phases (P1, P2, P3); everything here is required from P1 unless a requirement says otherwise.

## Concepts

| Term | Definition |
|---|---|
| **Core** | The headless Go server process (`bonnyread` binary). Holds all business logic and state access. |
| **Client** | Anything that talks to the core over its API: the embedded web SPA, the Tauri desktop app, later Tauri mobile apps, the CLI, generated SDKs, and scraping or alerting systems. |
| **Node** | One running core process. In HA there are N nodes sharing one PostgreSQL database. Each node has a stable `node_id`. |
| **Role** | A set of services a node runs: `api` (HTTP, SSE) and `worker` (job workers, scheduler loop, lease manager). Default: both. |
| **Platform** | The runtime kernel packages under `internal/platform/`. They know nothing about media. |
| **Domain** | The pipeline packages (acquisition, evaluation, downloads, import, library, profiles, metadata, subtitles, identity, requests). |
| **Media-kind module** | A package under `internal/kinds/<kind>/` (tv, movie, music, book, adult, comic) that contributes kind-specific behaviour through `registry.Module`. |
| **Job** | A durable unit of background work in the database-backed queue (`12-platform.md` §4). |
| **Schedule row** | A database row that periodically enqueues a job (`12-platform.md` §5). Replaces "leader-scheduled" work. |
| **Lease** | A database row granting one node exclusive ownership of a long-lived connection, with a fencing token (`12-platform.md` §6). |
| **Doorbell** | A best-effort Postgres `NOTIFY` that tells other nodes to look at the database now. Never the carrier of anything that must be delivered. |
| **Fencing** | Guarding a write so that a node which lost ownership of a job attempt or lease cannot commit. |
| **Capability** | A native tool a node advertises (for example `ffprobe`, `fpcalc`, `alass`, `ffsubsync`), used to route jobs. |
| **Expand/contract** | A migration discipline: release N only adds schema; destructive changes land in a later release once no N-1 code needs the old shape. |

Alias: design documents say "leader election" for D14 singleton work. After D20 and open-issues A1 that term applies only to leases for long-lived connections; this specification says **lease**.

## Requirements

### 1. System context

```
                         ┌──────────────── users ────────────────┐
                         │ browser     Tauri desktop     phone    │
                         └────┬────────────┬───────────────┬──────┘
                              │ HTTPS: /api/v1 (REST), /api/v1/events (SSE), SPA
                              ▼
 Prometheus ──/metrics──► ┌───────────────────────────────────────────┐ ──► Alertmanager v2
 scripts/CLI ──API key──► │            bonnyread core (1..N nodes)     │ ──► notification targets,
                          │  platform · domain · kinds · api            │     webhooks, scripts
                          └──┬───────────┬─────────────┬───────────┬───┘
                             │           │             │           │
              PostgreSQL ≥15 or SQLite   │             │           │ file I/O
              (all durable state)        │             │           ▼
                                         │             │     media roots, download dirs,
          outbound HTTP (httpx, rate-limited)          │     staging (same paths on every node)
          ├─ indexers / trackers (Cardigann, Newznab/Torznab, native, DDL)
          ├─ metadata sources (TVDB, TMDb, MusicBrainz, CAA, AcoustID, Hardcover, …)
          ├─ subtitle providers (WASM plug-ins)
          ├─ definition/PCD/plug-in feeds (GitHub, by commit)
          └─ FlareSolverr/Byparr, proxies
                                                       │ IRC (long-lived, leased)
          download clients (qBittorrent, SABnzbd, …) ◄─┘ media servers (Plex/Jellyfin/Emby)
```

**ARC-001** (MUST) bonnyread **MUST** be a single Go binary (the core) that contains all business logic, the REST API, the SSE endpoint, the embedded web UI and every built-in provider. There **MUST NOT** be separate services for indexers, subtitles, requests or any other area. _Source: D2, D12; design/runtime.md §3.1._

**ARC-002** (MUST) All durable state **MUST** live in one relational database: PostgreSQL ≥ 15 or SQLite. Another datastore (Redis, message broker, object store, shared data volume) **MUST NOT** be required in any topology. _Source: D13 as amended by D20; open-issues A2, B5._

**ARC-003** (MUST) The core **MUST NOT** depend on any hosted service operated by the project in v1. Outside services are called directly (metadata sources, trackers, definition feeds). _Source: D3c._

**ARC-004** (MUST NOT) The core **MUST NOT** expose compatibility façades for other applications' APIs (Sonarr/Radarr v3, Lidarr/Bookshelf v1, Seerr v1, Newznab/Torznab server, Bazarr). Webhooks and custom scripts use bonnyread's own versioned format only (`12-platform.md` §9). _Source: D7._

**ARC-005** (MUST) The core **MUST** expose Prometheus-scrapeable metrics and dispatch alerts to external alerting systems as first-class features, built on the same event system as user notifications (`12-platform.md` §§8–12). _Source: D7._

### 2. Client–server model and clients

**ARC-006** (MUST) The core **MUST** be headless: every user-facing function **MUST** be reachable through the public API (`/api/v1`, described by an OpenAPI 3.1 document) and the live event stream (SSE). The web UI **MUST NOT** use private endpoints. _Source: D12; design/identity-api-deployment.md §3.6._

**ARC-007** (MUST) All interactive clients **MUST** share one SvelteKit static SPA (Svelte 5, static adapter, D6). The core **MUST** embed the SPA build with `go:embed` and serve it at the configured base path. _Source: D6, D12._

**ARC-008** (MUST) The supported client set is:

| Client | Phase | Transport | Credential |
|---|---|---|---|
| Web SPA served by the core | P1 | same-origin fetch + `EventSource` | session cookie |
| Tauri desktop app | P1 | native (Rust) HTTP client exposed to the UI; SSE via fetch stream | device credential in the OS keychain |
| Tauri mobile apps | later | as desktop; HTTPS only | device credential |
| CLI (`bonnyread` subcommands and the generated Go client) | P1 | HTTP, or direct DB access for offline admin commands (`migrate`, `backup`, `restore`, `db …`) | API key, or local OS access |
| Generated SDKs | P1 (spec published only) | HTTP | API key |
| Metrics scrapers and alert sinks | P1 | `/metrics`, outbound Alertmanager v2 | service-account API key |

Details are in `11-api-and-clients.md`. _Source: D12, D18; design/identity-api-deployment.md §3.8._

**ARC-009** (MUST NOT) Clients **MUST NOT** contain business logic that the server does not also enforce. Crux or any shared client-side core **MUST NOT** be used. _Source: D12._

**ARC-010** (MUST) Native apps **MUST** negotiate compatibility through `GET /api/v1/meta` (`apiVersion`, `minClientApiVersion`). When the bundled UI is incompatible with the server, the app **MUST** offer the server's own web UI in the system browser and **MUST NOT** load remote UI code into its privileged webview. _Source: open-issues A7; design/identity-api-deployment.md §3.8._

**ARC-011** (MUST NOT) v1 **MUST NOT** include mobile push notifications (APNs/FCM). Background notification on phones is served by existing notification targets (for example ntfy, Pushover, Telegram). _Source: D20; open-issues B13._

### 3. Deployment topologies

The three topologies share one binary and one code path; they differ only in database, number of nodes and how clients reach the core. Packaging detail is in `13-deployment.md`.

```
A. Desktop, single user                       B. Home server / NAS
┌──────────── one machine ─────────────┐      browser / Tauri / phone
│ Tauri app ──127.0.0.1:7979──► core    │           │ HTTPS
│  (UI, tray,          (OS service,     │           ▼
│   keychain)           SQLite)         │      reverse proxy ──► core :7979 (1 node)
│      └── local socket / named pipe ──►│                         SQLite or PostgreSQL
│          (pairing, OS peer creds)     │                         /data /media /downloads
└───────────────────────────────────────┘

C. Kubernetes HA
 clients ─HTTPS─► Ingress ─► Service ─► N core pods (stateless, all roles)
                                          │  admin listener :9979 (/metrics, /livez, /readyz)
                                          ├─► PostgreSQL ≥ 15 (managed or operator)
                                          └─► RWX volumes: media, downloads, staging
                                              (identical mount paths on every pod)
 Prometheus ─► per-pod /metrics (PodMonitor) + /metrics/cluster once via Service
```

**ARC-012** (MUST) Topology A (desktop) **MUST** run the core as an OS service (not as a child of the app) on SQLite, listening on loopback by default, and **MUST** pair the desktop app over a local socket or named pipe authenticated by OS peer credentials, issuing a revocable device credential. The service **MUST** keep running when the app is closed. _Source: D12 as amended by D20; D18; open-issues A15, B13._

**ARC-013** (MUST) Topology A **MUST** allow opting in to remote access ("allow access from other devices"), which binds non-loopback addresses and enables TLS; the security prerequisites for that switch are in `10-identity-and-access.md`. _Source: design/identity-api-deployment.md §3.10._

**ARC-014** (MUST) Topology B (home server) **MUST** support one node on SQLite or PostgreSQL, run from a container image or OS package, behind an optional reverse proxy, including hosting under a sub-path. _Source: D18; design/identity-api-deployment.md §3.10._

**ARC-015** (MUST) Topology C (Kubernetes HA) **MUST** support N ≥ 2 active-active nodes on PostgreSQL behind a load balancer with no sticky sessions. Running more than one node against SQLite **MUST** be impossible (`12-platform.md` §1, data-directory lock). _Source: D14, D18._

**ARC-016** (SHOULD) The macOS desktop install **SHOULD** default to a per-user agent, with a system daemon package for "server mode" [OPN-023]. _Source: open-issues Group C (identity)._

**ARC-017** (MUST) A single node **MUST** be the default deployment in every topology; HA is opt-in and needs no code or feature differences, only configuration. _Source: D14; design/runtime.md invariant N3._

**ARC-018** (MUST) A single-node SQLite installation **MUST** be upgradable to PostgreSQL (single node or HA) through `bonnyread db copy` [OPN-072]. _Source: open-issues Group C (runtime); design/runtime.md §3.14._

### 4. Code organisation

```
cmd/bonnyread/            main: flags → config → app.Run; subcommands
internal/
  platform/               runtime kernel — imports NO domain/kinds/api package
    config/ db/ store/ lifecycle/ jobs/ lease/ lock/ sched/ events/ notify/ health/
    housekeeping/ backup/ obs/ httpx/ ratelimit/ regex/ plugin/ registry/
  domain/<area>/          acquisition, evaluation, download, importing, library, profile,
                          metadata, subtitle, identity, request, …
  kinds/<kind>/           tv, movie, music, book, adult, comic
  api/                    REST + SSE handlers (generated from OpenAPI)
  app/                    composition root
plugins/<kind>/<id>/      first-party WASM plug-in sources, built to .wasm in CI
```

**ARC-019** (MUST) The repository **MUST** be one Go module, `github.com/gpdl49/bonnyread`, producing one binary, built with `CGO_ENABLED=0`. _Source: design/runtime.md §3.1; open-issues A18._

**ARC-020** (MUST) Wiring **MUST** be explicit in the composition root (`internal/app`). There **MUST NOT** be a dependency-injection container, `init()`-based registration or reflection-based dispatch. _Source: design/runtime.md §3.1; D19._

**ARC-021** (MUST) Package dependencies **MUST** obey these rules:

| # | Rule |
|---|---|
| 1 | `platform/*` imports only the standard library, third-party libraries, other `platform/*` packages and the generated `store/ent`. It never imports `domain`, `kinds`, `api` or `app`. |
| 2 | `domain/*` imports `platform/*` and other `domain/*` packages in an acyclic order. A would-be cycle is broken by an interface owned by the lower package or by a durable event subscriber. |
| 3 | `kinds/*` import `domain/*` and `platform/*`, never each other. |
| 4 | `api` imports `domain`, `kinds` (through interfaces) and `platform`. Nothing imports `api` or `app`. |

_Source: design/runtime.md §3.1._

**ARC-022** (MUST) The dependency rules **MUST** be enforced by an automated test in CI that walks `go list -deps` and fails on any violation. _Source: design/runtime.md §3.1._

**ARC-023** (MUST) Core control flow that must be atomic with a write **MUST** be a direct function call inside the same transaction, not an event. Events are used only for side effects that may run later (`12-platform.md` §7). _Source: design/runtime.md §3.1, §3.6; D19._

**ARC-024** (MUST) The ent schema **MUST** live in the single package `internal/platform/store/schema` (an ent constraint). Ownership **MUST** be expressed by file prefix (`platform_*.go`, `library_*.go`, `acquisition_*.go`, …) and each area's specification owns the tables in its files. Shared entities are owned by `02-domain-model.md`. _Source: design/runtime.md §3.1._

**ARC-025** (MUST) Every media kind and every group of built-in providers **MUST** contribute to the system only through the `registry.Module` interface:

```go
type Module interface {
    Name() string
    Register(r *registry.Registrar) error
}
// Registrar methods: Job, Schedule, Event, Settings, Health, Housekeeper, Provider, Metrics
```

The registrar **MUST** be frozen before the lifecycle starts; registration after freeze is a programming error that fails startup. The Registrar API is specified in `12-platform.md` §3. _Source: design/runtime.md §3.1._

**ARC-026** (MUST) The composition root **MUST** build the module list explicitly. Phase-3 media kinds (book, adult, comic) **MUST** be excludable with a build tag without breaking the build. _Source: design/runtime.md §3.1; D2._

**ARC-027** (MUST) The data model and pipeline **MUST** support every media kind from P1, so that adding a kind module in P2 or P3 needs no schema rewrite of shared entities. _Source: D2; open-issues A17._

**ARC-028** (MUST) WASM plug-ins **MUST** register into the same provider registry as built-in providers through an adapter, so that callers cannot distinguish a plug-in from a built-in provider. _Source: D15; design/runtime.md §3.1, §3.12._

### 5. Process model

**ARC-029** (MUST) Every node **MUST** be equal: there is no primary node and no leader for scheduled or queued work. A node's roles **MUST** be selectable (`ROLES=api,worker`, default both); a node with only `api` **MUST NOT** claim jobs, run the scheduler loop or hold leases. _Source: D14; open-issues A1; design/runtime.md §3.2._

**ARC-030** (MUST) Startup **MUST** proceed in this order, each step blocking the next: configuration → storage guard → database open → migrate (or verify compatibility) → build and freeze the registry → start services → ready. The detailed sequence is `12-platform.md` §3. _Source: design/runtime.md §3.2._

**ARC-031** (MUST) Readiness (`/readyz`) **MUST** reflect only the node's ability to serve (database reachable, schema compatible, event listener subscribed where applicable, not draining). Domain health (an indexer down, a disk full) **MUST NOT** affect readiness. _Source: design/runtime.md §3.8._

### 6. High-availability model

D14 as amended by D20 and open-issues A1–A4:

```
 node A                    node B                    node C
 ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
 │ scheduler    │──┐       │ scheduler    │──┐       │ scheduler    │──┐  claim due schedule
 │ workers      │  │       │ workers      │  │       │ workers      │  │  rows, claim jobs:
 │ lease mgr    │  │       │ lease mgr    │  │       │ lease mgr    │  │  FOR UPDATE SKIP LOCKED
 │ LISTEN conn  │  │       │ LISTEN conn  │  │       │ LISTEN conn  │  │
 └──────▲───────┘  │       └──────▲───────┘  │       └──────▲───────┘  │
        │ NOTIFY (doorbell only, best-effort)│              │          │
        └─────────────────┬──────────────────┴──────────────┘          │
                          ▼                                            ▼
      PostgreSQL: job · schedule · lease(irc:*, watch:*) · rate_limit · event_log · all domain state
```

**ARC-032** (MUST) Periodic and polling work (RSS, download-client polling, list sync, metadata refresh planning, definition and PCD feed checks, housekeeping, health, backups) **MUST** run from **schedule rows** that every node's scheduler claims with `SELECT … FOR UPDATE SKIP LOCKED`, turning each due row into a dedupe-keyed job. It **MUST NOT** depend on a leader. _Source: D20 (D14 amended); open-issues A1._

**ARC-033** (MUST) "At most one execution per subject" **MUST** be achieved with job dedupe keys and concurrency keys (`12-platform.md` §4), not with leader election. Download-client polling **MUST** be one schedule row per client with the concurrency key `dlclient:<id>`. PCD git pull, link, publish and update-preview **MUST** run as jobs with the concurrency key `pcd:<db_id>`. _Source: open-issues A1._

**ARC-034** (MUST) Leases **MUST** be used only for long-lived connections or sessions that cannot be expressed as jobs: IRC announce connections (`irc:<network_id>`), filesystem watchers on root folders (`watch:<root_id>`, `06-import.md` IMP-082), and any future long-lived external connection. _Source: D20; open-issues A1._

**ARC-035** (MUST) Runtime-fetched content — indexer (Cardigann) definitions, IRC definitions, PCD git data, and plug-in modules — **MUST** be stored in the database. Nodes **MAY** keep disposable local caches of it, keyed by content hash. A shared data volume **MUST NOT** be needed for this content. _Source: D20; open-issues A2._

**ARC-036** (MUST) Per-tracker request spacing, indexer query and grab budgets, and strictly limited metadata sources (MusicBrainz, ComicVine, Hardcover, Metron, AcoustID) **MUST** use the database-backed shared rate limiter (`12-platform.md` §18), so the limits hold across all nodes. Interactive traffic **MUST** have reserved lanes. _Source: D20; open-issues A3._

**ARC-037** (MUST) Postgres `LISTEN/NOTIFY` **MUST** be used only as a best-effort doorbell and cache-invalidation channel. Anything that must be delivered (notifications, webhooks, follow-up work, SSE replay) **MUST** go through the transactional job queue or a database table. A component **MUST NOT** behave incorrectly if a notification is lost; a lost notification only adds latency. _Source: D20; open-issues A4._

**ARC-038** (MUST) The LISTEN connection **MUST** be a session-level connection (direct, or PgBouncer in session mode). PgBouncer transaction pooling **MUST NOT** be used for the listener [OPN-071]. _Source: open-issues A4; Group C (runtime)._

**ARC-039** (MUST) Every coordination comparison (job claims, heartbeats, lease expiry, schedule due times, rate-limit slots) **MUST** use database time (`now()`), never node clocks. _Source: design/runtime.md invariant N2._

**ARC-040** (MUST) A single SQLite node and an N-node PostgreSQL cluster **MUST** run the same code paths. Only the claim SQL, the doorbell (no-op on SQLite), the lease backend (in-memory, always granted, on SQLite) and the rate-limiter backend (in-memory on SQLite) **MAY** differ, each behind one interface with a shared conformance test suite. _Source: design/runtime.md invariant N3._

**ARC-041** (MUST) Failover targets: work held under a lease **MUST** fail over within 20 s of a crash and within 1 s of a graceful shutdown; a job whose node crashed **MUST** become claimable within 60 s; new work **MUST** be claimable by surviving nodes immediately. _Source: design/runtime.md §3.5, §4.1._

Work placement summary (normative through the requirements above):

| Work | Placement | Mechanism |
|---|---|---|
| HTTP API, SSE | any `api` node | stateless; sessions, idempotency keys and rate counters in the DB |
| RSS polls, download-client polls, list syncs, refresh planning, feed checks | any `worker` node | schedule row → dedupe-keyed job |
| Searches, grabs, imports, renames, extraction | any `worker` node (capability permitting) | job with concurrency key per entity |
| IRC connections, file watchers | one node per lease, balanced | lease with fencing token |
| Tracker login, solver challenge | any node, single-flight | row claim on the session or clearance row (`12-platform.md` PLT-062) |
| Rate spacing, budgets, cooldowns | any node | shared rate limiter rows |
| Node-scoped health checks | every node | per-node schedule, node-affine jobs |

### 7. Consistency model

**ARC-042** (MUST) All reads and writes **MUST** go to the primary database. Read replicas **MUST NOT** be used in v1, so that a read on any node after a commit on any node observes that commit (read-after-commit). _Source: design/runtime.md §3.3, §4.2._

**ARC-043** (MUST) A business write and the jobs and events it causes **MUST** commit in one transaction (the job queue is the transactional outbox). An event is never delivered for a rolled-back write, and no durable side effect is lost for a committed one. _Source: design/runtime.md §3.4, §3.6; open-issues A4._

**ARC-044** (MUST) Job execution is at least once. Every job handler **MUST** be idempotent with respect to database state (via fenced transactions) and **SHOULD** be idempotent with respect to external systems (for example a grab tagged with its grab id, torrents deduplicated by infohash). _Source: design/runtime.md §3.4 (fencing), §4.1._

**ARC-045** (MUST) A worker that lost ownership of a job attempt or a lease **MUST NOT** be able to commit domain writes on its behalf (fencing, `12-platform.md` §§4, 6). _Source: design/runtime.md invariant N1._

**ARC-046** (MUST) Clients **MUST** treat live events as invalidation hints ("notify, then fetch"): change events carry resource ids and versions, and clients refetch. Optimistic concurrency uses row versions exposed as ETags (`11-api-and-clients.md`). _Source: design/identity-api-deployment.md §3.6–3.7; design/runtime.md §3.6._

**ARC-047** (MUST) Node-local caches (settings snapshot, compiled plug-ins, definition caches, discovery LRUs) **MUST** be either content-addressed (keyed by hash, never stale) or bounded by a TTL, and **MAY** additionally be invalidated early by a doorbell. Correctness **MUST NOT** depend on the doorbell arriving. _Source: open-issues A2, A4._

**ARC-048** (MUST) Transactions **MUST NOT** perform network calls or long file operations while open. Handlers do external I/O first, then commit results in a short fenced transaction. _Source: design/runtime.md §3.4, §4.1 (derived: short transactions keep the job queue, SQLite writer and event log unblocked)._

### 8. Rolling upgrades and version skew

**ARC-049** (MUST) Schema migrations **MUST** follow expand/contract: a release **MAY** only add tables, columns (nullable or defaulted), indexes and constraints that N-1 code tolerates; renames, drops and tightening constraints **MUST** land in a later release, after the release that stopped using the old shape. _Source: open-issues A7; design/runtime.md §3.2._

**ARC-050** (MUST) Release N's binary **MUST** work correctly against the schema of release N+1 during a rollout (N-1 compatibility). The schema **MUST** record the minimum binary version it supports, and a binary older than that **MUST** refuse to start (`12-platform.md` §1). _Source: open-issues A7._

**ARC-051** (MUST) The public API **MUST** only change additively within `/api/v1` (new resources, new optional fields, new enum values that clients must tolerate). CI **MUST** gate this with `oasdiff breaking` against the last release's specification. _Source: open-issues A7; design/identity-api-deployment.md §3.6._

**ARC-052** (MUST) In a mixed-version cluster, a node **MUST** claim only job kinds it has registered, **MUST** accept job arguments written by the previous release, and new job-argument fields **MUST** be optional. Event envelope changes **MUST** be additive within `bonnyread.event/1`. _Source: open-issues A7 (applied to jobs and events)._

**ARC-053** (MUST) The `node` table **MUST** record each node's binary and schema versions, and a health check **MUST** warn while more than one binary version is live. _Source: design/runtime.md §3.2._

### 9. Capability-routed jobs

**ARC-054** (MUST) Each node **MUST** detect and advertise, in its `node` row, the native tools it can run with their versions (at least `ffprobe`, `fpcalc`, `alass`, `ffsubsync`, `ffmpeg`). Each job kind **MUST** declare the tools it requires. A node **MUST** claim only jobs whose requirements it satisfies. _Source: open-issues A6._

**ARC-055** (MUST) Desktop installers and container images **MUST** bundle `ffprobe` and `fpcalc`. `alass` **MUST** be bundled where its licence allows; `ffsubsync` is optional. _Source: open-issues A6._

**ARC-056** (MUST) When jobs of a kind have waited longer than 5 minutes and no live node advertises the tools that kind requires, a health issue **MUST** be raised naming the missing tool. _Source: open-issues A6 (derived failure path)._

### 10. Storage assumptions

**ARC-057** (MUST) Media roots, download directories and extraction staging **MUST** be mounted at identical paths on every node that runs file jobs. _Source: D14._

**ARC-058** (MUST) The extraction staging root **MUST** be on the same filesystem as each library root it feeds. Downloads **MUST** be on the library's filesystem when hardlink or reflink imports are wanted; otherwise imports fall back to copying, and that fallback **MUST** be visible (a health issue and a per-import note), never silent. A health check **MUST** verify these conditions. _Source: open-issues A5._

**ARC-059** (MUST) Each root folder **MUST** carry a marker file (`.bonnyread-root`, containing the root's UUID), and a node-scoped health check **MUST** verify on every node that each root is mounted, matches, and is writable. File jobs **MUST** re-check the marker before writing. _Source: design/runtime.md §3.15._

**ARC-060** (MUST) Stored file paths **MUST** be a root folder id plus a path relative to that root. _Source: open-issues A12._

**ARC-061** (MUST) Each node **MUST** have a data directory (`DATA_DIR`) and a node-local, disposable state directory (`STATE_DIR`). In HA on PostgreSQL, `DATA_DIR` **MUST NOT** need to be shared between nodes; everything durable is in the database. _Source: open-issues A2; design/runtime.md §3.15; design/identity-api-deployment.md §3.12._

**ARC-062** (MUST NOT) The SQLite database file **MUST NOT** be placed on a network filesystem; this **MUST** be documented and guarded by the data-directory lock. _Source: design/runtime.md §3.15._

### 11. Security baseline

The detailed rules live in `10-identity-and-access.md`, `11-api-and-clients.md`, `12-platform.md` and `13-deployment.md`. This baseline applies to every area.

**ARC-063** (MUST) Authentication **MUST** always be on. There **MUST NOT** be a "no authentication" mode or a "trust local addresses" mode. The first account **MUST NOT** be claimable by whoever reaches the port first. _Source: design/identity-api-deployment.md invariants I1, I6._

**ARC-064** (MUST) Every entry point (UI session, API key, device credential, SSE stream, CLI) **MUST** pass through one authoriser. _Source: design/identity-api-deployment.md invariant I7; D17._

**ARC-065** (MUST NOT) Credentials **MUST NOT** be accepted in query strings; only headers and cookies. _Source: design/identity-api-deployment.md invariant I2._

**ARC-066** (MUST) Secrets (service keys, passwords, tokens) **MUST** be stored in the database, write-only through the API, redacted from logs, errors, history and sanitised exports. Encryption at rest is the database's (or disk's) responsibility. Bootstrap secrets that unlock the database (DSN, TLS key, OIDC bootstrap) **MUST** come from the bootstrap config file or environment (`*_FILE` supported). _Source: D16 as amended by D20; open-issues A8._

**ARC-067** (MUST) Backup archives **MUST** support optional passphrase encryption, and a clearly labelled sanitised export **MUST** exist (`12-platform.md` §16). _Source: D20; open-issues B10._

**ARC-068** (MUST) Outbound TLS verification **MUST** always be on; per-connection trust is expressed by a pinned certificate or a custom CA, never by disabling verification. _Source: design/identity-api-deployment.md invariant I4._

**ARC-069** (MUST) The binary **MUST** contain no telemetry and no third-party secrets other than project-registered, attributed, user-overridable public API keys. _Source: D3b, D16; design/identity-api-deployment.md invariant I5._

**ARC-070** (MUST NOT) Unverified native code **MUST NOT** be loaded into the process. Extension code runs only in the WASM sandbox with capability grants (`12-platform.md` §20). _Source: D15._

**ARC-071** (MUST) Outbound requests on behalf of plug-ins or non-admin input **MUST** pass an SSRF guard that blocks loopback, link-local and private ranges unless explicitly granted, re-checked after DNS resolution (`12-platform.md` §17). _Source: design/runtime.md §3.12._

**ARC-072** (MUST) Custom scripts **MUST** be disableable deployment-wide by bootstrap configuration (`ALLOW_SCRIPTS=false`). _Source: design/runtime.md §3.7._

**ARC-073** (MUST) Container images **MUST** run as a non-root user with a read-only root filesystem and no root entrypoint that rewrites UIDs. _Source: design/identity-api-deployment.md §3.11._

**ARC-074** (MUST) Forward-auth (trusted-header) login **MUST** be opt-in and off by default. _Source: D20; open-issues B13._

**ARC-075** (SHOULD) TOTP **SHOULD** be mandatory for Owner and Admin accounts unless the server is loopback-only [OPN-022]; the rule itself is specified in `10-identity-and-access.md`. _Source: open-issues Group C (identity)._

**ARC-076** (MUST) Before the first public release, a `go-licenses` scan of the final module graph **MUST** be run, and the D5b legal review **MUST** cover the items in open-issues B12, including runtime-fetched content stored in user databases and backups. _Source: D5, D20; open-issues A2, B12._

## Data model

Architecture introduces no entities of its own. The platform tables it relies on (`node`, `job`, `schedule`, `lease`, `event_log`, `rate_limit_*`, `schema_meta`, …) are defined in `12-platform.md` §Data model; shared domain entities in `02-domain-model.md`.

## Flows

### F1. A node dies in a three-node cluster

1. Nodes A, B, C run on PostgreSQL. A holds lease `irc:3` (token 41) and is running RSS job 9001 (attempt 1, dedupe key `sched:<rss schedule id>`). In a fenced transaction A has committed a grab job 9002 for an episode.
2. A is killed (OOM).
3. Job 9002 is `available`; B or C claims it within the idle-poll interval (or at once via the doorbell) and hands the release to the download client, tagged with its grab id.
4. After the lease TTL (15 s) B takes `irc:3` with token 42 and reconnects. Announces missed during the gap are covered by the next RSS poll; an RSS catch-up job is enqueued for the indexers bound to that network.
5. After 45 s without a heartbeat, the reaper on any node makes job 9001 `available`. C claims attempt 2 and re-runs RSS. Releases already acted on are rejected as pending, and the dedupe key blocks a second grab job.
6. **Failure path — A was only paused:** when A resumes, its lease renewer has already self-demoted (no renewal for 10 s), so its IRC context is cancelled; any fenced write with token 41 or attempt 1 fails; A's result for job 9001 is discarded and logged.

### F2. Request on node A, work on node B, live update on node C

1. The browser holds its SSE stream to C. `POST /api/v1/items/{id}/search` lands on A.
2. A commits the job (priority `interactive`, dedupe key per item and slot) and an `event_log` row; after commit it rings `bonnyread_jobs` and `bonnyread_events`, and returns `202` with the job.
3. B claims the job (`SKIP LOCKED`), reports progress (throttled to 1/s), and finally commits decisions, a grab job and events in one fenced transaction.
4. C, woken by the doorbell (or by its poll), reads new `event_log` rows, filters by the principal's permissions and writes them to the stream. The browser refetches; A or B serves the read and sees B's commit.
5. **Failure path:** if C's LISTEN connection drops, C still tails `event_log` by polling; if the browser reconnects to A with a `Last-Event-ID` older than the retained window, A sends `reset` and the browser refetches everything visible.

### F3. Rolling upgrade from N to N+1

1. The operator (Helm hook job, or the first node to start with auto-migrate enabled) applies N+1's **expand** migrations under the migration lock. N nodes keep running, because the schema's minimum binary version is still ≤ N.
2. Pods are replaced one by one. Each draining N pod releases its leases and returns interrupted jobs to `available` without consuming an attempt.
3. During overlap, N nodes claim only kinds they know; N+1-only kinds wait for N+1 nodes. A health warning shows mixed versions.
4. Release N+2 may ship the **contract** migrations for shapes N+1 stopped using, and raises the schema's minimum binary version to N+1.
5. **Failure path:** a pod older than the schema's minimum binary version refuses to start and stays unready; the rollout halts without damage.

### F4. Desktop startup

1. The OS service starts the core with SQLite in `DATA_DIR`; it takes the data-directory lock, opens the database, takes a pre-migration backup if migrations are pending, migrates, recovers jobs left `running` by a crash, and starts on loopback.
2. The Tauri app connects to `127.0.0.1:7979`, verifies `serverId` from `/api/v1/meta`, and, if it has no credential, pairs over the local socket; the core checks the peer's OS identity and issues a device credential, which the app stores in the OS keychain.
3. **Failure path:** if the lock is held, the second process exits with "another instance is running"; if the service is stopped, the app offers to start it.

## Non-goals

| Non-goal | Reason |
|---|---|
| MySQL/MariaDB, SQL Server | Dropped (D20, B5): single-node only, which SQLite already covers, at the cost of a third dialect. |
| Leader election for schedules and polling | Replaced by schedule rows and `SKIP LOCKED` (A1); fewer failover paths. |
| Shared data volume (RWX) for definitions, PCD data or plug-ins | Stored in the database (A2). |
| Compatibility API façades (Arr v3/v1, Seerr, Newznab server) | D7. |
| Read replicas | Would break read-after-commit consistency for SSE-driven refetches. |
| Hosted project infrastructure (metadata proxy, push relay) | D3c; no mobile push in v1 (B13). |
| Self-updater | Deferred (D16). |
| Racing mode (library-less grabs) | Later phase (D16). |
| Loading native plug-ins | D15; WASM only. |
| Offline writes in clients | Clients disable mutations when the server is unreachable; nothing to reconcile. |

## Traceability

| Reference behaviour | Preserved | Deliberately changed (D19) |
|---|---|---|
| Arr/Bazarr/Prowlarr … separate processes linked by API keys, SignalR and webhooks | Every user-visible function | One binary; in-process calls; inter-app keys, sync layers and polling mirrors removed |
| NzbDrone commands survive restart (Sonarr `CommandQueueManager`) | Durable background work | One DB table is the only source of truth; works across nodes |
| Sonarr `RequiresDiskAccess` global mutex | No two file operations race on the same files | Per-entity concurrency keys, cluster-wide |
| Multi-instance 4K/1080p and ebook/audiobook setups | Separate profiles per version | Slots in one instance (D4, D20) |
| autobrr/Arr single-process singletons | IRC and polling keep running | Active-active HA with leases only for long-lived connections |
| Lidarr in-process .NET plug-ins | Extensibility | WASM sandbox with capability grants |
| Tray apps and Windows service installers in six forks, Jackett, Unpackerr | Desktop install that "just runs" | One core OS service plus one Tauri app; pairing over a local socket |
| Arr "disable auth for local addresses" | Low-friction local use | Auth always on; local pairing instead |
