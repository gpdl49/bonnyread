# Kapowarr — Codebase Analysis

Source: `submodules/kapowarr` (commit `c191dda6`, 2026-09-14, `pyproject.toml` version **1.3.2**). All paths are relative to `submodules/kapowarr/`. `impl/` = `backend/implementations/`.

**Summary:** Kapowarr is an independent Python app. It is not an NzbDrone fork and shares no code with the .NET *arrs; only the UI copies their look. It has exactly **one indexer, GetComics.org**, which it reads by scraping HTML. Each GetComics article is resolved into links on file hosts (GetComics' own servers, Mega, MediaFire, Pixeldrain, WeTransfer) or into torrents. File-host links are downloaded in-process by Kapowarr's own HTTP and Mega clients. Torrents are handed to qBittorrent or Transmission. There is **no Usenet, no Newznab/Torznab, no Prowlarr**, and no notifications. Metadata comes only from **ComicVine**, and the user must supply their own API key.

---

## 1. Overview

**Purpose.** Kapowarr manages a comic library. The unit you add and monitor is a *volume* (a ComicVine volume, which is a series run). Each volume has *issues*. Kapowarr searches for monitored issues that have no file, downloads them from direct-download (DDL) hosts or torrents, then moves, renames and optionally converts the archives (cbz/cbr/zip/rar, or extracted to a folder). It can also import an existing library by matching filenames against ComicVine.

**Stack**
- **Python ≥ 3.8** (`pyproject.toml`). CI tests 3.8–3.12 (`.github/workflows/tests.yml`). The Docker image uses **3.13** on Debian bookworm slim, with a Rust builder stage for the wheels (`Dockerfile`).
- **Dependencies** (`requirements.txt`): Flask 3, waitress 3 (WSGI server), flask-socketio 5 + websocket-client, requests[socks] 2.31, aiohttp 3.9 (async scraping and API calls), beautifulsoup4 4.12 (all scraping), cryptography ≥ 46 (Mega AES), bencoding (parsing .torrent files), cron-converter (task schedules), typing_extensions.
- **DB:** SQLite through the stdlib `sqlite3` module and hand-written SQL. There is no ORM. A custom cursor lives in `backend/internals/db.py`.
- **Frontend:** server-rendered Jinja templates plus vanilla JS and CSS, with no build step. socket.io 4.7.5 is loaded from `cdn.socket.io` (`frontend/templates/base.html:24`).
- **Bundled binaries:** proprietary RARLAB `rar` executables for Linux x64, BSD x64 and Windows x64 (`backend/lib/rar_*`, ~640 KB each). Kapowarr runs them as subprocesses to create rar/cbr files and to extract rar archives (`backend/base/helpers.py:205 run_rar`).
- **Process model:** `Kapowarr.py` is a supervisor. It re-spawns itself as a child process (`KAPOWARR_RUN_MAIN=1`) and restarts that child when it exits with codes 131–133 (`StartType`, `backend/base/definitions.py:292`).

**Architecture (layers)**
- `backend/base/`: constants, enums and typed dicts (`definitions.py`, 1.9k lines), helpers (HTTP `Session`/`AsyncSession` with retry, proxy and CloudFlare hooks), the filename parser (`file_extraction.py`), file-system operations (`files.py`), logging and exceptions.
- `backend/internals/`: DB, migrations, settings, the Flask/waitress/socket.io server, status checks, and DB backup/import.
- `impl/`: the plugin-like registries, each a decorator-based registry that auto-imports every module in its folder:
  - `IndexerClients` (`indexer_client_manager.py`)
  - `QueryBuilders` (`query_builder_manager.py`)
  - `DownloadPreppers` (`download_prepper_manager.py`), which turn an indexer result into concrete downloads
  - `DownloadClients` (`download_client_manager.py`), the built-in downloaders
  - `ExternalClients` (`external_client_manager.py`), the torrent clients
  - `ConvertersManager` (`converters.py`)
  - `Credentials` (`credentials.py`)
  - `StatusHandlers` (`internals/status.py`)
  - Domain code: `volumes.py`, `comicvine.py`, `naming.py`, `matching.py`, `file_matching.py`, `root_folders.py`, `remote_mapping.py`, `blocklist.py`
- `backend/features/`: orchestration. Covers the download queue, full search (`search_full.py`), RSS "discover" (`search_discover.py`), post-processing, library import, mass edit and tasks.
- `frontend/`: `api.py` (REST API, 1.7k lines), `ui.py` (page routes), templates and static files.

**LOC** (non-blank and blank lines, `.py` unless noted)

| Area | LOC |
|---|---:|
| `backend/implementations` (all integrations and domain) | 12,175 |
| — of which `download_clients/Mega.py` (own MEGA protocol client) | 1,265 |
| — `download_preppers/ddl/GetComics.py` (article scraper) | 743 |
| — `indexer_clients/ddl/GetComics.py` (search scraper) | 235 |
| `backend/base` (definitions, parser, files, helpers) | 6,157 |
| `backend/internals` (db, migrations 1.3k, settings, server) | 4,381 |
| `backend/features` | 3,329 |
| `frontend/api.py` + `ui.py` | 1,841 |
| `Kapowarr.py` | 369 |
| **Backend total** | **~28k** |
| Frontend JS / CSS / HTML | 5.2k / 3.0k / 2.7k |
| Tests (`tests/`, mostly filename-parser cases) | 510 |

This is roughly **1/5 the size of Sonarr's backend**, and the domain is narrow.

---

## 2. Domain model

Schema: `backend/internals/db.py:379-580`.

| Table | Purpose / notable columns |
|---|---|
| `config` | Key/value store for all settings (`key`, `value BLOB`). It also holds `database_version`, `auth_salt`, `api_key` and `last_rss_sync`. |
| `root_folders` | `id`, `folder` (unique). |
| `volumes` | `comicvine_id`, `title`, `alt_title` (first CV alias), `year`, `publisher`, `volume_number`, `description`, `site_url`, `monitored`, `monitor_new_issues`, `root_folder` FK, `folder`, `custom_folder`, `last_cv_fetch`, `special_version`, `special_version_locked`. |
| `volumes_covers` | Cover image stored as a **BLOB in the DB** (1:1 with volume). |
| `issues` | `volume_id`, `comicvine_id` (unique), `issue_number` (string, e.g. "1.MU", "½"), `calculated_issue_number FLOAT` (sortable/rangeable), `title`, `date`, `description`, `monitored`. |
| `files` | `filepath` (unique), `size`. |
| `issues_files` | **M:N** file↔issue. One archive can cover an issue range or a whole TPB. `forced` = the user matched it manually. |
| `volume_files` | Files that belong to the volume rather than an issue: `file_type` ∈ {`cover`, `metadata`} (`GeneralFileType`). |
| `indexer_clients` | `download_type` (1=DDL, 2=TORRENT), `client_type` ("GetComics"), `url`, plus GetComics-only columns `gc_service_preference` and `gc_avoid_large_downloads`. A GetComics row is seeded on first start (`db_migration.py:66-99`). |
| `external_download_clients` | Torrent clients: `client_type`, `base_url`, `username`, `password`, `api_token`, `enabled`. |
| `download_queue` | Persistent queue. Holds `client_type` (the `DownloadClientIdentifier`: `direct`, `mf`, `mf_folder`, `mega`, `mega_folder`, `pd`, `pd_folder`, `torrent`, `wt`), `download_link`, `covered_issues` ("n" or "a,b"), `source_type` and `source_name`, and `web_link`/`web_title`/`web_sub_title` (the GetComics article and the group heading inside it). |
| `download_history` | Article and file titles, volume and issue, `source`, `downloaded_at`, `success`. |
| `blocklist` | `download_link` (unique), `download_service`, `web_link`, `reason` (1 link broken, 3 no working links, 4 added by user), `volume_id`, `issue_id`. |
| `credentials` | Per-service accounts: `source` ∈ {mega, pixeldrain}, `username`, `email`, `password`, `api_key`. Stored in **plaintext** (`impl/credentials.py:119`). |
| `remote_mappings` | Remote-to-local path mapping per external client. |
| `task_history`, `task_intervals` | Task log, and the cron `schedule` + `next_run` for each scheduled task. |
| `status` | Persisted health issues (`status_type`, `subtype`, `expires_at`). |

**Enums worth carrying over** (`backend/base/definitions.py`)
- `SpecialVersion`: `tpb`, `one-shot`, `hard-cover`, `omnibus`, `volume-as-issue`, `cover`, `metadata`, and `None` for a normal volume.
  - Kapowarr infers this per volume with heuristics over the title, the first sentence of the description, the issue count and the release date (`impl/volumes.py:1276 determine_special_version`).
  - Example: a volume with one issue released more than 30 days ago is treated as a TPB.
  - The user can override and lock the value.
- `MonitorScheme`: `all`, `missing`, `none`.
- `DownloadState`: `queued`, `paused`, `downloading`, `seeding`, `importing`, `failed`, `canceled`, `shutting down`.
- `DownloadType`: `DDL`, `TORRENT`. A code comment at `definitions.py:583` says usenet is planned.

**Migrations.** Kapowarr uses its own migrator (`backend/internals/db_migration.py`). It keeps an integer `database_version` in `config` and runs **50 registered Python handlers** (version 1 through 50) as raw SQL scripts, including temp-table rebuilds. It runs `VACUUM` after migrating. Recent handlers:
- `_migrate_add_gc_indexer` (47) moved GetComics from hard-coded to an indexer row.
- `_migrate_task_intervals_to_schedule` (49) switched task intervals to cron.

DB import and backup live in `internals/db_backup_import.py`. It refuses a DB with a newer version and reverts an import after 60 s unless an authenticated request arrives in the meantime.

---

## 3. Feature areas

### 3.1 Metadata: ComicVine (`impl/comicvine.py`)

- **Base URL:** `https://comicvine.gamespot.com/api` (`Constants.CV_API_URL`).
- **Auth:** query params `api_key=<user key>&format=json`.
  - The key is **required**. There is no shared or proxied key, and `ComicVine()` raises `InvalidKeyValue` without one.
  - It is set in Settings → General (`comicvine_api_key`) and tested via `/volume/4050-...`-style or `/volumes` calls (`test_key`, `comicvine.py:450`).
- **Endpoints used**

  | Call | Params | Used for |
  |---|---|---|
  | `GET /search/` | `query`, `resources=volume`, `limit=50`, `field_list=aliases,count_of_issues,deck,description,id,image,name,publisher,site_detail_url,start_year` | Add-volume search. The query `cv:NNN` or `4050-NNN` short-circuits to a direct lookup. |
  | `GET /volume/4050-{id}/` | `field_list=` the volume fields plus `issues` | Single volume fetch on add or refresh |
  | `GET /volumes/` | `filter=id:a\|b\|…` (100 ids per request), `field_list` | Bulk refresh. Batches of 10 concurrent requests (1,000 ids). |
  | `GET /issues/` | `filter=volume:a\|b\|…` (50 volumes per request), `field_list=id,issue_number,name,cover_date,store_date,description,volume`, `offset` paging by 100 | Issue lists |
  | Cover images | `image.small_url` from the volume response (CV static CDN) | Stored as a BLOB |

- **CV status codes handled:**
  - 100: bad key (`InvalidKeyValue`)
  - 101: not found (`VolumeNotMatched`)
  - 107: rate limit
  - Any network or JSON error raises `MetadataSourceRateLimitReached`.
- **Rate limiting:**
  - Between batches of 10 concurrent requests, Kapowarr sleeps `CV_BRAKE_TIME (1.0 s) × batch_size`, i.e. 10 s (`__sleep_iter`, `comicvine.py:421`).
  - When the limit is hit, a `CV_RATE_LIMIT` status is raised and expires after 1 h (`internals/status.py:254`).
  - The docs rely on CV's per-resource limit of about 200 requests/hour (up to 20k volumes/hour via 100-id batches) (`docs/src/general_info/implementation_details.md`).
- **Refresh strategy:** `update_all` runs hourly. It fetches volumes whose `last_cv_fetch` is older than 24 h, oldest first. Issues are re-fetched only when the CV `count_of_issues` differs from the local count, or when the volume has not been refreshed for 30 days (`impl/volumes.py:1356-1440`). This works around the limit.
- **Date type:** cover date or store date for the issue date (`date_type` setting).
- **Description cleanup:** HTML is stripped by BS4 (`_clean_description`).
- **Volume number:** derived heuristically from CV fields (`__determine_volume_number`).

### 3.2 Search & matching

- **Query generation** (`impl/query_builders/DDL.py`): fixed format tuples.
  - Volume: `"{title} ({year})"`, `"{title} Vol. {n} ({year})"`, `"{title} Vol. {n}"`, `"{title}"`.
  - Issue: `"{title} #{n} ({year})"`, `"{title} #{n}"`.
  - TPB: `"... TPB"`.
- **Search state machine:** `impl/search_action_planner.py` is a Mealy-type FSM. Its actions are SEARCH_VOLUME → SEARCH_ISSUE → FETCH_NEXT_PAGE → NEXT_QUERY_VARIATION → NEXT_TITLE_ALIAS → STOP. It decides, per iteration, whether to page, try the next query variation, fall back to per-issue searches, or try the alt title. `features/search_full.py SearchCoordinator` drives it across the indexers.
- **Filename/title parser:** `backend/base/file_extraction.py` (666 lines), a large regex set.
  - Extracts series, volume number (including roman numerals), issue number or range (`1-5`, `1 of 6`, `#½`, `∞`), year (many formats), special version (TPB/OS/HC/omnibus), annual, cover and page numbers.
  - Translates Russian, Chinese, Korean, Japanese and French volume/issue words.
  - `calculated_issue_number` maps strings to floats for range math.
- **Match filter:** `impl/matching.py check_search_result_match` rejects a result for any of: blocklisted link, annual mismatch, title mismatch (normalised, alphanumeric-only equality with the CV title or first alias), volume-number mismatch, special-version conflict, year outside the volume's start–end range, or issue numbers not in the volume or not the searched issue. Each rejection carries a human-readable reason shown in manual search.
- **Ranking:** `SearchCoordinator._rank_search_result`. Ranks by matched first, then title similarity, then issue coverage, etc. **There are no quality profiles or custom formats**; the only "quality" knob is the format preference used after download.
- **Selection:** `choose_downloads` greedily picks non-overlapping results that cover the most still-open issues (`search_full.py:366`). A special version or single-issue search takes the top result.
- **Manual search:** `GET /api/volumes/{id}/manualsearch` and `/issues/{id}/manualsearch`, then `POST .../download` with `link` and `force_match`.
- **Manual file matching:** `GET/PUT /api/volumes/{id}/manualmatch` (`impl/file_matching.py`).

### 3.3 Direct-download acquisition (the distinct part)

**Indexer = GetComics.org only** (`impl/indexer_clients/ddl/GetComics.py`, `allow_multiple_instances=False`). Kapowarr fetches the site anonymously with a browser user-agent; there are no credentials. Pipeline:

1. **Search** scrapes `GET {url}/page/{n}?s={query}` (WordPress search).
   - It parses `<article class="post">` → `h1.post-title a` (link and title) and a sibling "Size : …" text.
   - It skips the "News" and "Sponsored" categories and detects the next page via `.page-numbers`.
   - It sleeps 1 s once more than 5 requests have been made in a session.
   - Titles go through the filename parser, so a GetComics article title acts as the "release name".
2. **RSS sync** (`discover`) is not RSS. It scrapes `GET {url}/sitemap/` (an `lcp_catlist` list with dates) for posts newer than `last_rss_sync − 2 days`. It pre-matches them to monitored volumes by title, then runs the full matcher (`features/search_discover.py`).
3. **Prepping** (`impl/download_preppers/ddl/GetComics.py`): fetches the article page and extracts **download groups** from `section.post-contents`:
   - "Button" groups: a `<p>` containing "Language", followed by `div.aio-button-center a` buttons up to the next `<hr>`. Group title, Year and Size are parsed from the `<p>`.
   - "List" groups: `<ul><li>` items with links, used on multi-issue posts.
   - Each link is classified by its **button text** (`GC_DOWNLOAD_SERVICE_TERMS`, `definitions.py:568`): `mega`, `mediafire`, `wetransfer`, `pixeldrain`, GetComics ("download now", "main server", "mirror", "link 1/2"…), torrent ("torrent", "magnet").
   - Hard-coded exclusions: `sh.st` (ad shortener) and `torrentgalaxy.to` links. Torrent links are also skipped when no torrent client is configured.
   - Groups are **sorted by the user's service preference**. The `avoid_large_downloads` option pushes GetComics' own server to last place for groups of 400 MB or more.
4. **Link paths:** groups are combined into non-overlapping "paths". Example: a TPB + extras group, then a plain TPB group, then issues 1–5 + 6–10.
   - Paths are ranked with special versions first, then widest coverage, then fewest downloads.
   - For each path, Kapowarr "purifies" each group's links in parallel:
     - It follows redirects and reads the final URL and Content-Type. A 429 from GetComics raises a rate-limit error.
     - It detects Mega file vs folder, MediaFire `error.php` (broken) vs folder, and Pixeldrain `/l/` (list).
     - For a `.torrent` file it bencode-hashes the info dict, then **synthesises a magnet with ~17 hard-coded public trackers** (`GetComics.py:~330`, many long dead).
   - The first path with working links wins. Broken links are blocklisted automatically, and "no working links" blocklists the whole article.
5. **Built-in download clients** (`impl/download_clients/`), all streaming to `download_folder` with resume via `Range` where supported (`base.py BaseDirectDownload`):

   | Identifier | Service | How |
   |---|---|---|
   | `direct` | GetComics main/mirror servers, raw MediaFire CDN links | Plain HTTP GET, filename from Content-Disposition |
   | `mf` | MediaFire file | Scrapes the page for `window.location.href='https://downloadNN.mediafire.com/...'`, `#downloadButton[href]`, or base64 `data-scrambled-url` |
   | `mf_folder` | MediaFire folder | `POST https://www.mediafire.com/api/1.5/file/zip.php` (keys=folder key) → zip stream |
   | `mega`, `mega_folder` | MEGA | **Full in-house MEGA client** (`Mega.py`, 1.3k lines). API `https://eu.api.mega.co.nz/cs`, AES-ECB/CBC/CTR key handling, attribute decryption, CBC-MAC checksum, hashcash (`gen_cash`), anonymous login or account login (v1 password key and v2 PBKDF2). Detects transfer-quota exhaustion → `DownloadServiceRateLimitReached`. Folders are downloaded file-by-file. |
   | `pd`, `pd_folder` | Pixeldrain | `GET https://pixeldrain.com/api/file/{id}` and `/api/list/{id}/zip`. An optional API key (Basic `:key`) is checked against `/api/user` and `/api/misc/rate_limits` or the monthly transfer cap. |
   | `wt` | WeTransfer | `POST https://wetransfer.com/api/v4/transfers/{id}/download` `{intent: entire_transfer, security_hash}` → `direct_link` |
   | `torrent` | via external client | `TorrentDownload` wraps a magnet and delegates to qBittorrent or Transmission |

   - **Rate-limit handling:** when a service reports its limit, every other queued download of that service is removed (`download_queue.py _remove_all_of_service`). A `DOWNLOAD_SERVICE_RATE_LIMIT` status is raised with a timer.
   - **Concurrency:** `concurrent_direct_downloads` (default 1). Downloads run in threads (`DownloadThread-{id}`). The queue is persisted and **re-prepped on restart**.
6. **CloudFlare:** `Session` and `AsyncSession` (`backend/base/helpers.py:962,1044`) detect the `cf-mitigated: challenge` header.
   - If a FlareSolverr (or Byparr) URL is configured, Kapowarr calls FlareSolverr `POST {fs}/v1 {cmd: request.get}` (300 s timeout, at most 2 FlareSolverr sessions). It caches the `cf_clearance` cookie and user-agent per domain, then retries (`impl/flaresolverr.py`).
   - Otherwise it raises the `CF_CHALLENGE_WITH_NO_FS` status.

### 3.4 Torrent / Usenet

- **Torrent:** only as a GetComics link type; there is no torrent indexer. There are two external clients (`impl/external_clients/torrent/`):
  - **qBittorrent:** WebUI API v2 (`/api/v2/auth/login`, `/app/version`, `/torrents/add` with `category=kapowarr`, `/torrents/info`, `/torrents/delete`).
  - **Transmission:** RPC, requires rpc_version_semver ≥ 6.0.0, i.e. Transmission ≥ 4.1.0.
  - Status is polled every 30 s.
  - `failing_download_timeout` kills stalled torrents.
  - When several clients exist, the one with the fewest queued downloads is used (`get_least_used_client`).
  - `seeding_handling`: `complete` waits for seeding to finish, then moves; `copy` copies on reaching seeding, then deletes when done.
  - Remote path mappings apply.
- **Usenet:** **none.** There is no Newznab or Torznab. The `'Torznab'` strings in docstrings (`indexer_client_manager.py:242`) and the comment at `definitions.py:583` show a planned generic indexer design (`allow_multiple_instances`, `DownloadType`, per-type `QueryBuilder`).

### 3.5 File processing

- **Post-processing** (`features/post_processing.py`), for a successful direct download:
  1. Remove from queue.
  2. Write history.
  3. Move to the volume folder, generating the name if `rename_downloaded_files` is on.
  4. Fix the extension from magic bytes (`ARCHIVE_MAGIC_BYTES`: zip, rar4/5, 7z).
  5. Add the file to the DB and scan-match it.
  6. Convert.
  7. Set file properties.
- **Conversion** (`impl/converters.py`, `impl/conversion.py`): a registry of source→target converters over **zip, cbz, rar, cbr and "folder"** (extract).
  - Any rar/cbr target, and rar→folder, needs the bundled 64-bit `rar` binary.
  - Choice is driven by the ordered `format_preference` list.
  - `extract_issue_ranges` extracts multi-issue archives into per-issue files; the contents of extracted folders are renamed with `generate_image_name`.
  - **PDF, EPUB, CB7/7z, CBT, CBA and MOBI are recognised as media (`CONTAINER_EXTENSIONS`) but cannot be converted.** There is no PDF→CBZ.
  - There is no **ComicInfo.xml** reading or writing; `comicinfo.xml` and `cvinfo.xml` are only recognised as "metadata" general files.
- **Renaming** (`impl/naming.py`): templates for the volume folder, the issue file, an issue without a title, a special version, and volume-as-issue.
  - Keys: `series_name`, `clean_series_name`, `volume_number`, `comicvine_id`, `year`, `publisher`, `special_version`, `issue_comicvine_id`, `issue_number`, `issue_title`, `issue_release_date`, `issue_release_year`.
  - Padding options: `volume_padding`, `issue_padding`.
  - Illegal-character replacement, same-name indexing, and a mass-rename preview.
- **File properties** (`impl/file_processing.py`): optional mtime from the issue release date (`change_file_date`), `chmod_folder`, `chown_group`.
- **Scanning** (`impl/file_matching.py scan_files`): walks the volume folder, parses filenames, and links files to issues (M:N) or as volume general files. Optionally it unmonitors issues whose files were deleted.

### 3.6 Library import (`features/library_import.py`)

1. Scans the root folders for unmatched files, with a folder or glob filter, a limit of 20 folders by default, and a parent-folder option.
2. Groups the files by parsed series/year/volume.
3. Runs a batched **CV `/search` per group** (10 at a time with a sleep between batches), then picks the best result with `select_best_volume_result_for_file`. An option restricts matches to English.
4. The user confirms or edits the matches. Kapowarr then adds the volumes (the root folder is inferred from the path; the volume folder is the common parent), imports the files, and optionally renames them.

The step degrades silently when the CV rate limit is hit: files are left unmatched.

### 3.7 Scheduled tasks

See §6.

### 3.8 Auth

- Optional username + password, hashed with PBKDF2-SHA256 (100k iterations, per-install salt) and stored in `config` (`backend/base/helpers.py:286`, `frontend/api.py:223`).
- **`POST /api/auth` hands out the global `api_key`.** When no password is set, it hands the key to anyone who asks, so there is no auth by default.
- All other endpoints require `?api_key=` (from `request.values`: query or form). There is no header auth. The UI stores the key in `localStorage` (`static/js/auth.js`) and appends it to every URL.
- The socket.io connection checks `auth.api_key` (`internals/server.py:111`).
- There are no sessions, no OIDC, and no "disable for local addresses" option.

---

## 4. Integrations (full lists)

| Category | Integration | Mechanism | Fragility |
|---|---|---|---|
| Metadata | **ComicVine** | REST JSON with the user's API key | Medium. Strict hourly per-resource limit and velocity detection. The user must obtain a key. |
| Indexer | **GetComics.org** | HTML scraping of WordPress search, `/sitemap/` and article pages | **High.** It depends on CSS classes and button texts (`aio-button-center`, `post-contents`, "Language", "Size :"). The site sits behind CloudFlare, and its piracy status means domain changes are possible (the URL is configurable). |
| DDL host | GetComics main/mirror servers | HTTP after redirect | High (429 limits, CloudFlare) |
| DDL host | **MEGA** | In-house MEGA protocol reimplementation (crypto, hashcash, login v1/v2) | **High.** MEGA changes its API and login flows. There are transfer quotas (~5 GB/6 h anonymous). Accounts are supported. |
| DDL host | **MediaFire** | Page scraping (JS redirect, `#downloadButton`, `data-scrambled-url`), plus the folder zip API `api/1.5/file/zip.php` | High (scrape-based) |
| DDL host | **Pixeldrain** | Official API `/api/file`, `/api/list/{id}/zip`, `/api/user`, `/api/misc/rate_limits` | Medium. Documented API with free-tier transfer limits. An API key is supported. |
| DDL host | **WeTransfer** | Private web API `v4/transfers/{id}/download` | High (undocumented) |
| Link filter | `sh.st`, `torrentgalaxy.to` | Hard-coded avoid list | TorrentGalaxy is gone; the entry is dead weight. |
| Download client | **qBittorrent** | WebUI API v2 | Low |
| Download client | **Transmission** | RPC, ≥ 4.1.0 only | Low. Strict version floor. |
| Torrent trackers | ~17 public trackers hard-coded into synthesised magnets (`cyberia.is`, `port443.xyz`, `itzmx.com`, `skynetcloud.tk`, `re-tracker.uz`, …) | String constant | **Mostly dead or stale.** DHT does the real work. |
| Anti-bot | **FlareSolverr / Byparr** | `POST /v1 request.get` | Medium. FlareSolverr is itself fragile. |
| Proxy | HTTP/HTTPS/SOCKS5/SOCKS5h | requests[socks] + aiohttp | Low. `PROXY_TEST_URL = https://httpbin.org/ip` is a third-party dependency. |
| Frontend CDN | `cdn.socket.io` 4.7.5 | `<script>` with SRI | UI breaks offline or air-gapped. |
| Notifications | **None** | — | — |
| Media servers / readers | **None.** No Komga, Kavita or Plex hooks. | — | — |
| Usenet / Newznab / Torznab / Prowlarr | **None** | — | — |
| Update check / telemetry | **None** found | — | — |

---

## 5. External APIs / interfaces

- **REST API:** `/{url_base}/api/*`, implemented in `frontend/api.py`. It is JSON, documented only in `docs/src/other_docs/api.md` (8 lines, which point to the source). There is **no OpenAPI** and no versioning. Responses use the envelope `{error, result}`, with error names mapped from `custom_exceptions.py`. Route groups:
  - `auth`, `auth/check`, `public`
  - `system/about|status|logs|tasks|tasks/history|tasks/planning|tasks/{id}|power/shutdown|power/restart|database|database/backups[/{idx}]`
  - `settings` (GET/PUT/DELETE = reset), `settings/api_key` (regenerate), `settings/availableformats`
  - `rootfolder[/{id}]`, `remotemapping[/{id}]`
  - `indexers[/options|/test|/{id}]`, `externalclients[/options|/test|/{id}]`, `credentials[/{id}]`
  - `libraryimport`
  - `volumes/search` (CV search), `volumes`, `volumes/stats`, `volumes/{id}`, `/cover`, `/manualmatch`, `/rename`, `/convert`, `/manualsearch`, `/download`
  - `issues/{id}`, `/rename`, `/convert`, `/manualsearch`, `/download`
  - `activity/queue[/{id}]`, `activity/history`, `activity/folder` (empty the temp dir)
  - `blocklist[/{id}]`, `masseditor` (actions: delete, root_folder, rename, update, search, convert, monitor, unmonitor, monitoring_scheme, file_date, file_permissions, file_ownership)
  - `files/{id}` (download or delete a file)
- **Task trigger:** `POST /api/system/tasks {cmd: <task>, volume_id?, issue_id?}`. This is the rough analogue of Sonarr's commands.
- **WebSocket:** socket.io at `{url_base}/api/socket.io`. Events: `task_added`, `task_status`, `task_ended`, `queue_added`, `queue_status`, `queue_ended`, `mass_editor_status`, `downloaded_status`, `status_count` (`WebSocketEventType`, `definitions.py:256`).
- **Auth:** `api_key` in the query string or form data (see §3.8). CORS is `*` on the socket.
- **Not Sonarr-compatible:** there is no `/api/v3`, no `X-Api-Key` header and no webhook payloads, so third-party *arr tools cannot talk to it.

---

## 6. Background jobs

`features/tasks.py`. A single `TaskHandler` runs a FIFO queue, one task at a time, in its own thread. The scheduler table is `task_intervals`, with **cron** strings editable through `PUT /api/system/tasks/planning`.

| Task id | Trigger | What it does |
|---|---|---|
| `update_all` | cron `0 * * * *` (hourly) | CV refresh plus a file scan for volumes older than 24 h (skipping; see §3.1) |
| `rss_sync` | cron `0,30 * * * *` | GetComics sitemap discover → match → enqueue |
| `backup_db` | cron `0 0 * * 1` (Mon 00:00) | SQLite copy, keeps `db_backup_amount` (3) |
| `auto_search` | manual / mass edit / on add | Volume search for open issues |
| `auto_search_issue` | manual | Single-issue search |
| `refresh_and_scan` | manual / on add | Single-volume CV refresh plus scan |
| `mass_rename`, `mass_rename_issue` | manual | Rename per the templates |
| `mass_convert`, `mass_convert_issue` | manual | Convert per the format preference |

Other long-running threads:
- The download handler: one thread per download, with an external-client poll every 30 s.
- Status-expiry timers.
- Hosting/DB-change revert timers (60 s).
- The DB-connection reaper for waitress threads (`internals/server.py:46`).

---

## 7. Configuration surface

- **CLI flags** (`Kapowarr.py:255-290`): `-d/--DatabaseFolder`, `-t/--TempDownloadFolder`, `-l/--LogFolder`, `-f/--LogFile`, `-o/--Host`, `-p/--Port`, `-u/--UrlBase`.
- **Env:** `PUID`, `PGID`, `TZ` (Docker `entrypoint.sh` uses gosu), plus internal `KAPOWARR_RUN_MAIN` and `KAPOWARR_START_TYPE`.
- **Settings** (`backend/internals/settings.py:69-127`, stored in `config`):
  - General: `host`, `port`, `url_base`, `auth_username`, `auth_password`, `api_key`, `comicvine_api_key`, `flaresolverr_base_url`, `proxy_type/host/port/username/password/ignored_addresses`, `log_level`, `db_backup_amount`, `db_backup_folder`.
  - Media management: `rename_downloaded_files`, `replace_illegal_characters`, `volume_folder_naming`, `file_naming`, `file_naming_empty`, `file_naming_special_version`, `file_naming_vai`, `long_special_version`, `volume_padding`, `issue_padding`, `create_empty_volume_folders`, `delete_empty_folders`, `unmonitor_deleted_issues`, `change_file_date`, `chmod_folder`, `chown_group`, `convert`, `extract_issue_ranges`, `format_preference`.
  - Download: `download_folder`, `concurrent_direct_downloads`, `failing_download_timeout`, `seeding_handling`, `delete_completed_downloads`.
  - Metadata: `date_type`.
  - Private: `database_version`, `auth_salt`, `last_rss_sync`, and `backup_host/port/url_base` (used to revert a hosting change).
- **Per-row config:**
  - Indexers: GetComics URL, service preference order, avoid-large-downloads.
  - Torrent clients.
  - Credentials: Mega email/password, Pixeldrain API key.
  - Remote mappings, root folders, per-volume monitoring and special-version lock.
- **Hard-coded constants:** `backend/base/definitions.py` `Constants` (retries 5, timeout 30 s, CV brake 1 s, Mega/Pixeldrain URLs, max 32 DB connections, 10 hosting threads).
- **Missing:** no quality or format profiles, no tags, no delay profiles, no release restrictions.

---

## 8. Platform / infrastructure concerns

- **Bundled proprietary `rar` binaries** (x64 only; 32-bit and ARM cannot create rar). This is a licensing and ARM-support problem: an arm64 Docker image cannot run them, so rar/cbr output breaks there. Extraction of RAR also relies on them. There is no unrar library.
- **SQLite only**, raw SQL, cover images as BLOBs in the DB. A custom connection manager caps connections at 32 and reaps them per thread.
- **Threads plus nested `asyncio.run()`.** Sync Flask handlers call `run(coro)` for aiohttp. The mix of `requests` and `aiohttp` sessions means two HTTP stacks, each with its own CloudFlare and proxy hooks.
- **Supervisor/restart via subprocess** with exit codes, plus 60 s auto-revert of hosting or DB changes unless a UI request confirms them. This is a nice safety feature worth keeping.
- **Scraping fragility** is structural: every acquisition path depends on GetComics HTML and on file-host HTML or private APIs. CloudFlare requires an external FlareSolverr.
- **Security:**
  - No auth by default.
  - The API key travels in URLs (it ends up in proxy and access logs and in browser history).
  - Credentials and the torrent client password are stored in plaintext.
  - socket.io CORS is `*`.
- **Runtime CDN dependency** for socket.io.
- **Rate-limit statuses are in-memory timers** persisted to the `status` table.
- `requires-python >= 3.8` while CI tests up to 3.12 and Docker ships 3.13, so the code carries `typing_extensions` shims and `urllib3` v1/v2 branches (`helpers.py:940-960`).
- **Tests are minimal:** 510 lines, mostly filename-parser cases plus naming, status and library-import tests. There are no scraper fixtures.

---

## 9. Interactions with other *arr apps

- **No code integration with any *arr.** Kapowarr has:
  - no Prowlarr sync
  - no Newznab/Torznab client
  - no SABnzbd or NZBGet support
  - no Sonarr-style `/api/v3`
  - no webhooks
  - no Overseerr/Jellyseerr support
  - no Bazarr relevance
- It shares **qBittorrent and Transmission** with the other apps. It uses the category `kapowarr` (`Constants.EXTERNAL_DOWNLOAD_TAG`) and has its own remote path mappings.
- It shares **FlareSolverr** with Prowlarr and Jackett users: the same instance works, but the integration is separate.
- **Mylar3** (the incumbent comic manager) is not referenced anywhere in the code or docs:
  - There is no import of Mylar DBs.
  - There is no `cvinfo` generation, though `cvinfo.xml` files are recognised as metadata files.
  - It shares ComicVine as its source. Mylar supports Usenet, torrents and GetComics DDL, while Kapowarr supports DDL and torrents only.
- **Readers** (Komga, Kavita, Ubooquity) simply read the library folder. There is no refresh hook and no ComicInfo.xml writing.
- The UI copies Sonarr's look (menus, queue/history/blocklist, library import, mass editor), but none of the code is shared.

---

## 10. Outdated / legacy candidates

- The ~17 hard-coded public trackers in synthesised magnets (`download_preppers/ddl/GetComics.py`, `_purify_link`) are mostly dead. The code should pass the `.torrent` file itself, or use DHT only.
- The `torrentgalaxy.to` avoid entry (the site is gone) and `sh.st`.
- Bundled x64 `rar` executables should be replaced by a native RAR reader plus CBZ-only output. Writing CBR is legacy: the comic community standard is CBZ.
- **Python 3.8 support:** EOL since Oct 2024. This covers the `typing_extensions`/`urllib3` v1 shims and `Union`/`List` typing throughout.
- The "RSS sync" name for a sitemap scrape is misleading.
- `DownloadService` vs `GCDownloadService` duplication, kept "for future usenet/torrent" (`definitions.py:583`).
- The WeTransfer private API, and MediaFire page scraping, are prone to breakage.
- The in-house MEGA client (1.3k lines) is a large maintenance surface with its own crypto. It could be replaced by an existing library or by `megatools`/`rclone` subprocesses, or dropped.
- Covers stored as DB BLOBs, and the API key in the query string.
- `task_intervals` was migrated to cron (49), but `insert_task_intervals` still carries defaults. `search_all` was removed in migration 50.

---

## 11. Rewrite notes

**What maps directly onto the unified pipeline (a Sonarr-like series/episode model)**
- Volume ≈ Series and Issue ≈ Episode. Issues are monitored per item and volumes have a monitor scheme (`all`/`missing`/`none`) plus `monitor_new_issues`. The following all exist in the same shape: root folders, refresh-and-scan, wanted/missing, queue/history/blocklist, rename templates, a mass editor, library import and remote path mappings.
- **Differences to model in the shared schema:**
  1. **Files ↔ items are M:N**: one archive may cover issues 1–6 or a whole TPB (`issues_files`). Sonarr has multi-episode files, but only contiguous ones within a season.
  2. **Fractional/alphanumeric issue numbers** (`issue_number` text plus `calculated_issue_number` float) and issue *ranges* in release names.
  3. **Special versions** (TPB, one-shot, hardcover, omnibus, volume-as-issue). This is a release-coverage concept, roughly like a Sonarr season pack but orthogonal to numbering, and it needs heuristics or a user lock.
  4. **Volume-level general files** (cover, metadata).
  5. Annuals are separate CV volumes, and the matcher enforces the annual flag.
- **Metadata provider:** ComicVine behind the unified metadata-provider interface.
  - It needs a **per-user API key** with no project proxy, unlike the Servarr SkyHook model.
  - It needs a rate-limit-aware batch fetcher: 100-id filter batches, issue-count-diff skipping, hourly drip refresh.
  - Consider Metron (`metron.cloud`) or GCD as alternates or supplements. Kapowarr has none.
- **Search:** the query-builder + search-FSM + greedy `choose_downloads` design is a good fit for a generic "coverage planner" shared with season-pack logic. Filename parsing (`file_extraction.py`) should become the comic parser plugin, and its test corpus should be ported.
- **Torrent clients:** reuse the unified download-client layer (qBit, Transmission and more come for free). Keep `seeding_handling: copy|complete`, which is equivalent to Sonarr's hardlink/copy-while-seeding. Add **Usenet and Torznab/Newznab for comics** through the unified indexer layer (Prowlarr-equivalent). Kapowarr's code already anticipates this.

**Genuinely new for the monolith**
1. **"DDL" as a third download protocol** alongside usenet and torrent. It needs:
   - **Page-scraping indexers** whose "release" is an article with *multiple download groups × multiple mirror links*. The candidate model becomes Release → Groups (coverage + size) → Links (service, URL). The Newznab model is one release → one NZB or torrent.
   - **A resolution step** ("prepper") that turns a page into concrete downloads at grab time, validates links live, falls back across paths and mirrors, and auto-blocklists dead links or pages. Grab-time resolution does not exist in the Sonarr pipeline.
   - **In-process downloaders per file host** (HTTP with Range resume, MEGA crypto, MediaFire/Pixeldrain/WeTransfer resolvers). These act as an internal "download client" with per-service quota and rate-limit state, and they cancel queued siblings when a quota trips. A per-service credential store (MEGA account, Pixeldrain key) feeds higher quotas.
   - **CloudFlare solving** (FlareSolverr/Byparr) as a shared HTTP-layer capability. Prowlarr/Jackett need it too, so implement it once.
   - **Service preference ordering** (a user-ranked host list, with a large-file override) in place of quality profiles.

   Design this as a pluggable `DirectDownloadSource` (scrape/search, discover-latest, resolve) plus a `HostResolver` registry. It could also serve anime DDL sites or book DDL (the Bookshelf/Readarr analogue), not just comics.
2. **Archive handling / conversion** as a post-import step:
   - magic-byte extension fix
   - zip/cbz ↔ rar/cbr conversion and extract-to-folder
   - splitting multi-issue archives into issues
   - renaming page images

   In Go: use `archive/zip` natively plus a pure-Go RAR reader (e.g. `nwaples/rardecode`) for extraction. **Drop CBR/RAR writing** (it needs proprietary `rar`) and standardise on CBZ. Add PDF/CB7 support, and ComicInfo.xml writing, which Komga and Kavita consume.
3. **Special-version inference** and the coverage planner (see above).

**Drop or redesign**
- The custom supervisor and restart logic: use Go's single binary with graceful reload. Keep the "auto-revert risky hosting/DB changes" UX.
- The hand-rolled migrator: fold it into the monolith's migration system. Write a one-time importer from `Kapowarr.db` covering volumes, issues, files, blocklist and credentials.
- Auth: use the monolith's auth (header API key, sessions/OIDC, on by default). Encrypt stored credentials.
- The CDN socket.io: use the monolith's SSE or WebSocket.
- Hard-coded trackers and dead avoid-lists.

**Risk callout:** GetComics, MediaFire, WeTransfer and MEGA scraping are the most breakage-prone code in the entire *arr set, and they are piracy-adjacent. Keep the DDL layer as an optional, isolated module with fixture-based tests, so that site changes ship as small definition updates rather than core releases. That is the same lesson as Prowlarr's Cardigann YAML. A declarative "DDL site definition" format is worth considering.
