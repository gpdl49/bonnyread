# Bazarr — Codebase Analysis

Source: `submodules/bazarr` (commit `5dedccf28`, 2026-09-26). All paths below are relative to `submodules/bazarr/` unless noted. `SP/` = `custom_libs/subliminal_patch/`.

---

## 1. Overview

**Purpose:** a companion app to Sonarr and Radarr that manages subtitles. It mirrors series, episodes and movies (with their file paths) from Sonarr/Radarr into its own DB. It indexes the external and embedded subtitles already on disk, works out which languages are missing for each item's "languages profile", and searches about 60 subtitle providers. It scores the candidates, downloads the best one next to the video file, and can later upgrade, sync, translate, modify or delete it. It optionally refreshes Plex or Jellyfin afterwards. It never downloads media and never talks to download clients.

**Tech stack**
- Backend: **Python 3.10–3.13**. `bazarr.py:21-35` refuses anything below 3.10 and warns above 3.13. CI tests 3.10–3.14 (`.github/workflows/ci.yml:80`).
  - `bazarr.py` is a supervisor that re-spawns `bazarr/main.py` as a child process. Restart and stop work through marker files (`bazarr/literals.py`).
  - Web stack: Flask 3.1, flask-restx 1.3 (Swagger at `/api/`), Flask-SocketIO 5.6 for UI push (`bazarr/app/app.py:46`, path `{base_url}/api/socket.io`) and waitress 3.0 as the WSGI server (`bazarr/app/server.py`).
  - Data and config: SQLAlchemy 2.0 with Alembic 1.18 through Flask-Migrate; APScheduler 3.11; dynaconf 3.2 over `config/config.yaml`.
  - HTTP and media: requests; guessit 3.8 and rebulk (release parsing); knowit, pymediainfo and enzyme (media probing); ffsubsync 0.5 (sync); pysubs2 and srt (subtitle IO).
  - Integrations: apprise 1.11 (notifications); plexapi 4.18; deep_translator (Google); cloudscraper and js2py (Cloudflare challenges); ga4mp (Google Analytics 4 telemetry).
  - Native pip deps outside `libs/` (`requirements.txt`): lxml, numpy<2.4, webrtcvad-wheels, Pillow, pywin32 (Windows).
- DB: **SQLite** (`{config}/db/bazarr.db`) by default, or **PostgreSQL** (`postgresql.*` settings or `POSTGRES_*` env; `bazarr/app/database.py:30-82`).
- Frontend: React 19.3, TypeScript 7, Vite 8, Mantine 9 (UI kit), @tanstack/react-query 5 and react-table 9, react-router 8, axios, socket.io-client 4.8, and a Workbox PWA (`frontend/package.json`). Node version comes from `frontend/.nvmrc` (24.18.0).
- License GPL-3.0.

**Architecture**

```
bazarr.py (supervisor)
  -> bazarr/main.py
       ├─ Flask app + waitress (UI blueprint app/ui.py, REST blueprint api/ at /api, Socket.IO)
       ├─ APScheduler (app/scheduler.py)  --> jobs_queue (app/jobs_queue.py: pending/running queues, N concurrent)
       ├─ SignalR clients to Sonarr/Radarr (app/signalr_client.py) --> deque --> sync_one_episode / update_one_movie
       ├─ sonarr/ radarr/ : REST pull + DB mirror (sync/*.py)
       ├─ subtitles/ : indexer (disk scan), wanted, download, manual, upgrade, upload, embedded, sync, mods, translate
       └─ subliminal_patch (vendored/forked) : provider pool, video model, scoring, 60 providers
```

**LOC (Python, including blank lines and comments)**

| Area | LOC | Notes |
|---|---:|---|
| `bazarr/` (own backend) | **24,668** | api 6.4k, subtitles 6.7k, app 5.8k, utilities 1.9k, sonarr 1.4k, radarr 1.1k, plex 0.3k, jellyfin 0.3k, languages 0.4k |
| `migrations/` (own) | 1,601 | 16 Alembic revisions |
| `tests/` (own) | 8,855 | 57 files |
| `frontend/src` (own, ts/tsx) | ~43,300 | 378 ts/tsx files |
| `custom_libs/` (**forked third-party, maintained in-tree**) | 43,109 | `subliminal_patch` 27.2k (**providers 22.3k**), `subliminal` 2.1.0 fork 5.4k, `subzero` 3.5k (language + mods), py7zr 0.7 fork 3.4k, signalrcore fork 1.9k, signalr-client-threads 0.4k, deathbycaptcha, libfilebot, pyads, sseclient, flask_compress, `subscene_api` (empty dir) |
| `libs/` (**vendored pip packages, unmodified**) | ~1,308,000 | Biggest: sqlalchemy 239k, pygments 125k, js2py 115k, apprise 108k, wcwidth 66k, unidecode 49k, rich 39k, future 37k, dnspython 32k, alembic 29k, werkzeug 22k, guess_language 21k, idna 20k, plexapi 18k, dynaconf 18k |

Bazarr's own code is small (~25k backend and ~43k frontend). The provider ecosystem (22k) is effectively Bazarr's code even though it sits in `custom_libs/`. Upstream subliminal has diverged and is not used.

---

## 2. Domain model

All tables are declared in `bazarr/app/database.py:118-383` (SQLAlchemy declarative). IDs are **Sonarr/Radarr primary keys**; Bazarr has no identity of its own for media.

- **`table_shows`** (mirror of a Sonarr series). The PK is `sonarrSeriesId`. Columns:
  - Copied from Sonarr: tvdbId, imdbId, title, sortTitle, year, path (unique), overview, poster/fanart (Sonarr MediaCover URLs), alternativeTitles (a stringified Python list), tags (a stringified list of tag **labels**), seriesType, monitored (the **string** `'True'`/`'False'`), ended, lastAired, originalLanguage.
  - Bazarr-owned: `audio_language` (stringified list), `profileId` (FK to the languages profile, **Bazarr-owned**), created/updated timestamps.
- **`table_episodes`** (mirror of a Sonarr episode that has a file). The PK is `sonarrEpisodeId`, with an FK to `sonarrSeriesId`. Columns:
  - From Sonarr: season, episode, absoluteEpisode, tvdbId, title, path, sceneName, monitored.
  - From the Sonarr quality name: format and resolution.
  - From Sonarr mediaInfo: video_codec and audio_codec.
  - From Sonarr: episode_file_id, file_size, audio_language.
  - Bazarr-owned: `missing_subtitles` (stringified list like `['en', 'fr:hi']`), `failedAttempts` (the adaptive-search log), and `ffprobe_cache` (a **pickled** knowit/ffprobe result blob, `bazarr/utilities/video_analyzer.py:278,344`).
- **`table_movies`** (mirror of a Radarr movie that has a file). The PK is `radarrId`. Columns:
  - Same pattern as episodes: tmdbId (Text, unique), imdbId, movie_file_id, path (unique), sceneName, format/resolution/codecs, audio_language, tags, monitored, originalLanguage.
  - Bazarr-owned: profileId, missing_subtitles, failedAttempts, ffprobe_cache.
- **`table_episodes_subtitles` / `table_movies_subtitles`** (new in 2025-12, revision `0124f9e278fb`, which replaced a stringified `subtitles` column). One row per subtitle, **whether external or embedded**: language (alpha2), `hi` (bool), `forced` (bool), path (null when embedded), size, `embedded_track_id`. It has FKs to the episode/series or the movie. This is the result of the disk indexer.
- **`table_languages_profiles`**: profileId, name, `cutoff` (the item id at which the search stops; null means get all), `items` (JSON list of `{id, language, hi: "True"|"False"|"Excluded", forced: "True"|"False", audio_exclude, audio_only_include}`, `frontend/src/types/api.d.ts:27`), `mustContain` / `mustNotContain` (release-name term lists), `originalFormat` (keep the provider's format and skip conversion to SRT), and `tag` (auto-assign the profile by Sonarr/Radarr tag).
- **`table_history` / `table_history_movie`**: action, description, language (e.g. `en:hi`), provider, score, `score_out_of` (360 for episodes and 120 for movies, backfilled by `c00d40af333c`), subs_id, subtitles_path, video_path, matched / not_matched (stringified sets), `upgradedFromId` (a self-FK). Action codes are 0 delete, 1 auto download, 2 manual, 3 upgrade, 4 upload, 5 sync, 6 translate, 7 embedded extract (from the `history_log(...)` call sites).
- **`table_blacklist` / `table_blacklist_movie`**: provider, subs_id, language, timestamp, plus the media FK. It excludes a specific provider subtitle id from future automatic picks (`bazarr/sonarr/blacklist.py`).
- **`table_shows_rootfolder` / `table_movies_rootfolder`**: a mirror of Sonarr/Radarr root folders, with `accessible` and `error` (health check).
- **`table_settings_languages`**: code3 (PK), code2, code3b, name, enabled. This is the global enabled-language list, seeded from pycountry plus custom languages.
- **`table_settings_notifier`**: name (apprise provider name), enabled, url.
- **`table_announcements`** (dismissed announcements) and **`system`** (configured and updated flags).
- **Custom languages** (`bazarr/languages/custom_lang.py`): `pb`/`pob` Brazilian Portuguese, `zt`/`zht` Traditional Chinese, `ea`/`spl` Latin-American Spanish, plus `pt` overrides. They map onto `subzero.language.Language` with `hi` and `forced` flags.
- **Not in the DB:**
  - Provider throttle state goes to `config/throttled_providers.dat`, a stringified dict written to file (`bazarr/app/get_providers.py:575`).
  - Provider caches use a dogpile file cache.
  - Logs go to a rotating file, `log/bazarr.log`.
  - The job queue lives only in memory (`bazarr/app/jobs_queue.py`).

**Migrations:** 16 Alembic files in `migrations/versions/`. The chain is linear, from `dc09994b7e65` (2023-04, baseline) to `537e9b4d10e3` (2026-08, subtitle-table indexes). They are all "batch" migrations, for SQLite compatibility. `init_db()` still calls `metadata.create_all()` first, and `migrate_db()` drops leftover `_alembic_tmp_*` tables before upgrading (`bazarr/app/database.py:385-438`). The pre-2023 (peewee-era) schema has no migrations; it was rebuilt by the baseline.

**Data-quality smells:** booleans are stored as strings (`monitored = 'True'`). Lists are stored as Python `repr` strings and parsed back with `ast.literal_eval` (tags, audio_language, missing_subtitles, failedAttempts, mustContain). Tag filtering uses `LIKE "%'tag'%"` (`get_exclusion_clause`, `database.py:441`). Pickled blobs sit in DB rows.

---

## 3. Feature areas

- **Sync from Sonarr/Radarr** (`bazarr/sonarr/sync/*`, `bazarr/radarr/sync/*`):
  - A full sync (`update_series`, `update_movies`) pulls all series or movies.
  - For each series it then pulls episodes, and on Sonarr v3 to 4.0.9.2420 also episode files (`sonarr/sync/episodes.py:80-91`).
  - It diffs against the DB to insert, update or delete rows, then runs `store_subtitles` (disk index) and, unless deferred, a missing-subtitles search for new files.
  - Real-time updates come through SignalR (§9). Webhooks (§5) are an alternative trigger.
  - Options: `sync_only_monitored_series/episodes/movies`, `excluded_tags`, `excluded_series_types`, `exclude_season_zero`, and `only_monitored` (search filter).
  - Profile assignment on insert (`serie_default_profile`, `movie_default_profile`) or by tag (`serie_tag_enabled` plus `languages_profile.tag`). `remove_profile_tags` unsets the profile.
  - Files are ignored when smaller than `MINIMUM_VIDEO_SIZE` = 20 KB (`bazarr/constants.py:9`), except `.strm` files when `enable_strm_support` is on.
- **Disk indexer** (`bazarr/subtitles/indexer/{series,movies,utils}.py`):
  - Lists embedded subtitle tracks through the ffprobe/mediainfo cache.
  - Finds external subtitle files next to the video, or in a subfolder (`subfolder` = current / relative / absolute plus `subfolder_custom`).
  - Guesses language, HI and forced from the file name (`.en.hi.srt`, `.en.forced.srt`, `.en.cc`, `.sdh`), falling back to content detection (guess_language, chardet).
  - Computes `missing_subtitles` against the profile. Profile items with `audio_exclude` or `audio_only_include` are skipped when the audio language matches or doesn't match.
  - `embedded_subs_show_desired`, `ignore_pgs_subs`, `ignore_vobsub_subs` and `ignore_ass_subs` control which embedded tracks count as "present".
- **Search / download** (`bazarr/subtitles/download.py`, `wanted/*`, `mass_download/*`, `manual.py`, `pool.py`, `SP/core.py`):
  - Builds a subliminal `Video` from the path plus the DB (`refiners/database.py`: series title, year, alt titles, IMDb/TVDB ids, scene name, codecs, resolution, audio).
  - Refiners (`bazarr/subtitles/refiners/__init__.py`): `database`, `ffprobe` (codec, resolution and fps from the cached probe), `arr_history` (fetches Sonarr/Radarr grab history to get `nzbInfoUrl`, but only when AvistaZ/CinemaZ are enabled), `anidb` (the anidb.net HTTP API to map TVDB ids to AniDB ids for anime providers) and `anilist` (Fribb anime-lists JSON).
  - It then runs the enabled providers concurrently (`SZAsyncProviderPool` when `multithreading`), scores the candidates and downloads the best one per missing language above `minimum_score` (90% for episodes and 70% for movies, as percentages of the max).
  - After download it writes to disk, converting to SRT unless `originalFormat` is set, re-encoding to UTF-8 (`utf8_encode`) and applying `chmod`. It then applies mods and optionally sync, writes history and sends notifications, runs the post-processing command, refreshes Plex or Jellyfin, calls the external webhook, and tells Sonarr/Radarr to rescan (§9).
  - Manual (interactive) search lists every candidate with its score and matched/unmatched attributes (`providers/episodes` and `providers/movies` API).
  - Scene name: `use_scenename` refines with Sonarr/Radarr's `sceneName` (`subtitles/utils.py:37-41`).
- **Scoring** (`SP/score.py:31-64`):

  | Attribute | Episode points | Movie points |
  |---|---:|---:|
  | hash | 359 | 179 |
  | series (episode) / title (movie) | 160 | 60 |
  | year | 90 | 40 |
  | season | 30 | — |
  | episode | 30 | — |
  | source | 25 | 30 |
  | edition | — | 30 |
  | release_group | 20 | 15 |
  | audio_codec, resolution, video_codec, hearing_impaired, streaming_service | 1 each | 1 each |
  | **Max** | **360** | **180** |

  - The hash is always worth max − 1.
  - A hash match is only trusted if {series, season, episode, source} also match for episodes, or {video_codec, source} for movies, and never for specials (`score.py:101-130`).
  - Custom score overrides were removed; the old `series_scores` and `movie_scores` config sections are deleted at startup (`bazarr/app/config.py:630-634`).
  - Upgrades require `score < score_out_of - 3`.
- **Languages profiles** (see §2). Profile items carry hi (require / normal / **Excluded**) and forced. `constants.py:14-17` documents that the legacy "only / also / never" values were collapsed.
  - `cutoff` stops further searching once that item is satisfied.
  - `mustContain` / `mustNotContain` filter by release info.
  - `language_equals` (global) treats one language as another, e.g. `pt:pt-BR` or `en@hi:en` (`get_providers.py:152-183`).
  - `single_language` writes `.srt` without a language code.
  - `hi_extension` chooses the suffix: hi, cc or sdh.
  - Global defaults can be applied per tag.
- **Upgrade logic** (`bazarr/subtitles/upgrade.py`): every `upgrade_frequency` hours (default 12) it re-searches history rows from the last `days_to_upgrade_subs` days (default 7).
  - It considers actions 1 and 3 by default, or 1, 2, 3, 4 and 6 when `upgrade_manual` is set.
  - A row qualifies when its score is below max − 3, or when it is a translated subtitle with no score. It takes the latest row per (video_path, language) since the last delete.
  - It skips rows whose language is no longer desired or whose HI requirement has changed.
- **Adaptive searching** (`bazarr/subtitles/adaptive_searching.py`): after `adaptive_searching_delay` (default 3 weeks) of failed attempts for a language, it only searches once every `adaptive_searching_delta` (default 1 week). Attempts are stored per item in `failedAttempts`.
- **Embedded subtitles**:
  - Detection goes through knowit using ffprobe or mediainfo (`embedded_subtitles_parser`, default `ffprobe`; `bazarr/utilities/video_analyzer.py`). Results are cached in `ffprobe_cache` and keyed on file size and file id.
  - Audio-language detection from embedded tracks happens when `parse_embedded_audio_track` is on; otherwise Sonarr/Radarr's `languages` are used.
  - `default_und_audio_lang` and `default_und_embedded_subtitles_lang` handle `und` tracks.
  - **Extraction** of embedded tracks to external files uses `fese` plus ffmpeg (`bazarr/subtitles/embedded.py:70`, history action 7).
  - The **`embeddedsubtitles` provider** (`SP/providers/embeddedsubtitles.py`) also treats embedded text tracks as "downloadable" candidates.
- **Subtitle sync** (`bazarr/subtitles/tools/subsyncer.py`): **ffsubsync** only; there is **no alass**. The reference is the audio track (webrtcvad, or auditok as a fallback VAD), an embedded subtitle track, or the original-language audio (`subsync.use_original_language` / `auto_use_original_language`). `gss` is golden-section search. `max_offset_seconds` defaults to 60, and `no_fix_framerate` is on.
  - Auto-sync runs after download when the score is below `subsync_threshold` (90 for episodes, 70 for movies).
  - A newer **quality checker** (`subsync.checker`, `quality_min_score`, `quality_max_offset_seconds`, `quality_max_framerate_deviation`, and blacklisted providers/languages) rejects bad sync results.
  - "Audio sync" in Bazarr means exactly this audio-reference ffsubsync run. There is no separate audio-alignment feature.
- **Mods** (`custom_libs/subzero/modification/mods/*`, applied through `subtitles/tools/mods.py`, globally via `general.subzero_mods` or per file via `PATCH /api/subtitles`):
  - `common` (common fixes)
  - `remove_tags`
  - `reverse_rtl`
  - `fix_uppercase`
  - `emoji` (removal)
  - `OCR_fixes` (dictionary-based, with data files per language)
  - `remove_HI` (hearing-impaired removal)
  - `color` (fonts)
  - `shift_offset`
  - `change_FPS`
  - `two_point_fit`
- **Translation** (`bazarr/subtitles/tools/translate/services/*`): translates an existing subtitle into another profile language. History action 6 gets `translator.default_score` (50). The backends are:
  - `google_translate` (deep_translator; the default)
  - `gemini` (the `generativelanguage.googleapis.com/v1beta/...:generateContent` REST API, with key rotation over `gemini_keys`, `gemini_batch_size` 300 and model `gemini-2.0-flash`)
  - `lingarr` (a self-hosted Lingarr at `translator.lingarr_url`)
- **Whisper / AI**:
  - The `whisperai` provider (`SP/providers/whisperai.py`) sends extracted audio (ffmpeg) to a self-hosted **whisper-asr-webservice** (`POST {endpoint}/detect-language`, `POST {endpoint}/asr`).
  - `use_whisper_fallback` and `use_whisper_fallback_series` restrict Whisper to being the last resort (`subtitles/wanted/series.py:58`).
  - Providers also expose AI or machine-translated subs as opt-ins: opensubtitlescom (`include_ai_translated`, `include_machine_translated`), subdl (`ai_translate`, `include_ai_translated`), jimaku (`enable_ai_subs`) and subtitlecat.
- **Upload** (`subtitles/upload.py`, action 4, fake max score), **delete** (`tools/delete.py`, action 0) and **subtitle contents viewer** (`api/subtitles/subtitles_contents.py`).
- **Blacklist**: manually blacklisting a history entry deletes the file and records (provider, subs_id). A new search then runs.
- **Scheduled tasks and jobs queue**: see §6. There is a UI job monitor: `GET/POST/PATCH/DELETE /api/system/jobs`, with progress over Socket.IO.
- **Notifications**: **apprise** 1.11 (`bazarr/app/notifier.py`) for about 100 services. The enabled providers and URLs are stored in `table_settings_notifier`. A notification is sent on download, upgrade and manual actions (unless `dont_notify_manual_actions` is set). `notify_if_nothing_is_missing_for_signalr_event` is a separate option.
- **Post-processing**: a shell command (`general.postprocessing_cmd`) with `{{directory}}`, `{{episode}}`, `{{episode_name}}`, `{{subtitles}}`, `{{subtitles_language}}`, `{{episode_language}}`, `{{score}}`, `{{subtitle_id}}`, `{{provider}}`, `{{uploader}}`, `{{release_info}}`, `{{series_id}}` and `{{episode_id}}` (`bazarr/utilities/post_processing.py`). It can be gated by score threshold. There is also an **external webhook** (`general.use_external_webhook`, basic-auth) that is **Autopulse**-oriented (`bazarr/utilities/autopulse_webhook.py:201`).
- **Media servers**:
  - **Plex** (`bazarr/plex/operations.py`, `api/plex/oauth.py`) can refresh the item, or the whole library section, after a subtitle change, and can set `addedAt` to now. Auth is by legacy API key or plex.tv OAuth PIN; the token is encrypted at rest.
  - **Jellyfin** (`bazarr/jellyfin/*`) can refresh the item (`POST /Items/{id}/Refresh`) or the library (`POST /Library/Media/Updated`).
- **Backups** (`bazarr/utilities/backup.py`): a zip of `config.yaml` and `bazarr.db` (SQLite online copy), written to `backup.folder` with a retention of 31 days. The schedule is Daily, Weekly or Manually. **There is no DB dump on Postgres**; it backs up the config only. Restore works by extracting on the next start.
- **Auth** (`bazarr/app/ui.py:53-120`, `api/utils.py:100`):
  - `auth.type` is None, `basic` (HTTP Basic) or `form` (Flask session).
  - The password is stored as **unsalted MD5** (`utilities/helper.py:23`).
  - The REST API is guarded **only by the API key** (`X-API-KEY` header, `?apikey=` or form field). The SPA gets the key injected into the page after login.
  - `/api/system/ping` needs no auth. CORS can be switched on (`cors.enabled`).
- **Health** (`bazarr/utilities/health.py`): Sonarr/Radarr root folders not accessible under the path mapping, duplicate profile-item ids, and no profile existing or assigned.
- **Self-update** (`bazarr/app/check_update.py`): polls GitHub releases (`api.github.com/repos/morpheus65535/Bazarr/releases`), downloads `bazarr.zip`, unzips it over the install and restarts. Announcements come from `cdn.jsdelivr.net/gh/morpheus65535/bazarr-binaries@latest/announcements.json`, plus local ones.
- **Telemetry**: GA4 Measurement Protocol with a hard-coded `measurement_id` and `api_secret` (`bazarr/utilities/analytics.py:28`). `analytics.enabled` defaults to **True**. It tracks provider usage and errors.
- **Misc**:
  - Path mappings (§9).
  - A proxy (`proxy.*`: http, socks4/5) used by provider sessions.
  - Anti-captcha: anti-captcha.com, deathbycaptcha and captchaai (`general.anti_captcha_provider`).
  - `disable_all_providers_ssl_verify`.
  - A file browser for Bazarr, Sonarr or Radarr, used for path mapping.
  - Badges (counts of wanted items).
  - History statistics.
  - A log viewer with include/exclude regex filters.
  - Swagger UI.

---

## 4. Integrations (full lists)

### 4.1 Subtitle providers

Providers are auto-registered by scanning `SP/providers/*.py` for `*Provider` classes (`SP/providers/__init__.py:95-111`). The UI list is `frontend/src/pages/Settings/Providers/list.tsx`.

Status legend:
- **UI**: offered in Settings.
- **Hidden**: code registered but not in the UI, so it can't be enabled any more.
- **Dead?**: the site is known or likely to be gone, or the code is stale.

Technique shorthand: html = scraping, json = API, rar/7z = archive extraction, hash = file-hash matching, auth = login.

| Key | Site / backend | Technique | Status |
|---|---|---|---|
| addic7ed | addic7ed.com | html, auth, anti-captcha, VIP | UI; heavily throttled, captcha-dependent |
| animekalesi | animekalesi.com (TR anime) | html | UI |
| animesubinfo | animesub.info (PL anime) | html | UI |
| animetosho | feed.animetosho.org (needs AniDB client) | json + html | UI |
| animetosho_xyz | feed.animetosho.xyz (mirror) | json | UI |
| assrt | api.assrt.net (ZH) | json, token | UI |
| avistaz / cinemaz | avistaz.to / cinemaz.to (private trackers; base `avistaz_network.py`) | html + json, cookies, rar, hash | UI; uses `arr_history` for `nzbInfoUrl` |
| bayflix | bayflix.sb (BG) | json | UI |
| betaseries | api.betaseries.com (FR/EN, TV only) | json, token | UI |
| bsplayer | api.bsplayer-subtitles.com SOAP | xml/soap, hash | UI; plain http, old SOAP API; **Dead?** (flaky for years) |
| embeddedsubtitles | local ffmpeg/ffprobe (fese) | local | UI |
| gestdown | api.gestdown.info (Addic7ed proxy) | json | UI |
| greeksubs | greeksubs.net | html, cloudscraper | UI |
| greeksubtitles | gr.greek-subtitles.com / greeksubtitles.info | html, rar | UI; plain http; **Dead?** |
| hdbits | hdbits.org API (private tracker) | json, passkey | UI |
| jimaku | jimaku.cc API (JA anime) | json, api key | UI |
| karagarga | karagarga.in + forum (private) | html, auth | UI |
| ktuvit | ktuvit.me (HE) + TMDb lookup | html + json, auth | UI |
| legendasdivx | legendasdivx.pt (PT) | html, auth, rar, cloudscraper | UI; daily limit (Lisbon midnight) |
| legendasnet | legendas.net (BR) | json, auth, rar | UI |
| napiprojekt | napiprojekt.pl (PL) | hash (MD5 of first 10 MB) + html | UI |
| napisy24 | napisy24.pl (PL) | http agent API, auth | UI; plain http |
| opensubtitlescom | api.opensubtitles.com REST v1 | json, auth, hash, **hard-coded API key** in `get_providers.py` | UI; the main provider |
| pipocas | pipocas.tv (PT) | html, auth, rar | UI |
| prijevodionline | prijevodi-online.org (HR/SR) | html, rar, cloudscraper | UI |
| regielive | api.regielive.ro/bazarr (RO; Bazarr-specific endpoint) | json | UI |
| soustitreseu | sous-titres.eu (FR) | html, rar, cloudscraper | UI |
| subclub | subclub.eu (ET) | html, rar | UI |
| subdl | api/dl.subdl.com | json, api key, rar | UI; largest provider file (1,278 LOC) |
| subf2m | subf2m.co (Subscene clone) | html | UI |
| subs4free | subs4free.info (GR) | html, rar | UI; UI itself says "Broken, may not work" → **Dead?** |
| subs4series | subs4series.com (GR) | html, rar, cloudscraper, captcha | UI |
| subsarr | self-hosted slimcdk/subsarr (Subscene archive) | json | UI |
| subsdump | self-hosted D3lphi3r/SubsDump (Subscene archive) | json | UI |
| subsource | subsource.net API | json, api key | UI |
| subsro | api.subs.ro (RO) | json, api key, rar | UI |
| subssabbz | subs.sab.bz (BG) | html, rar | UI; plain http |
| subsunacs | subsunacs.net (BG) | html, rar, 7z | UI |
| subtis | api.subt.is (ES movies) | json | UI |
| subtitlecat | subtitlecat.com (machine translations) | html | UI |
| subtitrarinoi | subtitrari-noi.ro (RO) | html | UI |
| subtitriid | subtitri.do.am (LV) | html, rar | UI |
| subtitulamostv | subtitulamos.tv (ES) | html + json | UI |
| subx | subx-api.duckdns.org (Subdivx wrapper; ES) | json, api key | UI; community-hosted on DuckDNS |
| titlovi | kodi.titlovi.com API (HR/SR/BS) | json, auth, rar, cloudscraper, captcha | UI |
| titrari | titrari.ro (RO) | html, rar | UI |
| titulky | premium.titulky.com (CZ/SK, VIP only) | html, auth, rar | UI; daily limit (Prague midnight) |
| tsukihime | api.tsukihime.org (anime) | json | UI |
| turkcealtyaziorg | turkcealtyazi.org (TR) | html, cookies, cloudscraper | UI |
| vladoonmooo | vladoon.mooo.com (BG) | json | UI |
| whisperai | self-hosted whisper-asr-webservice | json + ffmpeg audio | UI |
| xsubs | xsubs.tv (GR) | html, auth | UI; plain http |
| yavkanet | yavka.net (BG) | html, rar | UI |
| zimuku | srtku.com (ZH; formerly zimuku.la) | html, rar, anti-captcha | UI |
| hosszupuska | hosszupuskasub.com (HU) | html, rar | UI; **Dead** (domain does not resolve; http only) |
| nekur | subtitri.nekur.net (LV) | html, rar | UI; site up (2026-09-27) |
| shooter | shooter.cn (subliminal core provider) | hash | **Hidden / Dead** (shooter.cn closed in 2014) |
| subscenter | subscenter.info (HE) | html | UI; **Dead** (parked domain; both `SP/` and `subliminal/` copies remain) |
| subsynchro | subsynchro.com (FR) | json | UI; site up (2026-09-27) |
| supersubtitles | feliratok.eu (HU) | html + json | **Hidden** (still edited 2026-08; could be revived) |
| tvsubtitles | tvsubtitles.net | html | **Hidden** (the patched version was edited 2026-07) |
| wizdom | wizdom.xyz (HE) + TMDb | json | UI; status unclear |
| yifysubtitles | yifysubtitles.ch | html | UI; site up (2026-09-27) |

Other provider notes:
- **Upstream subliminal core providers** remain in `custom_libs/subliminal/providers/`: addic7ed, argenteam (**dead**, the site closed in 2022), legendastv (**dead**, closed in 2021), napiprojekt, shooter (**dead**), subscenter (**dead**), thesubdb (**dead**, closed in 2021) and tvsubtitles. Only four are registered in `subliminal/extensions.py:104-108` (addic7ed, shooter, thesubdb, tvsubtitles). Bazarr's registry uses the `SP` versions.
- **Removed providers still named in docs:** the README (`README.md:60-114`) still lists **OpenSubtitles.org (VIP)**, **Subscene** and **TuSubtitulo**, none of which have code. `custom_libs/subscene_api/` is an empty directory. The `podnapisi` config section is deleted at startup (`config.py:636`).
- **Throttling** (`bazarr/app/get_providers.py:58-141`):
  - Default backoff: 1 h after `TooManyRequests`, 3 h after a download limit, 20 min when the service is unavailable, 6 h after a parse error, 12 h after an auth or config error, and 1 h after a timeout.
  - Per-provider overrides for opensubtitlescom, addic7ed, titlovi, titrari, titulky, legendasdivx, whisperai, regielive, subsource and subdl, including "until local midnight" in Prague, Lisbon or GMT.
  - Providers that raise `MustGetBlacklisted` are blacklisted automatically.
- `PROVIDERS_FORCED_OFF` (addic7ed, tvsubtitles, legendasdivx, napiprojekt, shooter, hosszupuska, supersubtitles, titlovi, assrt) is only referenced in commented-out code (`get_providers.py:143,215-219`). It is dead.

### 4.2 Other integrations

- **Sonarr v3/v4** and **Radarr v3+** (§9). Legacy code paths remain for Sonarr v2 and Radarr v0.2.
- **Plex**: plexapi 4.18; plex.tv `api/v2/pins`, `api/v2/user`, `api/v2/ping`, `pms/resources`; `app.plex.tv/auth`. Bazarr can **create, list and delete a Plex webhook** in the user's Plex account pointing at `/api/webhooks/plex`.
- **Jellyfin**: its own small client (`bazarr/jellyfin/client.py`: `/System/Info`, `/Library/VirtualFolders`, `/Items`, `/Items/{id}/Refresh`, `/Library/Media/Updated`).
- **Autopulse**: generates an Autopulse config template (`GET /api/plex/autopulse/config`) and calls an external webhook.
- **Notifications**: apprise (the vendored `libs/apprise`, about 100 services).
- **Translation**: Google Translate (deep_translator), Google Gemini REST, Lingarr.
- **Metadata / anime ids**: AniDB HTTP API (`api.anidb.net:9001/httpapi`, needs a registered client), Fribb `anime-list-mini.json` (GitHub raw), TMDb (`api.tmdb.org/3`, used by ktuvit and wizdom), and IMDb HTML scraping. The Plex webhook falls back to scraping `imdb.com/title/{id}` to map episode to series (`api/webhooks/plex.py`).
- **Captcha solvers**: anti-captcha.com (`python_anticaptcha`), deathbycaptcha, captchaai.
- **External tools / binaries**: ffmpeg and ffprobe (probing, extraction, audio for Whisper and ffsubsync), mediainfo (optional parser), unrar (rarfile), 7z (py7zr, in-process), and webrtcvad (C extension).
  - `bazarr/utilities/binaries.json` lists per-OS/arch **downloadable binaries from `github.com/morpheus65535/bazarr-binaries`**, with MD5 checksums: unrar for Linux aarch64/armv5tel/i386/x86_64 and so on, and ffmpeg/ffprobe for Linux i386/x86_64, Windows and macOS.
  - `get_binary()` downloads any that are missing on demand (`bazarr/utilities/binaries.py:43-90`).
- **GitHub** (releases for self-update) and **jsDelivr** (announcements).
- **Google Analytics 4** (telemetry).

---

## 5. External APIs / interfaces

Bazarr's REST API is served by flask-restx under `/api`, with Swagger UI at `/api/`. Every endpoint except `system/ping` requires the API key.

| Group | Endpoints (method → path) |
|---|---|
| Badges | GET `/api/badges` |
| Series | GET/POST/PATCH `/api/series` (list; set profile; actions scan-disk / search-missing / sync), GET `/api/series/tags` |
| Episodes | GET `/api/episodes`; GET `/api/episodes/wanted`; GET `/api/episodes/history`; GET/POST/DELETE `/api/episodes/blacklist`; POST (download) / PATCH (upload) / DELETE `/api/episodes/subtitles` |
| Movies | GET/POST/PATCH `/api/movies`, GET `/api/movies/tags`, GET `/api/movies/wanted`, GET `/api/movies/history`, GET/POST/DELETE `/api/movies/blacklist`, POST/PATCH/DELETE `/api/movies/subtitles` |
| Providers | GET/POST `/api/providers` (status, reset throttling); GET (search) / POST (download) `/api/providers/episodes` and `/api/providers/movies` (interactive search) |
| Subtitles | GET/PATCH `/api/subtitles` (apply mods / sync / translate to a file), GET `/api/subtitles/info` (guess season, episode and lang from file names), GET `/api/subtitles/contents` |
| History | GET `/api/history/stats` |
| Files | GET `/api/files`, `/api/files/sonarr`, `/api/files/radarr` (file browsers; the last two proxy Sonarr/Radarr `filesystem`) |
| System | POST `/api/system` (shutdown/restart), POST `/api/system/account` (login/logout), GET/POST `/api/system/announcements`, GET/POST/PATCH/DELETE `/api/system/backups`, GET `/api/system/health`, GET/POST/PATCH/DELETE `/api/system/jobs`, GET `/api/system/languages`, GET `/api/system/languages/profiles`, GET/DELETE `/api/system/logs`, PATCH `/api/system/notifications` (test), GET `/api/system/ping` (no auth), GET `/api/system/releases`, GET `/api/system/searches`, GET/POST `/api/system/settings` (the whole config plus languages, profiles and notifiers in one form POST), GET `/api/system/status`, GET/POST `/api/system/tasks`, POST `/api/system/webhooks/test` |
| Plex | GET/POST `/api/plex/oauth/pin`, GET `/api/plex/oauth/pin/{id}/check`, GET `/api/plex/oauth/validate`, GET `/api/plex/oauth/servers`, GET `/api/plex/oauth/libraries`, POST `/api/plex/oauth/logout`, POST `/api/plex/encrypt-apikey`, POST `/api/plex/apikey`, GET/POST `/api/plex/test-connection`, GET/POST `/api/plex/select-server`, POST `/api/plex/webhook/{create,delete}`, GET `/api/plex/webhook/list`, GET `/api/plex/autopulse/config` |
| Jellyfin | POST `/api/jellyfin/test-connection`, GET `/api/jellyfin/libraries` |
| **Webhooks (inbound)** | POST `/api/webhooks/sonarr`, POST `/api/webhooks/radarr`, POST `/api/webhooks/plex` |

Non-API UI routes (`bazarr/app/ui.py`):
- `/` and `/<path>`: the SPA, with the API key injected after login.
- `/bazarr.log`.
- `/images/series/<url>` and `/images/movies/<url>`: **proxies of Sonarr/Radarr MediaCover images, with the arr API key appended**.
- `/system/backup/download/<file>`.
- `/api/swaggerui/static/*`.
- `/test/<protocol>/<url>`: an **open-ish HTTP GET proxy** used by the Settings "Test" button to reach `{arr}/api/system/status` or `{arr}/api/v3/system/status`. It is login-guarded but lets the browser make Bazarr fetch arbitrary URLs.

**Push to UI:** Socket.IO at `{base}/api/socket.io`. Event types: `badges`, `series`, `episode`, `movie`, `episode-wanted`, `movie-wanted`, `reset-episode-wanted`, `reset-movie-wanted`, `episode-history`, `movie-history`, `episode-blacklist`, `movie-blacklist`, `jobs`, `task`, `backup` (`bazarr/app/event_handler.py`).

**Webhooks accepted** (all require Bazarr's API key, as `?apikey=` in the URL configured in the arr or Plex):
- **Sonarr** `POST /api/webhooks/sonarr` (`bazarr/api/webhooks/sonarr.py`): the Sonarr "Webhook" connection payload.
  - It reads `eventType` (a `Test` event returns 200), `episodeFiles[].id` and `episodes[].id`.
  - For each episode file id it looks up `table_episodes.episode_file_id`. If the file isn't found and the episode ids line up, it calls `sync_one_episode` (a pull from the Sonarr API).
  - It then runs `store_subtitles` and `episode_download_subtitles`.
  - It always returns 200 so that Sonarr doesn't mark the connection unhealthy.
- **Radarr** `POST /api/webhooks/radarr`: reads `eventType`, `movieFile.id` and `movie.id`. It looks up `movie_file_id` and otherwise calls `update_one_movie(radarr_id)`, then indexes and searches.
- **Plex** `POST /api/webhooks/plex` (multipart `payload`): handles only `media.play` and `playback.started`.
  - It filters by server UUID and library id or name.
  - It maps the Plex Guid (`tvdb://`, `imdb://`) to Bazarr media, falling back to IMDb scraping for episode to series.
  - It then triggers a search when the item is playing (a "search on play").

**Auth summary:** the API key is sent as the `X-API-KEY` header, the `apikey` query parameter or a form field. UI auth is None, Basic or Form, with a single user and an MD5 password. There is no OIDC or external auth, and no per-user model.

---

## 6. Background jobs

APScheduler jobs are defined in `bazarr/app/scheduler.py:200-340`. The interval-job start time is randomized to 75–100% of the interval at startup (`__randomize_interval_task`). "Manually" and "Never" schedules are implemented as a cron job **in a century** (`in_a_century()`). Heavy jobs are pushed onto the in-memory `jobs_queue`, which runs `general.concurrent_jobs` jobs at once (default min(4, CPUs)).

| Job id | Name | Schedule (default) | Condition |
|---|---|---|---|
| `update_series` | Sync with Sonarr | interval `sonarr.series_sync` min (60; values below 15 are forced to 60, `config.py:604`) | use_sonarr |
| `update_movies` | Sync with Radarr | interval `radarr.movies_sync` min (60) | use_radarr |
| `series_full_scan_subtitles` | Index All Existing Episodes Subtitles (disk rescan) | cron Daily 04:00 / Weekly (day 6) / Manually | use_sonarr |
| `movies_full_scan_subtitles` | Index All Existing Movies Subtitles | same, `radarr.full_update*` | use_radarr |
| `wanted_search_missing_subtitles_series` | Search for Missing Series Subtitles | interval `wanted_search_frequency` h (6; a value of 3 is forced to 6) | use_sonarr |
| `wanted_search_missing_subtitles_movies` | Search for Missing Movies Subtitles | interval `wanted_search_frequency_movie` h (6) | use_radarr |
| `upgrade_subtitles` | Upgrade Previously Downloaded Subtitles | interval `upgrade_frequency` h (12) | sonarr or radarr |
| `cache_cleanup` | Cache Maintenance | 24 h | always |
| `check_health` | Check Health | 6 h | always |
| `backup` | Backup Database and Configuration File | cron Weekly, day 6, 03:00 (Daily / Manually) | always |
| `update_bazarr` | Update Bazarr | 6 h if `auto_update`, otherwise manual only | not `--no-update` |
| `update_release` | Update Release Info (GitHub) | 3 h | always |
| `update_announcements` | Update Announcements File | 6 h | always |

**Event-driven (not scheduled):**
- The SignalR consumer threads (two daemon threads polling deques every 100 ms, `signalr_client.py:428-447`).
- The SignalR on-connect handler, which triggers `update_series` or `update_movies` immediately when `series_sync_on_live` / `movies_sync_on_live` is set.
- The webhooks.
- On-play Plex searches.
- Periodic provider un-throttling, which is lazy and checked on each `get_providers()` call.

---

## 7. Configuration surface

- **Files:**
  - `{config_dir}/config/config.yaml` is managed by dynaconf plus about 250 `Validator` defaults (`bazarr/app/config.py:79-420`). A legacy `config.ini` is auto-converted (`config.py:498-523`).
  - `{config_dir}/db/bazarr.db`.
  - `{config_dir}/log/bazarr.log`.
  - `{config_dir}/config/throttled_providers.dat` and `announcements.json`.
  - `{config_dir}/backup/`.
  - `{config_dir}/cache/`: dogpile provider caches and the anidb cache.
  - `{config_dir}/restore/` and `update/`.
- **CLI** (`bazarr/app/get_args.py`): `-c/--config DIR`, `-p/--port`, `--no-update`, `--debug`, `--release-update`, `--dev`, `--no-tasks`, `--no-signalr`, `--create-db-revision`.
- **Env:**
  - `POSTGRES_ENABLED/HOST/PORT/DATABASE/USERNAME/PASSWORD/URL`.
  - `NO_UPDATE`, `NO_CLI`, `BAZARR_CONFIG_BASE_FILE_PATH`.
  - `HTTP(S)_PROXY`, `NO_PROXY`.
  - `SZ_USER_AGENT`, `SZ_HI_EXTENSION`, `SZ_KEEP_ENCODING`, `SZ_DISABLE_SSL_VERIFY`.
  - `ANTICAPTCHA_CLASS` and `ANTICAPTCHA_ACCOUNT_KEY`, which are set internally.
  - `BAZARR_PACKAGE_VERSION/AUTHOR` and `BAZARR_UPDATE_ALLOWED/MESSAGE` (from the package_info file).
- **Sections** (keys and defaults are listed in `config.py`):
  - `general`: about 75 keys. Covers listen ip/port/base_url, path_mappings and path_mappings_movie, providers, scores, search frequencies, upgrade, adaptive searching, subfolder, chmod, embedded-subs handling, post-processing, external webhook, Whisper fallback, language_equals, concurrent_jobs, theme, page_size, analytics and so on.
  - `auth`: apikey, type, username, password (MD5).
  - `cors`, `backup`, `log` (include/exclude filters).
  - `proxy`: type, url, port, username, password, exclude.
  - `postgresql`.
  - `sonarr` and `radarr`: ip, port, base_url, ssl, apikey, http_timeout, sync interval, full_update schedule, only_monitored, excluded_tags, excluded_series_types, exclude_season_zero, sync_only_monitored_*, *_sync_on_live, defer_search_signalr, use_ffprobe_cache.
  - `plex`: about 30 keys, including OAuth token, encryption key and migration flags. `jellyfin`: 9 keys.
  - `translator`: type, gemini_keys/model/batch_size, lingarr_url/token, default_score, translator_info.
  - `subsync`: about 20 keys.
  - `anticaptcha`, `deathbycaptcha`, `captchaai`.
  - One section per provider with credentials or options: opensubtitlescom, addic7ed, napiprojekt, animetosho, anidb, avistaz, cinemaz, subf2m, hdbits, whisperai, legendasdivx, legendasnet, pipocas, ktuvit, xsubs, assrt, napisy24, betaseries, jimaku, titlovi, titulky, embeddedsubtitles, karagarga, subdl, turkcealtyaziorg, subsource, subsarr, subsdump, subx, subsro, subtitlecat.
  - `analytics.enabled`.
- **DB-held configuration:** languages profiles, the enabled-languages table and notifier URLs. The UI posts all of these together with the YAML settings to `POST /api/system/settings`.
- **Startup rewrites of user config** (`config.py:600-647`):
  - Sync intervals below 15 become 60, and a search frequency of 3 becomes 6.
  - `unknown_as_english` becomes `unknown_as_fallback`.
  - The `series_scores`, `movie_scores` and `podnapisi` sections are deleted.
  - `gemini_key` becomes `gemini_keys`.
  - A Plex API-key to OAuth migration runs (`config.py:~1440`).

---

## 8. Platform / infrastructure concerns

- **Python** 3.10–3.13 is supported, with 3.14 in CI. The runtime check is in `bazarr.py:21`. Local announcements still mention dropping 3.8/3.9 for "Bazarr 1.6" (`announcements.py:116-123`).
- **Vendoring:**
  - The whole dependency tree is committed to `libs/` (~1.3M LOC, pinned in `libs/version.txt`).
  - Forked libraries are in `custom_libs/` (`custom_libs/custom_version.txt`):
    - subliminal 2.1.0 "modified specifically for Bazarr".
    - py7zr 0.7.0, modified.
    - signalrcore ≤0.9.5, with a patched FD leak; it is pinned for arr compatibility.
    - signalr-client-threads 0.0.12 with sseclient 0.0.27, both "modified to work with Sonarr v3" and "not used anymore with v4".
    - Flask-Compress 1.14.
    - libfilebot and pyads, installed from git.
    - deathbycaptcha, "only found on gist".
  - Only lxml, numpy, webrtcvad, Pillow and pywin32 are pip-installed (`requirements.txt`), because they are native.
- **Native binaries:** ffmpeg, ffprobe, mediainfo (optional) and unrar. They are auto-downloaded from the `bazarr-binaries` GitHub repo for some OS/arch combinations (`binaries.json`); otherwise they must be on PATH. 7z is handled in-process by py7zr. webrtcvad is a C extension, with auditok as the fallback.
- **Packaging:** a GitHub release `bazarr.zip`, extracted over the install by the self-updater (`check_update.py:149-207`). Docker images come from linuxserver.io and hotio, not from this repo; the repo only has a dev compose in `dev-setup/` (Alpine 3.22 with ffmpeg, mediainfo and p7zip). The Windows installer is external, but the wiki link in announcements points at a "Windows installer reinstall" page. pywin32 is used on Windows.
- **Process model:**
  - A supervisor process runs one Python child process.
  - Threads are used for waitress, the two SignalR clients plus two queue consumers, APScheduler, the jobs-queue worker and the provider thread pool.
  - Because of the GIL, CPU-heavy work (ffsubsync, guessit, OCR mods) competes with the web server.
- **Security smells:**
  - The Sonarr/Radarr API keys travel in **query strings** (`?apikey=`), and TLS verification is disabled for arr calls (`verify=False` everywhere in `sonarr/*`, `radarr/*`, `signalr_client.py`).
  - Passwords are MD5-hashed.
  - The API key is injected into the HTML.
  - GA4 secrets are hard-coded, and so is the OpenSubtitles API key (`get_providers.py:252`).
  - Pickle blobs are stored in the DB.
  - The `/test/` fetch proxy is exposed.
- **Frontend build:** Vite 8, output served by Flask from `frontend/build`. PWA assets.

---

## 9. Interactions with other *arr apps

### 9.1 Sonarr: every call Bazarr makes

- **Base URL:** `url_sonarr()` = `http(s)://{ip}:{port}{base_url}`. `url_api_sonarr()` = `{base}/api/v3/`, or `{base}/api/` when the version starts with `0.` or `2.` (`bazarr/sonarr/info.py:111`).
- **Auth:** always `?apikey=` in the query string, plus the header `User-Agent: SZ_USER_AGENT`. `verify=False`.
- **Timeout:** `sonarr.http_timeout` (60 s).

| # | Method + path (relative to `/api/v3/`) | Purpose / consumer | File |
|---|---|---|---|
| 1 | GET `/api/system/status`, then fallback GET `/api/v3/system/status` | Version detection (cached 60 s). Picks v2 legacy vs v3, v4 SignalR and the `episodeFile` workaround | `sonarr/info.py:33-42` |
| 2 | GET `series` | Full sync of all series | `sonarr/sync/utils.py:72` |
| 3 | GET `series/{id}` | Single series (SignalR, webhook) | same |
| 4 | GET `episode?seriesId={id}&includeEpisodeFile=true` | Episodes plus embedded `episodeFile` (Sonarr ≥ 4.0.9.2421 embeds it) | `sonarr/sync/utils.py:106` |
| 5 | GET `episode/{id}` | Single episode (SignalR `episode` event) | same |
| 6 | GET `episodeFile?seriesId={id}` | Only on Sonarr v3 to 4.0.9.2420, to merge files into episodes | `sonarr/sync/utils.py:140`, `episodes.py:80-91` |
| 7 | GET `episodeFile/{id}` | Single file for a synced episode (v3+) | `episodes.py:258-266` |
| 8 | GET `tag` | Map tag ids to labels (profile-by-tag, exclusions) | `sonarr/sync/utils.py:55` |
| 9 | GET `languageprofile` (**v3 only**; v2 used `profile`; v4 returns `[]`) | Series "audio language" when not probing | `sonarr/sync/utils.py:12-44` |
| 10 | GET `rootfolder` | Mirror root folders; health check that Bazarr can reach them after path mapping | `sonarr/rootfolder.py:19` |
| 11 | GET `filesystem?path=…&allowFoldersWithoutTrailingSlashes=true&includeFiles=false` | Browse Sonarr's filesystem in the path-mapping UI | `sonarr/filesystem.py:14` |
| 12 | GET `history?eventType=1&episodeId={id}` (1 = Grabbed; first page only) | `nzbInfoUrl` for AvistaZ/CinemaZ matching | `sonarr/sync/utils.py:175`, `subtitles/refiners/arr_history.py` |
| 13 | **POST `command`** `{"name":"RescanSeries","seriesId":N}` | After each download, upload, delete or extraction, so that Sonarr picks up the new subtitle as an extra file | `sonarr/notify.py:13` |
| 14 | GET `{base}/api/v3/MediaCover/{seriesId}/poster-500.jpg…` (the `images[].url` from the series resource, joined onto the API base) | UI image proxy | `app/ui.py:155-180` |
| 15 | GET `{base}/api/v3/system/status` (or `/api/system/status`), browser → `/test/` proxy | Settings "Test" button | `frontend/src/apis/raw/utils.ts:19-38` |

**Sonarr fields consumed** (`sonarr/sync/parser.py`):
- Series: `id, title, path, tvdbId, imdbId, overview, images[].coverType/url, alternateTitles[].title, tags, sortTitle, year, seriesType, monitored, ended, lastAired, originalLanguage.name, languageProfileId (v3), qualityProfileId (v2)`.
- Episode: `id, seriesId, title, seasonNumber, episodeNumber, absoluteEpisodeNumber, tvdbId, monitored, hasFile, episodeFileId`.
- Episode file: `episodeFile.{id, path, size, sceneName, languages[].name (or language.name), quality.quality.name/resolution, mediaInfo.videoCodec/audioCodec}`.

### 9.2 Radarr: every call Bazarr makes

`url_api_radarr()` = `{base}/api/v3/`, or `/api/` for `0.x` (`radarr/info.py`). Version detection uses the same two-step `system/status` probe. Radarr `0.x` and `3.x` are flagged "deprecated".

| # | Method + path | Purpose | File |
|---|---|---|---|
| 1 | GET `/api/system/status`, then `/api/v3/system/status` | Version | `radarr/info.py:33-42` |
| 2 | GET `movie` / `movie/{id}` | Full sync / single movie (404 means deleted) | `radarr/sync/utils.py:69` |
| 3 | GET `qualityprofile` (v3) / `profile` (legacy) | Profile id → `language.name` as the movie's "audio language" fallback (legacy semantics) | `radarr/sync/utils.py:10-38` |
| 4 | GET `tag` | Tag labels | `radarr/sync/utils.py:46` |
| 5 | GET `rootfolder` | Root-folder mirror and health | `radarr/rootfolder.py:19` |
| 6 | GET `filesystem?path=…&allowFoldersWithoutTrailingSlashes=true&includeFiles=false` | Path-mapping browser | `radarr/filesystem.py:15` |
| 7 | GET `history?eventType=1&movieIds={id}` | `nzbInfoUrl` for AvistaZ/CinemaZ | `radarr/sync/utils.py:100` (it uses `sonarr.http_timeout`, a bug) |
| 8 | **POST `command`** `{"name":"RescanMovie","movieId":N}` | After a subtitle change | `radarr/notify.py` |
| 9 | GET MediaCover image (poster `-500`) | UI image proxy | `app/ui.py:183-208` |

**Radarr fields consumed** (`radarr/sync/parser.py`):
- Movie: `id, title, tmdbId, imdbId, year, sortTitle, overview, monitored, tags, images[0]/[1].url, alternateTitles[].title (alternativeTitles on legacy), originalLanguage.name`.
- Movie file: `movieFile.{id, path, size, sceneName, languages[].name, quality.quality.name/resolution, mediaInfo.videoCodec/videoCodecID/videoCodecLibrary/audioCodec/audioCodecID/audioProfile/audioAdditionalFeatures (+legacy videoFormat/audioFormat/audioLanguages)}`.
- **Only movies that have a `movieFile` are stored.**

### 9.3 SignalR feeds

The code is in `bazarr/app/signalr_client.py`.

| Client | Used when | Transport / URL | Auth | Subscribed |
|---|---|---|---|---|
| `SonarrSignalrClientLegacy` | Sonarr version starts with `0.`, `2.` or `3.` (the check at `:450`) | classic SignalR (signalr-client-threads + sseclient) at `{sonarr}/signalr`, unnamed hub | `qs={'apikey': key}` | client methods `series`, `episode` |
| `SonarrSignalrClient` | Sonarr v4+ | ASP.NET Core SignalR (signalrcore ≤0.9.5), `{sonarr}/signalr/messages?access_token={key}`; keep-alive 5 s, reconnect every 180 s, unlimited attempts | `access_token` query | hub method **`receiveMessage`** |
| `RadarrSignalrClient` | always (Radarr v3+) | `{radarr}/signalr/messages?access_token={key}` | same | `receiveMessage` |

**Messages used:** `{name: 'series'|'episode'|'movie', body: {action: 'updated'|'deleted'|…, resource: {...}}}`. Every other `name` (queue, command, health, calendar and so on) is dropped.
- `series`: reads `resource.id`, `title` and `year`. It also reads **`resource.episodesChanged`** (Sonarr sets this in `SeriesController` on `SeriesEditedEvent`). If that is set, all episodes of the series are resynced (`sync_episodes`); otherwise `update_one_series(id, action)` runs, which re-GETs `series/{id}`.
- `episode`: reads `resource.id`, `seriesId`, `seasonNumber`, `episodeNumber` and `title`. It calls `sync_one_episode(id)`, which re-GETs `episode/{id}` plus `episodeFile/{fileId}`.
- `movie`: reads `resource.id`, `title` and `year`, plus `action == 'deleted'`. It calls `update_one_movie(id, action)`, which re-GETs `movie/{id}`.

Events are de-duplicated against the previous event of the same type after the `statistics` key is stripped (`feed_queue`). They are then queued, and a search runs unless `defer_search_signalr` is set. On (re)connect a full sync is triggered when `*_sync_on_live` is set.

### 9.4 How Bazarr learns of new files

Bazarr never watches the filesystem for new videos. It relies on:
1. **SignalR `episode` / `movie` events.** When an arr imports a file, the episode or movie resource is re-broadcast. Bazarr sees a new `episodeFileId` or path, inserts or updates the row, runs `store_subtitles`, and searches.
2. **Webhooks** (the Sonarr/Radarr "On Import / On Upgrade" connection pointing at `/api/webhooks/{sonarr,radarr}`). These are needed when SignalR is disabled (`--no-signalr`) or unreliable.
3. **Periodic full sync**, every 60 min by default.
4. **Plex on-play.**

Existing subtitles on disk are found only by the per-item index at sync time and by the daily or weekly full disk scan.

### 9.5 Path mappings

`general.path_mappings` (Sonarr) and `general.path_mappings_movie` (Radarr) are lists of `[arr_path, bazarr_path]` pairs, applied by string replacement with slash normalisation (`bazarr/utilities/path_mappings.py:17-50`). A reverse mapping is used for API responses and history.

Every path in the DB is stored **as the arr sees it**, and mapped on every file access. Health checks verify that each mapped root folder is accessible. The `/api/files/{sonarr,radarr}` browsers exist only to help users build these mappings.

### 9.6 What becomes in-process in a monolith

| Today (separate app) | In a monolith |
|---|---|
| `table_shows`, `table_episodes`, `table_movies`, `*_rootfolder` mirrors plus the sync jobs, SignalR clients, webhooks, dedupe queues and version sniffing | **Deleted.** Subtitle state hangs off the shared `Series`, `Episode`, `EpisodeFile`, `Movie` and `MovieFile` tables through FKs to the **file** (not the episode). The FKs follow file replacement for free. |
| Detecting a new or changed file (SignalR `episode`/`movie`, webhook `episodeFiles[].id`) | Subscribe to the internal "file imported / upgraded / deleted / renamed" domain events |
| `POST command RescanSeries/RescanMovie` so the arr registers the `.srt` as an extra file | Direct call into the extra-files service (Sonarr/Radarr `SubtitleFiles` / `ExtraFiles` tables become the single source of truth for external subtitles, instead of **two parallel indexes**) |
| `GET history?eventType=1` for `nzbInfoUrl` | Direct query of grab history |
| mediaInfo codecs copied from the arr **plus** Bazarr's own ffprobe cache (pickled) | A single media-info probe per file, shared by quality detection, custom formats and subtitles |
| Audio language from arr `languages[]` or Bazarr's own probe | One field |
| Tag and profile lookups by label string (`LIKE "%'tag'%"`) | Real tag FKs, and the languages profile as a first-class profile next to the quality profile |
| Path mappings (arr path vs Bazarr path) | **Gone.** One process sees one filesystem. Remote path mappings for download clients remain an arr concern. |
| MediaCover image proxy with the arr API key | Local media-cover service |
| Separate root folder / health / backup / auth / logs / updates / notifications (apprise vs the arr's Connect providers) | Shared |

---

## 10. Outdated / legacy candidates

| Candidate | Evidence |
|---|---|
| Sonarr v2 and Radarr v0.2 code paths (`/api/` without `/v3`, `profile` endpoint, `alternativeTitles`, `videoFormat`/`audioFormat`/`audioLanguages` fields) | `sonarr/info.py:63-80`, `radarr/info.py:64-81`, `radarr/sync/parser.py` (`is_legacy()` branches), about 20 `is_legacy` call sites. Sonarr v2 has been EOL since 2020. |
| Classic SignalR client for Sonarr ≤ v3 (signalr-client-threads, sseclient, both forked) | `signalr_client.py:37-114`; `custom_version.txt` says "Not used anymore with v4" |
| Sonarr v3 `languageprofile` as the source of "audio language"; Radarr quality-profile `language` | `sonarr/sync/utils.py:12-44`, `radarr/sync/utils.py:10-38`. Language profiles were removed in Sonarr v4, and profile language is legacy in Radarr. |
| `episodeFile` merge for Sonarr < 4.0.9.2421 | `sonarr/sync/episodes.py:80-91` |
| `signalrcore` pinned ≤ 0.9.5 with a local FD-leak patch | `custom_libs/custom_version.txt` |
| Dead providers still registered: hosszupuska, nekur, shooter, subscenter, subsynchro, wizdom, yifysubtitles (not in the UI) | §4.1; `SP/providers/*.py`, the UI list diff |
| Dead upstream subliminal providers: argenteam, legendastv, thesubdb, shooter, subscenter | `custom_libs/subliminal/providers/`, `subliminal/extensions.py:104-108` |
| README lists OpenSubtitles.org, Subscene, TuSubtitulo, Hosszupuska, Nekur, Subscenter, SubSynchro, Wizdom, YIFY, TVSubtitles, Supersubtitles | `README.md:60-114` versus the UI list |
| `custom_libs/subscene_api/` (empty), `podnapisi` config removal, `PROVIDERS_FORCED_OFF` (commented out) | as above; `get_providers.py:143,215` |
| Unused subliminal_patch refiners: `drone`, `filebot` (plus libfilebot), `omdb`, `tvdb`, `symlinks`, `file_info_file`, `metadata` | Registered in `SP/extensions.py:62-68`, but Bazarr never calls `subliminal_patch.core.refine` (it has its own `bazarr/subtitles/refiners`) |
| Custom scoring sections (`series_scores`, `movie_scores`) | Deleted at startup (`config.py:630-634`) |
| `config.ini` → YAML converter; `unknown_as_english`; `gemini_key`; Plex API key → OAuth migration flags | `config.py:498-523, 620-647, ~1440` |
| Python 2 compatibility libraries: `future` (37k LOC), `six`, `past`, `libfuturize`, `libpasteurize`, `backports.functools_lru_cache`, `imghdr.py` shim | `libs/` |
| `js2py` (115k LOC, unmaintained) plus `pyjsparser`, used only for Cloudflare JS challenges through cloudscraper 1.2.58, which is abandoned | `libs/js2py`, `SP/http.py` |
| `pydantic` 1.10 (v1), `jsonschema` 4.17 (2022), `py7zr` 0.7 fork (2020) | `libs/*dist-info`, `custom_version.txt` |
| Google Analytics 4 telemetry on by default with a hard-coded secret | `utilities/analytics.py:28`, `config.py` `analytics.enabled=True` |
| MD5 password hashing; API key in query strings; `verify=False` | §8 |
| Stringified-list and string-boolean columns, pickled ffprobe cache | §2 |
| `Radarr` history call uses `sonarr.http_timeout` | `radarr/sync/utils.py:100` (bug) |
| Announcement about Python 3.8/3.9 and "Bazarr 1.6" while ≥3.10 is already enforced | `announcements.py:116-123`, `bazarr.py:21` |
| Self-updater that unzips over the install (a pattern that conflicts with Docker and package installs) | `check_update.py:149-207` |
| Old Gemini default model `gemini-2.0-flash` | `config.py:206` |

---

## 11. Rewrite notes

### Preserve (behavioural contracts users depend on)

- **Scoring model:**
  - The weights in `SP/score.py:31-60`: episode max 360, movie max 180, hash = max − 1. (120 is only the value migration `c00d40af333c` backfills into old movie history rows.)
  - Hash validity guards: {series, season, episode, source} for episodes, {video_codec, source} for movies, and never for specials.
  - Percent thresholds: `minimum_score` 90 for episodes and 70 for movies.
  - The upgrade rule (score < max − 3 within `days_to_upgrade_subs`).
  - The per-provider `get_matches()` logic, which uses guessit on release names (source, release_group, streaming_service, edition). This is where most match quality comes from; each provider has its own release-name heuristics.
  - Stored history uses `score_out_of` so that old scores stay comparable.
- **Language semantics:**
  - The ISO 639-1/2 codes plus Bazarr's custom codes (`pb`, `zt`, `ea`).
  - `hi` has three values: True (require), False (normal, accept either) and Excluded. `forced` is a separate flag.
  - `language_equals` mappings.
  - Filename conventions: `name.<lang>[.hi|.cc|.sdh][.forced].<ext>`, single-language mode, and the subfolder modes.
  - Audio-based exclusion (`audio_exclude`, `audio_only_include`) and `cutoff`.
  - `mustContain` / `mustNotContain`.
  - "Original format" (don't convert to SRT).
  - The UTF-8 re-encode and the chmod.
- **Provider behaviours:**
  - Per-provider throttle and backoff tables, including the "until local midnight" resets and captcha handling.
  - Blacklist by (provider, subs_id).
  - Adaptive searching.
  - Anime id mapping (AniDB and the Fribb list) for anime providers.
  - The `nzbInfoUrl` trick for AvistaZ/CinemaZ, which becomes trivial in-process.
- **Embedded-track handling:** treat embedded text tracks as present, per profile. Ignore PGS, VobSub and ASS as configured. Treat `und` tracks with a default language. Extract to external files.
- **Post-download pipeline order:** write → mods → sync (score-gated, with the quality checker) → history → notify → post-process command → media-server refresh.
- **History action codes** (0–7), so that history and statistics imports and API consumers stay meaningful.
- **Bazarr's own API as an external contract:**
  - Low priority. Its consumers are mainly the Bazarr UI, plus third-party dashboards and mobile clients (e.g. Homepage widgets, nzb360), which, as far as is known, read wanted, badge and status data. This was not verified in this repo.
  - The **inbound webhooks** `/api/webhooks/{sonarr,radarr,plex}` are only needed for external arrs, so they disappear in a monolith. The Plex one could be kept as a generic "on play" trigger, since Seerr and the monolith will already know Plex.

### Accidental complexity from being a separate app (drop)

- The mirrored media DB, the three sync paths (a periodic full sync that makes N+1 HTTP calls per series, SignalR with two client libraries and version sniffing, and webhooks), dedupe queues, root-folder mirroring, and the `episodesChanged` handling.
- **Path mappings** (two lists), the file browsers for arr filesystems, and the health checks for mapped paths.
- `RescanSeries` / `RescanMovie` round-trips so the arr notices the `.srt`. The arr and Bazarr keep **two separate indexes of the same subtitle files**: Sonarr/Radarr `SubtitleFiles`/`ExtraFiles` and Bazarr's `table_*_subtitles`. Merge them into one.
- Two media-info probes of the same file (the arr's ffprobe `MediaInfo` and Bazarr's knowit/ffprobe pickled cache).
- Re-implemented shared plumbing: auth (weaker), backups (none on Postgres), notifications (apprise instead of the arr Connect providers; see the duplication notes in `comparison.md`), log viewer, self-updater, scheduler UI, Plex/Jellyfin refresh (the arrs already have Plex/Jellyfin/Emby "update library" notifications), and MediaCover proxying.
- Series/movie split tables (`table_history` vs `table_history_movie`, blacklist ×2, subtitles ×2, wanted ×2, API ×2). One subtitle domain keyed on a media file removes half the code.

### Duplication with other apps (for the cross-app pass)

- **Notifications:** apprise here versus the Sonarr/Radarr/Prowlarr Connect providers. A monolith should pick one. Apprise's coverage (~100 services) is a strong argument, but it is Python. Go options are shoutrrr or a port.
- **Media probing:** knowit/ffprobe here versus Sonarr/Radarr's FFProbe-based `MediaInfo`.
- **Release-name parsing:** guessit (Python) here versus the arrs' .NET parsers. Subtitle matching needs the same fields (source, group, edition, streaming service). A single Go parser should serve both. D1 (regex engine) applies too: guessit and rebulk patterns are Python regex.
- **Language model:** the arrs have their own `Language` enum; Bazarr has ISO codes plus hi/forced. The monolith needs one language type with ISO 639 codes and flags.
- **Plex/Jellyfin clients and Plex OAuth:** duplicated with Seerr (Plex OAuth, Jellyfin auth) and with the arrs' media-server notifications.
- **Tags, profiles-by-tag, root folders, backups, auth, scheduler, health, logs, updates:** the same NzbDrone-style infrastructure, re-implemented in Python.

### Porting the provider ecosystem to Go: feasibility

- **Scale:** about 65 provider modules and 22k LOC; the UI offers 55, of which about 50 are live remote providers. About 35 modules scrape HTML (BeautifulSoup/lxml), about 25 call JSON APIs (some do both), and about 30 need archive extraction (rar in ~30, 7z in 1). About 10 use cloudscraper or Cloudflare workarounds, and 5 need a captcha solver.
- **Straightforward in Go:**
  - The JSON-API providers: opensubtitlescom, subdl, subsource, jimaku, gestdown, animetosho, tsukihime, subsro, subtis, subx, assrt, betaseries, hdbits, regielive, bayflix, vladoonmooo, subsarr, subsdump and whisperai. They are about 150–650 LOC each.
  - Hashing: the OpenSubtitles hash, and the napiprojekt MD5 of the first 10 MB.
  - Subtitle formats: go-astisub, or a small SRT/ASS/SSA/VTT/MicroDVD converter.
- **Harder:**
  - The HTML scrapers (goquery works, but each is brittle and region-specific, and the maintainers are scattered).
  - RAR extraction: nwaples/rardecode is pure Go and avoids the unrar binary. 7z has bodgit/sevenzip.
  - Cloudflare challenges: there is no js2py equivalent. Drop them, or use an external FlareSolverr, which Prowlarr/Jackett users already run. This duplicates Prowlarr's FlareSolverr proxy concept, which is good for reuse.
  - The captcha services (plain HTTP APIs, easy).
  - Encoding detection: chardet → saintfile/chardet or `golang.org/x/net/html/charset` plus heuristics.
  - Language detection from content: guess_language → lingua-go or whatlanggo.
- **Hardest / keep external:**
  - **ffsubsync** (numpy FFT alignment plus a VAD). Reimplementing it in Go is a real project. The options are to shell out to `ffsubsync` (a Python dependency again), to use **alass** (a Rust CLI and a single static binary, arguably better suited), or to write a native Go FFT aligner later.
  - The **subzero mods**: OCR fixes are dictionary-driven with data files, HI removal and colour are regex-heavy, and FPS/offset/two-point-fit are trivial. The port is moderate (~3.5k LOC) and needs the dictionaries carried over.
  - Translation and Whisper are plain HTTP, so they are easy.
- **Recommendation:**
  - Port the provider framework (interface, pool, throttle map, blacklist, scoring, matching) and the top ~15 providers by usage first: opensubtitlescom, subdl, subsource, embedded, gestdown/addic7ed, whisperai, animetosho, jimaku, and the regional ones with active users.
  - Freeze or drop the hidden and dead ones.
  - Consider a **provider plug-in boundary**: an HTTP or sidecar protocol, or data-driven definitions in the style of Cardigann YAML for the simple HTML scrapers, so the long tail can be community-maintained without recompiling. That mirrors how Prowlarr and Jackett handle indexers (see D1 and Cardigann in `comparison.md`).
  - The guessit-based matching must be replaced by the monolith's shared Go release parser, which is the largest hidden dependency of the provider port.
