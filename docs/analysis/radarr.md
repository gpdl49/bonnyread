# Radarr — Codebase Analysis

Source: `submodules/radarr` @ `c90668a5` (2026-09-20). Paths below are relative to `submodules/radarr/` unless noted; `Core/` = `src/NzbDrone.Core/`.

---

## 1. Overview

**Purpose:** Automated movie PVR. It monitors a movie library, searches indexers (Usenet/torrent) via RSS and search, picks releases using quality/custom-format/language rules, sends them to download clients, then imports, renames and organizes the files. It also notifies media servers and other services. Movie metadata comes from TMDb through Radarr's own proxy (`api.radarr.video`).

**Tech stack**
- Backend: C# on **.NET 8** (`global.json` SDK 8.0.421; 22 projects `net8.0`, 1 `net8.0-windows` = `src/NzbDrone/Radarr.csproj`, a WinForms tray app).
- Web: ASP.NET Core MVC controllers, SignalR (`/signalr/messages`), Swashbuckle OpenAPI (`src/Radarr.Api.V3/openapi.json`).
- DI: DryIoc (`DryIoc.dll 5.4.3`) plus assembly scanning (`src/NzbDrone.Common/Composition/`).
- DB: **SQLite** (`System.Data.SQLite 2.0.4`, `SourceGear.sqlite3 3.53.4`) or **PostgreSQL** (`Npgsql 9.0.5`). ORM is **Dapper** with a custom SQL builder (`Core/Datastore/SqlBuilder.cs`, `WhereBuilderSqlite.cs`, `WhereBuilderPostgres.cs`). Migrations use **FluentMigrator 6.2**.
- Notable libs: NLog 5 (+Syslog, CLEF JSON), Sentry 5 (`sentry.servarr.com`), Polly, FluentValidation 9, Newtonsoft.Json + System.Text.Json (both), RestSharp 106 (legacy), MailKit, Ical.Net (iCal feed), SixLabors.ImageSharp (poster resize), Servarr.FFMpegCore/FFprobe (media info), MonoTorrent (.torrent parsing), Mono.Posix.NETStandard (Unix syscalls), SharpZipLib, Diacritical.Net, IPAddressRange, ImpromptuInterface.
- Frontend: React 18.3 + Redux 4 (legacy JS store `frontend/src/Store/Actions/*.js`) that is being migrated to TypeScript + `@tanstack/react-query` 5. Also react-router 5.2, `@microsoft/signalr` 8, webpack 5, TypeScript 5.7. There are 1,015 `.ts/.tsx` files and 426 `.js` files.

**Size (C# LOC, incl. blanks)**

| Project | LOC |
|---|---|
| NzbDrone.Core (`Radarr.Core`) | 101,119 |
| Radarr.Api.V3 | 11,191 |
| NzbDrone.Common | 13,559 |
| Radarr.Http | 3,354 |
| NzbDrone.Host | 1,624 |
| NzbDrone.Mono | 1,401 |
| NzbDrone.Update | 667 |
| NzbDrone.Windows | 296 |
| NzbDrone (tray) / Console / ServiceHelpers / SignalR | 243 / 130 / 140 / 94 |
| Tests (Core.Test 49k, Common.Test 5.4k, Integration 2.7k, others) | ~60,000 |
| frontend/src (ts/tsx/js) | ~100,600 |

Largest Core areas by LOC: Download 19.6k, Datastore 12.9k (mostly migrations), Notifications 12.4k, ImportLists 7.2k, MediaFiles 6.8k, Indexers 6.4k, Parser 3.8k, Movies 3.7k, Extras 3.5k, HealthCheck 2.9k, DecisionEngine 2.8k.

---

## 2. Domain model

Tables are mapped in `Core/Datastore/TableMapping.cs`. Entities derive from `ModelBase` (`int Id`).

| Entity (table) | Key fields / notes | File |
|---|---|---|
| **Movie** (`Movies`) | Per-library, user-owned state: `MovieMetadataId`, `Monitored`, `MinimumAvailability` (MovieStatusType), `QualityProfileId`, `Path`, `RootFolderPath`, `Added`, `Tags`, `AddOptions`, `LastSearchTime`, `MovieFileId` (**single file per movie**). Compatibility pass-through props `Title/TmdbId/ImdbId/Year`. Contains `IsAvailable(delay)` and `GetReleaseDate()` logic | `Core/Movies/Movie.cs` |
| **MovieMetadata** (`MovieMetadata`) | TMDb-sourced and shared (split from Movie in migration 207). Fields: `TmdbId`, `ImdbId`, `Title`, `CleanTitle`, `SortTitle`, `OriginalTitle`, `CleanOriginalTitle`, `OriginalLanguage`, `Status`, `Overview`, `Year`, `SecondaryYear`, `InCinemas`, `PhysicalRelease`, `DigitalRelease`, `Certification`, `Runtime`, `Website`, `YouTubeTrailerId`, `Studio`, `Genres`, `Keywords` (mig 242), `Ratings` (multi-source: imdb/tmdb/metacritic/rottenTomatoes/trakt, mig 206), `Images`, `CollectionTmdbId`, `CollectionTitle`, `Recommendations` (List<int> TmdbIds), `Popularity`, `LastInfoSync`. Plus lazy `AlternativeTitles` and `Translations` | `Core/Movies/MovieMetadata.cs` |
| **MovieStatusType** | `Deleted=-1, TBA=0, Announced=1, InCinemas=2, Released=3` (PreDB was removed in mig 202) | `Core/Movies/MovieStatusType.cs` |
| **MovieCollection** (`Collections`) | `TmdbId`, `Title`, `CleanTitle`, `SortTitle`, `Overview`, `Monitored`, `QualityProfileId`, `RootFolderPath`, `SearchOnAdd`, `MinimumAvailability`, `Images`, `Tags`, `LastInfoSync`. Movies are linked through `MovieMetadata.CollectionTmdbId`, not an FK table | `Core/Movies/Collections/MovieCollection.cs` |
| **Credit** (`Credits`) | `CreditTmdbId`, `PersonTmdbId`, `MovieMetadataId`, `Name`, `Character`, `Department`, `Job`, `Order`, `Type` (Cast/Crew), `Images` | `Core/Movies/Credits/Credit.cs` |
| **AlternativeTitle** (`AlternativeTitles`) | `MovieMetadataId`, `Title`, `CleanTitle`, `SourceType` (TMDB/Mappings/User/Indexer) | `Core/Movies/AlternativeTitles/AlternativeTitle.cs` |
| **MovieTranslation** (`MovieTranslations`) | `MovieMetadataId`, `Title`, `CleanTitle`, `Overview`, `Language` | `Core/Movies/Translations/MovieTranslation.cs` |
| **MovieFile** (`MovieFiles`) | `MovieId`, `RelativePath`, `Size`, `DateAdded`, `OriginalFilePath`, `SceneName`, `ReleaseGroup`, `Quality` (QualityModel, embedded JSON), `IndexerFlags`, `MediaInfo` (embedded JSON from FFprobe), `Edition`, `Languages` | `Core/MediaFiles/MovieFile.cs` |
| **QualityProfile** (`QualityProfiles`) | `Name`, `Cutoff`, `Items` (nested groups), `UpgradeAllowed`, `MinFormatScore`, `CutoffFormatScore`, `MinUpgradeFormatScore`, `FormatItems` (CF→score), **`Language`** (Radarr-only; may be `Any`=-1 or `Original`=-2) | `Core/Profiles/Qualities/QualityProfile.cs` |
| **QualityDefinition** (`QualityDefinitions`) | min/preferred/max size per quality | `Core/Qualities/QualityDefinition.cs` |
| **Quality** (static) | Unknown, WORKPRINT, CAM, TELESYNC, TELECINE, DVDSCR, REGIONAL, SDTV, DVD, DVDR, HDTV720/1080/2160p, WEBDL480–2160p, WEBRip480–2160p, Bluray480/576/720/1080/2160p, Remux1080/2160p, BRDISK, RAWHD. Source enum: CAM, TELESYNC, TELECINE, WORKPRINT, DVD, TV, WEBDL, WEBRIP, BLURAY. Modifier enum: NONE, REGIONAL, SCREENER, RAWHD, BRDISK, REMUX | `Core/Qualities/Quality.cs`, `QualitySource.cs`, `Modifier.cs` |
| **CustomFormat** (`CustomFormats`) | `Name`, `IncludeCustomFormatWhenRenaming`, `Specifications` (polymorphic JSON) | `Core/CustomFormats/` |
| **ReleaseProfile** (`ReleaseProfiles`) | `Required`/`Ignored` terms, `IndexerId`, `Tags` (converted from "Restrictions" in mig 229) | `Core/Profiles/Releases/ReleaseProfile.cs` |
| **DelayProfile** (`DelayProfiles`) | usenet/torrent delay, preferred protocol, bypass-if-highest-quality / above-CF-score, tags | `Core/Profiles/Delay/DelayProfile.cs` |
| **MovieHistory** (`History`) | `MovieId`, `SourceTitle`, `Quality`, `Languages`, `EventType` (Grabbed, DownloadFolderImported, DownloadFailed, MovieFileDeleted, MovieFolderImported [unused], MovieFileRenamed, DownloadIgnored), `DownloadId`, `Data` dictionary | `Core/History/History.cs` |
| **DownloadHistory** (`DownloadHistory`) | per-download-id event log for tracked downloads | `Core/Download/History/` |
| **Blocklist**, **PendingRelease**, **ImportListMovie** (`ImportListMovies`, cached list results that feed "Discover"), **ImportListExclusion** (`ImportExclusions`, by TmdbId), **RootFolder**, **RemotePathMapping**, **Tag**, **AutoTag** (`AutoTagging`), **User**, **CommandModel** (`Commands`, persisted command queue), **ScheduledTask**, **Config** (key/value), **NamingConfig**, **CustomFilter**, **UpdateHistory**, **Log** (separate logs DB), **MetadataFile/SubtitleFile/OtherExtraFile** (`MetadataFiles`/`SubtitleFiles`/`ExtraFiles`) | | |
| **Provider definitions** (`Indexers`, `DownloadClients`, `ImportLists`, `Notifications`, `Metadata`) plus `*Status` tables | ThingiProvider pattern: `Implementation` + `ConfigContract` + JSON `Settings` | `Core/ThingiProvider/` |

**Relationships:** Movie 1–1 MovieMetadata; MovieMetadata 1–N AlternativeTitle, MovieTranslation, Credit; Movie 1–0..1 MovieFile (the `MovieFileId` back-pointer). Collection–MovieMetadata goes through `CollectionTmdbId`. Movie N–1 QualityProfile. Tags are int sets stored as JSON across most entities.

**Migrations:** `Core/Datastore/Migration/` has **140 migration files** numbered 000–242 (193 and 213 are missing), plus `Framework/`. `002_103_sonarr_squashed.cs` is the Sonarr-inherited squash. `119_add_youtube_trailer_id_table .cs` has a stray space in its filename. Logs live in a separate DB (`LogDatabase.cs`, `logs.db`).

---

## 3. Feature areas

One entry per top-level folder in `Core/`:

| Folder | What it does |
|---|---|
| **Analytics** | `AnalyticsService` reports whether the install is active (used in the update check `active=` param). |
| **Annotations** | `FieldDefinitionAttribute` etc. Drives dynamic provider settings forms (UI schema) via `Radarr.Http/ClientSchema`. |
| **Authentication** | `User` (username, hashed password with salt; mig 215), `UserService`, `AuthenticationType` (None, Basic [Obsolete], Forms, External), `AuthenticationRequiredType` (Enabled, DisabledForLocalAddresses). |
| **AutoTagging** | Rules that add or remove tags on movies. Specs: Genre, Keyword, Monitored, OriginalLanguage, QualityProfile, RootFolder, Runtime, Status, Studio, Tag, Year. |
| **Backup** | Zip backups of config.xml + DB (`MakeDatabaseBackup` uses SQLite online backup), scheduled/manual/update types, retention, restore. |
| **Blocklisting** | Blocklisted releases (by title/infohash/indexer), `ClearBlocklistCommand`. |
| **Configuration** | `ConfigService` (DB key/value), `ConfigFileProvider` (config.xml + env/`AppOptions`), `DeploymentInfoProvider` (reads `package_info` for PackageAuthor/UpdateMechanism/global message). |
| **CustomFilters** | Saved UI filters (JSON) per page. |
| **CustomFormats** | CF engine. Specs: Edition, IndexerFlag, Language, QualityModifier, ReleaseGroup, ReleaseTitle (regex), Resolution, Size, Source, Year. `CustomFormatCalculationService` scores releases and files. |
| **Datastore** | Dapper DB layer, lazy loading, converters, SQLite/Postgres builders, migrations, DB restore. |
| **DecisionEngine** | Release acceptance and ranking (see below). |
| **DiskSpace** | Free-space reporting for root folders and mounts. |
| **Download** | Clients, tracked downloads, completed/failed handling, pending/delay queue, aggregation (language). |
| **Exceptions** | Shared exception types. |
| **Extras** | Metadata consumers, subtitles, and "other" extra files. Importing and renaming alongside movie files. |
| **HealthCheck** | 34 checks plus `ServerSideNotificationService` (pulls messages from `radarr.servarr.com/v1/notification`). |
| **History** | Movie history events. |
| **Housekeeping** | 38 daily cleanup tasks. |
| **Http** | `HttpProxySettingsProvider`, `TorcacheHttpInterceptor` (dead site), `CloudFlare/` captcha interceptor. |
| **ImportLists** | Lists that add movies automatically, plus exclusions, cached list movies, and library cleaning on sync. |
| **Indexers** | Indexer implementations, RSS parsing, RSS sync, indexer status/backoff, seed config. |
| **IndexerSearch** | Movie search (automatic, missing, cutoff-unmet) and interactive release search. |
| **Instrumentation** | DB log target, log reconfiguration, Sentry reconfig, log cleanup commands. |
| **Jobs** | `TaskManager` (scheduled tasks), `Scheduler`. |
| **Languages** | `Language` (60 static languages incl. `Unknown=0`, `Any=-1`, `Original=-2`), field converters. |
| **Lifecycle** | Startup/shutdown/restart events and commands. |
| **Localization** | `LocalizationService`, 45 JSON locale files in `Core/Localization/Core/`. |
| **MediaCover** | Downloads, caches and resizes posters/fanart via ImageSharp. Also a remote-URL image proxy (`MediaCoverProxy`). |
| **MediaFiles** | Disk scan, import pipeline, renaming, moving, recycle bin, media info (FFprobe), script import, torrent file info. |
| **Messaging** | Command queue (DB-persisted, priorities, dedup), `EventAggregator` (sync + async `IHandle`/`IHandleAsync`). |
| **MetadataSource** | `SkyHookProxy` calls `api.radarr.video/v1` (Radarr's TMDb proxy) for movie, collection, bulk, IMDb lookup, search, changed movies, trending/popular. |
| **Movies** | Movie CRUD, add/refresh/move, collections, credits, alt titles, translations, availability. |
| **MovieStats** | Aggregated size/file stats per movie. |
| **Notifications** | 25 notification providers (listed in §4). |
| **Organizer** | `FileNameBuilder` (tokens), `NamingConfig`, validation, samples. |
| **Parser** | Release title parser, quality/language/release-group parsers, `ParsingService` (maps a release to a Movie). |
| **Profiles** | Quality, delay and release profiles. |
| **ProgressMessaging** | Pushes command progress to the UI through NLog target plus SignalR. |
| **Qualities** | Quality definitions, sizes, revision (proper/repack/real). |
| **Queue** | Merges tracked downloads and pending releases into the UI queue. |
| **RemotePathMappings** | Maps download-client host paths to local paths. |
| **RootFolders** | Root folders, free space, unmapped folders (for library import). |
| **Security** | TLS certificate validation policy (`CertificateValidationType`: Enabled / DisabledForLocalAddresses / Disabled). |
| **Tags** | Tags plus `TagDetails` usage map. |
| **ThingiProvider** | Generic pluggable-provider framework (definitions, factory, status/backoff, test, schema). Used by Indexers, DownloadClients, ImportLists, Notifications, Metadata. |
| **Update** | Update check (`radarr.servarr.com/v1/update/{branch}`), install via separate `Radarr.Update` process, update history. |
| **Validation** | FluentValidation helpers, path validators (root folder/ancestor/system folder/startup folder/recycle bin/mapped network drive). |

### Deeper dives

**Parsing** (`Core/Parser/`)
- `Parser.ParseMovieTitle` runs an ordered list of `ReportMovieTitleRegex` patterns (`Parser.cs:27-70`). Anime-style `[SubGroup] Title (Year) [hash]` patterns come first, then German/TrueFrench, then edition+year, then year-based patterns, then a year-first fallback. The year must be 1800–2099 (`(1(8|9)|20)\d{2}`).
- **Editions**: `EditionRegex` (`Parser.cs:18`) covers Director's/Collector's/Theatrical/Ultimate/Extended/Despecialized/Special/Rouge/Final/Assembly/Imperial/Diamond/Signature/Hunter/Rekall Cut/Edition/Version, N-th Anniversary, Uncensored, Remastered, Unrated, Uncut, Open Matte, IMAX, Fan Edit, Restored, 2in1/3in1/4in1. Stored on `ParsedMovieInfo.Edition` and `MovieFile.Edition`, and used by the `{Edition Tags}` token and `EditionSpecification`.
- Also parses: IMDb id (`tt\d{7,8}`) and TMDb id (`tmdb(id)?-\d+`) from titles, hardcoded subs (`HardcodedSubsRegex`), AKA and slash alternative titles (`MovieTitles` list), reversed titles, hashed/obfuscated release rejection, German umlaut replacement, Roman numeral normalization (`Parser/RomanNumerals/`), and article/"part/special/edition" stripping for clean titles.
- `QualityParser.cs` (789 lines): source/resolution/modifier/revision (PROPER/REPACK/REAL/version) detection. `LanguageParser.cs` (596 lines): language tokens, "MULTI", subtitle file language parsing. `ReleaseGroupParser.cs`.
- `ParsingService.GetMovie` match order: IMDb id → TMDb id → title+year → title only, with the year checked against `Year` or `SecondaryYear`. In search context it matches against search-criteria titles. Title lookup covers `CleanTitle`, `CleanOriginalTitle`, `AlternativeTitles.CleanTitle` and `MovieTranslations.CleanTitle` (`Core/Movies/MovieRepository.cs:134-210`).

**Decision engine** (`Core/DecisionEngine/`)
- General specs: AcceptableSize, AlreadyImported, BlockedIndexer, Blocklist, CustomFormatAllowedByProfile, FreeSpace, HardcodeSubs, **Language** (profile language / Original / Any), MaximumSize, MinimumAge, NotSample, Protocol, QualityAllowedByProfile, Queue, RawDisk, ReleaseRestrictions, Repack, RequiredIndexerFlags, Retention, TorrentSeeding, Upgradable, UpgradeAllowed, UpgradeDisk.
- RSS-only: **Availability** (`Movie.IsAvailable(AvailabilityDelay)`), Delay, History, IndexerTag, MonitoredMovie, Pending, Proper.
- Search-only: `MovieSpecification`.
- `DownloadDecisionPriorizationService` / `DownloadDecisionComparer` rank by quality, CF score, protocol, indexer priority, seeders, age and size.

**Download handling** (`Core/Download/`)
- `DownloadService` grabs a release. `ProcessDownloadDecisions` handles grab vs pending (delay profiles) plus a per-indexer download client override (mig 205).
- `TrackedDownloads/`: `DownloadMonitoringService` polls clients every `CheckForFinishedDownloadInterval` (≥1 min) through `RefreshMonitoredDownloadsCommand`. `TrackedDownloadService` maps client items to movies via history (`DownloadId`).
- `CompletedDownloadService` handles import-on-complete. `FailedDownloadService` blocklists and redownloads (`AutoRedownloadFailed`, `...FromInteractiveSearch`). `IgnoredDownloadService`.
- Completed download handling is per client (`RemoveCompletedDownloads`, `RemoveFailedDownloads`; mig 200).
- `NzbValidationService`, `TorrentClientBase` (magnet ↔ torrent, `TorrentFileInfoReader`), `UsenetClientBase`.
- `ManualInteractionRequiredEvent` fires when an import can't be matched automatically.

**Import** (`Core/MediaFiles/`, `MediaFiles/MovieImport/`)
- `DownloadedMovieImportService` and `ImportDecisionMaker`, with aggregators (Edition, Language, Movie, Quality, ReleaseGroup, ReleaseInfo, SubtitleInfo, plus augmenters) and specs: AlreadyImported, FreeSpace, GrabbedReleaseQuality, HasAudioTrack, MatchesFolder, MatchesGrab, NotMultiPart, NotSample, NotUnpacking, Upgrade.
- `ImportApprovedMovie` decides `ImportMode` (Auto/Move/Copy). `TransferMode` covers Move/Copy/HardLink/HardLinkOrCopy; reflinks go through `src/NzbDrone.Mono/Disk/RefLinkCreator.cs`.
- `ScriptImportDecider` (`UseScriptImport`/`ScriptImportPath`) lets a user script perform the transfer.
- `DetectSample` uses runtime via FFprobe. `UpgradeMediaFileService` sends replaced files to the recycle bin.
- Manual import lives in `MovieImport/Manual/` (API `manualimport`). Library import of existing folders uses `movie/import` plus `RootFolder.UnmappedFolders`.

**RSS sync** (`Core/Indexers/RssSyncService.cs`, `FetchAndParseRssService.cs`)
- Runs every `RssSyncInterval` (default 30, min 10, 0 disables). Fetches all RSS-enabled indexers, runs `DownloadDecisionMaker`, and processes the decisions.
- Indexer status/backoff lives in `IndexerStatusService` (ThingiProvider `Status/`).

**Search** (`Core/IndexerSearch/`)
- `ReleaseSearchService` builds `MovieSearchCriteria` (titles from metadata, alt titles, translations; year; IMDb/TMDb ids). Commands: `MoviesSearchCommand`, `MissingMoviesSearchCommand`, `CutoffUnmetMoviesSearchCommand`.
- `Movie.LastSearchTime` was added in mig 234.

**Renaming** (`Core/Organizer/FileNameBuilder.cs`)
- Tokens: `{Movie Title}`, `{Movie TitleThe}`, `{Movie CleanTitle}`, `{Movie CleanTitleThe}`, `{Movie TitleFirstCharacter}`, `{Movie OriginalTitle}`, `{Movie CleanOriginalTitle}`, `{Movie Collection}`, `{Movie CollectionThe}`, `{Movie CleanCollectionThe}`, `{Movie Certification}`, `{Release Year}`, `{ImdbId}`, `{TmdbId}`, `{Edition Tags}`, `{Quality Full/Title/Proper/Real}`, `{MediaInfo Simple/Full/Video/VideoCodec/VideoBitDepth/Audio/AudioCodec/AudioChannels/AudioLanguages(All)/SubtitleLanguages(All)/3D}` (dynamic-range tokens are also referenced), `{Custom Format}`, `{Custom Formats}`, `{Release Group}`, `{Original Title}`, `{Original Filename}`.
- There are also language-suffixed title tokens (e.g. `{Movie Title:DE}`) via translations.
- `NamingConfig` has `RenameMovies`, `ReplaceIllegalCharacters`, `ColonReplacementFormat`, `StandardMovieFormat` and `MovieFolderFormat`. File-property tokens are **deprecated in the folder format** (`Organizer/FileNameValidation.cs:12`, `HealthCheck/Checks/NamingConfigCheck.cs`).

**Metadata** (`Core/Extras/Metadata/`)
- Consumers write NFO and image files: Kodi/Emby (Xbmc), Emby (Legacy)/MediaBrowser, Roksbox, WDTV, Kometa (deprecated).
- Subtitles (`Extras/Subtitles/`, languages parsed from filenames, mig 238) and other extras (`ExtraFileExtensions`) are imported and renamed with the movie.

**Collections** (`Core/Movies/Collections/`, `RefreshCollectionService.cs`)
- Collections are auto-created from `MovieMetadata.CollectionTmdbId` on movie refresh, then refreshed daily (`RefreshCollectionsCommand`). Refresh cadence per collection is under 6h skip and over 15d force.
- If `Monitored`, missing movies are auto-added with the collection's profile, root folder, availability and tags, skipping import exclusions (`RefreshCollectionService.cs:126-160`).
- On add, `MonitorTypes.MovieAndCollection` also monitors the collection.
- Health check: `MovieCollectionRootFolderCheck`. Housekeeper: `CleanupOrphanedCollections`.

**Housekeeping** (`Core/Housekeeping/Housekeepers/`, 38)
- Orphan cleanup for AlternativeTitles, Blocklist, Collections, Credits, DownloadClientStatus, ExtraFiles, HistoryItems, ImportListMovies, IndexerStatus, MetadataFiles, MovieFiles, MovieMetadata, MovieMovieFileIds, Movies, MovieTranslations, NotificationStatus, PendingReleases, SubtitleFiles.
- Also: CleanupAbsolutePathMetadataFiles, AdditionalNamingSpecs, AdditionalUsers, CommandQueue, DownloadClientUnavailablePendingReleases, DuplicateMetadataFiles, DuplicateMovieTranslations, QualityProfileFormatItems, TemporaryUpdateFiles, UnusedTags, DeleteBadMediaCovers.
- Status-time fixes: FixFuture{DownloadClient,ImportList,Indexer,Notification,Provider}StatusTimes, FixFutureRunScheduledTasks.
- Others: FixWronglyMatchedMovieFiles, TrimLogDatabase, UpdateCleanTitleForMovies.

**Scheduled tasks**: see §6.

**Notifications**: see §4. `NotificationService` fans events out to providers with per-event toggles. `MediaServerUpdateQueue` batches library refreshes.

**Health checks** (`Core/HealthCheck/Checks/`, 34)
- AllowedHosts, ApiKeyValidation, AppDataLocation, DownloadClient, DownloadClientRemovesCompletedDownloads, DownloadClientRootFolder, DownloadClientSorting, DownloadClientStatus, ImportListRootFolder, ImportListStatus, ImportMechanism, IndexerDownloadClient, **IndexerJackettAll**, IndexerLongTermStatus, IndexerRss, IndexerSearch, IndexerStatus.
- MetadataCheck (Kometa deprecated), Mount (read-only mounts), MovieCollectionRootFolder, NamingConfig (deprecated tokens), NotificationStatus, PackageGlobalMessage, Proxy, **PTPOldSettings**, RecyclingBin, ReleaseBranch, RemotePathMapping, **RemovedMovie** (TMDb-deleted movies), RootFolder, **SlackUrl** (Discord URL in Slack provider), SqliteVersion (Linux, ≥3.9.0), SystemTime (vs servarr cloud), Update.

**Backups** (`Core/Backup/`): `BackupInterval` in days (clamped 1–7, default 7), `BackupRetention`, `BackupFolder`. Restore goes through the API (`system/backup/restore`, upload).

**Updates** (`Core/Update/`, `src/NzbDrone.Update/`)
- `UpdateMechanism`: BuiltIn, Script, External, Apt, Docker. The package can override it via `package_info`.
- Branch is set in config.xml. The updater is a separate exe that backs up and restores, stops the process or service (`TerminateNzbDrone`), replaces files and restarts (`StartNzbDrone`). It detects `AppType` Normal/Console/Service.

**Auth**: see §5.

---

## 4. Integrations (full lists)

### Indexers (`Core/Indexers/`) — 9

| Name (UI) | Class | Protocol | Status |
|---|---|---|---|
| Newznab | `Newznab/Newznab.cs` | Usenet | Active. Presets: DOGnzb, DrunkenSlug, Nzb.su, NZBCat, NZBFinder.ws, NZBgeek, nzbplanet.net, SimplyNZBs, Tabula Rasa, **Usenet Crawler** (preset likely stale) |
| Torznab | `Torznab/Torznab.cs` | Torrent | Active. Jackett preset. Validation rejects Jackett `/all` endpoint |
| Torrent RSS Feed | `TorrentRss/TorrentRssIndexer.cs` | Torrent | Active (generic RSS, auto-detects parser settings via `TorrentRssSettingsDetector`) |
| HDBits | `HDBits/` | Torrent | Active (private tracker API) |
| PassThePopcorn | `PassThePopcorn/` | Torrent | Active. `PTPOldSettingsCheck` flags legacy settings. PTP-specific `IndexerFlags` (PTP_Golden/PTP_Approved) |
| IP Torrents | `IPTorrents/` | Torrent | Active (RSS only) |
| FileList.io | `FileList/` | Torrent | Active |
| Nyaa | `Nyaa/` | Torrent | Active but **anime-focused, odd in a movie app** |
| TorrentPotato | `TorrentPotato/` | Torrent | **Obsolete**: CouchPotato-era JSON protocol, effectively dead |

Removed historically (migrations): BitMeTV (111), TorrentLeech (112), BroadcastheNet (113), Fanzub (114), Wombles (127/130), KickassTorrents (128/130), AwesomeHD (190/191), omgwtfnzbs (217), Rarbg (222). Obsolete `IndexerFlags.AHD_Internal`/`AHD_UserRelease` are still in the enum (`Core/Parser/Model/IndexerFlags.cs:39,58`).

In practice most users feed indexers through Prowlarr, which syncs Newznab/Torznab definitions.

### Download clients (`Core/Download/Clients/`) — 19 implementations

| Name | Class | Protocol | Status |
|---|---|---|---|
| SABnzbd | `Sabnzbd/` | Usenet | Active |
| NZBGet | `Nzbget/` | Usenet | Active (nzbget-ng fork). Still has `DownloadNzbLegacy12/13` paths for NZBGet <16 (`NzbgetProxy.cs:72-106`), **legacy** |
| NZBVortex | `NzbVortex/` | Usenet | macOS-only, **low relevance** |
| Pneumatic | `Pneumatic/` | Usenet | **Obsolete** (Kodi add-on strm-based, dead) |
| Usenet Blackhole | `Blackhole/UsenetBlackhole.cs` | Usenet | Active (watch folder) |
| Download Station (Usenet) | `DownloadStation/UsenetDownloadStation.cs` | Usenet | Synology, active |
| qBittorrent | `QBittorrent/` | Torrent | Active. `QBittorrentProxyV1` (API v1, qBit <3.2/4.1) is **legacy** alongside V2 |
| Deluge | `Deluge/` | Torrent | Active |
| Transmission | `Transmission/` | Torrent | Active |
| Vuze | `Vuze/Vuze.cs` (Transmission RPC subclass) | Torrent | **Obsolete** (Vuze discontinued; BiglyBT fork) |
| rTorrent | `rTorrent/` (XML-RPC) | Torrent | Active |
| uTorrent | `uTorrent/` | Torrent | **Legacy** (WebUI token API, abandoned by most) |
| Flood | `Flood/` | Torrent | Active |
| Aria2 | `Aria2/` | Torrent | Active (niche) |
| Hadouken | `Hadouken/` | Torrent | **Obsolete** (project dead ~2017) |
| Download Station (Torrent) | `DownloadStation/TorrentDownloadStation.cs` | Torrent | Synology, active (DS v1 and DS2 task APIs) |
| Freebox Download | `FreeboxDownload/` | Torrent | Niche (French ISP box) |
| RQBit | `RQBit/` | Torrent | New (2024+) |
| Torrent Blackhole | `Blackhole/TorrentBlackhole.cs` | Torrent | Active |

### Import lists (`Core/ImportLists/`) — 20 implementations

| Name | Class | Min refresh | Notes / Status |
|---|---|---|---|
| TMDb List | `TMDb/List/TMDbListImport.cs` | 12h | Direct `api.themoviedb.org` with embedded bearer token (`NzbDrone.Common/Cloud/RadarrCloudRequestBuilder.cs:19`) |
| TMDb Popular | `TMDb/Popular/` | 12h | Filters: `TMDbFilterSettings` |
| TMDb User | `TMDb/User/` | 12h | OAuth |
| TMDb Person | `TMDb/Person/` | 12h | cast/crew filters |
| TMDb Company | `TMDb/Company/` | 12h | |
| TMDb Keyword | `TMDb/Keyword/` | 12h | |
| Trakt List | `Trakt/List/` | 12h | OAuth via servarr auth relay |
| Trakt User | `Trakt/User/` | 12h | watchlist/watched/collection |
| Trakt Popular List | `Trakt/Popular/` | 12h | `TraktPopularListType` has `[Obsolete]` members (`TraktPopularListType.cs:26,39`) |
| Simkl User Watchlist | `Simkl/User/` | 6h | OAuth relay `auth.servarr.com/v1/simkl/*` |
| Plex Watchlist | `Plex/PlexImport.cs` | 6h | plex.tv OAuth (`Notifications/Plex/PlexTv/`) |
| Plex Watchlist RSS | `Rss/Plex/PlexRssImport.cs` | 6h | Duplicates Plex Watchlist via RSS |
| IMDb Lists | `RadarrList2/IMDb/IMDbListImport.cs` | 12h | Via `api.radarr.video/v1/list/imdb/{id}`; user lists (`ur…`) use IMDb export |
| StevenLu List | `RadarrList2/StevenLu/StevenLu2Import.cs` | 12h | Via `api.radarr.video/v1/list/stevenlu` |
| StevenLu Custom | `StevenLu/StevenLuImport.cs` | 24h | **Redundant** with StevenLu List (legacy direct URL; mig 241 updated URL) |
| Custom Lists | `RadarrList/RadarrListImport.cs` | 12h | Generic JSON list URL (legacy Radarr list-server format) |
| RSS List | `RSSImport/RSSImport.cs` | 12h | Generic RSS with IMDb ids in items. **Overlaps** `Rss/RssImportBase` |
| Radarr | `Radarr/RadarrImport.cs` | 15m | Pulls another Radarr instance via `/api/v3/movie`, profile/tag/rootfolder filters |
| CouchPotato | `CouchPotato/CouchPotatoImport.cs` | 30m | **Obsolete** (CouchPotato dead since ~2020; a migration path from the pre-Radarr era) |

Supporting pieces: `ImportExclusions/` (by TmdbId; API `exclusions`) and `ImportListMovies/` (cached list results that power the "Discover" UI and recommendations). `ImportListSyncService.CleanLibrary` follows `ListSyncLevel` = `CleanLibraryLevel` {Disabled, LogOnly, KeepAndUnmonitor, RemoveAndKeep, RemoveAndDelete}. Each list's `Monitor` = `MonitorTypes` {MovieOnly, MovieAndCollection, None}.

### Notifications / Connect (`Core/Notifications/`) — 25

| Name | Class | Status / notes |
|---|---|---|
| Apprise | `Apprise/` | Active (meta-notifier) |
| Custom Script | `CustomScript/` | Active (env-var contract, §5) |
| Discord | `Discord/` | Active |
| Email | `Email/` (MailKit) | Active (multiple addresses mig 174, encryption mig 235) |
| Emby / Jellyfin | `MediaBrowser/` | Active (library refresh) |
| Gotify | `Gotify/` | Active |
| Join | `Join/` | **Low-activity/legacy** (Joaoapps) |
| Kodi | `Xbmc/` (JSON-RPC) | Active |
| Mailgun | `Mailgun/` | Active |
| Notifiarr | `Notifiarr/` | Active (servarr-adjacent) |
| ntfy.sh | `Ntfy/` | Active |
| Plex Media Server | `Plex/Server/` | Active. Still carries `PlexSectionLegacy`/`PlexMediaContainerLegacy`/`PlexPreferencesLegacy` for old PMS versions (`Plex/Server/PlexSection.cs:40-55`, `PlexServerProxy.cs:43`) |
| Prowl | `Prowl/` | **Legacy** (iOS Growl-era) |
| Pushbullet | `PushBullet/` | Declining |
| Pushcut | `Pushcut/` | Active (iOS) |
| Pushover | `Pushover/` | Active |
| Pushsafer | `Pushsafer/` | Active |
| SendGrid | `SendGrid/` | Active |
| Signal | `Signal/` (signal-cli REST) | Active |
| Simplepush | `Simplepush/` | Niche |
| Slack | `Slack/` | Active. `SlackUrlCheck` detects Discord URLs (mig 201 migrated these) |
| Synology Indexer | `Synology/` (runs `/usr/syno/bin/synoindex`) | Platform-specific, **legacy-ish** |
| Telegram | `Telegram/` | Active |
| Trakt | `Trakt/` | Active (adds to Trakt collection) |
| Twitter | `Twitter/` (OAuth1, `api.twitter.com`) | **Obsolete/broken in practice** (X API v1.1 paywalled) |
| Webhook | `Webhook/` | Active (JSON payloads, §5) |

Removed historically: Growl (mig 157), Plex Home Theater (158), Plex Client, NMA, etc. Events are OnGrab, OnDownload, OnUpgrade, OnRename, OnMovieAdded, OnMovieDelete, OnMovieFileDelete(+ForUpgrade), OnHealthIssue(+IncludeHealthWarnings), OnHealthRestored, OnApplicationUpdate and OnManualInteractionRequired (`NotificationDefinition.cs`).

### Metadata consumers (`Core/Extras/Metadata/Consumers/`) — 5
| Name | Status |
|---|---|
| Kodi (XBMC) / Emby — `Xbmc/` | Active (NFO + images, `XbmcNfoDetector`) |
| Emby (Legacy) — `MediaBrowser/` | **Deprecated** (in-UI warning `MetadataMediaBrowserDeprecated`, for Emby < v6) |
| Kometa — `Kometa/` | **Deprecated** (settings `Deprecated=true`, `MetadataCheck` health warning) |
| Roksbox — `Roksbox/` | **Obsolete** (Roku channel, dead; mig 223 cleans images) |
| WDTV — `Wdtv/` | **Obsolete** (WD TV players discontinued ~2016) |

### Metadata sources
- **SkyHook / RadarrAPI** (`Core/MetadataSource/SkyHook/SkyHookProxy.cs`) calls `https://api.radarr.video/v1/{route}`. Routes: `movie/{tmdbId}`, `movie/bulk`, `movie/imdb/{id}`, `movie/collection/{id}`, `movie/changed?since=`, `search?q=&year=`, `list/tmdb/trending`, `list/tmdb/popular`, `list/imdb/{id}`, `list/stevenlu`. Search accepts `imdb:`, `tmdb:`, IMDb and TMDb URLs, or a parsed title plus year. The name "SkyHook" is inherited from Sonarr (Sonarr's TVDB proxy).
- **TMDb direct** (import lists only), `api.themoviedb.org` with a bearer token.
- **Servarr services**: `radarr.servarr.com/v1/` (update, server-side notifications, time/proxy checks), `sentry.servarr.com`, `auth.servarr.com` (OAuth relay for Trakt/Simkl/TMDb/Plex).

### Media servers (update/refresh)
Plex Media Server, Emby/Jellyfin, Kodi (as notifications), Synology Indexer, and Trakt (collection sync). The Plex Watchlist is an import list.

### Grep results: "Obsolete / Deprecated / legacy"
- `Core/Authentication/AuthenticationType.cs:8`: `Basic` marked `[Obsolete("Use Forms authentication instead")]`.
- `Core/Parser/Model/IndexerFlags.cs:39,58`: AHD flags obsolete.
- `Core/ImportLists/Trakt/Popular/TraktPopularListType.cs:26,39`: obsolete list types.
- `src/Radarr.Api.V3/MovieFiles/MovieFileController.cs:108`: non-bulk endpoint obsolete.
- `src/Radarr.Api.V3/ImportLists/ImportListExclusionController.cs:35`: deprecated endpoint.
- `src/Radarr.Api.V3/Queue/QueueResource.cs:44,47`: `Size`/`Timeleft` superseded by `SizeLeft`/`TimeLeft`.
- `Core/Download/TrackedDownloads/DownloadMonitoringService.cs:164`: deprecated `CheckForFinishedDownload` command kept for third-party apps.
- `Core/MetadataSource/SkyHook/Resource/MovieResource.cs:15`: deprecated rating field.
- `Core/History/History.cs:41,45`: Sonarr-derived event ids commented out.

---

## 5. External APIs / interfaces

**HTTP API**: `/api/v3/*` (`VersionedApiControllerAttribute`), plus `/api` info (`Radarr.Http/ApiInfoController.cs`), `/ping`, `/login` + `/logout`, `/initialize.json` (UI bootstrap), `/feed/v3/calendar/Radarr.ics` (iCal) and static UI. There is no v1/v2 API anymore. Controllers (`src/Radarr.Api.V3/`), grouped:

- **Movies**: `movie` (CRUD, bulk `movie/editor`, `movie/{id}/folder`, `movie/lookup` (+`/tmdb`, `/imdb`), `movie/import` (bulk add existing folders), `alttitle`, `credit`, `rename` (preview), `moviefile` (+bulk), `extrafile`, `mediacover`), `collection`.
- **Wanted / Calendar / Queue**: `wanted/missing`, `wanted/cutoff`, `calendar`, `queue` (+ `queue/details`, `queue/status`, queue actions grab/bulk remove), `history` (+since/movie), `blocklist`.
- **Search / Release**: `release` (interactive search + grab), `release/push` (external release push, used by Prowlarr/autobrr etc.), `parse`, `manualimport`.
- **Providers** (via `ProviderControllerBase`: `schema`, `test`, `testall`, `action/{name}`, bulk PUT/DELETE): `indexer`, `downloadclient`, `importlist`, `notification`, `metadata`. Also `indexerflag`, `importlist/movie` (discover), `exclusions`.
- **Profiles / Formats**: `qualityprofile` (+ `/schema`), `qualitydefinition` (+limits), `customformat` (+schema, bulk), `delayprofile`, `releaseprofile`, `language`, `autotagging`, `customfilter`, `tag` (+ `tag/detail`), `rootfolder`, `remotepathmapping`.
- **Config**: `config/host`, `config/ui`, `config/naming` (+examples), `config/mediamanagement`, `config/indexer`, `config/downloadclient`, `config/importlist`, `config/metadata`.
- **System**: `system/status`, `system/routes`, `system/routes/duplicate`, `system/restart`, `system/shutdown`, `system/backup` (+restore/upload), `system/task`, `health`, `diskspace`, `update`, `log`, `log/file`, `log/file/update`, `command` (POST to run commands), `filesystem` (browse, type, mediafiles), `localization` (+language).

**SignalR** (`src/NzbDrone.SignalR/MessageHub.cs`, hub `/signalr/messages`, auth policy "SignalR" with `access_token` query or `X-Api-Key`). Resource-change broadcasts come from `RestControllerWithSignalR` (`src/Radarr.Http/REST/RestControllerWithSignalR.cs`). Message names handled by the UI (`frontend/src/Components/SignalRConnector.js:151-325`): `calendar`, `command`, `moviefile`, `downloadclient`, `health`, `importlist`, `indexer`, `metadata`, `notification`, `movie`, `collection`, `qualitydefinition`, `queue`, `queue/details`, `queue/status`, `version`, `wanted/cutoff`, `wanted/missing`, `system/task`, `rootfolder`, `tag`. Payload: `{action: updated|deleted|sync, resource}`.

**Webhook** (`Core/Notifications/Webhook/`): event types `Test, Grab, Download, Rename, MovieDelete, MovieFileDelete, Health, ApplicationUpdate, MovieAdded, HealthRestored, ManualInteractionRequired` (`WebhookEventType.cs`). Payload types include `WebhookMovie`, `WebhookRemoteMovie`, `WebhookMovieFile` (+MediaInfo), `WebhookRelease`, `WebhookGrabbedRelease`, `WebhookCustomFormatInfo` and `WebhookDownloadClientItem`. Methods are POST/PUT. Notifiarr reuses `WebhookBase`.

**Custom script env vars** (`Core/Notifications/CustomScript/CustomScript.cs`), all prefixed `Radarr_`:
- General: `EventType`, `InstanceName`, `ApplicationUrl`.
- `Movie_*`: `Id, Title, Year, Path, ImdbId, TmdbId, OriginalLanguage, Genres, Tags, In_Cinemas_Date, Physical_Release_Date, Overview, Folder_Size, DeletedFiles, AddMethod`.
- `Release_*`: `Title, Indexer, Size, Quality, QualityVersion, ReleaseGroup, CustomFormat, CustomFormatScore`, plus `IndexerFlags`.
- `Download_*`: `Client, Client_Type, Id, Size, Title`.
- `MovieFile_*`: `Id, Ids, Path, Paths, RelativePath, RelativePaths, PreviousPaths, PreviousRelativePaths, Quality, QualityVersion, ReleaseGroup, SceneName, Size, SourcePath, SourceFolder, CustomFormat, CustomFormatScore, DeleteReason`, and `MediaInfo_{AudioChannels, AudioCodec, AudioLanguages, Languages, Height, Width, Subtitles, VideoCodec, VideoDynamicRangeType}`.
- Delete-related: `IsUpgrade`, `DeletedPaths`, `DeletedRelativePaths`, `DeletedRecycleBinPaths`, `DeletedDateAdded`.
- `Health_Issue_{Level,Message,Type,Wiki}`, `Health_Restored_{...}`, `Update_{Message,NewVersion,PreviousVersion}`.

**Auth** (`src/Radarr.Http/Authentication/`)
- Schemes: `None`, `External` (reverse-proxy auth, no-op handler), `Forms` (cookie login page), `Basic` (obsolete, maps to Forms), and API key (`X-Api-Key` header, `apikey` query, or `Authorization: Bearer`).
- `AuthenticationRequired`: Enabled / DisabledForLocalAddresses (with `TrustCgnatIpAddresses`).
- `UiAuthorizationHandler` decides local-address bypass and treats `X-Forwarded-For` as non-local.
- Other controls: `AllowedHosts` host filtering, `TrustedNetworks`, forwarded-headers config (`NzbDrone.Host/ForwardedHeadersConfigurator.cs`), and a single-user `Users` table.

**Import-list OAuth relay**: `auth.servarr.com` (Trakt, Simkl, Plex, TMDb user).

---

## 6. Background jobs / commands

**Scheduled tasks** (`Core/Jobs/TaskManager.cs:66-135`)

| Command | Interval |
|---|---|
| `MessagingCleanupCommand` | 5 min |
| `ApplicationCheckUpdateCommand` | 6 h |
| `CheckHealthCommand` | 6 h (plus event-driven) |
| `RefreshMovieCommand` (all movies, `ShouldRefreshMovie` heuristics; uses `movie/changed` if last run < 14 days) | 24 h |
| `HousekeepingCommand` | 24 h |
| `CleanUpRecycleBinCommand` | 24 h |
| `RefreshCollectionsCommand` | 24 h |
| `BackupCommand` | `BackupInterval` days (1–7, default 7) |
| `RssSyncCommand` | `RssSyncInterval` min (default 30, min 10, 0 = off) |
| `ImportListSyncCommand` | 5 min tick. Each list is gated by its `MinRefreshInterval` (15 m–24 h, see §4) |
| `RefreshMonitoredDownloadsCommand` | `CheckForFinishedDownloadInterval` (default 1 min) |

`ShouldRefreshMovie` (`Core/Movies/ShouldRefreshMovie.cs`) always refreshes if the last sync was over 180 days ago and never if it was under 12 hours ago. Recently released movies (≤30 days since physical release) get frequent refreshes.

**All command types**: ApplicationCheckUpdate, ApplicationUpdate, Backup, BulkMoveMovie, CheckForFinishedDownload (deprecated alias), CheckHealth, CleanUpRecycleBin, ClearBlocklist, ClearLog, CutoffUnmetMoviesSearch, DeleteLogFiles, DeleteUpdateLogFiles, DownloadedMoviesScan (external "drone factory"/script trigger), Housekeeping, ImportListSync, ManualImport, MessagingCleanup, MissingMoviesSearch, MoveMovie, MoviesSearch, ProcessMonitoredDownloads, RefreshCollections, RefreshDownloads (RefreshMonitoredDownloads), RefreshMovie, RenameFiles, RenameMovie, RenameMovieFolder, RescanMovie, ResetApiKey, ResetQualityDefinitions, Restart, RssSync, Shutdown, Test, Unknown.

The command queue is persisted in the `Commands` table (`Core/Messaging/Commands/`), with priorities, exclusivity, dedup (`CommandEqualityComparer`), and the result stored (mig 219).

---

## 7. Configuration surface

**config.xml** (`Core/Configuration/ConfigFileProvider.cs`; overridable via env `RADARR__<SECTION>__<KEY>` through `NzbDrone.Common/Options/{App,Auth,Log,Server,Update}Options.cs`):
- Server: BindAddress, Port, SslPort, EnableSsl, SslCertPath, SslCertPassword, UrlBase, LaunchBrowser, InstanceName, AllowedHosts, TrustedNetworks, TrustCgnatIpAddresses.
- Auth: ApiKey, AuthenticationMethod, AuthenticationRequired.
- Logging: LogLevel, ConsoleLogLevel, ConsoleLogFormat (Standard/CLEF), LogSql, LogRotate, LogSizeLimit, LogDbEnabled, SyslogServer, SyslogPort, SyslogLevel.
- Updates / telemetry: AnalyticsEnabled, FilterSentryEvents, Branch, UpdateAutomatically, UpdateMechanism, UpdateScriptPath.
- UI: Theme, UiFolder.
- Postgres: Host, Port, User, Password, MainDb, LogDb, and full connection strings.

**DB Config table** (`Core/Configuration/ConfigService.cs`), grouped as in UI:
- *Media Management*: AutoUnmonitorPreviouslyDownloadedMovies, RecycleBin, RecycleBinCleanupDays, CreateEmptyMovieFolders, DeleteEmptyFolders, FileDate, SkipFreeSpaceCheckWhenImporting, MinimumFreeSpaceWhenImporting, CopyUsingHardlinks, EnableMediaInfo, UseScriptImport, ScriptImportPath, ImportExtraFiles, ExtraFileExtensions, AutoRenameFolders, RescanAfterRefresh, SetPermissionsLinux, ChmodFolder, ChownGroup, DownloadPropersAndRepacks.
- *Indexers*: MinimumAge, Retention, MaximumSize, RssSyncInterval, PreferIndexerFlags, AvailabilityDelay, AllowHardcodedSubs, WhitelistedHardcodedSubs.
- *Download clients*: EnableCompletedDownloadHandling, AutoRedownloadFailed, AutoRedownloadFailedFromInteractiveSearch, CheckForFinishedDownloadInterval, DownloadClientHistoryLimit, DownloadClientWorkingFolders.
- *Import lists*: ListSyncLevel, ImportExclusions.
- *Metadata*: CertificationCountry, CleanupMetadataImages.
- *UI*: FirstDayOfWeek, CalendarWeekColumnHeader, MovieRuntimeFormat, ShortDateFormat, LongDateFormat, TimeFormat, ShowRelativeDates, EnableColorImpairedMode, MovieInfoLanguage (metadata language), UILanguage.
- *General/host*: ProxyEnabled, ProxyType, ProxyHostname, ProxyPort, ProxyUsername, ProxyPassword, ProxyBypassFilter, ProxyBypassLocalAddresses, CertificateValidation, ApplicationUrl, BackupFolder, BackupInterval, BackupRetention.
- *Internal secrets*: RijndaelPassphrase/Salt, HmacPassphrase/Salt, PlexClientIdentifier.

Other settings live in their own entities: NamingConfig, quality definitions, profiles, custom formats, delay/release profiles, auto tagging, tags, root folders, remote path mappings, and provider settings (JSON per definition).

---

## 8. Platform / infrastructure concerns

- **Entry points**: `src/NzbDrone.Console` (Radarr.Console.exe, cross-platform), `src/NzbDrone` (Radarr.exe WinForms **tray app**, `SysTray/`, `net8.0-windows`), `src/NzbDrone.Host` (shared bootstrap: `Bootstrap.cs`, `Startup.cs`, `ApplicationModes` = Interactive/Help/InstallService/UninstallService/Service/RegisterUrl), and `src/ServiceHelpers/ServiceInstall`, `ServiceUninstall` (Windows service helper exes).
- **Windows-specific**:
  - Windows Service hosting (`Microsoft.Extensions.Hosting.WindowsServices`, `WindowsServiceHelpers.IsWindowsService()` in `Bootstrap.cs:240`, `RestartableServiceLifetime.cs`), `ServiceProvider.cs` (ServiceController).
  - `NzbDrone.Host/AccessControl/FirewallAdapter.cs` (COM `HNetCfg.FwMgr` to open firewall ports) and `RemoteAccessAdapter.cs`.
  - `RegisterUrl` mode (netsh urlacl, an HttpListener/OWIN-era leftover).
  - `NzbDrone.Windows/Disk/DiskProvider.cs`, `FolderMount.cs`.
  - `LongPathSupport` (`NzbDrone.Common/Disk/LongPathSupport.cs`).
  - Inno Setup installer `distribution/windows/setup/radarr.iss`.
- **Unix-specific (`NzbDrone.Mono`, name is historical; runs on .NET 8 using Mono.Posix.NETStandard)**: `DiskProvider` (chmod/chown, `SetPermissionsLinux`), `ProcMountProvider` (/proc/mounts), `RefLinkCreator` (btrfs/xfs reflinks), `SymbolicLinkResolver`, OS version adapters (FreeBSD, Issue file, macOS, os-release, **Synology**). There is also a macOS `.app` bundle at `distribution/osx/Radarr.app`.
- **OS matrix**: `Os` = Windows, Linux, Osx, LinuxMusl, Bsd (`NzbDrone.Common/EnvironmentInfo/OsInfo.cs:106`). Docker/Podman detection via `/.dockerenv` etc. There are 62 `IsWindows/IsLinux/IsOsx/...` branch sites in Core/Common/Host.
- **App data migration**: `AppFolderFactory.MigrateAppDataFolder` moves the legacy `nzbdrone.db` / `.config/Radarr` locations and sets `XDG_CONFIG_HOME` ("Tell mono/netcore…" comment at `AppFolderFactory.cs:147`).
- **Self-updater**: `NzbDrone.Update` (separate process): `InstallUpdateService`, `BackupAndRestore`, `BackupAppData`, `DetectApplicationType`, `DetectExistingVersion`, `TerminateNzbDrone`, `StartNzbDrone`. Update mechanisms (BuiltIn/Script/External/Apt/Docker) plus `package_info` override. `UpdateVerification` checks the package signature/hash.
- **Logging**: NLog. Targets: rolling files (`radarr.txt`, `radarr.debug.txt`, `radarr.trace.txt`), console (Standard or CLEF JSON), syslog, DB (`Logs` table in logs DB), SignalR progress target, and Sentry (`NzbDrone.Common/Instrumentation/Sentry/`, with log cleansing `CleanseLogMessage`).
- **DB**: SQLite by default (`radarr.db`, `logs.db`) or Postgres (main + log DBs). `000_database_engine_version_check` enforces minimum engine versions. Postgres support required migration 212 (timestamptz) and 204 (identity columns). SQLite handling includes `CorruptDatabaseException` and `DatabaseRestorationService` (restore from backup on start).
- **Caching**: in-process `ICacheManager`/`Cached<T>` (`NzbDrone.Common/Cache/`), e.g. qBittorrent proxy/version cache and NZBGet version cache. There is no external cache.
- **HTTP client**: custom `NzbDrone.Common/Http/HttpClient.cs` + `ManagedHttpDispatcher` (SocketsHttpHandler), cookie containers per indexer, rate limiting, proxy (`ManagedWebProxyFactory`: HTTP/SOCKS4/SOCKS5), certificate validation service, interceptors (`TorcacheHttpInterceptor`, `CloudFlareHttpInterceptor`), JSON-RPC and XML-RPC request builders (rTorrent, Deluge, Transmission), User-Agent builder. **RestSharp 106** is also referenced (legacy).
- **Web host**: Kestrel, response caching and If-Modified middleware, UrlBase middleware, static SPA serving via mappers (`Radarr.Http/Frontend/Mappers/`: Index, Login, Manifest, BrowserConfig, Favicon, RobotsTxt, MediaCover(+Proxy), Backup, LogFile, UpdateLogFile). The OpenAPI spec is checked in.
- **Media probing**: FFprobe binary shipped via `Servarr.FFprobe` NuGet (mig 199 moved from MediaInfo lib to FFmpeg). `MediaInfoModel` is stored as JSON.
- **Legacy by .NET/Mono history**: NzbDrone.* naming everywhere, the separate `NzbDrone.Mono` project, `RegisterUrl` / firewall COM code, a tray app, service installer exes, dual JSON serializers, RestSharp, `System.Drawing.Common` reference, `Switch.System.IO.UseLegacyPathHandling`.

---

## 9. Interactions with other *arr apps

- **Prowlarr → Radarr**: Radarr has no Prowlarr-specific code. Prowlarr uses Radarr's public API: `indexer` CRUD (creates Newznab/Torznab definitions pointing at Prowlarr's proxy endpoints), `indexer/schema`, `system/status`, `tag`. Prowlarr's "push release" flow uses `release/push`. The Jackett "all" endpoint is explicitly rejected (`Core/Indexers/Torznab/Torznab.cs:153`, `IndexerJackettAllCheck`).
- **Radarr ↔ Radarr**: `Radarr` import list (`Core/ImportLists/Radarr/RadarrV3Proxy.cs`) pulls `/api/v3/movie`, `/qualityprofile`, `/rootfolder` and `/tag` from another instance. This is used for 4K/1080p instance syncing.
- **Sonarr**: no direct integration. The code is shared by fork only (see §11).
- **Profilarr**: no Radarr-side code. Profilarr pushes via API: `customformat` (+bulk), `qualityprofile`, `qualitydefinition`, `config/naming`, `config/mediamanagement`, `language`, `releaseprofile`, `delayprofile`.
- **Overseerr/Jellyseerr/Ombi/Notifiarr/autobrr** (ecosystem): use `movie`, `movie/lookup`, `command` (MoviesSearch), `queue`, `release/push`, and Webhook/Notifiarr providers.
- **CouchPotato**: import list for migration (obsolete). TorrentPotato indexer (obsolete).
- **Servarr shared infra**: `radarr.servarr.com` (updates, server notices), `api.radarr.video` (metadata and list proxy), `auth.servarr.com` (OAuth relay), `sentry.servarr.com`.

---

## 10. Outdated / legacy candidates

| Item | Evidence | Why |
|---|---|---|
| TorrentPotato indexer | `Core/Indexers/TorrentPotato/` | CouchPotato-era protocol, dead |
| CouchPotato import list | `Core/ImportLists/CouchPotato/` | CouchPotato abandoned |
| Nyaa indexer (in a movie app) | `Core/Indexers/Nyaa/` | Anime torrent site; covered by Prowlarr/Torznab |
| IPTorrents / FileList / HDBits / PTP native indexers | `Core/Indexers/*` | Redundant with Prowlarr; keep only Newznab/Torznab + generic RSS in a monolith that includes Prowlarr |
| Hadouken client | `Core/Download/Clients/Hadouken/` | Project dead |
| Vuze client | `Core/Download/Clients/Vuze/Vuze.cs` | Vuze discontinued |
| uTorrent client | `Core/Download/Clients/uTorrent/` | Legacy WebUI, adware, little use |
| Pneumatic client | `Core/Download/Clients/Pneumatic/` | Kodi strm add-on, dead |
| NZBVortex | `Core/Download/Clients/NzbVortex/` | macOS niche, stagnant |
| NZBGet <16 legacy RPC paths | `Core/Download/Clients/Nzbget/NzbgetProxy.cs:72-106` | Ancient versions |
| qBittorrent API v1 | `Core/Download/Clients/QBittorrent/QBittorrentProxyV1.cs` | qBit < 4.1 |
| Twitter notification | `Core/Notifications/Twitter/` | X API v1.1 OAuth1 effectively unavailable |
| Prowl, Join, Pushbullet, Simplepush | `Core/Notifications/{Prowl,Join,PushBullet,Simplepush}/` | Legacy or low-usage push services (Apprise/ntfy cover them) |
| Synology Indexer notification | `Core/Notifications/Synology/` | Platform-specific synoindex |
| Plex legacy response models | `Core/Notifications/Plex/Server/PlexSection.cs:40-55`, `PlexPreferences.cs:19` | Old PMS versions |
| Roksbox, WDTV metadata | `Core/Extras/Metadata/Consumers/{Roksbox,Wdtv}/` | Dead devices |
| Emby (Legacy) metadata | `Core/Extras/Metadata/Consumers/MediaBrowser/MediaBrowserMetadata.cs:29-31` | Self-declared deprecated |
| Kometa metadata | `Core/Extras/Metadata/Consumers/Kometa/`, `HealthCheck/Checks/MetadataCheck.cs:28` | Self-declared deprecated |
| StevenLu Custom (dup of StevenLu List) | `Core/ImportLists/StevenLu/` vs `RadarrList2/StevenLu/` | Duplicate |
| Plex Watchlist RSS (dup of Plex Watchlist) | `Core/ImportLists/Rss/Plex/` vs `Plex/` | Duplicate |
| RSS List vs RssImportBase | `Core/ImportLists/RSSImport/` vs `Rss/RssImportBase.cs` | Two RSS list frameworks |
| "Custom Lists" (RadarrList) | `Core/ImportLists/RadarrList/` | Legacy list-server JSON format |
| Trakt popular obsolete types | `Core/ImportLists/Trakt/Popular/TraktPopularListType.cs:26,39` | Dead enum values |
| AHD indexer flags | `Core/Parser/Model/IndexerFlags.cs:39,58` | AwesomeHD tracker is gone (mig 191) |
| PTP old-settings health check | `Core/HealthCheck/Checks/PTPOldSettingsCheck.cs` | One-time migration nag |
| Torcache interceptor | `Core/Http/TorcacheHttpInterceptor.cs` | torcache.net dead since 2016 |
| CloudFlare captcha interceptor + UI captcha flow | `Core/Http/CloudFlare/*`, `Core/Indexers/HttpIndexerBase.cs:246,378`, `frontend/src/Store/Actions/captchaActions.js` | Obsolete anti-bot approach (FlareSolverr/Prowlarr handle this) |
| Basic auth | `Core/Authentication/AuthenticationType.cs:8` | Marked obsolete |
| Deprecated API bits | `Radarr.Api.V3/MovieFiles/MovieFileController.cs:108`, `ImportLists/ImportListExclusionController.cs:35`, `Queue/QueueResource.cs:44,47`, `CheckForFinishedDownloadCommand` | API cruft |
| Sonarr leftovers | `Core/Organizer/AbsoluteEpisodeFormat.cs` (EpisodeFormat class), `Core/Parser/Model/SeriesTitleInfo.cs`, `Core/Movies/MonitoringOptions.cs` (`IgnoreEpisodesWithFiles/WithoutFiles`), `Parser.cs:136` `SpecialEpisodeWordRegex`, `History.cs:41,45`, `Radarr.Http/Extensions/RequestExtensions.cs:12` (Readarr ref), `002_103_sonarr_squashed.cs` | Dead fork residue |
| Windows tray / service installer / firewall COM / RegisterUrl | `src/NzbDrone/`, `src/ServiceHelpers/`, `src/NzbDrone.Host/AccessControl/`, `ApplicationModes.RegisterUrl` | Go binary + OS service managers replace these |
| NzbDrone.Mono naming & Mono.Posix | `src/NzbDrone.Mono/` | Historical Mono runtime |
| Legacy appdata / nzbdrone.db migration | `NzbDrone.Common/EnvironmentInfo/AppFolderFactory.cs:74-150` | Old install layouts |
| Built-in updater (BuiltIn/Script) | `src/NzbDrone.Update/`, `Core/Update/` | Most installs are Docker/package managed |
| Dual serializers / RestSharp 106 | csproj refs | Tech debt |
| Usenet Crawler preset | `Core/Indexers/Newznab/Newznab.cs:58` | Likely stale preset list |
| 140 migrations incl. removal migrations | `Core/Datastore/Migration/` | A rewrite starts at a fresh schema. Keep only an importer |

---

## 11. Rewrite notes

**Subtle behaviors to preserve**
1. **Movie title parsing.**
   - The regex list order matters (`Parser.cs:27-70`).
   - Year bounds are 1800–2099. Parsed years below 1800 mean "unknown" and skip year checks.
   - `SecondaryYear` from TMDb premiere date tolerates off-by-one years (`SkyHookProxy.cs:253`).
   - AKA/slash splitting produces multiple `MovieTitles`.
   - `CleanMovieTitle`/`NormalizeRegex` strip articles and "a/an/the/and/or/of" except in edge positions.
   - Roman ↔ Arabic numeral equivalence.
   - German umlaut expansion. `ReversedTitleRegex` (`p027`/`p0801`).
   - Rejection of hashed/obfuscated titles (`RejectHashedReleasesRegex`, `Parser.cs:73-100`).
   - Anime-style `[Group]` prefixes.
   - IMDb/TMDb ids embedded in names (`tt\d{7,8}`, `tmdbid-123`).
   - There is a large fixture suite in `src/NzbDrone.Core.Test/ParserTests/` worth porting as golden tests.
2. **Editions.** `EditionRegex` (`Parser.cs:18`) has lookaheads so "Special/Final/etc." only match when followed by Cut/Edition/Version. The edition is stripped from the title before matching and exposed via `{Edition Tags}` and the CF `EditionSpecification`.
3. **Availability logic** (`Movie.IsAvailable`, `Movie.cs:77-118`).
   - TBA/Announced means always available.
   - InCinemas uses the cinema date.
   - Released uses min(physical, digital). If neither exists, it falls back to InCinemas+90 days, or never.
   - `AvailabilityDelay` (days, can be negative) is applied except for sentinel values.
   - `MovieStatusType.Deleted` is set when TMDb returns 404 (`RefreshMovieService.cs:87-91`), which surfaces in `RemovedMovieCheck`.
   - PreDB was removed (mig 202; frontend migrator `migratePreDbToReleased.js`).
4. **Collections.** Auto-created from metadata. A monitored collection auto-adds missing movies while respecting import exclusions. Refresh cadence is 6h/15d. `MonitorTypes.MovieAndCollection` applies on add. Movie→collection linkage is by `CollectionTmdbId`.
5. **Translations / multi-language.**
   - `MovieInfoLanguage` selects the display title/overview from `MovieTranslations`.
   - Matching searches translations, alt titles and original title.
   - Search criteria include translated titles.
   - The quality profile has a single `Language` with the `Original` (movie's `OriginalLanguage`) and `Any` sentinels.
   - Language CFs and the `LanguageParser` "MULTI" handling, plus the per-indexer "multi languages" setting feeding `AggregateLanguages` (`Core/Download/Aggregation/Aggregators/AggregateLanguages.cs`).
   - Localized renaming tokens.
   - 45 UI locales.
6. **Release dates.** Three dates (InCinemas, Physical, Digital) plus Certification per `CertificationCountry`. The calendar/iCal feed shows all three.
7. **Single-file model.** One `MovieFile` per movie. Upgrades replace it and send the old file to the recycle bin.
8. **Scoring.** Minimum, cutoff and min-upgrade CF scores, upgrade-allowed, and delay-profile bypass by CF score all interact in `UpgradableSpecification`/`UpgradeDiskSpecification`.
9. **Import-list cleanup semantics** (`ListSyncLevel`) and exclusions shared with collections and "Discover".
10. **Metadata refresh.** `movie/changed` delta sync is only used if the last full run was within 14 days.
11. **Tracked-download matching.** Matching goes by download id via history, and falls back to title parsing. Manual-interaction-required event when that fails.
12. **Custom script / webhook contracts** (§5) are a de-facto public API used by many third-party tools.

**Accidental complexity**
- ThingiProvider reflection plus attribute-driven UI schemas. This is flexible but heavy. A Go rewrite can use a registry with typed settings structs and generated JSON schema.
- Two metadata endpoints (RadarrAPI proxy plus direct TMDb for lists).
- Many overlapping import-list types (RadarrList, RadarrList2, RSSImport, Rss/*).
- Lazy-loaded entity graphs with hand-rolled SQL joins (`MovieRepository.cs:60-210`), and embedded-JSON columns (Quality, MediaInfo, Tags, Settings).
- Command queue persisted to DB with priorities and exclusivity.
- Split Redux + react-query frontend state.
- Dual OS abstraction projects.
- 140 migrations, including a "sonarr_squashed" base.
- The `Movie`/`MovieMetadata` split, done to share metadata with import-list and collection movies, plus compatibility pass-through properties.

### Divergence from Sonarr
**Shared or forked (likely identical in concept to Sonarr and dedupable in a monolith):**
- ThingiProvider
- Datastore/Dapper
- Messaging/Commands
- Jobs/TaskManager
- HealthCheck framework
- Housekeeping pattern
- Backup
- Update + `NzbDrone.Update`
- Authentication/Radarr.Http
- SignalR
- Download client implementations (nearly the same set as Sonarr, same class names)
- Newznab/Torznab/TorrentRss indexers, RSS parsing, indexer status/backoff
- Blocklist, History (pattern), Queue, Pending releases, Delay profiles, Release profiles, RemotePathMappings, RootFolders, Tags, AutoTagging (framework)
- CustomFormats (framework)
- Notification framework + most providers (Discord, Telegram, Pushover, Email, Plex, Emby, Kodi, Webhook, CustomScript, ...)
- Metadata consumer framework
- Localization
- MediaCover
- Common (HTTP client, disk, processes, env info)
- Mono/Windows projects
- Host bootstrap
- Frontend scaffolding (Components, Settings pages, Store)

**Movie-specific (Radarr divergence):**
- `Movie` + `MovieMetadata` split; the single `MovieFile`.
- `MovieStatusType` + `MinimumAvailability` + `AvailabilityDelay` (Sonarr uses air dates per episode).
- **Collections** (`Core/Movies/Collections/`, `Collections` table, collection monitoring/auto-add).
- **Credits** (cast/crew) and **Recommendations** / "Discover" (`ImportListMovies`, `frontend/src/DiscoverMovie`).
- **Translations** table + `MovieInfoLanguage`. **Alternative titles** from TMDb (Sonarr uses scene mappings/XEM instead).
- **Editions** parsing + `{Edition Tags}` + CF `EditionSpecification`. CF `QualityModifierSpecification` and `YearSpecification`.
- Movie-oriented quality set: CAM/TELESYNC/TELECINE/WORKPRINT/DVDSCR/REGIONAL/BRDISK/RAWHD/Remux + `Modifier`.
- Quality profile **Language** field (Sonarr v4 dropped profile language in favor of CFs).
- Metadata source: `api.radarr.video` (TMDb) vs Sonarr's SkyHook (TVDB). Search accepts IMDb/TMDb URLs.
- Import lists focused on TMDb (List/Popular/User/Person/Company/Keyword), IMDb, StevenLu, Simkl, Plex Watchlist, CouchPotato, Radarr.
- Naming: a single `StandardMovieFormat` + `MovieFolderFormat` (no episode/daily/anime formats; removed in migs 143/148/240).
- No season packs or episode/scene-numbering logic.
- Indexer flags include PTP-specific flags. PassThePopcorn and HDBits movie APIs search by IMDb id.
- Movie-specific commands: Rename/Move/RefreshCollections, and "movie/import" library import.
- Calendar shows cinema, digital and physical dates.
- Webhook/CustomScript payload names (`Radarr_Movie_*`, `MovieFile_*`).
