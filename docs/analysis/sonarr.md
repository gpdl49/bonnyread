# Sonarr — Codebase Analysis

Source: `submodules/sonarr` (commit `a8a82905`, 2026-09-25). All paths below are relative to `submodules/sonarr/` unless noted; `Core/` = `src/NzbDrone.Core/`.

---

## 1. Overview

**Purpose:** a PVR for TV series. It tracks series/seasons/episodes (metadata from Sonarr's own SkyHook proxy over TheTVDB), watches indexer RSS feeds and runs searches, decides which releases to grab (quality, custom formats, profiles), sends them to usenet/torrent download clients, follows download progress, imports and renames finished files into the library, and notifies media servers and other services.

**Tech stack**
- Backend: C# on **.NET 10** (`global.json` SDK 10.0.401; `<TargetFrameworks>net10.0</TargetFrameworks>` in most csproj, `net10.0-windows` for the tray app `src/NzbDrone`). ASP.NET Core (controllers + SignalR). DryIoc for dependency injection. Mono is no longer a runtime target, but the Mono-named projects are still there (see §8).
- DB: **SQLite** by default (System.Data.SQLite 2.0.4 + SourceGear.sqlite3 3.53.4) or **PostgreSQL** (Npgsql 10.0.3). Schema is managed by FluentMigrator 8.0.1. Data access is Dapper 2.1.89 plus a small in-house ORM (`Core/Datastore/BasicRepository.cs`, `SqlBuilder.cs`, `WhereBuilder{Sqlite,Postgres}.cs`, `TableMapping.cs`). There are two databases: main (`sonarr.db`) and logs (`logs.db`).
- Notable libraries: Newtonsoft.Json 13 and System.Text.Json (both used), FluentValidation 9.5.4 (old major), NLog 5.5 (+ Syslog, CLEF JSON layout), Sentry 5.16 (error reporting to `sentry.sonarr.tv`), Polly 8.8, MailKit 4.18 (email), Ical.Net 5.2 (iCal feed), MonoTorrent 3.0.2 (only for reading .torrent files), Openur.FFMpegCore/FFprobeStatic (media info through ffprobe), SixLabors.ImageSharp (poster resizing), SharpZipLib (backup/update archives), Lib.Harmony 2.4 (runtime patches), MiniProfiler, Swashbuckle (OpenAPI; `src/Sonarr.Api.V3/openapi.json` and `src/Sonarr.Api.V5/openapi.json` are checked in), Microsoft.AspNetCore.Authentication.OpenIdConnect (OIDC login), RestSharp 106 (test/automation only), Diacritical.Net, IPAddressRange.
- Frontend: React 18.3, TypeScript 5.7 (893 .ts/.tsx files against 17 .js), Vite 8, @tanstack/react-query 5, zustand 5, react-router 7, @microsoft/signalr. jQuery 3.7, lodash and moment are still dependencies. Redux has been removed. `frontend/src/Utilities/Fetch/fetchJson.ts` sets `apiRoot = '/api/v5'`.

**Size (LOC of .cs files / file count)**

| Project | LOC | Files |
|---|---:|---:|
| NzbDrone.Core (domain) | 111,669 | 1,657 |
| Sonarr.Api.V3 | 10,733 | 156 |
| Sonarr.Api.V5 | 10,535 | 178 |
| Sonarr.Http (shared web plumbing) | 4,241 | 76 |
| NzbDrone.Common | 14,210 | 182 |
| NzbDrone.Host | 1,638 | 14 |
| NzbDrone.Update (updater exe) | 657 | 10 |
| NzbDrone.Mono (Unix disk/OS helpers) | 1,303 | 13 |
| NzbDrone.Windows | 296 | 3 |
| Sonarr.RuntimePatches | 524 | 6 |
| NzbDrone.SignalR | 96 | 3 |
| NzbDrone.Console / NzbDrone (tray) | 134 / 233 | 1 / 3 |
| NzbDrone.Core.Test | 65,130 | 451 |
| frontend/src (ts/tsx/js) | ~85,300 | ~910 |

Larger Core areas by file count: Datastore 283 (mostly migrations), Download 255, Notifications 217, MediaFiles 116, ImportLists 92, Indexers 88, DecisionEngine 54, Tv 43, HealthCheck 40.

Other repo content: `distribution/` (Debian `install.sh` + systemd `sonarr.service`, Dockerfile, macOS `Sonarr.app`, Windows Inno setup), `schemas/torznab.xsd`, `src/ServiceHelpers/{ServiceInstall,ServiceUninstall}` (Windows service helper executables).

---

## 2. Domain model

Tables are registered in `Core/Datastore/TableMapping.cs`. Main entities:

- **Series** (`Core/Tv/Series.cs`, table `Series`): TvdbId (the primary external key), TvRageId (dead), TvMazeId, ImdbId, TmdbId, MalIds, AniListIds, Title/CleanTitle/SortTitle/TitleSlug, Status (`SeriesStatusType`: Deleted=-1, Continuing, Ended, Upcoming), Overview, AirTime, Monitored, MonitorNewItems (`NewItemMonitorTypes` All/None), QualityProfileId, SeasonFolder, SeriesType (`SeriesTypes`: Standard, Daily, Anime), UseSceneNumbering, Network, Runtime, Path, RootFolderPath, Year, Ratings, Genres, Actors, Certification, FirstAired/LastAired, OriginalLanguage, OriginalCountry, **Seasons** (embedded JSON list of `Season {SeasonNumber, Monitored}`), Tags, AddOptions (embedded). There is no language profile any more; it was removed in v4, and v3 serves a stub for it (see §5).
- **Episode** (`Core/Tv/Episode.cs`, `Episodes`): SeriesId, TvdbId, EpisodeFileId (0 means no file), SeasonNumber, EpisodeNumber, **AbsoluteEpisodeNumber**, **SceneAbsoluteEpisodeNumber / SceneSeasonNumber / SceneEpisodeNumber** (from XEM), UnverifiedSceneNumbering, AiredAfterSeasonNumber / AiredBeforeSeasonNumber / AiredBeforeEpisodeNumber (placement of specials), AirDate (string) and AirDateUtc, Monitored, Runtime, FinaleType, LastSearchTime, Ratings.
- **EpisodeFile** (`Core/MediaFiles/EpisodeFile.cs`, `EpisodeFiles`): SeriesId, SeasonNumber, RelativePath, Size, DateAdded, OriginalFilePath, SceneName, ReleaseGroup, ReleaseHash, Quality (`QualityModel` JSON: quality + revision), Languages, IndexerFlags, ReleaseType, MediaInfo (embedded JSON from ffprobe). One file can map to many episodes (multi-episode files).
- **QualityProfile** (`Core/Profiles/Qualities/QualityProfile.cs`): Name, UpgradeAllowed, Cutoff, Items (a nested tree of qualities and groups, each Allowed, **with per-profile MinSize/MaxSize/PreferredSize**, see `QualityProfileSizeLimit.cs`), FormatItems (CF id → score), MinFormatScore, CutoffFormatScore, MinUpgradeFormatScore. Newer: **QualityProfileQualityRanks** table (ProfileId, QualityId, Score) from migration 233 (`Core/Profiles/Qualities/QualityProfileQualityRank.cs`).
- **QualityDefinition** (`Core/Qualities/QualityDefinition.cs`): Quality, Title, GroupName, Weight, MinSize/MaxSize/PreferredSize (MB per minute). The quality list in `Core/Qualities/Quality.cs` is fixed: Unknown, SDTV, DVD, WEBDL480p, HDTV720p, WEBDL720p, Bluray720p, HDTV1080p, WEBDL1080p, Bluray1080p, Bluray1080pRemux, RAWHD, HDTV2160p, WEBDL2160p, Bluray2160p, Bluray2160pRemux (plus WEBRip variants). `Revision` holds version, real and isRepack.
- **CustomFormat** (`Core/CustomFormats/CustomFormat.cs`): Name, IncludeCustomFormatWhenRenaming, Specifications (polymorphic JSON). Spec types: ReleaseTitle, ReleaseGroup, Language, IndexerFlag, Resolution, Source, Size, ReleaseType (`Core/CustomFormats/Specifications/`).
- **ReleaseProfile** (`Core/Profiles/Releases/ReleaseProfile.cs`): Required / Ignored terms (plain text or `/regex/`, via `PerlRegexFactory.cs`), IndexerIds, Tags, ExcludedTags, and newer air-date options (AirDateRestriction, AirDateGracePeriod, AllowSeasonPackWithoutAllEpisodesAired, migration 232).
- **DelayProfile**: per-protocol enable flags, delays, preferred protocol, bypass-if-highest-quality or bypass-if-above-CF-score, Tags, Order.
- **AutoTagging** (`Core/AutoTagging/`): rules that add tags automatically. Specs: Genre, Monitored, Network, OriginalCountry, OriginalLanguage, QualityProfile, RootFolder, SeriesType, Status, Tag, Year.
- **Provider definitions** (the "ThingiProvider" pattern, `Core/ThingiProvider/`): the `Indexers`, `DownloadClients`, `Notifications`, `ImportLists` and `Metadata` tables all store Name, Implementation (class name), ConfigContract (settings class name), Settings (JSON), Tags, plus type-specific flags. Each has a `*Status` table (failure escalation and backoff, `ThingiProvider/Status/EscalationBackOff.cs`).
- Other tables: `Config` (key/value settings), `RootFolders`, `ScheduledTasks`, `SceneMappings`, `History` (EpisodeHistory), `DownloadHistory`, `Blocklist`, `PendingReleases`, `MetadataFiles`, `SubtitleFiles`, `ExtraFiles`, `NamingConfig`, `RemotePathMappings`, `Tags`, `ReleaseProfiles`, `DelayProfiles`, `Users`, `Commands` (the persisted command queue), `CustomFilters`, `UpdateHistory`, `ImportListExclusions`, `ImportListItems`, `Logs` (log DB).

**Migrations:** `Core/Datastore/Migration/` has 226 files, numbered `000_database_engine_version_check` to `233_quality_profile_quality_ranks`. The log DB has its own `LogDbUpgrade` path (6 migrations). The framework is in `Core/Datastore/Migration/Framework/` and includes a custom SQLite processor and schema dumper. Every migration runs on both SQLite and Postgres through `IfDatabase(...)` branches. SQLite uses WAL, except TRUNCATE journaling on macOS (`Core/Datastore/ConnectionStringFactory.cs:64`).

---

## 3. Feature areas

One entry per top-level folder in `Core/`:

- **Analytics**: `AnalyticsService` only exposes `IsEnabled`, which gates Sentry and the update check. There is no separate telemetry.
- **Annotations**: `FieldDefinitionAttribute` / `SelectOption` metadata on provider settings classes. It drives the dynamic settings forms in the UI (`Sonarr.Http/ClientSchema/SchemaBuilder.cs`).
- **Authentication**: users table (a single user), password hashing, the `AuthenticationType` enum (None, Basic [Obsolete], Forms, External, **Oidc**), and `OidcDiscoveryService`.
- **AutoTagging**: rule-based tagging of series (see §2). It is re-evaluated on refresh.
- **Backup**: scheduled (every 1–7 days, configurable) and manual zip of `config.xml` and the database (`MakeDatabaseBackup.cs` uses the SQLite online backup; on Postgres there is no DB dump). Keeps a retention count. Restore works by upload through the API.
- **Blocklisting**: failed or rejected releases (by title, torrent info hash, indexer). They are checked by `BlocklistSpecification` and `ClearBlocklistCommand`.
- **Configuration**: `ConfigFileProvider` reads `config.xml` plus env/options overrides (`Sonarr:Server/Auth/Log/Update/App/Postgres` sections; `src/NzbDrone.Host/Bootstrap.cs:109`). `ConfigService` holds the DB-backed key/value settings (`IConfigService.cs`). `DeploymentInfoProvider` reads the `package_info` file from the packager.
- **CustomFilters**: saved UI filters (JSON), per UI page.
- **CustomFormats**: CF specifications, the `CustomFormatCalculationService` that scores releases and files, and the "custom format" rename token.
- **DataAugmentation**: external data that corrects metadata:
  - `Scene/`: **scene mappings** (alternate and scene titles, per-season scene naming, search terms, `FilterRegex`, `SearchMode`, origin) from `services.sonarr.tv/v1/scenemapping` (`ServicesProvider`), refreshed every 3h by `UpdateSceneMappingCommand`.
  - `Xem/`: **TheXEM** (`https://thexem.info/map/`: `havemap`, `all`, `allNames`) supplies scene season/episode/absolute numbering (`XemService.cs`) and XEM alternate names.
  - `DailySeries/`: the list of TVDB IDs to force to the Daily type, from `services.sonarr.tv/v1/dailyseries`. It is used in `SkyHookProxy.cs:249`.
- **Datastore**: the DB layer, the custom ORM, type converters (`Converters/`: embedded JSON documents, provider settings JSON, Quality and Language int converters, OsPath, UTC), migrations, the DB-restore-on-startup service, and paging.
- **DecisionEngine**: grab decisions (see the deep dive below).
- **DiskSpace**: free space per root folder and download folder, for the UI and the FreeSpace specs.
- **Download**: download client providers, `DownloadService` (sends a release to a client), **Pending** (delay-profile holding pen, `PendingReleaseService`), **TrackedDownloads** (maps client items to series/episodes using `DownloadHistory` and the parser), `CompletedDownloadService`, `FailedDownloadService` + `RedownloadFailedDownloadService`, `IgnoredDownloadService`, `RejectedImportService`, `DownloadProcessingService`, `Aggregation/` (augments a RemoteEpisode), `NzbValidationService`, seed config (`DownloadSeedConfigProvider`).
- **Exceptions**: domain exception types (ReleaseUnavailable, ReleaseBlocked, SearchFailed, …).
- **Extras**: sidecar files. `Metadata/` (NFO and images for Kodi/Plex/Roksbox/WDTV), `Subtitles/` (imports and renames .srt/.ass etc. with language and title tags), `Others/` (other extra extensions from `ExtraFileExtensions`), existing-extra importers.
- **HealthCheck**: 28 checks (listed in the deep dive). Event-driven through `[CheckOn(typeof(Event))]` plus the scheduled `CheckHealthCommand`. `ServerSideNotificationService` fetches server-side warnings from `services.sonarr.tv` (via `PackageGlobalMessageCheck` / update service).
- **History**: `EpisodeHistory`, with event types Grabbed, SeriesFolderImported, DownloadFolderImported, DownloadFailed, EpisodeFileDeleted, EpisodeFileRenamed, DownloadIgnored.
- **Housekeeping**: a daily set of 31 cleanup tasks (listed below).
- **Http**: Core HTTP interceptors: `CloudFlare/CloudFlareHttpInterceptor` (detects CF captcha), `TorcacheHttpInterceptor` (**dead service**), and `HttpProxySettingsProvider`.
- **ImportLists**: list providers, `ImportListSyncService` (adds series and, depending on `ListSyncLevel`, cleans up series no longer listed: Disabled / LogOnly / KeepAndUnmonitor / KeepAndTag), `Exclusions`, `ImportListItems` (caches fetched items).
- **IndexerSearch**: search orchestration (`ReleaseSearchService`) and search criteria types: SingleEpisode, Season, Daily episode, Daily season, Anime episode, Anime season, Special episode, plus scene mapping expansion (`SceneEpisodeMapping`, `SceneSeasonMapping`). Commands: Episode, Season, Series, Missing, CutoffUnmet search.
- **Indexers**: indexer providers, `RssSyncService`, `FetchAndParseRssService`, generic RSS/Torznab/Newznab parsers, caps discovery (`NewznabCapabilitiesProvider`), and indexer status/backoff.
- **Instrumentation**: NLog targets (DB log target, Sentry reconfiguration), and log file commands (ClearLog, DeleteLogFiles, DeleteUpdateLogFiles).
- **Jobs**: `TaskManager` (scheduled task table, see §6) and `Scheduler`.
- **Languages**: the `Language` enum-like class (54 languages) and `LanguagesComparer`.
- **Lifecycle**: application start and shutdown events, plus `RestartCommand` and `ShutdownCommand`.
- **Localization**: UI and backend strings (`Localization/Core/*.json`, 45 locales).
- **MediaCover**: downloads, caches and resizes posters, fanart and banners (ImageSharp). `MediaCoverProxy` proxies remote covers.
- **MediaFiles**: disk scan, import pipeline (`EpisodeImport/`), rename, move and hardlink/copy (`EpisodeFileMovingService`), recycle bin, deletion, media info extraction (`MediaInfo/`, ffprobe), `.torrent` info reading, **script import** (`ScriptImportDecider`: a user script decides the move or copy itself).
- **Messaging**: in-process command bus and event aggregator (`Messaging/Commands/CommandQueueManager.cs`, `CommandExecutor.cs`, `EventAggregator.cs`). Commands are persisted to the `Commands` table and have priorities and exclusivity.
- **MetadataSource**: `SkyHook/SkyHookProxy.cs` fetches series info and search results from `https://skyhook.sonarr.tv/v1/tvdb/{route}/{language}/`. Search supports the prefixes `tvdb:`, `imdb:`, `tmdb:`, `mal:`, `anilist:`.
- **Notifications**: the "Connect" providers (see §4) and `NotificationService`, which fans events out.
- **Organizer**: `FileNameBuilder` (naming token engine), naming validation, and sample generation.
- **Parser**: release and file title parsing (see the deep dive).
- **Profiles**: quality, delay and release profiles.
- **ProgressMessaging**: pushes command progress messages to the UI.
- **Qualities**: quality definitions, `QualityModelComparer`, the proper/repack policy (`ProperDownloadTypes`: PreferAndUpgrade / DoNotUpgrade / DoNotPrefer), and `ResetQualityDefinitionsCommand`.
- **Queue**: `QueueService` (per-download queue, used by v5) and **`ObsoleteQueueService`** (per-episode queue, kept only for v3; `Core/Queue/ObsoleteQueueService.cs`, `[Obsolete]` on `Queue.cs:19`).
- **RemotePathMappings**: maps download-client paths to local paths (host + remote → local).
- **RootFolders**: root folders with free space and unmapped folders (for library import).
- **Security**: TLS certificate validation policy (`CertificateValidationType`: Enabled / DisabledForLocalAddresses / Disabled).
- **SeriesStats**: aggregate episode, file and size counts per series and season (SQL).
- **Statistics**: newer library-wide stats (library, per quality profile, per quality, per tag), exposed only in v5 (`/api/v5/statistics`).
- **Tags**: tags plus `TagDetails` (usage across entities). Tags link series to indexers, download clients, notifications, delay/release profiles, import lists and auto-tagging.
- **ThingiProvider**: the generic provider framework (factory, repository, definition, config contract, status and backoff, provider events).
- **Tv**: series, episodes and seasons, `AddSeriesService`, `RefreshSeriesService` / `RefreshEpisodeService` (sync from SkyHook), `EpisodeMonitoredService` (monitor options: All, Future, Missing, Existing, FirstSeason, LastSeason, LatestSeason [Obsolete], Pilot, Recent, MonitorSpecials, UnmonitorSpecials, None, Skip), `MoveSeriesService`, `SeriesTitleNormalizer`, `ShouldRefreshSeries` (skips refresh for ended series).
- **Update**: update check (`services.sonarr.tv/v1/update/{branch}`), install (downloads the package, extracts it and launches `Sonarr.Update`), `UpdateHistory`, and mechanisms (`UpdateMechanism`: BuiltIn, Script, External, Apt, Docker).
- **Validation**: FluentValidation extensions (paths, URLs, root folder in use, and similar).
- Files at the top of `Core/`: `Fluent.cs` (string helpers) and `Hashing.cs`.

### Deep dives

**Parsing** (`Core/Parser/`)
- `Parser.cs` (1,362 lines, **142 regexes**) is an ordered cascade of `ReportTitleRegex` patterns. The order is load-bearing. It covers:
  - SxxEyy and NxM forms, including multi-episode ranges and repeats, split episodes (`S01E05a`), 3-digit seasons, 4-digit seasons (`S2016E05`), and 3-digit NMM / 4-digit NNMM episode numbering ("103/113 naming", "1103/1113 naming").
  - Daily dates in several formats (`2018.04.28`, `04.28.2018`, `28.04.2018`, `20180428`, short month names, Plex DVR format, dates plus parts).
  - A large family of anime patterns: `[SubGroup] Title Absolute`, absolute plus SxxEyy, 4-digit absolute numbers, batches with tilde, hash suffixes `[ABCD1234]`, French anime titles.
  - Mini-series (`Part01`, `XofY`, `E1-E2`), full, partial and multi-season packs.
  - Locale-specific forms: Spanish tracker, Turkish "Bölüm", Dutch/Flemish, Chinese + English dual titles, Japanese variety shows with leading dates.
  - "Season 01 Episode 03" wording.
- Other parsers: `QualityParser.cs` (source, resolution, modifier, proper/repack/real/version detection, v2 anime versions), `LanguageParser.cs` (release and subtitle language), `ReleaseGroupParser.cs`, `SceneChecker.cs` (is this a scene-style name?), `IsoLanguages.cs`.
- `ParsingService.cs` (729 lines) maps parsed info to Series and Episodes, including scene mapping lookup by title and season, XEM scene numbering, anime absolute to season/episode, daily air date to episode, and specials by title.
- `ValidateParsedEpisodeInfo.cs` holds sanity checks.
- There are 1,631 `[TestCase]` parser tests in `src/NzbDrone.Core.Test/ParserTests`. **This is a golden corpus to port verbatim.**

**Decision engine** (`Core/DecisionEngine/`)
- `DownloadDecisionMaker` runs every `IDownloadDecisionEngineSpecification`, ordered by `SpecificationPriority`.
- General specs: AcceptableSize, AirDate, AlreadyImported, AnimeVersionUpgrade, BlockedIndexer, Blocklist, CustomFormatAllowedByProfile, FreeSpace, FullSeason, MaximumSize, MinimumAge, MultiSeason, NotSample, Protocol, QualityAllowedByProfile, Queue, RawDisk, ReleaseRestrictions, Repack, Retention, SameEpisodes, SameEpisodesGrab, SceneMapping, SeasonPackOnly, SplitEpisode, TorrentSeeding, Upgradable, UpgradeAllowed, UpgradeDisk.
- RSS-only specs (`RssSync/`): Delay, DeletedEpisodeFile, History, IndexerTag, MonitoredEpisode, Pending, Proper.
- Search-only specs (`Search/`): EpisodeRequested, SeasonMatch, Series, SingleEpisodeSearchMatch.
- Prioritization (`DownloadDecisionComparer.cs`) compares, in order: Quality → CustomFormatScore → Protocol (delay-profile preference) → EpisodeCount (season packs preferred) → EpisodeNumber → IndexerPriority → Peers (torrent) or Age (usenet) → Size (closest to preferred).
- `ProcessDownloadDecisions` grabs, or puts the release into Pending (reasons: Delay, DownloadClientUnavailable, Fallback).

**Download handling**
- `RefreshMonitoredDownloadsCommand` runs every 1 minute. `DownloadMonitoringService` polls every enabled client (`GetItems()`), and `TrackedDownloadService` matches items to history by download ID (`DownloadHistory`) or by parsing the title.
- States: Downloading, ImportBlocked, ImportPending, Importing, Imported, FailedPending, Failed, Ignored.
- Completed download handling imports from the client's output path through remote path mappings, then optionally removes the item from the client (`RemoveCompletedDownloads` / `RemoveFailedDownloads` per client) once seeding goals are met.
- Failed downloads are blocklisted and trigger an automatic redownload (`AutoRedownloadFailed`, `AutoRedownloadFailedFromInteractiveSearch`).
- `CheckForFinishedDownloadCommand` is a deprecated alias (`DownloadMonitoringService.cs:165`).

**Import** (`Core/MediaFiles/EpisodeImport/`)
- `ImportDecisionMaker` runs aggregators over `LocalEpisode`:
  - episodes, quality and language, each with **augmenters** that combine file name, folder name, download client item, media info and release name by confidence;
  - release group, release hash, subtitle info.
- Import specs: AbsoluteEpisodeNumber, AlreadyImported, EpisodeTitle (EpisodeTitleRequired: Always / BulkSeasonReleases / Never), FreeSpace, FullSeason, HasAudioTrack, MatchesFolder, MatchesGrab, NotSample (`DetectSample`, runtime-based), NotUnpacking, SameEpisodesImport, SplitEpisode, UnverifiedSceneNumbering, Upgrade.
- `ImportApprovedEpisodes` moves, copies or hardlinks files, handles upgrades (recycle bin), imports extras, and writes history.
- Manual import has its own API and command (`Manual/`).
- Season pack upgrade policy: `SeasonPackUpgradeType` plus a threshold.

**RSS sync**
- `RssSyncService` runs every `RssSyncInterval` (min 10 min, 0 = disabled; `Jobs/TaskManager.cs:185`).
- It fetches all RSS-enabled indexers (`FetchAndParseRssService`), runs the decision engine and processes decisions. It also re-evaluates pending releases.

**Search**
- Automatic search runs on add (`AddSeriesOptions.SearchForMissingEpisodes` / `SearchForCutoffUnmetEpisodes`) and on commands. Interactive search goes through `/release`.
- It builds criteria per series type (standard, daily, anime) and expands scene mappings. Newznab id search order is tvdbid → imdbid → rid (TVRage) → tvmazeid → tmdbid, then query and title text search (`Indexers/Newznab/NewznabRequestGenerator.cs`).
- `SeasonSearchMaximumSingleEpisodeAge` per indexer decides between a season search and individual episode searches.

**Renaming / organizing**
- `Organizer/FileNameBuilder.cs` tokens: `{Series Title|TitleYear|TitleWithoutYear|TitleThe|TitleTheYear|TitleTheWithoutYear|CleanTitle*|TitleFirstCharacter}`, `{Series Year}`, `{Season}`, `{Episode}`, `{Air Date}`, `{Episode Title|CleanTitle}`, `{Quality Full|Title|Proper|Real}`, `{MediaInfo Simple|Full|Video|VideoCodec|VideoBitDepth|Audio|AudioCodec|AudioChannels|AudioLanguages[All]|SubtitleLanguages[All]}`, `{Release Group}`, `{Release Hash}`, `{Original Title}`, `{Original Filename}`, `{Custom Formats}`, `{Custom Format:Name}`, `{ImdbId}`, `{TvdbId}`, `{TmdbId}`, `{TvMazeId}`. Absolute numbering tokens are available for anime.
- Multi-episode styles: Extend, Duplicate, Repeat, Scene, Range, PrefixedRange.
- Colon replacement modes include a custom string.
- Separate standard, daily and anime episode formats, plus series, season and specials folder formats.

**Metadata**
- Metadata consumers write NFO and images (see §4).
- The metadata source is SkyHook only. SkyHook (hosted by the Sonarr team) wraps TheTVDB and adds TVMaze, TMDB, IMDb, MAL and AniList IDs.

**Housekeeping** (`Core/Housekeeping/Housekeepers/`)
- CleanupAbsolutePathMetadataFiles, CleanupAdditionalNamingSpecs, CleanupAdditionalUsers, CleanupCommandQueue, CleanupDownloadClientUnavailablePendingReleases, CleanupDuplicateMetadataFiles, CleanupExtraFilesInExcludedFolders.
- Orphan cleanups: CleanupOrphaned{Blocklist, DownloadClientStatus, EpisodeFiles, Episodes, ExtraFiles, HistoryItems, ImportListStatus, IndexerStatus, MetadataFiles, NotificationStatus, PendingReleases, SubtitleFiles}.
- CleanupQualityProfileFormatItems, CleanupTemporaryUpdateFiles, CleanupUnusedTags, DeleteBadMediaCovers.
- FixFuture{DownloadClient, ImportList, Indexer, Notification, Provider}StatusTimes, FixFutureRunScheduledTasks.
- TrimLogDatabase, UpdateCleanTitleForSeries. Several of these repair clock skew or bad data.

**Health checks** (`Core/HealthCheck/Checks/`)
- AllowedHosts, ApiKeyValidation, AppDataLocation.
- DownloadClient, DownloadClientRemovesCompletedDownloads, DownloadClientRootFolder, DownloadClientSorting (SABnzbd/NZBGet sorting enabled), DownloadClientStatus.
- ImportListRootFolder, ImportListStatus, ImportMechanism.
- IndexerDownloadClient, **IndexerJackettAll**, IndexerLongTermStatus, IndexerRss, IndexerSearch, IndexerStatus.
- Mount (read-only mounts), NotificationStatus, OidcConfiguration, PackageGlobalMessage, Proxy, RecyclingBin, RemotePathMapping, RemovedSeries (deleted on TVDB), RootFolder, SystemTime, Update.

**Notifications:** see §4 and §5 for events.

**Backups / updates / auth:** see §8.

---

## 4. Integrations (full lists)

Legend: ⚠️ = obsolete, dead or legacy candidate.

### Indexers (`Core/Indexers/`)

| Implementation | Protocol | Notes |
|---|---|---|
| Newznab | usenet | Generic. Ships preset definitions (disabled): DOGnzb, DrunkenSlug, Nzb.life, NZBCat, NZBFinder.ws, NZBgeek, nzbplanet.net, SimplyNZBs, Tabula Rasa (`Newznab/Newznab.cs:51-59`). ⚠️ The preset list is stale and hard-coded (NZBCat / Nzb.life / Tabula Rasa are of doubtful status); in practice Prowlarr replaces it. |
| Torznab | torrent | Generic. Includes a Jackett "all" endpoint validation and health check (⚠️ Jackett-specific legacy). |
| BroadcasTheNet | torrent | Custom JSON-RPC API. Still searches by TVRage ID. |
| HDBits | torrent | Custom JSON API. |
| IPTorrents | torrent | RSS only. |
| TorrentLeech | torrent | RSS only, default `http://rss.torrentleech.org` (http). |
| FileList | torrent | API. |
| Nyaa | torrent | RSS/search. No default URL. |
| Torrent RSS Feed (`TorrentRss`) | torrent | Generic RSS with auto-detected parser settings (`TorrentRssSettingsDetector.cs`). |
| Fanzub | usenet | ⚠️ **Dead**: fanzub.com shut down around 2017. Default `http://fanzub.com/rss/` (`Fanzub/FanzubSettings.cs:25`). |

Supporting code: `EzrssTorrentRssParser.cs` ⚠️ (EZRSS namespace `xmlns.ezrss.it`; EZTV's ezRSS is long gone), `RssParser.cs`, `TorrentRssParser.cs`, `XmlCleaner.cs`, and the seed criteria (`SeedCriteriaSettings`, `SeasonPackSeedGoal`). Torznab also has settings to reject blocklisted hashes and blocked file extensions while grabbing.

### Download clients (`Core/Download/Clients/`)

Usenet:
- SABnzbd
- NZBGet: the proxy still has `DownloadNzbLegacy12/13` paths for NZBGet v12/v13 (⚠️ `Nzbget/NzbgetProxy.cs:73-107`)
- NZBVortex ⚠️ (macOS-only, effectively abandoned)
- Pneumatic ⚠️ (writes `.strm` for the Kodi/XBMC Pneumatic add-on; dead)
- Usenet Blackhole
- Download Station (Synology, usenet)

Torrent:
- qBittorrent: `QBittorrentProxyV1` targets the pre-4.1 WebUI API (⚠️)
- Transmission (min 2.40; labels need 4.0 or later)
- Deluge
- rTorrent (XML-RPC)
- uTorrent ⚠️ (WebUI; the client is widely considered abandonware or adware)
- Vuze ⚠️ (Vuze/Azureus discontinued around 2017; implemented on top of the Transmission RPC plugin, `Vuze/Vuze.cs`)
- Hadouken ⚠️ (project abandoned around 2017; requires v5.1 or later)
- Flood
- Aria2 (JSON-RPC)
- Freebox Download (French ISP box)
- Download Station (Synology, torrent; `DownloadStationTaskProxyV1` and `V2` handle legacy DSM versions ⚠️)
- Tribler (new)
- RQBit (new)
- Torrent Blackhole (watch folder with `ScanWatchFolder`)

### Notifications / Connect (`Core/Notifications/`)

- **Apprise**
- **Custom Script** (`CustomScript/`, env vars in §5)
- **Discord**
- **Email** (MailKit SMTP)
- **Emby / Jellyfin** (`MediaBrowser`: library refresh)
- **Gotify**
- **Join** (joaoapps; niche)
- **Kodi** (`Xbmc`: JSON-RPC library update and GUI notify)
- **Mailgun**
- **Notifiarr**
- **ntfy.sh**
- **Plex Media Server** (`Plex/Server`: library section refresh; includes `*Legacy` response models for old PMS versions ⚠️; `Plex/PlexTv` handles plex.tv OAuth PIN)
- **Prowl** ⚠️ (iOS, legacy but alive)
- **Pushbullet** (in maintenance mode)
- **Pushcut**
- **Pushover**
- **SendGrid**
- **Signal** (via signal-cli REST)
- **Simplepush**
- **Slack**
- **Synology Indexer** (runs `synoindex` locally on a Synology NAS)
- **Telegram**
- **Trakt** (collection sync)
- **Twitter** ⚠️: OAuth 1.0a with `api.twitter.com/1.1/statuses/update.json` and **`direct_messages/new.json`**, which was removed in 2018. The v1.1 free tier ended in 2023, so this is effectively dead (`Twitter/TwitterProxy.cs:53,72`). It also depends on the bundled OAuth 1.0 library `src/NzbDrone.Common/OAuth/`.
- **Webhook**

Events (`NotificationDefinition.cs`): OnGrab, OnDownload, OnUpgrade, OnImportComplete, OnRename, OnSeriesAdd, OnSeriesDelete, OnEpisodeFileDelete, OnEpisodeFileDeleteForUpgrade, OnHealthIssue (+IncludeHealthWarnings), OnHealthRestored, OnApplicationUpdate, OnManualInteractionRequired.

### Import lists (`Core/ImportLists/`)

Minimum refresh intervals are shown in parentheses.
- AniList List (12h; OAuth through `auth.servarr.com`)
- MyAnimeList (6h)
- Plex Watchlist (API via `discover.provider.plex.tv`, 6h)
- Plex Watchlist RSS (6h)
- Generic RSS list base
- Simkl User (6h; OAuth via servarr)
- **Sonarr** (another Sonarr instance, 5 min; `SonarrV3Proxy` calls `/api/v3/series|qualityprofile|languageprofile|rootfolder|tag`. The `LanguageProfileIds` field is ⚠️ deprecated, `Sonarr/SonarrSettings.cs:49`)
- Trakt List, Trakt Popular (`TraktPopularListType` has `[Obsolete]` members), Trakt User (12h; OAuth via `auth.servarr.com`)
- Custom List (a JSON URL returning TVDB IDs, 6h)

Types (`ImportListType`): Program, Plex, Trakt, Simkl, Other, Advanced. Also exclusions and `ListSyncLevel` / `ListSyncTag`.

### Metadata consumers (`Core/Extras/Metadata/Consumers/`)
- **Kodi (XBMC) / Emby**: NFO (tvshow.nfo, episode nfo, episodeguide) and images.
- **Plex**: `.plexmatch` file.
- **Roksbox** ⚠️: Roku Roksbox channel, dead since around 2014.
- **WDTV** ⚠️: WD TV Live devices, discontinued in 2016.

### Metadata sources
- **SkyHook** (`skyhook.sonarr.tv`): TheTVDB-backed. Carries TVMaze, TMDB, IMDb, MAL and AniList IDs. It is the only series info and search source.
- **services.sonarr.tv**: scene mappings, daily series list, update packages and changes, server-side messages.
- **TheXEM** (`thexem.info`): scene and absolute numbering.
- **TVRage** ⚠️: shut down in 2015. `Series.TvRageId`, Newznab `rid` search, BTN `Tvrage` parameter, and Newznab/Torznab `GetTvRageId` parsing remain (`Indexers/Newznab/NewznabRequestGenerator.cs:101-139`, `Indexers/BroadcastheNet/BroadcastheNetRequestGenerator.cs:239`).
- **auth.servarr.com**: OAuth relay for Trakt, AniList and Simkl.
- **Sentry** (`sentry.sonarr.tv`): crash reporting.

### Media servers
- Plex (notification plus `.plexmatch` metadata plus watchlist import lists), Kodi (notification plus NFO metadata), Emby/Jellyfin (notification plus Kodi-format NFO), Synology Indexer, Trakt.

### Grep results for obsolete / legacy markers
`[Obsolete]` or "Deprecated" appears in:
- `Core/Authentication/AuthenticationType.cs:8` (Basic)
- `Core/Tv/MonitoringOptions.cs:23` (LatestSeason) and `Core/Tv/EpisodeMonitoredService.cs:93` (LegacySetEpisodeMonitoredStatus)
- `Core/Queue/Queue.cs:19`
- `Core/ImportLists/Trakt/Popular/TraktPopularListType.cs:23,36`
- `src/Sonarr.Api.V3/Series/SeriesResource.cs:74`
- `src/Sonarr.Api.V3/EpisodeFiles/EpisodeFileController.cs:115`
- `src/Sonarr.Api.V3/ImportLists/ImportListExclusionController.cs:39`
- `src/Sonarr.Api.V3/Queue/QueueResource.cs:50,53`
- `src/Sonarr.Api.V3/Profiles/Languages/LanguageProfileController.cs` and `LanguageProfileSchemaController.cs` (stub that returns a single "Deprecated" profile)
- `src/Sonarr.Http/ApiInfoController.cs:16` (`Deprecated = ["v3"]`)
- `src/Sonarr.Http/Extensions/RequestExtensions.cs:16-18`

---

## 5. External APIs / interfaces

**API versions**
- `GET /api` returns `{current: "v5", deprecated: ["v3"]}`.
- **v3** (`src/Sonarr.Api.V3`) is the long-standing public API and is still what third-party tools (Prowlarr, Profilarr, Overseerr/Jellyseerr, Notifiarr, and others) use.
- **v5** (`src/Sonarr.Api.V5`) is new and is used by the frontend.

Both use API key auth through the `X-Api-Key` header or the `?apikey=` query parameter (`access_token` for SignalR) (`src/Sonarr.Http/Authentication/AuthenticationBuilderExtensions.cs:49-55`).

**v3 controllers** (`/api/v3/...`)
- **Series and episodes:** series, series/lookup, series/import, series/editor, series/{id}/folder, seasonpass, episode, episodefile (+bulk), rename, manualimport, parse.
- **Wanted and activity:** wanted/missing, wanted/cutoff, calendar (+ `feed/v3/calendar/Sonarr.ics` iCal feed), queue (+ queue/{id} actions grab/bulk/remove, queue/details, queue/status), history, blocklist, release (interactive search + grab), release/push (external release injection, used by autobrr and similar tools).
- **Providers:** indexer, downloadclient, notification, importlist, metadata. Each has schema, test, testall, action/{name} and bulk.
- **Profiles and formats:** qualityprofile (+schema), qualitydefinition, customformat (+schema, bulk), delayprofile, releaseprofile, autotagging, languageprofile (+schema; ⚠️ deprecated stub), language, indexerflag.
- **Library settings:** importlistexclusion, remotepathmapping, rootfolder, tag, tag/detail, customfilter.
- **Config:** config/host|ui|naming|mediamanagement|indexer|downloadclient|importlist.
- **System:** system (status, routes, restart, shutdown), system/backup (+restore), system/task, command, health, diskspace, log, log/file, log/file/update, update, filesystem (server-side directory browser), mediacover, localization.

**v5 differences**
- Notification is renamed to **connection** (`/api/v5/connection`).
- config/* is renamed to **settings/*** (general, ui, naming, mediamanagement, indexer, downloadclient, importlist, **update**). Host config becomes "general" and update settings are split out.
- New `/api/v5/statistics`.
- `languageprofile` is gone.
- The queue is per download (`QueueService`), not per episode (the v3 queue still uses `ObsoleteQueueService`).
- Release endpoints were moved into a `Release/` namespace.
- Provider base classes were refactored (`Provider/`).

**Non-versioned endpoints (Sonarr.Http):** `/ping`, `/initialize.json` (frontend bootstrap: apiRoot, apiKey, urlBase, version, instanceName, theme), `/login` (GET/POST), `/login/sso` (OIDC), `/logout`, `/content/*` and SPA fallback, `/api`, and `/signalr/messages` (hub).

**SignalR** (`src/NzbDrone.SignalR/MessageHub.cs`)
- A single hub broadcasts `receiveMessage` with `{name, body:{action, resource}}`. `RestControllerWithSignalR` sends it on model changes.
- Names handled by the UI (`frontend/src/Components/SignalRListener.tsx`): calendar, command, connection, downloadclient, episode, episodefile, health, importlist, indexer, metadata, qualitydefinition, queue, queue/details, queue/status, rootfolder, series, system/task, tag, version, wanted/cutoff, wanted/missing.

**Webhooks** (`Core/Notifications/Webhook/`)
- JSON POST (or PUT) with `eventType` ∈ Test, Grab, Download, Rename, SeriesAdd, SeriesDelete, EpisodeFileDelete, Health, HealthRestored, ApplicationUpdate, ManualInteractionRequired. The enum is PascalCase (`WebhookEventType.cs`, with a TODO to change it).
- Payloads include series, episodes, episodeFile(s), release, downloadClient, customFormatInfo, and similar. Notifiarr reuses `WebhookBase`.

**Custom script** (`Core/Notifications/CustomScript/CustomScript.cs`)
- Passes environment variables `Sonarr_EventType` plus about 95 others:
  - `Sonarr_Series_*` (Id, Title, TitleSlug, Path, TvdbId, TvMazeId, TmdbId, ImdbId, Type, Year, Genres, Tags, OriginalLanguage, OriginalCountry)
  - `Sonarr_Release_*` (Title, Indexer, Size, Quality, QualityVersion, ReleaseGroup, ReleaseType, IndexerFlags, EpisodeNumbers, AbsoluteEpisodeNumbers, EpisodeAirDates[Utc], EpisodeTitles, EpisodeOverviews, FinaleTypes, CustomFormat, CustomFormatScore, SeasonNumber, EpisodeCount)
  - `Sonarr_EpisodeFile_*` (Id(s), Path(s), RelativePath(s), Quality, SceneName, ReleaseGroup, SourcePath, SourceFolder, PreviousPaths, MediaInfo_*)
  - `Sonarr_Download_{Client,Client_Type,Id,Size,Title}`, `Sonarr_Deleted*`, `Sonarr_Health_{Issue,Restored}_*`, `Sonarr_Update_*`, `Sonarr_IsUpgrade`, `Sonarr_InstanceName`, `Sonarr_ApplicationUrl`
- **Script import** (`MediaFiles/ScriptImportDecider.cs`) passes a similar set plus `Sonarr_SourcePath`, `Sonarr_DestinationPath` and `Sonarr_TransferMode`. The script's stdout can override the move result.

**Auth methods**
- None, Basic (⚠️ obsolete), Forms (cookie), External (reverse proxy), **OIDC** (new: Authority, ClientId, ClientSecret, UserIdentifier, Scopes).
- `AuthenticationRequired` can be Enabled or DisabledForLocalAddresses.
- `TrustedNetworks` and `TrustCgnatIpAddresses` control what counts as local. `AllowedHosts` provides host filtering.
- API key via header or query.

**Other interfaces**
- iCal feed (`Sonarr.ics`, with past/future days, tags, unmonitored and premieres-only filters).
- RSS-like `release/push` for external grabbers.
- OpenAPI JSON files.

---

## 6. Background jobs / commands

**Scheduled tasks** (`Core/Jobs/TaskManager.cs:66-135`, interval in minutes)

| Command | Interval |
|---|---|
| RefreshMonitoredDownloadsCommand | 1 |
| MessagingCleanupCommand | 5 |
| ImportListSyncCommand | 5 (each list is additionally gated by its MinRefreshInterval: 5 min to 12h) |
| UpdateSceneMappingCommand | 180 |
| ApplicationUpdateCheckCommand | 360 |
| CheckHealthCommand | 360 |
| RefreshSeriesCommand | 720 (refreshes stale series per `ShouldRefreshSeries`) |
| HousekeepingCommand | 1440 |
| CleanUpRecycleBinCommand | 1440 |
| BackupCommand | `BackupInterval` days (1–7) × 1440 |
| RssSyncCommand | `RssSyncInterval` (0 disables, otherwise at least 10) |

**All command types:** ApplicationUpdate, ApplicationUpdateCheck, Backup, BulkMoveSeries, CheckForFinishedDownload (⚠️ deprecated alias), CheckHealth, CleanUpRecycleBin, ClearBlocklist, ClearLog, CutoffUnmetEpisodeSearch, DeleteLogFiles, DeleteSeriesFiles, DeleteUpdateLogFiles, DownloadedEpisodesScan (external trigger to import a path; used by download-client post-processing scripts), EpisodeSearch, Housekeeping, ImportListSync, ManualImport, MessagingCleanup, MissingEpisodeSearch, MoveSeries, ProcessMonitoredDownloads, RefreshMonitoredDownloads, RefreshSeries, RenameFiles, RenameSeries, RescanSeries, ResetApiKey, ResetQualityDefinitions, Restart, RssSync, SeasonSearch, SeriesSearch, Shutdown, Test, Unknown, UpdateSceneMapping.

Commands are persisted in the `Commands` table and run on a priority queue with an exclusivity model (`Core/Messaging/Commands/CommandQueueManager.cs`). Completed commands are cleaned by MessagingCleanup and housekeeping.

---

## 7. Configuration surface

- **config.xml / env** (`ConfigFileProvider`, `src/NzbDrone.Common/Options/*`):
  - BindAddress, Port, SslPort, EnableSsl, SslCertPath / SslKeyPath / SslCertPassword, UrlBase, LaunchBrowser, InstanceName
  - AuthenticationMethod, AuthenticationRequired, ApiKey, AllowedHosts, TrustedNetworks, TrustCgnatIpAddresses, Oidc*
  - LogLevel, ConsoleLogLevel, ConsoleLogFormat, LogSql, LogRotate, LogSizeLimit, LogDbEnabled, Syslog{Server,Port,Level}, FilterSentryEvents, AnalyticsEnabled
  - Branch, UpdateAutomatically, UpdateMechanism, UpdateScriptPath
  - Theme, UiFolder, ProfilerEnabled / ProfilerPosition
  - Postgres{Host,Port,User,Password,MainDb,LogDb,*ConnectionString}
- **DB config** (`Core/Configuration/IConfigService.cs`):
  - *Media management:* AutoUnmonitorPreviouslyDownloadedEpisodes, RecycleBin + CleanupDays, DownloadPropersAndRepacks, CreateEmptySeriesFolders, DeleteEmptyFolders, FileDate (None / LocalAirDate / UtcAirDate), free-space checks when grabbing and importing, MinimumFreeSpaceWhenImporting, CopyUsingHardlinks, EnableMediaInfo, UseScriptImport / ScriptImportPath, ImportExtraFiles / ExtraFileExtensions, RescanAfterRefresh, EpisodeTitleRequired, UserRejectedExtensions, SeasonPackUpgrade + Threshold, SetPermissionsLinux / ChmodFolder / ChownGroup.
  - *Indexers:* Retention, RssSyncInterval, MaximumSize, MinimumAge.
  - *Import lists:* ListSyncLevel, ListSyncTag.
  - *Download clients:* DownloadClientWorkingFolders, DownloadClientHistoryLimit, EnableCompletedDownloadHandling, AutoRedownloadFailed (+FromInteractiveSearch).
  - *UI:* FirstDayOfWeek, CalendarWeekColumnHeader, Short/LongDateFormat, TimeFormat, TimeZone, ShowRelativeDates, EnableColorImpairedMode, UILanguage.
  - *General / host:* proxy (Http / Socks4 / Socks5, bypass filter and local addresses), CertificateValidation, backup folder / interval / retention, ApplicationUrl.
  - *Internal:* PlexClientIdentifier, CleanupMetadataImages, and ⚠️ **RijndaelPassphrase / HmacPassphrase / RijndaelSalt / HmacSalt**, which are defined in `IConfigService` / `ConfigService` but **not referenced anywhere else** (dead leftovers from the old Nancy forms-auth).
- **Other settings entities:** naming (NamingConfig), quality profiles and definitions, custom formats, delay and release profiles, auto-tagging, tags, root folders, remote path mappings, indexers, download clients, connections, import lists and exclusions, metadata consumers, custom filters, UI settings.
- **Frontend settings pages** (`frontend/src/Settings/`): MediaManagement, Profiles, Quality, CustomFormats, Indexers, DownloadClients, ImportLists, Connect, Metadata, MetadataSource (Plex/Trakt auth), Tags, General, UI.

---

## 8. Platform / infrastructure concerns

- **Hosting:** `src/NzbDrone.Host` (ASP.NET Core Kestrel, `Bootstrap.cs`, `Startup.cs`). Application modes: Interactive, Service, InstallService, UninstallService, RegisterUrl, Help (`ApplicationModes.cs`). Startup flags: `/data=`, `/nobrowser`, `/i`, `/u`, `/terminateexisting`, `/restart`, `/registerurl`, `/nosingleinstancecheck`.
  - ⚠️ `RegisterUrl` and `Host/AccessControl/FirewallAdapter.cs` (Windows COM `HNetCfg.FwMgr` firewall) and `RemoteAccessAdapter.cs` are HttpListener-era leftovers.
  - `SingleInstancePolicy` and PID file.
- **Windows:** `src/NzbDrone` (WinForms tray app `SysTray/SysTrayApp.cs`, `net10.0-windows`), `src/ServiceHelpers/ServiceInstall|ServiceUninstall` (separate exes), `src/NzbDrone.Windows` (DiskProvider, FolderMount, WindowsVersionInfo), `Microsoft.Extensions.Hosting.WindowsServices`, `LongPathSupport.cs`, and the Inno installer in `distribution/windows`.
- **Unix ("Mono" project, now just Unix):** `src/NzbDrone.Mono` (Mono.Unix: chmod/chown, `/proc/mounts`, drive type, symlink resolution, OS version adapters for FreeBSD, macOS, Synology, `/etc/issue`, `os-release`). ⚠️ The name "Mono" is historical; it is not the Mono runtime any more.
- ⚠️ **`src/Sonarr.RuntimePatches`**: Harmony patches for **Mono runtime bugs** (`Mono/BoringTLSVerifyFlagsPatch.cs`, `Mono/DeflateStreamAsyncPatch.cs`, "Mono 6.0 - 6.x bug 16122"). They only activate when `Mono.Runtime` exists, which it never does on .NET 10, so this is dead code. Env vars `DISABLE_RUNTIMEPATCHES` and `DEBUG_RUNTIMEPATCHES`.
- ⚠️ **`AppFolderFactory` legacy AppData migration** (`src/NzbDrone.Common/EnvironmentInfo/AppFolderFactory.cs:107-148`, `LegacyAppDataFolder`) moves the NzbDrone-era data folder.
- **Namespaces are still `NzbDrone.*`** (the original 2011 name), while assemblies are `Sonarr.*`.
- **Self-updater:** `src/NzbDrone.Update` (a separate exe). It detects the app type (service, tray or console), terminates Sonarr, backs up the app and AppData, copies the new files, restarts, and rolls back on failure (`UpdateEngine/*`). Update mechanisms are BuiltIn, Script, External, Apt and Docker; for package installs, `DeploymentInfoProvider` reads `package_info`. Updates come from `services.sonarr.tv/v1/update/{branch}`, with `UpdateVerification` hash checks.
- **Logging:** NLog to rolling files (`logs/sonarr.txt`, debug and trace), a DB log target (`logs.db` table `Logs`, trimmed by housekeeping), console (text or CLEF JSON), Syslog, and Sentry (with a debounce). `CleanseLogMessage` redacts API keys and passwords.
- **DB:** SQLite (default) or Postgres (separate main and log DBs), FluentMigrator, and a custom lightweight ORM with LazyLoaded relations. JSON columns hold settings, quality models, seasons, and similar. The migration for rank indexes uses `json_extract` on SQLite and `::jsonb` on Postgres. `DatabaseRestorationService` restores a backup on the next startup.
- **Caching:** an in-memory `ICacheManager` / `ICached<T>` (`src/NzbDrone.Common/Cache`). No external cache.
- **HTTP client:** `src/NzbDrone.Common/Http`. Custom `HttpClient` wrapper over `SocketsHttpHandler` (`ManagedHttpDispatcher`) with **Happy Eyeballs** (`HttpHappyEyeballs.cs`), proxy support (HTTP / SOCKS4 / SOCKS5), cookie containers per provider, rate limiting (`TPL/RateLimitService.cs`), JSON-RPC and XML-RPC builders, a Cloudflare captcha detector, and a Torcache interceptor (⚠️ dead).
- **Web:** controllers and middleware (URL base, caching headers, if-modified, versioning, starting-up gating), the error pipeline, Swagger, MiniProfiler, and forwarded-headers configuration.
- **Media:** ffprobe (FFMpegCore) for media info, ImageSharp for covers.
- **Packaging:** Docker (`distribution/docker-build`), Debian `install.sh` + systemd unit, macOS `.app`, Windows installer.

---

## 9. Interactions with other *arr apps

- **Prowlarr → Sonarr:** Prowlarr pushes indexers into Sonarr through **v3** `POST/PUT/DELETE /api/v3/indexer` (plus `/indexer/schema`, `/indexer/test`, `/system/status`) as **Newznab/Torznab** definitions pointing at Prowlarr's proxy URL. Sonarr has no Prowlarr-specific code: `grep -ri prowlarr` in `src/` finds nothing, and the relevant pieces are generic Newznab/Torznab plus the `SeasonSearchMaximumSingleEpisodeAge`, `AnimeCategories`, `AnimeStandardFormatSearch`, `MultiLanguages` and `FailDownloads` settings. Prowlarr can also use `release/push`.
- **Sonarr → Sonarr:** the `Sonarr` import list pulls series from another Sonarr instance, filtered by quality profile, tag or root folder, with optional season monitoring sync (`Core/ImportLists/Sonarr/`). It calls v3 endpoints.
- **Radarr:** no direct integration.
- **Profilarr → Sonarr:** Profilarr uses the v3 API: `/api/v3/customformat` (+schema, bulk), `/api/v3/qualityprofile` (+schema), `/api/v3/qualitydefinition`, `/api/v3/releaseprofile`, `/api/v3/language`, `/api/v3/tag`. Sonarr has no Profilarr-specific code. Things a rewrite must keep for Profilarr compatibility:
  - the CF spec JSON shape: implementation name plus `fields[]` array, `negate`, `required`;
  - quality IDs and names;
  - profile `items` groups;
  - per-profile item sizes (new).
- **Shared Servarr infrastructure:** `auth.servarr.com` (OAuth relay for Trakt, AniList and Simkl), `wiki.servarr.com` help links, the Notifiarr integration, and Sentry.
- **Other ecosystem consumers of the v3 API:** Overseerr/Jellyseerr (series add and lookup), Bazarr (episode files, history, SignalR), Tautulli, LunaSea/nzb360, Organizr, Homepage, and autobrr (`release/push`). Retiring v3 affects all of these.
- **Download-client post-processing** triggers `DownloadedEpisodesScan` through `POST /api/v3/command` (for example from nzbToMedia and SABnzbd scripts).

---

## 10. Outdated / legacy candidates

| Item | Evidence | Why |
|---|---|---|
| Fanzub indexer | `Core/Indexers/Fanzub/` | Service dead (around 2017) |
| EZRSS parser | `Core/Indexers/EzrssTorrentRssParser.cs` | ezRSS namespace long dead; generic RSS covers it |
| TVRage ID support | `Series.TvRageId`, `Indexers/Newznab/NewznabRequestGenerator.cs:101`, `NewznabRssParser.cs:175`, `TorznabRssParser.cs:169`, `BroadcastheNet/BroadcastheNetRequestGenerator.cs:239`, migration `002_remove_tvrage_imdb_unique_constraint.cs` | TVRage shut down in 2015 |
| Torcache interceptor | `Core/Http/TorcacheHttpInterceptor.cs` | torcache.net dead since 2016 |
| Twitter notification | `Core/Notifications/Twitter/`, `src/NzbDrone.Common/OAuth/` | v1.1 endpoints removed or paid; DM endpoint gone |
| Vuze client | `Core/Download/Clients/Vuze/` | Discontinued |
| Hadouken client | `Core/Download/Clients/Hadouken/` | Abandoned |
| uTorrent client | `Core/Download/Clients/uTorrent/` | Abandonware or adware; WebUI token API |
| Pneumatic client | `Core/Download/Clients/Pneumatic/` | Kodi add-on dead; writes `.strm` files |
| NZBVortex client | `Core/Download/Clients/NzbVortex/` | Essentially unmaintained |
| NZBGet v12/v13 paths | `Core/Download/Clients/Nzbget/NzbgetProxy.cs:73-107` | Ancient NZBGet versions (NZBGet itself was revived as nzbget-ng) |
| qBittorrent API v1 | `Core/Download/Clients/QBittorrent/QBittorrentProxyV1.cs` | qBittorrent before 4.1 (2018) |
| Synology DownloadStation task API v1 | `Core/Download/Clients/DownloadStation/Proxies/DownloadStationTaskProxyV1.cs` | Old DSM |
| Plex "Legacy" server responses | `Core/Notifications/Plex/Server/PlexSection.cs:40-55`, `PlexPreferences.cs:19`, `PlexSectionItem.cs:27` | Old PMS API |
| Roksbox metadata | `Core/Extras/Metadata/Consumers/Roksbox/` | Platform dead |
| WDTV metadata | `Core/Extras/Metadata/Consumers/Wdtv/` | Hardware discontinued in 2016 |
| Prowl, Join, Pushbullet notifications | `Core/Notifications/{Prowl,Join,PushBullet}` | Niche or low-maintenance; Apprise and ntfy cover them |
| Stale Newznab preset list | `Core/Indexers/Newznab/Newznab.cs:51-59` | Hard-coded third-party indexers; Prowlarr supersedes it |
| Jackett "all" check | `Core/Indexers/Torznab/Torznab.cs:126`, `HealthCheck/Checks/IndexerJackettAllCheck.cs` | Jackett-specific; Prowlarr era |
| Basic auth | `Core/Authentication/AuthenticationType.cs:8` | `[Obsolete]` |
| Language profiles (v3 stub + Sonarr list field) | `src/Sonarr.Api.V3/Profiles/Languages/*`, `Core/ImportLists/Sonarr/SonarrSettings.cs:49` | Removed in v4; only stubs remain |
| ObsoleteQueueService / per-episode queue | `Core/Queue/ObsoleteQueueService.cs`, `Queue.cs:19` | Exists only for v3 |
| MonitorTypes.LatestSeason + LegacySetEpisodeMonitoredStatus | `Core/Tv/MonitoringOptions.cs:23`, `EpisodeMonitoredService.cs:93` | Obsolete monitor mode |
| Trakt Popular obsolete list types | `Core/ImportLists/Trakt/Popular/TraktPopularListType.cs:23,36` | `[Obsolete]` |
| CheckForFinishedDownloadCommand | `Core/Download/CheckForFinishedDownloadCommand.cs`, `TrackedDownloads/DownloadMonitoringService.cs:165` | Deprecated alias |
| Deprecated v3 fields / endpoints | `src/Sonarr.Api.V3/Series/SeriesResource.cs:74`, `EpisodeFiles/EpisodeFileController.cs:115`, `ImportLists/ImportListExclusionController.cs:39`, `Queue/QueueResource.cs:50,53` | Marked `[Obsolete]` |
| Entire v3 API | `src/Sonarr.Http/ApiInfoController.cs:16` | Declared deprecated; about 10.7k LOC duplicated with v5 |
| Unused Rijndael/HMAC config | `Core/Configuration/IConfigService.cs` (RijndaelPassphrase etc.) | No consumers |
| Mono runtime patches | `src/Sonarr.RuntimePatches/Mono/*` | Never run on .NET 10 |
| "Mono" project naming, NzbDrone namespaces | `src/NzbDrone.Mono`, namespaces throughout | Historical |
| Legacy AppData migration | `src/NzbDrone.Common/EnvironmentInfo/AppFolderFactory.cs:107-148` | NzbDrone-era folder move |
| Windows firewall / URL ACL registration, tray app, service installer exes | `src/NzbDrone.Host/AccessControl/*`, `ApplicationModes.RegisterUrl`, `src/NzbDrone/SysTray`, `src/ServiceHelpers/*` | HttpListener and Windows-desktop heritage |
| FluentValidation 9, RestSharp 106, Newtonsoft + System.Text.Json both used | csproj files | Library debt |
| jQuery / moment / lodash in frontend | `package.json` | Legacy alongside React Query and zustand |
| Webhook PascalCase enum TODO | `Core/Notifications/Webhook/WebhookEventType.cs` | Pending breaking change |
| macOS SQLite TRUNCATE journal | `Core/Datastore/ConnectionStringFactory.cs:64` | Platform workaround |

---

## 11. Rewrite notes

**Must preserve (subtle behavior)**
- **Parser cascade order** and the 1,631 parser test cases (`src/NzbDrone.Core.Test/ParserTests`). This covers:
  - anime sub-group formats with hashes, versions (`v2`) and batches;
  - daily date formats and ambiguity (MM.DD vs DD.MM);
  - multi-episode styles, split episodes (`S01E05a`), NNMM numbering, 4-digit seasons, mini-series `Part N` / `XofY`;
  - locale-specific trackers (Spanish, Turkish, Dutch, Chinese, Japanese);
  - season and multi-season packs, and partial packs.
- Title normalization and clean-title matching (`Parser.CleanSeriesTitle`, `SeriesTitleNormalizer`).
- **Scene numbering:** the TVDB order vs XEM scene season/episode/absolute mappings (`Episode.Scene*`, `UnverifiedSceneNumbering`). Scene mappings carry per-season scene titles, `SceneOrigin`, `SearchMode`, `FilterRegex` and search terms. Both parsing (release to episode) and searching (episode to indexer query, `SceneEpisodeMapping` / `SceneSeasonMapping`) use them.
- **Anime:** absolute episode numbering (`AbsoluteEpisodeNumber`, `SceneAbsoluteEpisodeNumber`, `AbsoluteEpisodeNumberAdded`), the `AnimeEpisodeSearchCriteria` / `AnimeSeasonSearchCriteria` searches, Newznab `AnimeCategories` / `AnimeStandardFormatSearch`, and `AnimeVersionUpgradeSpecification`.
- **Daily series:** matching by air date, the services.sonarr.tv daily list overriding the type, the daily naming format, and daily search criteria.
- **Specials:** AiredBefore/After placement and matching specials by title (`SpecialEpisodeSearchCriteria`, `ParsingService` special handling).
- Season packs:
  - `SeasonSearchMaximumSingleEpisodeAge`
  - `SeasonPackUpgradeType` and its threshold
  - the release-profile `AllowSeasonPackWithoutAllEpisodesAired`
  - `SeasonPackSeedGoal`
  - episode-count preference in the comparer
- Decision comparer ordering and delay-profile semantics (bypass rules, pending release reasons, the fallback when a download client is unavailable).
- The proper/repack/real revision model and anime versioning. `DownloadPropersAndRepacks` interacts with CF scores.
- Tracked download matching: download ID via `DownloadHistory` first, then title parse. Remote path mappings. Import blocked vs. pending states. The manual-interaction-required event.
- The import pipeline's augmenter confidence model for quality and language (file name vs folder vs client item vs media info).
- Naming token grammar: separators, casing modifiers, `{Custom Format:Name}`, truncation (`{Episode Title:30}`), the multi-episode styles and colon replacement.
- `ProviderDefinition.Implementation` / `ConfigContract` strings and settings JSON, needed to migrate existing DBs and to stay compatible with Prowlarr's indexer push and with Profilarr's CF and profile payloads (field names and ordering in `fields[]`).
- The custom-script env var names and the webhook payload shapes, which are a public contract.
- The Newznab ID-search fallback order and caps parsing (`NewznabCapabilitiesProvider`).
- Provider failure escalation and backoff (`EscalationBackOff`) and the "fix future timestamps" housekeepers.

**Accidental complexity / consolidation opportunities**
- Two full API surfaces (v3 and v5, about 21k LOC together) plus two queue services. A Go rewrite could keep one canonical internal model and a thin v3-compatible façade only for third-party consumers (Prowlarr, Profilarr, Overseerr, Bazarr).
- The ThingiProvider pattern (indexers, download clients, notifications, import lists, metadata) is identical across the *arrs. It is a prime candidate for one shared provider framework in the monolith, together with provider status and backoff, tags, and the schema-driven settings UI.
- Nearly identical cross-app pieces (expect duplicates with Radarr): download-client implementations, notifications, the HTTP client, config, backup, update, health checks, housekeeping, log DB, SignalR hub, auth including OIDC, command queue, custom formats, quality profiles, delay and release profiles, remote path mappings, root folders, blocklist and history.
- The custom ORM plus JSON-in-column storage and dual SQLite/Postgres migrations (226 of them). A rewrite should start from a clean schema and a one-shot importer.
- Windows and Mono heritage: tray app, service installers, firewall and URL ACL code, runtime patches, legacy AppData migration.
- The hard-coded Newznab presets, Jackett-specific checks and several dead integrations (§10).
