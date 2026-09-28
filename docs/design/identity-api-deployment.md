# Design: identity, API, clients and deployment

Status: proposal, 2026-09-28. Scope: authentication and authorisation (D17), the public API and live events (D7, D12), clients and pairing (D6, D12, D18), and deployment from Kubernetes HA to a single desktop (D14, D16, D18). Domain models, the job queue, leader election and the event bus internals belong to other design documents; this one states what it needs from them.

Paths are relative to `submodules/` unless they start with `docs/`. Tool facts were checked on 2026-09-28 against current docs (Context7: oapi-codegen v2, ogen, Huma v2, Tauri 2 and plugins-workspace; web: kardianos/service, Tauri Stronghold status).

---

## 1. Outcomes and invariants to preserve

The references agree on these outcomes, even though every one of them implements them differently. Each outcome is something bonnyread must keep.

| # | Outcome | Evidence |
|---|---|---|
| O1 | A self-hoster can sign in with a local password, or delegate to an OIDC provider and optionally hide local login. | Sonarr `AuthenticationType` {None, Basic, Forms, External, Oidc} (`sonarr/src/NzbDrone.Core/Authentication/AuthenticationType.cs`, `OidcDiscoveryService.cs`); autobrr `oidcDisableBuiltInLogin` (`autobrr/internal/auth/oidc.go`, `internal/http/oidc.go`); Profilarr `OIDC_DISCOVERY_URL` (`profilarr/src/lib/server/utils/auth/oidc.ts`); Cleanuparr OIDC + Plex login (`cleanuparr/code/backend/Cleanuparr.Api/Auth`) |
| O2 | Household members sign in with the media-server account they already have (Plex, Jellyfin, Emby), and only people who have access to *this* media server get in. | Seerr `S/routes/auth.ts`: Plex PIN flow, checks server access via `plex.tv/api/users`/`resources`; Jellyfin `/Users/AuthenticateByName` and Quick Connect; `newPlexLogin` (`seerr/server/lib/settings/index.ts:147`) |
| O3 | Different users get different rights: admins manage settings, trusted users get auto-approval, others request and wait; rights can differ per media kind and per quality tier (4K). | Seerr's 30-flag bitmask (`seerr/server/lib/permissions.ts`); default for new users is `REQUEST` only (`settings/index.ts:415`) |
| O4 | Request volume can be limited per user: N requests per D days, per media kind, with a global default and per-user override; TV counts seasons. Managers can bypass the quota for a request. | Seerr `User.getQuota` (`seerr/server/entity/User.ts:126-135` quota columns), `ignoreQuota` needs MANAGE_REQUESTS ([seerr.md](../analysis/seerr.md) §requests) |
| O5 | Automation and scripts reach the API with a key that can be limited in scope, named, expired and revoked. | Profilarr: named, bcrypt-hashed keys with per-area read/write and expiry, enforced from `x-permission` in the OpenAPI spec (`profilarr/docs/api/v1/openapi.yaml:77`, `profilarr/src/lib/server/utils/auth/apiPermissions.ts`); Unpackerr: `area:resource:verb` permissions and custom roles (`unpackerr/pkg/unpackerr/permissions.go`, `apikeys.go`) |
| O6 | Brute-force login attempts are throttled and the throttle survives restarts. | Profilarr `rateLimit.ts` (DB-backed, 15-minute window, category thresholds 3/10/10) and `loginAnalysis.ts` |
| O7 | The UI updates live when state changes (queue, items, health, jobs, logs). | SignalR hub `receiveMessage {name, body:{action, resource}}` (Sonarr `NzbDrone.SignalR/MessageHub.cs`, Prowlarr, Cleanuparr); autobrr SSE (`r3labs/sse`, `autobrr/internal/http/server.go`); Unpackerr WebSocket (`unpackerr/pkg/unpackerr/livews.go`); Profilarr SSE `/jobs/events` |
| O8 | Long operations (searches, refreshes, imports, renames) run in the background and report progress and outcome. | Arr `POST /command` + command resource ([sonarr.md](../analysis/sonarr.md)); Prowlarr commands; Cleanuparr jobs over SignalR |
| O9 | The API is documented and machine-readable, so clients can be generated. | Seerr `express-openapi-validator` over `seerr-api.yml`; Profilarr contract-first `openapi.yaml` → `src/lib/api/v1.d.ts`; Unpackerr `pkg/unpackerr/openapi.json` |
| O10 | Prometheus can scrape metrics without exposing the whole API. | autobrr separate metrics server with basic auth; Unpackerr `/metrics` with API key or bearer (`unpackerr/pkg/unpackerr/metrics.go:108-148`); D7 |
| O11 | Runs anywhere: Windows service/tray, macOS, Linux packages, Docker, behind a reverse proxy under a sub-path. | Six forks + Prowlarr + Jackett ship tray, Windows service and Inno Setup ([verification/outdated-and-security.md](../verification/outdated-and-security.md) §3.3); `urlbase` everywhere; Profilarr `ORIGIN` for proxies |
| O12 | Secrets are never shown back to the browser after they are saved. | Profilarr strips secrets from responses and backups (`tests/integration/auth/specs/secretExposure.test.ts`, `utils/backup/sanitize.ts`) |

Invariants bonnyread adds, because references violate them ([verification/outdated-and-security.md](../verification/outdated-and-security.md) §3.4):

- **I1 Auth is always on.** No "None" mode and no "disabled for local addresses". The references that allow either default to open: Kapowarr returns the API key to anyone when no password is set (`kapowarr/frontend/api.py:223-251`); Jackett auto-signs-in without a password (`jackett/src/Jackett.Server/Services/SecurityService.cs:17-18`); Bazarr defaults `auth.type` to None (`bazarr/bazarr/app/config.py:186`).
- **I2 Credentials only in headers or cookies, never in query strings.** The Arrs accept `?apikey=`/`?access_token=` (`sonarr/src/Sonarr.Http/Authentication/AuthenticationBuilderExtensions.cs:49-55`), and Seerr and Bazarr send it (`seerr/server/api/servarr/base.ts:104-107`); query strings end up in proxy logs and browser history.
- **I3 Strong password hashing only (argon2id).** Bazarr uses unsalted MD5; Jackett uses one SHA-512 round salted with the API key.
- **I4 TLS verification always on for outbound calls**, with explicit per-connection trust (pinned certificate or custom CA) instead of `verify=False` (Bazarr has 28 such calls).
- **I5 No telemetry and no hard-coded third-party secrets** (Sentry DSNs in the forks, Bazarr GA4 default on; D3b, D16).
- **I6 The first account cannot be claimed by whoever reaches the port first.** autobrr's `POST /api/auth/onboard` and the Arrs' first-run form are first-come, first-served.
- **I7 One permission check for every entry point**: UI session, API key, device, SSE stream and CLI all go through the same authoriser. Seerr's global admin key plus `X-API-User` impersonation (`seerr/server/middleware/auth.ts:13-22`) and Bazarr's "API key injected into the SPA" are the anti-patterns.

---

## 2. How the references do it today, and the accidental complexity

| Concern | Today | Accidental complexity (D19: drop) |
|---|---|---|
| Accounts | Every app has its own: single-user Arrs, Prowlarr, autobrr, Profilarr, Bazarr, Kapowarr, Jackett (password only), Unpackerr; Seerr is the only multi-user app | A person logs into up to 16 apps; 16 API keys are copied between apps so they can call each other. In a monolith the inter-app keys disappear and only human and automation principals remain |
| Auth modes | Sonarr: None/Basic/Forms/External/OIDC + `AuthenticationRequired` + `TrustedNetworks` + `TrustCgnatIpAddresses` + `AllowedHosts` (`sonarr/src/Sonarr.Http/Authentication/*`, 10 files); Prowlarr the same minus OIDC; Cleanuparr `DisableAuthForLocalAddresses`, `TrustForwardedHeaders`, `TrustedNetworks` | "Local means trusted" exists because users could not be bothered to log in to many apps. Pairing (§3.4) replaces it with a one-time local handshake |
| Permissions | Seerr bitmask of 30 flags where 4K doubles most of them (`REQUEST_4K_MOVIE`, `AUTO_APPROVE_4K_TV`, ...); ADMIN implies all; VOTE unused. Unpackerr/Profilarr: per-area read/write on API keys only. Everyone else: all or nothing | 4K flags exist because 4K is a second Radarr (D4 removes that). Per-kind flags are hand-multiplied (MOVIE × TV × 4K). bonnyread uses permission × scope instead |
| API keys | Arrs: one global key, header or query. Seerr: one global admin key + impersonation header. autobrr: named keys, `scopes` column unused (`autobrr/internal/http/middleware.go:73`). Profilarr: hashed, scoped, expiring | Only Profilarr's model is kept |
| Media-server login | Seerr: 1,065-line `routes/auth.ts`; user rows carry `plexToken`, `jellyfinAuthToken`, `jellyfinDeviceId`; `userType` enum; one active server type | Identity is a column set per provider; bonnyread uses an `identity` table (one row per linked login) so a user can have OIDC + Plex + password |
| Sessions | Seerr express-session in DB (30 days) + csurf; autobrr scs in DB; Profilarr server sessions with sliding 168 h; Sonarr ASP.NET cookie | Fine; unify in one DB-backed session table (needed for HA anyway) |
| Live updates | SignalR in 7 .NET apps (+ Bazarr's three SignalR *clients* to consume them, `bazarr/bazarr/app/signalr_client.py`), SSE in autobrr/Profilarr, WebSocket in Unpackerr | Bazarr's SignalR clients and the Arrs' hub-for-other-apps role vanish in-process. The UI needs one one-way stream: SSE |
| API style | Arr: `/api/v3` and `/api/v5` side by side with 9 `[Obsolete]` files, `POST /command` with a name string, page/pageSize paging (Seerr and Unpackerr trip over default page sizes); Seerr `PUT` with id in body; Profilarr has no public CRUD API (form actions only) | Commands as name strings, two API versions in one binary, offset paging |
| Frontend bootstrap | Sonarr `/initialize.json` returns the API key to the SPA; Bazarr injects the key into the page | The SPA authenticates with its session cookie only |
| Packaging | Tray apps, Windows service installers, Inno Setup, firewall COM and `RegisterUrl` in six forks + Prowlarr + Jackett; Unpackerr systray + zenity; self-updaters in forks, Bazarr, Jackett, Unpackerr; PUID/PGID root entrypoints in Profilarr, Kapowarr (`ENV PUID=0`), Cleanuparr | One core binary with a `service` subcommand; one Tauri app is the tray and UI; no self-updater (D16); containers run non-root without PUID/PGID rewriting |
| Telemetry | Sentry DSNs in six forks + Prowlarr (`NzbDroneLogger.cs:~75`), Bazarr GA4 on by default (`bazarr/bazarr/utilities/analytics.py:28`) | None; metrics are local and scraped (D7) |
| Metrics | autobrr: separate server + htpasswd created by `autobrrctl`; Unpackerr: `/metrics` behind key | One `/metrics` handler; access via a Service principal with `metrics:read` |

---

## 3. Proposed design

### 3.1 Identity model

```
Principal = User (kind: human | service)
User ──< Identity      (provider: password | oidc:<issuer> | plex | jellyfin | emby; subject; claims snapshot)
User ──< Credential    (kind: session | device | api_key; hashed secret; permissions subset; expiry; last_used)
User ─── RoleAssignment(role, scope, source: manual | oidc_group | media_server_default)
User ─── QuotaOverride (per media kind)
Role ──< Permission    (resource:action)
Group mapping rules    (provider, claim value → role [+ scope])
AuditEvent             (actor credential, action, target, diff, ip, outcome)
```

- **One user table for everyone.** Humans (owner, household, request users) and service accounts (automation, metrics, a CLI on another host). A service account cannot log in interactively and owns only API keys.
- **Identities are linked logins**, not user types. A user created by Plex login can later add a password or an OIDC link. Linking requires the user to be signed in to both sides; **never auto-link by e-mail** (account takeover risk) unless an admin enables "link by verified e-mail" for a specific OIDC provider that asserts `email_verified`.
- **Credentials are one table with three kinds.** A session (browser cookie), a device (native app after pairing or login) and an API key differ only in how they are presented, their lifetime and whether they are narrowed. All three resolve to `(user, effective permissions)` through the same authoriser (I7).

### 3.2 Authentication methods

**Built-in accounts (argon2id).**
- Parameters: argon2id, m = 64 MiB, t = 3, p = 2, 16-byte salt, 32-byte key, PHC string format so parameters can be raised later and re-hashed on next login. OWASP's minimum (m = 19 MiB, t = 2) is the floor for low-memory devices; the owner can lower it in bootstrap config, never below the floor. autobrr's `pkg/argon2id` is a usable reference (GPL-2.0-or-later, usable under D5a).
- A global semaphore limits concurrent hashes to `GOMAXPROCS` so the login endpoint cannot be used to exhaust memory.
- Minimum password length 12; check against a bundled top-100k breached-password list (offline; no HIBP call, D3c spirit). No composition rules.
- **TOTP second factor** (RFC 6238) with ten single-use recovery codes, as Cleanuparr has. Required for Owner and Admin when the server is reachable beyond loopback (configurable). Passkeys (WebAuthn) are an open question (§6).
- Password reset: by an Admin, or by e-mail only if SMTP is configured (single-use, 30-minute link, hashed token). CLI break-glass: `bonnyread admin reset-password <user>` needs file-system access to the data directory or DB credentials.

**OIDC.**
- Any number of providers. Each has: issuer URL (discovery via `/.well-known/openid-configuration`), client id, client secret (or none for public clients), scopes (default `openid profile email` plus `groups` if configured), and claim mapping.
- Flow: authorisation code + PKCE (S256) always, `state` and `nonce`, ID-token validation (issuer, audience, expiry, signature via JWKS with cached keys and refresh on unknown `kid`). Userinfo is fetched only if the mapped claims are missing from the ID token.
- Discovery failure is **not fatal** (autobrr exits on it, `main.go`): the provider is marked unhealthy, retried with backoff and shown as a health issue; password login keeps working unless disabled.
- **Claim mapping:** `username` (default `preferred_username`), `display name` (`name`), `email` (+`email_verified`), `groups` (a dotted path, so `groups`, `realm_access.roles` or `resource_access.bonnyread.roles` all work).
- **Group → role mapping rules:** an ordered list of `{claim value, role, scope?}`. On every login bonnyread recomputes the user's `oidc_group` assignments: new matches are added, stale ones removed (**group sync**). Manual assignments are left alone. The effective role is the most privileged of all assignments. If no rule matches, the provider's `default role` applies (default: none, meaning "deny login"), so an IdP with thousands of users does not flood the user list.
- **Just-in-time provisioning** is a per-provider toggle (default on). Off means only pre-created users (matched by `sub`, or by username if an admin pre-linked them) may sign in.
- Logout: local session end plus RP-initiated logout when the provider advertises `end_session_endpoint`. Back-channel logout is optional (open question).
- "Disable local login" hides the password form for everyone except the break-glass path: the Owner can always sign in with a password over loopback, or with a CLI-minted one-time login link.

**Media-server identity (Plex, Jellyfin, Emby).**
- Configured once as part of the media-server integration (one client per server serves auth, library scan and refresh; [comparison.md](../analysis/comparison.md) §2 media-server clients). Several servers may be configured; each enables or disables sign-in separately.
- Plex: the browser/app runs the plex.tv PIN flow (`plex.tv/api/v2/pins` → `app.plex.tv/auth`); the core validates the token with plex.tv and checks the account has access to the configured server (owner, home user or shared user). Jellyfin/Emby: username + password posted to the core, which calls `/Users/AuthenticateByName`, or Quick Connect.
- Sign-in policy per server: `off | existing users only | anyone with access to this server`, plus the default role (default **Requester**) and default scope (all non-adult kinds). This is Seerr's `mediaServerLogin`/`newPlexLogin` pair expressed once.
- The media-server token is stored only if the user opts into a feature that needs it (Plex watchlist auto-requests); otherwise it is discarded after verification. Server access is re-checked at most daily and at each login; losing access disables the identity and revokes its sessions.

**Trusted-header (forward-auth) mode** is not in D17 and is **not** proposed as a default. It is listed in §6 as an open question because Authelia/Authentik forward-auth users will ask for it.

### 3.3 Sessions, devices and API keys

| Credential | Presented as | Lifetime | Narrowing | Stored |
|---|---|---|---|---|
| Browser session | Cookie `__Host-br_session` (HttpOnly, Secure, SameSite=Lax, Path=/); falls back to `br_session` without Secure only when served over plain HTTP on a non-loopback address, with a persistent UI warning | Sliding 7 days idle, 30 days absolute | none (full user permissions) | SHA-256 of a 256-bit random token |
| Device | `Authorization: Bearer brd_<id>_<secret>` | Sliding 90 days idle, no absolute limit; listed under *Devices* and revocable | optional cap (e.g. a phone paired as "read-only") | SHA-256 of secret |
| API key | `Authorization: Bearer brk_<id>_<secret>` or `X-Api-Key` header | Optional expiry (default 1 year, "never" allowed for Service accounts) | **required** permission list, always ⊆ owner's effective permissions at each request | SHA-256 of secret; first 4 chars kept as a hint |

- A 256-bit random secret does not need a slow hash; SHA-256 lookup keeps every request cheap on every HA node. The `<id>` part makes lookups O(1) and lets secret-scanning tools recognise the prefix.
- **Query-string credentials are rejected** with 400 and a problem type that explains why (I2). This includes `/metrics` and SSE (see §3.7 for how native EventSource works without it).
- **CSRF:** unsafe methods with a cookie session must pass an `Origin`/`Sec-Fetch-Site` same-origin check; requests without either header are rejected. Bearer-authenticated requests are exempt (no ambient credential). CORS is closed by default; the only allowed cross origins are the Tauri app origins, and only for bearer requests.
- **Effective permissions are computed per request** from DB state cached for a few seconds and invalidated by a `principal_version` bump on any role, assignment or credential change (the bump is broadcast on the event bus so all nodes drop their cache). Downgrades therefore apply within one request, and SSE streams of that principal are closed and must reconnect.
- **Login throttling:** DB-backed counters shared across HA nodes. Per (IP, username) and per username: 5 failures in 15 minutes → exponential delay (1 s doubling to 15 min); per IP: 30 failures in 15 minutes → 429 for 15 minutes. Profilarr's "suspicious username" category (`loginAnalysis.ts`) is kept: attempts with names like `admin` or `root` that do not exist count triple. Responses and timing are identical for unknown users and wrong passwords (dummy hash). Same limiter covers pairing claims, TOTP and password reset.
- API rate limiting: a loose per-credential token bucket (default 20 req/s burst 100) to protect the server from runaway scripts; not a quota.

### 3.4 Authorisation: permissions, scopes, roles

**Permission = `resource:action`.** Actions: `read`, `create`, `update`, `delete`, `execute` (search, grab, import, rename, test), `approve`, `manage` (settings of that resource family). The spec lists every operation's permission (§3.6), so the table below is the complete vocabulary for phase 1–3; domain designs may add resources, never new actions.

| Resource | Covers |
|---|---|
| `library` | items, slots, files, history of an item, calendar, wanted lists, manual import |
| `request` | requests (own vs all is a scope), approvals |
| `issue` | issue reports on items |
| `discover` | metadata search/browse for requesting and adding |
| `activity` | queue, download progress, blocklist, history (global) |
| `indexer` | indexers, IRC networks, feeds, indexer proxies, DDL hosts |
| `download_client` | download clients and download-handling policies |
| `profile` | quality profiles, custom formats, PCD databases, naming presets |
| `media_management` | root folders, naming, file handling, remote path mappings, extraction |
| `import_list` | import lists and followed entities |
| `subtitle` | subtitle providers and subtitle operations |
| `notification` | notification/alert targets and webhooks |
| `media_server` | Plex/Jellyfin/Emby connections |
| `user` | users, role assignments, quotas, identities |
| `role` | custom roles, OIDC group mapping rules |
| `credential` | other users' API keys, devices and sessions (own ones are always manageable) |
| `auth_settings` | password policy, OIDC providers, media-server sign-in, pairing |
| `system` | general settings, backups, logs, jobs, health, plug-ins |
| `metrics` | `/metrics` |
| `audit` | audit log |

**Scopes** narrow a grant. Only three scope dimensions exist, because they are the ones the references need:
- **media kind** (`tv`, `movie`, `music`, `book`, `adult`, `comic`),
- **library** (a root-folder group; e.g. "Kids TV"), and
- **ownership** (`own` vs `any`) for `request` and `issue`.

A scope is attached to a *role assignment*, not to each permission: "Member, kinds = {tv, movie}". Every resource row that is scoped carries its kind and library, so filters apply to lists, reads, writes and SSE events alike. Configuration resources (indexers, profiles, …) are not scoped; a role either may or may not manage them.

**Adult content is opt-in per user.** No built-in role includes kind `adult` in its default scope except Owner; an Admin must add it explicitly. Discovery, search results, calendar and SSE events for adult items are filtered by the same scope check, so hiding is not a UI-only concern.

**Slots instead of 4K flags.** Seerr's `REQUEST_4K*`/`AUTO_APPROVE_4K*` become a per-slot setting (D4): each slot definition has `requestable by: [roles]` and `auto-approve for: [roles]`. So "Members may request 1080p; only Managers get 4K" is two settings, not six flags.

**Default roles** (built in, not deletable; their permission sets are fixed so upgrades can extend them predictably; custom roles can be cloned from them):

| Permission | Owner | Admin | Manager | Member | Requester | Viewer | Service (template) |
|---|---|---|---|---|---|---|---|
| `library:read` | ✓ | ✓ | ✓ | ✓ | – (discover only) | ✓ | ✓ |
| `library:create/update/delete` | ✓ | ✓ | ✓ | – | – | – | opt |
| `library:execute` (search, grab, import, rename) | ✓ | ✓ | ✓ | search only | – | – | opt |
| `discover:read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `request:create` | ✓ | ✓ | ✓ | ✓ | ✓ | – | opt |
| `request:read` | any | any | any | any | own | own | opt |
| `request:approve` | ✓ | ✓ | ✓ | – | – | – | – |
| auto-approve (per slot setting) | ✓ | ✓ | ✓ | ✓ (default slot) | – | – | – |
| quota applies | no | no | no | yes (default unlimited) | yes (default 10 movies + 10 seasons / 7 days) | n/a | no |
| bypass quota on a request | ✓ | ✓ | ✓ | – | – | – | – |
| `issue:create` / `issue:read` | ✓/any | ✓/any | ✓/any | ✓/any | ✓/own | –/own | – |
| `issue:update` (resolve) | ✓ | ✓ | ✓ | – | – | – | – |
| `activity:read` | ✓ | ✓ | ✓ | ✓ | – | ✓ | ✓ |
| `activity:update/delete` (remove, blocklist) | ✓ | ✓ | ✓ | – | – | – | opt |
| `indexer`, `download_client`, `media_management`, `import_list`, `subtitle`, `notification`, `media_server`: read | ✓ | ✓ | ✓ | – | – | – | opt |
| same: create/update/delete/execute | ✓ | ✓ | – | – | – | – | opt |
| `profile:*` | ✓ | ✓ | ✓ | – | – | – | opt |
| `user:*` (except Owner/Admin assignments) | ✓ | ✓ | – | – | – | – | – |
| assign Admin, `role:*`, `auth_settings:*` | ✓ | – (read only) | – | – | – | – | – |
| `credential:*` (other users) | ✓ | ✓ (not Owner's) | – | – | – | – | – |
| `system:read` (health, logs, jobs) | ✓ | ✓ | health, jobs | health | – | – | opt |
| `system:manage` (settings, backups, plug-ins) | ✓ | ✓ | – | – | – | – | – |
| `metrics:read` | ✓ | ✓ | – | – | – | – | opt |
| `audit:read` | ✓ | ✓ | – | – | – | – | – |
| default scope | all kinds incl. adult | all non-adult | all non-adult | all non-adult | all non-adult | all non-adult | – |

- **Owner:** at least one, at most a few; created by the bootstrap handshake; the only role that manages authentication settings and Admins; cannot be removed if it is the last one; ownership transfer is explicit.
- **Admin:** runs the server; everything except auth settings and Owner accounts.
- **Manager:** curates the library and the request queue (the OIDC walkthrough maps a group here).
- **Member:** trusted household user: browses the library, requests with auto-approval on the default slot, reports issues, sees activity.
- **Requester:** the Seerr default user: discovers and requests under a quota, sees own requests and issues. This is the default for media-server sign-ins.
- **Viewer:** read-only (library, calendar, activity); cannot request.
- **Service:** a *template* for service accounts: it has no permissions until the creator ticks them (the "opt" column). Two presets are offered: **Metrics scraper** (`metrics:read`) and **Automation** (`library:read`, `library:execute`, `activity:read`, `request:create`).

**Quotas** (Seerr semantics kept, O4): per media kind, `limit` per rolling `days`; TV counts seasons, music counts albums, books count books, comics count issues-or-volumes as the domain design decides. Resolution order: user override → role default → global default. A request that would exceed the quota fails with `quota_exceeded` and the reset time. Declined requests do not count. A Manager may approve with `bypassQuota`. Quota state is computed from the request table, not a counter, so it survives edits.

**Authoriser.** One Go function `Authorize(principal, permission, target)` where `target` carries `{kind, library, owner}`; list endpoints call `ScopeFilter(principal, permission)` that returns a SQL predicate (ent predicate) so lists and cursors stay correct. No other code checks roles.

### 3.5 Audit log

Append-only `audit_event`: time, actor (`user`, `credential id + kind`, or `system`/job id), action (the operation's `x-permission` + operation id), target (type, id, name), `before/after` JSON diff with every field marked `writeOnly` in the spec redacted, IP, user agent, request id, outcome (`ok`, `denied`, `error`).
- Recorded for: all writes, all denied requests, logins (success and failure), pairing, credential creation/revocation, role changes, OIDC group-sync changes, settings reads of secret-bearing resources.
- Not recorded: reads (too noisy), SSE traffic.
- Retention default 180 days, pruned by a scheduled job. Read via `GET /api/v1/audit-events` (cursor-paged, `audit:read`), and exported as structured log lines (JSON) so a log shipper can forward it.

### 3.6 API conventions and code generation

**Base and versioning.**
- Everything under `/api/v1`. Within v1 only additive changes (new resources, optional fields, new enum values that clients must tolerate). A breaking change means `/api/v2` served alongside v1 for at least one minor release with `Deprecation` (RFC 9745) and `Sunset` (RFC 8594) headers on v1.
- `GET /api/v1/meta` (public): `serverId` (UUID, stable), `name`, `version`, `apiVersion` (semver of the spec), `minClientApiVersion`, enabled sign-in methods, whether setup is complete. Nothing else is public except `/livez`, `/readyz` and the login/pairing endpoints.

**Resources.**
- Plural kebab-case nouns: `/items`, `/items/{itemId}/slots`, `/requests`, `/download-clients`, `/indexers/{indexerId}/test`.
- IDs are UUIDv7 strings (sortable, opaque; nothing parses them).
- JSON, `camelCase` fields, RFC 3339 UTC timestamps, durations in seconds, sizes in bytes, enums as lowercase strings.
- **Actions are sub-resources**, not a command bus: `POST /items/{id}/search`, `POST /requests/{id}/approve`, `POST /indexers/{id}/test`. This replaces the Arr `POST /command {name:"SeriesSearch"}` pattern.
- Updates: `PATCH` with JSON Merge Patch (RFC 7396); `PUT` only for whole-document settings resources.
- Secrets are `writeOnly` in the schema: responses return `"apiKeySet": true`, never the value or a hash fragment (Jackett leaks 10 chars of its hash, `ServerConfig.cs:83`). Sending the field replaces it; omitting it keeps it.

**Lists.**
- Cursor pagination: `?limit=50&cursor=<opaque>`; response `{ "items": [...], "nextCursor": "…" | null }`. `limit` max 500. The cursor encodes the sort key and id and is tied to the filter set (a mismatched cursor is a 400). `?include=totalCount` is opt-in because counting is expensive on big tables.
- Filters are named query parameters per resource (`?kind=tv&status=missing&tag=4k`), repeated for OR (`?status=pending&status=approved`); range filters use `…Before`/`…After`. `sort=-createdAt,title`. No generic filter language in v1.
- Sparse expansion only where the UI needs it: `?expand=slots,files` (whitelisted per resource).

**Errors.** `application/problem+json` (RFC 9457): `type` (URI under `https://bonnyread.dev/problems/<code>`), `title`, `status`, `detail`, `instance` (request id), plus `code` (stable machine string, e.g. `quota_exceeded`, `precondition_failed`, `validation_failed`) and `errors[] {pointer, code, message}` for field validation. 401 vs 403 vs 404: a resource outside the caller's scope returns **404**, not 403, so scope does not leak existence.

**Concurrency.** Every single-resource GET returns `ETag: "<row version>"`. `PATCH`/`PUT`/`DELETE` accept `If-Match`; for settings and profile resources it is **required** (428 if missing, 412 on mismatch), because two admins editing a profile is the realistic conflict. `If-None-Match` on GET gives 304 for polling clients.

**Idempotency.** `Idempotency-Key` header (IETF httpapi draft semantics) accepted on every `POST` and required on `POST` operations marked `x-idempotent-required` (requests, grabs, job-starting actions) for API-key callers. Stored per (principal, key) for 24 hours with a hash of the body: same key + same body replays the stored response; same key + different body is 422. Works across HA nodes because it is a DB table.

**Long-running operations as jobs.** Actions that can take more than about a second return `202 Accepted`, `Location: /api/v1/jobs/{jobId}` and the job body. A job is `{id, type, state: queued|running|succeeded|failed|cancelled, progress {current,total,message}, result, error (problem), createdBy, target, createdAt, startedAt, finishedAt}`. `POST /jobs/{id}/cancel`. `GET /jobs?state=running&targetId=…`. Job state changes stream as SSE `job` events. The job rows are the D14 durable queue (runtime design); the API exposes them read-only apart from cancel, and filters them by the permission of the action that created them.

**Authorisation metadata in the spec.** Each operation carries:
```yaml
x-permission: library:execute
x-scope: { kind: item.kind, library: item.library }   # how to resolve target scope; omitted for unscoped resources
x-idempotent-required: true
```
The generator emits a route table from these; a CI lint (as Profilarr's `scripts/lint/api-permissions`) fails the build if any operation lacks `x-permission`. API-key permission pickers in the UI are generated from the same list, so they cannot drift.

**Toolchain.** The OpenAPI 3.1 document (`api/openapi/` split into files, bundled to `api/openapi.yaml`) is the source of truth (D6, D12).

| Piece | Choice | Why |
|---|---|---|
| Lint | Redocly CLI or vacuum with a ruleset for the conventions above | Enforces naming, problem responses, `x-permission` |
| Go server | **oapi-codegen v2**, `std-http-server` + `strict-server` on Go's `net/http` ServeMux | Spec-first, typed request/response objects per operation, OpenAPI 3.1 supported (pkg.go.dev FAQ). Strict server does not validate requests, so a **kin-openapi validation middleware** (oapi-codegen's `nethttp-middleware`) runs in front, on by default |
| Alternatives considered | **ogen**: generates validation and fast JSON, but server-side SSE is not supported ("only SSE client generation", ogen README) and generated code is large. **Huma v2**: excellent, has an SSE package, but is code-first, so the spec becomes an output of Go code, which inverts D6/D12's "generated from OpenAPI" and makes the TS client depend on the Go build | Re-evaluate at implementation start with a 3-endpoint spike |
| SSE endpoint | Hand-written handler, documented in the spec as `text/event-stream` with an `Event` schema (`oneOf` discriminated by `type`) | Generators do not produce SSE servers well; the event *payload types* are still generated |
| TS client (SvelteKit, Tauri) | **openapi-typescript** (types) + **openapi-fetch** (thin typed fetch) | Zero-runtime-cost types; Profilarr already uses openapi-typescript (`src/lib/api/v1.d.ts`) |
| Go client (CLI, tests) | oapi-codegen `client` | Same generator as server |
| Other SDKs | Not maintained in v1; the spec is published at `/api/v1/openapi.json` and with each release so users can run openapi-generator | Avoid maintaining SDKs nobody uses (D7 dropped the consumers) |
| Breaking-change gate | **oasdiff** `breaking` against the last release's spec in CI | Enforces the additive-only rule |

### 3.7 Live updates (SSE)

`GET /api/v1/events?topics=items,activity,jobs,health,requests,logs` — one stream per client (browser tabs share it, below).

- **Framing:** `id: <global sequence>`, `event: <type>`, `data: <JSON>`. Types: `resource.changed {resource, id, action: created|updated|deleted, version}`, `activity.progress {downloadId, itemIds, progress, eta, speed}`, `job {id, state, progress}`, `health {checks}`, `log {…}` (admins), `request {id, status}`, plus control events `reset` and `principal.changed`.
- **Notify, then fetch.** Change events carry ids and versions, not full resources; the UI invalidates its query cache (TanStack Svelte Query) and refetches with ETags. This keeps events small (Postgres `NOTIFY` payloads are limited to 8,000 bytes) and means permission-filtered *fields* never leak through events. Only `activity.progress` carries data inline, throttled to 1 update per second per download and coalesced.
- **Heartbeat** comment every 15 s; `retry: 3000` hint.
- **Resume:** the client reconnects with `Last-Event-ID`. The server replays from the event log if the id is still retained (target: at least 10 minutes or 50,000 events); otherwise it sends `reset`, and the client refetches everything visible. Sequence ids are cluster-global, so reconnecting to a different HA node works.
- **Per-permission filtering:** every event is published with `{permission, kind?, library?, ownerId?}`. Each connection evaluates it against the principal's cached effective permissions (same code as §3.4). On `principal_version` change the server sends `principal.changed` and closes; the client reconnects with fresh rights.
- **Browser limits:** over HTTP/1.1 browsers allow about six connections per origin, so the SPA opens the stream in a `SharedWorker` (fallback: leader tab via `BroadcastChannel`). The recommended reverse-proxy config serves HTTP/2.
- **Native clients:** browser `EventSource` cannot send an `Authorization` header, and query credentials are banned (I2). Browsers use the cookie; Tauri clients use a fetch-stream SSE reader through the native transport (§3.8), which sends the bearer token.
- **Proxies:** responses set `Cache-Control: no-cache`, `X-Accel-Buffering: no`; the Helm ingress sets read timeouts to 1 hour; nodes send a final `retry` and close streams on shutdown so clients move to other nodes during rolling updates.

**Dependency on the runtime design (event bus).** This API needs from it: (1) a cluster-wide monotonic event sequence; (2) a replay window (an outbox/event table, not only `LISTEN/NOTIFY`, which drops messages for disconnected listeners); (3) ACL metadata on every event; (4) cross-node fan-out (`LISTEN/NOTIFY` per D14, in-process on SQLite); (5) a `principal_version` broadcast.

### 3.8 Clients

**Web SPA (D6).** SvelteKit static adapter, embedded with `go:embed`, served at `/` with SPA fallback. Sub-path hosting (`/bonnyread/`) is a runtime setting: the build uses relative asset paths and the core injects `<base href>` and a `window.__BR_BASE__` into `index.html` at serve time, so one build works at any path. The SPA never sees an API key; it authenticates with the session cookie.

**One UI, two transports.** The UI calls the API through an interface with two implementations:
- *browser*: same-origin `fetch` + cookie + native `EventSource`;
- *native* (Tauri desktop/mobile): a Rust-side HTTP client exposed as Tauri commands (or `tauri-plugin-http`, which is reqwest-based), adding `Authorization: Bearer brd_…`, doing **certificate pinning** for self-signed servers, and streaming SSE. Keeping HTTP in Rust avoids CORS, keeps the device secret out of the webview, and lets the app trust a pinned self-signed certificate that a webview would reject.

**Tauri desktop (D12, D18).**
- Bundles the same SvelteKit build plus a small native shell: server list, pairing, tray icon (replaces the tray apps of six forks, Jackett and Unpackerr), native notifications, deep links (`bonnyread://`), "Open in browser".
- Device secrets are stored in the **OS keychain** (Keychain, Credential Manager, Secret Service), via a keyring plugin. Not Stronghold: Tauri documents it as no longer recommended and slated for removal in v3 (v2.tauri.app/plugin/stronghold).
- **Version skew:** because the self-updater is deferred (D16), a desktop app may connect to an older or newer remote server. The app checks `/meta.apiVersion`: same major and server ≥ app's required minor → use the bundled UI; otherwise it shows "this server runs version X", and offers to open the server's own web UI in the browser. The app never loads remote UI code into its privileged webview.

**Tauri mobile (later).** Same UI and native transport. Pairing by QR (`tauri-plugin-barcode-scanner`) or deep link (`tauri-plugin-deep-link`), OIDC by the system browser with PKCE and an app-claimed redirect (RFC 8252). iOS App Transport Security blocks plain HTTP, so mobile requires HTTPS (public CA or the pinned self-signed certificate). SSE runs only in the foreground; on resume the app reconnects with `Last-Event-ID`. Push notifications while backgrounded need APNs/FCM relays, which conflicts with "no hosted infrastructure" (§6).

**Server discovery and selection.**
- A *server profile* is `{serverId, name, urls: [ordered], certFingerprint?, credentialRef, lastSeen}`. The app can hold many profiles and switch between them (Settings → Servers), like a mail client with several accounts.
- Candidate discovery: (1) **local** — the installed service writes `<data>/run/discovery.json` (`serverId`, port, TLS fingerprint) readable by local users, and the app probes `127.0.0.1:<port>/api/v1/meta`; (2) **LAN** — optional mDNS/DNS-SD `_bonnyread._tcp` advertisement (off until LAN access is enabled); (3) **manual URL**; (4) **QR/deep link** carrying URLs, `serverId` and fingerprint.
- Connection: try URLs in order (loopback, LAN, remote), 2 s timeout each, and **verify `serverId`** from `/meta` matches the profile before sending the credential; a mismatch (different server behind the same address) stops and warns. With a pinned fingerprint, a TLS mismatch is fatal, not a prompt.

**Offline and degraded behaviour.**
- Unreachable server: banner with retry (exponential backoff to 30 s) and the last successfully fetched views shown read-only from the query cache (memory plus a small persisted snapshot of the dashboard, calendar and queue). No offline writes: mutations are disabled, never queued, so there is nothing to reconcile.
- Local service stopped (desktop): the app offers *Start service* (the installer grants the installing user start/stop rights on the service) and shows the tail of the service log.
- Partial failure (a DB or leader problem): `/readyz` false on a node sends clients elsewhere via the load balancer; the UI shows health events from `/health`.

### 3.9 Local pairing and bootstrap (D12 in detail)

One mechanism solves three problems: first-run ownership (I6), the desktop app's credential, and adding devices later.

**Pairing secrets.** A pairing secret is 256 bits, single-use, expires after 10 minutes, and is stored hashed in the DB with its *purpose*:
- `bootstrap` — create the first Owner (exists only while no Owner exists);
- `local` — issue a device credential for an existing user, only over loopback;
- `remote` — issue a device credential for the user who created it, from anywhere.

**Local handshake (same machine).**
1. The core creates a `bootstrap` secret on first start (and a `local` one on demand: `bonnyread pair`, or the tray's "Pair this app"). It writes the plaintext to `<data>/pairing/<id>.token` and logs "pairing token written to …" (the value itself is printed to the console only for containers, see §3.10).
2. File access is the proof of locality. The `pairing/` directory is readable only by the service account and by the **installing user** (Windows ACL on the user's SID; a `bonnyread` group on Linux/macOS that the installer adds the user to). On a shared PC other local users cannot pair.
3. The app reads the token and calls `POST /api/v1/pairing/claim {token, device: {name, platform, appVersion}}` on `127.0.0.1`. The server accepts `local`/`bootstrap` claims only when the TCP peer is loopback **and** the `Host` header is `localhost`/`127.0.0.1`/`[::1]` (DNS-rebinding guard) **and** no forwarding headers are present (so a local reverse proxy cannot relay a remote claim).
4. For `bootstrap`, the response is `{step: "create-owner"}`; the app shows *Create owner account* (username, password, optional TOTP) and calls `POST /api/v1/pairing/complete`. The Owner is created, the secret is consumed, and a device credential `brd_…` is returned.
5. The app stores the credential in the OS keychain, deletes the token file, and connects. The device appears under *Devices* with its name and platform.

**Remote pairing (phone, second computer).**
1. A signed-in user opens *Devices → Pair a device*, optionally capping it (e.g. read-only).
2. The server creates a `remote` secret bound to that user and cap and shows a QR code and a 10-character code:
   `bonnyread://pair?s=<serverId>&u=<url1>,<url2>&fp=<sha256 SPKI, if self-signed>&c=<secret>`
3. The phone scans it, tries the URLs, checks `serverId` and the fingerprint, and calls `POST /pairing/claim {code, device}`. The server returns a device credential whose permissions are the user's (or the cap). The originating browser gets an SSE event and shows "Pixel 9 paired".
4. Without a camera: enter the URL, then sign in normally (password, OIDC via the system browser, Plex/Jellyfin); a successful login from a native app also yields a device credential.

**Why a device credential instead of an API key** (D12 says "exchanged for a scoped API key"): they are the same table and mechanism (§3.3); a device just has a sliding lifetime, is named after the device, appears on the *Devices* page rather than *API keys*, and carries the user's full rights unless capped. See §6.

### 3.10 Deployment topologies

```
A. Single-user desktop (default for entry-level users)
   ┌──────────── one machine ─────────────────────────────┐
   │  Tauri app  ──http://127.0.0.1:7979──►  bonnyread    │
   │  (UI, tray, keychain)                   (OS service, │
   │                                          SQLite,     │
   │                                          <data>/)    │
   └──────────────────────────────────────────────────────┘
   optional: "Allow access from other devices" → binds 0.0.0.0, enables TLS
   (self-signed + pinning) and mDNS; requires Owner password + TOTP.

B. Home server / NAS (container or package)
   browser / Tauri / phone ──HTTPS──► reverse proxy (Caddy, Traefik, NPM) ──► bonnyread:7979
                                                                              SQLite or Postgres
                                                                              /data, /media, /downloads

C. Kubernetes HA (D14, D18)
   clients ──HTTPS──► Ingress ──► Service ──► Deployment (N replicas, stateless)
                                               │   ├─ leader-elected singletons via DB lease
                                               │   └─ admin listener :9979 (/metrics, /livez, /readyz)
                                               ├─► PostgreSQL (CloudNativePG or managed)
                                               └─► RWX PVCs: /media, /downloads (same mount paths on every pod)
   Prometheus ──ServiceMonitor──► :9979/metrics (bearer token of a Service account)
```

**Ports.** 7979 for HTTP/API/UI (no reference app uses it: Sonarr 8989, Radarr 7878, Lidarr 8686, Readarr/Bookshelf 8787, Prowlarr 9696, Seerr 5055, Bazarr 6767, autobrr 7474, Profilarr 6868, Unpackerr and Kapowarr 5656, Jackett 9117, Cleanuparr 11011, Whisparr 6969). Optional **admin listener** (default off single-node, on in Helm) at 9979 serving `/metrics`, `/livez`, `/readyz` and `/debug/pprof` (opt-in), so the ingress never exposes them.

**Bootstrap configuration** (not in the DB, because it is needed to reach the DB): `<data>/config.toml`, overridable by `BONNYREAD_*` env vars and `BONNYREAD_*_FILE` (for Kubernetes/Docker secrets): listen addresses, base path, DB driver + DSN, TLS mode (`off | self-signed | files`), trusted proxy CIDRs (for `X-Forwarded-*` and client IPs; empty by default), log level/format, argon2 parameters, optional bootstrap owner (`BONNYREAD_BOOTSTRAP_OWNER_OIDC_GROUP` or username + password-file) for unattended installs. Everything else is in the DB (D16).

**Data directory layout** (single node; in HA only `cache/` is node-local):
```
<data>/                      Linux: /var/lib/bonnyread   macOS: /Library/Application Support/bonnyread
                             Windows: %ProgramData%\bonnyread   container: /data
  config.toml
  bonnyread.db (+ -wal, -shm)      SQLite only
  pairing/        0750, service account + installing user/group
  run/discovery.json               serverId, port, TLS fingerprint (world-readable, no secrets)
  tls/            self-signed cert + key (0600)
  definitions/    Cardigann definitions, downloaded at runtime (D5b)
  profiles/       PCD git clones (D9)
  plugins/        WebAssembly plug-ins (D15)
  cache/          images, metadata cache (disposable)
  backups/        DB backups, incl. automatic pre-migration snapshot
  logs/
```

**Upgrades** (self-updater deferred, D16): the package manager, installer, container tag or Helm release replaces the binary. On start, single-node cores take a SQLite snapshot (`VACUUM INTO backups/pre-<version>.db`) and run Atlas migrations; in HA, migrations run once in a Helm hook Job, and pods refuse readiness if the DB schema is newer than they understand. Migrations must be expand/contract so N and N+1 pods can overlap during a rolling update (runtime/DB design to confirm).

### 3.11 Container image

- Multi-stage build: SvelteKit build (Node) → Go build with `CGO_ENABLED=0` (embeds the SPA) → final image.
- Final base: `gcr.io/distroless/static-debian12:nonroot` (or Chainguard static) plus a statically linked **ffprobe** (media info, as the Arrs use) and CA certificates. No shell, no package manager. RAR extraction is pure Go (xtractr/rardecode, [unpackerr.md](../analysis/unpackerr.md)); 7-Zip support should also be pure Go so no binaries are added. The SQLite driver must therefore be pure Go (modernc or a WASM build); D13's ent/Atlas owner confirms the choice.
- Runs as UID/GID 65532, works under any arbitrary UID (OpenShift); **no PUID/PGID/UMASK root entrypoint** (Profilarr, Kapowarr, Cleanuparr do this; it forces the container to start as root). Users set `user:` in compose or `runAsUser`/`fsGroup` in Kubernetes; the `umask` for created media files is an app setting.
- Read-only root filesystem; writable `/data` and `/tmp`. `EXPOSE 7979 9979`. `HEALTHCHECK CMD ["/bonnyread","healthcheck"]` (no curl in distroless).
- First start in a container with no Owner prints a one-line setup URL containing the bootstrap secret to stdout (`https://<host>/setup#t=…`), which the operator opens once; or the bootstrap owner is supplied via env/secret.
- Multi-arch `linux/amd64`, `linux/arm64` (and `arm/v7` if the pure-Go stack allows). Images signed with cosign, SBOM and SLSA provenance attached; `go-licenses` report included (D5).

### 3.12 Helm chart shape

```
charts/bonnyread/
  values.yaml
  templates/
    deployment.yaml         replicas, probes, securityContext, volumes, topologySpread
    service.yaml            http 7979, admin 9979
    ingress.yaml            SSE-friendly annotations
    migrate-job.yaml        helm.sh/hook: pre-install,pre-upgrade → `bonnyread migrate up`
    servicemonitor.yaml     if metrics.serviceMonitor.enabled
    pdb.yaml                if replicaCount > 1
    networkpolicy.yaml      optional
    secret.yaml             only if not using existingSecret
    NOTES.txt               how to get the setup URL / bootstrap owner
```

Key values and rules:
- `replicaCount: 1`. The chart **fails** (`fail` in template) if `replicaCount > 1` and `database.driver != postgres` (D14).
- `database`: `driver`, `existingSecret` with `dsn` key; optional CloudNativePG `Cluster` via a subchart flag.
- `persistence.data`: RWO PVC for single-replica SQLite installs; for HA the data dir is an `emptyDir` cache (everything durable is in Postgres, see §6 on definitions and PCD clones).
- `media.volumes[]`: `{name, existingClaim, mountPath}`; RWX; identical `mountPath` on every pod (D14).
- `auth`: `oidc.providers[]` (issuer, clientId, `clientSecretRef`, groups claim, group→role rules) seeded at install and then editable in the UI; `bootstrapOwner` (OIDC group or username + password secret).
- Probes: `startupProbe` `/livez` (tolerates slow first boot), `livenessProbe` `/livez` (process responsive), `readinessProbe` `/readyz` (DB reachable, schema compatible, not draining). `terminationGracePeriodSeconds: 30` with a `preStop` sleep of 5 s so the load balancer stops sending before SSE streams are closed.
- **Leader election uses database leases (D14), so the chart needs no Kubernetes RBAC for `Lease` objects**; the ServiceAccount has no permissions.
- Sessions, idempotency keys, rate-limit counters and SSE replay are all in the DB: **no sticky sessions**.
- Ingress defaults: `nginx.ingress.kubernetes.io/proxy-buffering: "off"`, `proxy-read-timeout: "3600"`, and the equivalents for Traefik documented.
- `metrics.serviceMonitor`: `enabled`, `interval`, `bearerTokenSecret` (an API key of a *Metrics scraper* service account created by `bonnyread admin create-service-account --preset metrics` in a post-install hook, or supplied).
- Pod security: `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, drop ALL capabilities, `seccompProfile: RuntimeDefault`.
- `trustedProxies`: the ingress controller's pod CIDR, so client IPs in audit and rate limits are real.

### 3.13 OS service installation

**One implementation in the core:** `bonnyread service install|uninstall|start|stop|status [--data DIR] [--user ACCOUNT] [--grant-pairing USER]` built on `kardianos/service` (Windows SCM, systemd, launchd; maintained, last release July 2025). Every installer calls this rather than writing unit files itself.

| OS | Package (Tauri 2 bundler) | Service |
|---|---|---|
| Windows | NSIS (per-machine) with `NSIS_HOOK_POSTINSTALL` / `NSIS_HOOK_PREUNINSTALL` running `bonnyread.exe service install/uninstall`; MSI alternative via WiX fragments (both documented at tauri.app/distribute/windows-installer) | Windows service under a virtual account `NT SERVICE\bonnyread` (not LocalSystem); data in `%ProgramData%\bonnyread`; installing user's SID gets read on `pairing\` and start/stop rights; no firewall rule unless LAN access is enabled |
| macOS | Tauri produces `.app`/`.dmg` only, and a dragged DMG cannot run install scripts. Two paths: (a) default **per-user LaunchAgent** registered by the app on first launch via `SMAppService.agent` (macOS 13+, no admin, runs while the user is logged in); (b) **system LaunchDaemon** via a separate signed `.pkg` built with `pkgbuild` (or `SMAppService.daemon` with user approval in System Settings) for always-on servers | Agent: data in `~/Library/Application Support/bonnyread`; daemon: `/Library/Application Support/bonnyread`, dedicated `_bonnyread` user. Both require a Developer ID–signed, notarised app (ad-hoc signatures cannot register with SMAppService) |
| Linux | `.deb` and `.rpm` from Tauri with `postInstallScript`/`preRemoveScript` (documented for RPM; Debian equivalent) creating the `bonnyread` system user and group, adding the installing user (`$SUDO_USER`/`$PKEXEC_UID`) to the group and running `service install`; AppImage cannot install system services, so it offers a **systemd user unit** (`systemctl --user`, with `loginctl enable-linger` suggested) | systemd unit with `NoNewPrivileges`, `ProtectSystem=full`, `ProtectHome=read-only` (media paths under `/home` need a drop-in `ReadWritePaths=`, which the settings UI explains), `PrivateTmp`, `Restart=on-failure` |
| Headless (no desktop app) | goreleaser + nfpm: `.deb`, `.rpm`, `.apk`, tarballs, a Windows zip, and a Homebrew formula; same `service install` | Same as above; pairing from another machine via the container-style setup URL |

The desktop bundle ships the core binary as a **resource installed as a service**, not a Tauri sidecar started by the app: the server must keep running when the app is closed or the user logs out (system-service variants). Because both are installed together, desktop app and local core are always the same version.

---

## 4. Walkthroughs

### 4.1 First run on a desktop (Windows)

1. The user runs `bonnyread-setup.exe` (NSIS, per-machine, UAC prompt).
2. `NSIS_HOOK_POSTINSTALL` runs `bonnyread.exe service install --data "%ProgramData%\bonnyread" --grant-pairing <installing user SID>` and starts the service.
3. The core starts: no `config.toml` → writes defaults (listen `127.0.0.1:7979`, SQLite, TLS off because loopback only). Creates `bonnyread.db`, runs migrations, writes `run/discovery.json`. No Owner exists → creates a `bootstrap` pairing secret, writes `pairing/01J….token` (ACL: service account + installing user).
4. The installer launches the Tauri app. The app reads `discovery.json`, probes `127.0.0.1:7979/api/v1/meta` (`setupComplete: false`), reads the token file and calls `POST /pairing/claim` from loopback with `Host: 127.0.0.1:7979`.
5. The server validates (loopback, Host, no forwarding headers, token hash, unexpired, purpose `bootstrap`) and answers `create-owner`. The app shows the owner form; the user enters a username and password (TOTP offered, optional while loopback-only).
6. `POST /pairing/complete` creates the user with role **Owner**, scope all kinds, consumes the secret, writes audit events `pairing.claimed` and `user.created`, and returns `brd_…`. The app stores it in Windows Credential Manager, deletes the token file, opens the SSE stream and shows the setup checklist (media server, indexers, download client, root folders).
7. Later the user enables *Allow access from other devices*: the server requires TOTP for the Owner, generates a self-signed certificate, binds `0.0.0.0:7979` with TLS, adds a firewall rule (the service asks the tray app to trigger an elevated helper), and starts mDNS.

### 4.2 OIDC login that maps a group to Manager

Setup (Owner): *Settings → Authentication → OIDC → Add provider*: issuer `https://auth.example.com/application/o/bonnyread/`, client id and secret, groups claim `groups`, rules: `media-admins → Admin`, `media-curators → Manager`, `family → Member (kinds: tv, movie)`; default role none; JIT provisioning on.

1. Alice clicks *Sign in with Authentik*. The core creates `state`, `nonce` and a PKCE verifier (stored in a short-lived signed cookie) and redirects to the discovered `authorization_endpoint`.
2. Authentik redirects back with a code. The core exchanges it, validates the ID token (issuer, `aud`, `exp`, signature by JWKS, `nonce`), reads `sub`, `preferred_username=alice`, `email`, `groups=["family","media-curators"]`.
3. Lookup by identity `(oidc:https://auth.example.com/…, sub)`: none. JIT on → create user `alice` with identity row.
4. Group sync: rules matched → assignments `Manager (all non-adult kinds)` and `Member (tv, movie)`, source `oidc_group`. Effective role is the more privileged (Manager); scopes union to all non-adult kinds.
5. Session cookie issued; audit `auth.login {method: oidc}` and `role.assigned {source: oidc_group}`.
6. A month later Alice is removed from `media-curators` in Authentik. At her next login sync removes the Manager assignment; `principal_version` bumps, her other sessions keep working but with Member rights from the next request, and her open SSE stream is closed with `principal.changed`. If an admin needs it immediately, *Users → Alice → Revoke sessions*.

### 4.3 A requester with a quota requests a series

Context: Bob signed in with Plex (server policy "anyone with access", default role Requester, scope non-adult). Global TV quota for Requesters: 10 seasons per 7 days; Bob has used 7. Slots: `1080p` (requestable by Requester+, auto-approve for Member+), `4K` (requestable by Manager+).

1. Bob searches discovery (`GET /api/v1/discover/search?q=severance&kind=tv`, permission `discover:read`, scope filter removes adult results).
2. He opens the show; the UI calls `GET /api/v1/users/me/quota` → `tv: {limit: 10, days: 7, used: 7, remaining: 3, resetsAt: …}` and shows the available slots (only `1080p`).
3. He selects seasons 1–2 and submits `POST /api/v1/requests` with `Idempotency-Key` (the SPA sets one per submit, so a double click cannot create two requests): `{kind: tv, externalIds: {tvdb: …}, seasons: [1,2], slot: "1080p"}`.
4. Server checks in order: `request:create` in scope `tv` → slot requestable by Requester → quota (7 + 2 ≤ 10) → blocklist → duplicates/already available seasons → creates request `pending` (not auto-approved for Requester). Audit `request.created`. SSE `request` event to Bob (own) and to Managers (`request:approve`).
5. Carol (Manager) approves via `POST /requests/{id}/approve`; she could also change the slot or tick `bypassQuota`. The request becomes `approved`; the domain layer adds or monitors the series and starts a search, returned as a job (`202`, `Location: /jobs/…`).
6. If Bob had asked for 4 seasons, step 4 fails with `403 problem {code: quota_exceeded, detail: "3 seasons left until …"}`. Declined requests free the quota; completed ones keep counting until the window passes.

### 4.4 A phone app connecting to a remote HA cluster

Context: bonnyread runs on Kubernetes (3 replicas, Postgres, ingress `https://media.example.com` with a public certificate). Dana has the Member role.

1. Dana installs the mobile app, taps *Add server*, types `media.example.com`. The app calls `GET https://media.example.com/api/v1/meta` → `serverId`, `apiVersion 1.4.0`, sign-in methods `[oidc: Authentik, password]`.
2. She chooses Authentik. The app opens the system browser to `/api/v1/auth/oidc/authentik/start?client=native&redirect=bonnyread://auth` with PKCE; after login the core redirects to `bonnyread://auth?code=<one-time exchange code>`; the app exchanges it with its PKCE verifier at `POST /api/v1/auth/native/exchange` and receives a device credential `brd_…` (device name "Dana's iPhone"). Stored in the iOS Keychain.
   (Alternative: on her laptop she opens *Devices → Pair a device* and scans the QR code; the result is the same device credential.)
3. The app saves the server profile `{serverId, urls: ["https://media.example.com"]}` and opens `GET /api/v1/events?topics=items,activity,requests` through the native transport with `Authorization: Bearer`. The ingress routes it to pod 2.
4. Pod 2 is drained during a rolling update: it sends `retry` and closes. The app reconnects with `Last-Event-ID: 88412`; pod 3 replays events after 88412 from the shared event log, filtered by Dana's permissions (no adult items, no indexer events).
5. Dana locks the phone; iOS suspends the app and the stream. Hours later she reopens it: `Last-Event-ID` is outside the replay window → `reset` → the app refetches the visible screens.
6. If the admin revokes the device, the next request returns 401 `credential_revoked` and the app returns to the sign-in screen for that server; her other server profiles are unaffected.

---

## 5. Simplifications versus the references

| Reference behaviour | bonnyread | Why it is safe |
|---|---|---|
| 16 separate login systems and inter-app API keys | One user table, one authoriser | The apps are one process (D12) |
| Auth modes None / Basic / Forms / External / DisabledForLocalAddresses / TrustedNetworks / CGNAT trust (Sonarr, Prowlarr, Cleanuparr) | Password, OIDC, media-server login; locality proven by the pairing file | Pairing gives local convenience without an unauthenticated mode (I1) |
| Seerr 30-flag bitmask with 4K and per-kind duplicates | ~20 resources × 7 actions, 3 scope dimensions, 7 default roles; 4K becomes a slot setting | D4 removes 4K duplication; scopes remove per-kind multiplication |
| Global admin API key + `X-API-User` impersonation (Seerr) | Per-user API keys narrowed to a subset | Automation acts as a real principal and is audited |
| API keys in query strings (Arrs, Seerr, Bazarr, Kapowarr) | Header only | No compatibility façades (D7), so nothing needs query keys |
| API key injected into the SPA (Sonarr `/initialize.json`, Bazarr) | SPA uses the session cookie only | Nothing to leak from page source |
| SignalR hubs + Bazarr's SignalR clients + Unpackerr WebSocket + autobrr SSE | One SSE stream with resume and ACL filtering | Server-to-client only is all the UI needs; inter-app consumers are gone |
| `POST /command {name}` + separate command resource, `/api/v3` and `/api/v5` coexisting | Action sub-resources returning jobs; one `/api/v1` | Typed, documented per action; one version |
| Page/pageSize paging with surprising defaults (Seerr sees 10 queue items, Unpackerr sets 2,000) | Cursor paging with explicit `limit` and `nextCursor` | Stable under inserts; no silent truncation |
| Tray apps, Windows service wrappers, Inno Setup, firewall COM, `RegisterUrl` in 8 codebases; Unpackerr systray + zenity | Core `service` subcommand + one Tauri app | One implementation per OS |
| Self-updaters in forks, Bazarr, Jackett, Unpackerr | None (D16); packages, images and Helm upgrade | Deferred by decision |
| PUID/PGID root entrypoints (Profilarr, Kapowarr, Cleanuparr) | Non-root image, arbitrary UID | Standard container practice; no root start |
| Separate metrics servers with their own basic auth (autobrr) | `/metrics` on the admin listener with a Service-account key | Same authoriser |
| Sentry and GA4 telemetry | None | I5 |
| Seerr `userType` enum and per-provider token columns | Identity rows per provider | Multiple logins per user; no special cases |

---

## 6. Issues with decisions, and open questions

### Issues with decisions

1. **D12 — "exchanged for a scoped API key".** A long-lived API key is the wrong credential for an interactive app: it does not expire on inactivity, does not show up as a device, and invites copy-paste. This design keeps D12's mechanism (localhost discovery + one-time token in the data directory) but issues a **device credential**, stored in the same credential table as API keys (§3.3, §3.9). Please confirm this reading of D12.
2. **D12 — "one-time token in the data directory" on multi-user machines.** A system service's data directory is not readable by the desktop user, and if it were, any local user could claim the server. The design restricts `pairing/` to the service account plus the installing user (Windows ACL, Unix group). Alternative with no file: pairing over a local socket or named pipe with OS peer credentials (`SO_PEERCRED`, named-pipe client identity), which is stronger but differs from D12's wording.
3. **D12/D16 — bundled UI versus version skew.** D12 has the Tauri apps bundle the UI; D16 defers the self-updater. A desktop or mobile app will meet servers of other versions. The design adds `apiVersion`/`minClientApiVersion` negotiation and a fallback to the server's own web UI (§3.8). This makes the additive-only v1 rule (§3.6, enforced by oasdiff) a hard requirement.
4. **D3c ("no hosted infrastructure") versus mobile push.** Background notifications on iOS/Android need APNs/FCM, which need a project-run relay holding the push credentials. Options: no push in v1 (foreground SSE only, plus existing notification targets such as ntfy/Pushover/Telegram), UnifiedPush on Android only, or a relay later behind an interface. Recommendation: no push in v1.
5. **D16 (secrets in the DB, encryption is the DB's job) versus the pairing token and bootstrap config.** The DB DSN, TLS key and OIDC seed secrets cannot live in the DB they unlock. The design adds `config.toml` + env + `*_FILE` for these bootstrap values only (§3.10). Separately, D16 means a stolen SQLite backup exposes third-party credentials and Plex tokens; consider optional envelope encryption of `writeOnly` fields with a key outside the DB (open question, not a deviation).
6. **D5b/D9 (definitions and PCD git clones "in the data directory") versus D14 (stateless nodes).** In HA the data directory is per pod. Either the leader materialises definitions and PCD content into Postgres and nodes cache locally (recommended), or the chart needs an RWX data volume. The runtime design should settle this; the Helm chart above assumes the first.
7. **D17 does not mention trusted-header (forward-auth) login.** Not proposed as a default (it bypasses I1 if misconfigured), but Authelia/Authentik forward-auth users will expect it. Proposal if wanted: off by default, only from `trustedProxies`, maps a header to an *existing* user or to a group rule, never creates Owners.

### Open questions

- Passkeys (WebAuthn) as a first-class login and second factor in v1, or later?
- Default quotas for Requester (proposed 10 movies + 10 seasons per 7 days) and whether music/books/comics count units the same way (domain designs).
- Should TOTP be mandatory for Owner/Admin whenever the server is not loopback-only (proposed yes)?
- macOS default: per-user LaunchAgent (no admin, stops on logout) versus system LaunchDaemon via `.pkg` (always on). Proposed: agent by default, daemon for "server mode".
- OIDC back-channel logout: needed for v1?
- API-key default expiry of one year: acceptable for automation users, or default to none with a UI warning?
- SSE replay window size (10 minutes / 50,000 events proposed) must be sized with the runtime design's event table.
- oapi-codegen strict server versus Huma versus ogen: confirm with a 3-endpoint spike (spec-first, validation, SSE doc, TS client) before committing.
- Pure-Go SQLite driver compatible with ent/Atlas (D13) so the image can stay `CGO_ENABLED=0`.

### Sources

- oapi-codegen v2 (strict server, OpenAPI 3.1 FAQ): https://pkg.go.dev/github.com/oapi-codegen/oapi-codegen/v2
- ogen SSE limitation: https://github.com/ogen-go/ogen (README, "Only SSE client generation is supported")
- Huma SSE and OpenAPI generation: https://github.com/danielgtaylor/huma/blob/main/docs/docs/features/server-sent-events-sse.md
- Tauri Windows installer hooks and WiX fragments: https://tauri.app/distribute/windows-installer
- Tauri RPM scripts: https://tauri.app/distribute/rpm ; DMG/app bundle: https://v2.tauri.app/distribute/dmg/
- Tauri plugins (deep-link, barcode-scanner, http): https://github.com/tauri-apps/plugins-workspace
- Stronghold deprecation: https://v2.tauri.app/plugin/stronghold/ ; keychain alternative: https://docs.rs/tauri-plugin-keyring-store/latest/tauri_plugin_keyring_store/
- SMAppService daemon and pkg installs: https://developer.apple.com/forums/thread/763035
- kardianos/service: https://github.com/kardianos/service
- RFC 9457 (problem details), RFC 7396 (merge patch), RFC 9745 (Deprecation header), RFC 8594 (Sunset), RFC 8252 (OAuth for native apps), RFC 6238 (TOTP), RFC 9106 (Argon2)
