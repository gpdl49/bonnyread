# Cleanuparr — Codebase Analysis

Submodule: `submodules/cleanuparr` at `v2.10.8-5-g2998442d` (commit 2998442, 2026-09-25). Paths below are relative to the submodule. Short forms:

- `Infra/` = `code/backend/Cleanuparr.Infrastructure/`
- `Persist/` = `code/backend/Cleanuparr.Persistence/`
- `Api/` = `code/backend/Cleanuparr.Api/`
- `Domain/` = `code/backend/Cleanuparr.Domain/`

**Purpose:** an external "janitor" for the *arr download queues and for torrent clients. It polls the *arr `/queue` API and the torrent clients, gives each bad download "strikes", and when a download reaches its strike limit it deletes the item through the *arr queue API (`blocklist=true&skipRedownload=true`). It then runs its own rate-limited replacement search. It also cleans up torrent clients (seeding limits, dead torrents, unlinked downloads, orphaned files on disk) and runs a scheduled "Seeker" that searches for missing items and upgrades. **Torrent only.** Usenet queue items are processed only by the failed-import rule, which reads *arr state alone.

---

## 1. Overview

**Stack**
- Backend: **.NET 10** / ASP.NET Core (`<TargetFramework>net10.0`). Main libraries:
  - EF Core 10 on SQLite or Postgres (`Npgsql.EntityFrameworkCore.PostgreSQL`, `EFCore.NamingConventions`).
  - **Quartz.NET 3.15** for scheduling.
  - **MassTransit 8.5** with only the in-memory transport.
  - Serilog, SignalR, Polly (`Microsoft.Extensions.Http.Polly`).
  - `FLM.QBittorrent` / `FLM.Transmission` client libraries.
  - `Mono.Unix` (reads the inode link count from `stat`).
  - Auth: `Otp.NET`, `BCrypt.Net-Next`, JWT, OIDC.
  - `CliWrap` (runs the Apprise CLI).
- Frontend: **Angular 22**, TypeScript 6, standalone components, zoneless, signals, Signal Forms, its own glassmorphism component set, `@microsoft/signalr`, PWA (`CLAUDE.md`).
- Docs: Docusaurus (`docs/`). E2E tests: Playwright plus a docker-compose that includes Keycloak and LazyLibrarian seeds (`e2e/`).
- Distribution: Docker image (`code/Dockerfile`, runs on `aspnet:10.0`, installs `apprise==1.9.6` into a Python venv), Windows and macOS installers, standalone executables (`.github/workflows/build-*.yml`). The default port is **11011**.

**Architecture:** clean-architecture projects:
- `Api` (controllers, Quartz job host, SignalR hub, DI)
- `Infrastructure` (all feature logic, split into `Features/*`)
- `Domain` (DTOs for the external APIs, enums)
- `Persistence` (three EF contexts)
- `Persistence.Sqlite` / `Persistence.Postgres` (migrations)
- `Shared`

Jobs are Quartz `GenericJob<T>` wrappers around `IHandler` classes (`Api/Jobs/GenericJob.cs`). Queue removals and notifications go through in-memory MassTransit queues.

**LOC (non-test, excluding migrations)**

| Project | LOC |
|---|---|
| Infrastructure | 26.6k |
| Api | 12.9k |
| Persistence | 4.7k |
| Domain | 3.1k |
| Shared | 0.5k |
| **Backend total** | **~47.8k** |
| Migrations | SQLite 78 (Data 60, Events 14, Users 4); Postgres 16 |
| Tests | ~80k (Infrastructure.Tests 61k) |
| Frontend | ~14.1k TS + 5.2k HTML + 8.2k SCSS |

The biggest files are:
- `Infra/Features/Jobs/Seeker.cs` (1102)
- `DownloadClient/DownloadService.cs` (766)
- `CustomFormatScoreSyncer.cs` (638)
- `EventPublisher.cs` (547)
- `ArrClient.cs` (513)
- `ForceImportService.cs` (472)

---

## 2. Domain model

There are three EF contexts (`Persist/DataContext.cs`, `EventsContext.cs`, `UsersContext.cs`), each with its own migration set.

**DataContext (configuration)**
- `GeneralConfig`:
  - `DryRun`
  - HTTP: `HttpMaxRetries`, `HttpTimeout` (100s), `HttpCertificateValidation`, `HttpSendUserAgent`
  - Status check: `StatusCheckEnabled`
  - Global `IgnoredDownloads`
  - Connectivity check: `ConnectivityCheckEnabled` + `ConnectivityCheckUrls`
  - `StrikeInactivityWindowHours` (default 24, max 168), `HistoryRetentionDays` (365)
  - `Log` (level, rolling size, archive)
  - `Auth` (`DisableAuthForLocalAddresses`, `TrustForwardedHeaders`, `TrustedNetworks` CIDRs)
  - Source: `Persist/Models/Configuration/General/GeneralConfig.cs`
- `ArrConfig` (one per `InstanceType`, plus a per-app override `FailedImportMaxStrikes`: -1 = use the global value, 0 = disabled) → `ArrInstance` (Name, Url, ExternalUrl, ApiKey, `Version` float used to choose Whisparr v2 or v3). Source: `Persist/Models/Configuration/Arr/*`.
- `DownloadClientConfig` (table `download_clients`) holds TypeName, Type, Host, UrlBase, credentials, ExternalUrl and a path remap (`DownloadDirectorySource`/`Target`). Supported types: qBittorrent, Deluge, Transmission, uTorrent, rTorrent.
- `QueueCleanerConfig` (cron, `FailedImport` complex type, `IgnoredDownloads`, `ProcessNoContentId`, `DownloadingMetadataMaxStrikes`) → `StallRule[]`, `SlowRule[]` (both derive from `QueueRule`).
- `ContentBlockerConfig` (Malware Blocker) has one `BlocklistSettings{Enabled, BlocklistType Blacklist|Whitelist, BlocklistPath}` per app. The apps are Sonarr, Radarr, Lidarr, Readarr, Whisparr, Sportarr and LazyLibrarian.
- `DownloadCleanerConfig` (cron, ignored downloads). The other download-cleaner configs are per client:
  - `{QBit,Deluge,Transmission,UTorrent,RTorrent}SeedingRule`
  - `UnlinkedConfig`
  - `DeadTorrentConfig`
  - `OrphanedFilesConfig`
- `BlacklistSyncConfig` + `BlacklistSyncHistory` (hash, client).
- `SeekerConfig` + `SeekerInstanceConfig` (one per ArrInstance).
- `NotificationConfig` + one provider table each for Notifiarr, Apprise, Ntfy, Pushover, Telegram, Discord and Gotify.

**EventsContext (state and audit)**
- **`DownloadItem`**: `DownloadId` (hash, unique, lower-cased by a migration), Title, `IsMarkedForRemoval`, `IsRemoved`, `IsReturning`.
- **`Strike`**: DownloadItemId, `JobRunId`, `StrikeType` (Stalled, DownloadingMetadata, FailedImport, SlowSpeed, SlowTime, DeadTorrent), CreatedAt, `LastDownloadedBytes` (used for progress detection), IsDryRun. Source: `Persist/Models/State/Strike.cs`, `DownloadItem.cs`.
- `JobRun` (type, started, completed, status).
- `AppEvent` / `ManualEvent` (audit log and "needs attention" items, e.g. `RecurringDownload`, `SearchNotTriggered`).
- `SearchQueue` (pending replacement searches).
- `SeekerHistory`, `SeekerCommandTracker`.
- `CustomFormatScoreEntry` / `CustomFormatScoreHistory`.

**UsersContext:** a single `User` with:
- password (bcrypt), TOTP and `RecoveryCode`
- Plex account link, OIDC config
- `ApiKey`
- lockout, `RefreshToken`, `UserFeatureView`

**Strike semantics** (`Infra/Features/ItemStriker/Striker.cs`)
- `StrikeAndCheckLimit(hash, name, maxStrikes, type, lastBytes)`:
  - `maxStrikes == 0` disables the rule.
  - Each call inserts one Strike row, so the count is per (item, type).
  - `count >= max` sets `IsMarkedForRemoval` and returns true.
  - `count > max` means the item came back after removal. Its hash is added to the in-memory `RecurringHashes`, and a "recurring" manual event is raised; no replacement search is done for it.
  - A new strike on an item already `IsRemoved` sets `IsReturning`.
- `ResetStrikeAsync` deletes every strike of one type for the item.
- `EventCleanupService` (every 4h) deletes all strikes of an item whose last strike is older than `StrikeInactivityWindowHours`. Strikes therefore **decay** after 24h of silence by default.
- Validation: every strike threshold that is > 0 must be **≥ 3** (`QueueRule.Validate`, `FailedImportConfig.Validate`, `QueueCleanerConfig.Validate`, `DeadTorrentConfig.Validate`).

**Caching:** `IMemoryCache` holds:
- `DownloadMarkedForRemoval` (so the same item is not queued for removal twice)
- the blocklist patterns and regexes
- force-import state (first-seen time, tries, gave-up, pending)
- uTorrent tokens

`ContextProvider` (`Infra/Features/Context/ContextProvider.cs`) is an AsyncLocal bag that carries the current job run, instance, record and client into event publishing.

---

## 3. Feature areas

### 3.1 Queue Cleaner (`Infra/Features/Jobs/QueueCleaner.cs`)

Default cron `0 0/5 * * * ?` (every 5 min). Each run:
- Skips entirely if the connectivity check fails.
- Loads the enabled Stall and Slow rules, ordered by MaxCompletion% then MinCompletion% descending.
- Walks each enabled instance of every app. For each **queue page** (`pageSize=200`), records are grouped by `downloadId`. `isPack = group.Count() > 1` and decides whether a season search or an episode search follows.

Per-record pipeline, in order:
1. Skip if `downloadId` is empty.
2. Skip if ignored. The global and job ignore lists match on download id or *arr download-client name. In the torrent client they also match on hash, category, tag and tracker domain (`QueueRecordExtensions.IsIgnored`, `QBitItemWrapper.IsIgnored`).
3. Skip if there is no content id (e.g. `EpisodeId`/`SeriesId`, or `MovieId` is 0), unless `ProcessNoContentId` is set. Without a content id the item is removed but no search follows.
4. Skip if the item is already marked for removal (cache).
5. **If the protocol contains "torrent"**, ask each torrent client `ShouldRemoveFromArrQueueAsync(hash)` until one finds it. The result carries `IsPrivate`. Checks run in this order (qBit shown: `QBitServiceQC.cs`; the other clients are equivalent):
   - **All files skipped** (every file priority = Skip): remove immediately with no strikes.
     - `AllFilesSkippedByQBit` if it is marked completed with 0 bytes downloaded (qBit's own `excluded_file_names` blocked everything).
     - Otherwise `AllFilesSkipped`.
     - `DeleteFromClient=true`.
   - **Slow** (`DownloadService.CheckIfSlow`): only when the state is "downloading" and speed > 0. Uses the single matching `SlowRule`; if several rules match, the item is skipped with a warning (`QueueRuleManager`).
   - **Stalled / metadata** (`CheckIfStuck`):
     - qBit `metaDL`/`forcedMetaDL` → `DownloadingMetadata` strike with `DownloadingMetadataMaxStrikes`. qBit only.
     - Otherwise, if the client reports stalled → the matching `StallRule` → `Stalled` strike.
6. If `ShouldRemove`, publish a removal request:
   - `removeFromClient = !changeCategory && (!isPrivate || rule.DeletePrivateTorrentsFromClient)`.
   - Private torrents therefore stay in the client by default (protects ratio and hit-and-run standing), but are still removed from the *arr and blocklisted.
7. **Force import** (`ForceImportService.TryImportAsync`, when enabled). If it tried or deferred, stop here: no strikes while the *arr works.
8. If the item is a torrent, torrent clients are enabled, it was not found in any client, and `FailedImport.SkipIfNotFoundInClient` (default true) is set: skip.
9. **Failed import** (`ArrClient.ShouldRemoveFromQueue`), see §3.1.3.
10. After the loop, `ForceImportService.ReconcileAsync` looks for pending force-imports that have left the queue. It confirms them with `/history?downloadId&eventType=3` (DownloadFolderImported), then resets the FailedImport strikes and emits `ForceImported`.

#### 3.1.1 Stall rules (`Persist/Models/Configuration/QueueCleaner/StallRule.cs`, `Infra/Services/QueueRuleEvaluator.cs`)

- Fields:
  - `Name`, `Enabled`, `MaxStrikes` (≥3, default 3)
  - `PrivacyType` Public|Private|Both
  - `MinCompletionPercentage` (exclusive when > 0) / `MaxCompletionPercentage` (inclusive)
  - `ResetStrikesOnProgress` (default true), `MinimumProgress` (ByteSize, e.g. "50MB")
  - `DeletePrivateTorrentsFromClient`, `ChangeCategory` (mutually exclusive)
- Matching: privacy + completion% band. Exactly one rule must match.
- Progress reset:
  - If `ResetStrikesOnProgress` is set, compare the current `DownloadedBytes` with `LastDownloadedBytes` on the latest Stalled strike.
  - If it grew (by ≥ `MinimumProgress` when set), all Stalled strikes are deleted.
  - Then a new strike is added, recording the current byte count.
- "Stalled" per client:
  - qBit: `stalledDL`.
  - Transmission: status 4 and `rateDownload<=0` and `eta<=0`.
  - Deluge: Downloading and speed<=0 and `eta==0`.
  - uTorrent: started+checked, no error, speed 0 and ETA 0.
  - rTorrent: `state==1 && complete==0 && down.rate<=0 && eta<=0`.
  - Sources: `*/…ItemWrapper.cs`.

#### 3.1.2 Slow rules (`SlowRule.cs`)

- Adds these fields to the QueueRule base:
  - `MinSpeed` (ByteSize/s)
  - `MaxTimeHours` (ETA ceiling)
  - `IgnoreAboveSize` (skip torrents ≥ this size)
  - `IgnoreWhileAltSpeedActive` (default true)
- At least one of `MinSpeed` or `MaxTimeHours` is required.
- Speed below `MinSpeed` → `SlowSpeed` strike. Otherwise, if `ResetStrikesOnProgress` is set, the SlowSpeed strikes are reset.
- `eta > MaxTimeHours` → `SlowTime` strike, reset the same way.
- The alt-speed probe exists for qBit (`GetAlternativeSpeedLimitsEnabledAsync`) and Transmission only.

#### 3.1.3 Failed import (`Infra/Features/Arr/ArrClient.cs:ShouldRemoveFromQueue`, `FailedImportConfig.cs`)

- Trigger (reads *arr queue fields only):
  - `trackedDownloadStatus == "warning"` **and** `trackedDownloadState ∈ {importBlocked, importPending, importFailed}`
  - **or** Lidarr `status ∈ {failed, completed}` + warning
  - **or** the edge case: `state == downloading` with a status message starting "Unable to import automatically"
- Filters:
  - `IgnorePrivate`.
  - The record must have status messages.
  - `Patterns` + `PatternMode`: the messages and titles are matched by substring, case-insensitive.
    - **Include** (default): strike only if some pattern matches. At least one pattern is required.
    - **Exclude**: never strike on a match.
- Threshold: the per-app `ArrConfig.FailedImportMaxStrikes` when it is > 0, otherwise the global `FailedImport.MaxStrikes`. 0 disables.
- Action: when removing, `removeFromClient = !ChangeCategory && (!isPrivate || DeletePrivate)`. Delete reason `FailedImport`.
- This is the only rule that also applies to **usenet** queue items.

#### 3.1.4 Force import (`Infra/Features/Arr/ForceImport/ForceImportService.cs`)

Added in migration `20260908230836_AddForceImport`. Supported for Sonarr, Radarr and Whisparr v3 (`SupportsForceImport`). It tries to import instead of deleting when a blocked import is "safe".
- Applies only when state is `importBlocked|importPending|importFailed` + warning **and every** status message contains one of:
  - "matched to series by ID"
  - "matched to movie by ID"
  - "Unable to determine if file is a sample"
- Requires a sighting in an earlier run (6h window), except for `importBlocked`.
- Waits if the *arr has a file-moving command queued or running (`RefreshMonitoredDownloads`, `ProcessMonitoredDownloads`, `DownloadedEpisodesScan`, `DownloadedMoviesScan`, `ManualImport`, `Rescan*`, `Rename*`).
- Calls `GET /api/v3/manualimport?downloadId=…&filterExistingFiles=true`. It then keeps candidates whose only rejections are "safe" and that map to the record's series/episodes or movie.
- Posts `POST /api/v3/command {name:"ManualImport", importMode:"auto", files:[…]}`.
- Tries: `ForceImportMaxTries` (default 3, max 5000) per 6h window. It then gives up for 6h, and strikes resume.

#### 3.1.5 Removal flow (`Infra/Features/Jobs/GenericHandler.cs`, `Infra/Features/DownloadRemover/QueueItemRemover.cs`)

- `PublishQueueItemRemoveRequest`:
  - sets the removal cache flag
  - publishes to the MassTransit `download-remover-queue` (`ConcurrentMessageLimit=1`)
  - emits a `DownloadMarkedForDeletion` event
- The consumer:
  - calls `DELETE /api/{v}/queue/{id}?blocklist=true&skipRedownload=true&changeCategory=…&removeFromClient=…`
  - sets `DownloadItem.IsRemoved`
  - emits `QueueItemDeleted`
  - **enqueues a `SearchQueueItem`** unless the item is recurring, `SkipSearch` is set, or `Seeker.SearchEnabled=false`
- Search item by app:

  | App | Search item |
  |---|---|
  | Sonarr / Sportarr / Whisparr v2 | Episode (single) or Season (pack) |
  | Radarr / Whisparr v3 | MovieId |
  | Lidarr | AlbumId |
  | Readarr | BookId |

- A 404 on delete gives "Item might have already been deleted by your arr". Any failure clears the removal flag.
- `changeCategory=true` uses the *arr's own "post-import category" behaviour. The *arr moves the item to that category, which suits private trackers.

### 3.2 Malware Blocker (`Infra/Features/Jobs/MalwareBlocker.cs`, `Infra/Features/MalwareBlocker/*`)

- Default cron `0/5 * * * * ?` (**every 5 seconds**). This is the only job exempt from the 30s minimum (`BackgroundJobManager.cs:154`).
- Runs only for apps with an enabled blocklist.
- For each queue record, the torrent client's `BlockUnwantedFilesAsync(hash)` (qBit version: `QBitServiceCB.cs`):
  - skips if ignored, or private while `IgnorePrivate` is set
  - reads the file list
  - files already at priority Skip count as unwanted
  - evaluates every other file name against the app's list (`FilenameEvaluator`):
    - **Blacklist**: invalid if any pattern or regex matches.
    - **Whitelist**: invalid unless some pattern or regex matches.
    - Pattern syntax: `*x*` contains, `*x` ends-with, `x*` starts-with, otherwise equals (all case-insensitive). A line prefixed `regex:` is compiled as a .NET regex.
- Actions (`DownloadService.ApplyFileBlockingAsync`):
  - Unwanted files are **set to "do not download"** in the client (qBit `SetFilePriority Skip`; the equivalent file-priority call on the other clients).
  - If **all** files are unwanted → `AllFilesBlocked`.
  - If `DeleteIfAnyFileBlocked` is set, the first hit → `AtLeastOneFileBlocked`.
  - Either way the *arr item is removed (blocklist + client delete, with the private exception controlled by `DeletePrivate`) and a replacement search is queued.
- Blocklist loading (`BlocklistProvider.cs`):
  - Sources: an HTTP(S) URL or a local file (`Helpers/FileReader.cs`).
  - Reload interval: every 4h for remote URLs; every **5 min** for `cleanuparr.pages.dev` and for local files. Also reloaded whenever the settings hash changes.
- The earlier "known malware" option was removed (`20260306144520_RemoveKnownMalwareOption`).

**Bundled blocklists** (repo root, published to Cloudflare Pages by `.github/workflows/cloudflare-pages-blocklists.yml`):

| URL | Entries | Content |
|---|---|---|
| `https://cleanuparr.pages.dev/static/blacklist` | 850 | Samples (`*sample.mkv`, `*(sample).*`), archives (`*.7z`, `*.001`, `*.ace` …), executables, scripts, office/macro formats |
| `…/blacklist_permissive` | 409 | Archives and executables only |
| `…/whitelist` | 3 | `*.avi`, `*.mp4`, `*.mkv` |
| `…/whitelist_with_subtitles` | 7 | Adds subtitle extensions |

Format: one glob per line, or a `regex:` line.

### 3.3 Blacklist Sync (`Infra/Features/BlacklistSync/BlacklistSynchronizer.cs`)

- Default cron `0 0 * * * ?` (hourly).
- Reads one pattern file (URL or path) and pushes it into **qBittorrent's `excluded_file_names` preference** (`QBitService.UpdateBlacklistAsync` → `SetPreferencesAsync`). qBittorrent then skips matching files at add time, without polling.
- Idempotent through `BlacklistSyncHistory(hash, clientId)`: a client is synced once per content hash, and old hashes are purged.
- qBittorrent only.

### 3.4 Download Cleaner (`Infra/Features/Jobs/DownloadCleaner.cs`)

Default cron hourly. Steps:
1. Log in to every client and fetch "seeding" (completed) torrents. qBit uses `filter=completed`.
2. Wait **10 seconds**.
3. Collect every `downloadId` in any *arr queue, plus hashes claimed by LazyLibrarian. These are **excluded**, so the cleaner never touches items the *arr is still tracking.
4. Per client, run in order: Unlinked → Dead torrents → Seeding rules. Then Orphaned files across all clients.

**Seeding rules** (`Persist/Models/Configuration/DownloadCleaner/*SeedingRule.cs`, `Infra/Services/SeedingRuleEvaluator.cs`, `DownloadService.CleanDownloadsAsync`)
- Selection: the first rule by `Priority` whose filters all match:
  - `Categories` (required)
  - `TrackerPatterns` (tracker-domain suffix)
  - `TagsAny`/`TagsAll` (qBit and Transmission only)
  - `PrivacyType`
- Conditions: `MaxRatio` (-1 = off), `MinSeedTime` h (used with ratio), `MaxSeedTime` h (-1 = off), `MinSeeders` (skip while the swarm has fewer seeders; not rTorrent), `MaxInactiveDays` (qBit only; skip while there was activity inside the window).
- Clean when (ratio ≥ MaxRatio **and** seedTime ≥ MinSeedTime) **or** seedTime ≥ MaxSeedTime.
- Action: `Delete` (with or without `DeleteSourceFiles`) or `Stop` (pause). Events: `DownloadCleaned` / `DownloadStopped`.

**Unlinked downloads** (`UnlinkedDownloadsService.cs`, `DownloadService.ChangeCategoryForNoHardLinksAsync`, `Features/Files/*HardLinkFileService.cs`)
- Scope: torrents in `Categories` whose files have **no other hard links**.
  - Unix: `stat.st_nlink == 1`. Windows: its own implementation.
  - `IgnoredRootDirs` handles cross-seed: links inside those dirs are counted from an inode map and subtracted.
- Action: move to `TargetCategory` (default `cleanuparr-unlinked`), or add it as a tag when `UseTag` is set and the client supports tags.
- The category is created if needed.
- Transmission "category" means the last directory segment; moving it calls `TorrentSetLocation`. rTorrent's category is `d.custom1`. Deluge uses the Label plugin.
- The intent is for a seeding rule on that category to delete the item later.

**Dead torrents** (`DeadTorrentService.cs`, added 2026-06)
- Candidates: torrents in `Categories` that are not already in `TargetCategory` (default `cleanuparr-dead`).
- Strike (`DeadTorrent`, MaxStrikes ≥3) when:
  - `SeederCount == 0`, **or**
  - the tracker reports **Unregistered** and the torrent is older than 1h.
- Seeders > 0 and registered → strikes are reset.
- At the limit → change category or tag. It is not deleted.
- Tracker health (qBit and Transmission only; everyone else reports `Unsupported`) uses `Infra/Services/TrackerMessageClassifier.cs`. It holds lists of "unregistered", "tracker down" and "inconclusive" messages, which the code says were taken from **qbit_manage** and **autobrr/qui**.

**Orphaned files** (`OrphanedFilesCleanupService.cs`, `OrphanedFilesConfig.cs`, added 2026-05)
- Config per client: `ScanDirectories`, `OrphanedDirectory`, `ExcludePatterns` (globs), `MinFileAgeHours` (24), `PurgeAfterHours` (optional).
- Claimed paths: the `SavePath` of every torrent from **all** clients (`GetAllTorrentsLite`), plus the first path segment of each of its files, remapped through `DownloadDirectorySource→Target`.
- It lists the **top-level entries only** of each scan dir. Symlinks and reparse points are skipped. An unclaimed entry older than `MinFileAgeHours` is **moved** into `OrphanedDirectory` (renamed with a timestamp on collision; falls back to copy+delete across devices).
- Entries in the orphaned dir older than `PurgeAfterHours` are deleted.
- If any client's torrent list fails or looks collapsed (`ThrowIfTorrentListCollapsed`), that client's scan is skipped.
- Validation stops scan and orphaned dirs from overlapping across clients and other clients' download dirs.

### 3.5 Seeker (search) (`Infra/Features/Jobs/Seeker.cs`, `SeekerCommandMonitor.cs`, `CustomFormatScoreSyncer.cs`)

- Interval: `SearchInterval` minutes, one of {2,3,4,5,6,10,12,15,20,30,60,120,180,240,360}, default 10. Cron comes from `SeekerConfig.ToCronExpression`. Each run adds jitter of 20% of the interval, clamped to 30–120s.
- **One replacement search per run**:
  - The oldest `SearchQueueItem` → `POST /command` (EpisodeSearch/SeasonSearch/MoviesSearch/AlbumSearch/BookSearch).
  - If a replacement ran, **proactive search is skipped** for that run.
  - This throttles re-searching after mass removals.
- **Proactive search** (`ProactiveSearchEnabled`, **Sonarr and Radarr only**). Picks one instance per run, round-robin by `LastProcessedAt`, and stops after the first instance that searches.
  - Skips an instance if its active downloads (queue items with `sizeleft>0`) reach `ActiveDownloadLimit` (default 3). With `IgnoreStruckDownloads`, items struck in the last completed QueueCleaner run are not counted.
  - Candidates:
    - `Missing`
    - `QualityCutoffNotMet` (`UseCutoff`; Radarr `movieFile.qualityCutoffNotMet`, Sonarr `episodefile.qualityCutoffNotMet`)
    - `CustomFormatScoreBelowCutoff` (`UseCustomFormatScore`; score < the profile's `cutoffFormatScore`, from the syncer cache)
  - Filters: `MonitoredOnly`, `SkipTags`, a post-release grace period (`PostReleaseGraceHours` 6, max 72), and not already in the active queue (downloading/importing/importPending/importBlocked).
  - Sonarr searches at season level: it picks the least-recently-searched qualifying season.
  - Selection strategies (`Features/Seeker/Selectors/*`): BalancedWeighted (default), OldestSearchFirst, OldestSearchWeighted, NewestFirst, NewestWeighted, Random.
  - "Cycles": each item is searched once per `CurrentCycleId`. A new cycle starts only after `MinCycleTimeDays` (7). History from older cycles is pruned after 30 days.
- **SeekerCommandMonitor** (hosted service, polls every 60s):
  - polls `GET /command/{id}`
  - a command times out at 30 min and is abandoned at 90 min
  - on completion, checks the queue for a grab of the target → `SearchItemGrabbed` event/notification
- **CustomFormatScoreSyncer** (cron `0 0/30 * * * ?`, every 30 min):
  - pages through the library in chunks of 200 with 5 concurrent fetches
  - reads the CF score for each file (Radarr `moviefile?movieFileIds=…`; Sonarr `episodefile?seriesId=`) and each profile's cutoff score
  - stores the current score and history (history kept 120 days) for the dashboard and the Seeker

### 3.6 LazyLibrarian (`Infra/Features/LazyLibrarian/*`, added 2026-08)

This is not an *arr. It is handled through its `/api?apikey&cmd=` API: `getHistory`, `getDownloadProgress`, `queueBook`, `searchBook`, `getVersion`.
- Queue items are evaluated against the torrent clients with the same QC and CB checks.
- Removal:
  - deletes the torrent from the client directly, unless LazyLibrarian "adopted" it
  - resets the book to Wanted with `queueBook` (audiobook uses `type=AudioBook`)
  - triggers `searchBook` only once `getDownloadProgress` reports -1

### 3.7 Notifications (`Infra/Features/Notifications/*`)

- Events and their per-provider flags (`NotificationEventFlags.cs`):

  | Flag | Covers |
  |---|---|
  | `OnFailedImportStrike` | failed-import strikes |
  | `OnStalledStrike` | stalled strikes |
  | `OnSlowStrike` | SlowSpeed and SlowTime strikes |
  | `OnQueueItemDeleted` | queue item deleted |
  | `OnDownloadCleaned` | seeding rule deleted a download |
  | `OnDownloadStopped` | seeding rule stopped a download |
  | `OnCategoryChanged` | category or tag changed |
  | `OnSearchTriggered` | search triggered |
  | `OnSearchItemGrabbed` | search result grabbed |
  | `OnForceImported` | force import confirmed |

  Plus Test. Metadata and DeadTorrent strikes are events only; they have no notification flag.
- Providers and endpoints:
  - **Notifiarr**: `https://notifiarr.com/api/v1/notification/passthrough/{key}`
  - **Apprise**: API mode `{url}/notify/{key}`, or CLI mode through `apprise` in the container
  - **Ntfy**: none / basic / token auth, priority
  - **Pushover**: `https://api.pushover.net/1/messages.json`, priorities -2..2
  - **Telegram**: `https://api.telegram.org/bot{token}/sendMessage|sendPhoto`
  - **Discord**: webhook
  - **Gotify**
- Delivery goes through the MassTransit `notification-queue`.

### 3.8 Events, stats and UI

- `EventPublisher` writes `AppEvent`s and pushes them over SignalR (`Api/Hubs/AppHub`). `ManualEvent`s (recurring item, search not triggered) need the user to resolve them.
- Stats: `api/v2/stats`, `Infra/Stats/*` (timeline buckets, strikes, removals, searches, health).
- UI pages (`code/frontend/src/app/app.routes.ts`):
  - dashboard, logs (live Serilog sink), events, strikes, seeker-stats (CF scores and search stats)
  - settings: general, queue-cleaner, malware-blocker, download-cleaner, blacklist-sync, seeker, `arr/:type`, download-clients, notifications, account, appearance
  - auth: login, setup, oidc/callback, plex/callback

### 3.9 Cross-cutting

- **Dry run** (`Infra/Interceptors/DryRunInterceptor.cs`): wraps every mutating call. Strikes are still recorded with `IsDryRun`.
- **Connectivity check** (`Infra/Services/ConnectivityChecker.cs`): QueueCleaner skips the run if none of the URLs respond. This avoids striking everything during an outage.
- **Health checks** (`Infra/Health/*`): every 5 min. Checks *arr `/system/status` and download-client login and version, and broadcasts over SignalR.
- **App status** (`AppStatusRefreshService`): polls `https://cleanuparr-status.pages.dev/status.json` every 10 min for the latest version.

---

## 4. Integrations (full lists)

**Arr apps** (`Domain/Enums/InstanceType.cs`, `Infra/Features/Arr/*Client.cs`)

| App | API | Client | Notes |
|---|---|---|---|
| Sonarr | v3 | `SonarrClient` | Episode/Season/Series search; force import; Seeker; CF sync |
| Radarr | v3 | `RadarrClient` | MoviesSearch; force import; Seeker; CF sync |
| Lidarr | v1 | `LidarrClient` | AlbumSearch; special failed rule (status failed/completed + warning) |
| Readarr | v1 | `ReadarrClient` | BookSearch |
| Whisparr v2 | v3 | `WhisparrV2Client : SonarrClient` | selected by `ArrInstance.Version == 2` |
| Whisparr v3 | v3 | `WhisparrV3Client` (Radarr-shaped) | force import |
| Sportarr | v3 | `SportarrClient : SonarrClient` | Sonarr-shaped |
| LazyLibrarian | `/api?cmd=` | `LazyLibrarianService` | not queue-API based |

**Download clients** (torrent only; `Domain/Enums/DownloadClientTypeName.cs`)

| Client | Transport | Metadata strike | Tags | Tracker health | Alt-speed | Min seeders | Inactivity |
|---|---|---|---|---|---|---|---|
| qBittorrent | WebUI API v2 (`FLM.QBittorrent`) | yes | yes | yes | yes | yes | yes |
| Transmission | RPC (`FLM.Transmission`) | no | labels | yes | yes | yes | no |
| Deluge | Web JSON-RPC (`DelugeClient.cs`) | no | no (Label plugin = category) | no | no | yes | no |
| µTorrent | WebUI `/gui/?token=` (`UTorrent*`) | no | no (label) | no | no | yes | no |
| rTorrent | XML-RPC (`RTorrentClient.cs`) | no | no (`d.custom1`) | no | no | no | no |

**Notification services:** Notifiarr, Apprise (API/CLI), Ntfy, Pushover, Telegram, Discord, Gotify (`Domain/Enums/NotificationProviderType.cs`).

**Blocklist sources:**
- The four Cloudflare Pages lists above.
- Any HTTP(S) URL or local file.
- Tracker-message patterns hard-coded from qbit_manage and autobrr/qui.

**Auth:** local user with password + TOTP 2FA + recovery codes, OIDC (Keycloak in e2e), Plex login, `X-Api-Key` header, trusted-network bypass (`Api/Auth/*`).

---

## 5. External APIs / interfaces

### 5.1 *arr endpoints used

The header is `X-Api-Key` (`ArrClient.SetApiKey`). `{v}` = v3 for Sonarr, Radarr, Whisparr and Sportarr; v1 for Lidarr and Readarr.

| Purpose | Method + path | Notes |
|---|---|---|
| Health | `GET /api/{v}/system/status` | |
| Queue | `GET /api/{v}/queue?page=N&pageSize=200&includeUnknownSeriesItems=true&includeSeries=true&includeEpisode=true` (Sonarr) | Radarr/Whisparr3: `includeUnknownMovieItems&includeMovie`; Lidarr: `…ArtistItems&includeArtist&includeAlbum`; Readarr: `…AuthorItems&includeAuthor&includeBook`. Fields read: `id, downloadId, title, protocol, status, trackedDownloadStatus, trackedDownloadState, statusMessages[{title,messages}], seriesId/episodeId/seasonNumber/movieId/albumId/artistId/bookId/authorId, sizeleft, downloadClient` |
| Remove | `DELETE /api/{v}/queue/{id}?blocklist=true&skipRedownload=true&changeCategory={bool}&removeFromClient={bool}` | Always blocklists; always suppresses the *arr's own redownload |
| Search | `POST /api/{v}/command` body `{name:"EpisodeSearch",episodeIds}` / `{name:"SeasonSearch",seriesId,seasonNumber}` / `{name:"SeriesSearch",seriesId}` / `{name:"MoviesSearch",movieIds}` / `{name:"AlbumSearch",albumIds}` / `{name:"BookSearch",bookIds}` | Reads the command `id` from the response |
| Command status | `GET /api/{v}/command/{id}`, `GET /api/{v}/command` | The latter checks for in-flight import/rename commands before a force import |
| Manual import candidates | `GET /api/v3/manualimport?downloadId=…&filterExistingFiles=true` | Hard-coded v3 |
| Force import | `POST /api/v3/command {name:"ManualImport",importMode:"auto",files:[{path,folderName,downloadId,releaseGroup,indexerFlags,quality,languages,seriesId,episodeIds,releaseType \| movieId}]}` | |
| Import confirmation | `GET /api/v3/history?downloadId=…&eventType=3&page=1&pageSize=1` | Reads `totalRecords` |
| Seeker library | Sonarr `GET /series`, `/series/{id}`, `/episode?seriesId=`, `/episode?episodeIds=`, `/episodefile?seriesId=`, `/episodefile?episodeFileIds=`; Radarr `GET /movie`, `/movie/{id}`, `/moviefile?movieFileIds=`; Lidarr `GET /api/v1/album?albumIds=`; Readarr `GET /api/v1/book/{id}` (the last two build search log context only) | |
| Profiles and tags | `GET /api/v3/qualityprofile`, `GET /api/v3/tag` | Cutoff score; `SkipTags` |

### 5.2 Download-client APIs used

- **qBittorrent** (via `FLM.QBittorrent`):
  - `auth/login`, `app/version`/`webapiVersion`, `transfer/info` (session), `transfer/speedLimitsMode` (alt speed)
  - `torrents/info?hashes=` / `?filter=completed` / all, `torrents/properties` (`is_private`), `torrents/trackers`, `torrents/files`
  - `torrents/filePrio` (0 = skip), `torrents/delete?deleteFiles=`, `torrents/stop`/`pause`
  - `torrents/setCategory`, `torrents/createCategory`, `torrents/categories`, `torrents/addTags`
  - `app/setPreferences {excluded_file_names}`
- **Transmission RPC** (`FLM.Transmission`):
  - `torrent-get` with fields `files, fileStats, hashString, id, eta, name, status, isPrivate, downloadedEver, downloadDir, secondsSeeding, trackerStats, uploadRatio, trackers, rateDownload, totalSize, labels, isFinished, addedDate…`
  - `torrent-set` (`files-unwanted`, `labels`), `torrent-set-location` (the "category" move), `torrent-remove`, `torrent-stop`, `session-get` (alt speed)
- **Deluge Web JSON-RPC:**
  - session: `auth.login`, `auth.delete_session`, `web.connected`, `web.get_hosts`, `web.connect`, `system.client_version`
  - torrents: `web.get_torrent_status`, `core.get_torrents_status`, `web.get_torrent_files`, `core.set_torrent_options` (file_priorities), `core.remove_torrents`, `core.pause_torrents`
  - labels: `label.get_labels`, `label.add`, `label.set_torrent`
- **µTorrent WebUI:**
  - `GET /gui/token.html` to scrape the token, with the GUID cookie
  - `list=1`, `action=getfiles`, `action=getprops`, `action=setprio`, `action=removedatatorrent`/`removetorrent`, `action=stop`, `action=setprops` (label)
- **rTorrent XML-RPC:**
  - `d.multicall2` with `d.hash=, d.name=, d.is_private=, d.size_bytes=, d.completed_bytes=, d.down.rate=, d.ratio=, d.state=, d.complete=, d.timestamp.finished=, d.base_path=, d.directory=, d.custom1=`
  - `f.multicall` (`f.path=, f.size_bytes=, f.priority=, f.completed_chunks=, f.size_chunks=`), `t.multicall` (`t.url=`)
  - `f.priority.set`, `d.custom1.set`, `d.erase`, `d.stop`, `system.client_version`

### 5.3 Own API (`Api/Features/**`, `Api/Controllers/*`)

REST under `/api`:

| Area | Routes |
|---|---|
| Configuration | `api/configuration/*` (general, queue cleaner, malware blocker, download cleaner, blacklist sync, seeker, 42 arr routes, download clients), `api/configuration/notification_providers` (24) |
| Rules and per-client configs | `api/queue-rules`, `api/seeding-rules`, `api/unlinked-config`, `api/dead-torrent-config`, `api/orphaned-files-config` |
| Seeker | `api/seeker/cf-scores`, `api/seeker/search-stats` |
| Runtime state | `api/strikes`, `api/events`, `api/manualevents`, `api/jobs` (trigger and status) |
| Health and status | `api/health`, `/health`, `api/status`, `api/v2/stats` |
| Accounts | `api/auth` (16), `api/account` (18) |

Plus the SignalR hub (logs, events, job status, health, stats).

---

## 6. Background jobs

| Job | Kind | Default schedule | Constraints |
|---|---|---|---|
| QueueCleaner | Quartz cron | `0 0/5 * * * ?` (5 min) | 30s–6h (`Shared/Helpers/Constants.cs`) |
| MalwareBlocker | Quartz cron | `0/5 * * * * ?` (5 s) | ≤6h; no minimum |
| DownloadCleaner | Quartz cron | `0 0 * * * ?` (1 h) | 30s–6h |
| BlacklistSynchronizer | Quartz cron | `0 0 * * * ?` (1 h) | 30s–6h |
| Seeker | Quartz cron | every `SearchInterval` (10 min) + 30–120s jitter | ≥1 min |
| CustomFormatScoreSyncer | Quartz cron | `0 0/30 * * * ?` (30 min) | fixed |
| SeekerCommandMonitor | HostedService | 60 s poll | 10 s startup delay |
| HealthCheckBackgroundService | HostedService | 5 min | |
| EventCleanupService | HostedService | 4 h | prunes events (`HistoryRetentionDays`), strikes (inactivity window), resolved manual events, orphan job runs |
| AppStatusRefreshService | HostedService | 10 min | external status JSON |
| OidcCleanupService | HostedService | 1 min | |
| BlocklistProvider reload | inside MalwareBlocker | 4h for remote; 5 min for cleanuparr.pages.dev and local files | |

Scheduling details (`Api/Jobs/BackgroundJobManager.cs`, `GenericJob.cs`):
- Every job is `[DisallowConcurrentExecution]`, and misfires are dropped (`WithMisfireHandlingInstructionDoNothing`).
- `UseAdvancedScheduling` switches the UI between a simple interval and a raw cron.
- Jobs can be triggered by hand through `api/jobs`.

---

## 7. Configuration surface

- **Environment variables** (`Shared/Configuration/ConfigurationKeys.cs`):
  - server: `PORT` (11011), `BIND_ADDRESS`, `BASE_PATH` (validated: starts with `/`, letters/digits/`-`/`_` only, no dots)
  - paths: `CLEANUPARR_CONFIG_PATH`, `CLEANUPARR_LOGS_PATH`
  - database: `DATABASE_PROVIDER=sqlite|postgres`, `POSTGRES_HOST/PORT/USER/PASS/DB/EXTRA_PARAMS`
  - Docker: `PUID/PGID` etc.
- **Everything else lives in the DB and is edited in the UI**, each area with its own validation. The main thresholds are:
  - strikes ≥3
  - completion % bands
  - ByteSize strings ("1MB", "500KB")
  - hours as doubles
  - cron strings
  - glob patterns
- Ignore lists are layered:
  - Global `GeneralConfig.IgnoredDownloads` + per job (QueueCleaner, MalwareBlocker, DownloadCleaner).
  - A token matches hash, category, tag, tracker domain (suffix) or *arr download-client name.
- There is a SQLite → Postgres migrator (`Infra/Features/DatabaseMigration/SqliteToPostgresMigrator.cs`).

---

## 8. Platform / infrastructure concerns

- **Single-process design:**
  - AsyncLocal `ContextProvider` state.
  - An in-memory MassTransit bus used only to serialize removals (concurrency 1).
  - A global `DataContext.Lock` semaphore when a job loads config (`GenericHandler.cs`).
  - Process-wide static `Striker.RecurringHashes`, which is lost on restart.
- **Heavy polling:**
  - Every 5s the MalwareBlocker pages every *arr queue and makes about 3–4 client calls per torrent (info, trackers, properties, files).
  - QueueCleaner does the same every 5 min.
  - This is the cost of being external. Cleanuparr has no event hook into the *arr, so it has to rediscover state.
- **Hard links:** a Unix inode count through `Mono.Unix` `stat`; Windows has a separate service. The *arr/client paths must be mounted in Cleanuparr's container at identical or remapped paths (`DownloadDirectorySource/Target`, used by orphan and unlinked scans).
- **Dependence on external sites:**
  - `cleanuparr.pages.dev` (blocklists)
  - `cleanuparr-status.pages.dev` (version)
  - `cdn.jsdelivr.net` (logo in notifications)
  - optional connectivity URLs
- **Container:** ships a Python venv for the Apprise CLI.
- **Logging:** Serilog with rolling files, an archive policy and a live UI sink.

---

## 9. Interactions with other *arr apps

**Precise semantics**
- Cleanuparr reads the *arr's **computed** tracked-download state (`trackedDownloadState`/`Status` + `statusMessages`). Its failed-import rule therefore acts on text the *arr produces.
  - Pattern matching works on localized message substrings. This is fragile if the *arr's wording or language changes.
  - "Unable to import automatically" is a special case.
- **Deletion always goes through the *arr** (`DELETE /queue/{id}`). The *arr then:
  - removes the item from the client (unless private or changing category)
  - **blocklists** the release (by hash/title/indexer)
  - writes DownloadFailed history, so the next grab avoids it
  - `skipRedownload=true` suppresses the *arr's `AutoRedownloadFailed`, and Cleanuparr queues its own search. Its reasons: throttling (one per Seeker tick), recurrence detection, and not re-searching items without a content id.
- `changeCategory=true` relies on the *arr's "post-import category" to move the torrent aside rather than delete it (useful for private trackers).
- The Download Cleaner **excludes anything still in any *arr queue**. It acts only on client items the *arr has already let go of (imported, or never tracked). This is the gap left by the *arr's "Remove Completed", which needs the torrent to be *paused* and at the ratio.
- The Seeker tops up what the *arr's RSS-only upgrade model misses: a periodic, rate-limited backlog search. It never runs while the queue is busy (`ActiveDownloadLimit`).
- Lidarr, Readarr, Whisparr and Sportarr get queue cleaning only. Force import and the proactive Seeker are Sonarr/Radarr(/Whisparr3) only.

**Rule → native *arr equivalent → monolith home**

The "native" column was checked against `submodules/sonarr` (Radarr matches).

| Cleanuparr rule / behaviour | Native Sonarr/Radarr equivalent (if any) | Monolith home |
|---|---|---|
| **Stalled** (stall rules: privacy, completion band, strikes, reset on progress, min progress bytes) | **None.** qBit `stalledDL` becomes `DownloadItemStatus.Warning` "stalled" (`Clients/QBittorrent/QBittorrent.cs:319`) and is only displayed. No timeout or auto-fail. | Tracked-download state machine: a `StalledPolicy` evaluated on each client poll. Progress is tracked natively (bytes over time), so strikes are unnecessary. |
| **Downloading metadata** (qBit metaDL, strikes) | **None.** `metaDL` → Queued if DHT is on, Warning if off (`QBittorrent.cs:329`). No timeout. | Same policy, as a "no-metadata-after T" timeout. |
| **Slow speed / slow ETA** (min speed, max ETA hours, size ceiling, alt-speed aware) | **None.** | Download-tracking policy: speed and ETA from the client poll, suppressed when the client's alt-speed mode is on. |
| **All files skipped / completed with 0 bytes** | **None directly.** The *arr later sees "no eligible files" → `importBlocked` warning. | Grab-time and poll-time check in the download tracker → fail + blocklist immediately. |
| **Failed import** (importBlocked/Pending/Failed + warning, include/exclude message patterns, per-app strikes, private safeguards) | **Partial.** The *arr sets these states and waits for manual action. It auto-*fails* only for `DangerousFile`/`ExecutableFile`/`UserRejectedExtension` rejections when the indexer's `FailDownloads` allows it (`Download/RejectedImportService.cs`). | Import pipeline: typed rejection reasons (not message text) → policy per reason: retry N times / fail + blocklist / leave for manual. |
| **Force import on "safe" rejections** (matched by ID, sample undetermined) | **Manual only** (Manual Import UI/command). | Import pipeline: treat these rejection reasons as overridable, with automatic retry, no external round-trip. |
| **Malware Blocker** (per-app blacklist/whitelist globs + regex, mark files skipped in client, delete if all or any blocked) | **Partial.** Per-indexer `FailDownloads` {PotentiallyDangerous `.arj .lnk .lzh .ps1 .scr .vbs .zipx`, Executables `.bat .cmd .exe .sh`, UserDefinedExtensions (`UserRejectedExtensions` config)}. Checked at grab from the `.torrent` file list (not magnets) in `TorrentClientBase.cs:~290-335`, and at import (`DownloadedEpisodesImportService.cs:278`). No file-level skipping, no whitelist, no sample list. | Grab decision (reject on file list when the .torrent is available) + post-metadata hook for magnets (set file priorities in the client) + import rejection. One file-policy list per library type. |
| **Blacklist sync → qBit `excluded_file_names`** | **None.** | Download-client provider setup: optionally push the file policy into qBit prefs. Cheap, and qBit then enforces it itself. |
| **Replacement search after removal** (queued, one per tick, suppressed for recurring and no-content-id items) | **Yes:** `AutoRedownloadFailed` (+ `…FromInteractiveSearch`) on DownloadFailed. It is not throttled and has no recurrence guard. | Failed-download handler: keep native redownload, add a rate limiter and a "same release keeps coming back" guard. |
| **Recurring download detection** (strike count > max after removal → manual event, no search) | **Partially**, through the blocklist (same hash or title is rejected). | Blocklist + history. Largely redundant once blocklisting is native and reliable. |
| **Seeding rules** (per category, tracker suffix, tags, privacy; max ratio, min/max seed time, min seeders, max inactive days; delete (± files) or stop) | **Partial.** Per-indexer seed ratio/time/season-pack time is pushed to the client at add. "Remove Completed" deletes only when the client has *paused* at the ratio (`QBittorrent.cs:285`). Only for items the *arr still tracks. | Seeding manager in the download subsystem. It owns the torrent lifecycle after import, keyed by indexer/tracker, not by category strings. |
| **Unlinked downloads** (no hard links → category or tag; cross-seed ignore dirs) | **None.** | Seeding manager: the monolith knows which library files came from which torrent (import records hard links). "Library file deleted or upgraded → torrent is now unlinked" is an event, not an inode scan. |
| **Dead torrents** (0 seeders or tracker "unregistered" → strikes → category) | **None.** | Seeding manager + indexer health: a tracker-message classifier shared with the indexer subsystem. |
| **Orphaned files on disk** (entries in download dirs not claimed by any torrent → move, then purge) | **None** (Sonarr "unmapped folders" covers root folders only). | Download-dir housekeeping job. It needs every client's file list plus the monolith's own import records. |
| **Proactive missing / cutoff-unmet / CF-below-cutoff search** (throttled, cycles, active-download cap, grace hours, tags) | **Manual bulk only:** `MissingEpisodeSearch`, `CutoffUnmetEpisodeSearch` (Radarr `MissingMoviesSearch`, `CutoffUnmetMoviesSearch`). No scheduled trickle. CF-score upgrades happen only via RSS. | Search scheduler: a "backlog trickle" with an indexer-budget-aware rate limit (ties into Prowlarr-merged indexer limits). |
| **CF score tracking/history** | Scores are computed natively per file and not stored as history. | Store a score-change history when a file is imported or rescanned. No sync job. |
| **Connectivity gate** (skip strikes during an outage) | **None.** | Global "network degraded" health state that pauses time-based policies. |
| **Strike decay** (inactivity window) | n/a | Unnecessary with native time-based state (first-stalled-at, last-progress-at). |
| **Private-tracker safeguards** (keep in client, change category instead of delete) | Partial: per-client "post-import category". | A first-class `isPrivate` on the tracked download; policies choose a remove / keep / relabel action. |
| **Dry run** | n/a | Useful policy "simulate" mode; cheap to keep. |

---

## 10. Outdated / legacy candidates

- **µTorrent** support (HTML token scraping, `UTorrent*` about 1.6k LOC): the client is effectively legacy. Drop it, or keep it only if Sonarr's own list keeps it.
- **Readarr** (the project is retired upstream) and **Whisparr v2**: low value.
- **LazyLibrarian and Sportarr** are niche. They are not relevant unless the monolith covers books or sports.
- **Duplicated per-client seeding-rule tables** (5 tables with near-identical columns), caused by per-client capability differences. Use one table with capability flags.
- **Message-text matching** for failed imports (`"Unable to import automatically"`, the force-import "safe reasons" strings) is brittle, and exists only because the *arr API exposes nothing better.
- Migration churn: 78 SQLite migrations in about 15 months, including remove/re-add pairs (`RemoveProcessMissingId` / `AddProcessMissingId`, `RemoveKnownMalwareOption`). There are two DB engines and three contexts for a small app.
- The 5-second MalwareBlocker default is costly polling.
- Moq is being phased out for NSubstitute (a test detail only).

---

## 11. Rewrite notes

**Becomes native download-pipeline policy in the monolith**

1. **One tracked-download state machine with time-based policies** replaces strikes. Keep per download:
   - `firstSeenAt`, `lastProgressAt`, `lastProgressBytes`
   - `stalledSince`, `metadataSince`, `belowMinSpeedSince`
   - `importBlockedSince` + attempts
   - `isPrivate`, tracker/indexer

   Policies are "condition held for ≥ T (or N polls)". This keeps Cleanuparr's useful knobs:
   - scoping by completion % band and by privacy
   - minimum-progress reset
   - ignore while alt-speed is on
   - ignore above size
   - pause everything while offline
2. **Policy actions** (a single enum): `fail+blocklist+remove`, `fail+blocklist+keep-in-client` (private), `relabel`, `stop`, `retry-import`, `notify-only`. Redownload after failure stays native, with a rate limiter and a recurrence guard.
3. **File-content policy** at three points:
   - grab (.torrent file list)
   - after metadata arrives (magnets; set file priorities in the client, as Cleanuparr does)
   - import

   One per-library-type blacklist/whitelist in the glob + `regex:` format, with the 850-entry list as the default. qBit `excluded_file_names` sync is an optional provider feature.
4. **Import-rejection policy by typed reason** instead of text patterns. "Matched by ID" and "sample undetermined" get automatic accept or retry, which replaces the ForceImport round-trip (`manualimport` GET + `ManualImport` command + history polling + 6h caches).
5. **Seeding/torrent-lifecycle manager** owning torrents after import:
   - ratio, seed-time, min-seeders and inactivity rules keyed by indexer or tracker
   - delete or stop
   - unlinked detection driven by library events (upgrade or delete of the imported file), not inode scans; cross-seed awareness (`IgnoredRootDirs`) becomes "other torrents referencing the same inode"
   - dead or unregistered detection using the shared tracker-message classifier (`TrackerMessageClassifier.cs` lists)
6. **Backlog search scheduler** (from the Seeker):
   - missing, cutoff-unmet and CF-below-cutoff
   - cycle-based fairness and selection strategies
   - post-release grace, tag skip, active-download cap
   - driven by the indexer budget
7. **Housekeeping:** an orphaned-files job for download dirs, with the same safety rules: minimum age, exclude globs, a quarantine dir with purge, and skipping the scan when a client's list fails.

**Accidental complexity that exists only because Cleanuparr is external (drop)**
- Rebuilding download ↔ media mapping from queue pagination, `includeUnknown*Items`, `HasContentId` guessing and the `isPack` heuristic (group size) → the monolith already has the tracked download ↔ episode/movie link.
- The strike table, strike decay (`StrikeInactivityWindowHours`) and the recurring-hash static set → replaced by durable timestamps on the tracked download and blocklist history.
- `skipRedownload=true` + its own `SearchQueue` + `SeekerCommandTracker` + `SeekerCommandMonitor` polling `/command/{id}` → call search in-process and observe the grab event.
- Excluding from download-client cleanup whatever is "in any *arr queue" (`_downloadsProcessedByArrs`, plus the 10s delay) → one owner of the torrent state.
- Force-import bookkeeping (first-seen, tries, gave-up and pending caches; the "is a file-moving command running" check; history `eventType=3` confirmation).
- Double polling of every client (the 5s malware scan + the 5-min queue scan + the *arr's own 1-min poll) → one poll loop feeding all policies.
- Path remapping between the Cleanuparr container and the clients (`DownloadDirectorySource/Target`) → reuse the monolith's remote path mappings.
- `CustomFormatScoreSyncer` re-reading the whole library every 30 min → store the scores when they are computed.
- Seven per-app blocklist slots and per-app clients (`WhisparrV2Client : SonarrClient`, etc.) → one domain.
- MassTransit in-memory bus, `ContextProvider` AsyncLocal, three EF contexts and double-engine migrations → plain Go goroutines, a job queue and one schema.
- The connectivity check exists partly to avoid mass strikes. With time-based state and a health subsystem, it becomes a global "degraded" flag.
