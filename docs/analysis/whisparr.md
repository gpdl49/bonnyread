# Whisparr (v2) — Codebase Analysis

Source: `submodules/whisparr`, branch `v2-develop`, commit `c55fdc66` (2026-09-19, tagged `v2.2.0-develop.404`). The working tree is a shallow clone, so history facts come from `gh api repos/Whisparr/Whisparr`. Paths are relative to `submodules/whisparr/`, and `Core/` means `src/NzbDrone.Core/`. File-level comparisons against Sonarr use `submodules/sonarr` (commit `a8a82905`, 2026-09-26), the same checkout that `sonarr.md` analyses.

**Summary.** Whisparr v2 is a **Sonarr fork** (tracking Sonarr `v5-develop`). It models a studio/site as a Sonarr *Series*, a scene as an *Episode*, and the release year as the *Season*. Scenes are matched almost entirely by **release date** (the same path Sonarr uses for daily shows), with performer and title tie-breaking when a site released several scenes on one date. JAV releases are matched by product code ("External Id"). Metadata comes from a Whisparr-hosted proxy, `api.whisparr.com/v3`, which returns ThePornDB (TPDB) IDs. Whisparr v3 ("Eros") is a separate **Radarr-based** codebase with first-class Studio and Performer entities and StashDB support (see §1.4).

---

## 1. Overview

### 1.1 Stack and versions
- **Backend:** C# on **.NET 10** (`net10.0`; `net10.0-windows` for the tray app). It was bumped to 10.0.12 in the HEAD commit, and to .NET 10 at all only on 2026-08-27 (#1197). Packages match Sonarr's current set (see `sonarr.md` §1), with a few minor-version differences:
  - Dapper 2.1.79 (Sonarr 2.1.89), Polly 8.7 (Sonarr 8.8), MailKit 4.17 (Sonarr 4.18).
  - FluentMigrator 8.0.1, System.Data.SQLite 2.0.4 + SourceGear.sqlite3 3.53.4, Npgsql 10.0.3.
  - Sentry 5.16.3, Swashbuckle 10.2.3, and OIDC (`Microsoft.AspNetCore.Authentication.OpenIdConnect`, added 2026-09-19 in #1324, cherry-picked from `Sonarr/Sonarr@9f0cb9284`).
  - `Semver` 3.0 is an addition, used by the GitHub update provider.
- **Frontend: behind Sonarr.**
  - It still uses **webpack 5**, **Redux 4 / react-redux 7**, `connected-react-router` and **react-router 5**. Sonarr now uses Vite 8, has dropped Redux, and is on react-router 7.
  - React 18.3, TypeScript 5.7. There are 1,166 .ts/.tsx files against **161 .js files**; Sonarr has 17 .js files left.
  - `@tanstack/react-query` 5 is being adopted incrementally (#1211–#1219).
  - `frontend/src/Utilities/Fetch/fetchJson.ts:38` deliberately diverges from upstream's `/api/v5`: `apiRoot = window.Whisparr.apiRoot`, which is `/api/v3` (`src/Whisparr.Http/Frontend/InitializeJsonController.cs:48`).
- **Versioning:** `VERSION: 2.2.0` in `.github/workflows/build.yml` (branches `v2-develop` and `v2`). `package.json` says `2.0.0`. `azure-pipelines.yml` is stale (`dotnetVersion: '6.0.427'`, branches `develop/master/v1`).
- **Database:** SQLite `whisparr2.db` (`src/NzbDrone.Common/Extensions/PathExtensions.cs:15`) or Postgres, plus `logs.db`. Default port **6969** (`Core/Configuration/ConfigFileProvider.cs:206`). App data folder is `Whisparr`.

### 1.2 Size (LOC of .cs files / file count)

| Project | Whisparr LOC | Files | Sonarr LOC (for ref.) |
|---|---:|---:|---:|
| NzbDrone.Core | 91,340 | 1,323 | 111,669 / 1,657 |
| Whisparr.Api.V3 | 10,453 | 155 | 10,733 (+ V5 10,535) |
| Whisparr.Http | 4,020 | 76 | 4,241 |
| NzbDrone.Common | 13,921 | 174 | 14,210 |
| NzbDrone.Host | 1,474 | 19 | 1,638 |
| NzbDrone.Update / Mono / Windows | 657 / 1,303 / 296 | 10 / 13 / 3 | same |
| Whisparr.RuntimePatches / SignalR | 524 / 94 | 6 / 3 | same |
| NzbDrone.Core.Test | 50,547 | 362 | 65,130 / 451 |
| frontend/src (ts/tsx/js) | ~89,500 | ~1,327 | ~85,300 / ~910 |

**Duplication measured file by file.** The comparison was run against the Sonarr checkout after substituting `Sonarr→Whisparr`:
- Of Core's 1,323 files, **1,282 share a path with Sonarr**. **875 of those (68%) are byte-identical**. The 407 that differ account for about 12.9k diff lines in total.
- Only **41 files are Whisparr-only**: 24 are migrations; the rest are ThePornDb import list ×5, Stash notification ×4, `Parser/ExternalIdPattern.cs`, `Update/GithubUpdatePackageProvider.cs`, `Organizer/BasicNamingConfig.cs`, `History/HistoryData*`, `Tv/EpisodeAddedService.cs`, `Validation/ProfileExistsValidator.cs` and `Languages/LanguagesBelowCutoff.cs`.
- **375 Sonarr files are absent**, mostly deliberate removals: DataAugmentation (Scene/XEM/DailySeries), Trakt, Simkl, AniList, MAL, anime and season-pack logic, and newer upstream features not yet picked (§11.2).
- The largest diffs are the genuinely adult-specific code:
  - `Parser/Parser.cs` (1,057 diff lines)
  - `Indexers/Newznab/NewznabRequestGenerator.cs` (628)
  - `Organizer/FileNameBuilder.cs` (587)
  - `IndexerSearch/ReleaseSearchService.cs` (462)
  - `Parser/ParsingService.cs` (455)
  - `Extras/Metadata/Consumers/Xbmc/XbmcMetadata.cs` (358)
  - `ImportLists/ImportListSyncService.cs` (326)
  - `Tv/EpisodeRepository.cs` / `EpisodeService.cs` (~260 each)

### 1.3 Lineage and upstream lag
- **Upstream is Sonarr, not Radarr.** README: "Version 2.x (based on Sonarr), focuses on studios". The fork kept every Sonarr namespace (`NzbDrone.Core.Tv.Series/Episode`, `SeriesTypes`, `SkyHookProxy`).
- **The sync mechanism is cherry-pick based, not merge based.** `.github/workflows/upstream_pull.yml`:
  - Runs daily against `upstream/v5-develop` (`https://github.com/Sonarr/Sonarr.git`).
  - Finds the sync point from `Sonarr/Sonarr@<sha>` provenance lines in commit bodies.
  - Opens an issue per upstream commit. Commits that apply cleanly also get a PR; conflicted commits get an issue only.
  - The header says a real merge "produces roughly 1550 conflicted files and reintroduces anime, season, Trakt and TVDB code that was removed on purpose."
- **History of the lag:**
  - Batches 11–16 (upstream March–August 2024) were applied in Jan–Feb 2026, so the fork was roughly 18–22 months behind at that point.
  - Batches 17–43 (upstream Sept 2024 → Aug 2026) were applied in a burst between 2026-08-25 and 2026-08-29 (#1139–#1221). That burst covered the TypeScript conversions, react-query, quality size limits moving into profiles, and .NET 8 → 10.
- **Current lag:**
  - The sync point is **`Sonarr/Sonarr@900cc21d3` (2026-08-29)**, from the batch 43 commit body. Sonarr `v5-develop` HEAD (`a8a82905e`, 2026-09-26) is **54 commits ahead** (`gh api repos/Sonarr/Sonarr/compare/900cc21d3...v5-develop`).
  - Of 100 `sonarr-pull` issues and PRs, **24 conflicted issues are still open**.
  - Commit-level lag is therefore small (about 1 month). **Feature-level lag is larger**, because many upstream features were skipped or dropped: no API v5, the old frontend stack, and no quality ranks or release-type or multi-stream mediainfo (see §11.2).
- **The schema baseline is squashed.** Whisparr's `001_initial_setup.cs` recreates a Sonarr-v4-era schema; `014_monitor_new_items` corresponds to Sonarr's `200_monitor_new_items`, which dates the fork to roughly mid-2023 Sonarr v4 develop. Migrations numbered 202 and above re-use upstream numbers, but **the numbering diverges** (§2.3).

### 1.4 The `eros` branch and Whisparr v3 (context only)
- The `eros` branch in `Whisparr/Whisparr` (last commit 2026-08-28, only translation updates) and the active repo **`Whisparr/Whisparr-Eros`** (`eros-develop`, pushed 2026-09-24, `package.json` 3.1.0) hold **Whisparr v3, a Radarr fork**. Its layout is `Core/Movies`, `MovieStats`, `LibrarySearch` and `Statistics`.
- `Movies/MovieMetadata.cs` has `ForeignId`, `TpdbId`, `StashId`, `TmdbId`, `ImdbId`, `Code`, `ItemType` (scene vs movie), `MetadataSource`, `StudioForeignId`/`Studio`, and `PerformerForeignIds`/`PerformerNames`.
- It has first-class `Movies/Performers/` and `Movies/Studios/` entities with add/refresh services.
- Cloud endpoints (`NzbDrone.Common/Cloud/WhisparrCloudRequestBuilder.cs` on that branch):
  - `https://api.whisparr.com/v4/{route}`
  - `https://stashdb.org/graphql`
  - `https://api.themoviedb.org/...`
  - `https://whisparr.servarr.com/v1/`
- The rest of this document covers v2 only.

---

## 2. Domain model

### 2.1 Entity mapping (Sonarr name → Whisparr meaning)

| Sonarr class/table | Whisparr meaning | Notes |
|---|---|---|
| `Series` (`Core/Tv/Series.cs`) | **Site / studio** | `TvdbId` holds the **TPDB site id** (the name is kept for API compatibility; `SkyHookProxy.cs:176` sets `series.TvdbId = show.ForeignId`). `Network` holds the parent studio/network. `SeriesType` is `Standard=0 \| Jav=1` (`Core/Tv/SeriesTypes.cs`), replacing Standard/Daily/Anime. Removed: TvRageId, TvMazeId, ImdbId, TmdbId, MalIds, AniListIds, AirTime, UseSceneNumbering semantics, OriginalCountry, SeasonFolder (mig 002). |
| `Season` (embedded JSON) | **Release year** | `SeasonNumber` = year (`SkyHookProxy.cs:204-207`, `MapEpisode`: `episode.SeasonNumber = oracleEpisode.Year`). Has `Monitored` and `Images`. |
| `Episode` (`Core/Tv/Episode.cs`) | **Scene** | `TvdbId` = TPDB scene id. **`ExternalId`** (mig 015, index mig 221) = product code such as a JAV ID, stored lowercase. `AirDate`/`AirDateUtc` = release date. **`Actors`** is a JSON list of embedded `Actor {TpdbId, Name, Character, Gender(Female/Male/Other), Images}` (`Core/Tv/Actor.cs`). `AbsoluteEpisodeNumber` is kept. Removed in mig 008: `EpisodeNumber`, scene numbering, AiredBefore/After, UnverifiedSceneNumbering. |
| Performer | **Not an entity.** | It exists only as embedded `Actors` on episodes, and in the "TPDb Performer" import list (by TPDB performer GUID or int). There is no Performer table and no performer monitoring. |
| `EpisodeFile` | Scene file | Same as Sonarr, minus ReleaseHash and ReleaseType (`Core/MediaFiles/EpisodeFile.cs`). |
| `NamingConfig` | Adds `JavEpisodeFormat` (mig 016), drops `SeasonFolderFormat` (mig 002) | `Core/Organizer/NamingConfig.cs` |
| `ImportListDefinition` | `ShouldMonitor` is `None \| SpecificEpisode \| EntireSite`; adds `SiteMonitorType` | `Core/ImportLists/ImportListDefinition.cs:48` |
| `ImportListExclusions` | `TvdbId` changed to Int32 (mig 017) | Keyed on the TPDB site id |
| `Qualities` | Sonarr's list **plus `VR`** (id 22, source `VR`, 1080; matched by `VR180`) | `Core/Qualities/Quality.cs:126`, `Parser/QualityParser.cs:19,132` |

All other tables match Sonarr's `TableMapping.cs` (Config, RootFolders, ScheduledTasks, providers plus their `*Status` tables, History, DownloadHistory, Blocklist, PendingReleases, extra files, Tags, ReleaseProfiles, DelayProfiles, Users, Commands, CustomFilters, UpdateHistory, AutoTagging, CustomFormats, QualityProfiles/Definitions).

**Missing compared with Sonarr:**
- `SceneMappings`
- `ImportListItems`
- `QualityProfileQualityRanks`

### 2.2 How it maps onto the unified model
- The model is effectively **Sonarr "Daily" series with the season number set to the year**:
  - A site is a container (monitoring, root folder, profile).
  - A scene is a leaf with a date, an optional product code, and a list of performers.
- Multi-episode files and season packs are effectively disabled: the specs and aggregators are removed (§11.2). A scene maps to one file.
- The JAV `SeriesType` changes only two behaviours:
  - the naming template (`FileNameBuilder.cs:206`);
  - `ExternalId` is the primary key for matching.

### 2.3 Migrations (`Core/Datastore/Migration/`, 32 files)
- `000_database_engine_version_check`, `001_initial_setup` (squashed baseline).
- **002–018 are Whisparr-specific:**
  - `002_remove_season_folders`, `003_remove_rarbg`, `004_add_result_to_commands`, `005_add_on_series_add_to_notifications`, `006_add_colon_replacement_to_naming_config`, `007_health_restored_notification`
  - **`008_clean_episode_model`**
  - `009_import_list_monitor`, `010_add_download_client_tags`, `011_add_notification_status`, `012_parse_language_tags_from_existing_subtitle_files`, `013_list_add_missing_search`, `014_monitor_new_items`
  - **`015_add_episode_external_id`**, **`016_add_series_type_and_jav_format`**, **`017_import_exclusion_type`**, **`018_series_last_aired`**
- **202–221 are upstream-derived, but numbered differently from Sonarr:**
  - `202_add_indexer_flags`, `209`–`215` (these match Sonarr numbers).
  - `216_telegram_link_preview` (Sonarr 218), `217_add_size_to_quality_profiles` (Sonarr 207), `218_add_exclusion_tags_to_release_profiles` (Sonarr 221), `219_add_source_to_blocklist` (Sonarr 223), `220_add_air_date_filtering_to_release_profiles` (Sonarr 226).
  - `221_add_episode_external_id_index` is Whisparr's own.
- **Upstream migrations not present:** 203 release_type, 204 release_hash, 206 tmdbid, 217 mal/anilist, 219 nzb.su, 220 season-pack seeding, 224 release-profile indexer ids, 225 mediainfo multi-streams, 227 original_country, 230 scene-mapping id, 231 tag-existing importlists, 232 season-pack air date, 233 quality ranks.
- **Consequence:** the DB schema is **not Sonarr-compatible**. A one-time importer for a unified app needs its own mapping and cannot reuse a Sonarr importer as-is.

---

## 3. Feature areas

Top-level `Core/` folders. **S** = shared with Sonarr (identical or near-identical); **A** = adult-specific changes; **S−** = shared but with parts removed.

- **Analytics** (S): gates Sentry and the update check.
- **Annotations** (S): field metadata for provider settings.
- **Authentication** (S): Forms, External and OIDC (new).
- **AutoTagging** (S−): Genre, Monitored, Network, OriginalLanguage, QualityProfile, RootFolder, Status. OriginalCountry, SeriesType, Tag and Year are removed.
- **Backup** (S): also recognises `whisparr2.db` / `nzbdrone.db` (`Backup/BackupService.cs:148-154`).
- **Blocklisting** (S).
- **Configuration** (A, small): `WhisparrMetadata` and `GithubOwnerRepo` in config.xml (§7). Sonarr's list-sync level/tag, season-pack upgrade and time zone are gone.
- **CustomFilters** (S).
- **CustomFormats** (S−): no ReleaseType spec.
- **Datastore** (A): own migration chain (§2.3).
- **DecisionEngine** (S−): anime/season/scene-mapping/split-episode specs removed (§11.2). `AirDateSpecification` (release-profile air-date restriction) is shared with upstream.
- **DiskSpace** (S).
- **Download** (S + A):
  - All clients are identical.
  - `CompletedDownloadService.cs:102-122,363-374` falls back to matching completed downloads by an ExternalId parsed from the download title.
  - `TrackedDownloadService` and `PendingReleaseService` diverge (partly lag).
- **Exceptions** (S).
- **Extras** (S + A): Kodi/XBMC NFO writer adds performers, studio and TPDB IDs "for Stash compatibility" (§4.5).
- **HealthCheck** (S): identical check list (28 checks). Its `Services` calls go to `whisparr.servarr.com/v1`.
- **History** (S): `HistoryDataKeys`/`HistoryDataExtensions` are helper additions.
- **Housekeeping** (S): identical housekeepers.
- **Http** (S).
- **ImportLists** (A): TPDb Performer and Whisparr lists; the TV-specific lists are removed (§4.4).
- **IndexerSearch** (A): date-, performer-, title- and external-ID-based criteria (§3.4).
- **Indexers** (A): Newznab/Torznab request generation rewritten. BTN, Nyaa and Torrentleech removed.
- **Instrumentation** (S): Sentry DSN `sentry.servarr.com/38|40`.
- **Jobs** (S−): no scene-mapping task (§6).
- **Languages** (S): `LanguagesBelowCutoff.cs` is a leftover from language profiles.
- **Lifecycle, Messaging, ProgressMessaging, Security, Tags, ThingiProvider, Validation, RemotePathMappings, RootFolders, Queue, SeriesStats, DiskSpace** (S).
- **Localization** (S + A): 44 locale files. `en.json` has about 2,216 lines, about 96 of them "Site" keys, reworded from Series to Site.
- **MediaCover** (S): images come from URLs returned by the metadata proxy.
- **MediaFiles** (S−):
  - Import specs for absolute numbering, full season, split episode and unverified scene numbering are removed.
  - `AggregateEpisodes.cs:100-190` adds `TryMatchByExternalId`.
  - `DownloadedEpisodesImportService.cs:246-273` maps a folder to a site by ExternalId.
  - No multi-stream mediainfo models.
- **MetadataSource** (A): a single `SkyHook/SkyHookProxy.cs` pointed at the Whisparr proxy (§3.1).
- **Notifications** (S + A): Stash added, Trakt removed.
- **Organizer** (A): site, release-date, performer and external-ID tokens (§3.3).
- **Parser** (A): a date-first scene parser plus external-ID patterns (§3.2).
- **Profiles** (S−): no quality ranks. Release profiles have a single `IndexerId` (not upstream's `IndexerIds`).
- **Qualities** (A, tiny): `VR`.
- **Tv** (A): domain model (§2), date/performer episode matching, and `EpisodeAddedService` (an older upstream name).
- **Update** (A): `GithubUpdatePackageProvider` reads GitHub Releases of `whisparr/whisparr` instead of `services.sonarr.tv/v1/update`.
- **Removed Sonarr folders:** `DataAugmentation/` (Scene mappings, XEM, DailySeries) and `Statistics/`.

### 3.1 Metadata source (deep dive)
- **The only metadata backend is the Whisparr-hosted proxy `https://api.whisparr.com/v3/{route}`.** The default is set in three places:
  - `Core/MetadataSource/SkyHook/SkyHookProxy.cs:43`
  - `Core/Configuration/ConfigFileProvider.cs:110`
  - `src/NzbDrone.Common/Cloud/WhisparrCloudRequestBuilder.cs:19`
- The URL can be overridden with the `WhisparrMetadata` key in config.xml; `{route}` is appended if missing. `SkyHookProxy.cs:54` writes the final URL to `Console.WriteLine`, which is debug leftover.
- **Routes used:**
  - `GET site/{id}` returns `ShowResource` with embedded `Episodes[]` (`GetSeriesInfo`). A 404 raises `SeriesNotFoundException`.
  - `GET site/search?q=<lowercased title>` for search.
  - Lookup terms `tpdb:<id>` or `tpdbid:<id>` fetch a site directly (`SkyHookProxy.cs:100-124`).
  - `GET performer/{performerId}/scenes` returns `[{SiteId, EpisodeId, SiteName}]`, used by the TPDb Performer import list (`Core/ImportLists/ThePornDb/TPDbImportProxy.cs:62-78`).
- **Resources** (`MetadataSource/SkyHook/Resource/`):
  - `ShowResource`: ForeignId, Title, Overview, Slug, FirstAired, Status (only `"ended"` is recognised), Runtime, TimeOfDay, Network, OriginalLanguage, Genres, **Types** (`"JAV"` sets `SeriesType=Jav`), ContentRating, Rating, Images, Years, Episodes.
  - `EpisodeResource`: ForeignId, **ExternalId**, **Year**, EpisodeNumber, AbsoluteEpisodeNumber, AiredAfter/Before*, Title, **ReleaseDate**, Overview, Image (a single screenshot), Duration, Type, **Credits** (`ActorResource {Name, Character, Gender, ForeignId, Images}`).
- **ID space.** Every ID is a TPDB integer ID; the proxy fronts ThePornDB. Evidence:
  - `Actor.TpdbId`
  - the `{TpdbId}` / `{TpdbSceneId}` naming tokens
  - `ImportListItemInfo.TpdbSiteId/TpdbEpisodeId`
  - Stash identify using `https://theporndb.net/graphql` (§4.3)
- **StashDB is not used anywhere in v2.** Only v3/Eros uses it.
- No refresh deltas: the whole site and all its scenes are fetched on every `RefreshSeriesCommand` (12h).
- **Refresh cost grows with site size.** Large sites have thousands of scenes, and every one is refetched and diffed each time.

### 3.2 Parsing (deep dive, `Core/Parser/Parser.cs`)
- **`ReportTitleRegex` (lines 104-135) contains only date-based patterns.** Sonarr's SxxEyy, anime and absolute patterns are all gone. In order:
  1. `[Site] DD-MM-YYYY …` (day-first, full year).
  2. `[Site] <tokens> - YYYY-MM-DD` (date after title/performer; the text between is captured as `releasetoken`).
  3. `Site - Performers - Title (MonthName D, YYYY)` (long month name).
  4. `Title YYYYMMDD` (non-separated).
  5. `Title YY.MM.DD` or `YYYY.MM.DD` (any of `-_. ` separators; the common scene format `Site.YY.MM.DD.Performer.Title.XXX.1080p…`).
  6. `YYYY-MM-DD …` at the start (strict 1960+).
  7. `Title Dth-mon-YYYY` (short month).
- **Pre-processing:**
  - `SixDigitAirDateRegex` rewrites `YYMMDD` to `20YY.MM.DD`.
  - Website prefix/suffix stripping and the reversed-title fix are kept.
  - Hashed-release rejection is kept, including the `b00bs`/`abc` entries.
  - Chinese/anime pre-substitutions are still present; this is dead weight.
- **Date validation (`ParseMatchCollection`, lines 743-900):**
  - 2-digit years go through `ToFourDigitYear`.
  - The month is swapped with the day if it is greater than 12.
  - Dates more than a day in the future are rejected, as are dates before 1970 unless there is a title year.
  - On success it sets `ParsedEpisodeInfo.AirDate` and `ReleaseTokens`: the remainder after the date, which holds the performer names and scene title.
  - If no year is matched, the path is dead code left from season/episode parsing and yields no episode.
- **`ParsedEpisodeInfo`** (`Parser/Model/ParsedEpisodeInfo.cs`): SeriesTitle ("Site Title"), AirDate ("Release Date"), ReleaseTokens, **ExternalId**, Quality, Languages, ReleaseGroup, ReleaseHash. `IsDaily` = AirDate present. There are no season/episode fields.
- **External-ID parsing** (`ExternalIdPatterns`, lines 43-101, plus `Parser/ExternalIdPattern.cs`):
  - Base pattern: `[A-Z]{2,10}[-_](?=[A-Z0-9]*\d)[A-Z0-9]{2,10}` (e.g. `ABC-123`).
  - Ten variants: bracketed, `[01/10] - "X.mp4"`, bare, followed by resolution, followed by a title, part-numbered, `site.com@ID`, dotted `ABC.123` → `ABC-123` (3+ digits to avoid dates), letter suffix (`-C`), and bracket suffix (`_[4K]`).
  - Used in:
    - `DownloadDecisionMaker.cs:75-94`: if `ParseTitle` fails but an ID is present, it synthesises a `ParsedEpisodeInfo`.
    - `AggregateEpisodes` (import).
    - `CompletedDownloadService`.
    - `DownloadedEpisodesImportService`.
- **Matching a release to a scene (`ParsingService.cs` + `Tv/EpisodeService.cs`):**
  1. **Site lookup:** clean title, then all-titles, then title+year, then **`FindByTitleSlug`**, then the TPDB id from the indexer, then the site owning the ExternalId episode (`ParsingService.cs:348-360`). There are no scene mappings or alternate titles.
  2. **Scene lookup:** `GetDailyEpisode(series, airDate, releaseTokens)`:
     - One scene on that date: take it.
     - Several: `FindOneByAirDate` calls `MatchEpisodes(normalized releaseTokens, episodes)` (`EpisodeService.cs`), a **graded matcher** that tries each level in turn:
       - exact title;
       - title ignoring spaces;
       - title contained in the tokens;
       - punctuation-normalised containment;
       - a performer name equal to the tokens;
       - performer *and* title both contained;
       - all female performers contained.
     - If more than one scene matches, it recurses with a stricter `matchLevel` (up to 2). Anything ambiguous is **rejected** rather than guessed.
     - During searches, the search-criteria episode is still validated against the date/title matcher when several scenes share the date (`ParsingService.GetDailyEpisode`).
  3. **Fallback:** `FindByExternalId` (`EpisodeRepository.cs:62-76`). The ID is lower-cased. If more than one episode has the ID, it returns null and the file needs manual import.
- `ParseSpecialEpisodeTitle` still looks for "season 0" scenes by title (`ParsingService.cs:225-277`). This is a leftover from Sonarr specials.

### 3.3 Search criteria (`Core/IndexerSearch/`)
- **`SingleEpisodeSearchCriteria`**: `EpisodeTitle`, `ReleaseDate` (DateOnly), `Performer` (the first actor), `ExternalId`. It is populated in `ReleaseSearchService.cs:84-116`.
- **`SeasonSearchCriteria`**: `Year` (the season). A multi-scene "season" search is a site+year query.
- **`SceneTitles`** = `[series.Title, series.TitleSlug]` (`ReleaseSearchService.cs:125`). Scene mappings are not used.
- **Newznab/Torznab** (`Indexers/Newznab/NewznabRequestGenerator.cs:78-195`, Torznab inherits it):
  - Always sends `t=search&q=<site>+<date>`. The date format is chosen by the per-indexer `DateSearchFormat`: `YearMonthDay` (`yy.MM.dd`), `DayMonthYear` (`dd.MM.yy`) or `Both`.
  - Optional query variants, each enabled by a per-indexer checkbox:
    - **Search Site + Title**: `q=<site>+<episode title>`.
    - **Search Title Only**: `q=<episode title>`.
    - **Search External ID Only**: `q=<external id>`.
    - **Network or Site** (`SeriesNameSource` Site/Network/Both): adds `q=<network>+<title>`.
  - A season (year) search sends `q=<site>+<yy>` and `q=<site>+<yyyy>`.
  - `tvsearch` and TVDB/IMDb/TVMaze id parameters are not used; everything goes through `t=search`.
- **Default categories are Newznab `6000,6010,6020,6030,6040,6045,6050,6070,6080,6090`** (the XXX range) (`NewznabSettings.cs:72`). Prowlarr's Whisparr app sync defaults to the same set (§9).
- **HDBits / FileList:** query `<title> <yy.MM.dd>` (`HDBits/HDBitsRequestGenerator.cs`, `FileList/FileListRequestGenerator.cs`). FileList defaults to its `XXX = 7` category. IPTorrents and TorrentRss are RSS-only.

### 3.4 Naming tokens (`Core/Organizer/FileNameBuilder.cs:500-710`)
- **Site tokens:** `{Site Title}`, `{Site CleanTitle}`, `{Site TitleYear}` and the other Sonarr title variants renamed from `Series` to `Site` (…`TitleThe`, `TitleWithoutYear`, `TitleSlug`, `TitleFirstCharacter`), plus `{Site Year}` and `{Site Network}`.
- **Scene tokens:**
  - `{Episode Year}` (the season).
  - `{Release Date}` / `{Release ShortDate}`: the separator follows the token separator. The value is `"Unknown"` if there is no date.
  - `{Episode Title}` / `{Episode CleanTitle}`.
  - **`{Episode Performers}`, `{Episode PerformersFemale}`, `{Episode PerformersMale}`, `{Episode PerformersOther}`**: at most 4 names each, space-joined.
  - **`{TpdbSceneId}`** (Episode.TvdbId), **`{External Id}`** and **`{TpdbId}`** (site id).
- **Shared tokens:** Quality*, MediaInfo*, Release Group, Original Title/Filename, Custom Formats.
- **Defaults** (`NamingConfig.cs`):
  - `StandardEpisodeFormat = "{Site Title} - {Release-Date} - {Episode Title} [{Quality Full}]"`
  - **`JavEpisodeFormat = "{External Id}"`**, used when `SeriesType == Jav` (`FileNameBuilder.cs:206`)
  - `SeriesFolderFormat = "{Site Title}"`
  - There is no season folder, and `RenameEpisodes` is false by default.
- **Validation:** `ValidJavEpisodeFormatValidator` requires `{External Id}` or equivalent tokens (`Organizer/FileNameValidation.cs:21,88-104`). The sample service has a JAV sample (`FileNameSampleService.cs:136-181`).

---

## 4. Integrations (full lists)

### 4.1 Indexers (`Core/Indexers/`) — 7
Newznab, Torznab, TorrentRss, IPTorrents, HDBits, FileList, **Fanzub** (obsolete: a dead anime usenet indexer, irrelevant here).
- Removed from Sonarr: BroadcastheNet, Nyaa, Torrentleech.
- **HDBits** does not carry adult content in practice, so it is a candidate to drop.

### 4.2 Download clients (`Core/Download/Clients/`) — identical to Sonarr (18 folders)
Aria2, Blackhole (torrent/usenet), Deluge, DownloadStation (torrent/usenet), Flood, FreeboxDownload, **Hadouken** (obsolete), NzbGet, **NzbVortex** (obsolete), **Pneumatic** (obsolete), QBittorrent, RQBit, rTorrent, Sabnzbd, Transmission, Tribler, uTorrent, **Vuze** (obsolete).

### 4.3 Notifications (`Core/Notifications/`) — 25
- Apprise, CustomScript, Discord, Email, Gotify, **Join** (obsolete), Mailgun, MediaBrowser (Emby/Jellyfin), Notifiarr, Ntfy, Plex (server), **Prowl** (obsolete), PushBullet, Pushcut, Pushover, SendGrid, Signal, Simplepush, Slack, **Stash** (new), Synology Indexer, Telegram, **Twitter** (obsolete), Webhook, Xbmc/Kodi.
- Trakt is removed.
- **Stash** (`Notifications/Stash/`):
  - Posts GraphQL to `http(s)://<host>:<port 9999>/graphql`, with an optional `ApiKey` header.
  - Triggers: OnDownload, OnRename, OnEpisodeFileDelete and OnSeriesDelete run `metadataScan(paths:[<site path>])`. Settings control covers, previews, image previews, sprites and phashes.
  - If `MetadataIdentify` is set, it also runs `metadataIdentify` with sources `stash_box_endpoint: "https://theporndb.net/graphql"` and `builtin_autotag`. Field strategies are MERGE plus createMissing for studio, performers and tags, with options `includeMalePerformers` and `setOrganized`.
  - Supports `MapFrom`/`MapTo` path mapping.
  - `CheckForError` is a TODO: GraphQL errors are ignored (`StashProxy.cs:125-130`).
  - The test calls `systemStatus`.

### 4.4 Import lists (`Core/ImportLists/`) — 6 plus exclusions
- **Custom** (a JSON list of `{tvdbId}` = TPDB site ids).
- **Plex** (watchlist, via `Plex/PlexParser.cs:41`) and **Plex RSS** (`Rss/Plex`). These map Plex's TVDB id into `TpdbSiteId`, which is **semantically wrong**: a TVDB id is not a TPDB id. This is a candidate to drop.
- **RSS** (generic).
- **Whisparr** (another Whisparr v2 instance via `/api/v3/series|qualityprofile|rootfolder|tag`). Bug: in `WhisparrImport.cs:40-55` the profile filter adds items whose profile is *not* selected, and can add them twice.
- **TPDb Performer** (`ThePornDb/`):
  - `ListType.Advanced`, `MinRefreshInterval` 6h.
  - Takes `PerformerId` (GUID or int) and a hidden `ExcludedSiteIds`.
  - It is proxied through `api.whisparr.com/v3/performer/{id}/scenes`; it does not call TPDB directly.
  - Returns (site, scene) pairs. With `ShouldMonitor=SpecificEpisode`, the sync adds the site and monitors only that scene: `AddOptions.EpisodesToMonitor` (`ImportListSyncService.cs:124-175`, `Tv/MonitoringOptions.cs`).
  - `RequestAction("previewPerformer")` groups the scenes by site for the "Select Performer Sites" UI (`frontend/src/Settings/ImportLists/ImportLists/SelectPerformerSites*`).
- Removed from Sonarr: Trakt (list/popular/user), Simkl, AniList, MyAnimeList, Sonarr, `ImportListItems` persistence.

### 4.5 Metadata consumers (`Core/Extras/Metadata/Consumers/`) — Kodi/XBMC, Plex, Roksbox, WDTV (same as Sonarr)
The **Xbmc NFO is customised** (`XbmcMetadata.cs:240-300`):
- Adds `<studio>` (= Network) and `<actor>` elements "for Stash compatibility".
- Adds uniqueids `whisparr` (the local id) and `tpdb`.
- **Two ID bugs:**
  - The TPDB scene/site id is written as `uniqueid type="tvdb"`.
  - The ExternalId (a product code) is written as `type="tpdb"`.

### 4.6 Metadata sources
Only the proxy `api.whisparr.com/v3` (§3.1). There is no StashDB, no direct TPDB, and no TMDb in v2.

---

## 5. External APIs / interfaces
- **REST API v3 only** (`src/Whisparr.Api.V3`, 68 controllers). Controller names and routes are **identical to Sonarr v3**: `/api/v3/series`, `series/lookup`, `series/import`, `series/editor`, `episode`, `episodefile`, `calendar`, `queue`, `release`, `release/push`, `wanted/missing|cutoff`, `seasonpass`, `languageprofile` (stub), `rename`, etc.
  - **Schema differences:**
    - `EpisodeResource` has `ReleaseDate` (DateOnly) and `Actors`, but no `EpisodeNumber`/`AirDate`. It also does not expose `ExternalId` (`Episodes/EpisodeResource.cs`).
    - `SeriesResource.SeriesType` is `standard|jav`. There are no alternate titles and no `SeriesFolderController` (`/series/{id}/folder`).
  - **Upstream's API v5 is not ported**; Whisparr keeps its UI on v3.
  - `ImportListConfigController` and bulk import-exclusion resources are absent.
  - Additions: `FileSystemMediaFileResource`/`FileSystemTypeResource`, `BackupRestoreResource`, and `RestartResource`/`ShutdownResource`.
- **OpenAPI:** Swagger is served at **`/docs`**, with `docs/{documentName}/openapi.json` published anonymously ahead of auth (`src/NzbDrone.Host/Startup.cs:259-268`, #1301). No `openapi.json` is checked in.
- **Calendar iCal feed:** `ProductId "-//whisparr.com//Whisparr//EN"` (`Calendar/CalendarFeedController.cs:48`).
- **SignalR** at `/signalr/messages` (same as Sonarr).
- **Webhooks:** the payload shape is Sonarr's (`series`, `episodes[]`, `episodeFile`, `release`, …).
  - `WebhookSeries.TvdbId` = TPDB site id, and `Type` = SeriesTypes.
  - `WebhookEpisode.EpisodeNumber` = `AbsoluteEpisodeNumber ?? 0`, and `SeasonNumber` = year (`Notifications/Webhook/WebhookEpisode.cs:12-24`).
  - There are no performer or external-id fields in webhooks.
- **Custom script env vars:** the prefix is **`Whisparr_`**, with the same key set as Sonarr's `Sonarr_*` minus a few:
  - `Whisparr_EventType`, `Whisparr_InstanceName`, `Whisparr_ApplicationUrl`
  - `Whisparr_Series_{Id,Title,TitleSlug,Path,TvdbId,Type,Year,Genres,Tags,DeletedFiles}`
  - `Whisparr_Release_{Title,Indexer,Size,Quality,QualityVersion,ReleaseGroup,SeasonNumber,EpisodeCount,EpisodeAirDates(Utc),EpisodeTitles,EpisodeOverviews,AbsoluteEpisodeNumbers,IndexerFlags,CustomFormat(Score)}`
  - `Whisparr_EpisodeFile_*` (Id(s), Path(s), RelativePath(s), PreviousPaths, SeasonNumber, EpisodeIds/AirDates/Titles/Overviews/Count, Quality, ReleaseGroup, SceneName, SourcePath/Folder, MediaInfo_*, CustomFormat, DeleteReason)
  - `Whisparr_Download_{Client,Client_Type,Id,Size,Title}`, `Whisparr_IsUpgrade`, `Whisparr_Deleted*`
  - `Whisparr_Health_{Issue,Restored}_*`, `Whisparr_Update_*`
  - Source: `Notifications/CustomScript/CustomScript.cs`. There are no performer or external-id variables.

---

## 6. Background jobs
`Core/Jobs/TaskManager.cs`:

| Command | Interval |
|---|---|
| RefreshMonitoredDownloads | 1 min |
| MessagingCleanup | 5 min |
| ImportListSync | 5 min (per-list `MinRefreshInterval`: TPDb Performer 6h, Whisparr 5 min) |
| ApplicationUpdateCheck | 6 h (GitHub Releases) |
| CheckHealth | 6 h |
| RefreshSeries (full site + all scenes refetch) | 12 h |
| Housekeeping | 24 h |
| CleanUpRecycleBin | 24 h |
| Backup | configurable, 1–7 days (default 7) |
| RssSync | configurable, default 15 min |

- Differences from Sonarr: no `UpdateSceneMappingCommand` (3h), and nothing else.
- On-demand commands: `EpisodeSearch`, `SeasonSearch` (year), `SeriesSearch`, `MissingEpisodeSearch`, `CutoffUnmetEpisodeSearch`, rename, rescan, manual import, etc. (the same Sonarr set).
- **Event-driven search** is handled by `Tv/EpisodeAddedService.cs`. Scenes newly added with a release date within [−14 days, +1 day] are searched after refresh.

---

## 7. Configuration surface (specific parts only)
- **config.xml** (`ConfigFileProvider.cs`):
  - `WhisparrMetadata` (metadata proxy URL; default `https://api.whisparr.com/v3/{route}`).
  - `GithubOwnerRepo` (default `whisparr/whisparr`), which is the update source.
  - Removed compared with Sonarr: `LogDbEnabled`, `ProfilerEnabled`, `ProfilerPosition`, `ConsoleLogFormat`.
- **DB config** (`ConfigService.cs`): removed `SeasonPackUpgrade`, `SeasonPackUpgradeThreshold`, `ListSyncLevel`, `ListSyncTag`, `TimeZone`. Nothing adult-specific was added.
- **Per-indexer (Newznab/Torznab)** (`NewznabSettings.cs:92-114`):
  - `SearchTitleOnly`, `SearchSiteTitleOnly`, `SearchExternalIdOnly`
  - `DateSearchFormat` (YMD/DMY/Both)
  - `SeriesNameSource` (Site/Network/Both)
  - `FailDownloads`
  - default XXX categories
- **Naming:** `JavEpisodeFormat`, with Site/Release/Performer tokens.
- **Import list:** `ShouldMonitor` (None/SpecificEpisode/EntireSite), `SiteMonitorType`.
- **Release profile:** `AirDateRestriction` + `AirDateGracePeriod` (upstream feature), single `IndexerId`.
- **Stash notification:** Host, Port (9999), UseSsl, ApiKey, MapFrom/MapTo, Generate{Covers,Previews,ImagePreviews,Sprites,Phashes}, MetadataIdentify, IncludeMalePerformers, SetOrganized.
- **Lookup syntax:** `tpdb:<id>` / `tpdbid:<id>`.

---

## 8. Platform/infrastructure concerns (divergences only)
- **Cloud endpoints** (`src/NzbDrone.Common/Cloud/WhisparrCloudRequestBuilder.cs`):
  - `https://whisparr.servarr.com/v1/` for ping, time and server-side notifications (`HealthCheck/Checks/SystemTimeCheck.cs`, `ProxyCheck.cs`, `HealthCheck/ServerSideNotificationService.cs:48`).
  - `https://api.whisparr.com/v3/` for metadata.
  - `https://api.github.com/repos/{owner/repo}/releases` for updates.
  - Sentry: `sentry.servarr.com` projects 38/40 (`NzbDrone.Common/Instrumentation/NzbDroneLogger.cs:72-78`).
- **Updates come from GitHub Releases** (`Core/Update/GithubUpdatePackageProvider.cs`, using Semver), not from a Servarr update service. Branches are `v2` and `v2-develop`; release tags look like `v2.2.0-develop.404`.
- **CI:** GitHub Actions `build.yml` builds freebsd-x64, linux-arm and the other runtimes. `azure-pipelines.yml` is dead: it still references .NET 6 and old branches.
- `distribution/debian/sonarr.service` still carries the Sonarr name, as a leftover.
- `SkyHookProxy` writes to `Console.WriteLine`, which is debug output.
- Everything else (DryIoc, Mono-named projects, RuntimePatches, SQLite/Postgres, NLog) is identical to Sonarr.

---

## 9. Interactions with other *arr apps
- **Prowlarr → Whisparr indexer sync.** `submodules/prowlarr/src/NzbDrone.Core/Applications/Whisparr/`:
  - Uses `AppApiRoute = "/api/v3"` (`WhisparrV3Proxy.cs:27`) and pushes Newznab/Torznab indexers pointing at `{ProwlarrUrl}/{id}/api`.
  - Default `SyncCategories = 6000…6090` (`WhisparrSettings.cs:28`).
  - It keeps user fields not managed by Prowlarr, so Whisparr-specific toggles such as `searchExternalIdOnly` and `dateSearchFormat` survive a sync (`Whisparr.cs:183-184`).
  - The same Prowlarr app type serves v2 and v3/Eros, because both expose `/api/v3/indexer`.
- **Whisparr → Whisparr** via the "Whisparr" import list (§4.4).
- **Stash** (a separate media server/organizer): a push-only GraphQL scan/identify (§4.3), plus NFO output shaped for Stash's NFO scraper.
- There is no Overseerr/Jellyseerr integration (they do not support Whisparr), and no Bazarr.
- **Hosted services:**
  - `api.whisparr.com` (metadata proxy, closed source).
  - `whisparr.servarr.com` (services).
  - `theporndb.net/graphql`: used only indirectly, through Stash's identify.
  - GitHub Releases.

---

## 10. Outdated / legacy candidates
- **Indexers:** Fanzub (dead). HDBits is irrelevant for this content.
- **Download clients:** Hadouken, Vuze, Pneumatic, NzbVortex (same as Sonarr).
- **Notifications:** Twitter, Join, Prowl (same as Sonarr).
- **Import lists:** Plex watchlist and Plex RSS (they map TVDB ids to TPDB ids, which is wrong for this domain).
- **Parser:** the dead TV leftovers are:
  - Chinese/anime `PreSubstitutionRegex` (`Parser.cs:27-45`)
  - the `AnimeReleaseGroupRegex` / sub-group handling
  - the season/episode branch in `ParseMatchCollection`
  - "season 0 specials" matching (`ParsingService.ParseSpecialEpisodeTitle`)
  - `SeasonFolderRegexes`
- **Model:** `Series.UseSceneNumbering`, `AbsoluteEpisodeNumber` (only feeds `WebhookEpisode.EpisodeNumber`), `Season`-as-year, `TvdbId` naming throughout, the `languageprofile` API stub, and `Languages/LanguagesBelowCutoff.cs`.
- **Build:** `azure-pipelines.yml` (.NET 6), `distribution/debian/sonarr.service`, the `Console.WriteLine` in `SkyHookProxy`.
- **Frontend:** the Redux/webpack/react-router 5 stack and 161 remaining .js files. Upstream has already moved on.
- **Bugs spotted:**
  - Xbmc NFO id type labels (§4.5).
  - `WhisparrImport` profile filter (§4.4).
  - Stash `CheckForError` TODO.
  - `EpisodeResource` not exposing `ExternalId`.
  - Duplicated log line in `EpisodeAddedService.SearchForRecentlyAdded`.

---

## 11. Rewrite notes

### 11.1 What is genuinely unique and what is configuration
Almost all of Whisparr v2 can be expressed as **a media type on the unified TV pipeline**: a "series-like container with date-keyed items and a different metadata provider". What is genuinely unique:

1. **Metadata provider adapter:** Site/Scene/Performer from TPDB. It can go through `api.whisparr.com/v3` or, better, call TPDB and/or StashDB directly. A direct TPDB API needs an API key; StashDB needs an API key and GraphQL.
   - Shape: container (site: title, slug, network/parent studio, types incl. JAV, images) → items (scene: id, date, title, product code, duration, performers with gender and images).
   - Year-grouping is only a presentation choice.
2. **Date-first release parser.** This is Sonarr's daily-series parser with extra formats:
   - `[Site]` prefixes, DD-MM-YYYY, month names and YYMMDD.
   - Day/month swap, future-date rejection.
   - Keeping the post-date remainder as `releaseTokens`.
   - The **same-date disambiguation matcher** using title and performer names, with graded strictness and female-performer sets (`EpisodeService.MatchEpisodes`, `FindOneByAirDate`). This is the main new algorithm, and its "reject if ambiguous" policy is important to keep.
3. **Product-code (ExternalId) identity:** the regex family in `Parser.cs:43-101`, a unique index on it, and use as a match key in decisions, completed-download tracking and import. This generalises well as "alternate item identifiers with a parser plugin".
4. **Search query builder:** free-text `t=search` queries (`site+yy.MM.dd`, `site+dd.MM.yy`, `site+title`, `title`, `code`, `network+title`), with per-indexer toggles and XXX categories. In a unified design this is a per-media-type query template list rather than special code.
5. **Performer-driven acquisition:** a performer is a *list source* producing (container, item) pairs with **item-level monitoring** (`EpisodesToMonitor`). A unified model with item-level monitoring from list sources covers this; a first-class performer entity (as in v3) is optional.
6. **Naming tokens:** Release Date, Performers (all/female/male/other, max 4), External Id, TPDB ids, and a per-container-type template (JAV).
7. **Stash integration:** a media-server-style notifier (GraphQL scan + identify) and Stash-flavoured NFO. It is small and fits the existing "media server refresh" plug-in slot.
8. **VR quality:** one extra quality/source value.

**Expressible as pure configuration of the unified pipeline:**
- Quality and custom formats, delay/release profiles (including the air-date restriction).
- Download clients, the remaining notifications, blocklist/queue/history, backups, health, auth.
- Standard import and rename.

**v2 against v3 for the monolith.** v3/Eros (Radarr-based, scene-as-movie, first-class Studio/Performer, StashDB + TPDB + TMDb) represents the upstream project's direction:
- If the monolith supports only one adult model, the **v3 model is the better target**: an item with an optional container and many performers.
- The v2 behaviours to port are the date parser, the same-date matcher, the external-ID handling and the query templates, which v3 presumably has too (not analysed).
- Both versions share the Prowlarr contract (`/api/v3/indexer`).

### 11.2 Divergence from upstream (Sonarr v5-develop @ `a8a82905`)
- **Tracking:** cherry-picks from `Sonarr/Sonarr` `v5-develop`. Last sync point is `900cc21d3` (2026-08-29); 54 commits behind; 24 conflicted upstream commits pending as issues. Radarr code is never pulled.
- **Deliberately removed:**
  - DataAugmentation (scene mappings, XEM, daily-series list) and alternate titles.
  - Trakt (import and notify), Simkl, AniList, MAL, the Sonarr import list.
  - BTN, Nyaa, Torrentleech.
  - Anime: `AnimeVersionUpgradeSpecification`, absolute-number import spec, anime parsing.
  - Season packs and multi-episode: `FullSeason`, `MultiSeason`, `SeasonPackOnly`, `SplitEpisode`, `SeasonMatch`, `SeasonPackSeedGoal`.
  - `SceneMappingSpecification`, `UnverifiedSceneNumbering`, season folders.
  - The TV-specific AutoTagging specs (SeriesType, OriginalCountry, Tag, Year).
- **Upstream features not (yet) present:**
  - API v5 and the Vite/zustand/react-router-7 frontend.
  - `QualityProfileQualityRanks` (mig 233), `ReleaseType` (CF spec, parser), `ReleaseHash` aggregation.
  - Multiple mediainfo audio/subtitle streams (mig 225), release-profile `IndexerIds` (mig 224), `OriginalCountry`, tag-existing import lists.
  - `Statistics/` (library statistics), `SeriesEditedService`/`EpisodeRefreshedService` renames, `ObsoleteQueueService`, and config options `LogDbEnabled`/`Profiler*`/`ConsoleLogFormat`/`TimeZone`.
- **Adult additions:** everything in §11.1, plus the `VR` quality, `Episode.ExternalId` and `Actors`, `SeriesTypes.Jav`, `NamingConfig.JavEpisodeFormat`, the GitHub-based updater, and the `Whisparr_*` env prefix.
- **Schema:** the squashed baseline and the renumbered 216–220 migrations make the DB incompatible with Sonarr's migration history (§2.3).
- **For the Go monolith:** porting Sonarr's pipeline once and adding (1) a TPDB/StashDB provider, (2) the date/performer/product-code matcher, (3) query templates and (4) item-level monitoring from performer lists would reproduce Whisparr v2 with no other forked code. That covers about 12.9k diff lines, of which the adult logic is roughly 3–4k. The rest is lag and removals.
