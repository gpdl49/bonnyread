# 10. Identity and Access

This file specifies who can use a bonnyread server and what they may do. It covers users (people and service accounts) and their linked identities, the sign-in methods (built-in password with TOTP, OIDC, Plex/Jellyfin/Emby, and opt-in forward-auth), the single credential table (sessions, device credentials, API keys), the permission model and default roles, adult-content opt-in, per-slot request rights, request quotas, the audit log, login throttling and the secure defaults. The API surface is in `11-api-and-clients.md`; pairing flows that issue device credentials are in `11-api-and-clients.md` §Pairing. Applies to **P1** (all of it). Media kinds added in P2/P3 (music, book, adult, comic) join the existing scope dimension without schema change.

## Concepts

| Term | Definition |
|---|---|
| Principal | The authenticated subject of a request: a **User** plus the **Credential** it presented. |
| User | One row per person or automation. `kind` is `human` or `service`. A service account cannot sign in interactively and owns only API keys. |
| Identity | A linked login method of a user: `password`, `totp`, `oidc:<issuer>`, `plex`, `jellyfin`, `emby`, `forward_auth`, `local_os`. A user may have several. Replaces per-provider columns and user types. |
| Credential | A secret presented on a request. Kinds: `session` (browser cookie), `device` (native app), `api_key` (automation). All resolve through the same authoriser. |
| Device credential | A credential issued to a native app by pairing or by a native sign-in. Named after the device, sliding lifetime, revocable (A15). |
| Permission | A pair `resource:action`, optionally with a facet: `resource:action:facet` (for example `library:execute:search`). |
| Scope | A narrowing attached to a role assignment. Three dimensions: **media kind**, **library group**, **ownership** (`own`/`any`). |
| Library group | A named set of RootFolders (RootFolder is owned by `02-domain-model.md`). The design calls this scope dimension "library". |
| Role | A named set of permissions. Seven built-in roles; custom roles may be cloned from them. |
| Role assignment | `(user, role, scope, source)`. Source is `manual`, `oidc_group`, `forward_auth_group` or `media_server_default`. |
| Request | An **Intent** with `origin = user_request` (`02-domain-model.md`). "Request" is used for the user-facing concept and API resource. |
| Loopback-only | The server's HTTP listeners bind only loopback addresses, `TRUSTED_PROXIES` is empty and forward-auth is disabled. A local reverse proxy therefore makes a server *not* loopback-only. |
| Owner, Admin, Manager, Member, Requester, Viewer, Service | The built-in roles (§Default roles). |

## Requirements

### Secure defaults

**IAM-001** (MUST) Authentication is always on. There **MUST NOT** be an unauthenticated mode, a "disabled for local addresses" mode, a trusted-networks bypass or a CGNAT trust setting. Every endpoint except those listed in API-006 requires a valid credential. _Source: design/identity-api-deployment.md §1 I1._

**IAM-002** (MUST) The first Owner **MUST NOT** be claimable by whoever reaches the port first. The first Owner is created only through the bootstrap paths in `11-api-and-clients.md` (local socket, setup URL, or bootstrap-owner configuration). _Source: id §1 I6, §3.9; D20._

**IAM-003** (MUST) Credentials are accepted only in headers or cookies. A request carrying a credential in the query string (`apikey`, `api_key`, `access_token`, `token` or any parameter matching a credential prefix `brd_`/`brk_`) **MUST** be rejected with 400 and problem code `query_credential_rejected`, including on `/metrics` and the SSE endpoint. _Source: id §1 I2, §3.3._

**IAM-004** (MUST) Outbound TLS certificate verification is always on. Per-connection trust is expressed as a pinned certificate (SHA-256 of the SPKI) or a custom CA bundle on that connection; there is no "verify off" option. This applies to every outbound client (indexers, download clients, media servers, OIDC providers, metadata sources, notification targets, plug-in HTTP). _Source: id §1 I4._

**IAM-005** (MUST) bonnyread sends no telemetry, analytics or crash reports, and the binary embeds no third-party secrets other than project-registered public API keys (D3b, D16). _Source: id §1 I5._

**IAM-006** (MUST) Secrets are never returned to clients after they are saved. Secret fields are `writeOnly` (API-015) and are redacted in logs, audit diffs and sanitised exports. _Source: id §1 O12._

**IAM-007** (MUST) One authoriser handles every entry point: browser sessions, device credentials, API keys, forward-auth requests, the SSE stream, the admin listener and the CLI. No code outside the authoriser checks roles (IAM-060). _Source: id §1 I7._

**IAM-008** (MUST) CORS is closed by default. The only allowed cross origins are the Tauri app origins, and only for bearer-authenticated requests.

**IAM-009** (MUST) Unsafe methods (`POST`, `PUT`, `PATCH`, `DELETE`) authenticated by a session cookie **MUST** pass a same-origin check on `Origin` or `Sec-Fetch-Site`; a request with neither header is rejected with 403 `csrf_rejected`. Bearer-authenticated requests are exempt. _Source: id §3.3._

### Users and identities

**IAM-010** (MUST) All principals live in one user table with `kind ∈ {human, service}`. A `service` user **MUST NOT** hold `password`, `totp`, `oidc`, media-server, `forward_auth` or `local_os` identities, and **MUST NOT** hold session or device credentials.

**IAM-011** (MUST) A user may hold any number of identities; an identity is unique by `(provider, subject)`. Linking a new identity to an existing user **MUST** require the user to be signed in to bonnyread and to complete the provider's sign-in in the same flow.

**IAM-012** (MUST NOT) bonnyread **MUST NOT** auto-link identities by e-mail address. Exception: an admin **MAY** enable "link by verified e-mail" on a specific OIDC provider; linking then happens only when that provider asserts `email_verified = true`, and never for a user holding the Owner role.

**IAM-013** (MUST) Usernames are unique, case-insensitive (compared after Unicode NFKC case folding), 1–64 characters.

**IAM-014** (MUST) At least one user always holds the Owner role. Removing, disabling or demoting the last Owner **MUST** fail with 409 `last_owner`. Ownership is transferred only by an explicit action that assigns Owner to another user.

**IAM-015** (MUST) Disabling a user revokes all its credentials in the same transaction and bumps its `principal_version` (IAM-016).

**IAM-016** (MUST) Every user row carries `principal_version`, bumped in the same transaction as any change to its roles, assignments, identities, credentials or disabled state, and to any role's permission set (bumping all holders). Nodes cache effective permissions for at most 5 seconds keyed by `(user, principal_version)`; the bump is broadcast on the event bus as a best-effort invalidation (A4), so the cache TTL bounds staleness when a broadcast is lost. Open SSE streams of an affected principal are sent `principal.changed` and closed (API-062).

### Built-in password and TOTP

**IAM-020** (MUST) Passwords are hashed with argon2id, stored in PHC string format. Defaults: m = 64 MiB, t = 3, p = 2, 16-byte salt, 32-byte key. The owner **MAY** lower the parameters in bootstrap configuration (`13-deployment.md` DEP-061), never below m = 19 MiB, t = 2. When stored parameters are below the configured ones, the password is re-hashed at the next successful login. _Source: id §3.2; RFC 9106._

**IAM-021** (MUST) Concurrent password hashes are limited by a process-wide semaphore of size `GOMAXPROCS`. Waiting requests queue with the request deadline.

**IAM-022** (MUST) Password policy: minimum length 12, maximum 1024 bytes, checked against a bundled offline list of the 100,000 most common breached passwords. No composition rules. No outbound breach-lookup call.

**IAM-023** (MUST) TOTP (RFC 6238, SHA-1, 6 digits, 30-second step, ±1 step tolerance) is available to every human user, with ten single-use recovery codes stored hashed. A TOTP code **MUST NOT** be accepted twice within its validity window.

**IAM-024** (MUST) TOTP is mandatory for users holding Owner or Admin whenever the server is not loopback-only. When the server stops being loopback-only (for example "Allow access from other devices", a trusted proxy or a non-loopback bind), an Owner or Admin without TOTP **MUST** enrol at the next sign-in before any other operation. The requirement is a setting (`auth.require_totp_for_admins`, default `true`) that only the Owner may change. [OPN-022] _Source: open-issues.md C (Identity)._

**IAM-025** (MUST) Password reset is possible (a) by a user with `user:update` on the target, which sets a new password and revokes the target's sessions and devices, (b) by e-mail only when SMTP is configured: a single-use token, stored hashed, valid 30 minutes, or (c) by the CLI break-glass commands in `11-api-and-clients.md` API-094. Reset by e-mail **MUST NOT** disclose whether the address exists.

**IAM-026** (MAY) Passkeys (WebAuthn) are not in P1; the Identity model reserves provider `webauthn`. [OPN-020]

### OIDC

**IAM-030** (MUST) Any number of OIDC providers may be configured. Each has: display name, slug, issuer URL, client id, client secret (`writeOnly`; absent for public clients), scopes (default `openid profile email`, plus `groups` when a groups claim is configured), claim mapping, group mapping rules, default role, JIT toggle, "link by verified e-mail" toggle.

**IAM-031** (MUST) The flow is authorisation code with PKCE (S256), `state` and `nonce`, always. The ID token is validated for issuer, audience, expiry, `nonce` and signature against the provider's JWKS. JWKS keys are cached and refreshed when an unknown `kid` is seen (at most once per minute per provider). Userinfo is fetched only when mapped claims are missing from the ID token. The PKCE verifier, `state` and `nonce` are held in a short-lived (10 minutes) signed, HttpOnly cookie.

**IAM-032** (MUST) Discovery failure is not fatal. The provider is marked unhealthy, retried with exponential backoff (capped at 15 minutes) and reported as a health issue; other sign-in methods keep working.

**IAM-033** (MUST) Claim mapping: `username` (default `preferred_username`), `displayName` (default `name`), `email` and `email_verified`, `groups`. Each mapping is a dotted path into the claims, so `groups`, `realm_access.roles` and `resource_access.bonnyread.roles` all work.

**IAM-034** (MUST) Group mapping rules are an ordered list of `{claimValue, role, scope?}` per provider. At every OIDC login bonnyread recomputes the user's `oidc_group` assignments for that provider (**group sync**): matching rules are added, assignments whose rule no longer matches are removed. `manual` assignments are never touched. Changes are audited (`role.assigned`/`role.unassigned`, source `oidc_group`) and bump `principal_version`.

**IAM-035** (MUST) If no rule matches and the user holds no `manual` assignment, the provider's default role applies. The default for a new provider is **none**, meaning the login is denied with 403 `no_role`.

**IAM-036** (MUST) Group mapping rules **MUST NOT** assign Owner. Assigning Admin through a rule is allowed, and only the Owner may create such a rule (`role:*`, IAM-073).

**IAM-037** (MUST) Just-in-time provisioning is a per-provider toggle, default on. When off, only users that already hold an identity for that provider (matched by `sub`), or were pre-linked by an admin by username, may sign in.

**IAM-038** (MUST) Logout ends the local session and, when the provider advertises `end_session_endpoint`, performs RP-initiated logout. OIDC back-channel logout is not supported in v1. [OPN-024]

**IAM-039** (MUST) "Disable local login" hides password sign-in for all users. Break-glass remains: an Owner can always sign in with a password over a loopback connection that carries no forwarding headers, and the CLI can mint a one-time login link (API-094).

### Media-server sign-in (Plex, Jellyfin, Emby)

**IAM-040** (MUST) Media-server sign-in is configured on the media-server connection (the same connection used for library scans and refreshes). Several servers may be configured; each has its own sign-in policy: `off | existing_users | anyone_with_access`, plus a default role (default **Requester**) and default scope (all non-adult media kinds). _Source: id §3.2 (Seerr `mediaServerLogin`/`newPlexLogin`)._

**IAM-041** (MUST) Plex: the client runs the plex.tv PIN flow; the core validates the resulting token with plex.tv and checks that the account has access to the configured server (owner, home user or shared user). Jellyfin/Emby: the client posts username and password (or a Quick Connect code on Jellyfin) to the core, which authenticates against the configured server. Only accounts with access to *this* server get in.

**IAM-042** (MUST) The media-server token is stored only if the user opts into a feature that needs it (for example Plex watchlist auto-requests); otherwise it is discarded after verification. Server access is re-checked at each login and at most daily for stored identities. Losing access disables that identity and revokes the credentials it issued.

**IAM-043** (MUST) A user created by media-server sign-in receives a role assignment with source `media_server_default`. Later changes to the server's default role do not modify existing assignments.

### Forward-auth (trusted headers)

**IAM-045** (MUST) Forward-auth sign-in (Authelia, Authentik and similar reverse-proxy authentication) is **opt-in and off by default** (D20). It is enabled only by the Owner (`auth_settings:manage`).

**IAM-046** (MUST) Trusted headers are honoured only when the TCP peer is inside `TRUSTED_PROXIES` (`13-deployment.md` DEP-061). From any other peer these headers are ignored and the request is authenticated as if they were absent. Enabling forward-auth with an empty `TRUSTED_PROXIES` **MUST** be rejected with 422 `validation_failed`.

**IAM-047** (MUST) Header names are configurable, with defaults `Remote-User`, `Remote-Groups` (comma-separated), `Remote-Email`, `Remote-Name`. The user header maps to an existing user holding a `forward_auth` identity with that subject, or, if the admin enables "match by username", to a human user with that username. Group headers map through the same group mapping rules as OIDC (IAM-034), with source `forward_auth_group`. JIT provisioning is a toggle, default **off**. [OPN-119]

**IAM-048** (MUST NOT) Forward-auth **MUST NOT** create an Owner, assign Owner or Admin, or authenticate as a user holding Owner. A request carrying both a trusted user header and a cookie or bearer credential for a different user is rejected with 400 `ambiguous_credentials`.

**IAM-049** (MUST) A forward-auth request is authenticated per request (credential kind `forward_auth`, nothing stored). The first forward-auth request of a user in each 24-hour period is audited as `auth.login {method: forward_auth}`.

### Credentials

**IAM-050** (MUST) All stored credentials are rows of one `Credential` table, kinds `session`, `device`, `api_key`. Each stores only a SHA-256 hash of a 256-bit random secret. A slow hash is not used for these secrets.

**IAM-051** (MUST) Presentation, lifetime and narrowing:

| Kind | Presented as | Lifetime | Narrowing |
|---|---|---|---|
| `session` | Cookie `__Host-br_session` (HttpOnly, Secure, SameSite=Lax, Path=/) | Sliding 7 days idle, 30 days absolute | none |
| `device` | `Authorization: Bearer brd_<id>_<secret>` | Sliding 90 days idle, no absolute limit | optional cap (a permission subset, for example read-only) |
| `api_key` | `Authorization: Bearer brk_<id>_<secret>` or `X-Api-Key: brk_<id>_<secret>` | Optional expiry; default 1 year for human-owned keys; `never` allowed only for service accounts | required permission list |

_Source: id §3.3; A15._

**IAM-052** (MUST) When the server is served over plain HTTP on a non-loopback address, the session cookie falls back to name `br_session` without `Secure`, and the UI shows a persistent warning. On loopback plain HTTP the `__Host-` cookie is used (browsers treat loopback as secure).

**IAM-053** (MUST) The default API-key expiry is 1 year. [OPN-025] The UI warns 14 days before expiry, and an expiring-credential health issue is raised for service-account keys.

**IAM-054** (MUST) Effective permissions of a credential are: the user's effective permissions (IAM-063) intersected with the credential's cap or permission list, recomputed at each request. An API key's permission list is validated at creation to be a subset of the creator's effective permissions; at request time the intersection applies, so a later downgrade of the owner narrows the key.

**IAM-055** (MUST) The `<id>` part gives O(1) lookup; the first 4 characters of the secret are stored as a display hint. Credential lists show name, kind, hint, created, last used (updated at most once per minute), last IP, expiry, cap.

**IAM-056** (MUST) Users may always list and revoke their own credentials. Managing other users' credentials requires `credential:*` (IAM-073). Revoking a credential takes effect on the next request on every node (IAM-016), and closes SSE streams using it.

**IAM-057** (MUST) A revoked or expired credential returns 401 with problem code `credential_revoked` or `credential_expired`, so native apps can return to sign-in for that server.

### Login throttling

**IAM-058** (MUST) Login throttling uses database-backed counters shared across HA nodes and surviving restarts:

| Key | Threshold | Effect |
|---|---|---|
| (IP, username) and username | 5 failures in 15 minutes | Exponential delay starting at 1 s, doubling per further failure, capped at 15 minutes |
| IP | 30 failures in 15 minutes | 429 `login_throttled` for 15 minutes, with `Retry-After` |

Failures against usernames that do not exist and match the suspicious-name list (`admin`, `root`, `administrator`, `test`, `user`, and similar, bundled) count three times. The same limiter covers TOTP verification, recovery codes, password reset, pairing claims and native code exchange. _Source: id §3.3 (Profilarr `rateLimit.ts`, `loginAnalysis.ts`)._

**IAM-059** (MUST) Login responses and timing are identical for unknown users and wrong passwords: an unknown user is verified against a dummy argon2id hash with the current parameters.

### Permissions, scopes and the authoriser

**IAM-060** (MUST) There is exactly one authorisation entry point, `Authorize(principal, permission, target)`, where `target` carries `{kind, libraryGroup, ownerId}` (absent for unscoped resources), and one list helper `ScopeFilter(principal, permission)` that returns a query predicate so lists and cursors respect scope. Every API operation declares its permission (`11-api-and-clients.md` API-030). _Source: id §3.4._

**IAM-061** (MUST) Actions are exactly: `read`, `create`, `update`, `delete`, `execute` (search, grab, import, rename, test), `approve`, `manage` (settings of that resource family). Other spec files **MAY** add resources, **MUST NOT** add actions. Facets narrow an action to a named sub-area; the defined facets are:

| Permission | Facets |
|---|---|
| `library:execute` | `search`, `grab`, `import`, `rename` |
| `system:read` | `health`, `jobs`, `logs` |
| `activity:update` | `processing` (external processing-hold callbacks, `05-downloads.md` DLD-079) |

A grant without a facet includes all facets.

**IAM-062** (MUST) Resources:

| Resource | Covers | Scoped |
|---|---|---|
| `library` | Titles, Items, TitleSlots, ItemSlots, MediaFiles, item history, calendar, wanted lists, manual import | kind, library group |
| `request` | requests (Intent `user_request`), approvals | kind, library group, ownership |
| `issue` | issue reports on items | kind, ownership |
| `discover` | metadata search and browse for requesting and adding | kind |
| `activity` | queue, download progress, blocklist, global history | kind |
| `indexer` | indexers, IRC networks, feeds, indexer proxies, DDL hosts | no |
| `download_client` | download clients and download-handling policies | no |
| `profile` | quality profiles, custom formats, PCD databases, naming presets | no |
| `media_management` | root folders, library groups, naming, file handling, remote path mappings, extraction | no |
| `import_list` | import lists and Follows | no |
| `subtitle` | subtitle providers and subtitle operations | no |
| `notification` | notification and alert targets, webhooks | no |
| `media_server` | Plex/Jellyfin/Emby connections | no |
| `user` | users, role assignments, quotas, identities | no |
| `role` | custom roles, group mapping rules | no |
| `credential` | other users' API keys, devices and sessions | no |
| `auth_settings` | password policy, OIDC providers, media-server sign-in, forward-auth, pairing settings | no |
| `system` | general settings, backups, logs, jobs, health, plug-ins | no |
| `metrics` | `/metrics`, `/metrics/cluster` | no |
| `audit` | audit log | no |

**IAM-063** (MUST) A scope is attached to a role assignment, not to a permission. A principal has permission *p* on target *t* if any of its assignments grants *p* and that assignment's scope contains *t* (kinds ∋ t.kind; library groups empty-or-∋ t.libraryGroup; ownership `any`, or `own` and t.ownerId = user). Unscoped resources ignore scope. Effective permissions are therefore the union over assignments, each with its own scope.

**IAM-064** (MUST) Every scoped row carries its media kind and library group (derived from its RootFolder), so the same scope check applies to lists, reads, writes and SSE events. A scoped resource outside the caller's scope is reported as 404, not 403 (API-021).

**IAM-065** (MUST) Media kinds in scope are `tv`, `movie`, `music`, `book`, `adult`, `comic` (the SlotDefinition kind enum, `02-domain-model.md`).

### Adult content

**IAM-066** (MUST) Adult content is opt-in per user. No built-in role's default scope includes kind `adult` except Owner's. Adding `adult` to an assignment is an explicit action requiring `user:update`, audited as `scope.adult_granted`. Media-server and OIDC default scopes **MUST NOT** include `adult` unless the rule or policy names it explicitly.

**IAM-067** (MUST) Adult Titles, Items, discovery results, search results, calendar entries, activity rows, requests, issues and SSE events are filtered by the scope check on the server. Hiding adult content is never a UI-only concern.

### Version slots replace 4K permissions

**IAM-068** (MUST) Request rights per version are settings on each SlotDefinition (`02-domain-model.md`): `requestableBy: [roleId]` and `autoApproveFor: [roleId]`. There are no per-quality-tier permission flags. _Source: id §3.4; D4._

**IAM-069** (MUST) Defaults: for the default slot of each media kind, `requestableBy = {Owner, Admin, Manager, Member, Requester}` and `autoApproveFor = {Owner, Admin, Manager, Member}`; for every other slot, `requestableBy = autoApproveFor = {Owner, Admin, Manager}`. A request is accepted only if the principal holds `request:create` in scope for the kind and holds at least one role in the slot's `requestableBy`; it is auto-approved if the principal holds a role in `autoApproveFor`. Custom roles are added to these lists explicitly.

### Default roles

**IAM-070** (MUST) The built-in roles are Owner, Admin, Manager, Member, Requester, Viewer and Service. They cannot be deleted or edited; their permission sets are fixed per release, so upgrades extend them predictably. Custom roles **MAY** be cloned from any built-in role and edited.

**IAM-071** (MUST) The built-in permission matrix is (✓ granted, – not granted, *opt* ticked per service account, `own`/`any` ownership scope):

| Permission | Owner | Admin | Manager | Member | Requester | Viewer | Service |
|---|---|---|---|---|---|---|---|
| `library:read` | ✓ | ✓ | ✓ | ✓ | – | ✓ | opt |
| `library:create/update/delete` | ✓ | ✓ | ✓ | – | – | – | opt |
| `library:execute` | ✓ | ✓ | ✓ | `:search` only | – | – | opt |
| `discover:read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | opt |
| `request:create` | ✓ | ✓ | ✓ | ✓ | ✓ | – | opt |
| `request:read` | any | any | any | any | own | own | opt |
| `request:update` (edit, withdraw) | any | any | any | own | own | – | – |
| `request:approve` (approve, decline, change slot, bypass quota) | ✓ | ✓ | ✓ | – | – | – | – |
| `issue:create` | ✓ | ✓ | ✓ | ✓ | ✓ | – | – |
| `issue:read` | any | any | any | any | own | own | – |
| `issue:update` (resolve) | ✓ | ✓ | ✓ | – | – | – | – |
| `activity:read` | ✓ | ✓ | ✓ | ✓ | – | ✓ | opt |
| `activity:update/delete` (remove, blocklist) | ✓ | ✓ | ✓ | – | – | – | opt |
| `indexer`, `download_client`, `media_management`, `import_list`, `subtitle`, `notification`, `media_server`: `read` | ✓ | ✓ | ✓ | – | – | – | opt |
| same resources: `create/update/delete/execute/manage` | ✓ | ✓ | – | – | – | – | opt |
| `profile:*` | ✓ | ✓ | ✓ | – | – | – | opt |
| `user:*` | ✓ | ✓ (not Owner/Admin assignments) | – | – | – | – | – |
| `role:*` | ✓ | read | – | – | – | – | – |
| `auth_settings:*` | ✓ | read | – | – | – | – | – |
| `credential:*` (other users) | ✓ | ✓ (not Owners') | – | – | – | – | – |
| `system:read` | ✓ | ✓ | `:health`, `:jobs` | `:health` | – | – | opt |
| `system:manage` (settings, backups, plug-ins) | ✓ | ✓ | – | – | – | – | – |
| `metrics:read` | ✓ | ✓ | – | – | – | – | opt |
| `audit:read` | ✓ | ✓ | – | – | – | – | – |
| Auto-approve (IAM-069 defaults) | all slots | all slots | all slots | default slot | – | – | – |
| Quota applies (IAM-080) | no | no | no | yes (default unlimited) | yes | n/a | no |
| Default scope | all kinds incl. adult | all non-adult | all non-adult | all non-adult | all non-adult | all non-adult | none |

_Source: id §3.4. `request:update` own/any is derived from the design's "requests (own vs all is a scope)"; the design's matrix had no row for it._

**IAM-072** (MUST) Role intents: **Owner** holds everything and is the only role that manages authentication settings, custom roles and Admin/Owner assignments. **Admin** runs the server: everything except auth settings, custom roles and Owner accounts. **Manager** curates the library and the request queue. **Member** is a trusted household user. **Requester** discovers and requests under a quota and sees only its own requests and issues; it is the default for media-server sign-ins. **Viewer** is read-only and cannot request. **Service** is a template with no permissions until the creator ticks them.

**IAM-073** (MUST) Only an Owner may assign or remove Owner or Admin, create, edit or delete custom roles and group mapping rules, or change `auth_settings`. An Admin **MUST NOT** revoke, disable or edit an Owner.

**IAM-074** (MUST) Two service-account presets are offered: **Metrics scraper** (`metrics:read`) and **Automation** (`library:read`, `library:execute`, `activity:read`, `request:create`). A service account's permissions **MUST** be a subset of its creator's effective permissions.

**IAM-075** (MUST) A custom role's permissions **MUST** be a subset of the creating Owner's permissions and **MUST NOT** include `auth_settings:manage`, `role:*` or the ability to assign Owner.

### Request quotas

**IAM-080** (MUST) Quotas are per media kind: `limit` units per rolling `days`. The unit is the `Intent.quota_units` value stored at request creation by `02-domain-model.md` DOM-070 (TV: requested seasons, all seasons for a whole-series request; movie: 1; music: albums; book: books; adult: items; comic: volumes); this file does not redefine it. [OPN-118] _Source: id §3.4 (Seerr `User.getQuota`)._

**IAM-081** (MUST) Resolution order: user override → role default (the most generous among the user's assignments that grant `request:create` for that kind) → global default. `limit = null` means unlimited. Built-in defaults: Requester 10 movies and 10 seasons per 7 days; Member unlimited; other kinds unlimited until configured. [OPN-021]

**IAM-082** (MUST) Quota usage is the sum of `quota_units` over Intent rows (`origin = user_request`, `requested_by = user`, created within the window, status `pending` or `approved`), not a counter, so edits and declines are reflected at once. Declined and withdrawn requests do not count. Requests created with `ignore_quota = true` do not count.

**IAM-083** (MUST) A request that would exceed the quota fails with 403 `quota_exceeded`, including `limit`, `used`, `remaining` and `resetsAt` in the problem body. A principal with `request:approve` in scope **MAY** set `bypassQuota` when creating or approving a request, which sets `ignore_quota` and is audited.

**IAM-084** (MUST) `GET /api/v1/users/me/quota` (and `/users/{id}/quota` with `user:read`) returns per kind `{limit, days, used, remaining, resetsAt}`.

### Audit log

**IAM-090** (MUST) The audit log is append-only. Each event records: time, actor (user id and credential id and kind, or `system` with job id), action (the operation's `x-permission` plus operation id, or a named auth event), target (type, id, display name), a before/after JSON diff with every `writeOnly` field redacted, IP (resolved through `TRUSTED_PROXIES`), user agent, request id, outcome (`ok`, `denied`, `error`).

**IAM-091** (MUST) Recorded: all writes; all denied requests; logins (success and failure, with method); logouts; pairing claims and completions; credential creation, use of a new IP by a device, revocation and expiry; role and scope changes including group sync; adult-scope grants; quota bypasses; reads of secret-bearing resources' secret state. Not recorded: other reads and SSE traffic.

**IAM-092** (MUST) Retention defaults to 180 days (setting `audit.retention_days`, minimum 7), pruned by a scheduled job. Audit events are readable through `GET /api/v1/audit-events` (cursor-paged, filterable by actor, action, target, outcome and time; `audit:read`) and are also emitted as JSON log lines with `log_type = audit` for log shippers.

**IAM-093** (MUST NOT) No API operation updates or deletes individual audit events.

## Data model

Shared entities (Intent, SlotDefinition, RootFolder, MediaServer connection, DOM-111) are owned by `02-domain-model.md`. This file owns the following.

```
User {
  id uuidv7; kind enum(human, service); username string unique (folded); display_name string
  email string?; email_verified bool; disabled bool; principal_version int64
  created_by -> User?; created_at, updated_at time; last_login_at time?
  edges: identities, credentials, role_assignments, quota_overrides
}

Identity {
  provider string            // password | totp | oidc:<issuer> | plex | jellyfin | emby | forward_auth | local_os | webauthn (reserved)
  subject string             // sub, media-server account id, OS principal (local_os: "<machine_id>:<uid|SID>")
  secret_hash string?        // argon2id PHC (password), encrypted TOTP seed (totp)
  claims json                // last claims snapshot (redacted)
  token string? (secret)     // media-server token, only when a feature needs it (IAM-042)
  disabled bool; last_verified_at time?
  edges: user -> User; oidc_provider -> OIDCProvider?; media_server -> MediaServer?
  unique (provider, subject)
}

RecoveryCode { user -> User; code_hash string; used_at time? }

Credential {
  id string (the <id> part); kind enum(session, device, api_key)
  secret_hash bytes (sha256); hint string(4); name string
  permissions []string?      // api_key: required list; device: optional cap; session: null
  device json?               // {name, platform, appVersion}
  created_at, last_used_at, expires_at?, absolute_expires_at?, revoked_at?; last_ip string?
  edges: user -> User; issued_by_identity -> Identity?; created_by_credential -> Credential?
  index (user_id, kind); index (expires_at)
}

Role { id; name unique; builtin bool; cloned_from -> Role?; permissions []string }

RoleAssignment {
  edges: user -> User; role -> Role; source_provider -> OIDCProvider?; source_rule -> GroupMappingRule?
  source enum(manual, oidc_group, forward_auth_group, media_server_default)
  kinds []enum(tv, movie, music, book, adult, comic); library_groups []uuid (empty = all)
  ownership enum(own, any)?  // applies to request and issue
  unique (user_id, role_id, source, source_rule_id)
}

LibraryGroup { id; name unique; edges: root_folders M2M RootFolder }

OIDCProvider {
  slug unique; display_name; issuer string; client_id; client_secret (secret)?; scopes []string
  claim_map json; default_role -> Role?; default_scope json; jit bool; link_by_verified_email bool
  health enum(ok, unhealthy); jwks_cache json; jwks_fetched_at time?
}

GroupMappingRule {
  provider enum(oidc, forward_auth); edges: oidc_provider -> OIDCProvider?; role -> Role
  position int; claim_value string; scope json?
  unique (provider, oidc_provider_id, position)
}

MediaServerSignInPolicy {   // one per MediaServer connection (02-domain-model.md DOM-111)
  edges: media_server -> MediaServer (unique); default_role -> Role
  mode enum(off, existing_users, anyone_with_access); default_scope json
}

QuotaPolicy   { scope enum(global, role); edges: role -> Role?; kind enum; limit int?; days int; unique (scope, role_id, kind) }
QuotaOverride { edges: user -> User; kind enum; limit int?; days int; unique (user_id, kind) }

LoginThrottle { key string (hashed "ip:…", "user:…", "ipuser:…"); failures int; window_start time; blocked_until time?; unique (key) }

PasswordResetToken { edges: user -> User; token_hash bytes; expires_at time; used_at time? }

AuditEvent {
  id uuidv7; at time; actor_user -> User?; actor_credential_id string?; actor_credential_kind enum?
  actor_system string?       // "system" | "job:<id>"
  action string; operation_id string?; target_type, target_id, target_name string?
  diff json; ip string?; user_agent string?; request_id string; outcome enum(ok, denied, error)
  index (at); index (actor_user_id, at); index (target_type, target_id, at)
}

AuthSettings (settings namespace "auth"): disable_local_login bool; require_totp_for_admins bool;
  forward_auth {enabled, user_header, groups_header, email_header, name_header, match_by_username, jit, default_role, default_scope}
```

TOTP seeds are reversible secrets; like other secrets they rely on database-level encryption at rest (D16).

## Flows

### F1. Password sign-in

1. `POST /api/v1/auth/login {username, password}`. The limiter (IAM-058) is checked for (IP, username), username and IP; a block returns 429.
2. Look up the `password` identity by folded username; if absent, verify against the dummy hash (IAM-059) and fail.
3. On success with TOTP enrolled, return `{step: "totp", challenge}`; the client posts `POST /auth/login/totp {challenge, code | recoveryCode}`.
4. If IAM-024 requires TOTP and none is enrolled, return `{step: "enrol-totp"}`; only enrolment endpoints are allowed until done.
5. Create the session credential, set the cookie, audit `auth.login {method: password}`. Re-hash if parameters changed (IAM-020).
6. Failure paths: wrong password or code → counters increment, audit `auth.login` outcome `denied`, generic 401 `invalid_credentials`.

### F2. OIDC sign-in with group sync

1. `GET /api/v1/auth/oidc/{slug}/start` creates `state`, `nonce`, PKCE verifier in the signed cookie and redirects to the discovered authorisation endpoint. If the provider is unhealthy → 503 `provider_unavailable`.
2. Callback: exchange the code, validate the ID token (IAM-031), fetch userinfo only if needed.
3. Look up identity `(oidc:<issuer>, sub)`. If absent: JIT on → create user and identity; JIT off → pre-linked user by username, else 403 `not_provisioned`; "link by verified e-mail" (IAM-012) is applied here.
4. Group sync (IAM-034). If the user ends with no assignment → default role, or 403 `no_role`.
5. If `principal_version` changed, other sessions keep working with the new rights from their next request; open SSE streams are closed with `principal.changed`.
6. Issue a session (browser) or a native exchange code (`11-api-and-clients.md` F4). Audit `auth.login {method: oidc}` and any `role.*` events.

### F3. Media-server sign-in (Plex)

1. The client obtains a Plex token via the PIN flow and posts `POST /api/v1/auth/media-server/{serverId} {plexToken}`.
2. The core validates the token with plex.tv and checks access to the configured server; no access → 403 `no_server_access`.
3. Identity `(plex, <account id>)` found → sign in. Not found: policy `anyone_with_access` → create user with the default role and scope (source `media_server_default`); `existing_users` → 403 `not_provisioned`; `off` → 404.
4. Discard the token unless a feature needs it (IAM-042). Issue session or device credential. Audit.

### F4. Authorising a request

1. Resolve the credential: cookie session, `Authorization: Bearer`, `X-Api-Key`, or trusted forward-auth headers (IAM-046). None → 401 `unauthenticated`. Query credential → 400 (IAM-003).
2. Load `(user, principal_version)` from the ≤5 s cache or the database; disabled user or revoked/expired credential → 401.
3. Compute effective permissions (IAM-054, IAM-063).
4. For single-resource operations, `Authorize(principal, x-permission, target)`; out-of-scope scoped target → 404; missing permission → 403 `forbidden` (audited). For lists, apply `ScopeFilter`.

### F5. Request with quota and slot rights

1. `POST /api/v1/requests` with an `Idempotency-Key`. Checks in order: `request:create` in scope for the kind → the slot's `requestableBy` → quota (IAM-080–083) → exclusions and duplicates (`02-domain-model.md`) → create Intent `pending`, or `approved` if the principal is in the slot's `autoApproveFor`.
2. Over quota → 403 `quota_exceeded` with `resetsAt`. Slot not requestable → 403 `slot_not_requestable`.
3. A principal with `request:approve` approves via `POST /requests/{id}/approve`, optionally changing the slot or setting `bypassQuota`.

## Non-goals

| Excluded | Reason |
|---|---|
| Authentication type "None", "disabled for local addresses", trusted networks, CGNAT trust (Sonarr, Prowlarr, Cleanuparr, Bazarr, Jackett) | Violates IAM-001. Local convenience comes from pairing (`11-api-and-clients.md`). |
| Basic authentication | Credentials on every request with no revocation; sessions and bearer credentials replace it. |
| Global admin API key and `X-API-User` impersonation (Seerr) | Automation acts as a real, audited principal (IAM-074). |
| Query-string API keys | IAM-003; no compatibility façades need them (D7). |
| Seerr's 30-flag permission bitmask, 4K permission duplicates, unused VOTE flag | Replaced by resources × actions × scopes and slot settings (IAM-068). |
| Inter-app API keys | The apps are one process (D12). |
| Passkeys in P1 | Deferred (IAM-026). |
| OIDC back-channel logout in v1 | Deferred (IAM-038). |
| Mobile push | `11-api-and-clients.md` (D20). |

## Traceability

| Reference | Preserved | Changed (D19) |
|---|---|---|
| Sonarr/Radarr | Forms login, OIDC option, "hide local login" (O1) | No None/Basic/External modes; no trusted networks; argon2id instead of PBKDF2 |
| autobrr | Named API keys; argon2id; OIDC with "disable built-in login" | Scopes enforced; discovery failure non-fatal; no first-come onboarding |
| Profilarr | Hashed, scoped, expiring API keys; `x-permission` in OpenAPI; DB-backed login throttle with suspicious-username category; secrets never echoed | Throttle shared across HA nodes; same limiter for pairing and TOTP |
| Seerr | Plex/Jellyfin/Emby sign-in limited to server members (O2); request/approve/auto-approve (O3); quotas per kind with TV counting seasons and manager bypass (O4); REQUEST as the default | Identity rows instead of per-provider columns and `userType`; 4K flags become slot settings; bitmask becomes resource × action × scope |
| Unpackerr | `area:resource:verb` permissions and custom roles | Fixed action vocabulary; scopes on assignments |
| Cleanuparr | TOTP with recovery codes | Mandatory for Owner/Admin off loopback |
| Kapowarr, Jackett, Bazarr | — | Their open-by-default, weak-hash and TLS-off behaviours are the anti-patterns IAM-001–005 prohibit |
