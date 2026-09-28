# Seerr — Codebase Analysis

Source: `submodules/seerr` (commit `4fd0da04`, 2026-09-26). All paths below are relative to `submodules/seerr/` unless noted. `S/` = `server/`.

Seerr is the merger of **Overseerr** (Plex-only, archived) and **Jellyseerr** (its Jellyfin/Emby fork). The merged codebase is the Jellyseerr tree plus a one-shot migrator for Overseerr databases (`S/lib/overseerrMerge.ts`).

---

## 1. Overview

**Purpose:** a request and discovery front end for a media server. Users sign in with their Plex, Jellyfin or Emby account (or a local account), browse TMDb-backed discovery pages, and request movies or TV seasons. Requests go through permissions, quotas and approval. Approved requests are sent to Radarr or Sonarr over their `/api/v3` API. Seerr then tracks download progress (from the Arr queue) and availability (from the media server and from the Arrs), notifies users, and handles issue reports ("the audio is broken on S02E03").

**Tech stack**
- Backend: Node 22 (`Dockerfile`: `node:22.23.2-alpine3.23`), TypeScript 5.4.5, **Express 5.2** as a custom server around **Next.js 16.2** (`S/index.ts`, `next({ dev })`), TypeORM 0.3.31, node-schedule 2.1.1 (cron jobs), axios 1.15 + axios-rate-limit, winston 3.19 (+ daily rotate), express-session with `connect-typeorm` (sessions stored in the DB), `@dr.pogodin/csurf` (CSRF), express-openapi-validator 5.6 against the checked-in `seerr-api.yml`, swagger-ui-express (served at `/api-docs`), bcrypt 6, nodemailer 8 + email-templates 13 + pug 3, openpgp 6 (PGP-encrypted email), web-push 3.6, sharp (image cache), xml2js (anime-list mapping), zod 4 and yup 1.
- Frontend: React 19.2, Next.js **pages router** (`src/pages/`), SWR 2.4, Tailwind 3.4, Headless UI, react-intl 7 (FormatJS), Formik + yup, react-select, react-spring.
- Package manager pnpm 10; Cypress E2E (`cypress/`); Helm chart (`charts/seerr-chart`); Docusaurus docs (`gen-docs/`, `docs/`).
- License: MIT.

**Size (LOC, non-test unless noted)**

| Area | LOC | Notes |
|---|---:|---|
| `S/api` (external API clients) | 6,542 | TMDb, TVDB, Plex, plex.tv, Jellyfin/Emby, Radarr/Sonarr, Tautulli, RT, IMDb proxy, GitHub, anime-list |
| `S/lib` | 10,652 | scanners 2,562; notifications 3,155; availabilitySync 1,218; settings; overseerrMerge 563 |
| `S/routes` | 9,926 | the `/api/v1` surface |
| `S/entity` + `S/subscriber` | 2,462 + 1,672 | domain model plus TypeORM subscribers holding the request→Arr side effects |
| `S/migration` | 4,398 | 57 SQLite + 21 Postgres migrations |
| `S/job`, `S/utils`, `S/middleware` | ~1,400 | |
| Server tests | ~8,400 | `*.test.ts`, Node test runner (`S/test/index.mts`) |
| `src/` (frontend) | ~52,500 | components 48k across 336 files |
| `seerr-api.yml` | 8,307 | OpenAPI 3 spec, 168 paths |
| i18n | 41 locales | `src/i18n/locale/*.json`, `S/i18n/locale/*.json` |

**Architecture:** one Node process that serves Next.js SSR/static pages and `/api/v1` JSON, plus `/imageproxy` and `/avatarproxy`. Settings live in a JSON file (`config/settings.json`, `S/lib/settings/index.ts`). Domain data lives in SQLite or Postgres. Side effects (sending to Arr, notifications, updating parent status) happen in **TypeORM entity subscribers** (`S/subscriber/*.ts`) on `afterInsert`/`afterUpdate`, so request state changes cause remote calls as a side effect of saving.

---

## 2. Domain model

Entities in `S/entity/`:

- **User** (`User.ts`): email, username, displayName, password (bcrypt, local users), resetPasswordGuid + recoveryLinkExpirationDate, `userType` (`S/constants/user.ts`: PLEX=1, LOCAL=2, JELLYFIN=3, EMBY=4), plexId/plexUsername/plexToken, jellyfinUserId/jellyfinUsername/jellyfinDeviceId/jellyfinAuthToken, avatar (+ ETag/version cache), `permissions` (bitmask), per-user quota overrides (`movieQuotaLimit/Days`, `tvQuotaLimit/Days`), requestCount. Relations: requests, watchlists, settings (1:1), pushSubscriptions, createdIssues. **User id 1 is special:** it is the admin/owner, and API-key calls act as user 1 by default (`S/middleware/auth.ts:17`).
- **UserSettings** (`UserSettings.ts`): locale, discoverRegion, streamingRegion, originalLanguage, pgpKey, `discordIds` (JSON array, migrated from a single `discordId` in `1779783365432-AddDiscordIdsColumn`), pushbullet/pushover tokens and sound, telegramChatId/MessageThreadId/SendSilently, `watchlistSyncMovies/Tv`, `notificationTypes` (per-agent bitmask JSON).
- **Permission** is not a table; it is a bitmask enum (`S/lib/permissions.ts`), 30 flags: ADMIN, MANAGE_SETTINGS, MANAGE_USERS, MANAGE_REQUESTS, REQUEST, VOTE (unused), AUTO_APPROVE(_MOVIE/_TV), REQUEST_4K(_MOVIE/_TV), REQUEST_ADVANCED, REQUEST_VIEW, AUTO_APPROVE_4K(_MOVIE/_TV), REQUEST_MOVIE/TV, MANAGE/VIEW/CREATE_ISSUES, AUTO_REQUEST(_MOVIE/_TV), RECENT_VIEW, WATCHLIST_VIEW, MANAGE/VIEW_BLOCKLIST. ADMIN implies everything.
- **Media** (`Media.ts`): one row per (tmdbId, mediaType). Holds tmdbId, tvdbId, imdbId, `status` and `status4k` (`MediaStatus`: UNKNOWN=1, PENDING, PROCESSING, PARTIALLY_AVAILABLE, AVAILABLE, BLOCKLISTED, DELETED), `serviceId(4k)` (which Arr server in settings), `externalServiceId(4k)` (Radarr movie id / Sonarr series id), `externalServiceSlug(4k)`, `ratingKey(4k)` (Plex), `jellyfinMediaId(4k)`, mediaAddedAt, lastSeasonChange. Computed after load: `serviceUrl(4k)` (link into Arr UI: `{externalUrl}/movie/{slug}` or `/series/{slug}`), `mediaUrl(4k)` (Plex/Jellyfin/Emby deep link), `tautulliUrl(4k)`, `downloadStatus(4k)` (in-memory from the download tracker). **Every availability field is duplicated for 4K.**
- **Season** (`Season.ts`): media → seasons, seasonNumber, status, status4k.
- **MediaRequest** (`MediaRequest.ts`): status (`MediaRequestStatus`: PENDING=1, APPROVED, DECLINED, FAILED, COMPLETED), media, requestedBy, modifiedBy, type, is4k, **Arr routing overrides**: `serverId` (Arr server id in settings.json), `profileId` (remote quality profile id), `rootFolder` (remote path), `languageProfileId` (Sonarr v3 only), `tags` (remote tag ids), plus isAutoRequest, ignoreQuota, seasonCount. Children: **SeasonRequest** (seasonNumber, status).
- **Issue** (`Issue.ts`): issueType (VIDEO, AUDIO, SUBTITLES, OTHER), status (OPEN, RESOLVED), problemSeason, problemEpisode, media, createdBy, modifiedBy, comments → **IssueComment** (user, message).
- **Blocklist** (`Blocklist.ts`): (tmdbId, mediaType) unique, user, media (1:1), `blocklistedTags` (the TMDb keyword ids that caused an automatic blocklist).
- **Watchlist** (`Watchlist.ts`): a local (Jellyfin/Emby-era) watchlist: (tmdbId, mediaType, requestedBy) unique, ratingKey, title, media.
- **OverrideRule** (`OverrideRule.ts`): radarrServiceId/sonarrServiceId, conditions (`users`, `genre`, `language`, `keywords` as CSV strings), actions (`profileId`, `rootFolder`, `tags`).
- **DiscoverSlider** (`DiscoverSlider.ts`): type, order, isBuiltIn, enabled, title, data (custom discover rows).
- **UserPushSubscription**: endpoint, p256dh, auth, userAgent.
- **Session**: express-session store.

Relationships: User 1–N MediaRequest N–1 Media 1–N Season; MediaRequest 1–N SeasonRequest; Media 1–N Issue 1–N IssueComment; Media 1–1 Blocklist; User 1–N Watchlist N–1 Media.

**Settings file** (`config/settings.json`, `S/lib/settings/index.ts`, 908 LOC) holds everything that is not a row: main, network, plex, jellyfin (also used for Emby), tautulli, **`radarr[]` / `sonarr[]` server lists**, notification agents, job cron strings, metadata provider choice, VAPID keys, clientId and sessionSecret. It has its own migration chain (`S/lib/settings/migrations/0001…0008`).

**DB engines:** SQLite (default, `config/db/db.sqlite3`) or Postgres (`DB_TYPE=postgres`, `DB_HOST/PORT/USER/PASS/NAME/SOCKET_PATH`, `DB_USE_SSL` + `DB_SSL_*`), `S/datasource.ts`.

**Migrations:** 57 SQLite (`S/migration/sqlite/`, from `1603944374840-InitialMigration`, Overseerr era) and 21 Postgres (`S/migration/postgres/`, from `1734786061496-InitialMigration`, added Dec 2024 in Jellyseerr). The two chains are not parallel. Postgres was squashed at an initial migration, and some fixes exist on only one side (e.g. `DropPushSubscriptionAuthUnique` is SQLite-only; `FixNullFields`/`FixIssueTimestamps`/`FixBlocklistIdDefault` are Postgres-only). `S/lib/overseerrMerge.ts` rewrites the migration list at startup when it detects an Overseerr DB, swapping 12 Jellyseerr-only migrations for a synthetic `SeerrMigration1759769291608`.

---

## 3. Feature areas

**Requests / approvals / auto-approve / quotas** (`S/entity/MediaRequest.ts:50-500`, `S/routes/request.ts`)
- `MediaRequest.request()` checks, in order: REQUEST vs REQUEST_MOVIE/TV (or REQUEST_4K*) permission; the quota; the blocklist (`BlocklistedMediaError`); duplicates (`DuplicateMediaRequestError`, per is4k); for TV, which seasons are still requestable (`NoSeasonsAvailableError`); override rules; then auto-approve.
- Requesting on behalf of another user needs MANAGE_USERS or MANAGE_REQUESTS.
- **Quotas:** N requests per D days, per media type. The user-level value overrides the global `main.defaultQuotas`. It counts non-DECLINED requests, and **TV counts seasons, not requests** (`User.getQuota`, `User.ts:~280-330`). `ignoreQuota` bypasses it, needs MANAGE_REQUESTS, and is stored on the request (migration `1781732098511`).
- **Auto-approve** comes from AUTO_APPROVE(_MOVIE/_TV) or AUTO_APPROVE_4K*, or MANAGE_REQUESTS. Auto-approved requests are created as APPROVED, and the subscriber sends them to the Arr straight away.
- **Advanced requests** (REQUEST_ADVANCED or MANAGE_REQUESTS) let the requester pick the Arr server, quality profile, root folder, language profile and tags. The UI populates these from `GET /service/radarr/:id` / `sonarr/:id`, which proxy the Arr.
- **Override rules** (`S/lib/overrideRules.ts`): per Arr server, if user ∈ users AND genre ∈ genres AND original language matches AND keyword matches, set profileId / rootFolder / tags. Anime TV skips rules unless the rule names the anime keyword (TMDb keyword `210024`).
- Status actions: `POST /request/:id/approve|decline`, `POST /request/:id/retry` (re-sends FAILED requests), `PUT /request/:id` (edit seasons or overrides), `DELETE`.
- **Partial requests** (`main.partialRequestsEnabled`) allow requesting individual seasons; otherwise all seasons are requested. `main.enableSpecialEpisodes` controls season 0.

**4K handling:** a first-class parallel track. Every Arr server has `is4k`, and there is one default per (type, is4k). There are separate permissions (REQUEST_4K*, AUTO_APPROVE_4K*), separate status columns (`status4k`, `Season.status4k`), separate Arr ids and slugs, separate Plex ratingKey and Jellyfin id, separate download status, and separate notifications ("4K Movie Request Now Available"). If the same Radarr host appears as both a 4K and a non-4K server, `availabilitySync` uses the file resolution (width ≥ 2000) to tell them apart (`S/lib/availabilitySync.ts:664-738`). `movie4kEnabled`/`series4kEnabled` in the public settings simply mean "a 4K server exists".

**Discovery (TMDb)** (`S/routes/discover.ts`, 1,005 LOC; `S/api/themoviedb/index.ts`)
- Rows: trending, popular/upcoming movies and TV, by genre, studio, network, language, keyword, and watch provider/region. There is a genre slider and custom sliders (DiscoverSlider). Filters cover certification, runtime, vote, release dates and watch providers.
- TMDb endpoints used: `/discover/{movie,tv}`, `/trending/{all,movie,tv}/{window}`, `/movie/upcoming`, `/search/{multi,movie,tv,keyword,company}`, `/movie/{id}` (+`/recommendations`, `/similar`), `/tv/{id}` (+`/season/{n}`, `/recommendations`, `/similar`), `/person/{id}` (+`/combined_credits`), `/collection/{id}`, `/company/{id}`, `/network/{id}`, `/keyword/{id}` (+`/movies`), `/find/{externalId}`, `/genre/{movie,tv}/list`, `/certification/{movie,tv}/list`, `/configuration/{countries,languages}`, `/watch/providers/{movie,tv,regions}`.
- Search supports `tmdb:`, `imdb:`, `tvdb:` and `year:` prefixes (`S/lib/search.ts`).
- Hide-available, hide-requested and hide-blocklisted filters.
- **Ratings:** Rotten Tomatoes (scraped through RT's own Algolia index, `S/api/rating/rottentomatoes.ts:106`) and IMDb (**through Radarr's hosted proxy `api.radarr.video/v1/movie/imdb/{id}`**, `S/api/rating/imdbRadarrProxy.ts:160`).
- **Optional TVDB** as the TV or anime metadata provider (`S/api/metadata.ts`, `S/api/tvdb/index.ts`): `api4.thetvdb.com/v4` `/login`, `/series/{id}/extended?meta=episodes&short=true`, `/series/{id}/episodes/default/{lang}`, `/seasons/{id}/extended`. It uses a hardcoded project key and an optional user PIN.

**Availability sync from media servers**
- Plex scanner (`S/lib/scanners/plex/index.ts`): `/library/sections`, `/library/sections/{id}/all?includeGuids=1`, recently-added, `/library/metadata/{key}` (+`/children`). It maps ids from GUIDs: new Plex agent (`plex://` → Guid array), legacy `com.plexapp.agents.{imdb,themoviedb,thetvdb}`, HAMA `hama://tvdb-`/`anidb-` (the AniDB→TVDB/TMDb mapping comes from the Anime-Lists XML, `S/api/animelist.ts`), and custom agents.
- Jellyfin/Emby scanner (`S/lib/scanners/jellyfin/index.ts`): `/Library/MediaFolders`, `/Users/{id}/Views`, `/Items?ParentId=…`, `/Items/Latest`, `/Shows/{id}/Seasons`, `/Shows/{id}/Episodes`. It uses ProviderIds `Tmdb`/`TheMovieDb`/`Imdb`/`Tvdb`/`AniDB`.
- Both scanners decide 4K per file from resolution. Season availability is derived from episode counts versus TMDb/TVDB episode counts.
- `availabilitySync` (`S/lib/availabilitySync.ts`) goes the other way. It pages through all AVAILABLE/PARTIALLY_AVAILABLE media and marks them **DELETED** if they are absent from **both** the media server and the Arr (per 4K track, per season).

**Availability from Sonarr/Radarr** (`S/lib/scanners/{radarr,sonarr}/index.ts`, `baseScanner.ts:100-260`)
- Radarr: `hasFile` → AVAILABLE; `monitored && !hasFile` → PROCESSING.
- Sonarr: per season, `statistics.episodeFileCount` against `totalEpisodeCount` gives AVAILABLE or PARTIALLY_AVAILABLE; monitored with 0 files gives PROCESSING.
- The scan also writes serviceId, externalServiceId and slug, so media added directly in the Arr gets linked.
- **Orphan cleanup:** PROCESSING media not found on any Arr server of that 4K class is reset to UNKNOWN, and its requests are declined. This only happens if every server of that class has `syncEnabled` and returned a non-empty library.

**Download progress** (`S/lib/downloadtracker.ts`): every minute, per Arr server, it runs `POST /command RefreshMonitoredDownloads`, then `GET /queue`. It keeps the result in memory keyed by server id and Arr movie/series id, and attaches it to Media as `downloadStatus(4k)`. For Sonarr it keeps the per-episode breakdown (`episode` object).

**Issues** (`S/routes/issue.ts`, `issueComment.ts`, `S/subscriber/Issue*Subscriber.ts`): users with CREATE_ISSUES report a problem against media, season or episode. There are threaded comments, and MANAGE_ISSUES can resolve or reopen. Notifications go out on create, comment, resolve and reopen. **Issues are not forwarded to the Arrs.** There is no automatic re-search or blocklist of the bad file; an admin acts by hand.

**Notifications** (`S/lib/notifications/`): 10 agents (see §4). Event types are a bitmask (`S/lib/notifications/index.ts`): MEDIA_PENDING, MEDIA_APPROVED, MEDIA_AVAILABLE, MEDIA_FAILED, TEST, MEDIA_DECLINED, MEDIA_AUTO_APPROVED, ISSUE_CREATED, ISSUE_COMMENT, ISSUE_RESOLVED, ISSUE_REOPENED, MEDIA_AUTO_REQUESTED. Notifications are per agent and per user (UserSettings.notificationTypes), and admins get admin-scoped events (MANAGE_REQUESTS/MANAGE_ISSUES). Email templates are Pug (`S/templates/email/`: media-request, media-issue, resetpassword, generatedpassword, test-email) with optional PGP encryption.

**User import and auth** (`S/routes/auth.ts`, 1,065 LOC)
- **Plex**: a PIN/OAuth flow in the browser against `plex.tv/api/v2/pins` and `app.plex.tv/auth` (`src/utils/plex.ts`). The server validates the token with `plex.tv/users/account.json`, checks server access through `plex.tv/api/users` and `/api/resources`, and refreshes via `/api/v2/ping`. `newPlexLogin` lets unknown Plex users with server access sign in.
- **Jellyfin/Emby**: `/Users/AuthenticateByName`, Quick Connect (`/QuickConnect/Initiate`, `/QuickConnect/Connect`, `/Users/AuthenticateWithQuickConnect`), and an `Authorization: MediaBrowser Client="Seerr", Device=…, DeviceId=…, Token=…` header. On first setup it creates an API key with `POST /Auth/Keys?App=Seerr`.
- **Local**: email + bcrypt password, a reset link emailed with an expiry, and an optional generated password emailed on creation.
- Import users: `POST /user/import-from-plex`, `/user/import-from-jellyfin`.
- Linking and unlinking a Plex or Jellyfin account on a user: `/user/:id/settings/linked-accounts/*`.
- **No OIDC/SSO**: grep for `oidc|openid` finds nothing. It is a long-requested feature, and trusted-header auth is also absent.

**Blocklist** (`S/routes/blocklist.ts`, `S/job/blocklistedTagsProcessor.ts`): manual per-title blocklisting (MANAGE_BLOCKLIST), including a whole collection. There is also **automatic blocklisting by TMDb keyword**: `main.blocklistedTags` plus `blocklistedTagsLimit`, run weekly, which walks discover results for those keywords and inserts Blocklist rows. Blocklisted media gets `status = BLOCKLISTED` and cannot be requested. `blocklistRegion`/`blocklistLanguage` settings hide content by region or language. The feature was renamed from "blacklist" (settings migration `0008`, DB `RenameBlacklistToBlocklist`, a deprecated `/api/v1/blacklist` alias).

**Watchlist**
- Plex watchlist sync (`S/lib/watchlistsync.ts`): every 3 minutes, for every user with a plexToken and AUTO_REQUEST(_MOVIE/_TV), it reads `discover.provider.plex.tv/library/sections/watchlist/all` (first 20 items) and creates requests with `isAutoRequest=true`.
- Local watchlist (Jellyfin/Emby users): `POST/DELETE /watchlist`, shown as a discover row. **Local watchlist entries are not auto-requested.**

**Scheduled jobs:** see §6. Jobs can be run, cancelled and rescheduled from the UI (`/settings/jobs`).

**i18n:** 41 locales (react-intl on the client; `S/i18n` for server-rendered notification text). Per-user locale, and a per-agent locale for Discord/Gotify/ntfy/Slack.

**Other:** Tautulli integration (`S/api/tautulli.ts`, `GET /api/v2?cmd=get_tautulli_info|get_item_watch_time_stats|get_item_user_stats|get_user_watch_time_stats|get_history`) gives per-media and per-user watch data; the image proxy and cache (`S/lib/imageproxy.ts`, TMDb images on disk, `cacheImages`); an avatar proxy; demo mode (`UNSAFE_DO_NOT_USE_DEMO`); a version check against GitHub releases and commits (`S/api/github.ts`); PWA with web push (VAPID).

---

## 4. Integrations (full lists)

### Media servers (exactly one active, `main.mediaServerType`: PLEX=1, JELLYFIN=2, EMBY=3, NOT_CONFIGURED=4)
| Server | Used for | Files |
|---|---|---|
| Plex Media Server | library scan, availability, deep links | `S/api/plexapi.ts`, `S/lib/scanners/plex/` |
| plex.tv | auth, server discovery, user list, watchlist | `S/api/plextv.ts` |
| Jellyfin | auth, Quick Connect, scan, user import | `S/api/jellyfin.ts`, `S/lib/scanners/jellyfin/` |
| Emby | the same client as Jellyfin (`ServerType.EMBY`, URL differences only) | same |
| Tautulli (optional) | watch stats | `S/api/tautulli.ts` |

### *arr apps
| App | Versions | Files |
|---|---|---|
| Radarr | `/api/v3` (v3/v4/v5/v6 all serve v3) | `S/api/servarr/radarr.ts` |
| Sonarr | `/api/v3`; the major version is read from `system/status` to decide on language profiles (≤3) | `S/api/servarr/sonarr.ts` |
| Lidarr, Readarr, Whisparr, Prowlarr, Bazarr | **none** (no music or book requests) | — |

### Metadata / ratings
| Source | Use | Notes |
|---|---|---|
| TMDb v3 (`api.themoviedb.org/3`) | primary for everything; the core key | **hardcoded project API key** `S/api/themoviedb/index.ts:188`, rate-limited to 50 rps |
| TheTVDB v4 (`api4.thetvdb.com/v4`) | optional TV/anime seasons and episodes | hardcoded project key `S/api/tvdb/index.ts:95` + optional user PIN |
| Rotten Tomatoes | critic/audience scores | unofficial scrape of RT's Algolia index with an embedded key (`rottentomatoes.ts:106-113`); fragile |
| IMDb | rating | via **`api.radarr.video/v1`** (Servarr-hosted) |
| Anime-Lists (`raw.githubusercontent.com/Anime-Lists/anime-lists`) | AniDB→TVDB/TMDb mapping for HAMA/Jellyfin AniDB ids | refreshed every 24h into `config/anime-list.xml` |
| GitHub API | update check | `S/api/github.ts` |

### Notification agents (`S/lib/notifications/agents/`) — 10
Discord (webhook, role/thread, user mentions via `discordIds`), Email (SMTP, PGP), Gotify, ntfy, Pushbullet, Pushover (with a sound list fetched from `api.pushover.net`), Slack (webhook), Telegram (bot API, thread id), Webhook (custom JSON template with `{{…}}` variables), WebPush (VAPID).

**Obsolete / deprecated:**
- **LunaSea** agent removed (settings migration `0006_remove_lunasea.ts`); LunaSea is abandoned.
- Pushbullet is a legacy service with low usage.
- The **default webhook JSON template still uses `{{requestedBy_settings_discordId}}` / `reportedBy_…discordId` / `commentedBy_…discordId`** (base64 blob at `S/lib/settings/index.ts:543`), but the KeyMap now only defines `…_discordIds` (`S/lib/notifications/agents/webhook.ts:26-62`). New installs therefore get placeholders that never resolve.
- IMDb via Radarr's proxy, RT via Algolia, and the hardcoded TMDb/TVDB keys are all "borrowed" access (see §10).

---

## 5. External APIs / interfaces

**Own API:** `/api/v1/*` (`S/routes/index.ts`), documented in `seerr-api.yml` (168 paths) and validated at runtime by express-openapi-validator. Swagger UI is at `/api-docs`. Groups:
- `status`, `status/appdata`, `settings/public`, `settings/discover`, `settings/notifications/pushover/sounds`.
- `auth`: `me`, `plex`, `jellyfin`, `jellyfin/quickconnect/{initiate,check,authenticate}`, `local`, `logout`, `reset-password[/:guid]`.
- `user`: CRUD, `import-from-plex`, `import-from-jellyfin`, `:id/requests`, `:id/quota`, `:id/watch_data`, `:id/watchlist`, `:id/pushSubscription(s)`, and `:id/settings/{main,password,linked-accounts/{plex,jellyfin[/quickconnect]},notifications,permissions}`.
- `request`: list/create, `count`, `:id` get/put/delete, `:id/retry`, `:id/{approve|decline}`.
- `media`: list, `:id/{status}` (manually mark available / pending / …), `DELETE :id`, `DELETE :id/file` (**deletes in the Arr, with files**), `:id/watch_data`.
- `movie/:id` (+recommendations, similar, ratings, ratingscombined), `tv/:id` (+season/:n, recommendations, similar, ratings), `person/:id` (+combined_credits), `collection/:id`, `search` (+keyword, company).
- `discover`: movies, tv (and by language/genre/studio/network/keyword), upcoming, trending, genreslider, watchlist.
- `service`: `radarr`, `radarr/:id`, `sonarr`, `sonarr/:id`, `sonarr/lookup/:tmdbId` (Arr profile, root folder and tag passthrough for the request UI).
- `issue` (+count, `:id/comment`, `:id/{resolved|open}`), `issueComment/:id`, `blocklist` (+collection), `watchlist`, `overrideRule` (+advancedRequest).
- `settings/*` (ADMIN): main, network, plex (+library, sync, devices/servers, users), jellyfin (+library, sync, users), tautulli, radarr/sonarr CRUD + `test` + `:id/profiles`, notifications/{agent} + test, metadata (+test), discover sliders, jobs (+run/cancel/schedule), cache (+flush, dns flush), logs, about, initialize, main/regenerate.
- Deprecated alias: `/api/v1/blacklist` → `/blocklist`, with RFC 8594 `Deprecation`/`Link` headers (`S/middleware/deprecation.ts`).
- Non-API: `/imageproxy/*`, `/avatarproxy/*`.

**Auth methods:** session cookie (express-session in the DB, 30-day, sameSite strict when CSRF is on); **`X-API-Key` header = global admin API key**, with **`X-API-User: <id>`** to impersonate any user (`S/middleware/auth.ts:13-22`). The API key can also come from the `API_KEY` env var. CSRF (`network.csrfProtection`) applies to cookie sessions. There are no per-user API keys and no OAuth/OIDC.

**Webhooks:** outbound only (the Webhook agent). **Seerr accepts no inbound webhooks.** It does not register as a Sonarr/Radarr Connect target or as a Plex/Jellyfin webhook, and everything is polled.

**Known third-party consumers of Seerr's API** (from ecosystem knowledge, not the code): Maintainerr and Janitorr (read requests and delete media), Homepage/Homarr/Organizr widgets (`request/count`, `issue/count`), Requestrr and Doplarr (Discord bots: search + `POST /request`), LunaSea/nzb360/Ruddarr mobile apps, Home Assistant, the Jellyfin "Jellyfin Enhanced" plugin, and Plex/Discord notification relays. All use `X-Api-Key` (+ `X-API-User`).

---

## 6. Background jobs

Defaults from `S/lib/settings/index.ts:578-616`; registered in `S/job/schedule.ts`. The cron strings include seconds and can be edited.

| Job id | Default schedule | What it does |
|---|---|---|
| `plex-recently-added-scan` | every 5 min | Plex recently added → availability (Plex only) |
| `plex-full-scan` | daily 03:00 | full Plex library scan |
| `plex-watchlist-sync` | every 3 min | plex.tv watchlist → auto-requests |
| `plex-refresh-token` | daily 05:00 | ping/refresh stored Plex tokens |
| `jellyfin-recently-added-scan` | every 5 min | Jellyfin/Emby recently added |
| `jellyfin-full-scan` | daily 03:00 | full Jellyfin/Emby scan |
| `radarr-scan` | daily 04:00 | pull all movies from every Radarr server (§9) |
| `sonarr-scan` | daily 04:30 | pull all series from every Sonarr server (§9) |
| `availability-sync` | daily 05:00 | mark media DELETED if gone from both the media server and the Arr |
| `download-sync` | **every 1 min** | `RefreshMonitoredDownloads` + `GET /queue` on every Arr server |
| `download-sync-reset` | daily 01:00 | clear the in-memory download cache |
| `image-cache-cleanup` | daily 05:00 | prune the image cache |
| `process-blocklisted-tags` | 01:30 every 7 days | TMDb keyword → auto-blocklist |

Also: anime-list refresh (24h, lazy); the session store cleanup (connect-typeorm `cleanupLimit: 2`). Jobs only start once at least one user exists (`S/index.ts:155`).

---

## 7. Configuration surface

- **Env:** `PORT`, `HOST`, `CONFIG_DIRECTORY`, `LOG_LEVEL`, `TZ`, `API_KEY`, `DB_TYPE`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS`, `DB_NAME`, `DB_SOCKET_PATH`, `DB_USE_SSL` + `DB_SSL_*` (CA/cert/key/rejectUnauthorized), `WITH_MIGRATIONS`, `PRESERVE_DB` (test), `JELLYFIN_TYPE=emby` (legacy, read only by settings migration 0003), `UNSAFE_DO_NOT_USE_DEMO`, `ALLOW_NETWORK` (test), `COMMIT_TAG` (build).
- **settings.json** sections:
  - `main`: apiKey, applicationTitle/Url, cacheImages, defaultPermissions, defaultQuotas, hide{Available,Blocklisted,Requested}, localLogin, mediaServerLogin, newPlexLogin, discoverRegion, streamingRegion, originalLanguage, blocklistRegion/Language/Tags/TagsLimit, mediaServerType, partialRequestsEnabled, enableSpecialEpisodes, locale, youtubeUrl, versionCheck.
  - `network`: csrfProtection, forceIpv4First, trustProxy, proxy {hostname, port, auth, bypassFilter, bypassLocalAddresses}, dnsCache {enabled, forceMin/MaxTtl}, apiRequestTimeout.
  - `plex`: {name, ip, port, useSsl, machineId, libraries[], webAppUrl}.
  - `jellyfin`: {name, ip, port, useSsl, urlBase, externalHostname, forgotPasswordUrl, libraries[], serverId, apiKey}.
  - `tautulli`: {…, externalUrl}.
  - **`radarr[]` / `sonarr[]`**: see §9.
  - `notifications.agents.*`; `jobs.*.schedule`; `metadataSettings` {tv, anime: tmdb|tvdb}; `vapidPublic/Private`; `clientId`; `sessionSecret`; `migrations[]`.
- **UI-managed DB config:** override rules, discover sliders, per-user permissions, quotas and notification settings.

---

## 8. Platform/infrastructure concerns

- **Single process, JSON settings file plus DB.** Settings are cached in memory and written as a whole file. Multi-instance or HA is impossible, and a restart is needed for some network settings (`S/utils/restartFlag.ts`).
- **Side effects inside ORM subscribers.** `MediaRequestSubscriber.afterUpdate/afterInsert` calls Radarr/Sonarr detached (`.then/.catch`, not awaited). A failure flips the request to FAILED and sends MEDIA_FAILED. There is no retry queue; the retry is manual (`/request/:id/retry`). Nested-transaction workarounds exist for TypeORM bugs (`S/utils/nestedTransaction.ts`, `S/utils/requestLock.ts`, `S/utils/asyncLock.ts`).
- **Polling everywhere:** 1-minute queue polling per Arr server (each poll also forces a `RefreshMonitoredDownloads` command on the Arr), 5-minute media server scans, 3-minute watchlist polling, and daily full Arr library pulls (`GET /movie` and `GET /series` with no paging, so a large library means large payloads).
- **In-memory caches** (`S/lib/cache.ts`: tmdb, tmdbscan, radarr, sonarr, rt, imdb, github, plextv, plexwatchlist, tvdb) and a DNS cache (`S/utils/dnsCache.ts`).
- Outbound HTTP proxy support, IPv4-first, and a custom User-Agent (`S/utils/customProxyAgent.ts`, `userAgent.ts`).
- Next.js SSR in the same process makes the image large (Dockerfile prunes native binaries), and startup is slow.
- Two migration chains (SQLite and Postgres) maintained by hand, plus a settings-file migration chain, plus an Overseerr import path.
- Logs: winston to `config/logs/seerr-%DATE%.log` (daily rotate), viewable via `/settings/logs`.

---

## 9. Interactions with other *arr apps

Seerr only talks to **Radarr and Sonarr**, always at `{useSsl?https:http}://{hostname}:{port}{baseUrl}/api/v3` (`S/api/servarr/base.ts:83`, every call site passes `'/api/v3'`). **Auth is the `apikey` query parameter** set as a default axios param (`base.ts:103-106`), not the `X-Api-Key` header. Timeout = `network.apiRequestTimeout`. `qualityProfile` and `rootfolder` are cached for 1h with a rolling refresh; lookups use the `radarr`/`sonarr` node caches (5 min).

### 9.1 Per-server settings (`DVRSettings`, `S/lib/settings/index.ts:62-104`)
`id, name, hostname, port, apiKey, useSsl, baseUrl, externalUrl, activeProfileId, activeProfileName, activeDirectory, tags[] (remote tag ids), is4k, isDefault, syncEnabled, preventSearch, tagRequests, overrideRule[]`
- Radarr adds `minimumAvailability` (`announced|inCinemas|released`).
- Sonarr adds `seriesType`, `animeSeriesType`, `activeAnimeProfileId/Name`, `activeAnimeDirectory`, `activeLanguageProfileId`, `activeAnimeLanguageProfileId`, `animeTags[]`, `enableSeasonFolders`, `monitorNewItems (all|none)`.

**Seerr persists remote Arr ids**: quality profile id, root folder path, tag ids and language profile id, both in settings.json and in each `MediaRequest` row (`profileId`, `rootFolder`, `languageProfileId`, `tags`, `serverId`) and `OverrideRule` row. `Media.externalServiceId(4k)` holds the Radarr movie id / Sonarr series id, and `serviceId(4k)` is the Seerr-side server index.

### 9.2 Exact calls — shared (`ServarrBase`, `S/api/servarr/base.ts`)
| Call | When | Fields read |
|---|---|---|
| `GET /system/status` | settings test (`S/routes/settings/{radarr,sonarr}.ts` `/test`), `GET /service/sonarr/:id` | `version` (Sonarr major → language profiles), `urlBase` (pre-fills baseUrl) |
| `GET /qualityProfile` | test, service detail, `/settings/radarr/:id/profiles` | `id`, `name` |
| `GET /rootfolder` | test, service detail | `id`, `path`, `freeSpace`, `totalSpace` |
| `GET /tag` | test, service detail, per request when `tagRequests` | `id`, `label` |
| `POST /tag` `{label}` | `tagRequests`: creates user tag `"{userId}-{sanitizedDisplayName}"` | returns `{id,label}` |
| `PUT /tag/{id}` `{id,label}` | settings migration `0007_migrate_arr_tags.ts` only (renames old `"{id} - {name}"` tags) | |
| `GET /queue?includeEpisode=true` | download-sync, every minute | `records[]`: `size, sizeleft, timeleft, estimatedCompletionTime, status, title, downloadId, movieId` / `seriesId, episodeId, episode{seasonNumber, episodeNumber,…}` |
| `POST /command` `{name:"RefreshMonitoredDownloads"}` | download-sync, before each queue read | |

**Queue bug:** `getQueue` sends no `page`/`pageSize`. Sonarr and Radarr default `PageSize ?? 10` (`submodules/sonarr/src/Sonarr.Http/PagingResource.cs:33`, same in Radarr). So **Seerr only ever sees the first 10 queue items per server.** `includeEpisode` is also sent to Radarr, which ignores it.

### 9.3 Exact calls — Radarr (`S/api/servarr/radarr.ts`)
| Call | When |
|---|---|
| `GET /movie` (no params) | `radarr-scan` (full library). Reads `id, tmdbId, title, titleSlug, hasFile, monitored` |
| `GET /movie?tmdbId={id}` | orphan cleanup `existsInAnyServer` |
| `GET /movie/{id}` | `availability-sync`: reads `tmdbId`, `hasFile`, `movieFile.mediaInfo.resolution` (as `"WxH"`) |
| `GET /movie/lookup?term=tmdb:{id}` | before add, and before remove. Needs `id` (0/absent if not in library), `hasFile`, `monitored`, `tags` |
| `POST /movie` | new movie (payload below) |
| `PUT /movie` (id in body, no id in path) | movie exists but unmonitored: `{...lookupResult, title, qualityProfileId, profileId, titleSlug, minimumAvailability, tmdbId, year, tags: union(existing, new), rootFolderPath, monitored, addOptions:{searchForMovie}}` |
| `POST /command` `{name:"MoviesSearch", movieIds:[id]}` | after re-monitoring, or when already monitored without a file and `!preventSearch` |
| `DELETE /movie/{id}?deleteFiles=true&addImportExclusion=false` | `DELETE /api/v1/media/:id/file` (admin "remove from Radarr") |

`POST /movie` payload (`radarr.ts:233-245`, built in `MediaRequestSubscriber.ts:~420`):
```json
{ "title": "<TMDb title>", "qualityProfileId": <id>, "profileId": <id>,
  "titleSlug": "<tmdbId as string>", "minimumAvailability": "released",
  "tmdbId": <int>, "year": <int from TMDb release_date>, "rootFolderPath": "<path>",
  "monitored": true, "tags": [<ids>], "addOptions": { "searchForMovie": <!preventSearch> } }
```
`profileId` is a Radarr v2 field that is ignored by v3+. `titleSlug` is overwritten by Radarr. After success, Seerr stores `id` and `titleSlug` back on Media.

### 9.4 Exact calls — Sonarr (`S/api/servarr/sonarr.ts`)
| Call | When |
|---|---|
| `GET /series` | `sonarr-scan` (full library). Reads `id, tvdbId, title, titleSlug, seasons[].{seasonNumber, monitored, statistics.{episodeFileCount,totalEpisodeCount}}` |
| `GET /series?tvdbId={id}` | orphan cleanup |
| `GET /series/{id}` | `availability-sync`: `tvdbId`, `statistics.episodeFileCount`, `seasons[]` (per-season existence) |
| `GET /series/lookup?term=tvdb:{id}` | before add/remove. Needs `id` (absent if not in library), `seasons[]`, `tags` |
| `GET /series/lookup?term={title}` | `GET /api/v1/service/sonarr/lookup/:tmdbId` (manual TVDB match; uses `sonarr[0]` only) |
| `POST /series` | new series (payload below) |
| `PUT /series` (full lookup object, id in body) | existing series: `monitored`, tags ∪ new, requested seasons set to `monitored:true` |
| `GET /episode?seriesId={id}` | after PUT: find unmonitored episodes in the requested seasons |
| `PUT /episode/monitor` `{episodeIds:[…], monitored:true}` | re-monitor those episodes |
| `POST /command` `{name:"MissingEpisodeSearch", seriesId}` | after PUT on an existing series if `!preventSearch` |
| `GET /languageprofile` | only when Sonarr major ≤ 3 (`S/routes/service.ts:131`, settings test). **Deprecated in Sonarr v4** (`LanguageProfileController` `[Obsolete]`) |
| `DELETE /series/{id}?deleteFiles=true&addImportExclusion=false` | admin "remove from Sonarr" |

`POST /series` payload (`sonarr.ts:282-305`):
```json
{ "tvdbId": <int>, "title": "<TMDb name>", "qualityProfileId": <id>,
  "languageProfileId": <id|undefined>,
  "seasons": [ {"seasonNumber": n, "monitored": <n in requested>} ...all seasons from lookup ],
  "tags": [<ids>], "seasonFolder": <enableSeasonFolders>, "monitored": true,
  "monitorNewItems": "all|none", "rootFolderPath": "<path>",
  "seriesType": "standard|daily|anime",
  "addOptions": { "ignoreEpisodesWithFiles": true, "searchForMissingEpisodes": <!preventSearch> } }
```
- **Anime routing:** the TMDb keyword `210024` switches to the anime root folder, profile, language profile, tags and seriesType (`MediaRequestSubscriber.ts:~590-620`).
- **TVDB id:** comes from TMDb `external_ids.tvdb_id` (or Media.tvdbId). If there is none, the request and media are **deleted** and it throws (`MediaRequestSubscriber.ts:~580`).
- Sonarr's primary key being TVDB means Seerr requires a TVDB id for every TV request, even when TMDb is the metadata source.

### 9.5 Server selection and 4K
- **Server choice:** `request.serverId` if set; otherwise the server with `isDefault && is4k === request.is4k`. If there is none, the request silently stays APPROVED (a warning is logged, `MediaRequestSubscriber.ts:~230`).
- **Precedence:** Arr server defaults → override rules (at request time) → the user's advanced overrides.
- **Tags:** server `tags`/`animeTags`, replaced by request tags if they differ, plus a per-user tag if `tagRequests` is on.
- A 4K server is just another Radarr/Sonarr instance flagged `is4k`. Seerr keeps two parallel status tracks (§3). Duplicate host:port:baseUrl entries are de-duplicated for scans and queue polling (`uniqWith`, `downloadtracker.ts:69`), and queue data is copied to the "matching" servers.

### 9.6 How availability is detected (summary)
1. **Arr scan** (daily): Radarr `hasFile`; Sonarr per-season `episodeFileCount` vs `totalEpisodeCount`. Monitored-without-files → PROCESSING.
2. **Media server scan** (5 min recent + daily full): Plex GUIDs or Jellyfin ProviderIds → AVAILABLE / PARTIALLY_AVAILABLE with 4K by resolution.
3. **Availability sync** (daily): DELETED if absent from both.
4. **Download tracker** (1 min): `queue` only drives the progress bar, never status.

Status changes on Media fire `MediaSubscriber` → requests move to COMPLETED → MEDIA_AVAILABLE notifications.

### 9.7 Monolith implications: in-process vs external contract
| Today (HTTP to Arr) | In a monolith |
|---|---|
| `system/status`, `qualityProfile`, `rootfolder`, `tag`, `languageprofile` | in-process reads of the local profile, root folder and tag tables. Language profiles disappear (Sonarr v4 removed them) |
| `movie/lookup`, `series/lookup` then `POST`/`PUT` `movie`/`series`, `episode/monitor`, `command MoviesSearch/MissingEpisodeSearch` | one in-process "request → ensure media item monitored for {seasons} with {profile, root, tags} and optionally search" call, in the same transaction as the request approval. Removes the FAILED-on-network-error path and the detached promises |
| `queue` + `RefreshMonitoredDownloads` every minute | subscribe to the internal queue/download events; no polling, no 10-item truncation |
| `radarr-scan`/`sonarr-scan`, orphan cleanup, `externalServiceId`/`serviceId` bookkeeping | gone. Request, media item and file state live in one schema, and "availability" is a query (has file? per quality-slot) |
| `availability-sync` (Arr half) | gone. The media-server half remains if you still want "visible in Plex/Jellyfin" as a signal |
| `DELETE movie/series?deleteFiles=true` | in-process delete |
| Per-user Arr tags `"{id}-{name}"` | a real FK: media item → requested_by user(s); tags only if users want them |
| Anime routing via TMDb keyword → separate root/profile | a native "anime" series type/profile selection rule in the monolith's own rules engine (overlaps Sonarr's seriesType and Radarr/Sonarr auto-tagging) |
| `api.radarr.video` for IMDb ratings | must be replaced regardless (D3) |

**External contracts the monolith must keep, or decide to drop:**
- **Seerr's own `/api/v1`** (if the monolith absorbs Seerr): the consumers in §5 (Maintainerr, Homepage, Requestrr/Doplarr, mobile apps, Home Assistant) depend on `request`, `request/count`, `media`, `search`, `movie/:id`, `tv/:id`, `issue/count`, `status`, and `X-Api-Key` + `X-API-User`. This is a bigger third-party surface than the Arr v3 subset Seerr itself uses.
- **If Seerr stays external** (users keep running Seerr against the monolith), the monolith's v3 façade must implement exactly: `system/status` (with `version` whose major is ≥4, to skip language profiles, and `urlBase`), `qualityProfile`, `rootfolder`, `tag` (GET/POST/PUT), `movie` (GET all, GET `?tmdbId=`, GET `/{id}` with `hasFile` and `movieFile.mediaInfo.resolution`, POST, PUT without path id, DELETE `?deleteFiles&addImportExclusion`), `movie/lookup?term=tmdb:`, `series` (GET all with `seasons[].statistics`, GET `?tvdbId=`, GET `/{id}` with `statistics.episodeFileCount`, POST, PUT without path id, DELETE), `series/lookup?term=tvdb:` and `?term=<title>`, `episode?seriesId=`, `episode/monitor`, `queue` (paged records with `movieId`/`seriesId`/`episode`), `command` (`RefreshMonitoredDownloads`, `MoviesSearch`, `MissingEpisodeSearch`), and `?apikey=` query auth. Lookup results must return `id` = 0/absent for items not in the library (Seerr branches on `movie.id`/`series.id`).
- The 4K-as-separate-instance model would have to be emulated (two "virtual instances" or ports) unless Seerr users reconfigure to one server; see D-entry on multiple versions per item in `docs/decisions.md`.

### 9.8 Overseerr/Jellyseerr merge leftovers
- `S/lib/overseerrMerge.ts` (563 LOC): detects an Overseerr DB (no `mediaServerType` in settings) and replaces 12 Jellyseerr migrations with one synthetic migration.
- `RemapOverseerrDeletedStatus` (SQLite `1789254652493`, Postgres `1789254652580`): Overseerr's `MediaStatus.DELETED = 6` collided with Jellyseerr's `BLOCKLISTED = 6`. This is a hand-written data fix, and `down()` is a no-op.
- The dual migration chains (the SQLite chain dates from Overseerr 2020; the Postgres chain is a Jellyseerr Dec-2024 squash).
- `JELLYFIN_TYPE=emby` env var → `mediaServerType = EMBY` (settings migration `0003`). Emby is served by the Jellyfin client class, and `jellyfin*` column names are used for Emby users too (`jellyfinUserId`, `jellyfinAuthToken`).
- Settings migrations: `0001_migrate_hostname` (Jellyfin `hostname` → ip/port/ssl), `0002_migrate_apitokens`, `0004_migrate_region_setting`, `0005_migrate_network_settings`, `0006_remove_lunasea`, `0007_migrate_arr_tags` (renames `"{id} - {name}"` tags in the Arrs, which is also why the request path still looks for both tag formats, `MediaRequestSubscriber.ts:~340`), `0008_migrate_blacklist_to_blocklist`.
- Plex-specific naming spread through generic code: `ratingKey` is the Plex id, while Jellyfin uses `jellyfinMediaId`, and the `Watchlist` entity has a `ratingKey` column.
- The two watchlist implementations: Plex (remote sync + auto-request) and local (Jellyfin, no auto-request).

---

## 10. Outdated / legacy candidates

| Item | Evidence | Why |
|---|---|---|
| Sonarr v3 language profiles | `S/api/servarr/sonarr.ts` `getLanguageProfiles`, `languageProfileId` on `MediaRequest`, `SonarrSettings.active*LanguageProfileId`, `S/routes/service.ts:131` | Sonarr v4 removed them. `/languageprofile` is `[Obsolete]` in Sonarr (`submodules/sonarr/src/Sonarr.Api.V3/Profiles/Languages/LanguageProfileController.cs:12`) |
| Radarr `profileId` field | `radarr.ts:233-245` (`profileId` alongside `qualityProfileId`) | Radarr v2 field, ignored since v3 |
| Tag format `"{id} - {name}"` fallback | `MediaRequestSubscriber.ts` tag lookup, `0007_migrate_arr_tags.ts` | Pre-v4 Arr tag labels allowed spaces |
| Queue polling without paging | `S/api/servarr/base.ts` `getQueue` | Truncated to 10 items; also forces an Arr `RefreshMonitoredDownloads` every minute |
| IMDb via `api.radarr.video` | `S/api/rating/imdbRadarrProxy.ts:160` | Borrowed Servarr infra (D3) |
| RT via Algolia scrape with embedded key | `S/api/rating/rottentomatoes.ts:106-113` | Unofficial, breaks when RT changes |
| Hardcoded TMDb and TVDB project keys | `S/api/themoviedb/index.ts:188`, `S/api/tvdb/index.ts:95` | Should be user- or project-supplied config |
| Legacy Plex agents | `S/lib/scanners/plex/index.ts:25-37` (`com.plexapp.agents.*`, `themoviedb://`, HAMA) | Plex deprecated legacy agents; HAMA is community-only |
| LunaSea agent | `S/lib/settings/migrations/0006_remove_lunasea.ts` | Removed; the app is dead |
| Pushbullet agent | `S/lib/notifications/agents/pushbullet.ts` | Low-usage legacy push service |
| Stale webhook default template (`discordId`) | `S/lib/settings/index.ts:543` vs `S/lib/notifications/agents/webhook.ts:26-62` | Placeholders no longer exist |
| `VOTE` permission | `S/lib/permissions.ts:8` | Declared, not used by any feature |
| `/api/v1/blacklist` alias | `S/routes/index.ts:123-130` | Deprecated rename |
| `JELLYFIN_TYPE` env | `S/lib/settings/migrations/0003_emby_media_server_type.ts` | Pre-merge Emby switch |
| Overseerr import path | `S/lib/overseerrMerge.ts`, `RemapOverseerrDeletedStatus*` | One-time migration; Overseerr is archived |
| Separate SQLite/Postgres migration chains (78 files) | `S/migration/{sqlite,postgres}` | A rewrite starts at a fresh schema plus an importer |
| JSON settings file + its own migration chain | `S/lib/settings/`, 8 migrations | Should be DB config in the monolith |
| Next.js pages router + custom Express server | `S/index.ts`, `src/pages/` | Pages router is legacy in Next 13+; custom server blocks many Next features |
| `sonarr/lookup/:tmdbId` uses `sonarr[0]` only | `S/routes/service.ts:170` | Ignores multiple servers; manual TVDB matching by title |
| 4K-as-separate-server model | everywhere (`status4k`, `*4k` columns, `is4k` settings) | Workaround for the Arrs' one-file-per-item limit (decisions.md, versions-per-item entry) |

---

## 11. Rewrite notes

**Preserve (the real product value)**
- The **user and permission model**: bitmask permissions (drop VOTE, collapse the _MOVIE/_TV/_4K explosion into scoped grants), per-user quotas (TV counted by seasons), and per-user notification preferences and locale.
- **Media-server identity**: Plex OAuth/PIN, Jellyfin/Emby username+password and Quick Connect, user import, and the "must have access to the server" check. This is the reason people run Seerr. Adding **OIDC** would be a clear improvement, since it is missing today.
- The **request workflow**: PENDING → APPROVED/DECLINED → (FAILED) → COMPLETED, season-level requests, auto-approve, requesting on behalf of others, advanced overrides, and override rules (generalise to "request routing rules").
- **Issues** (media/season/episode, comments, resolve). In a monolith these can link to the actual file, and trigger "blocklist release + re-search", which Seerr cannot do.
- **Blocklist by title and by TMDb keyword**, and region/language hiding.
- **Discovery UX**: TMDb rows, custom sliders, watch-provider filters, and `tmdb:/imdb:/tvdb:` search prefixes.
- **Plex watchlist → auto-request**, and extending auto-request to Jellyfin-side lists.
- **Notification events** (12 types) and the webhook template variables. These are user-facing contracts for people's webhook receivers.

**Accidental complexity from being a separate app**
- Everything in §9.7: remote ids stored locally (profile, root, tag, series/movie id, server index); lookup-then-POST/PUT idempotency dances; detached promises and FAILED states on network errors; two full-library pulls a day; per-minute queue polling plus forced refresh commands; orphan cleanup; and "exists in any server" fallbacks.
- **Its own availability tracking** (`Media.status/status4k`, `Season.status/status4k`, three scanners + availabilitySync, ~5k LOC with tests) duplicates what Sonarr/Radarr know exactly (episode files, movie file, quality). In a monolith, availability is derived: requested item → media item → files per quality slot. The media-server scan only needs to remain as an optional "playable in Plex/Jellyfin" signal (and for deep links and Tautulli watch data).
- **Per-user Arr tags** exist only to answer "who requested this" inside the Arr. The monolith has the FK.
- The **4K server duplication** disappears if the monolith supports multiple versions per item.
- The **JSON settings file** and its migrations, and the **dual DB migration chains**.
- The anime routing by TMDb keyword to a different Sonarr root/profile duplicates Sonarr's seriesType and auto-tagging rules.

**Overlap with other apps in scope**
- **TMDb client**: Seerr (full discovery) versus Radarr (via `api.radarr.video` proxy, plus the Radarr "Discover" page built from import lists/recommendations). The monolith needs one TMDb client and cache (D3). Seerr's is the more complete discovery surface, so keep it as the discovery UI and drop Radarr's Discover.
- **TVDB**: Seerr optionally talks to TVDB v4 directly with its own project key. Sonarr uses Skyhook. This should be one metadata service in the monolith.
- **Plex watchlist**: Seerr (auto-request) versus Sonarr/Radarr "Plex Watchlist" import lists (auto-add). This is the same feature twice; keep one, with request/approval semantics.
- **Notifications**: Seerr's 10 agents overlap Sonarr/Radarr Connect (Discord, Email, Gotify, ntfy, Pushbullet, Pushover, Slack, Telegram, Webhook) almost completely. The only additions are WebPush and per-user targeting (Discord mentions, per-user Telegram chat, per-user Pushover/Pushbullet). The monolith should have one notification subsystem with user-scoped routing.
- **Media-server integration**: Sonarr/Radarr notify Plex/Jellyfin/Emby to refresh libraries, and Seerr scans them. One client per media server can do both.
- **Tags, root folders, quality profiles**: Seerr only mirrors these for its UI. In the monolith they are the same tables Profilarr/Recyclarr write to.
- **Issues** overlap with Cleanuparr/blocklist flows (bad release → blocklist + re-search).
- **Queue/download progress** duplicates the Arr queue (and Cleanuparr's queue view).

**API decision input (for the deferred compatibility decision)**
- Seerr is a **consumer** of a small, well-defined v3 subset (§9.7 list, ~20 operations). If the monolith keeps a v3 façade for external Seerr, that subset is the minimum. It needs the quirks: `apikey` query auth, `PUT` without path id, lookup returning `id` absent for non-library items, and `system/status.version` major ≥ 4.
- If the monolith absorbs Seerr, the Arr-side contract Seerr needs vanishes. Seerr's own `/api/v1` then becomes the contract to consider (Maintainerr, Requestrr/Doplarr, Homepage, mobile apps), along with webhook payload variables and `X-Api-Key`/`X-API-User` semantics.
