# 11. API and Clients

This file specifies the public REST API (`/api/v1`) and its conventions, the OpenAPI contract and code generation, the live event stream (SSE), and the clients that use them: the embedded SvelteKit web UI, the Tauri desktop app and, later, Tauri mobile apps. It also specifies server discovery and switching, version negotiation, first-owner setup, desktop pairing over a local socket or named pipe, pairing further devices by code or QR, and offline behaviour. Authentication methods, credentials and permissions are in `10-identity-and-access.md`; the job queue and event bus internals are in `12-platform.md`; installation is in `13-deployment.md`. Applies to **P1**, except Tauri mobile (**later**, after P1; not scheduled to a phase by D2).

## Concepts

| Term | Definition |
|---|---|
| Operation | One OpenAPI operation (method + path), with an `operationId` and an `x-permission`. |
| Action sub-resource | A `POST` on a resource path naming a verb (`/items/{id}/search`), replacing the Arr `POST /command {name}` bus. |
| Job | The durable queue row (`12-platform.md`) exposed read-only through `/api/v1/jobs`, plus cancel. Long-running actions return one. |
| Event | One SSE message. Change events are invalidation hints (ids and versions), not resource bodies. |
| Replay window | How far back a reconnecting client can resume by `Last-Event-ID`. |
| Server profile | A client-side record of one server: `{serverId, name, urls[], certFingerprint?, credentialRef, lastSeen}`. |
| Local endpoint | The core's local IPC endpoint: a Unix domain socket (Linux, macOS, container) or a named pipe (Windows). Used for local pairing, first-owner setup, discovery and CLI break-glass. |
| Peer credentials | The OS identity of the process at the other end of the local endpoint, as reported by the kernel (`SO_PEERCRED`, `getpeereid`/`LOCAL_PEERCRED`, named-pipe client token). |
| Pairing secret | A 256-bit, single-use, 10-minute secret with a purpose: `bootstrap` (first Owner via setup URL) or `remote` (a device credential for the creating user). |
| Native transport | The Rust-side HTTP client in Tauri apps, exposed to the UI as commands. |

## Requirements

### Base and versioning

**API-001** (MUST) All API operations live under `/api/v1`, relative to the configured base path (`13-deployment.md` DEP-061 `BASE_URL`).

**API-002** (MUST) Within v1 only additive changes are allowed: new operations, new optional request fields, new response fields, new enum values. Clients **MUST** tolerate unknown fields and unknown enum values. _Source: A7._

**API-003** (MUST) A breaking change requires `/api/v2`, served alongside v1 for at least one minor release; v1 responses then carry `Deprecation` (RFC 9745) and `Sunset` (RFC 8594) headers.

**API-004** (MUST) CI runs `oasdiff breaking` of the bundled spec against the last released spec and fails on any breaking change within v1. _Source: A7._

**API-005** (MUST) `GET /api/v1/meta` returns `{serverId (UUID, stable for the database), name, version, apiVersion (semver of the spec), minClientApiVersion, signInMethods[], setupComplete}`. `signInMethods` lists `password` (unless local login is disabled), each OIDC provider `{slug, displayName}`, each media server with sign-in enabled, and `forwardAuth` when enabled.

**API-006** (MUST) Unauthenticated access is allowed only to: `GET /api/v1/meta`; `/livez` and `/readyz`; the sign-in endpoints (`/api/v1/auth/login*`, `/api/v1/auth/oidc/{slug}/start|callback`, `/api/v1/auth/media-server/{serverId}`, `/api/v1/auth/native/exchange`, `/api/v1/auth/password-reset*`); `POST /api/v1/pairing/claim`; `POST /api/v1/setup/claim` and `/complete`; and the static web UI assets. The bundled spec is published at `GET /api/v1/openapi.json` (any authenticated principal) and with each release.

### Resources and representation

**API-010** (MUST) Resource paths are plural kebab-case nouns (`/items`, `/items/{itemId}/slots`, `/requests`, `/download-clients`). IDs are opaque strings to clients; UUIDv7 where the owning table uses UUID keys, while `02-domain-model.md` DOM-001 gives domain tables int64 keys [OPN-206].

**API-011** (MUST) Bodies are JSON with `camelCase` fields; timestamps are RFC 3339 in UTC; durations are integer seconds; sizes are integer bytes; enums are lowercase strings.

**API-012** (MUST) Actions are sub-resources: `POST /items/{id}/search`, `POST /requests/{id}/approve`, `POST /indexers/{id}/test`. There is no generic command endpoint.

**API-013** (MUST) Partial updates use `PATCH` with JSON Merge Patch (RFC 7396, `application/merge-patch+json`). `PUT` is used only for whole-document settings resources.

**API-014** (SHOULD) Related resources are expandable with `?expand=a,b`, whitelisted per operation in the spec. Unknown expansions return 400 `validation_failed`.

**API-015** (MUST) Secret fields are `writeOnly` in the schema. Responses carry a companion boolean (`apiKeySet: true`) and never the value, a hash, or any fragment of it. Sending the field replaces the secret; omitting it keeps it; sending `null` clears it where the secret is optional.

### Errors

**API-020** (MUST) Errors use `application/problem+json` (RFC 9457) with `type` = `https://bonnyread.dev/problems/<code>`, `title`, `status`, `detail`, `instance` (the request id), plus `code` (stable machine string) and, for validation, `errors[] {pointer (JSON Pointer), code, message}`. Every operation's error responses reference this schema.

**API-021** (MUST) 401 means no valid credential; 403 means authenticated but not permitted; a scoped resource outside the caller's scope returns **404** so scope does not reveal existence (`10-identity-and-access.md` IAM-064).

**API-022** (MUST) Error codes are an enum in the spec, extended additively. The base set:

| Code | Status | Meaning |
|---|---|---|
| `unauthenticated` | 401 | No credential |
| `invalid_credentials` | 401 | Sign-in failed |
| `credential_revoked`, `credential_expired` | 401 | Credential no longer valid |
| `forbidden` | 403 | Permission missing |
| `csrf_rejected` | 403 | Same-origin check failed |
| `quota_exceeded`, `slot_not_requestable` | 403 | Request rules (IAM-069, IAM-083) |
| `not_found` | 404 | Absent or out of scope |
| `conflict`, `last_owner` | 409 | State conflict |
| `title_exists`, `slot_in_use`, `root_folder_in_use` | 409 | Library conflicts (`02-domain-model.md` DOM-050, DOM-042, DOM-099) |
| `drafts_pending` | 409 | Publish blocked by pending drafts (`07-profiles.md` §5.3) |
| `root_kind_mismatch`, `root_slot_mismatch`, `naming_collision`, `quality_family_mismatch` | 422 | Slot configuration invalid (`02-domain-model.md` DOM-045, DOM-046, DOM-040) |
| `no_role`, `not_provisioned`, `no_server_access`, `local_peer_not_allowed` | 403 | Sign-in or local peer refused (`10-identity-and-access.md` IAM-035, F2, F3; API-093) |
| `idempotency_in_progress` | 409 | Same key still executing |
| `precondition_failed` | 412 | `If-Match` mismatch |
| `validation_failed`, `idempotency_key_reused` | 422 | Body invalid; key reused with a different body |
| `precondition_required` | 428 | `If-Match` missing where required |
| `rate_limited`, `login_throttled` | 429 | With `Retry-After` |
| `query_credential_rejected`, `cursor_mismatch`, `ambiguous_credentials` | 400 | Request malformed |
| `provider_unavailable` | 503 | OIDC or media server unreachable |

**API-023** (MUST) Responses **MUST NOT** include stack traces or internal error text. The request id is returned in `X-Request-Id` on every response and logged.

### Lists, pagination and filtering

**API-024** (MUST) Lists use cursor pagination: `?limit=<n>&cursor=<opaque>`, default `limit` 50, maximum 500. Responses are `{items: [...], nextCursor: string | null}`. Offset/page paging is not offered.

**API-025** (MUST) A cursor encodes the sort key and id and is bound to the filter and sort set; a cursor used with different filters returns 400 `cursor_mismatch`. `?include=totalCount` adds `totalCount` and is opt-in.

**API-026** (MUST) Filters are named query parameters defined per operation (`?kind=tv&status=missing`); repeating a parameter means OR; range filters use `<field>Before`/`<field>After`. Sorting is `sort=-createdAt,title` over fields whitelisted per operation. There is no generic filter language in v1.

**API-027** (MUST) Lists apply the caller's scope filter (IAM-060) inside the query, so pages and cursors are correct for the caller.

### Authorisation metadata

**API-030** (MUST) Every operation declares `x-permission: <resource>:<action>[:<facet>]` (vocabulary in `10-identity-and-access.md` IAM-061/062). Operations on scoped resources also declare `x-scope`, which says how to resolve the target's `{kind, libraryGroup, ownerId}` (for example `x-scope: {kind: item.kind, library: item.libraryGroup}`). Public operations (API-006) declare `x-permission: public`.

**API-031** (MUST) The server route table and authorisation middleware are generated from these extensions; no handler checks permissions by hand. A CI lint fails the build if any operation lacks `x-permission`, names an unknown resource, action or facet, or declares a scoped resource without `x-scope`.

**API-032** (MUST) The API-key and device-cap permission pickers in the UI are generated from the same vocabulary.

### Concurrency (ETags)

**API-035** (MUST) Every single-resource `GET` returns `ETag: "<row version>"`. `If-None-Match` returns 304 when unchanged.

**API-036** (MUST) `PATCH`, `PUT` and `DELETE` accept `If-Match`. For settings resources (every `PUT` document and every settings namespace), quality profiles, custom formats, naming presets and other profile resources, `If-Match` is **required**: missing → 428 `precondition_required`; mismatch → 412 `precondition_failed` with the current ETag. _Source: id §3.6._

### Idempotency

**API-040** (MUST) Every `POST` accepts an `Idempotency-Key` header (1–255 printable ASCII characters). Operations marked `x-idempotent-required: true` (request creation, grabs, and every action that starts a job) **MUST** reject API-key callers that omit it with 400 `validation_failed`. The web UI and native apps send a key per user submission.

**API-041** (MUST) Keys are stored per `(user, key)` for 24 hours with a hash of the method, path and body and the stored response. Same key and same hash → the stored status and body are replayed with `Idempotent-Replayed: true`; same key and different hash → 422 `idempotency_key_reused`; same key still executing → 409 `idempotency_in_progress`. The store is a database table, so it works across HA nodes.

### Long-running operations

**API-045** (MUST) An action that may take more than about one second returns `202 Accepted`, `Location: /api/v1/jobs/{jobId}` and the job representation. Actions that finish synchronously return 200 with their result.

**API-046** (MUST) A job is represented as `{id, type, state: queued|running|succeeded|failed|cancelled, progress: {current, total, message}, result, error (problem), createdBy, target: {type, id}, createdAt, startedAt, finishedAt}`. Progress messages are structured (`message` is a code plus args, localised by the UI).

**API-047** (MUST) `GET /api/v1/jobs` supports filters `state`, `type`, `targetId`, `createdBy`; `GET /jobs/{id}`; `POST /jobs/{id}/cancel`. Jobs are otherwise read-only. A job is visible to principals holding the permission of the action that created it (in the target's scope), or `system:read:jobs`. Completed jobs remain visible for 7 days. [OPN-069]

### Rate limiting

**API-049** (MUST) Each credential has a token bucket (default 20 requests/s, burst 100; setting `api.rate_limit`). Exceeding it returns 429 `rate_limited` with `Retry-After`. This protects the server from runaway scripts; it is not a quota. The SSE stream counts once per connection.

### OpenAPI contract and code generation

**API-050** (MUST) An OpenAPI 3.1 document is the source of truth, split under `api/openapi/` and bundled to `api/openapi.yaml`. Handlers, the TS client and the Go client are generated from it; the spec is never generated from code. _Source: D6, D12._

**API-051** (MUST) The spec is linted in CI (Redocly CLI or vacuum) with a ruleset enforcing API-010–API-036: naming, problem responses on every operation, `x-permission`, pagination shape, `writeOnly` secrets.

**API-052** (MUST) Server code is generated with **oapi-codegen v2** (`std-http-server` + `strict-server` on `net/http` `ServeMux`), with the kin-openapi request-validation middleware in front of the strict handlers, enabled by default. This choice is confirmed or replaced (ogen, Huma v2) by a three-endpoint spike before implementation starts, covering spec-first generation, request validation, SSE documentation and the TS client. [OPN-027]

**API-053** (MUST) The TypeScript client for the web UI and Tauri apps uses **openapi-typescript** (types) and **openapi-fetch** (typed fetch). The Go client (CLI, tests) uses oapi-codegen's client generator. No other SDKs are maintained in v1.

**API-054** (MUST) The SSE endpoint is hand-written, documented in the spec as `text/event-stream` with an `Event` schema (`oneOf` discriminated by `type`); event payload types are generated.

### Live events (SSE)

**API-060** (MUST) One endpoint, `GET /api/v1/events?topics=<list>`, serves all live updates. Topics: `items`, `activity`, `jobs`, `health`, `requests`, `searches`, `logs`, `system`. A client opens one stream (browser tabs share it, API-073).

**API-061** (MUST) Framing: `id: <cluster-global sequence>`, `event: <type>`, `data: <JSON>`. Types:

| Type | Payload |
|---|---|
| `resource.changed` | `{resource, id, action: created|updated|deleted, version}` |
| `activity.progress` | `{downloadId, itemIds, progress, eta, speed}`, at most 1 per second per download, coalesced |
| `job` | `{id, state, progress}` |
| `health` | `{checks}` (codes and args) |
| `request` | `{id, status}` |
| `search.indexer_completed` | `{searchId, indexerId, outcome, count}`, one per indexer as it finishes an interactive search (API-107) |
| `pairing.claimed` | `{credentialId, device}` (to the pairing user only) |
| `log` | log record (`system:read:logs` only) |
| `reset` | none: the client must refetch everything visible |
| `principal.changed` | none: sent before the server closes the stream |

**API-062** (MUST) Change events carry ids and versions, never resource bodies; the UI invalidates its query cache and refetches with ETags. Only `activity.progress`, `job`, `health`, `request`, `search.indexer_completed` and `log` carry small inline payloads. When the principal's `principal_version` changes (IAM-016), the server sends `principal.changed` and closes; the client reconnects with fresh rights.

**API-063** (MUST) Every event is published with ACL metadata `{permission, kind?, libraryGroup?, ownerId?}` and delivered to a connection only if the principal passes the same check as IAM-063. Adult-kind and out-of-scope events are never delivered.

**API-064** (MUST) The server sends a heartbeat comment every 15 seconds and a `retry: 3000` hint. Responses set `Cache-Control: no-cache` and `X-Accel-Buffering: no`.

**API-065** (MUST) Resume: a client reconnects with `Last-Event-ID`. If that id is within the replay window, the server replays later events (permission-filtered) and continues; otherwise it sends `reset`. The replay window is at least 10 minutes or 50,000 events, whichever retains more. Event ids are cluster-global, so a client may resume on any HA node. This requires a retained, cluster-ordered event log from `12-platform.md`; NOTIFY alone is not sufficient (A4). [OPN-026]

**API-066** (MUST) On graceful shutdown a node sends `retry: 1000` and closes its streams, so clients reconnect to other nodes during rolling updates.

**API-067** (MUST) Credentials for SSE are the session cookie (browsers) or `Authorization: Bearer` (native transport). Query-string credentials are rejected (IAM-003). A per-principal connection cap applies (setting `api.sse.max_connections_per_principal`); exceeding it returns 429 `rate_limited`.

### Web client

**API-070** (MUST) The web UI is a SvelteKit static SPA (Svelte 5, static adapter) embedded in the core binary with `go:embed`, served at the base path with SPA fallback to `index.html`. _Source: D6._

**API-071** (MUST) One build works under any base path: assets use relative paths, and at serve time the core injects `<base href>` and `window.__BR_BASE__` into `index.html`.

**API-072** (MUST) The web UI authenticates only with the session cookie. It never receives, stores or embeds an API key or device credential. No bootstrap endpoint returns a credential to the SPA.

**API-073** (SHOULD) The SPA opens the SSE stream in a `SharedWorker`, falling back to one leader tab coordinating over `BroadcastChannel`, so several tabs use one connection.

**API-074** (MUST) The UI calls the API through a transport interface with two implementations: *browser* (same-origin `fetch`, cookie, native `EventSource`) and *native* (API-080). UI code is identical across clients.

### Desktop client (Tauri)

**API-080** (MUST) The Tauri desktop app bundles the same SvelteKit build. All API calls, including SSE, go through the native transport in Rust (`tauri-plugin-http` or equivalent Tauri commands), which adds `Authorization: Bearer brd_…`, enforces certificate pinning and streams SSE by fetch. The device secret **MUST NOT** be exposed to the webview's JavaScript. _Source: D12, D18._

**API-081** (MUST) Device credentials are stored in the OS keychain (macOS Keychain, Windows Credential Manager, Secret Service on Linux) via a keyring plugin. Tauri Stronghold is not used.

**API-082** (MUST) Certificate pinning: when a server profile has `certFingerprint` (SHA-256 of the SPKI), the native transport accepts only that key, and a mismatch is a fatal connection error, not a prompt. Without a fingerprint, normal WebPKI verification applies.

**API-083** (MUST) The desktop shell provides: server list and switching, pairing, a tray icon, native OS notifications derived from the SSE stream while the app runs, deep links (`bonnyread://`), "Open in browser", and start/stop of the local service (API-104).

**API-084** (MUST) Version negotiation: on connecting, the app reads `/meta`. It uses its bundled UI only if the server's `apiVersion` has the same major as the app's and a minor ≥ the minor the app was built against, and the app's API version ≥ the server's `minClientApiVersion`. Otherwise it shows "this server runs version X" and offers to open the server's own web UI in the system browser. The app **MUST NOT** load remote UI code into its webview. _Source: A7; D16 (self-updater deferred)._

**API-085** (MUST) The locally installed core and the desktop app are always the same version, because they are installed together (`13-deployment.md` DEP-030).

### Mobile clients (later)

**API-086** (MUST) Tauri mobile apps (later) reuse the UI and native transport. They require HTTPS (public CA or a pinned self-signed certificate). SSE runs only in the foreground; on resume the app reconnects with `Last-Event-ID` and handles `reset`.

**API-087** (MUST NOT) There is no mobile push notification in v1 (D20): no APNs/FCM relay and no project-hosted infrastructure. Background alerts use the existing notification targets (for example ntfy, Pushover, Telegram).

### Server discovery and switching

**API-088** (MUST) A client may hold any number of server profiles and switch between them. Credentials are stored per profile; revoking one server's device does not affect other profiles.

**API-089** (MUST) Discovery candidates, in order: (1) **local**: the local endpoint at its default path (API-091) answers `GET /api/v1/meta` with an added `local: {httpUrls[], certFingerprint?}`; (2) **LAN**: mDNS/DNS-SD `_bonnyread._tcp`, advertised only when LAN access is enabled; (3) **manual URL**; (4) **QR code or deep link** carrying URLs, `serverId` and fingerprint.

**API-090** (MUST) Connecting tries a profile's URLs in order (loopback, LAN, remote) with a 2-second timeout each and verifies that `/meta.serverId` matches the profile **before** sending any credential. A mismatch stops the connection and warns the user.

### Local endpoint and desktop pairing

**API-091** (MUST) The core serves a local endpoint: a Unix domain socket at `<data>/run/bonnyread.sock` (mode `0660`, owned by the service account and group `bonnyread`), or on Windows a named pipe `\\.\pipe\bonnyread` created with `FILE_FLAG_FIRST_PIPE_INSTANCE` and a DACL granting only the principals of API-093. The path is configurable (`LOCAL_ENDPOINT`, DEP-061). It speaks HTTP/1.1 and serves only `GET /api/v1/meta`, `/api/v1/local/*` and nothing else. _Source: D20; A15._

**API-092** (MUST) The server identifies the caller solely by peer credentials obtained from the kernel for the connection: `SO_PEERCRED` (Linux), `getpeereid`/`LOCAL_PEERCRED` (macOS), the client token of the named pipe (`ImpersonateNamedPipeClient`, Windows). Nothing in the request body or headers identifies the local caller. No pairing secret is written to any file.

**API-093** (MUST) Allowed local peers: the service account; `root` (Unix) or an elevated member of `BUILTIN\Administrators` (Windows); OS principals granted at install (`bonnyread service install --grant-pairing <user>`, stored as `LOCAL_PAIRING_PRINCIPALS`); and, on Unix, members of the OS group `bonnyread`. Other peers receive 403 `local_peer_not_allowed` and are audited.

**API-094** (MUST) CLI break-glass commands run over the local endpoint and require the peer to be the service account, `root` or an elevated Administrator: `bonnyread admin reset-password <user>`, `bonnyread admin login-link <user>` (a one-time sign-in link valid 10 minutes), `bonnyread admin create-service-account [--preset metrics|automation]`, `bonnyread setup-url`. Each **MAY** instead run directly against the database with `--dsn` and database credentials. Every use is audited.

**API-095** (MUST) The client verifies the server end before sending anything: the socket or pipe server process must run as the expected service account (Unix: peer credentials of the listening process; Windows: `GetNamedPipeServerProcessId` and its token SID, for example `NT SERVICE\bonnyread`). A mismatch aborts, which prevents another local process from squatting the endpoint.

**API-096** (MUST) First Owner over the local endpoint: while no Owner exists, `POST /api/v1/local/pair {device}` returns `{step: "create-owner", pairingId}` (bound to the peer principal, valid 10 minutes). `POST /api/v1/local/pair/complete {pairingId, username, password, totp?, device}` creates the Owner with scope all kinds, links a `local_os` identity for the peer principal (not for `root`/SYSTEM), and returns a device credential. TOTP is offered and is optional while the server is loopback-only (IAM-024).

**API-097** (MUST) Later local pairing: `POST /api/v1/local/pair {device}` from a peer principal that is linked (`local_os`) to a human user who has not disabled local pairing (user setting `allowLocalPairing`, default `true`) returns a device credential for that user. Otherwise it returns `{step: "sign-in"}`, and the app signs in normally over loopback HTTP (F4); after a successful sign-in the app **MAY** offer to link the OS account. All local pairings are audited as `pairing.claimed {method: local_endpoint}`. [OPN-120]

**API-098** (MUST) The HTTP pairing claim (API-099) **MUST NOT** accept local or bootstrap-by-file purposes; local pairing exists only on the local endpoint.

### Remote pairing and first-owner setup without a desktop app

**API-099** (MUST) Remote pairing: a signed-in user opens *Devices → Pair a device*, optionally choosing a cap. The server creates a `remote` pairing secret bound to that user and cap (256 bits, single use, 10 minutes, stored hashed) and shows a QR code and a 10-character entry code:
`bonnyread://pair?s=<serverId>&u=<url1>,<url2>&fp=<sha256 SPKI, if self-signed>&c=<secret>`.
The device verifies `serverId` and fingerprint and calls `POST /api/v1/pairing/claim {secret | code, device}`; the server returns a device credential with the user's permissions or the cap, and emits `pairing.claimed` to the user's streams. Claims are throttled by IAM-058. QR scanning applies to mobile apps (later); code entry works for a second desktop in P1.

**API-100** (MUST) Headless first Owner (container, headless package): when no Owner exists and no bootstrap owner is configured, the core creates a `bootstrap` pairing secret and, if `SETUP_URL_LOG` is true (default in the container image and headless packages, false for desktop installs), prints one line `https://<host>/setup#t=<secret>` to stdout. The secret travels in the URL fragment; the SPA posts it to `POST /api/v1/setup/claim` and then `POST /api/v1/setup/complete {username, password, totp?}`. An expired secret is replaced and the line printed again while no Owner exists. `bonnyread setup-url` (API-094) prints a fresh one on demand. Once an Owner exists, both endpoints return 404.

**API-101** (MUST) Unattended first Owner: `BOOTSTRAP_OWNER_OIDC_GROUP` (members of that group at the first OIDC login of the configured provider become Owner) or `BOOTSTRAP_OWNER_USERNAME` with `BOOTSTRAP_OWNER_PASSWORD_FILE` (DEP-061). When set, no setup URL is printed.

### Native sign-in

**API-102** (MUST) Native apps sign in with OIDC via the system browser (RFC 8252): `GET /api/v1/auth/oidc/{slug}/start?client=native&redirect=<uri>&codeChallenge=<S256>`, where `redirect` is `bonnyread://auth` or a loopback `http://127.0.0.1:<port>/…` URI. After the provider login the core redirects to it with a one-time exchange code (single use, 60 seconds). `POST /api/v1/auth/native/exchange {code, codeVerifier, device}` returns a device credential. Password and media-server sign-ins with `client: native` and a `device` object also return a device credential instead of a cookie.

### Offline and degraded behaviour

**API-103** (MUST) When the server is unreachable, clients show a banner with retry (exponential backoff capped at 30 s) and show the last fetched views read-only from the query cache (memory, plus a small persisted snapshot of the dashboard, calendar and queue in native apps). Mutations are disabled, never queued; there are no offline writes.

**API-104** (MUST) Desktop: when the local service is stopped, the app offers *Start service* (the installer grants the installing user start/stop rights, DEP-031) and shows the tail of the service log when it is readable.

**API-105** (MUST) When a node reports `/readyz` false, the load balancer routes elsewhere; the UI shows domain health from `health` events and `GET /api/v1/health`, which never affect readiness.

### Operations defined by other files

**API-106** (MUST) The spec **MUST** include the operations that other files define, with the stated permission and the behaviour of the owning requirement (this file owns only the representation conventions above):

| Operation | `x-permission` | Defined by |
|---|---|---|
| `GET /livez`, `GET /readyz` | `public` | API-006; `12-platform.md` PLT-095; `13-deployment.md` DEP-012 |
| `GET /metrics`, `GET /metrics/cluster` | `metrics:read` | `12-platform.md` PLT-096, PLT-097, PLT-099 |
| `POST /api/v1/items/{id}/search` (automatic search, returns a job) | `library:execute:search` | `01-architecture.md` F2; `05-downloads.md` DLD-110 |
| `POST /api/v1/searches`, `GET /api/v1/searches/{searchId}/results` (interactive search, API-107) | `library:execute:search` | `03-acquisition.md` ACQ-091, ACQ-092; `04-evaluation.md` EVL-100 |
| `POST /api/v1/indexers/{id}/announce` (inject raw announce lines) | `indexer:execute` | `03-acquisition.md` ACQ-145 |
| `POST /api/v1/download-clients/{id}/notify` (push hint; body ignored beyond an optional client reference) | `download_client:execute` | `05-downloads.md` DLD-026 |
| `POST /api/v1/downloads/{id}/processing` (processing-hold result; `409 conflict` when not in `processing`) | `activity:update:processing` | `05-downloads.md` DLD-079 |
| `GET /api/v1/metadata/sources` | `system:read` | `08-metadata.md` MET-096 |
| `GET /api/v1/profile-databases/{db}/conflicts`, `POST …/conflicts/{patchId}/resolve`, `GET …/update-preview` | `profile:read` / `profile:update` | `07-profiles.md` PRF-044, PRF-094, PRF-107 |
| `POST /api/v1/profiles/test/custom-format`, `…/regex`, `…/profile`, `…/compare` | `profile:read` | `07-profiles.md` PRF-090–PRF-093 |
| `POST /api/v1/<aggregate>/{id}/reset`, `…/clone` | `profile:update` | `07-profiles.md` PRF-037, PRF-107 |
| `POST /api/v1/requests`, `POST /api/v1/requests/{id}/approve` | `request:create` / `request:approve` | `10-identity-and-access.md` F5, IAM-069, IAM-083 |
| `GET /api/v1/users/me/quota`, `GET /api/v1/users/{id}/quota` | authenticated / `user:read` | `10-identity-and-access.md` IAM-084 |
| `GET /api/v1/audit-events` | `audit:read` | `10-identity-and-access.md` IAM-092 |
| `POST /api/v1/system/shutdown` | `system:manage` | `12-platform.md` PLT-026 |

Profile operations use the permission resource `profile` of `10-identity-and-access.md` IAM-062, with the proposed mapping of `07-profiles.md` PRF-108 onto IAM-061's actions [OPN-205].

**API-107** (MUST) Interactive search **MUST** stream per indexer: `POST /api/v1/searches` (with an `Idempotency-Key`) starts the search and returns `202` with `{searchId}`; as each indexer completes, the server emits `search.indexer_completed` on the `searches` topic to the requesting principal; the client then fetches that indexer's results, with per-slot verdicts (`04-evaluation.md` EVL-100), from `GET /api/v1/searches/{searchId}/results?indexerId=…` (cursor-paged, API-024). Results are served from the 60-minute `SearchResult` store (`03-acquisition.md` ACQ-091), so any node can serve them. _Source: ACQ-092; API-062 (notify, then fetch)._

## Data model

Job, the event log and settings are owned by `12-platform.md`; User, Credential and Identity by `10-identity-and-access.md`.

```
IdempotencyKey {
  edges: user -> User; key string; request_hash bytes (sha256 of method, path, body)
  state enum(in_progress, completed); status int?; response_body bytes?; response_headers json?
  created_at time; expires_at time (created_at + 24 h)
  unique (user_id, key); index (expires_at)
}

PairingSecret {
  id; purpose enum(bootstrap, remote, local_pending)   // local_pending = pairingId of API-096
  secret_hash bytes; code_hash bytes?                  // 10-character entry code (remote)
  edges: user -> User?                                 // remote: the creating user
  cap []string?; peer_principal string?                // local_pending: bound OS principal
  expires_at time; consumed_at time?
  index (expires_at)
}

// Client side (Tauri), not in the server database:
ServerProfile { serverId uuid; name; urls []string (ordered); certFingerprint string?; credentialRef (keychain id); lastSeen time }
```

## Flows

### F1. First run on a Windows desktop

1. The NSIS installer runs `bonnyread.exe service install --data "%ProgramData%\bonnyread" --grant-pairing <installing user SID>` and starts the service (DEP-030).
2. The core writes default bootstrap config (loopback `127.0.0.1:7979`, SQLite, TLS off), migrates, opens `\\.\pipe\bonnyread`. No Owner exists.
3. The installer launches the app. The app opens the pipe, verifies the server process runs as `NT SERVICE\bonnyread` (API-095), and calls `GET /api/v1/meta` (`setupComplete: false`, `local.httpUrls: ["http://127.0.0.1:7979"]`).
4. `POST /api/v1/local/pair`. The server reads the client token SID; it is in `LOCAL_PAIRING_PRINCIPALS`. Response `create-owner`.
5. The user enters username and password (TOTP optional). `POST /local/pair/complete` creates the Owner, links `local_os` for the SID, audits `pairing.claimed` and `user.created`, returns `brd_…`.
6. The app stores the credential in Credential Manager, saves the server profile, and connects over `http://127.0.0.1:7979` with the bearer token, then opens SSE.
7. Failure paths: another Windows user on the same PC runs the app → 403 `local_peer_not_allowed`, and the app offers normal sign-in (after the owner has created an account for them). The pipe server is not the service account → abort with a warning.

### F2. Container first run

1. The container starts as UID 65532 with no Owner and no bootstrap owner. It prints `https://media.example.com/setup#t=…` to stdout.
2. The operator opens it. The SPA posts the fragment to `/setup/claim`, then `/setup/complete`. The Owner is created and signed in with a session.
3. Alternative: `docker exec <container> /bonnyread setup-url` prints a fresh URL (the peer is the service account).
4. If the secret expires unused, a new one is created and printed.

### F3. Pairing a second device by code

1. Signed in on the web, the user opens *Devices → Pair a device*, cap "read-only". The server shows a QR code and `K7Q2-M9XA-3P`.
2. On a second computer the desktop app adds a server by URL, checks `serverId`, and submits the code to `/pairing/claim`.
3. The server validates (hash, unexpired, unconsumed, throttle), consumes the secret and returns a capped device credential. The web page receives `pairing.claimed` and shows the device.
4. Failure: wrong or expired code → 401 `invalid_credentials`, counted by the limiter.

### F4. Native OIDC sign-in and HA stream resume

1. The app opens the system browser at `/auth/oidc/authentik/start?client=native&redirect=bonnyread://auth&codeChallenge=…`.
2. After provider sign-in, the core redirects to `bonnyread://auth?code=…`; the app posts `/auth/native/exchange` with its verifier and gets `brd_…`.
3. The app opens `GET /api/v1/events?topics=items,activity,requests` via the native transport; the ingress routes to pod 2.
4. Pod 2 drains: `retry: 1000`, close. The app reconnects with `Last-Event-ID: 88412`; pod 3 replays later events filtered by the user's permissions.
5. After hours suspended, the id is outside the window → `reset` → the app refetches visible screens.
6. The admin revokes the device → next request 401 `credential_revoked` → the app returns to sign-in for that server only.

### F5. Settings edit conflict

1. Admin A and Admin B `GET /api/v1/quality-profiles/{id}` → `ETag: "17"`.
2. A sends `PATCH` with `If-Match: "17"` → 200, `ETag: "18"`.
3. B sends `PATCH` with `If-Match: "17"` → 412 `precondition_failed`; the UI shows the diff and refetches.
4. A script sends `PATCH` without `If-Match` → 428 `precondition_required`.

## Non-goals

| Excluded | Reason |
|---|---|
| Sonarr/Radarr/Lidarr/Seerr API façades, Newznab/Torznab server | D7: no external compatibility layers |
| `POST /command {name}` command bus; `/api/v3` and `/api/v5` side by side | Action sub-resources and one API version (API-012, API-003) |
| Offset/page paging | Unstable under inserts; silent truncation (API-024) |
| SignalR, WebSocket | The UI needs only server-to-client events; one SSE stream (API-060) |
| Generic filter language | Named filters are enough in v1 (API-026) |
| Maintained SDKs beyond TS and Go | The spec is published for users to generate their own (API-053) |
| Token file in the data directory for pairing; loopback HTTP pairing claims | Replaced by the local endpoint with peer credentials (D20) |
| Mobile push (APNs/FCM relay) in v1 | D20 |
| Offline writes and sync | Nothing to reconcile; mutations are disabled offline (API-103) |
| Loading a remote server's UI into the desktop webview | Privileged webview; fallback is the system browser (API-084) |

## Traceability

| Reference | Preserved | Changed (D19) |
|---|---|---|
| Sonarr/Radarr/Prowlarr | Background commands with progress (O8); live UI updates (O7); URL base for sub-path hosting (O11) | Commands become action sub-resources returning jobs; SignalR becomes SSE; no `/initialize.json` returning the API key |
| Seerr | OpenAPI-described API (O9) | No `PUT` with id in body; cursor paging instead of page/pageSize |
| Profilarr | Contract-first OpenAPI with openapi-typescript; `x-permission` enforced by lint; SSE for jobs | Extended to a public CRUD API; oasdiff gate |
| autobrr | SSE for live updates | Cluster-global ids with resume and ACL filtering |
| Unpackerr | `openapi.json` published | WebSocket replaced by SSE |
| Tray apps (six forks, Prowlarr, Jackett, Unpackerr) | Tray icon and start/stop | One Tauri app for UI and tray; pairing replaces "local means trusted" |
