# Prowlarr — Codebase Analysis

Submodule: `submodules/prowlarr` (HEAD `12c32780`, 2026-09-13). All paths below are relative to that root unless absolute. `Core/` = `src/NzbDrone.Core/`.

---

## 1. Overview

**Purpose.** Prowlarr is an indexer manager/proxy. It manages indexer configs (native C# indexers, Cardigann YAML scrapers, generic Newznab/Torznab), then exposes each one as a **Newznab/Torznab-compatible endpoint** (`/{id}/api`). It pushes ("syncs") those endpoints into downstream *arr apps (Sonarr, Radarr, Lidarr, Readarr, Whisparr, Mylar, LazyLibrarian). It also has its own multi-indexer search UI, grabs to download clients, per-indexer query/grab limits, history/stats, and HTTP/SOCKS/FlareSolverr proxies.

**Tech stack**
- Backend: .NET 8 (`global.json` SDK 8.0.421, `src/Directory.Build.props` → `net8.0`), ASP.NET Core, DryIoc DI, Dapper plus a custom SQL builder, FluentMigrator (SQLite and PostgreSQL), NLog, Sentry, Polly 8, YamlDotNet 16 (Cardigann), AngleSharp 1.5 (HTML/CSS selectors), MonoTorrent 3 (torrent metadata), MailKit, Newtonsoft.Json, FluentValidation 9, SignalR.
- Frontend: React 17, Redux + connected-react-router, SignalR client 8, chart.js 4, webpack. It is partly migrated to TS: about 400 `.ts/.tsx` files and about 474 `.js` files (`package.json`, `frontend/src`).
- This is a fork of the Sonarr v3 / "NzbDrone" codebase. Namespaces are still `NzbDrone.*`, and assemblies are renamed `Prowlarr.*.csproj`.

**Rough size (LOC, .cs or .ts/.tsx/.js, incl. blank lines)**

| Project | LOC |
|---|---|
| `src/NzbDrone.Core` (Prowlarr.Core) | ~82,000 |
| `src/NzbDrone.Common` | ~14,100 |
| `src/Prowlarr.Api.V1` | ~4,550 |
| `src/Prowlarr.Http` | ~3,350 |
| `src/NzbDrone.Host` | ~1,550 |
| `src/NzbDrone.Mono` | ~1,370 |
| `src/NzbDrone.Update` | ~650 |
| `src/NzbDrone.Windows` | ~290 |
| `src/NzbDrone` (Windows tray app) | ~240 |
| `src/NzbDrone.Console` | ~130 |
| `src/NzbDrone.SignalR` | ~95 |
| `frontend/src` | ~53,600 |

A large share of Core is `Core/Indexers/Definitions/` (native indexers plus the Cardigann engine, about 3.5k LOC in `Cardigann/`).

---

## 2. Domain model

Table mappings are in `Core/Datastore/TableMapping.cs`. Most "things" are ThingiProvider **ProviderDefinitions** (`Core/ThingiProvider/ProviderDefinition.cs`: `Name, Implementation, ImplementationName, ConfigContract, Enable, Tags (HashSet<int>), Settings (JSON blob IProviderConfig)`).

| Entity | File | Key fields / notes |
|---|---|---|
| `IndexerDefinition` (table Indexers) | `Core/Indexers/IndexerDefinition.cs` | `IndexerUrls[], LegacyUrls[]` (not persisted), `Description, Encoding, Language, Protocol (Usenet/Torrent), Privacy (Public/SemiPrivate/Private), SupportsRss/Search/Redirect/Pagination, Capabilities` (runtime), `Priority` (default 25), `Redirect` (bool), `DownloadClientId` (per-indexer client, mig 035), `Added, AppProfileId → AppSyncProfile, ExtraFields` (Cardigann settings). The settings JSON holds `BaseUrl`, `BaseSettings {QueryLimit, GrabLimit, LimitsUnit(Day/Hour)}` (`Core/Indexers/IndexerBaseSettings.cs`) and, for torrents, `TorrentBaseSettings {AppMinimumSeeders, SeedRatio, SeedTime, PackSeedTime, PreferMagnetUrl}` (`Core/Indexers/IndexerTorrentBaseSettings.cs`). |
| `IndexerStatus` | `Core/Indexers/IndexerStatus.cs` | ProviderStatusBase (`InitialFailure, MostRecentFailure, EscalationLevel, DisabledTill`), plus `LastRssSyncReleaseInfo`, **`Cookies` (dict) and `CookiesExpirationDate`**. Login session cookies are persisted here. |
| `ApplicationDefinition` (Applications) | `Core/Applications/ApplicationDefinition.cs` | `SyncLevel` enum {Disabled, AddOnly, FullSync} (`ApplicationSyncLevel.cs`). `Enable` = AddOnly or FullSync. Settings: ProwlarrUrl, BaseUrl, ApiKey, SyncCategories, etc. |
| `ApplicationStatus` | `Core/Applications/ApplicationStatus.cs` | Failure backoff per app. |
| `AppIndexerMap` | `Core/Applications/AppIndexerMap.cs` | `IndexerId, AppId, RemoteIndexerId, RemoteIndexerName` (the name is used by LazyLibrarian/Mylar, which lack ids; mig 011). |
| `AppSyncProfile` ("App Profile") | `Core/Profiles/AppSyncProfile.cs` | `Name, EnableRss, EnableAutomaticSearch, EnableInteractiveSearch, MinimumSeeders`. Each indexer references one. Its values are pushed to remote *arr indexer flags. |
| `IndexerProxyDefinition` | `Core/IndexerProxies/IndexerProxyDefinition.cs` | `Enable => Tags.Any()`. A proxy is only active when tagged. |
| `DownloadClientDefinition` / `DownloadClientStatus` | `Core/Download/` | Standard Sonarr-style. Also has `Categories` (mig 023, `DownloadClientCategory.cs`). |
| `NotificationDefinition` / `NotificationStatus` | `Core/Notifications/NotificationDefinition.cs` | `OnGrab, OnHealthIssue, OnHealthRestored, OnApplicationUpdate, IncludeManualGrabs, IncludeHealthWarnings`. |
| `History` | `Core/History/History.cs` | `IndexerId, Date, Successful, EventType {Unknown, ReleaseGrabbed, IndexerQuery, IndexerRss, IndexerAuth, IndexerInfo}, Data (dict: query, source, host, elapsedTime, url, title, …), DownloadId`. This table is the source for **stats and for rate-limit counting**. |
| `IndexerDefinitionVersion` | `Core/IndexerVersions/IndexerDefinitionVersion.cs` | `File, Sha, LastUpdated, DefinitionId`. **This is dead.** The table was created in mig 015, and its repo/service are registered, but nothing outside `IndexerVersions/` consumes it. |
| `Tag`, `CustomFilter`, `User` (+salt, mig 024), `Config` (key/value), `ScheduledTask`, `CommandModel`, `UpdateHistory`, `Log` (separate log DB) | various | Standard *arr infra. |

**Relationships.** Indexer → AppSyncProfile (N:1). Indexer ↔ Tags ↔ Application/IndexerProxy/Notification/DownloadClient (tag intersection gates sync and proxy use). Application × Indexer → AppIndexerMap (remote id). Indexer → History (1:N) → Stats. Indexer → DownloadClient (optional N:1).

**Migrations.** `Core/Datastore/Migration/`: 44 files, numbered 000–044 with gaps (009 and 012 are missing). The framework is in `Migration/Framework/`. Notable ones: 015 indexer_versions, 019 remove_showrss, 020 remove_torrentparadiseml, 028 remove_notwhatcd, 033 remove_uc, 036 postgres timestamptz, 040 newznab category → capabilities settings. Many of them are per-tracker settings-contract fixups (Redacted, Orpheus, SpeedCD, TorrentDay, AlphaRatio/GPW, MyAnonamouse, Gazelle freeleech).

---

## 3. Feature areas

Every top-level folder in `Core/`:

- **Analytics/**: `AnalyticsService`. Checks whether analytics is enabled and whether the install is active (recent history). Used for Sentry tagging.
- **Annotations/**: `FieldDefinitionAttribute`, `SelectOption`. Drives dynamic provider settings forms (schema endpoint → UI).
- **Applications/**: app sync (deep dive below).
- **Authentication/**: `User`, `UserService`, `AuthenticationType {None, Basic[Obsolete], Forms, External}`, `AuthenticationRequiredType`.
- **Backup/**: `BackupService`, `MakeDatabaseBackup`. Zips the DB and config.xml. Types are Scheduled/Manual/Update. Interval is `BackupInterval` (default 7 days) and retention is `BackupRetention` (28 days) (`Configuration/ConfigService.cs:177-179`).
- **Configuration/**: `ConfigService` (DB key/value) and `ConfigFileProvider` (config.xml). See §7.
- **CustomFilters/**: saved UI filters (JSON).
- **Datastore/**: Dapper repositories, `SqlBuilder`, `WhereBuilderSqlite/Postgres`, `LazyLoaded`, migrations, `DatabaseRestorationService`, separate `MainDatabase`/`LogDatabase`.
- **Download/**: download clients for manual grabs from the Prowlarr UI (`DownloadService.SendReportToClient`). Also covers the proxied `.torrent/.nzb` download (`DownloadReport`), `RecordRedirect`, `DownloadMappingService` (link encryption), `NzbValidationService`, and `TorrentFileInfoReader` (MonoTorrent infohash).
- **Exceptions/**: `ReleaseUnavailableException`, `ReleaseDownloadException`, `SearchFailedException`, `StatusCodeToExceptions`, etc.
- **HealthCheck/**: 20 checks (§6/§3 deep dive).
- **History/**: `HistoryService`. Records `IndexerQueryEvent`, `IndexerDownloadEvent`, `IndexerAuthEvent`, `IndexerInfoEvent` and grab events. Provides `CountSince`/`FindFirstForIndexerSince` for limits and `Between` for stats. `CleanUpHistoryCommand` prunes after `HistoryCleanupDays`, and `ClearHistoryCommand` clears it.
- **Housekeeping/**: 16 housekeepers (`Housekeepers/`): orphaned status and history cleanup, FixFuture*Times, CleanupUnusedTags, CleanupTemporaryUpdateFiles, CleanupAdditionalUsers, CleanupCommandQueue, TrimLogDatabase. Also VACUUMs the main DB (`HousekeepingService.cs:42`).
- **Http/**: `HttpProxySettingsProvider` (global proxy), `CloudFlare/CloudFlareDetectionService`, and **`TorcacheHttpInterceptor`** (torcache.net died in 2016; this is legacy).
- **IndexerProxies/**: per-indexer outbound proxies (deep dive below).
- **IndexerSearch/**: `ReleaseSearchService`, `NewznabRequest` (the query-string model), `NewznabResults` (RSS/Torznab XML writer), and search criteria (`Basic/Tv/Movie/Music/BookSearchCriteria`).
- **IndexerStats/**: `IndexerStatisticsService`. Computed on the fly from History (no stats table).
- **IndexerVersions/**: Cardigann definition fetch/update (`IndexerDefinitionUpdateService`), plus the dead `IndexerDefinitionVersion*`.
- **Indexers/**: the indexer framework and all definitions (deep dive below).
- **Instrumentation/**: NLog DB target, log service, `DeleteLogFilesService`, Sentry reconfig, `SlowRunningAsyncTargetWrapper`.
- **Jobs/**: `TaskManager` (scheduled tasks, §6), `Scheduler`.
- **Lifecycle/**: startup/shutdown events, `RestartCommand`, `ShutdownCommand`.
- **Localization/**: `LocalizationService`, with 45 JSON locale files in `Localization/Core/`.
- **Messaging/**: in-process command queue and event aggregator (`IExecute<T>`, `IHandle<T>`, `IHandleAsync<T>`). This is the classic NzbDrone design.
- **Notifications/**: 19 notification providers (§4).
- **Parser/**: much smaller than Sonarr's. Date parsing (`DateTimeRoutines`, `DateTimeUtil`), `ParseUtil` (size/number parsing), `StringUtil`, `RomanNumerals`, `TvCategoryFromQualityParser` (used only by Nebulance), and `Model/ReleaseInfo`, `TorrentInfo`.
- **Profiles/**: `AppSyncProfile` CRUD.
- **ProgressMessaging/**: pushes command progress to the UI.
- **Security/**: `ProtectionService` (AES encryption of proxied download links using config key `DownloadProtectionKey`), `X509CertificateValidationService`, `CertificateValidationType`.
- **Tags/**: tags plus `TagDetails` (usage per indexer/app/proxy/notification/download client).
- **ThingiProvider/**: the generic provider/factory/repository/status framework shared by indexers, apps, proxies, download clients and notifications.
- **Update/**: `UpdateCheckService`, `InstallUpdateService`, `UpdatePackageProvider` (`https://prowlarr.servarr.com/v1/update/{branch}`), `UpdateMechanism {BuiltIn, Script, External, Apt, Docker}`, and update history.
- **Validation/**: FluentValidation helpers (`UrlValidator`, `AppProfileExistsValidator`, `DownloadClientExistsValidator`, path validators).

### 3.1 Indexer framework (`Core/Indexers/`)
- Class hierarchy: `IIndexer` → `IndexerBase<TSettings>` → `HttpIndexerBase<TSettings>` → `TorrentIndexerBase` / `UsenetIndexerBase`. Family bases are `GazelleBase`, `AvistazBase`, `SpeedAppBase`, `Unit3dBase` (unused), and `CardigannBase`.
- Each indexer supplies `IIndexerRequestGenerator` (builds `IndexerPageableRequestChain` per search type) and `IParseIndexerResponse`.
- `HttpIndexerBase` (`HttpIndexerBase.cs`):
  - Per-indexer `RateLimit` defaults to 2 s (overrides: Xthor 2.5 s, Avistaz 6 s, BTN 5 s, and about 13 overrides in total). This is enforced by the shared `IRateLimitService` keyed on host.
  - Polly retry: 2 attempts, exponential backoff with jitter, on 5xx or 408.
  - Pagination via `PageSize`/`IsFullPage`, with `MaxNumResultsPerQuery = 1000`.
  - `CheckIfLoginNeeded`/`DoLogin` hooks. Cookies are loaded from and saved to `IndexerStatus.Cookies`, with a 30-day expiry after a successful request (`:710`).
  - `IsValidRelease` drops releases without title, size or categories unless the search is interactive.
- `IndexerFactory` (`IndexerFactory.cs`):
  - Merges native providers with Cardigann definitions. Each YAML file becomes a pseudo-"default definition" of implementation `Cardigann`.
  - Hides `[Obsolete]` providers from the add list (`:170`).
  - Maps YAML `caps.categorymappings` to Newznab cats.
- Status/backoff: `IndexerStatusService` (escalating DisabledTill, like Sonarr's).
- `IndexerLimitService`: see §3.6.
- `SeedConfigProvider`: per-indexer seed ratio/time for grabs to torrent clients.
- Parsers: `RssParser`, `TorrentRssParser`, `EzrssTorrentRssParser`, `XmlCleaner`, `MagnetLinkBuilder` (with a default tracker list).

### 3.2 Cardigann YAML engine (`Core/Indexers/Definitions/Cardigann/`)
- **Model.** `CardigannDefinition.cs` holds the fields `id, name, description, type (public/semi-private/private), language, encoding, requestDelay, links, legacylinks, followredirect, testlinktorrent, certificates, settings[], caps, login, ratio, search, download`.
  - `caps`: `categories` (map), `categorymappings` (id, cat, desc, default), `modes` (search/tv-search/movie-search/music-search/book-search → params), `allowrawsearch`.
  - `login`: `path, submitpath, cookies, method, form, selectors, inputs, selectorinputs, getselectorinputs, error[], test{path,selector}, captcha{type,selector,input}, headers`.
  - `search`: `path|paths[] {path, method, inputs, queryseparator, categories, inheritinputs, followredirect, response{type json|xml|html, noResultsMessage}}, headers, keywordsfilters, allowEmptyInputs, inputs, error, preprocessingfilters, rows{selector, after, dateheaders, count, multiple, missingAttributeEqualsNoResults}, fields (ordered multimap)`.
  - `download`: `selectors[], method, before{…, pathselector}, infohash{hash,title}, headers`.
  - `captcha.image` is deprecated and throws (`:87`).
- **Settings field types** (`CardigannBase.cs:296-341`, `Cardigann.cs:137`): `text, password, checkbox, select, info, info_cookie, info_flaresolverr, info_useragent, info_category_8000, cardigannCaptcha`. (`multi-select` exists only in Jackett, where no definition uses it; see jackett.md.)
- **Login methods** (`CardigannRequestGenerator.cs:202-563`):
  - `post`
  - `form`: fetches the landing page, scrapes form inputs, and auto-solves `simpleCaptcha` (`:387`). Image captcha is shown to the user via the `checkCaptcha` action (`Cardigann.cs:217`).
  - `cookie`: user pastes a cookie string.
  - `get`
  - `oneurl`
- **Template language** (`CardigannBase.cs:371-600`). This is a hand-rolled, regex-based subset of Go `text/template`, **not** a real template engine:
  - Supports `{{ .Var }}`, `{{ if … }}…{{ else }}…{{ end }}`, `{{ range … }}…{{.}}…{{end}}`, `{{ re_replace .X "pat" "rep" }}` and `{{ join .X "," }}`, plus logic functions `and/or/eq/ne` (`_LogicFunctionRegex`).
  - Variables: `.Config.*` (incl. `sitelink`), `.Today.Year`, `.Categories`, `.Result.*` (row fields), and `.Query.{Q, Keywords, Type, Categories, Limit, Offset, Extended, APIKey, Season, Ep, Episode, Series, Movie, Year, Genre, Album, Artist, Label, Track, Author, Title, Publisher, IMDBID, IMDBIDShort, TMDBID, TVDBID, TVRageID, TVMazeID, TraktID, DoubanID}`.
- **Filters** (`CardigannBase.cs`): `querystring, timeparse/dateparse, regexp, re_replace, split, replace, trim, prepend, append, tolower, toupper, urldecode, urlencode, htmldecode, htmlencode, timeago/reltime, fuzzytime, validfilename, diacritics, jsonjoinarray, hexdump, strdump, validate`. There are also selector-level `case`, `remove`, `attribute`, `optional`, `default`, and JSON selector syntax `:filter(key)` (`_jsonSelectorRegex`).
- **Row fields recognised** (`CardigannParser.cs`): `download, magnet, infohash, details, comments, title, description, category, categorydesc, size, files, leechers, seeders, grabs, date, poster, imdb/imdbid, tmdbid, tvdbid, rageid, tvmazeid, traktid, doubanid, year, genre, author, booktitle, publisher, artist, album, label, track, downloadvolumefactor, uploadvolumefactor, minimumratio, minimumseedtime, strdump`, and `andmatch` (filters rows whose title does not contain all query terms). The `noappend` modifier is deprecated (`:513`).
- **Response types**: HTML (AngleSharp CSS selectors), JSON (`response.type: json`, `:68`) and XML (`:220`).
- **Definition fetching/updating** (`Core/IndexerVersions/IndexerDefinitionUpdateService.cs`):
  - Constants: `DEFINITION_BRANCH = "master"`, `DEFINITION_VERSION = 11`. This is the schema version pin; the app accepts only definitions built for format v11.
  - Index list: `GET https://indexers.prowlarr.com/master/11`, which returns a JSON list of `CardigannMetaDefinition`. On failure it falls back to reading `{AppData}/Definitions/*.yml`. It always appends `{AppData}/Definitions/Custom/*.yml` (user custom definitions; names and files must be unique).
  - Single definition: local `{AppData}/Definitions/**/{file}.yml` first, then `GET https://indexers.prowlarr.com/master/11/{id}`. Results are cached in memory (`ICached<CardigannDefinition>`).
  - Bulk update: on `ApplicationStartedEvent` and daily via `IndexerDefinitionUpdateCommand`, it downloads `https://indexers.prowlarr.com/master/11/package.zip`, extracts it over `{AppData}/Definitions`, and clears the cache.
  - `_definitionBlocklist` (the YAML files superseded by C# indexers): `aither, animeworld, audiobookbay, beyond-hd-oneurl, beyond-hd, blutopia, brsociety, danishbytes, datascene, desitorrents, hdbits, lat-team, mteamtp, mteamtp2fa, reelflix, shareisland, skipthecommercials, tellytorrent`. Several of these (the UNIT3D ones) have **no** matching C# class any more. See §10.
  - Normalization (`CleanIndexerDefinition`): default settings are username/password, encoding defaults to UTF-8, login method defaults to `form`, and single `search.path` is converted to `paths[]`.
  - Source repo: the YAML lives in the **Prowlarr/Indexers** GitHub repo (`definitions/v{N}/*.yml`), which is published to `indexers.prowlarr.com`. Nothing in this repo pins a commit; the only pin is the schema version `11` plus branch `master`.
- Health: `IndexerNoDefinitionCheck` flags Cardigann indexers whose file no longer exists in the list. `OutdatedDefinitionCheck` flags `[Obsolete]` natives and blocklisted YAML files.

### 3.3 Native indexer implementations
There are 69 concrete classes. The full list is in §4.1. The families are:
- `GazelleBase` (Gazelle JSON API): AlphaRatio, BrokenStones, CGPeers, DesiGaane, DICMusic, GreatPosterWall, SecretCinema. Redacted and Orpheus have their own API implementations.
- `AvistazBase`: AvistaZ, AnimeZ, CinemaZ, ExoticaZ, PrivateHD.
- `SpeedAppBase`: SpeedApp, RetroFlix.
- Generic: Newznab (with presets), Torznab (with presets), TorrentRssIndexer (generic RSS with `TorrentRssSettingsDetector` auto-detection), TorrentPotato, Headphones.

### 3.4 Newznab/Torznab proxy endpoints it serves
`Prowlarr.Api.V1/Indexers/NewznabController.cs`:
- `GET /{id}/api` (alias `/api/v1/indexer/{id}/newznab`), with `t=caps|search|tvsearch|movie|music|book`. Query params come from `IndexerSearch/NewznabRequest.cs`: `q, cat, imdbid, tmdbid, tvdbid, rid, tvmazeid, traktid, doubanid, season, ep, album, artist, label, track, year, genre, author, title, publisher, extended, limit, offset, minage, maxage, minsize, maxsize, configured`.
- Error XML uses Newznab codes: 200 missing t, 201 bad imdbid, 202 no such function. It returns HTTP 429 plus `Retry-After` when the indexer is backed off or at its query/grab limit, and 410 when the indexer is disabled.
- `id == 0` is a fake "test" indexer. It returns full-category caps and one dummy release; the apps' test calls use it.
- In results, every `DownloadUrl`/`MagnetUrl` is rewritten to `{server}{urlBase}/{id}/download?apikey={key}&link={base64url(AES-protected original)}&file={title}` (`Core/Download/DownloadMappingService.cs`).
- `GET /{id}/download` (alias `/api/v1/indexer/{id}/download`):
  - Checks backoff and the grab limit, then decrypts the link.
  - Usenet indexers always get a **301 redirect**. Torrent indexers redirect if `SupportsRedirect && Redirect`. Otherwise Prowlarr fetches the file itself using indexer cookies, proxy and FlareSolverr, and streams it back.
  - If the bytes start with `magnet:`, it returns a 301 to the magnet.
- Output XML (`IndexerSearch/NewznabResults.cs`): namespace `torznab` (`http://torznab.com/schemas/2015/feed`) or `newznab`. Attrs: `category, seeders, peers, infohash, downloadvolumefactor, uploadvolumefactor, minimumratio, minimumseedtime, files, grabs, imdb, tmdbid, tvdbid, rageid, tvmazeid, traktid, doubanid, year, genre, author, booktitle, artist, album, label, track, coverurl, language, subs, tag`.
- Caps XML (`Indexers/IndexerCapabilities.cs`): `server, limits, searching{search, tv-search, movie-search, music-search, audio-search, book-search}, categories/subcat, tags`. The Torznab XSD is at `schemas/torznab.xsd`.

### 3.5 Search, categories and capabilities
- `ReleaseSearchService` (`IndexerSearch/ReleaseSearchService.cs`):
  - Builds criteria per `t` and dispatches to all enabled indexers in parallel (`Task.WhenAll`). Indexer ids `-1` and `-2` in the id list mean "all usenet" and "all torrent".
  - Skips indexers that support none of the requested categories, and skips indexers at their query limit.
  - After fetching, it filters by expanded categories, minage/maxage and minsize/maxsize. It publishes `IndexerQueryEvent` (which feeds history). It dedupes by GUID, keeping the lowest `IndexerPriority` value.
- UI search (`Prowlarr.Api.V1/Search/SearchController.cs`): `GET /api/v1/search?query&type&indexerIds&categories&limit&offset`. It supports inline id syntax such as `{TvdbId:123}{Season:1}` parsed by `NewznabRequest.QueryToParams()` (UI only). Results are cached for grab. `POST /api/v1/search` and `/bulk` grab to a download client.
- Categories:
  - `NewznabStandardCategory.cs` has 76 standard cats (0000 Other … 8000 Other), keeping Newznab numbering (1000 Console, 2000 Movies, 3000 Audio, 4000 PC, 5000 TV, 6000 XXX, 7000 Books, 8000 Other).
  - `IndexerCapabilitiesCategories.AddCategoryMapping` maps tracker cat → standard cat. **If a description is supplied, it also creates a custom category with id `100000 + trackerCatInt`**. For non-numeric tracker cats the id is `100000 + UInt16(SHA1(cat)[0..2])`. Stability matters because apps store these ids.
  - `ExpandTorznabQueryCategories` maps a parent to its children and maps custom ≥100000 cats back to tracker cats. `SupportedCategories` is used by app sync.
- Capabilities: `IndexerCapabilities` holds `LimitsMax/Default, SupportsRawSearch`, the param lists `Search/Tv/Movie/Music/Book SearchParams` (q, season, ep, imdbid, tvdbid, rid, tvmazeid, traktid, tmdbid, doubanid, genre, year, album, artist, label, track, author, title, publisher), `Flags` (`IndexerFlag`: internal, exclusive, freeleech, …), and `Categories`. Newznab/Torznab indexers fetch the remote caps (`Newznab/NewznabCapabilitiesProvider.cs`, cached). Since mig 040, the user can override categories in `NewznabCapabilitiesSettings`.

### 3.6 Rate limiting and query limits
- Transport rate limit: `HttpIndexerBase.RateLimit` (default 2 s per indexer) and Cardigann `requestDelay`, via `IRateLimitService` in `IndexerHttpClient`.
- User-configurable limits: `QueryLimit`/`GrabLimit` per `LimitsUnit` (Day or Hour), evaluated by **counting History rows** since `now - N hours` (`Core/Indexers/IndexerLimitService.cs`). RSS and search both count toward QueryLimit. The Retry-After value is computed from the Nth-most-recent history row.
- Failure backoff: `IndexerStatusService` escalation, also surfaced as 429.

### 3.7 App sync
Covered in §9.

### 3.8 Indexer proxies (`Core/IndexerProxies/`)
- Types: `Http`, `Socks4`, `Socks5`, `FlareSolverr`.
- A proxy applies to an indexer only when tags intersect. An indexer with no tags gets no proxies (`Core/Indexers/IndexerHttpClient.cs:55-79`). Proxies hook in as `PreRequest`/`PostResponse` interceptors.
- FlareSolverr (`FlareSolverr/FlareSolverr.cs`):
  - Acts only in `PostResponse` when `CloudFlareDetectionService.IsCloudflareProtected`. Detection requires a Server header of `cloudflare`, `cloudflare-nginx` or `ddos-guard`, plus a 503/403 status with a "Just a moment…"/"Access denied"/"Attention Required"/"DDOS-GUARD" title.
  - It POSTs `request.get` or `request.post` (urlencoded) to FlareSolverr with `maxTimeout` and passes through the global proxy.
  - It then replays the original request with the solution cookies and the UA, which is cached per host in memory.
- There is also a global proxy (`Core/Http/HttpProxySettingsProvider.cs`; settings `ProxyEnabled/Type/Hostname/Port/Username/Password/BypassFilter/BypassLocalAddresses`) and `ProxyCheck`.

### 3.9 Stats
`IndexerStatisticsService` reads `History.Between(start,end)` and aggregates in memory:
- Per indexer: avg response time, avg grab response time, #queries, #grabs, #rss, #auth, and failed counts of each.
- Per user agent (source app) and per host: queries and grabs.

The API is `GET /api/v1/indexerstats?startDate&endDate&indexers&protocols&tags`. The UI renders it with chart.js (`frontend/src/Indexer/Stats`).

### 3.10 History
History is written by event handlers for query, RSS, auth, info and grab events. The `Data` dict holds `source` (derived from the `User-Agent`, e.g. "Sonarr", via `UserAgentParser.ParseSource`, or from the `X-Prowlarr-Client` header), `host` (**reverse DNS of the caller per request**, `Prowlarr.Http/Extensions/RequestExtensions.cs:76-86`), query, elapsed time, url, title and grabMethod (Proxy/Redirect). The API is `/api/v1/history`, `/since`, `/indexer`.

### 3.11 Notifications
Events: `OnGrab` (with `IncludeManualGrabs`), `OnHealthIssue` (with `IncludeHealthWarnings`), `OnHealthRestored`, `OnApplicationUpdate` (`NotificationBase.cs:54-57`). These are fewer than in Sonarr/Radarr (no import/rename/delete). The provider list is in §4.

### 3.12 Health checks (`Core/HealthCheck/Checks/`)
`AllowedHostsCheck, ApiKeyValidationCheck, AppDataLocationCheck, ApplicationStatusCheck, ApplicationLongTermStatusCheck, DownloadClientStatusCheck, IndexerCheck, IndexerDownloadClientCheck, IndexerLongTermStatusCheck, IndexerNoDefinitionCheck, IndexerProxyStatusCheck, IndexerStatusCheck, IndexerVIPCheck, IndexerVIPExpiredCheck, NotificationStatusCheck, OutdatedDefinitionCheck, ProxyCheck, ReleaseBranchCheck, SystemTimeCheck (calls `prowlarr.servarr.com/v1/time`), UpdateCheck`.

### 3.13 Backups
See Backup/ above. The API is `/api/v1/system/backup` (list, delete, `restore/{id}`, `restore/upload`).

### 3.14 Updates
Self-updater via the Servarr update service. See §8.

---

## 4. Integrations (full lists)

### 4.1 Native indexers (C# classes in `Core/Indexers/Definitions/`)
Legend: **[OBS]** = marked `[Obsolete]` (hidden from the add list and flagged by health checks). **[dead?]** = service believed dead or niche; verify.

| Class | Base / notes |
|---|---|
| AlphaRatio | GazelleBase |
| Anidex | Torrent (public anime) |
| Anidub | Torrent (RU anime, user/pass) |
| AnimeBytes | Torrent API |
| Animedia | **[OBS] "Site is unusable due to a mix of HTTP errors"** (`Animedia.cs:21`) |
| AnimeZ | AvistazBase |
| AvistaZ | AvistazBase |
| BakaBT | Torrent |
| BeyondHD | Torrent API |
| BinSearch | Usenet. **[OBS] "Site has shutdown"** (`BinSearch.cs:22`) |
| BitHDTV | Cookie torrent |
| BroadcastheNet | JSON-RPC API; legacy URL `http://api.broadcasthe.net/` |
| BrokenStones | GazelleBase **[dead?]** |
| CGPeers | GazelleBase. **[OBS] "Moved to YML for Cardigann"** |
| CinemaZ | AvistazBase |
| DesiGaane | GazelleBase |
| DICMusic | GazelleBase |
| ExoticaZ | AvistazBase |
| FileList | Torrent API |
| FunFile | **[OBS] Moved to YML** |
| GazelleGames | Torrent API |
| GreatPosterWall | GazelleBase |
| HDBits | Torrent API |
| HDSpace | **[OBS] Moved to YML** |
| HDTorrents | Torrent |
| Headphones | Usenet (Headphones VIP indexer) **[dead?]** |
| ImmortalSeed | Torrent |
| IPTorrents | Cookie torrent |
| Knaben | Public meta-search |
| Libble | Torrent |
| MTeamTp | Torrent API |
| MyAnonamouse | Torrent API |
| Nebulance | Torrent API (uses `TvCategoryFromQualityParser`) |
| Newznab | Generic Usenet |
| NorBits | Torrent |
| NzbIndex | Usenet |
| Orpheus | Torrent API |
| PassThePopcorn | Torrent API |
| PixelHD | Torrent |
| PornoLab | **[OBS] Moved to YML** |
| PreToMe | Torrent |
| PrivateHD | AvistazBase |
| Rarbg | **[OBS] "Rarbg has shutdown 2023-05-31"** (`Rarbg/Rarbg.cs:23`) |
| Redacted | Torrent API |
| RetroFlix | SpeedAppBase |
| RevolutionTT | Torrent |
| RuTracker | Torrent |
| SceneHD | Torrent API |
| SceneTime | **[OBS] "Migrated to YAML for Torznab API"** |
| SecretCinema | GazelleBase |
| Shazbat | Torrent (TV) |
| Shizaproject | **[OBS] "Site unusable due to lack of new releases"** |
| SpeedApp | SpeedAppBase |
| SpeedCD | Torrent |
| SubsPlease | Public anime |
| Toloka | Torrent (UA) |
| TorrentBytes | Torrent |
| TorrentDay | Cookie torrent |
| TorrentPotato | CouchPotato "TorrentPotato" API **[dead?]** (CouchPotato is long dead) |
| TorrentRssIndexer | Generic torrent RSS |
| TorrentsCSV | Public |
| TorrentSyndikat | Torrent |
| Torznab | Generic torrent |
| Uniotaku | Torrent (BR anime) |
| XSpeeds | Torrent |
| Xthor | **[OBS] "Moved to YML for Cardigann v5"** |
| ZonaQ | **[OBS] "Site does not allow automation"** |
| Cardigann | YAML engine (one implementation, about 500+ definitions from the Indexers repo) |

- **Dead framework code**: `Definitions/UNIT3D/` (`Unit3dBase`, `Unit3dApi`, `Unit3dParser`, `Unit3dRequestGenerator`, `Unit3dSettings`) has **zero subclasses**. It is referenced only by migration 013.
- **Newznab presets** (`Newznab/Newznab.cs`): abNZB, altHUB, AnimeTosho (Usenet), DOGnzb, DrunkenSlug, GingaDADDY, Miatrix, Newz69, NinjaCentral, Nzb.life, NZBCat, NZBFinder, NZBgeek, NzbNoob, NZBNDX, NzbPlanet, NZBStars, Tabula Rasa, Generic Newznab.
- **Torznab presets**: AnimeTosho, Torrent Network, Generic Torznab.
- **Removed via migrations** (evidence of churn): showRSS (019), TorrentParadiseML (020), NotWhatCD (028), "UC" (033).

### 4.2 Cardigann
There is one C# implementation, `Cardigann`. Definitions are external (§3.2 and §11).

### 4.3 Synced Applications (`Core/Applications/`)

| App | Proxy / API | Min version | Default sync categories | Status |
|---|---|---|---|---|
| Sonarr | `SonarrV3Proxy` `/api/v3/indexer` | 3.0.5 | 5000,5010,5020,5030,5040,5045,5050,5090; anime 5070 | active |
| Radarr | `RadarrV3Proxy` `/api/v3/indexer` | v3 ≥ 3.1.1, v4 ≥ 4.0.4 | 2000–2090 | active |
| Lidarr | `LidarrV1Proxy` `/api/v1/indexer` | 1.0.2 | 3000,3010,3030,3040,3050,3060 | active |
| Readarr | `ReadarrV1Proxy` `/api/v1/indexer` | — | 3030,7000–7060 | **Readarr project retired (2025)**. Legacy candidate |
| Whisparr | `WhisparrV3Proxy` `/api/v3/indexer` | — | 6000–6090 | niche/adult |
| Mylar | `MylarV3Proxy` legacy `GET /api?cmd=listProviders/addProvider/changeProvider/delProvider&apikey=` | — | 7030 (BooksComics) | third-party |
| LazyLibrarian | `LazyLibrarianV1Proxy` `GET /api?cmd=listNabProviders/addProvider/changeProvider/delProvider` | — | books/audio | third-party |

### 4.4 Download clients (manual grabs only)
Aria2, Deluge, Flood, **Hadouken [dead: project abandoned ~2017]**, Nzbget (has `DownloadNzbLegacy12/13` code paths for NZBGet v12/13, `NzbgetProxy.cs:85-119`), **NzbVortex [niche/macOS-only]**, **Pneumatic [dead: Kodi/XBMC add-on]**, QBittorrent (`QBittorrentProxyV1` plus V2; **v1 API is for qBittorrent < 4.1 [legacy]**), RTorrent, Sabnzbd, TorrentBlackhole, UsenetBlackhole, TorrentDownloadStation and UsenetDownloadStation (Synology), TorrentFreeboxDownload, Transmission, **UTorrent [legacy]**, **Vuze [dead: Vuze discontinued, uses Transmission RPC plugin]**. That is 18 providers.

### 4.5 Notifications (19)
Apprise, CustomScript, Discord, Email (MailKit), Gotify, **Join [dead-ish; DeviceIds setting already deprecated, `JoinSettings.cs:29`]**, MailGun, Notifiarr (WebhookBase), Ntfy, **Prowl [legacy iOS]**, PushBullet, Pushcut, Pushover, SendGrid, Signal (signal-cli REST), Simplepush, Slack, Telegram, **Twitter [X API now paid; needs OAuth1 code in `NzbDrone.Common/OAuth` and `frontend/src/oauth.html`]**, Webhook.

### 4.6 Indexer proxies
Http, Socks4 (**legacy protocol**), Socks5, FlareSolverr.

### 4.7 Grep results: "Obsolete / Deprecated / legacy"
- `[Obsolete]` on 11 indexers (above) and on `AuthenticationType.Basic` (`Core/Authentication/AuthenticationType.cs:8`).
- `LegacyUrls` / `legacylinks` mechanism: old base URLs are auto-rewritten to the current default (`IndexerBase.cs:196`, `CardigannBase.cs:826`).
- Cardigann `captcha.image` deprecated (it throws). The `noappend` filter modifier is deprecated.
- `Prowlarr.Http/REST/RestController.cs:22` marks `[Obsolete]` endpoints as deprecated in `/api` info.

---

## 5. External APIs / interfaces

**Auth** (`Prowlarr.Http/Authentication/`):
- API key accepted as `?apikey=`, `X-Api-Key` header, or `Authorization: Bearer`.
- UI auth modes: Forms (cookie), Basic (obsolete), External (reverse proxy), None.
- `AuthenticationRequired`: Enabled or DisabledForLocalAddresses (with `TrustCgnatIpAddresses`).
- CORS policy `ApiCorsPolicy` on the Newznab endpoints.

**REST API v1** (`Prowlarr.Api.V1`, route `/api/v1/...`). Provider controllers (`ProviderControllerBase.cs`) share `GET, GET/{id}, POST, PUT/{id}, PUT bulk, DELETE/{id}, DELETE bulk, GET schema, POST test, POST testall, POST action/{name}`.

| Group | Endpoints |
|---|---|
| Indexers | `indexer` (provider CRUD), `indexer/categories` (default cat tree), `indexerstatus`, `indexerstats`, `indexer/{id}/newznab`, `indexer/{id}/download` |
| Apps | `applications` (provider CRUD; `SyncLevel`), `appprofile` (CRUD + schema) |
| Proxies | `indexerproxy` (provider CRUD) |
| Download clients | `downloadclient` (provider CRUD), `config/downloadclient` |
| Notifications | `notification` (provider CRUD) |
| Search | `GET search`, `POST search` (grab), `POST search/bulk` |
| History | `history`, `history/since`, `history/indexer` |
| Config | `config/host`, `config/ui`, `config/development` (and `config/downloadclient`) |
| System | `system/status`, `system/routes`, `system/routes/duplicate`, `system/shutdown`, `system/restart`, `system/task`, `system/backup`, `update`, `health`, `log`, `log/file`, `log/file/update`, `command` (POST/GET/DELETE), `filesystem`, `filesystem/type` |
| Misc | `tag`, `tag/detail`, `customfilter`, `localization`, `localization/options` |
| Unversioned | `GET /api` (api info), `/ping`, `/initialize.json`, `POST /login`, `GET /logout`, static SPA |

**Newznab/Torznab consumer API**: `/{id}/api` and `/{id}/download` (§3.4). This is the **only** interface Sonarr/Radarr use at search time.

**SignalR**: hub at `/signalr/messages` (`NzbDrone.Host/Startup.cs:296`), single client method `receiveMessage` with `{name, body:{action, resource}}`. `RestControllerWithSignalR` broadcasts model changes for indexer, application, downloadclient, notification, indexerproxy, indexerstatus, tag, task, command and health.

**Outbound services**:
- `https://prowlarr.servarr.com/v1/` (update and time)
- `https://releases.servarr.com/v1/`
- `https://indexers.prowlarr.com/{branch}/{ver}` (definitions)
- `sentry.servarr.com` (three DSNs)

Swagger is generated via Swashbuckle.

---

## 6. Background jobs / commands

Scheduled (`Core/Jobs/TaskManager.cs:66-111`):

| Command | Interval |
|---|---|
| `MessagingCleanupCommand` | 5 min |
| `ApplicationUpdateCheckCommand` | 6 h |
| `CheckHealthCommand` | 6 h (also event-triggered) |
| `HousekeepingCommand` | 24 h |
| `CleanUpHistoryCommand` | 24 h |
| `IndexerDefinitionUpdateCommand` | 24 h (also on startup) |
| `ApplicationIndexerSyncCommand` | 6 h (full sync, `removeRemote=true`) |
| `BackupCommand` | `BackupInterval` days (default 7) |

Manual/other commands: `ApplicationUpdateCommand`, `ResetApiKeyCommand`, `RestartCommand`, `ShutdownCommand`, `DeleteLogFilesCommand`, `DeleteUpdateLogFilesCommand`, `ClearLogCommand`, `ClearHistoryCommand`, `TestCommand`, `UnknownCommand`. `ApplicationIndexerSyncCommand` has a `ForceSync` flag, which the UI's "Sync App Indexers" button uses.

Event-driven work (not scheduled): app sync on provider add/update/delete (§9), and `ApiKeyChangedEvent` → full resync.

---

## 7. Configuration surface

- **config.xml** (`Configuration/ConfigFileProvider.cs`):
  - Host: `BindAddress, Port (9696), SslPort, EnableSsl, SslCertPath, SslCertPassword, UrlBase, LaunchBrowser, InstanceName, AllowedHosts, TrustedNetworks, TrustCgnatIpAddresses`.
  - Auth: `AuthenticationMethod, AuthenticationRequired, ApiKey`.
  - Logging: `LogLevel, ConsoleLogLevel, ConsoleLogFormat, LogSql, LogRotate, LogSizeLimit, LogDbEnabled, SyslogServer/Port/Level`.
  - Other: `AnalyticsEnabled, FilterSentryEvents`.
  - Updates: `Branch, UpdateAutomatically, UpdateMechanism, UpdateScriptPath`.
  - UI: `UiFolder, Theme`.
  - Postgres: `PostgresHost/Port/User/Password/MainDb/LogDb` and connection strings.
- **DB Config table** (`Configuration/ConfigService.cs`):
  - History/logging: `HistoryCleanupDays, LogIndexerResponse`.
  - UI: `FirstDayOfWeek, CalendarWeekColumnHeader, ShortDateFormat, LongDateFormat, TimeFormat, ShowRelativeDates, EnableColorImpairedMode, UILanguage`.
  - Proxy: `ProxyEnabled/Type/Hostname/Port/Username/Password/BypassFilter/BypassLocalAddresses`.
  - Other: `CertificateValidation, BackupFolder, BackupInterval, BackupRetention, ApplicationUrl, DownloadProtectionKey`.
  - Leftover keys: `RijndaelPassphrase, HmacPassphrase, RijndaelSalt, HmacSalt, PlexClientIdentifier`. **These are unused Sonarr leftovers.**
- **UI settings pages** (`frontend/src/Settings/`): Indexers (proxies), Applications (apps + Sync Profiles), DownloadClients, Notifications (Connect), Tags, General, UI, Development.
- **Per-provider settings**: dynamic via `FieldDefinition` attributes and Cardigann `settings:` blocks.

---

## 8. Platform/infrastructure concerns

- **Shared/forked NzbDrone core.** The following are essentially the Sonarr v3/v4 lineage, near-identical across Sonarr/Radarr/Lidarr/Readarr/Prowlarr and **fully duplicated** in a monolith:
  - `NzbDrone.Common`: HttpClient with `ManagedHttpDispatcher`, `HappyEyeballs`, cookie container, proxy factory, `CacheManager`, DiskProvider, ProcessProvider, EnvironmentInfo, `ServiceFactory`, Serializer, TPL `RateLimitService`, OAuth.
  - The `ThingiProvider` framework, `Messaging` (command queue/event aggregator), `Datastore` (Dapper, TableMapper, WhereBuilders, FluentMigrator), `Jobs`, `Housekeeping`, `HealthCheck`, `Backup`, `Update`, `Instrumentation`, `Localization`, `Notifications` (same provider set with fewer events), `Download/Clients` (same clients), `Authentication`.
  - The whole `Prowlarr.Http` (REST base, middleware, auth handlers), `NzbDrone.SignalR`, `NzbDrone.Host`, `NzbDrone.Update`, `NzbDrone.Mono`, `NzbDrone.Windows`, and `ServiceHelpers`.
  - The frontend Components/Store/Helpers.
- **OS-specific**:
  - `NzbDrone.Mono` (Linux/macOS/FreeBSD): disk via Mono.Posix, `/proc/mounts`, reflinks, and OS version adapters for macOS, Synology, FreeBSD, `/etc/issue` and `/etc/os-release`.
  - `NzbDrone.Windows`: disk and version info.
  - `NzbDrone` (Windows tray app `SysTrayApp`) and `NzbDrone.Console`.
  - `ServiceHelpers/ServiceInstall|ServiceUninstall` (Windows service helper exes).
  - `NzbDrone.Host/AccessControl/FirewallAdapter.cs` plus `src/Libraries/Interop.NetFwTypeLib.dll` (Windows Firewall COM) and `RemoteAccessAdapter` (URL ACLs).
  - `distribution/windows/setup` (Inno Setup) and `distribution/osx/Prowlarr.app`.
  - `OsInfo` detects Docker, Podman and containers.
- **Self-updater**:
  - `NzbDrone.Update` is a separate exe with `UpdateEngine/`: Backup/Restore, DetectApplicationType (Service/Tray/Console), Terminate/Start, and InstallUpdateService.
  - The core side is `Core/Update/*`. It checks `prowlarr.servarr.com/v1/update/{branch}`, verifies and extracts the package, and launches the updater. Mechanisms are BuiltIn/Script/External/Apt/Docker. `ReleaseBranchCheck` and `UpdateCheck` health checks cover it.
- **DB**: SQLite by default (System.Data.SQLite and SourceGear.sqlite3), or PostgreSQL (Npgsql). There are two databases, main and logs. `000_database_engine_version_check`.
- **HTTP client**: custom `NzbDrone.Common.Http.HttpClient` with interceptors (proxies, TorCache), `IndexerHttpClient` with `ExecuteProxiedAsync` (applies indexer proxies), cookie persistence, Polly retries, and `UserAgentBuilder`.
- **Things that exist only because of .NET/NzbDrone history**:
  - DryIoc plus `ImpromptuInterface`, which exists for dynamic interface proxies.
  - Hand-rolled `LazyLoaded`.
  - The namespace/assembly split `NzbDrone.*` vs `Prowlarr.*`.
  - `LongPathSupport` switches.
  - Mono naming even though it runs on .NET 8.
  - `System.Drawing.Common`.
  - The NLog `SlowRunningAsyncTargetWrapper`.
  - `RestSharp` is referenced only in tests.

---

## 9. Interactions with other *arr apps

**Mechanism (push model).** Prowlarr registers itself as a Newznab/Torznab indexer **inside** each app via that app's own REST API. After that, the apps query Prowlarr's `/{id}/api` at search/RSS time.

**Base class**: `Core/Applications/ApplicationBase.cs`. Per-app classes implement `Test`, `GetIndexerMappings`, `AddIndexer`, `UpdateIndexer`, `RemoveIndexer`.

**API calls, Sonarr as the canonical example** (`Applications/Sonarr/SonarrV3Proxy.cs`, `Sonarr.cs`). Radarr, Lidarr, Readarr and Whisparr are copy-paste variants that differ only in the API version and a few fields.

| Call | Purpose |
|---|---|
| `GET /api/v3/system/status` | version check. The `X-Application-Version` header on the test is compared to the minimum. |
| `GET /api/v3/indexer/schema` | fetch Newznab/Torznab field schema (cached 7 days per BaseUrl) |
| `GET /api/v3/indexer` | list remote indexers to discover mappings |
| `GET /api/v3/indexer/{id}` | compare before update |
| `POST /api/v3/indexer` | create. On HTTP 400 it retries with `?forceSave=true`, which bypasses the app's own indexer test (`SonarrV3Proxy.cs:94-99`) |
| `PUT /api/v3/indexer/{id}` | update (same `forceSave` retry, `:119`) |
| `DELETE /api/v3/indexer/{id}` | remove |
| `POST /api/v3/indexer/test` | app test using the fake indexer id 0 |

**What it creates in the remote app.** An indexer named `"{IndexerName} (Prowlarr)"` with:
- Implementation `Newznab` (usenet) or `Torznab` (torrent).
- `baseUrl = {ProwlarrUrl}/{indexerId}/`, `apiPath=/api`, `apiKey = Prowlarr API key`.
- `categories` = the intersection of the indexer's supported cats with the app's `SyncCategories`, and `animeCategories` likewise (Sonarr).
- `animeStandardFormatSearch`.
- For torrents: `minimumSeeders` (the indexer's `AppMinimumSeeders` override, else the AppProfile's `MinimumSeeders`), `seedCriteria.seedRatio/seedTime/seasonPackSeedTime`, and `rejectBlocklistedTorrentHashesWhileGrabbing`.
- `additionalParameters` is sent on first add only.
- `EnableRss/EnableAutomaticSearch/EnableInteractiveSearch` come from the **AppSyncProfile**, ANDed with indexer Enable. `Priority` comes from the indexer priority.
- On update it **retains remote-only fields, tags, `DownloadClientId`, and `SeasonSearchMaximumSingleEpisodeAge`**, so user edits in Sonarr survive.

**Skip rules.** No add happens if the indexer lacks tv-search/search caps, or if no category intersects. On update, if the caps or categories no longer match, the remote indexer is **deleted**.

**Mapping discovery** (`GetIndexerMappings`):
- Remote Newznab/Torznab indexers are considered only if the baseUrl starts with ProwlarrUrl **or** the apiKey equals Prowlarr's key.
- The Prowlarr id is parsed via the regex `\/(?<indexer>\d{1,3})(?:\/(?:api)?\/?)?$` (`ApplicationBase.cs:19`). **Note: `\d{1,3}` means indexer ids ≥ 1000 are never rediscovered.**
- Discovered mappings missing from the DB are inserted (to heal table wipes or manual adds).

**Sync orchestration** (`Core/Applications/ApplicationService.cs`):
- App added or updated → sync all enabled indexers to that app.
- Indexer added → `AddIndexer` on every sync-enabled app whose tags intersect.
- Indexer updated → `SyncIndexers` for **FullSync** apps only.
- Indexer deleted → `RemoveIndexer` on all sync-enabled apps.
- Bulk update → sync.
- `ApiKeyChangedEvent` → full resync with removeRemote and force.
- Scheduled `ApplicationIndexerSyncCommand` every 6 h → all apps, all indexers, `removeRemote=true`.
- **AddOnly** apps get only adds. **FullSync** apps get updates, and deletions for indexers that were removed or no longer match by tags or validity.
- **Tags**: an app with no tags gets every indexer. Otherwise the indexer must share ≥1 tag. The same rule is used for removal under FullSync.
- Errors feed `ApplicationStatusService` backoff (429 honours Retry-After, with a minimum of 1 h).

**LazyLibrarian and Mylar** use the legacy `GET /api?cmd=...&apikey=` command APIs and identify remote providers by `"{Type},{Name}"` (`RemoteIndexerName`) because those APIs lack ids (`LazyLibrarian/LazyLibrarian.cs:58-143`).

**Other interactions**:
- Apps' requests are identified in history and stats by User-Agent (`UserAgentParser.ParseSource`, e.g. "Sonarr/4.x").
- Grabs by apps go through `/{id}/download` (proxy or redirect).
- There is no reverse call (apps never call Prowlarr's management API, except Sonarr/Radarr "sync" test).
- `ApplicationBase.RequestAction` exists for UI actions.

**Monolith implication.** If the indexer layer is in-process with the TV/Movie search code, all of `Core/Applications/**` (7 apps × about 6 files: proxies, DTOs, field schemas, `AppIndexerMap`, `ApplicationStatus`, AppSyncProfile→remote flag translation, 6-hourly sync task, tag gating, mapping heal regex) becomes unnecessary. The only parts worth keeping are the *concepts* of per-consumer category filtering (SyncCategories/AnimeSyncCategories), per-consumer enable flags (RSS/auto/interactive), and seed criteria. Optionally the Newznab/Torznab server endpoint (§3.4) stays for external consumers (e.g. Lidarr/Mylar/LazyLibrarian/third parties) while in-process consumers call the search service directly.

---

## 10. Outdated / legacy candidates

| Item | Evidence |
|---|---|
| Obsolete native indexers kept as dead code: Animedia, BinSearch, CGPeers, FunFile, HDSpace, PornoLab, Rarbg, SceneTime, Shizaproject, Xthor, ZonaQ | `[Obsolete]` attrs in `Core/Indexers/Definitions/*.cs`; `IndexerFactory.cs:170`; `OutdatedDefinitionCheck.cs:32` |
| UNIT3D base: no subclasses | `Core/Indexers/Definitions/UNIT3D/*`; only `Migration/013_desi_gazelle_to_unit3d.cs` references it |
| Cardigann blocklist entries without a C# replacement (aither, blutopia, lat-team, reelflix, shareisland, danishbytes, desitorrents, …) | `IndexerDefinitionUpdateService.cs:36-55` |
| `IndexerDefinitionVersion` table/repo/service (unused) | `Core/IndexerVersions/IndexerDefinitionVersion*.cs`, `Migration/015` |
| TorrentPotato (CouchPotato API), Headphones (Headphones VIP), BrokenStones (verify) | `Definitions/TorrentPotato/`, `Definitions/Headphones/`, `BrokenStones.cs` |
| Torcache interceptor (torcache.net dead since 2016) | `Core/Http/TorcacheHttpInterceptor.cs` |
| Readarr app sync (Readarr retired) | `Core/Applications/Readarr/` |
| Mylar / LazyLibrarian legacy `cmd=` APIs | `Applications/Mylar/MylarV3Proxy.cs`, `Applications/LazyLibrarian/LazyLibrarianV1Proxy.cs` |
| Sonarr v2 message / Radarr v3 version branches | `Sonarr.cs` ("Sonarr v2 is not supported"), `RadarrV3Proxy.cs:27-153` |
| Download clients: Hadouken, Vuze, Pneumatic, uTorrent, NzbVortex; qBittorrent API v1; NZBGet v12/v13 legacy paths | `Core/Download/Clients/*`, `QBittorrentProxyV1.cs`, `NzbgetProxy.cs:85-119` |
| Notifications: Twitter (and the OAuth1 lib plus `oauth.html`), Join (deprecated DeviceIds), Prowl | `Core/Notifications/Twitter/`, `NzbDrone.Common/OAuth/`, `frontend/src/oauth.html`, `JoinSettings.cs:29` |
| Socks4 proxy | `Core/IndexerProxies/Socks4/` |
| Basic auth (obsolete) | `Core/Authentication/AuthenticationType.cs:8` |
| Sonarr config leftovers: Rijndael/Hmac passphrase and salt, PlexClientIdentifier; calendar UI settings (FirstDayOfWeek, CalendarWeekColumnHeader) in an app with no calendar; unused `RescanAfterRefreshType` enum | `Core/Configuration/ConfigService.cs`, `IConfigService.cs`, `Configuration/RescanAfterRefreshType.cs` |
| Windows-only infra: tray app, service installer exes, Firewall COM interop, Inno Setup | `src/NzbDrone/`, `src/ServiceHelpers/`, `src/Libraries/Interop.NetFwTypeLib.dll`, `NzbDrone.Host/AccessControl/`, `distribution/windows` |
| Self-updater exe and update mechanisms (irrelevant for container/Go binary) | `src/NzbDrone.Update/`, `Core/Update/` |
| Platform version adapters (Synology, FreeBSD, `/etc/issue`) | `src/NzbDrone.Mono/EnvironmentInfo/VersionAdapters/` |
| Tv quality → category heuristic used only by Nebulance | `Core/Parser/TvCategoryFromQualityParser.cs` |
| Legacy-URL rewriting lists per indexer (keep the concept but prune the data) | `LegacyUrls` in many definitions |
| Deprecated Cardigann syntax (`captcha.image`, `noappend`) | `CardigannDefinition.cs:87`, `CardigannParser.cs:513` |

---

## 11. Rewrite notes

**Behaviours to preserve**
1. **Category ids.** Standard Newznab ids, plus custom `100000 + trackerId` ids, or `100000 + first 2 bytes of SHA1(trackerCat)` for string cats (`IndexerCapabilitiesCategories.cs:57-81`). Parent→child expansion for queries (`ExpandTorznabQueryCategories`). Post-filtering of results by expanded cats, while results with **no** category are kept. Custom cats are excluded from `SupportedCategories` merges.
2. **Capability negotiation.** Search type is routed to the per-indexer supported params. Indexers that lack a mode or category are skipped rather than erroring. Newznab/Torznab upstream caps are fetched and cached, with an override allowed. `id=0` returns a fake caps/results response for app tests.
3. **Cardigann semantics.** These must be bug-compatible with the Prowlarr/Indexers YAML (and are largely shared with Jackett):
   - The regex-based template subset: if/else/range/re_replace/join, and/or/eq/ne.
   - Filter names and argument shapes.
   - `inheritinputs`, per-path `categories`, `followredirect`, `keywordsfilters`, `rows.after`/`dateheaders`/`count`/`multiple`, `missingAttributeEqualsNoResults`, and `andmatch`.
   - Login methods post/form/cookie/get/oneurl, `login.test`, `login.error`, `selectorinputs`/`getselectorinputs`.
   - simpleCaptcha auto-solve and image captcha round-trip.
   - The `download.before` + `pathselector` + `infohash` flow.
   - `legacylinks` rewrite and `requestDelay`.
   - Fuzzy date parsing (`Parser/DateTimeRoutines.cs`).
   - A Go port should consider using real `text/template` only if it remains compatible with the regex subset.
4. **Definition distribution.** YAMLs live in the **Prowlarr/Indexers** GitHub repo and are served from `https://indexers.prowlarr.com/{branch}/{schemaVersion}` (index JSON, per-file YAML, `package.zip`). Prowlarr pins **only** the schema version (`DEFINITION_VERSION = 11`) and the branch (`master`). There is no content hash or commit pin, and `IndexerDefinitionVersion.Sha` is unused. Local cache: `{AppData}/Definitions`; user overrides: `{AppData}/Definitions/Custom`. There is also a C#-supersedes-YAML blocklist.
5. **Cookie/login flows.** Cookies are persisted per indexer in `IndexerStatus` with a 30-day expiry, and re-login happens when `CheckIfLoginNeeded`. `IndexerAuth` history events are recorded. Cookie-auth indexers take the raw cookie string from the user.
6. **Cloudflare.** Detection is by header plus status plus title. FlareSolverr is used only on detection, only for tagged indexers. The UA is cached per host (in-memory, lost on restart) and solution cookies are injected per request. Global proxy settings are forwarded to FlareSolverr.
7. **Proxy download links.** Links are AES-protected with `DownloadProtectionKey`. `apikey` sits in the query string. Usenet always redirects. For torrents: `Redirect` flag, magnet detection by byte prefix, `PreferMagnetUrl`.
8. **Limits.** Query and grab limits are computed from History rows. 429 plus `Retry-After` is returned in Newznab error XML, which downstream apps rely on for their own backoff.
9. **Dedupe** by GUID with priority. `IsValidRelease` drops releases lacking size or categories for non-interactive (API) searches only.
10. **App-sync field retention** (if sync to external apps is kept): preserve remote user fields, tags and download client, and the `(Prowlarr)` name suffix. Mapping discovery uses the URL regex; fix the 3-digit id limit.

**Accidental complexity to drop**
- The entire app-sync layer in a monolith (see §9). Seven HTTP proxies, DTO copies of each app's indexer schema, a mapping table and a healing regex exist only because the apps are separate processes.
- `ThingiProvider` reflection plus `FieldDefinition` attributes → JSON schema → dynamic React forms. A monolith needs one consistent provider-settings schema mechanism shared by all apps, instead of the currently duplicated copy per app.
- Stats computed by loading all History rows in a date range into memory. `startDate` defaults to `DateTime.MinValue`, so the whole table is loaded (`IndexerStatsController.cs:31`). Use SQL aggregation instead.
- Reverse-DNS lookup of the caller on **every** API request for the `host` history field (`RequestExtensions.GetHostName`).
- Obsolete indexer classes kept only to raise health warnings. A small "retired implementation" table would do.
- Duplicate NzbDrone infra (§8): Common, Http, Datastore, Messaging, Jobs, Housekeeping, Update, Backup, Notifications, Download clients, Auth, SignalR. It is identical in spirit across Sonarr/Radarr/Prowlarr, so build it once.
- The Newznab XML server is still valuable if external consumers (Lidarr, Mylar, LazyLibrarian, AutoBrr, etc.) must be supported. Otherwise in-process search can bypass XML serialization entirely.
