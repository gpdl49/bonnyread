# autobrr — Codebase Analysis

Submodule: `submodules/autobrr` (HEAD `cd78a70b`, 2026-09-24). All paths below are relative to that root. autobrr is **already Go**, so this report also assesses it as a source of code to reuse directly in the monolith (see §1.4 and §11).

---

## 1. Overview

**Purpose.** autobrr watches private-tracker **IRC announce channels**. It also polls **RSS/Torznab/Newznab feeds** as a fallback. Each announce line is parsed into a `Release` using per-tracker YAML definitions and matched against user **filters**. A matched release runs **actions**:
- push it to a torrent or usenet client directly (qBittorrent, Deluge, rTorrent, Transmission, Porla, Aria2, SABnzbd, NZBGet, a watch folder);
- hand it to an *arr's `release/push` so the *arr's own decision engine decides;
- run a script or a webhook.

Its value is speed. It grabs within seconds of upload, for ratio building on private trackers. The *arrs' RSS polling runs every 15 minutes or more.

**License: confirmed GPL-2.0-or-later.**
- `LICENSE` is the GPLv2 text.
- 351 of 379 `.go` files carry `// SPDX-License-Identifier: GPL-2.0-or-later` with "Copyright (c) 2021 - 2025, Ludvig Lundgren and the autobrr contributors". The other 26 files have no header; examples are `pkg/migrator/migrator.go`, `pkg/sqlite3store/sqlite3store.go`, `internal/auth/oidc.go` and `internal/irc/channel_state_machine.go`. They fall under the repo license.
- 156 frontend files carry the same SPDX header.
- **GPL-2.0-or-later can be relicensed to GPL-3.0 or AGPL-3.0.** That makes it compatible with Sonarr, Radarr and Prowlarr (GPL-3.0) and with Profilarr (AGPL-3.0). Reusing it directly carries no license risk if the monolith is GPL-3.0 or later.

**Tech stack**
- Go: `go.mod` sets `go 1.27.0`, and `Dockerfile` builds with `golang:1.27-alpine`.
  - The code uses method-level type parameters: `func (i *Instance) ClientAs[T any]()` in `internal/downloader/cache.go`. So it needs a current toolchain.
  - Builds use `CGO_ENABLED=0` (pure-Go SQLite).
- Frontend: React 19, TypeScript, Vite, TanStack Router/Query/Form/Table, Tailwind 4, i18next (8 locales in `web/src/i18n/locales`), zod.
  - It is embedded into the binary with `//go:embed all:dist` (`web/build.go`).
- Real-time updates to the UI use SSE (`r3labs/sse`, autobrr fork).

**Architecture.** Layered monolith with one process. `cmd/autobrr/main.go` wires everything by hand in two dependency-ordered `var` blocks: 13 repos, then 17 services. There is no DI framework.
- **`internal/domain`** holds the entities plus a lot of behaviour: filter matching, announce parsing, macro templating and torrent download all live here.
- **`internal/database`** holds the repositories.
- **Services** live in `internal/<area>`, one package per domain area.
- **Interfaces are small and unexported, declared by each consumer.** Constructors return concrete `*Service` (documented in `AGENTS.md`).
- **Import cycles are broken by a typed event bus** (`internal/events`, generic `Topic[T]` over `maniartech/signals`).
- **`internal/server/server.go`** starts the long-running subsystems: IRC handlers, feeds, lists, cleanup jobs and the scheduler.

**Package layout and LOC** (Go, non-test / test, blank lines included)

| Area | Non-test | Test | Notes |
|---|---|---|---|
| `cmd/autobrr` | 273 | – | server main |
| `cmd/autobrrctl` | 648 | – | CLI: `create-user`, `change-password`, `db:convert` (SQLite→PG), `db:seed/reset`, `htpasswd`, `export-filters`, `version` |
| `internal/database` | 10,992 | 9,491 | repos + migrations; filter.go 1,978, release.go 2,100 |
| `internal/domain` | 6,802 | 6,334 | filter.go 1,357, release.go 1,403, indexer.go 796, releasetags.go 576 |
| `internal/http` | 4,893 | 1,603 | chi REST API, SSE, web handler |
| `internal/irc` | 4,831 | 2,681 | handler.go 2,226, connection and channel state machines |
| `internal/notification` | 3,183 | 2,254 | 9 senders |
| `internal/list` | 2,455 | 615 | *arr and external list → filter sync |
| `internal/feed` | 2,323 | 934 | Torznab/Newznab/RSS jobs + cache |
| `internal/action` | 2,227 | 493 | one `run_<client>.go` per action type |
| `internal/release` | 1,431 | 297 | processing pipeline, torrent download service, cleanup jobs |
| `internal/indexer` | 1,271 | 864 | definition loading, API service |
| `internal/filter` | 1,265 | 388 | filter service (external filters, dupes, smart episode) |
| `internal/downloader` | 1,043 | – | download-client registry and connection cache |
| `internal/logger` | 905 | 652 | zerolog + lumberjack + SSE log stream |
| `internal/config` | 626 | 59 | viper + env + hot reload |
| `internal/metrics` | 559 | – | Prometheus collectors |
| `internal/scheduler` | 505 | 360 | robfig/cron wrapper |
| other internal (`auth`, `events`, `proxy`, `announce`, `server`, `api`, `update`, `user`, …) | ~1,700 | | |
| **internal total** | **46,950** | **27,254** | |
| `pkg/arr/{sonarr,radarr,lidarr,readarr,whisparr,sportarr}` | 3,148 | 1,709 | *arr HTTP clients |
| `pkg/newznab`, `pkg/torznab` | 1,878 | 375 | feed + caps clients |
| `pkg/{btn,ggn,ops,red,ptp}` | 1,499 | 749 | tracker API clients (**`pkg/ptp` is unused** except by its own tests) |
| `pkg/migrator` | 727 | – | small embed-FS SQL migrator (SQLite + PG) |
| `pkg/{aria2,jsonrpc,nzbget,sabnzbd,porla,transmission}` | 1,150 | | lightweight client code |
| `pkg/{argon2id,errors,sanitize,sharedhttp,sqlite3store,version,wildcard,featureflags}` | ~1,240 | | utilities |
| **pkg total** | **9,660** | **3,602** | |
| `test/` (e2e harness, in-process ircd, mock indexer, mock discord) | 3,101 | 928 | |
| Indexer definitions YAML | 10,986 lines, 118 files (+16 tombstones) | | `internal/indexer/definitions/` |
| SQL migrations | 99 SQLite files (2,487 lines), 88 Postgres files | | `internal/database/migrations/` |
| Frontend `web/src` | ~30,800 TS/TSX, 159 files | + `web/test` (Vitest) | largest: `forms/settings/IndexerForms.tsx` 1.2k, `DownloaderForms.tsx` 1.1k |

**Total: about 57k lines of non-test Go** (backend plus the client code it bundles), **31k TS**, and **11k YAML definitions**. That is roughly **one-seventh the size of Sonarr's backend** (comparison.md §0).

### 1.4 Code quality assessment (for reuse)

**Strengths**
- Idiomatic, modern Go: `context` threaded through calls, structured zerolog fields, `errors.Wrap`, small consumer-side interfaces.
- High test ratio: 27k test lines against 47k in `internal`.
  - Build tags separate the heavier suites: `integration` (embedded Postgres), `irc_integration_test` (in-process ircd) and `e2e` (Playwright).
- `AGENTS.md` enforces style rules: constant log messages, no narration comments, migrations for both databases.
- The IRC layer is the most mature part (§3.1). It has explicit connection and channel state machines and tests for flapping, reconnect, invite failure, NickServ and recovery (`internal/irc/*_test.go`).
- Definitions carry their own regex test cases, which are executed in `internal/indexer/definition_test.go`.

**Weaknesses**
- **Fat domain.** `domain.Release` has about 90 fields, mixing announce, parser, torrent-file and runtime state (`internal/domain/release.go`). `domain.Filter` has about 80 columns with ad-hoc comma-separated string lists (`internal/domain/filter.go:108`).
- **Hand-written SQL everywhere.** Squirrel builders list every column. `database/filter.go` alone is 2k lines, and each filter field is repeated in the struct, the migration, the select, the scan, the update and the TS type.
- **Copy-paste in the *arr clients.** Each package in `pkg/arr/*` has an identical `client.go` (get, getJSON, postBody, …) and an identical `Push` error handling. Only the API version prefix and the payload fields differ.
- **Dead or unused fields.** Nothing evaluates `Filter.Scene`, `TagsAny`, `ExceptTagsAny` or `ExceptReleaseTypes`. They are stored and sanitized only; `CheckFilter` does not reference them. `Action.WebhookMethod`, `WebhookType` and `WebhookHeaders` are ignored: the webhook action always sends a POST with JSON and never checks the status (`internal/action/run_webhook.go:23`).
- **Probable bug in category exclusion.** The `ExceptCategories` check uses `contains(r.Category, …) && containsAny(categories, …)` (`internal/domain/filter.go:627`). A release that has only `Categories` (Torznab) and no `Category` is never excluded. The match side uses the opposite logic.
- **Filter delay is not durable.** It is a `time.Sleep` inside the processing goroutine (`internal/release/service.go:547`), so pending grabs are lost on restart.
- **Two major versions of the same libraries:**
  - `avast/retry-go` v3 and v4 are both imported.
  - The errors wrapper exists twice: `pkg/errors` alongside `github.com/pkg/errors`.
- **`sasha-s/go-deadlock` is used in production code**, not only in tests (`internal/irc/channel.go:86`).
- A few `//` commented-out code blocks and TODOs remain in older code.

**Verdict.** The pieces worth lifting nearly verbatim are the **IRC subsystem, announce parsing and the definition format, the download-client adapters, the Torznab/Newznab clients, and the migrator/SQLite tuning**. The **filter/action/release data model should be redesigned** so it fits the monolith's decision engine (§11).

**Key `go.mod` dependencies**

| Concern | Library |
|---|---|
| HTTP router | `go-chi/chi/v5`, `go-chi/render`, `rs/cors` |
| Sessions | `alexedwards/scs/v2` (+ `postgresstore`, custom `pkg/sqlite3store`) |
| OIDC/OAuth | `coreos/go-oidc/v3`, `golang.org/x/oauth2` |
| Password hash | custom `pkg/argon2id` (on `x/crypto`) |
| DB drivers | `modernc.org/sqlite` (pure Go), `lib/pq` |
| SQL builder | `Masterminds/squirrel` |
| IRC | `ergochat/irc-go` (`ircevent`) |
| Release parsing | `moistari/rls` **replaced by `autobrr/rls v0.8.1`** (fork) |
| Feeds | `mmcdole/gofeed` (RSS), custom `pkg/torznab`/`pkg/newznab` (encoding/xml) |
| Torrent clients | `autobrr/go-qbittorrent`, `autobrr/go-deluge`, `autobrr/go-rtorrent`, `hekmon/transmissionrpc/v3`, `autobrr/go-torrent` (metainfo) |
| Scheduling | `robfig/cron/v3` |
| Event bus | `maniartech/signals` |
| Config | `spf13/viper`, `spf13/pflag`, `fsnotify` |
| Logging | `rs/zerolog`, `natefinch/lumberjack.v2` |
| Templating | `text/template` + `Masterminds/sprig/v3` |
| Notifications | `nicholas-fedor/shoutrrr` (maintained fork of containrrr) |
| Metrics | `prometheus/client_golang` |
| Real-time UI | `r3labs/sse/v2` (autobrr fork) |
| Caching/maps | `autobrr/go-cache` (regexcache, ttlcache), `alphadose/haxmap` |
| Retry | `avast/retry-go` **v3 and v4** |
| Container tuning | `KimMachineGun/automemlimit`, `uber/automaxprocs` |
| Proxy | `golang.org/x/net/proxy` (SOCKS5) + custom HTTP CONNECT dialer (`internal/irc/httpproxy.go`) |
| Test only | `fergusstrange/embedded-postgres`, `mxschmitt/playwright-go`, `stretchr/testify` |

---

## 2. Domain model

Entities are in `internal/domain/*.go`. Tables come from `internal/database/migrations/{sqlite,postgres}/*.sql`.

| Entity | Table(s) | Key fields / notes |
|---|---|---|
| `IndexerDefinition` (YAML, not a table) | – | `version, name, identifier, description, language, urls[], privacy, protocol (torrent/nzb), supports[irc,rss,api,torznab,newznab], settings[] (name/type text\|secret/required/label/help/regex), irc{network,server,port,tls,auth.mechanism,settings[],channels[{name,announcers[],parse}]}`, `feed{minInterval,settings}` (`indexer.go:120`) |
| `Indexer` | `indexer` | `identifier, identifier_external` (name sent to *arrs), `implementation` (irc/torznab/newznab/rss), `base_url, settings` (JSON map; secret keys redacted in JSON, `indexer.go:44`), `use_proxy/proxy_id, archived/archived_at` |
| `IndexerDeprecation` | `indexer_deprecation` | tombstones from `definitions/deprecated/*.yaml` (identifier, reason, issue_url, alias_of, deprecated_at) |
| `IrcNetwork` | `irc_network` | `server, port, tls, tls_skip_verify, pass, nick, auth{mechanism NONE/SASL_PLAIN/NICKSERV, account, password}, invite_command, use_bouncer/bouncer_addr, use_proxy/proxy_id, bot_mode` (`irc.go`) |
| `IrcChannel` | `irc_channel` | `name, password, enabled, detached, monitoring state` |
| `Filter` | `filter`, `filter_indexer`, `filter_external`, `filter_notification`, `list_filter` | about 80 flat columns (§3.2). Arrays are stored as `TEXT[]` in both DBs (SQLite stores the text literal, PG a native array, via `pq.Array`). |
| `FilterExternal` | `filter_external` | `type EXEC\|WEBHOOK, exec_cmd/args/expect_status, webhook_host/method/data/headers/expect_status/retry_status/retry_attempts/retry_delay_seconds, on_error CONTINUE\|REJECT, index` (ordered) |
| `Action` | `action` | `type` (§4), `client_id`, `enabled`, qBit/Deluge/rTorrent options (`save_path, download_path, category, tags, label, paused, ignore_rules, first_last_piece_prio, skip_hash_check, content_layout, limit_up/down/ratio/seed_time, priority`), reannounce (`skip/delete/interval/max_attempts`), `exec_cmd/args`, `watch_folder`, `webhook_*`, `external_download_client_id/name` (for *arr push) (`action.go`) |
| `Downloader` ("Client") | `client` | `type` (§4), `host, port, tls, tls_skip_verify, username, password`, `settings{apikey, auth{type NONE/BASIC/DIGEST}, rules{max_active_downloads, ignore_slow_torrents(+condition), download/upload_speed_threshold}, external_download_client_id/name}` (`downloader.go`) |
| `Release` | `release` | Announce data (indexer, torrent_id, group_id, info/download URL, size, category, uploader, freeleech %, bonus, origin, tags, announce_type NEW/CHECKED/PROMO/PROMO_GP/RESURRECTED) plus rls-parsed fields (title, sub_title, season, episode, year/month/day, resolution, source, codec[], container, hdr[], audio[], group, proper, repack, hybrid, edition[], cut[], language[], website, other[]). Also `normalized_hash` (md5 of normalized name), `filter_status`, and `filter` / `filter_id`. **Only releases that matched a filter with actions are stored** (`release/service.go:512`). |
| `ReleaseActionStatus` | `release_action_status` | `status PENDING/PUSH_APPROVED/PUSH_REJECTED/PUSH_ERROR, action, action_id, type, client, filter, filter_id, rejections[], timestamp`. Used by duplicate and smart-episode checks. |
| `DuplicateReleaseProfile` | `release_profile_duplicate` | about 25 booleans choosing which parsed fields must match a prior PUSH_APPROVED release (protocol, release_name, hash, title, sub_title, year/month/day, source, resolution, codec, container, dynamic_range, audio, group, season, episode, website, proper, repack, edition, hybrid, language) (`release.go`) |
| `ReleaseCleanupJob` | `release_cleanup_job` | `schedule` (cron), `older_than` hours, `indexers`, `statuses`, last/next run |
| `Feed` | `feed`, `feed_cache` | `type TORZNAB/NEWZNAB/RSS, url, api_key, cookie, user_agent, interval, timeout, max_age, categories[], capabilities, tls_skip_verify, settings, last_run/last_run_data`. `feed_cache` holds GUID-seen entries with a TTL (cleaned by a daily job). |
| `List` | `list`, `list_filter` | `type` (§4), `client_id` (for *arr lists), `url, headers, api_key, match_release, tags_included/excluded, include_unmonitored, include_alternate_titles, include_year, skip_clean_sanitize, last_refresh_*`. A list **overwrites fields on its linked filters** (`FilterUpdate`: shows / match_releases / albums / artists / years). |
| `Notification` | `notification` | `type` (§4), per-type credentials, `events[]`. Filters can override the events per filter (`filter_notification`). |
| `Proxy` | `proxy` | `type` (HTTP / SOCKS5), `addr, user, pass, timeout`. Used by IRC, feeds, tracker APIs and torrent-file downloads. |
| `User` | `users` | username + argon2id hash (single-user onboarding) |
| `APIKey` | `api_key` | name, key, scopes (unused) |
| sessions | `sessions` | scs store (migration 74) |

**DB layer**
- `internal/database/database.go` wraps `*sql.DB`. The driver is chosen from the DSN prefix or from `databaseType`.
- Squirrel uses `sq.Dollar` placeholders for both engines. `ILike()` hides the SQLite/PG difference.
- Slow queries (500 ms or more) are logged.
- **SQLite tuning** (`database/sqlite.go:31`): `busy_timeout=10000`, WAL, `synchronous=NORMAL`, `mmap_size=256MB`, `cache_size=-64000`, `analysis_limit=400`, `foreign_keys=ON`, `PRAGMA optimize` on close.
- **Before migrating**, it runs an integrity check (`PRAGMA integrity_check`), reindexes if needed, backs up (keeping `DatabaseMaxBackups`) and prunes old backups (`sqlite.go:163-330`).

**Migrations**
- `pkg/migrator` applies files from an `embed.FS`, with a `schema_migrations` table and a pre-migration hook (used for the backup).
- Each engine keeps a separate numbered sequence: SQLite `0_base_schema.sql` … `97_add_filter_max_downloads_period.sql`, and PG `0_base_schema_postgres.sql` … `87_…`. The numbers differ per engine.
- `migrations/codegen/generate_migrations.go` (`//go:build generate`) produces the Go registration.
- SQLite changes use the `*_dg_tmp` table-rebuild pattern (DataGrip style).
- `autobrrctl db:convert` copies a SQLite database to Postgres (`internal/database/tools/convert.go`).

---

## 3. Feature areas

### 3.1 IRC announce ingestion

**Connection** (`internal/irc/handler.go`, 2.2k LOC)
- Built on `ergochat/irc-go/ircevent`, with one `Handler` per network.
- **Auth:**
  - Server password.
  - SASL PLAIN (`handler.go:363`, SASLOptional).
  - NickServ IDENTIFY, with a failed-attempt counter and a separate test file.
  - Channel keys.
  - **Invite commands:** comma-separated `/msg` commands sent to a bot (e.g. `Satsuki enter #announce nick key`), with states for waiting on the invite and handling the invite-bot response and timeouts (`channel_state_machine.go`).
- **Transport:**
  - TLS, optionally skipping verification.
  - Bouncer address override (`handler.go:268`, e.g. ZNC/soju).
  - Per-network proxy: SOCKS5 through `x/net/proxy`, or HTTP CONNECT through a custom dialer (`httpproxy.go`).
  - Optional `bot_mode` (+B).
- **Resilience:**
  - A connection state machine (`state_machine.go`): connecting → connected → authenticated → joining → operational/healthy.
  - A per-channel state machine: join, invite, retry, part/kick handling.
  - Reconnect with retry-go, and flapping detection.
  - Proxy change events restart affected networks (`service.go:110`).
- **Callbacks:** MODE, INVITE, PART, PRIVMSG, NOTICE, NICK, KICK, JOIN.
- **Observability:**
  - Message history per channel (ring buffer) is exposed at `GET /api/irc/network/{id}/channel/{ch}/history`.
  - A live `irc` SSE stream.
  - A manual raw command endpoint (`POST /api/irc/network/{id}/cmd`).
- Announcer matching uses the definition's `announcers` list. An optional **fuzzy announcer** feature flag (`IRC_FUZZY_ANNOUNCER`, `internal/irc/channel.go:212`) tolerates nick variations.

**Announce pipeline** (`internal/announce/announce.go`)
1. Each indexer gets an `announceProcessor` holding one buffered `chan string` (128) per channel. The IRC handler queues matching PRIVMSG lines into it.
2. A consumer goroutine per channel reads **N consecutive lines** for a `multi` parse (N = number of `lines`), or one line for `single`.
3. It runs each line's regex, then builds a `Release`. `Parse` → `MapVars` → template URLs → `rls.ParseString`.
4. It calls `go releaseSvc.Process(ctx, rls)`.

A `trace_id` is attached per announce.

**Definition YAML format** (`internal/indexer/definitions/*.yaml`, loaded from the embedded FS in `internal/indexer/definitions.go`, plus `CustomDefinitions` directory from config)

```yaml
version: 2
name / identifier / description / language / urls[] / privacy / protocol
supports: [irc, rss, api]
settings: [{name, type: text|secret, required, label, help, regex}]  # e.g. rsskey, passkey, authkey, torrent_pass
irc:
  network, server, port, tls
  auth: {mechanism: NICKSERV|SASL_PLAIN|NONE}
  settings: [nick, auth.account, auth.password, invite_command, ...]
  channels:
    - name: "#announce"
      announcers: [BotNick]
      parse:
        type: single|multi
        forcesizeunit: ""          # optional
        skipcleanmessage: false    # optional (strip IRC color codes by default)
        lines:
          - pattern: <regex with named groups>   # or legacy `vars: [..]` positional list
            ignore: false
            tests: [{line, expect: {var: value}}] # executed in definition_test.go
        match:
          infourl: "/torrent/{{ .torrentId }}"   # Go text/template + sprig; joined to baseUrl var
          downloadurl: "/rss/download/{{ .torrentId }}/{{ .rsskey }}/{{ .releaseName }}.torrent"
          torrentname / magneturi
          encode: [releaseName]                  # url-escape these vars first
        mappings: {varName: {value: {otherVar: newValue}}}  # e.g. category code → name
```

- **Recognised variables** (`domain/release.go:1086 MapVars`): releaseName/torrentName, torrentHash, torrentId, category, subCategory, announceType, freeleech, freeleechPercent, uploadVolumeFactor, uploader, recordLabel, torrentSize, torrentSizeBytes, scene, origin, internal, year, tags, title, author, releaseTags, resolution, releaseGroup, releaseEpisode, language, container, imdb, tmdb/tmdbid. Other variables stay in `RawVars` and can be used in templates.
- **Tracker-specific Go parsers** exist for three indexers (`domain/irc_parser.go`): GazelleGames, Orpheus and Redacted, which handle music and game tag layouts.
- **Parse types in use:** 107 `single`, 8 `multi` (alpharatio, btn, hebits, homiehelpdesk, pixelhd, torrentnetwork, tokyotoshokan, trancetraffic).
- 17 definitions use `mappings`. 10 declare `auth.mechanism: NICKSERV`.
- **Source of definitions:**
  - Community PRs to this repo: adding a tracker usually touches one YAML file.
  - Removing one requires a tombstone in `definitions/deprecated/` (CI enforced, `CONTRIBUTING.md`).
  - Definitions ship **in the binary**; there is no remote definition feed like Prowlarr's `indexers.prowlarr.com`.
  - The `update-indexers.yml` workflow only syncs the list to the docs site.
- `POST /api/release/process` (`ReleaseProcessReq{indexer_identifier, indexer_implementation, announce_lines[]}`) injects announce lines without IRC. It is used for testing and by external relays.

### 3.2 Filter engine

**Where it runs.** `domain.Filter.CheckFilter` (`internal/domain/filter.go:449`) does the pure matching. `filter.Service.CheckFilter` (`internal/filter/service.go:576`) adds the DB and network checks.
- Filters load **per release** by indexer identifier, ordered by `priority` (`FindByIndexerIdentifier`).
- The first filter that matches and has an accepted action wins.
- If an action returns *arr rejections, the pipeline tries the next filter, skipping action clients it already tried (`release/service.go:467`).

**Matching semantics**
- String lists are comma-separated with wildcard support (`*`, `?`, via `pkg/wildcard`). Matching is case-insensitive.
- `use_regex` switches `match_releases`/`except_releases` to regex (`regexcache`).
- Every failed check adds to `RejectReasons` (field, got, expected), which is logged and shown in the UI.

**Every filter field**

| Group | Fields |
|---|---|
| General | `name, enabled, priority, delay` (seconds, sleep before first action), `min_size, max_size` (humanized, e.g. "10 GB"), `max_downloads` + `max_downloads_unit` (MINUTE/HOUR/DAY/WEEK/MONTH/EVER) + `max_downloads_period` + `max_downloads_window_type` (FIXED/ROLLING), `announce_types[]`, `indexers[]` |
| Release names | `match_releases, except_releases, use_regex, match_release_groups, except_release_groups, match_release_tags, except_release_tags, use_regex_release_tags, match_description, except_description, use_regex_description` |
| Tracker metadata | `freeleech, freeleech_percent` (ranges like "50,75-100"), `bonus[]`, `origins[], except_origins[]` (P2P/Internal/SCENE/O-SCENE), `match_categories, except_categories, match_uploaders, except_uploaders, tags, except_tags, tags_match_logic, except_tags_match_logic` (ANY/ALL), `min/max_seeders, min/max_leechers` (feeds only) |
| TV / Movies | `shows` (title list), `seasons, episodes` (int ranges "1,3-5"), `smart_episode` (only newer than already-grabbed), `years, months, days`, `resolutions[], codecs[], sources[], containers[], match_hdr[], except_hdr[], match_other[], except_other[], match_language[], except_language[]` |
| Music | `artists, albums, match_release_types[]` (Album/Single/EP/…), `formats[]` (MP3/FLAC/…), `quality[]` (192/320/V0/Lossless/24bit…), `media[]` (CD/WEB/Vinyl/…), `perfect_flac` (bundle: FLAC+Lossless/24bit+Log100+Cue+CD), `cue, log, log_score, match_record_labels, except_record_labels` |
| Duplicates | `release_profile_duplicate_id` → `DuplicateReleaseProfile` (§3.6) |
| External | `external[]` (EXEC: command, args with macros, expected exit status; WEBHOOK: host, method, templated body, headers, expected status, retry statuses/attempts/delay; on_error CONTINUE/REJECT) (`filter/service.go:946`) |
| Notifications | per-filter `notifications[]` (notification id + events) |
| **Dead fields** (stored, never evaluated) | `scene`, `tags_any`, `except_tags_any`, `except_release_types` |

**Additional checks that use the network**
- If the announce lacks size, uploader or record label and the filter needs them (`AdditionalSizeCheckRequired` etc.), autobrr fetches them:
  - through the **tracker API** for btn, ggn, red and ops (`internal/indexer/api.go`), or
  - by **downloading the .torrent** and reading its metainfo (`filter/service.go:731`).
- `max_downloads` counts PUSH_APPROVED rows over the window (`database/filter.go`).

### 3.3 Actions

`internal/action/run.go` handles all action types in common. Macros are expanded first (`domain/macros.go`: about 80 template variables such as `{{ .TorrentName }}`, `{{ .TorrentPathName }}`, `{{ .TorrentHash }}`, `{{ .Indexer }}`, `{{ .Resolution }}`, `{{ .CurrentYear }}`, plus sprig functions). The .torrent is downloaded **only if needed** (`release/download_service.go`: cookie support, 3 retries, Retry-After handling, magnet links not downloadable). The per-type logic lives in one file each:

| Action | File | Behaviour |
|---|---|---|
| TEST | run.go | no-op |
| EXEC | `run_exec.go` | run a command with macro args (`go-shellquote`) |
| WATCH_FOLDER | `run_watchfolder.go` | write the .torrent (or magnet) to a folder |
| WEBHOOK | `run_webhook.go` | POST JSON macro body. **Ignores method/headers; does not check the status.** |
| QBITTORRENT | `run_qbittorrent.go` | add torrent/magnet with options. **Client-side rules** (max active downloads, ignore slow torrents by speed threshold) are checked first. **Reannounce loop** (`ReannounceTorrentWithRetry`) for trackers that are not yet registered: interval, max attempts, delete on failure. |
| DELUGE_V1 / DELUGE_V2 | `run_deluge.go` | via `autobrr/go-deluge` RPC; label plugin, max-active rule |
| RTORRENT | `run_rtorrent.go` | XML-RPC, label, paths, `d.*` options |
| TRANSMISSION | `run_transmission.go` | `transmissionrpc/v3`, labels, reannounce |
| PORLA | `run_porla.go` | JSON-RPC |
| ARIA2 | `run_aria2.go` | JSON-RPC |
| SABNZBD / NZBGET | `run_sabnzbd.go`, `run_nzbget.go` | add NZB by URL, category, priority |
| SONARR / RADARR / LIDARR / READARR / WHISPARR / WHISPARR_V3 / SPORTARR | `run_<arr>.go` | `POST release/push`. Rejections are returned to the pipeline and stored as PUSH_REJECTED (§9). |

### 3.4 RSS / Torznab / Newznab feeds

`internal/feed/` has `TorznabJob`, `NewznabJob` and `RSSJob`.
- **Scheduling:** each feed is scheduled with `ScheduleJobAnchored`, so the interval is anchored to `last_run` (`feed/service.go:690`). The minimum interval comes from the definition's `feed.minInterval`.
- **Fetching:**
  - Torznab and Newznab call `t=search` with categories and no query, which gives the latest items (`pkg/torznab/torznab.go:163`). `t=caps` is fetched and stored.
  - RSS uses gofeed with enclosure and size heuristics (`feed/rss.go`).
- **Deduplication:** items are compared by GUID against `feed_cache`, and `max_age` skips old items.
- **Handoff:** new items become `Release` objects (rls-parsed, with Torznab attributes for seeders, leechers and downloadvolumefactor→freeleech %) and go to `release.ProcessMultipleFromIndexer`.
- Feeds only **poll**; autobrr never searches.
- Generic definitions: `generic_torznab.yaml`, `generic_newznab.yaml` and `generic_rss.yaml` (the only three definitions without `irc:`). A Torznab feed can point at Prowlarr or Jackett.

### 3.5 Release parsing

- **Library:** `github.com/moistari/rls`, **replaced by the fork `github.com/autobrr/rls v0.8.1`** (`go.mod` replace directive). `rls.ParseString(title)` returns `Type` (episode/series/movie/music/game/app/book/…), Title, Subtitle, Series, Episode, Year/Month/Day, Resolution, Source, Codec, HDR, Audio, Channels, Container, Group, Edition, Cut, Language, Region, Other and more (`domain/release.go:688`).
- **autobrr extras on top:**
  - `ParseReleaseTagsString` / `releasetags.go` (576 LOC) parses tracker "release tags" strings such as `FLAC / Lossless / Log / 100% / Cue / CD`.
  - `extraParseSource` normalizes sources.
  - `NormalizedHash` is the md5 of the normalized name (`golang.org/x/text` transform).
- rls is a lexer/tag-based parser covering all media types. It is a **different engine from Sonarr/Radarr's regex cascades**, and its output (e.g. `Source: "WEB-DL"`, `Resolution: "2160p"`) does not map one-to-one onto the Sonarr/Radarr Quality enum.

### 3.6 Duplicate and upgrade handling

- **Duplicate release profiles** (`database/release.go` `CheckIsDuplicateRelease`) build a query over previously **PUSH_APPROVED** releases. Each enabled boolean adds a WHERE condition (title ILIKE, season/episode equality, resolution, source, codec, HDR, audio, group, proper, repack, edition/cut, hybrid, language, website, and so on). Built-in profiles cover common cases, e.g. "exact release", "TV episode", "movie". This is a **coarse substitute** for an *arr's "already have a better file" logic, and it has **no concept of quality ranking**.
- **Smart episode** (`CheckSmartEpisodeCanDownload`) rejects a release unless its season/episode (or date for daily shows) is **newer** than every approved grab with the same title prefix. Proper/repack from the same group is allowed.
- **Real upgrade logic** (cutoffs, custom-format scores) is **delegated to the *arr** through `release/push`. The *arr's rejection strings come back and the pipeline tries the next filter.

### 3.7 Lists (`internal/list/`)

- Lists run on a 6-hour cron and can also be triggered manually or by webhook.
- Each run fetches titles and **rewrites the linked filters**: `shows` / `match_releases` for TV and movies, `artists` / `albums` for music. Titles are sanitized (`title.go`) and optionally include year and alternate titles.
- **Sources:**
  - *arr libraries, i.e. monitored items with include/exclude tags: Sonarr, Radarr, Lidarr, Readarr, Whisparr v2 and v3, Sportarr.
  - External lists: MDBList, Trakt (list/watchlist/smart list through `api.trakt.tv`, `list/provider/trakt/trakt.go`), Metacritic, Steam wishlist, AniList, and plaintext files or URLs.
  - **Curated presets hosted at `https://api.autobrr.com/lists/...`**: trakt anticipated/popular TV, upcoming movies/bluray, StevenLu, Metacritic albums, AniList seasonal/trending/upcoming (`web/src/domain/constants.ts:1018`).

### 3.8 Notifications (`internal/notification/`)

- **Services:** Discord, Notifiarr, Pushover, Slack (via webhook), Telegram, Gotify, ntfy, LunaSea, Shoutrrr (a generic URL that covers dozens of services), and a generic Webhook.
- **Events:** `PUSH_APPROVED, PUSH_REJECTED, PUSH_ERROR, IRC_DISCONNECTED, IRC_RECONNECTED, APP_UPDATE_AVAILABLE, RELEASE_NEW, TEST`.
- Each filter can override which events it notifies on.
- `message_builder.go` builds rich messages that include the release's parsed fields.

### 3.9 Auth / OIDC

- **Accounts:** a single local user created at first-run onboarding (`POST /api/auth/onboard`). Passwords are hashed with argon2id (`pkg/argon2id`).
- **Sessions:** `scs` sessions stored in the DB (a custom SQLite store and scs postgresstore). The cookie is `user_session`.
- **OIDC** (`internal/auth/oidc.go`, `internal/http/oidc.go`):
  - discovery, PKCE when the provider supports it, a state cookie, and `userinfo`;
  - `oidcDisableBuiltInLogin` hides the local login.
  - A failed OIDC discovery at startup is fatal (`main.go`).
- **API keys:** `X-API-Token` header or `?apikey=` query (`internal/http/middleware.go:73`). Keys are managed through `/api/keys`, and there are no real scopes.
- `autobrrctl` can create a user, change a password and generate an htpasswd entry for the metrics basic auth.

### 3.10 Other features

- **Proxies:** HTTP/SOCKS5 per IRC network, feed and indexer.
- **Release history UI:** stats, heatmap, top indexers and top filters (`/api/release/stats/*`).
- **Release cleanup jobs:** cron-scheduled deletion of old releases by indexer and status.
- **Filter import/export** as JSON, **plus an importer for autodl-irssi config** (`web/src/screens/filters/_configParser.ts`).
- **Regex playground** screen.
- **Log viewer and download** (`/api/logs/files`) and a live log SSE stream.
- **Update check only:** it queries `api.autobrr.com/repos/autobrr/autobrr/releases/latest` every 2 hours and does not self-update.
- **Prometheus metrics:** a separate server with basic auth, exposing counters and gauges for feeds, filters, IRC, lists and releases (`internal/metrics/collector/`).
- **pprof profiling server** and a PGO build flag (`--pgo`).

---

## 4. Integrations (full lists)

**Indexer definitions: 118 YAML files** (`internal/indexer/definitions/`) plus 16 deprecation tombstones.
- 115 support IRC, 102 RSS and 4 a tracker API (btn, gazellegames, orpheus, red).
- 117 are torrent and 1 is nzb. 114 are private and 4 public (nyaa, subsplease, t66y, tokyotoshokan).
- 3 are generic: `generic_torznab`, `generic_newznab`, `generic_rss`.
- Named definitions: abnormal, acidlounge, aither, alpharatio, animebytes, animeworld, ant, aura4k, bemaniso, beyondhd, bit-hdtv, bithumen, bitsexy, bjshare, brokenstones, btfiles, btn, capybarabr, cathode-ray-tube, czteam, danishbits, danishbytes, darkpeers, dasunerwartete, digitalcore, docspedia, emp, enthralled, f1carreras, fappaizuri, filelist, finelite, funfile, fuzer, gazellegames, happyfappy, hd-space, hd-torrents, hdb, hdonly, hebits, homiehelpdesk, huno, immortalseed, infinityhd, insane, iplay, iptorrents, itatorrents, keepfrds, locadora, lst, luminarr, materialize, midnightscene, milkie, myanonamouse, ncore, nebulance, norbits, nordicbytes, nordicquality, nyaa, onlyencodes, orpheus, pbay, peergarden, phoenixproject, pixelhd, polishtracker, privatesilverscreen, ptfiles, ptm, ptp, pussytorrents, rastastugan, red, reelflix, retroflix, retromoviesclub, retrotoonworld, revolutiontt, rocket-hd, samaritano, satclubbing, scenehd, seedcore, seedpool, seedpool_music, shareisland, sharewood, simurg, skipthecommercials, speedapp, subsplease, sugoimusic, superbits, t66y, thedarkcommunity, theoldschool, tokyotoshokan, torrentbytes, torrentday, torrenthr, torrentleech, torrentnetwork, torrentsyndikat, trancetraffic, ultrabits, uploadcx, vietmediaf, xspeeds, xwt, yu-scene, zenith.
- Tombstoned: fnp, ianon, lillesky, lusthive, morethantv, nexum, oppaitime, polishsource, ptn, stt, tfm, torrentdb, torrentseeds, torrentseeds-music, tsc, uhdbits.
- **Tracker API clients** (`pkg/`): btn (JSON-RPC), ggn, ops, red, plus ptp (**unused**).

**Download clients / action targets** (`domain/downloader.go:178`, `domain/action.go:160`)

| Client | Library | Status |
|---|---|---|
| qBittorrent | `autobrr/go-qbittorrent` | keep (the most complete adapter: rules, reannounce, content layout) |
| Deluge v1 | `autobrr/go-deluge` | **obsolete**: Deluge 1.x has been dead since 2019 |
| Deluge v2 | `autobrr/go-deluge` | keep |
| rTorrent | `autobrr/go-rtorrent` | keep |
| Transmission | `hekmon/transmissionrpc/v3` | keep |
| Porla | `pkg/porla` | niche, candidate to drop |
| Aria2 | `pkg/aria2` | keep (also in the *arrs) |
| SABnzbd | `pkg/sabnzbd` | keep |
| NZBGet | `pkg/nzbget` | keep |
| Watch folder | – | keep (the *arrs' "blackhole") |
| Sonarr, Radarr, Lidarr | `pkg/arr/*` | become in-process in a monolith (§9) |
| Readarr | `pkg/arr/readarr` | **obsolete**: Readarr was retired upstream (Bookshelf replaces it) |
| Whisparr v2 / v3 (Eros) | `pkg/arr/whisparr` | in-process; the v2/v3 split is legacy |
| Sportarr | `pkg/arr/sportarr` | third-party *arr, out of scope |

Missing compared with the *arrs: Flood, rqbit, DownloadStation, Freebox, Tribler, NZBVortex, uTorrent and Hadouken. The last two are obsolete anyway.

**Notifications:** Discord, Notifiarr, Pushover, Slack, Telegram, Gotify, ntfy, LunaSea (**obsolete**: the LunaSea app is effectively unmaintained), Shoutrrr (catch-all), Webhook.

**Lists:** the *arr libraries (Radarr, Sonarr, Lidarr, Readarr (obsolete), Whisparr, Whisparr v3, Sportarr) plus MDBList, Metacritic, Plaintext, Trakt, Steam, AniList. There are also the api.autobrr.com presets, including StevenLu.

---

## 5. External APIs / interfaces

- **REST API** under `{baseUrl}/api` (chi, JSON), from `internal/http/server.go:188`:
  - `/auth` (login, logout, onboard, validate, `/oidc/*`, `PATCH user/{username}`);
  - `/healthz/{liveness,readiness}`;
  - `/actions`, `/config` (GET/PATCH, hot-reloadable settings), `/downloaders` (+`/test`, `/{id}/arr/tags`);
  - `/filters` (+`/{id}/duplicate`, `/enabled`, `/notifications`, `/external/test`, `/indexers/prune-deprecated`);
  - `/feeds` (+`/test`, `/caps`, `/{id}/latest`, `/forcerun`, `/cache`);
  - `/irc` (networks, channels, `/cmd`, `/restart`, history, `/channel/{ch}/announce/process`);
  - `/indexer` (`/schema` returns all definitions, `/options`, `/deprecations`, `/{id}/api/test`);
  - `/lists` (+`/refresh`);
  - `/keys`, `/logs/files`, `/notification` (+`/test`, `/pushover/sounds`), `/proxy` (+`/test`, `/usage`);
  - `/release` (list, recent, stats×6, indexers, `DELETE`, **`POST /process`**, `/{id}/actions/{statusId}/retry`, `/profiles/duplicate`, `/cleanup-jobs`);
  - `/updates/{latest,check}`;
  - **`/webhook/lists/trigger[/arr|/lists|/{listID}]`** (GET/POST). *arr "On Import/Upgrade" webhooks point here to refresh lists.
  - `/events` is the SSE stream (`logs`, `irc`).
- **Auth:** session cookie, `X-API-Token` header or `?apikey=` query. `/auth` and `/healthz` are public.
- **Web UI** is served from the embedded `web/dist`, with `baseUrl` and `baseUrlModeLegacy` (`internal/http/web_legacy.go`).
- A separate **metrics server** serves `/metrics` (optional basic auth, `metricsBasicAuthUsers` in htpasswd format).
- **Outbound calls:**
  - *arr APIs (§9) and torrent/usenet client APIs;
  - tracker download/API URLs and IRC servers;
  - `api.autobrr.com` (update check and curated lists), `api.trakt.tv`, MDBList, Steam, Metacritic, AniList;
  - notification endpoints and the OIDC issuer.
- **Outbound webhooks:** the WEBHOOK action, the WEBHOOK external filter and the Webhook notification service.

---

## 6. Background jobs

The scheduler is `internal/scheduler/service.go`: robfig/cron with `Recover` and `SkipIfStillRunning`, keyed by identifier so a job can be replaced.

| Job | Schedule | Source |
|---|---|---|
| App update check | every 2 h (if `checkForUpdates`) | `scheduler/service.go:66` |
| Temp dir cleanup (downloaded .torrent temp files owned by the process user) | `0 4 * * *` | `scheduler/jobs.go`, `ownership.go` |
| Feed refresh, one per feed | every `interval` minutes, anchored to `last_run` | `feed/service.go:699` |
| Feed cache cleanup | `5 3 * * *` | `feed/service.go:808` |
| List refresh (all lists) | `0 */6 * * *` | `list/service.go:295` |
| Release cleanup jobs (user-defined) | user cron | `release/service.go:825` |
| SQLite WAL/progress heartbeat | 30 s ticker during long operations | `database/sqlite.go:148` |

These do **not** go through the scheduler:
- **Long-running goroutines:** one IRC handler per network, one announce consumer per channel, and one release-processing goroutine per announce, which includes the in-memory **filter delay sleep**.
- **Reannounce retries** run inline inside the qBittorrent/Transmission action.

There is no persisted command queue. A crash loses in-flight delayed grabs and reannounce loops.

---

## 7. Configuration surface

**`config.toml`** in the config dir, set by `--config`; the file is written with defaults on first run (`internal/config/config.go`).
- Env overrides use the `AUTOBRR__<KEY>` prefix. `AUTOBRR__<KEY>_FILE` reads the value from a file (secrets), `config.go:455`.
- **Hot reload:** viper `WatchConfig` reloads the log level and some other settings (`config.go:523`).

**Keys** (`domain.Config`)
- Server: `host` (127.0.0.1), `port` (7474), `baseUrl`, `baseUrlModeLegacy`, `corsAllowedOrigins`.
- Logging: `logLevel`, `logPath`, `logMaxSize`, `logMaxBackups`.
- Other: `customDefinitions` (directory of extra YAML definitions), `checkForUpdates`.
- Database: `databaseType` (sqlite/postgres), `databaseDSN`, `databaseMaxBackups`, `databaseAutoMigrate`, `postgresHost/Port/Database/User/Pass/SSLMode/Socket/ExtraParams`.
- Profiling: `profilingEnabled/Host/Port`.
- OIDC: `oidcEnabled/Issuer/ClientId/ClientSecret/RedirectUrl/Scopes/DisableBuiltInLogin`.
- Metrics: `metricsEnabled/Host/Port/BasicAuthUsers`.

**In the DB (UI-managed):** indexers and their settings, IRC networks, filters, actions, download clients, feeds, lists, notifications, proxies, API keys, duplicate profiles and cleanup jobs.

**Feature flags:** `pkg/featureflags`, registered in `main.go` init. The only one today is `IRC_FUZZY_ANNOUNCER`.

---

## 8. Platform/infrastructure concerns

- **Single static binary** (`CGO_ENABLED=0`), plus `autobrrctl`.
  - Docker: `alpine:3.24` with `/config` as the volume and port 7474.
  - `distrib/docker/ci*.Dockerfile` also builds a Windows container.
- **Windows** is supported. Container users (`ContainerAdministrator`) are detected in `config.go:190`, and `scheduler/ownership_windows.go` handles file ownership there.
- **Container tuning:** automaxprocs and automemlimit (cgroup aware).
- **Process-local state:** IRC connections, announce queues and the scheduler are in-memory singletons, so **only one instance can run**. There is no HA or leader election; Postgres support does not make it horizontally scalable.
- **Time-sensitive hot path.** An announce goes through regex → rls → filter DB load → optional .torrent download → client add. Speed matters here: the goroutine-per-release design and the regex/template caches (`regexcache`, `sync.Map` of templates, a precomputed sprig map, `domain/indexer.go:560`) exist for it.
- **Temp files:** downloaded .torrent files go to OS temp and are cleaned per release and by a daily job.
- **Secrets are stored in plaintext in the DB.** They are redacted in API responses (`IsSecretIndexerSetting`, `RedactString`), and updates must echo the redacted value back.
- **CI** (`.github/workflows`): golangci-lint, eslint, CodeQL, e2e (Playwright), release (goreleaser-style), weekly toolchain update, and indexer-list docs sync.

---

## 9. Interactions with other *arr apps

**Outbound calls from autobrr** (all `pkg/arr/<app>/client.go`)
- Auth: `X-Api-Key: <key>`, plus optional HTTP basic auth (`BasicAuth`, `Username`, `Password`) for reverse proxies.
- Base path: `{host}/api/v3/` for Sonarr, Radarr, Whisparr and Sportarr; `{host}/api/v1/` for Lidarr and Readarr.

| App | Test | Push | List/lookup | Tags |
|---|---|---|---|---|
| Sonarr | `GET /api/v3/system/status` | **`POST /api/v3/release/push`** | `GET /api/v3/series` | `GET /api/v3/tag` |
| Radarr | `GET /api/v3/system/status` | **`POST /api/v3/release/push`** | `GET /api/v3/movie` | `GET /api/v3/tag` |
| Lidarr | `GET /api/v1/system/status` | **`POST /api/v1/release/push`** | `GET /api/v1/album`, `GET /api/v1/artist/{id}` | `GET /api/v1/tag` |
| Readarr | `GET /api/v1/system/status` | **`POST /api/v1/release/push`** | `GET /api/v1/book` | `GET /api/v1/tag` |
| Whisparr v2 | `GET /api/v3/system/status` (major-version check) | **`POST /api/v3/release/push`** | `GET /api/v3/series` | `GET /api/v3/tag` |
| Whisparr v3 (Eros, Radarr-based) | same | same | `GET /api/v3/movie` | same |
| Sportarr | `GET /api/v3/system/status` | `POST /api/v3/release/push` | `GET /api/v3/leagues` | `GET /api/v3/tag` |

**`release/push` payload** (`pkg/arr/sonarr/types.go` `ReleasePushRequest`):

```json
{
  "title": "<TorrentName>",
  "infoUrl": "...", "downloadUrl": "...", "magnetUrl": "...",
  "size": 123456789,
  "indexer": "<IdentifierExternal or Identifier>",
  "downloadProtocol": "torrent|usenet", "protocol": "torrent|usenet",
  "publishDate": "<now RFC3339>",
  "downloadClientId": 0, "downloadClient": "<name>",
  "indexerFlags": 0,
  "imdbId": "tt...",
  "tmdbId": 0
}
```

The fields vary by app:
- `indexerFlags` is sent to Sonarr and Radarr only.
  - Sonarr bits: Freeleech 1, Halfleech 2, DoubleUpload 4, Internal 8, Scene 16.
  - Radarr bits: G_Freeleech 1, G_Halfleech 2, G_DoubleUpload 4, PTP Golden 8, PTP Approved 16, G_Internal 32, G_Scene 128.
  - They are computed from `FreeleechPercent` and `Origin` (`sonarr.BuildIndexerFlags`).
- `imdbId` goes to Sonarr and Radarr, and `tmdbId` to Radarr only.
- Lidarr, Readarr and Whisparr omit the flags and IDs. Sportarr omits the download-client fields.

**Response handling**
- `200` returns `[ { approved, rejected, temporarilyRejected, rejections[] } ]`. A non-empty `rejections` becomes PUSH_REJECTED, and the pipeline tries the next filter.
- `400` returns `[ { propertyName, errorMessage, errorCode, attemptedValue, severity } ]`.
  - Errors on `DownloadClient` or `DownloadClientId` are treated as configuration errors (PUSH_ERROR).
  - Other 400s are returned as rejections.

**Inbound calls from the *arrs:** only the optional *arr "Webhook" connection pointing at `POST /api/webhook/lists/trigger/arr`. It refreshes list-driven filters after an import.

**Calls to Prowlarr or Jackett:** only when a user adds them as a generic Torznab/Newznab feed (§3.4).

**What becomes in-process in the monolith**
- **An IRC announce stream is just another real-time release source.** Each announce becomes a `ReleaseInfo` entering the **same decision engine** as RSS sync and search results. The `release/push` HTTP hop, `indexerFlags` bit mapping, `downloadClientId` pinning and rejection string parsing all go away. In their place: `decisionEngine.Evaluate(release)`, then grab.
- The *arr-library lists (§3.7) exist only so autobrr filters know what the *arrs monitor. **In a monolith, "monitored and wanted" is a direct query**, and a filter can say "match anything the library wants" instead of copying title lists every 6 hours.
- Whisparr v2/v3 version detection, per-app API prefixes and the six `pkg/arr` clients all disappear.
- The webhook-triggered list refresh becomes an internal event: "library changed".

---

## 10. Outdated / legacy candidates

| Item | Where | Why |
|---|---|---|
| Deluge v1 client type | `DownloaderTypeDelugeV1` | Deluge 1.x is end-of-life |
| Readarr action/list | `pkg/arr/readarr`, `run_readarr.go`, `process_arr_readarr.go` | Readarr was retired upstream; Bookshelf replaces it in scope |
| Whisparr v2 vs v3 split | `pkg/arr/whisparr` | monolith has one adult module |
| Sportarr | `pkg/arr/sportarr` | third-party, out of scope |
| LunaSea notifications | `notification/services/lunasea` | app unmaintained |
| `pkg/ptp` | unused package | dead code |
| Indexer definition v1 / "custom" format (`vars:` positional lists, `IndexerDefinitionCustom`, `IndexerImplementationLegacy ""`) | `domain/indexer.go:203, 356` | v2 format supersedes it |
| `DownloaderSettings.Basic` | `domain/downloader.go:63` | marked Deprecated, replaced by `Auth` |
| `baseUrlModeLegacy` | `http/web_legacy.go` | kept for proxies that rewrite URLs |
| Dead filter fields `scene`, `tags_any`, `except_tags_any`, `except_release_types`; action `webhook_method/type/headers` | `domain/filter.go`, `domain/action.go` | stored, never used |
| retry-go v3 alongside v4; `github.com/pkg/errors` alongside own `pkg/errors` | `go.mod` | duplicate deps |
| autodl-irssi config importer | `web/src/screens/filters/_configParser.ts` | autodl-irssi is long dead; one-off migration aid |
| Hard-coded list presets on `api.autobrr.com` | `web/src/domain/constants.ts` | external dependency on autobrr-hosted proxy |
| `go-deadlock` mutex in production | `internal/irc/channel.go` | debugging aid with runtime overhead |

---

## 11. Rewrite notes

### 11.1 Reuse directly (the code is Go, and GPL-2.0-or-later is compatible with a GPL-3.0+ monolith)

1. **IRC subsystem** (`internal/irc`, `internal/announce`, about 5k LOC plus 2.7k of tests). This is the highest-value lift and has no equivalent in any *arr.
   - Keep: the ergochat `ircevent` usage, the connection and channel state machines, NickServ/SASL/invite-bot handling, the bouncer and proxy dialers, message history, and the health model.
   - Replace: the `releaseService` dependency, with an interface that emits a normalized release into the monolith's ingest bus.
   - Drop `go-deadlock`.
2. **Definition format plus the 118 YAML definitions**, with their embedded regex tests.
   - Keep: `IndexerIRCV2Parse`, `ParseLine`, `MapVars`, template URL building, mappings and tombstones. Keep the three Go special-case parsers (GGn, OPS, RED).
   - Consider merging with Prowlarr's Cardigann YAML into **one indexer definition schema that has an optional `irc:` block**. Cardigann already carries the login, search and download sections and the same tracker identities, so a tracker would get its search, RSS and IRC announce from one file (see the overlap in §11.3).
3. **Download-client adapters.** autobrr's `go-qbittorrent`, `go-deluge` and `go-rtorrent` libraries are maintained Go clients.
   - The qBittorrent adapter already covers reannounce, content layout, rules and first/last piece priority.
   - Adopt these as the monolith's torrent-client layer; the .NET clients cannot be reused anyway. SABnzbd, NZBGet, Transmission and Aria2 are small.
   - **Check the licenses of the separate `autobrr/go-*` repositories before vendoring**; they are not in this repo.
4. **`pkg/torznab` / `pkg/newznab`** (XML types and caps parsing) are a starting point for the monolith's generic Newznab/Torznab client. The *arrs still need search modes (`t=tvsearch`/`movie`, id params) that autobrr never uses.
5. **Tracker API clients** (`pkg/btn`, `ggn`, `red`, `ops`, and `ptp` even though it is unused) serve as reference code for Prowlarr-native indexers.
6. **Infrastructure idioms worth adopting wholesale:**
   - `pkg/migrator` (tiny embed-FS migrator), or the monolith could use goose/atlas. Either way, keep **one migration set**; autobrr maintains two numbered SQL trees (99 vs 88 files).
   - SQLite pragma set, integrity check, pre-migration backup and backup rotation (`database/sqlite.go`).
   - `modernc.org/sqlite`, for no CGO.
   - chi, scs sessions, go-oidc with PKCE, argon2id.
   - zerolog with a per-release `trace_id` in context (`zerolog.Ctx(ctx)`), and a lumberjack log file mirrored to an SSE stream.
   - The `AUTOBRR__KEY` / `_FILE` env convention (Docker secrets).
   - Typed generic event bus (`events.Topic[T]`).
   - Consumer-side small interfaces and hand-wired `main.go`.
   - automaxprocs and automemlimit.
   - Build-tag-separated integration, IRC and e2e suites with an in-process ircd (`test/irc/ircd`) and embedded Postgres.
7. **Macro/template engine** (`domain/macros.go`: text/template, sprig, about 80 release variables) can be reused for Custom Script and Webhook payloads and for file naming previews.

### 11.2 Redesign

- **Data model.**
  - Replace the 80-column `filter` row and comma-string lists with typed condition rows, or JSON validated by a schema.
  - Replace `Release` (90 fields mixing concerns) with the monolith's `ReleaseInfo` / `ParsedRelease` / `RemoteItem` split.
  - Drop hand-written column lists; use sqlc or a thin mapper.
- **Filter engine versus decision engine** (the core overlap). The two systems implement the same concepts in different ways:

| autobrr filter concept | *arr / Profilarr equivalent | Monolith target |
|---|---|---|
| match/except releases, groups, release tags (wildcard/regex) | CF specs ReleaseTitle, ReleaseGroup; Release Profiles must/must-not | CF engine |
| resolutions/sources/codecs/containers/HDR/other/language | Quality definitions + CF specs Resolution, Source, Language, QualityModifier | quality model + CF |
| min/max size | quality size limits, AcceptableSize spec | same |
| freeleech/freeleech %/bonus/origin/indexer flags | CF IndexerFlag spec (lossy bitmask) | **native indexer flags** (the richer set from autobrr + Radarr's PTP/AHD) |
| uploader, category, tags, description, record label, log score/cue/perfect FLAC | none | **new CF spec types** (announce metadata) |
| announce types (NEW/PROMO/…) | none | release attribute + CF spec |
| seeders/leechers | MinimumSeeders spec | same |
| shows/seasons/episodes/years/artists/albums + lists | the monitored library itself | **replaced by "wanted by library"** |
| smart episode, duplicate profiles | Upgradable/Queue/History specs, cutoff | **replaced by the real upgrade logic** |
| max downloads per period | Prowlarr grab limits (per indexer) | per-indexer and per-rule rate limits |
| delay (seconds) | delay profiles (minutes, per protocol) | one delay mechanism, **persisted** |
| priority + first-match | CF score ranking | score ranking |
| external EXEC/WEBHOOK filters | none | "external check" spec (keep; users rely on it for racing tools) |
| action = client + options (category, save path, limits, reannounce) | download client + category + tags; remote path maps | per-grab client options on the download-client layer |

  **Recommendation.** Model an autobrr "filter" in the monolith as an **"autograb rule"** made of three parts:
  - (a) a scope: indexers, announce types and whether to require a library match;
  - (b) conditions expressed with the **same CF spec engine** plus the announce-metadata specs;
  - (c) an action profile.

  Two modes follow from this:
  - **Library-driven grabs** (the Sonarr/Radarr use case) go through the normal decision engine, which then sees IRC releases too.
  - **Library-less "racing" grabs**, i.e. ratio building (autobrr's main use today), skip library matching but reuse the same conditions and clients.

  Keep a way to show per-condition rejection reasons, like `RejectReasons`, because the UI depends on it.
- **Durability.** Filter delay, reannounce retries and pending pushes should become persisted jobs in the monolith's command/queue system instead of goroutine sleeps.
- **Hot-path latency.**
  - Keep filter evaluation in memory: autobrr loads filters from the DB on every announce (`FindByIndexerIdentifier`), and the monolith should cache compiled rules instead.
  - Preserve the "download .torrent only when needed" optimisation.
- **Release parser choice.** autobrr uses `rls` (Go, a lexer covering all media types); Sonarr and Radarr use regex cascades with large golden test corpora. Because the monolith is Go, rls (autobrr fork) is a candidate foundation for the unified parser (comparison.md §2.3):
  - it already covers TV, movies, music, games, apps and books in one pass;
  - it would need the Sonarr/Radarr test corpora ported onto it, a mapping into the canonical Source × Resolution × Modifier quality model, and anime/absolute numbering checks.
  - Evaluate this before committing, and **confirm the rls fork's license** (upstream moistari/rls is believed to be MIT; verify).
- **Lists** go away as a concept. *arr-library lists become native; external lists (Trakt, MDBList, AniList, Steam, Metacritic, plaintext) merge into the monolith's import-list framework. Drop the dependency on `api.autobrr.com` presets, or proxy them ourselves.
- **Notifications.** Merge into the unified notification framework and add autobrr's events: IRC disconnected/reconnected, push approved/rejected/error, new release. Shoutrrr as a catch-all transport is worth adopting.

### 11.3 Overlap with Prowlarr's indexer layer

- **Same concept, two definition formats.** Both have tracker identity, settings (passkey/rsskey/authkey), base URLs, protocol and privacy. Prowlarr's Cardigann definitions add login, search and download; autobrr's add IRC and announce parsing. Many trackers appear in both: btn, ptp, red, ops, ggn, ant, hdb, filelist, torrentleech, iptorrents, nebulance, animebytes, and others.
  - **The monolith should have one indexer entity with capabilities `{search, rss, irc, api}`.** An IRC network and channel attach to that indexer, not to a parallel "IRC indexer".
  - Definition merging must reconcile identifiers: autobrr's `identifier` against Prowlarr's definition `id`. autobrr also has `identifier_external` because *arrs match on indexer name. In a monolith the push indexer name disappears.
- **The download path is the same:** a .torrent URL with a passkey, cookies, rate limiting and Retry-After handling. Use one grab/download service; Prowlarr's per-indexer query and grab limits cover autobrr's `max_downloads`.
- **Proxies:** one proxy entity for HTTP, SOCKS and FlareSolverr, covering IRC too.
- **Health:** IRC network health (connected, joined, last announce) becomes indexer health checks in the unified health framework.

### 11.4 Suggested module shape in the monolith

```
indexer/            (unified definitions: cardigann + irc blocks; tombstones)
indexer/irc/        (lifted from autobrr internal/irc + announce)
indexer/feed/       (torznab/newznab/rss poller; one RSS-sync job for all)
release/parse/      (rls-based or unified parser)
decision/           (CF spec engine incl. announce-metadata specs; profiles; autograb rules)
grab/               (download service; client adapters from autobrr go-* libs)
downloadclient/     (qbit/deluge/rtorrent/transmission/sab/nzbget/aria2/watch)
```

Flow: IRC announce → `release.Ingest(source=irc)` → decision engine (library match or autograb rule) → grab → download client. This replaces autobrr → HTTP `release/push` → *arr → decision engine → download client.
