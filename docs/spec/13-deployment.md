# 13. Deployment

This file specifies how bonnyread is installed, configured, stored, backed up and upgraded across the three supported topologies: a single-user desktop with the core as an OS service, a home server (container or package), and Kubernetes with several replicas. It covers `bonnyread service install`, the per-OS installers, the container image, the Helm chart, storage requirements, supported databases, bootstrap configuration, ports and listeners, backups and the upgrade process. Runtime internals (job queue, leases, events, settings, health-check framework) are in `12-platform.md`; pairing and first-owner setup are in `11-api-and-clients.md`. Applies to **P1**.

## Concepts

| Term | Definition |
|---|---|
| Core | The single `bonnyread` Go binary (API, workers, embedded web UI). |
| Node | One running core process. In Kubernetes, one pod. |
| Topology A / B / C | Desktop single-user; home server; Kubernetes HA (DEP-001). |
| Data directory (`<data>`, `DATA_DIR`) | Holds bootstrap config, the SQLite database, backups, TLS material, the local endpoint and caches. In HA it is node-local and disposable (A2). |
| State directory (`STATE_DIR`) | Node-local: lock file, compilation cache, temp and extraction scratch. Defaults to `<data>/state`. |
| Bootstrap config | Values needed before the database is open or that differ per node (A8). Everything else is a database setting (D16). |
| Admin listener | An optional second HTTP listener for metrics, probes and profiling. |
| Local endpoint | The Unix socket or named pipe of `11-api-and-clients.md` API-091. |
| Expand/contract | A migration discipline: release N only adds; release N+1 removes what N stopped using (A7). |

## Requirements

### Topologies

**DEP-001** (MUST) Three topologies are supported with the same binary and the same database schema:

| | A. Desktop | B. Home server | C. Kubernetes HA |
|---|---|---|---|
| Install | Tauri app installer installs the core as an OS service | Container image or headless package | Helm chart |
| Nodes | 1 | 1 | 1..N |
| Database | SQLite | SQLite or PostgreSQL | PostgreSQL (required for N > 1) |
| Default bind | `127.0.0.1:7979`, TLS off | `0.0.0.0:7979` (container), `127.0.0.1:7979` (package) | `0.0.0.0:7979` behind an Ingress |
| Clients | Tauri app via local endpoint and loopback; others after LAN access is enabled | Browser, Tauri, phones via a reverse proxy | Browser, Tauri, phones via the Ingress |

_Source: D18; design/identity-api-deployment.md §3.10._

**DEP-002** (MUST) Single-node is the default in every topology (D14). Nothing in topologies A and B requires PostgreSQL, a shared filesystem or an external service.

**DEP-003** (MUST) Desktop "Allow access from other devices" (Owner only, `system:manage`): requires the Owner to have TOTP (IAM-024); sets TLS mode `self-signed` (DEP-013); binds `0.0.0.0:7979`; on Windows adds an inbound firewall rule through an elevated helper triggered from the tray app; starts mDNS `_bonnyread._tcp` advertisement. Disabling it reverses each step.

**DEP-004** (SHOULD) The documentation provides reverse-proxy recipes for Caddy, Traefik and Nginx Proxy Manager: HTTP/2 to clients, response buffering off and read timeout ≥ 1 hour for `/api/v1/events`, `X-Forwarded-*` headers set, and `TRUSTED_PROXIES` configured to the proxy's address.

### Ports and listeners

**DEP-010** (MUST) The main listener defaults to port **7979** and serves the web UI, `/api/v1` and SSE. Base-path hosting is set by `BASE_URL` (DEP-061).

**DEP-011** (MUST) The admin listener (`ADMIN_ADDR`, default off; `:9979` in the Helm chart) serves `/metrics` (per-node process metrics), `/metrics/cluster` (database-derived gauges, identical from any node, A9), `/livez`, `/readyz`, and `/debug/pprof` only when `PPROF=true`. `/metrics` and `/metrics/cluster` require a credential with `metrics:read`; `/debug/pprof` requires `system:manage`. The Ingress never routes to the admin listener. [OPN-137]

**DEP-012** (MUST) Probes are unauthenticated and return no detail beyond status: `/livez` is 200 while the process is responsive; `/readyz` is 200 only when the database is reachable, the schema is compatible (DEP-091), the Postgres LISTEN connection is subscribed (Postgres only) and the node is not draining. Both are served on the main listener and on the admin listener. Domain health (indexers, clients, storage) **MUST NOT** affect readiness. `bonnyread healthcheck` calls `/livez` for container health checks. [OPN-138]

**DEP-013** (MUST) TLS mode (`TLS_MODE`) is `off`, `self-signed` or `files`. `self-signed` generates an ECDSA P-256 key and certificate in `<data>/tls/` (mode `0600`) on first use and exposes the SPKI SHA-256 fingerprint for pinning (`11-api-and-clients.md` API-082, API-099). `files` reads `TLS_CERT_FILE` and `TLS_KEY_FILE` and reloads them when they change.

### Databases

**DEP-015** (MUST) Supported databases are **PostgreSQL ≥ 15** and **SQLite** (the pure-Go `modernc.org/sqlite` driver, A18). MySQL/MariaDB and SQL Server are not supported (D20). On PostgreSQL the core checks `server_version` at startup and refuses to start below 15. [OPN-071]

**DEP-016** (MUST) More than one node requires PostgreSQL (D14). On SQLite the core takes an exclusive `flock` on `<data>/bonnyread.lock` and refuses to start with "another instance is running" if it is held.

**DEP-017** (MUST) The SQLite database **MUST** be on a local filesystem. The documentation states that NFS/SMB are unsupported, and the core **SHOULD** raise a health issue when the OS reports a network filesystem type for `<data>`.

**DEP-018** (MUST) PgBouncer is supported only in **session** pooling mode. Transaction and statement pooling are unsupported. The LISTEN connection uses `DB_LISTEN_DSN` (default: `DB_DSN`), which **MUST** be a direct or session-pooled connection (A4).

**DEP-019** (MUST) The core does not use read replicas in v1; every node reads and writes the primary.

**DEP-020** (MUST) `bonnyread db copy --from sqlite:<path> --to postgres:<dsn>` is a supported, tested path from topology A or B to C: it streams all entities in dependency order into an empty, migrated PostgreSQL database and verifies row counts. [OPN-072]

**DEP-021** (MUST) `DB_DSN` accepts `sqlite://<path>` (default `sqlite://$DATA_DIR/bonnyread.db`) and `postgres://…`/`postgresql://…` URLs.

### OS service installation

**DEP-030** (MUST) The desktop bundle ships the core binary as a resource that is **installed as an OS service**, not a Tauri sidecar started by the app: the core keeps running when the app is closed, and for system services when the user logs out. The desktop app and its local core are installed and upgraded together and are always the same version. Uninstalling removes the service and keeps the data directory unless the user chooses to delete it.

**DEP-031** (MUST) The installer grants the installing user: start/stop rights on the service; local pairing (`--grant-pairing`, `11-api-and-clients.md` API-093); and, on Linux and macOS daemon installs, membership of the `bonnyread` group.

**DEP-032** (MUST) One implementation installs services on every OS: `bonnyread service install|uninstall|start|stop|status [--data DIR] [--user ACCOUNT] [--grant-pairing USER]`, built on `kardianos/service` (Windows SCM, systemd, launchd). Installers call it; no installer writes service definitions itself.

**DEP-033** (MUST) Windows: an NSIS per-machine installer (Tauri 2 bundler) whose `NSIS_HOOK_POSTINSTALL` runs `bonnyread.exe service install` and `NSIS_HOOK_PREUNINSTALL` runs `service uninstall`. An MSI via WiX fragments **MAY** be offered. The service runs under the virtual account `NT SERVICE\bonnyread`, never LocalSystem. Data: `%ProgramData%\bonnyread`. No firewall rule is added unless LAN access is enabled (DEP-003).

**DEP-034** (MUST) macOS: by default the app registers a **per-user LaunchAgent** via `SMAppService.agent` on first launch (macOS 13+, no admin rights; runs while the user is logged in), with data in `~/Library/Application Support/bonnyread`. For always-on "server mode", a separate signed `.pkg` (built with `pkgbuild`) installs a **system LaunchDaemon** running as the dedicated user `_bonnyread`, with data in `/Library/Application Support/bonnyread`. The app and `.pkg` **MUST** be Developer ID-signed and notarised. [OPN-023]

**DEP-035** (MUST) Linux `.deb` and `.rpm` (Tauri bundler for the desktop app; nfpm for headless packages): the post-install script creates the `bonnyread` system user and group, adds the installing user (`$SUDO_USER` or `$PKEXEC_UID`) to the group, and runs `service install`; the pre-remove script runs `service uninstall`. Data: `/var/lib/bonnyread`.

**DEP-036** (MUST) Linux AppImage cannot install a system service; it offers a **systemd user unit** (`systemctl --user`), suggests `loginctl enable-linger`, uses `$XDG_DATA_HOME/bonnyread` as data and `$XDG_RUNTIME_DIR/bonnyread/bonnyread.sock` as the local endpoint.

**DEP-037** (MUST) The systemd system unit sets `NoNewPrivileges=yes`, `ProtectSystem=full`, `ProtectHome=read-only`, `PrivateTmp=yes`, `Restart=on-failure`. Media paths under `/home` need a drop-in `ReadWritePaths=`; the root-folder settings UI explains this when a root under `/home` is not writable.

**DEP-038** (MUST) Headless builds (goreleaser + nfpm): `.deb`, `.rpm`, `.apk`, tarballs, a Windows zip and a Homebrew formula, all using `service install`. The first Owner is created with the setup URL or bootstrap owner (`11-api-and-clients.md` API-100, API-101).

### Container image

**DEP-040** (MUST) The image is built in stages: SvelteKit build (Node) → Go build with `CGO_ENABLED=0` embedding the SPA → final image.

**DEP-041** (MUST) The final base is `gcr.io/distroless/static-debian12:nonroot` or an equivalent static base. It contains the core binary, CA certificates, and statically linked **ffprobe** and **fpcalc** (A6). **alass** is included where its licence allows (A6); ffsubsync is not included. There is no shell and no package manager. Archive extraction (RAR, 7-Zip, zip) is pure Go, so no extraction binaries are added.

**DEP-042** (MUST) The image runs as UID/GID 65532 and works under any arbitrary UID (OpenShift). It has **no** PUID/PGID/UMASK root entrypoint and never starts as root. Users set `user:` (Compose) or `runAsUser`/`fsGroup` (Kubernetes). The umask for created media files is an application setting.

**DEP-043** (MUST) The root filesystem is read-only; `/data` and `/tmp` are writable. The image declares `EXPOSE 7979 9979`, `HEALTHCHECK CMD ["/bonnyread","healthcheck"]`, `DATA_DIR=/data`, `HTTP_ADDR=0.0.0.0:7979` and `SETUP_URL_LOG=true`.

**DEP-044** (MUST) Images are multi-arch `linux/amd64` and `linux/arm64`, and **SHOULD** include `linux/arm/v7` when the pure-Go stack and bundled tools build for it.

**DEP-045** (MUST) Images are signed with cosign and carry an SBOM, SLSA provenance and a `go-licenses` report (D5).

**DEP-046** (MUST) Nodes advertise which native tools they have (ffprobe, fpcalc, alass, ffsubsync) so capability-routed jobs run only where the tool exists (A6; `12-platform.md`).

### Helm chart

**DEP-050** (MUST) The chart `charts/bonnyread` contains: `deployment.yaml` (replicas, probes, securityContext, volumes, topologySpreadConstraints), `service.yaml` (http 7979, admin 9979), `ingress.yaml`, `migrate-job.yaml`, `servicemonitor.yaml` and `podmonitor.yaml` (optional), `pdb.yaml` (when `replicaCount > 1`), `networkpolicy.yaml` (optional), `secret.yaml` (only when no `existingSecret` is given) and `NOTES.txt` (how to obtain the setup URL or use the bootstrap owner).

**DEP-051** (MUST) `replicaCount` defaults to 1. Rendering **MUST** fail if `replicaCount > 1` and `database.driver` is not `postgres`.

**DEP-052** (MUST) `database`: `driver`, `existingSecret` with key `dsn` and optional `listenDsn`; an optional flag creates a CloudNativePG `Cluster`.

**DEP-053** (MUST) Migrations run in a `pre-install,pre-upgrade` hook Job executing `bonnyread migrate up`. Pods run with `MIGRATE=off` (DEP-061).

**DEP-054** (MUST) `persistence.data`: an RWO PVC for single-replica SQLite installs. With PostgreSQL the data directory is an `emptyDir`: fetched definitions, PCD data and plug-ins are stored in the database (A2), so no shared data volume is needed.

**DEP-055** (MUST) `media.volumes[]`: `{name, existingClaim, mountPath, subPath?}`, ReadWriteMany, mounted at the identical `mountPath` on every pod (D14). Storage layout follows DEP-070–DEP-073.

**DEP-056** (MUST) Probes: `startupProbe` on `/livez` (tolerating a slow first boot), `livenessProbe` on `/livez`, `readinessProbe` on `/readyz`. `terminationGracePeriodSeconds: 30`, a `preStop` sleep of 5 s and `READINESS_DRAIN=5s`, so the load balancer stops routing before SSE streams close; the core's shutdown grace fits inside 30 s (`12-platform.md`).

**DEP-057** (MUST) Long-lived connections use database leases and periodic work uses schedule rows (A1; `12-platform.md` §5–§6), so the chart creates **no** Kubernetes RBAC: the ServiceAccount has no permissions and `automountServiceAccountToken: false`.

**DEP-058** (MUST) Sessions, idempotency keys, rate-limit counters and SSE replay are in the database; the chart **MUST NOT** configure sticky sessions.

**DEP-059** (MUST) Ingress defaults are SSE-friendly: for ingress-nginx `nginx.ingress.kubernetes.io/proxy-buffering: "off"` and `proxy-read-timeout: "3600"`; Traefik equivalents are documented.

**DEP-060** (MUST) Other values:
- `metrics.podMonitor` scrapes `/metrics` on every pod; `metrics.serviceMonitor` scrapes `/metrics/cluster` once through the Service (A9). Both use `bearerTokenSecret`: an API key of a *Metrics scraper* service account, supplied or created by a post-install hook running `bonnyread admin create-service-account --preset metrics`.
- `auth.oidc.providers[]` (issuer, clientId, `clientSecretRef`, groups claim, group→role rules) are seeded into the database at install and are then edited in the UI; `auth.bootstrapOwner` (OIDC group, or username plus a password secret).
- Pod security: `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, drop ALL capabilities, `seccompProfile: RuntimeDefault`.
- `trustedProxies`: the ingress controller's pod CIDR, so client IPs in audit and throttling are real.

### Bootstrap configuration

**DEP-061** (MUST) Bootstrap configuration comes from, in precedence order: command-line flags, environment variables `BONNYREAD_<KEY>`, `BONNYREAD_<KEY>_FILE` (the value is read from that file with trailing newlines trimmed; setting both forms is an error), the config file (`<data>/config.toml`, or `--config <path>`), then defaults. Keys:

| Key | Default | Notes |
|---|---|---|
| `DATA_DIR` | per OS (DEP-075) | |
| `STATE_DIR` | `<data>/state` | node-local |
| `DB_DSN` | `sqlite://$DATA_DIR/bonnyread.db` | secret |
| `DB_LISTEN_DSN` | `DB_DSN` | Postgres LISTEN; secret |
| `DB_MAX_CONNS` | 20 (PostgreSQL) | pool size (`12-platform.md` PLT-016) |
| `HTTP_ADDR` | `127.0.0.1:7979` | container image sets `0.0.0.0:7979` |
| `ADMIN_ADDR` | empty (off) | Helm: `:9979` |
| `BASE_URL` | empty | external URL and sub-path |
| `TLS_MODE`, `TLS_CERT_FILE`, `TLS_KEY_FILE` | `off` | key is secret |
| `TRUSTED_PROXIES` | empty | CIDRs allowed to set `X-Forwarded-*` and forward-auth headers |
| `LOG_LEVEL`, `LOG_FORMAT`, `LOG_FILE` | `info`, `json` in containers / `text` on a TTY, none | |
| `NODE_ID`, `ROLES` | derived, `api,worker` | `12-platform.md` |
| `MIGRATE` | `auto` | `auto` migrates at startup; `off` never migrates (Helm) |
| `READINESS_DRAIN` | `0s` | Helm: `5s` |
| `PPROF` | `false` | |
| `ARGON2_MEMORY_KIB`, `ARGON2_TIME`, `ARGON2_PARALLELISM` | 65536, 3, 2 | floor 19456, 2 (IAM-020) |
| `LOCAL_ENDPOINT` | `<data>/run/bonnyread.sock` or `\\.\pipe\bonnyread` | API-091 |
| `LOCAL_PAIRING_PRINCIPALS` | empty | set by `--grant-pairing` |
| `SETUP_URL_LOG` | `false` (`true` in the image and headless packages) | API-100 |
| `BOOTSTRAP_OWNER_OIDC_GROUP`, `BOOTSTRAP_OWNER_USERNAME`, `BOOTSTRAP_OWNER_PASSWORD` | empty | password via `_FILE`; secret |
| `OIDC_SEED` | empty | providers seeded into the database at first start; secrets via `_FILE` |
| `PLUGINS_ALLOW`, `ALLOW_SCRIPTS`, `OTEL_*` | | `12-platform.md` |

[OPN-135] _Source: A8; D20 (D16 amended); runtime.md §3.10._

**DEP-062** (MUST) Only the keys above live in bootstrap configuration; every other setting is stored in the database (D16). Settings pinned by environment (`BONNYREAD_SET__…`) are specified in `12-platform.md`.

**DEP-063** (MUST) On first start without a config file, the desktop and package installs write one with the defaults and mode `0600`. The core logs a warning if the config file or any `_FILE` secret is readable by other OS users.

**DEP-064** (MUST) Seeded values (`OIDC_SEED`, bootstrap owner, Helm `auth.*`) are applied only when the corresponding database rows do not exist; later edits in the UI win.

### Storage

**DEP-070** (MUST) Media roots and download paths are mounted at identical paths on every node (D14). Each root folder has a `.bonnyread-root` marker containing its UUID; a node-scoped health check verifies it on every node, and imports re-check it before writing.

**DEP-071** (MUST) The extraction staging root **MUST** be on the same filesystem as each library root it serves (default `<root>/.bonnyread-staging`, configurable per root), so staged files move into the library by rename. _Source: A5; design/downloads.md; design/import.md._

**DEP-072** (MUST) When hardlink or reflink imports are wanted (seeding torrents), the download paths **MUST** be on the same filesystem as the library roots. Otherwise imports fall back to copying; the fallback is shown in the import plan preview and raised as a health issue, never applied silently. In Kubernetes this means one RWX volume holding media, downloads and staging (as sub-paths); separate PVCs are separate filesystems.

**DEP-073** (MUST) Health checks cover, per node: for each (download client, root folder) pair, a test hardlink and reflink at start-up and daily, reporting which transfer methods are available; each root's marker, mount and writability; free space in `STATE_DIR` temp and each staging root; SQLite on a network filesystem (DEP-017).

**DEP-074** (MUST) `STATE_DIR` is node-local and holds the lock file, the plug-in compilation cache, temp files and extraction scratch space not placed on a staging root.

**DEP-075** (MUST) Data directory defaults and layout (single node; in HA only `cache/` and `state/` are used, and they are disposable):

| OS / mode | `DATA_DIR` |
|---|---|
| Linux system service | `/var/lib/bonnyread` |
| Linux user unit (AppImage) | `$XDG_DATA_HOME/bonnyread` |
| macOS agent / daemon | `~/Library/Application Support/bonnyread` / `/Library/Application Support/bonnyread` |
| Windows | `%ProgramData%\bonnyread` |
| Container | `/data` |

```
<data>/
  config.toml                      0600
  bonnyread.db (+ -wal, -shm)      SQLite only
  bonnyread.lock                   SQLite only
  run/bonnyread.sock               local endpoint (Unix), 0660 service account:bonnyread
  tls/                             self-signed cert and key, 0600
  cache/                           images and metadata cache (disposable)
  backups/                         SQLite backups, including pre-migration snapshots
  restore-pending/                 staged restore (DEP-083)
  logs/                            optional rotating log file (desktop)
  state/                           STATE_DIR default
```

Fetched indexer definitions, PCD data and plug-ins are in the database, not in the data directory (A2).

### Backups

**DEP-080** (MUST) On SQLite the core makes backups with `VACUUM INTO`, packaged as `bonnyread-<version>-<timestamp>.tar.zst` with `manifest.json` (app version, schema version, dialect, created at, node). Kinds: scheduled (weekly by default), manual, and pre-migration. A configurable retention count applies; the directory defaults to `<data>/backups`.

**DEP-081** (MUST) Backup archives **MAY** be encrypted with a passphrase (age, scrypt recipient). The UI states that unencrypted archives contain every stored secret in plaintext. _Source: B10; D20 (D16 amended)._

**DEP-082** (MUST) A **sanitised export** strips all secrets and is labelled as not restorable without re-entering them. It is meant for sharing configurations and bug reports.

**DEP-083** (MUST) Restore is staged: select or upload an archive; validate the manifest (dialect, and schema version ≤ the binary's); extract to `<data>/restore-pending/`; run `PRAGMA integrity_check`; restart. At the next startup the core swaps files before opening the database, keeps the replaced database as `pre-restore-<timestamp>.db`, then applies newer migrations normally.

**DEP-084** (MUST) On PostgreSQL the core does not take physical backups; the operator's tooling (PITR, `pg_dump`, CloudNativePG backups) is responsible, and the documentation gives recipes. A health issue is raised until an admin sets the "external backups configured" acknowledgement.

**DEP-085** (MUST) `bonnyread db export` / `db import` (also available as a job and API action) stream all entities in dependency order as NDJSON inside the same archive format, for any dialect. `db copy` (DEP-020) uses the same code.

### Upgrades

**DEP-090** (MUST) There is no self-updater (D16). Upgrades replace the binary through the package manager, the installer, a new image tag or a Helm upgrade. The desktop installer upgrades app and core together (DEP-030).

**DEP-091** (MUST) Schema changes are expand/contract with N-1 compatibility (A7): release N may only add (tables, nullable or defaulted columns, indexes); removals and tightening happen in a later release once no supported binary uses the old shape. Each applied migration records `min_binary_version`. A binary **MUST** refuse to start (and report not ready) if the database's `min_binary_version` is above its own version; it **MUST** run against the schema of its own release and of the next release.

**DEP-092** (MUST) Migrations take a migration lock (`pg_advisory_lock` on PostgreSQL; the SQLite `flock` suffices). With `MIGRATE=auto`, a node applies pending migrations at startup. With `MIGRATE=off` and pending migrations required by the binary, the node is not ready and logs the reason.

**DEP-093** (MUST) On SQLite, before applying migrations the core runs `PRAGMA quick_check` and writes a pre-migration backup (`backups/pre-<version>-<timestamp>.db` via `VACUUM INTO`). On PostgreSQL it logs that the operator's backup is the safety net.

**DEP-094** (MUST) HA rolling upgrades move one release at a time: the Helm hook migrates the schema to N+1 while N pods run, then pods roll to N+1. Skipping releases requires stopping all pods first. Mixed node versions raise a health warning. Single-node installs **MAY** skip releases; migrations apply in order.

**DEP-095** (MUST) Downgrade is not supported other than by restoring a pre-migration backup (SQLite) or the operator's backup (PostgreSQL).

## Flows

### F1. Desktop install (Windows)

1. `bonnyread-setup.exe` (NSIS, per-machine, UAC). `NSIS_HOOK_POSTINSTALL` runs `service install --data "%ProgramData%\bonnyread" --grant-pairing <SID>` and starts the service.
2. The core writes `config.toml` with defaults (loopback, SQLite, TLS off), takes the lock, migrates, opens the named pipe.
3. The app pairs over the pipe and creates the Owner (`11-api-and-clients.md` F1).
4. Failure: port 7979 in use → the service logs the error and exits non-zero; the app reports "service failed to start" with the log tail and a link to change `HTTP_ADDR`.

### F2. Kubernetes HA install and upgrade

1. `helm install` with `replicaCount: 3`, `database.driver: postgres`, `database.existingSecret`, a media volume and `auth.bootstrapOwner.oidcGroup: media-admins`.
2. The pre-install Job runs `bonnyread migrate up` under the advisory lock.
3. Pods start with `MIGRATE=off`; `/readyz` turns true after the database check and LISTEN subscription. The post-install hook creates the Metrics scraper key for the PodMonitor and ServiceMonitor.
4. The first member of `media-admins` to sign in with OIDC becomes Owner.
5. Upgrade N → N+1: the pre-upgrade Job applies expand migrations; N pods keep working (DEP-091); pods roll one at a time with `preStop` 5 s, readiness drain and SSE `retry`.
6. Failure: the migration Job fails → Helm aborts the upgrade; N pods keep running on the unchanged schema.

### F3. Moving from SQLite to PostgreSQL

1. Stop the single node. Provision PostgreSQL ≥ 15 and run `bonnyread migrate up` against it.
2. `bonnyread db copy --from sqlite:/data/bonnyread.db --to postgres:<dsn>`; it verifies row counts.
3. Set `DB_DSN` to the PostgreSQL DSN and start. Scale up only after media, downloads and staging are on shared storage per DEP-070–DEP-072.
4. Failure: copy target not empty → abort without writing.

## Non-goals

| Excluded | Reason |
|---|---|
| Self-updater | Deferred (D16) |
| MySQL/MariaDB, SQL Server | D20, D13 |
| PgBouncer transaction/statement pooling | Breaks LISTEN and session semantics (A4) |
| PUID/PGID root entrypoints (Profilarr, Kapowarr, Cleanuparr) | The image never starts as root |
| Per-app tray programs, Inno Setup, firewall COM, `RegisterUrl` | One Tauri app and one `service` subcommand |
| Tauri sidecar core | The server must outlive the app (DEP-030) |
| Kubernetes `Lease` objects and RBAC | Leases are in the database (A1) |
| Sticky sessions | All session state is in the database |
| Shared RWX data volume in HA | A2 |
| App-managed physical PostgreSQL backups | Operator responsibility (DEP-084) |
| SQLite on network filesystems | Locking is unreliable (DEP-017) |
| Declarative provisioning of entities (GitOps) | Later; settings pinning only [OPN-067] |

## Traceability

| Reference | Preserved | Changed (D19) |
|---|---|---|
| Sonarr/Radarr/Lidarr forks, Prowlarr, Jackett | Windows service, tray, Linux packages, Docker, sub-path hosting (O11) | One `service` subcommand and one Tauri app; NSIS via Tauri; no self-updater; no telemetry |
| Unpackerr | Metrics endpoint | Admin listener with a `metrics:read` credential; per-node and cluster metrics split |
| autobrr | Separate metrics server | Same authoriser instead of htpasswd |
| Profilarr | Staged restore; sanitised backups | Optional encrypted archives; logical export and `db copy` |
| Profilarr, Kapowarr, Cleanuparr | — | PUID/PGID root entrypoint replaced by a non-root image |
| Sonarr/Radarr | Hardlink-on-import for seeding; copy fallback | Co-location is a stated requirement with health checks; reflink tried first |
