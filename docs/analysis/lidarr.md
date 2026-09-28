# Lidarr — Codebase Analysis

Source: `submodules/lidarr` @ `da7b4dfb1` (2026-09-13, version `3.1.6`, `azure-pipelines.yml:12`). Paths are relative to `submodules/lidarr/` unless noted; `Core/` = `src/NzbDrone.Core/`, `Api/` = `src/Lidarr.Api.V1/`.

Lidarr is a Sonarr fork. This report does **not** re-document the shared NzbDrone infrastructure (ThingiProvider, Datastore, Messaging, Jobs, Housekeeping pattern, HealthCheck framework, Backup, Update, Auth, Http, SignalR, Common). For that, see [sonarr.md](sonarr.md) §3/§8 and [radarr.md](radarr.md). It focuses on (1) the music domain and (2) where Lidarr diverges or lags.

---

## 1. Overview

**Purpose.** A music PVR. It monitors artists and their albums, searches indexers for album releases (and whole discographies), grabs them, then imports them. Importing works differently from the other apps: Lidarr identifies which exact MusicBrainz release and tracks the files are by scoring tags, durations and optional AcoustID fingerprints. It then optionally writes MusicBrainz tags into the files and renames or moves them. Metadata comes from **Lidarr's own metadata server** (`api.lidarr.audio`), which is built from MusicBrainz and not open to direct configuration apart from a free-text URL override.

**Stack**
- Backend: C# **.NET 8** (`global.json` SDK `8.0.421`). There are 22 `net8.0` projects and one `net8.0-windows` project (the WinForms tray app `src/NzbDrone`). This is the same runtime as Radarr and **two majors behind Sonarr (.NET 10)**.
- Libraries are the same as Radarr's (DryIoc 5.4.3, Dapper 2.1, FluentMigrator 6.2, System.Data.SQLite 2.0.4, Npgsql 9, NLog 5.5, Sentry 5.16, FluentValidation 9.5.4, Polly 8, MailKit, Ical.Net 4.3, ImageSharp 3.1, MonoTorrent 3.0.2, RestSharp 106, Swashbuckle 9), plus these music-specific ones:
  - **`TagLibSharp-Lidarr 2.2.0.27`**: Lidarr's own fork of TagLib#. It reads and writes tags and also supplies media info (codec, bitrate, sample rate, bit depth).
  - **`SpotifyAPI.Web 5.1.1`**: used by the Spotify import lists.
  - **`fpcalc`** (Chromaprint) binaries bundled per RID in `src/Runtimes/<rid>/fpcalc` (linux x64/arm/arm64/musl). They are used for AcoustID fingerprinting.
  - **No FFMpegCore/ffprobe.** Unlike Sonarr and Radarr, media info comes from TagLib.
- Frontend: React 18.3 + **Redux 4 / redux-thunk / redux-actions** (35 files in `frontend/src/Store/Actions`), react-router **5.2** plus connected-react-router, **webpack 5.95** + Babel, TypeScript 5.7, `@sentry/browser` 7, jQuery 3.7, moment, and **zero `@tanstack/react-query` usage**.
  - There are **865 `.js` files (84.6k LOC) against 569 `.ts/.tsx` files (18.6k LOC)**, so the frontend is about 82% JavaScript.
  - This makes it the **least migrated frontend** of the three PVRs. Sonarr is TS + Vite + react-query + zustand with Redux removed. Radarr is mid-migration, with 1,015 TS files against 426 JS files.
- DB: SQLite or Postgres. The main DB and log DB are separate (same as the other apps).

**LOC (C#, including blanks)**

| Project | LOC |
|---|---|
| NzbDrone.Core | 95,428 |
| Lidarr.Api.V1 | 10,865 |
| NzbDrone.Common | 13,636 |
| Lidarr.Http | 3,361 |
| NzbDrone.Host | 1,614 |
| NzbDrone.Mono / Windows / Update | 1,385 / 307 / 657 |
| NzbDrone (tray) / Console / ServiceHelpers / SignalR | 231 / 131 / 140 / 94 |
| Tests (Core.Test 49.5k, Common.Test 5.4k, Integration 2.8k, others ~3k) | ~61,000 |
| frontend/src (js+ts+tsx) | ~103,000 |

Largest Core areas: Download 18.6k, Notifications 10.9k, **MediaFiles 9.2k** (import and identification), Datastore 9.1k, Indexers 6.3k, **Music 5.7k**, ImportLists 3.9k, Parser 3.3k, Extras 3.0k, DecisionEngine 2.9k, HealthCheck 2.5k, **MetadataSource 1.1k**, **Plugins 0.6k**.

**Fork lineage**
- Git history starts at the NzbDrone initial commit (2010-09-22, `74ac3bb59`), which is Sonarr's history. There are 10,923 commits.
- The Lidarr fork began around 2017-03-30 (`a37add2f7 "Added MusicBrainz API project"`, then `a61cdeed0 "Removing the MusicBrainz API in favor of iTunes"`, a short-lived detour).
- Unlike Radarr, which has `002_103_sonarr_squashed`, Lidarr **restarted its migration numbering**: `001_initial_setup.cs` creates 30 tables from scratch.
- Namespaces are still `NzbDrone.*`. Assemblies are `Lidarr.*`.
- Sibling fork: Readarr (retired 2025) was forked *from Lidarr* and shared its `/api/v1`, MetadataProfile and "Entity + Metadata" patterns.

---

## 2. Domain model

Table mappings: `Core/Datastore/TableMapping.cs`. Music entities derive from `Core/Music/Model/Entity.cs`, which adds `UseMetadataFrom` / `UseDbFieldsFrom` / `ApplyChanges` merge semantics and memberwise equality used by the refresh diffing. Radarr and Sonarr have no equivalent generic base.

| Entity (table) | Key fields | File |
|---|---|---|
| **Artist** (`Artists`) | User-owned state: `ArtistMetadataId`, `CleanName`, `SortName`, `Monitored`, **`MonitorNewItems`** (All/None/New), `Path`, `RootFolderPath`, `QualityProfileId`, **`MetadataProfileId`**, `Tags`, `AddOptions`, `LastInfoSync`. Pass-through `Name` / `ForeignArtistId` come from Metadata. | `Core/Music/Model/Artist.cs` |
| **ArtistMetadata** (`ArtistMetadata`) | Shared, metadata-sourced (so that non-library "credit" artists on tracks can exist): `ForeignArtistId` (MusicBrainz artist MBID), **`OldForeignArtistIds`** (MBID merges), `Name`, `Aliases`, `Overview`, `Disambiguation`, `Type` (Person/Group/…), `Status` (Continuing/Ended/**Deleted**), `Images`, `Links`, `Genres`, `Ratings`, `Members` | `Core/Music/Model/ArtistMetadata.cs` |
| **Album** (`Albums`) = MusicBrainz **release group** | `ArtistMetadataId` (not ArtistId), `ForeignAlbumId` (RG MBID), `OldForeignAlbumIds`, `Title`, `Disambiguation`, `ReleaseDate`, `AlbumType` (primary type, string), `SecondaryTypes`, `Genres`, `Ratings`, `Links`, `Images`, `Monitored`, **`AnyReleaseOk`** (let the importer switch releases), `ProfileId`, `LastSearchTime` (mig 077), `AddOptions` | `Core/Music/Model/Album.cs` |
| **AlbumRelease** (`AlbumReleases`) = MusicBrainz **release** (one edition) | `AlbumId`, `ForeignReleaseId`, `OldForeignReleaseIds`, `Title`, `Status`, `Duration`, `Label[]`, `Country[]`, `Disambiguation`, `ReleaseDate`, **`Media`** (embedded `Medium{Number,Name,Format}`, e.g. CD/Vinyl/Digital), `TrackCount`, **`Monitored`** (exactly one release per album is monitored) | `Core/Music/Model/Release.cs`, `Medium.cs` |
| **Track** (`Tracks`) | `ForeignTrackId` (MB *track* id) + **`ForeignRecordingId`** (MB *recording* id; AcoustID returns these), `Old*Ids`, `AlbumReleaseId`, `ArtistMetadataId` (the track artist, which may differ from the album artist), `TrackNumber` (string, e.g. "A1"), `AbsoluteTrackNumber`, `MediumNumber`, `Title`, `Duration` (ms), `Explicit`, `TrackFileId` | `Core/Music/Model/Track.cs` |
| **TrackFile** (`TrackFiles`) | **Absolute `Path`** (Sonarr/Radarr use `RelativePath`), `Size`, **`Modified`** (mtime, mig 030, used to skip re-reading tags), `AlbumId`, `Quality`, `MediaInfo` (from TagLib), `ReleaseGroup`, `SceneName`, `IndexerFlags`, `OriginalFilePath`. **One file can map to many tracks, and `AlbumId=0` means "unmapped"** (`MediaFileRepository.GetUnmappedFiles`). | `Core/MediaFiles/TrackFile.cs` |
| **MetadataProfile** (`MetadataProfiles`), music-only | `Name`, `PrimaryAlbumTypes[]{type,allowed}`, `SecondaryAlbumTypes[]{…}`, `ReleaseStatuses[]{…}`. Controls **which release groups are pulled from metadata at all**. | `Core/Profiles/Metadata/*.cs` |
| **QualityProfile** | `Name`, `UpgradeAllowed`, `Cutoff`, `Items`, `MinFormatScore`, `CutoffFormatScore`, `FormatItems`. There is **no `MinUpgradeFormatScore`** (Sonarr/Radarr have it) and **no Language** (language profiles were removed in mig 034). | `Core/Profiles/Qualities/QualityProfile.cs` |
| **Quality** (static, 38) | Unknown, MP3-8…MP3-320 (bitrate buckets, many "for current files only"), MP3-VBR-V0/V2, AAC-192/256/320/VBR, OGG Vorbis Q5–Q10, WMA, WAV, FLAC, **FLAC 24bit**, ALAC, **ALAC 24bit** (mig 072), APE, WavPack. Groups: Trash / Poor / Low / Mid / High Quality Lossy, Lossless. No Source×Resolution×Modifier concept. | `Core/Qualities/Quality.cs:73-110,165+` |
| **RootFolder** | Adds **`DefaultMetadataProfileId`, `DefaultQualityProfileId`, `DefaultMonitorOption`, `DefaultNewItemMonitorOption`, `DefaultTags`**. These are used when a disk scan auto-adds unknown artists (mig 039). Sonarr/Radarr root folders are path-only. | `Core/RootFolders/RootFolder.cs` |
| **ImportListDefinition** | Adds `ShouldMonitor` (None / SpecificAlbum / EntireArtist), `ShouldMonitorExisting`, `MonitorNewItems`, `ShouldSearch`, `MetadataProfileId` | `Core/ImportLists/ImportListDefinition.cs` |
| **EntityHistory** (`History`) | Event types: Grabbed, ArtistFolderImported, TrackFileImported, DownloadFailed, TrackFileDeleted, TrackFileRenamed, **AlbumImportIncomplete**, DownloadImported, **TrackFileRetagged**, DownloadIgnored | `Core/History/EntityHistory.cs:39-49` |
| **LyricFile** (`LyricFiles`) | `.lrc` and similar sidecars (the music equivalent of subtitles) | `Core/Extras/Lyrics/` |
| Shared tables | Config, RootFolders, ScheduledTasks, Indexers, DownloadClients, ImportLists, Notifications, Metadata, `*Status`, QualityDefinitions, CustomFormats, NamingConfig, Blocklist, MetadataFiles, ExtraFiles, PendingReleases, RemotePathMappings, Tags, ReleaseProfiles, DelayProfiles, Users, Commands, CustomFilters, DownloadHistory, UpdateHistory, ImportListExclusions, AutoTagging | |

**Enumerations** (`Core/Music/Model/`)
- `PrimaryAlbumType`: Album(0), EP, Single, Broadcast, Other.
- `SecondaryAlbumType`: Studio(0, the synthetic "no secondary type"), Compilation, Soundtrack, Spokenword, Interview, **Audiobook(5, defined but missing from `All`, so it cannot be chosen in profiles; `SecondaryAlbumType.cs:73` vs its `All` list)**, Live, Remix, DJ-mix, Mixtape/Street, Demo, Audio drama.
- `ReleaseStatus`: Official, Promotion, Bootleg, Pseudo-Release.
- `MonitorTypes` (on add): All, Future, Missing, Existing, Latest, First, None, Unknown.
- `NewItemMonitorTypes`: All, None, New.

**Relationships**
- Artist 1–1 ArtistMetadata.
- ArtistMetadata 1–N Album. Albums hang off *metadata*, not the Artist row.
- Album 1–N AlbumRelease, with exactly one `Monitored`.
- AlbumRelease 1–N Track.
- Track N–1 ArtistMetadata (credit).
- Track N–1 TrackFile (`TrackFileId`); TrackFile N–1 Album.
- Artist N–1 QualityProfile, N–1 MetadataProfile.
- Four-level hierarchy against Sonarr's three (Series/Season/Episode) and Radarr's one.

**Migrations:** `Core/Datastore/Migration/` has **81 migrations** (000–081, with 048 missing) plus `Framework/`. Notable ones:
- 005 metadata_profiles
- 009/010 album_releases
- 023 release groups
- 028/031/032 artistmetadata split + old ids + aliases
- 034 remove_language_profiles
- 042 remove_album_folders
- 061 postgres timestamptz
- 063 custom formats
- 074 auto tagging
- 080 redacted.ch→.sh
- 081 images `url`→`remoteUrl`

---

## 3. Feature areas

One line per `Core/` top-level folder. **S** = shared NzbDrone (same concept as Sonarr/Radarr), **S\*** = shared but with music-specific changes, **M** = music-specific.

| Folder | Kind | Notes |
|---|---|---|
| Analytics | S | active-install flag |
| Annotations | S | FieldDefinition → UI schema |
| ArtistStats | M | per-artist/album track and file counts (≈ SeriesStats/MovieStats) |
| Authentication | S | None / Basic[Obsolete] / Forms / External. **No OIDC** (Sonarr has it). |
| AutoTagging | S\* | specs: Genre, **MetadataProfile**, Monitored, QualityProfile, RootFolder, Status, Tag (7; Radarr has 11) |
| Backup | S | |
| Blocklisting | S | |
| Configuration | S\* | adds fingerprinting, audio tag and metadata-source keys (§7) |
| CustomFilters | S | |
| CustomFormats | S\* | **only 4 specs**: IndexerFlag, ReleaseGroup, ReleaseTitle, Size (no Language/Source/Resolution/Edition/…) |
| Datastore | S | 81 migrations |
| DecisionEngine | S\* | adds Discography, EarlyRelease, RawDisk, SameTracks, SameTracksGrab, YearMatch, AlbumRequested, SingleAlbumSearchMatch specs |
| DiskSpace | S | |
| Download | S\* | clients, tracked downloads, CDH/failed handling; `Download/Aggregation` (RemoteAlbum) |
| Exceptions | S | |
| Extras | S\* | Metadata consumers (Kodi/Roksbox/WDTV), **Lyrics** (instead of Subtitles), Others |
| HealthCheck | S\* | 30 checks. Music-specific: **FpcalcCheck**, **RedactedGazelleCheck**, **PluginCheck**, RemovedArtistCheck |
| History | S\* | music event types |
| Housekeeping | S\* | 31 housekeepers, incl. CleanupOrphanedAlbums/Releases/Tracks/TrackFiles/ArtistMetadata |
| Http | S | proxy settings, CloudFlare detection, Torcache interceptor |
| ImportLists | S\* | music list providers (§4) + exclusions; `ShouldMonitor`/`ShouldSearch` |
| Indexers | S\* | Newznab/Torznab with `t=music` + Gazelle, Redacted, Headphones |
| IndexerSearch | M | AlbumSearch, ArtistSearch (discography), Missing / CutoffUnmet album search |
| Instrumentation | S | |
| Jobs | S | |
| Languages | S (vestigial) | 35 static languages, used only for UI/localization and RSS parse; no language decisions (mig 034) |
| Lifecycle | S | |
| Localization | S | |
| MediaCover | S\* | artist/album covers; images are served by the metadata server (`remoteUrl`) |
| **MediaFiles** | **M** | TagLib read/write (`AudioTag.cs`, `AudioTagService.cs`), **TrackImport pipeline with Identification (distance + Munkres + AcoustID)**, DiskScan, **RootFolderWatchingService** (FileSystemWatcher), retag, rename, recycle bin |
| Messaging | S | |
| **MetadataSource** | **M** | `SkyHookProxy` → `api.lidarr.audio/api/v0.4` |
| **Music** | **M** | Artist/Album/Release/Track services, refresh (entity diffing), add, move, monitor-new-albums, `AlbumYearMatcher` |
| Notifications | S\* | 25 providers + Subsonic; retag/album-delete events |
| Organizer | S\* | `FileNameBuilder` music tokens; Standard/MultiDisc track formats |
| **Parser** | **M** | album/discography title regexes, music QualityParser (codec/bitrate/bit depth), path parser, **FingerprintingService**, ISO countries |
| **Plugins** | **M** (Lidarr-only) | install/uninstall .NET plugin assemblies from GitHub releases |
| Profiles | S\* | Quality/Delay/Release + **Metadata** profiles |
| ProgressMessaging | S | |
| Qualities | S\* | audio qualities |
| Queue | S | |
| RemotePathMappings | S | |
| RootFolders | S\* | per-root add defaults |
| Security | S | |
| Tags | S | |
| ThingiProvider | S | |
| Update | S | `lidarr.servarr.com/v1/update/{branch}` |
| Validation | S | |

### 3.1 Metadata source (music-specific)

- **Hosts** (`src/NzbDrone.Common/Cloud/LidarrCloudRequestBuilder.cs:16-19`):
  - Metadata: **`https://api.lidarr.audio/api/v0.4/{route}`** (`Search` builder, keep-alive).
  - Services: **`https://lidarr.servarr.com/v1/`**, used for updates `/update/{branch}` (`Core/Update/UpdatePackageProvider.cs:38,67`), server-side notifications `/notification`, the SystemTime check and the Proxy check.
  - Override: config key **`MetadataSource`** (`ConfigService.cs:287`, exposed in `/api/v1/config/metadataprovider`). If it is set, `MetadataRequestBuilder` uses `{MetadataSource}/{route}` instead (`Core/MetadataSource/IMetadataRequestBuilder.cs:27-29`). This is the only supported way to point at a mirror or self-hosted server.
- **Routes used** (`Core/MetadataSource/SkyHook/SkyHookProxy.cs`):
  - `artist/{mbid}` returns the full artist with all release groups, releases and tracks. `albums` are then **filtered client-side by the metadata profile** (`FilterAlbums`, :137-148). Albums with no secondary type count as "Studio".
  - `album/{rgMbid}` returns the album, its releases and tracks, plus the ArtistMetadata of every credited artist.
  - `search?type=artist|album|all&query=…[&artist=…&includeTracks=1]`.
  - **`search/fingerprint`** (POST recording ids → albums).
  - `recent/artist?since=` / `recent/album?since=` (delta refresh; used only if the last full refresh was <14 days ago, `RefreshArtistService.cs:417`; 30-minute cache).
  - `chart/{listId}` (Lidarr Lists).
  - `series/{mbid}` (MusicBrainz Series list).
  - `spotify/lookup`, `spotify/artist/{id}`, `spotify/album/{id}` (Spotify → MBID mapping).
- Search accepts `lidarr:`, `lidarrid:` and `mbid:` prefixes (:435) and MusicBrainz URLs/GUIDs.
- **Refresh cadence:**
  - `RefreshArtistCommand` runs every 24h.
  - Artist: always if older than 30 days, never if refreshed within the last 12h, otherwise heuristics (`ShouldRefreshArtist.cs`).
  - Album: older than 60 days, or released within the last 30 days (`ShouldRefreshAlbum.cs`).
  - The `Entity<T>` diffing in `RefreshEntityServiceBase.cs` handles merges: `Old*Ids` lists let MBID merges on MusicBrainz be followed without losing files. A 404 marks the artist `Deleted`, which surfaces in `RemovedArtistCheck`.
- **Dependence:** there is no fallback to MusicBrainz directly. All catalogue data, search, cover images (URLs point at the Lidarr image cache), Spotify mapping and charts go through `api.lidarr.audio`. See §9 for reliability.

### 3.2 Release/album matching (indexer side)

- `Parser.ParseAlbumTitle` (`Core/Parser/Parser.cs:41-133`) is an ordered list of ~22 album regexes:
  - discography forms with start/end year ("Discography|Discografia");
  - `Artist - Album (Year)`, `Artist-Album-(Edition)-CD|WEB-Year` (scene), `Artist - Year - Album`, …
  - It extracts `ArtistName`, `AlbumTitle`, `ReleaseDate`, `Discography`, `DiscographyStart/End`.
- Hashed-release rejection (`RejectHashedReleasesRegex`), shared with the TV/movie parsers.
- `SimpleTitleRegex` is still TV/video-oriented and carries a `// TODO Rework this Regex for Music` comment (`Parser.cs:170`).
- **For search results, `ParseAlbumTitleWithSearchCriteria`** (`Parser.cs:341`) *fuzzy-locates* the known artist and album names inside the release title. It uses the bitap `FuzzyContains` from `Common/Extensions/FuzzyContains.cs` (diff-match-patch), not regexes. This is the main way search results map to albums.
- `ParsingService.GetAlbums` (`Core/Parser/ParsingService.cs:120`) looks albums up by title through `AlbumService.FindByTitle`. That method has a cascade of `FuzzyMatch` tries over clean title, title with brackets removed and title after dash (`Music/Services/AlbumService.cs:147-151`). Artist lookup works the same way, including aliases (`ArtistService.cs:117-119`). `AlbumYearMatcher` tolerates ±3 years (`AlbumYearMatchingOptions.cs:6`).
- **Quality from release title** (`Core/Parser/QualityParser.cs`):
  - It parses the codec (MP3/VBR/AAC/FLAC/ALAC/Vorbis/WMA/APE/WavPack/WAV), bitrate (`320`, `V0`, `V2`, …) and sample size (24bit).
  - For files, `ParseQuality(name, desc, fileBitrate, fileSampleSize)` prefers codec, bitrate and bit depth from TagLib media info.
  - Quality source is tracked (`QualityDetectionSource`).
- Music DecisionEngine specs (`Core/DecisionEngine/Specifications/`):
  - `DiscographySpecification`: rejects a discography if any album is unreleased.
  - `EarlyReleaseSpecification`: rejects releases posted before the album's release date.
  - `RawDiskSpecification`: rejects raw disc images.
  - `SameTracksSpecification` / `SameTracksGrabSpecification`.
  - `YearMatchSpecification`.
  - `Search/AlbumRequestedSpecification`, `SingleAlbumSearchMatchSpecification`, `ArtistSpecification`.
  - `RssSync/MonitoredAlbumSpecification`, `DeletedTrackFileSpecification`.

### 3.3 Track identification on import (the core music-specific logic)

Location: `Core/MediaFiles/TrackImport/`.

**Pipeline:** `ImportDecisionMaker.GetImportDecisions` runs these steps in order:
1. Read tags for every file with TagLib (`AudioTagService.ReadTags`). Files whose `Modified` time is unchanged skip this.
2. **Aggregate** per-file info (`Aggregation/Aggregators/AggregateFilenameInfo.cs` fills track number, title and artist from the filename when tags are missing; `AggregateQuality`; `AggregateReleaseGroup`).
3. **Group** files into candidate albums (`Identification/TrackGroupingService.cs`). Grouping goes by directory (with `CD1`/`Disc 2` subfolder merging), then by album tag, then by (artist, album) tag. It detects Various Artists.
4. **Identify** each group (`Identification/IdentificationService.cs`).
5. Run the import specs.
6. `ImportApprovedTracks` does the work: it may switch the monitored release, upgrades files, writes tags and publishes events.

**Candidate generation** (`Identification/CandidateService.cs`):
- By MBIDs in tags (release id, then release-group id).
- By DB fuzzy match of artist tag → album tag.
- Via overrides from a download (a grabbed album) or from manual import.
- If fingerprints are available, by AcoustID recording ids: DB releases whose tracks share **more than 60%** of the recording ids (`:214`). There is also a remote `search/fingerprint` call.

**Distance scoring** (a port of beets' matcher, `Identification/Distance.cs:11-31`, `DistanceCalculator.cs`). Weighted normalised penalties; `NormalizedDistance` = Σpenalty·weight / Σmax·weight.

| Key | Weight | Key | Weight |
|---|---|---|---|
| album_id (release MBID mismatch) | 5.0 | recording_id (MBID or AcoustID mismatch) | **10.0** |
| artist, album (string distance) | 3.0 each | track_title | 3.0 |
| source | 2.0 | track_artist, track_length, tracks (mean track distances) | 2.0 each |
| media_count, media_format ("Unknown" format penalised), year, track_index | 1.0 each | unmatched_tracks | 0.9 |
| missing_tracks | 0.6 | country, label, catalog_number, album_disambiguation | 0.5 each |

- Track length tolerance: **10 s grace**, after which the penalty ramps up over 30 s (`DistanceCalculator.cs:43-47`).
- Country: if not tagged, **preference order US > UK > Europe > [Worldwide]** (`:19-25`). A release with no country gets the full penalty.
- The Various Artists MBID `89ad4ac3-…` skips the artist comparison.
- Year: exact match with the album *or* release year is free; otherwise the penalty is a ratio to (now − year).
- **Track↔file assignment** uses the **Hungarian algorithm** (`Identification/Munkres.cs`, 504 LOC) over a track-distance cost matrix (`IdentificationService.cs:365`).
- **Fingerprinting** (`Core/Parser/FingerprintingService.cs`):
  - Runs bundled `fpcalc` (version-sniffed; `-json` if ≥1.4.3; 10 s timeout).
  - POSTs gzip batches to **`https://api.acoustid.org/v2/lookup`** with a **hard-coded client key `QANd68ji1L`** (:35-36), `meta=recordingids`, threshold 0.5.
  - It is triggered when there are no candidates, or when the best match has **distance > 0.15 or worst track > 0.40** (`IdentificationService.cs:155-165`).
  - Gated by `AllowFingerprinting` (Never / NewFiles / AllFiles) and surfaced by `FpcalcCheck`.
- **Acceptance thresholds** (`Specifications/CloseAlbumMatchSpecification.cs:12-13`): album distance ≤ **0.20** (i.e. ≥80% match) and worst track ≤ **0.40**. For single-track re-imports, `CloseTrackMatchSpecification` requires ≤ 0.40.
- Other import specs:
  - `NoMissingOrUnmatchedTracksSpecification`: all-or-nothing for new downloads.
  - `MoreTracksSpecification`: a new release must not have fewer tracks than existing files.
  - `SameTracksImportSpecification`.
  - `AlbumUpgradeSpecification` / `UpgradeSpecification`: album-level upgrade (the new set must be ≥ the existing quality for the album).
  - `ReleaseWantedSpecification`, `AlreadyImported`, `FreeSpace`, `NotUnpacking`, `ArtistPathInRootFolder`.
  - Rejections produce `AlbumImportIncomplete` history rather than per-file failures.
- **Release switching:** if identification picks a different AlbumRelease than the monitored one (and `AnyReleaseOk`), `ImportApprovedTracks.cs:125-131` re-monitors that release, and existing files of the old release are replaced.
- **Auto-add:** `DiskScanService.Scan(addNewArtists: true)` (triggered by `RootFolderWatchingService`) adds unknown artists and albums found on disk, using the root folder defaults (`ImportApprovedTracks.cs:355-407`). Sonarr and Radarr require explicit library import instead.
- Identification regression fixtures live in `src/NzbDrone.Core.Test/Files/Identification/*.json` (7 JSON cases: FilesWithMBIds, FilesWithoutTags, InconsistentTyposInAlbum, PenalizeUnknownMedia, PreferMissingToBadMatch, SucceedWhenManyAlbumsHaveSameTitle, CorruptFile), with sample media in `Files/Media/nin.*`.

### 3.4 Audio tags (TagLib)

- `Core/MediaFiles/AudioTag.cs` (612 LOC) reads and writes artist, album artist, title, track/disc numbers and counts, year/date/original date, genres, label, catalog number, ISRC, country, publisher, media format, **MusicBrainz ids** (release, release-group, artist, release-artist, recording, release-track, status, type, country, disambiguation comment), AcoustID, lyrics and cover art. The Picard mapping tables are referenced at `:112,375,593`.
- `AudioTagService.cs`:
  - `WriteTags` runs after import (`ImportApprovedTracks.cs:255`).
  - `SyncTags` rewrites tags on metadata refresh when `WriteAudioTags=Sync`.
  - Retag preview/diff is exposed through the `/retag` API and the frontend `Retag` page. `RetagFiles` / `RetagArtist` commands exist, along with the `TrackFileRetagged` history and notification event.
  - `ScrubAudioTags` removes all tags before writing (`:191-195`); `EmbedCoverArt` embeds cover images (`:117`).
  - `RemoveMusicBrainzTags` runs on artist/album/release delete.
- Supported extensions (`Core/MediaFiles/MediaFileExtensions.cs`): mp2, mp3, m4a, m4b, m4p, ogg, oga, opus, wma, wav, wv, flac, ape, aif/aiff/aifc. Quality for files is decided by codec, bitrate and bit depth rather than by extension.

### 3.5 Metadata profiles

- Default profiles (`Core/Profiles/Metadata/MetadataProfileService.cs`):
  - **"Standard"**: Album + Studio + Official (:138).
  - **"None"**: a reserved, non-editable profile with nothing allowed (:25,54). It is used for artists added only because a single album was added. Their discography is never pulled; renamed variants are called `None.N`.
- A profile filters release groups at fetch time (§3.1). Changing it and refreshing removes or adds albums from the DB, which `CleanupOrphanedAlbums` then cleans up. The metadata server returns every release group, and filtering is local.
- AutoTagging has a `MetadataProfileSpecification`.

### 3.6 Naming tokens

`Core/Organizer/FileNameBuilder.cs`, `NamingConfig.cs`. There are 3 formats:
- `StandardTrackFormat`, default `{Album Title} ({Release Year})/{Artist Name} - {Album Title} - {track:00} - {Track Title}`.
- `MultiDiscTrackFormat`, default `…/{Medium Format} {medium:00}/…`.
- `ArtistFolderFormat` = `{Artist Name}`.

The album folder is part of the *track* format (album folders were removed in mig 042). Plus `RenameTracks`, `ReplaceIllegalCharacters` and `ColonReplacementFormat` (mig 068).

Tokens:
- **Artist:** `{Artist Name|NameThe|CleanName|CleanNameThe|NameFirstCharacter|Disambiguation|Genre|MbId}`.
- **Album:** `{Album Title|TitleThe|CleanTitle|CleanTitleThe|Type|Disambiguation|Genre|MbId}`, `{Release Year}`.
- **Medium:** `{Medium Name}`, `{Medium Format}`, `{medium:00}`.
- **Track:** `{track:00}`, `{Track Title|CleanTitle}`, `{Track ArtistName|ArtistNameThe|ArtistCleanName|ArtistCleanNameThe|ArtistMbId}`.
- **Quality:** `{Quality Full|Title|Proper|Real}`.
- **MediaInfo:** `{MediaInfo AudioCodec|AudioBitRate|AudioChannels|AudioBitsPerSample|AudioSampleRate}`.
- **Other:** `{Release Group}`, `{Custom Format(s)}`, `{Original Title}`, `{Original Filename}`, `{ellipsis}`.

The dead `SeasonEpisodePatternRegex` is inherited from Sonarr (`FileNameBuilder.cs:46`).

### 3.7 Search criteria

- `Core/IndexerSearch/Definitions/`:
  - `AlbumSearchCriteria` has `AlbumTitle`, `AlbumYear`, `Disambiguation`; `AlbumQuery` = title + `+disambiguation`.
  - `ArtistSearchCriteria` is a discography search.
  - Base: `Artist`, `Albums`, `Tracks`, clean query (drops leading "the", normalises quotes, non-word → `+`).
- Newznab/Torznab (`Indexers/Newznab/NewznabRequestGenerator.cs:44-175`):
  - If caps advertise `audio-search` with `artist` + `album`, it sends `t=music&artist=…&album=…` (`&year=` when artist==album, i.e. self-titled).
  - Otherwise it sends `t=search&q=artist+album[+year]`.
  - Artist search uses `&artist=` or `q=`.
  - `TextSearchEngine`/`AudioTextSearchEngine` = `raw` switch.
  - Default categories: **3000, 3010, 3030, 3040** (`NewznabSettings.cs:56`).
- Gazelle/Redacted query `ajax.php?action=browse` with artist/album/year and format filters.
- `MissingAlbumSearch` / `CutoffUnmetAlbumSearch` iterate albums. There is no automatic "switch to artist search when many albums are missing" (unlike Sonarr's season search), but discography releases can satisfy several albums at once.

---

## 4. Integrations (full lists)

⚠️ = obsolete or dead; ⚑ = music-only.

### Indexers (`Core/Indexers/`) — 10
- Newznab (10 hard-coded presets: DOGnzb, DrunkenSlug, NZB.su, NZBCat ⚠️ (dead), NZBFinder, NZBgeek, NZBPlanet, SimplyNZBs, Tabula Rasa, VeryCouch; `Newznab.cs:49-58`), Torznab, Torrent RSS.
- ⚑ **Gazelle API** (generic Gazelle trackers, preset **Orpheus** `https://orpheus.network`, `Gazelle.cs:70`; cookie-login auth cache).
- ⚑ **Redacted** (API key, `https://redacted.sh`; mig 080 rewrote `.ch`; `RedactedGazelleCheck` nags if Redacted is configured through Gazelle).
- ⚑ **Headphones VIP** ⚠️ (`https://indexer.codeshy.com`, a Newznab clone tied to the Headphones project, effectively dead).
- IPTorrents, Nyaa, FileList, TorrentLeech (generic, as in Sonarr).
- EZRSS parser ⚠️ (`EzrssTorrentRssParser.cs`).

### Download clients (`Core/Download/Clients/`) — 19
SABnzbd, NZBGet, NZBVortex, Pneumatic ⚠️, Usenet Blackhole, Torrent Blackhole, qBittorrent (**V1 proxy** ⚠️ `QBittorrentProxyV1.cs` + V2), Transmission, Vuze ⚠️, Deluge, rTorrent, Flood, Aria2, uTorrent ⚠️, Hadouken ⚠️, DownloadStation (torrent + usenet), Freebox.
- Missing compared with Sonarr: **RQBit, Tribler**. Missing compared with Radarr: RQBit.
- The same obsolete clients as Sonarr and Radarr (comparison §3.1).

### Import lists (`Core/ImportLists/`) — 10 implementations
| List | Notes |
|---|---|
| ⚑ Spotify Followed Artists / Spotify Playlists / Spotify Saved Albums | OAuth via **`spotify.lidarr.audio/auth` + `/renew`** (Lidarr-hosted relay; hard-coded client id `848082790c…`, `SpotifySettingsBase.cs:26-35`); Spotify ids mapped to MBIDs through `api.lidarr.audio/spotify/*` |
| ⚑ Last.fm User / Last.fm Tag | `ws.audioscrobbler.com/2.0/`, **hard-coded API key** `204c7664…` (`LastFmUserSettings.cs:23`) |
| ⚑ Lidarr Lists | `api.lidarr.audio/chart/{id}`, with 7 preset definitions: **iTunes Top/New Albums, Apple Music Top/New Albums, Billboard Top Albums, Billboard Top Artists, Last.fm Top Artists** (`LidarrLists.cs:29-40`) |
| ⚑ MusicBrainz Series | `api.lidarr.audio/series/{mbid}` (e.g. "1001 Albums You Must Hear") |
| ⚑ Headphones ⚠️ | `{url}/api?cmd=getIndex`, a migration aid from the Headphones app (dormant project) |
| Lidarr (instance→instance) | `/api/v1` of another Lidarr |
| Custom List | generic JSON of MBIDs |

There are no Trakt, Plex, IMDb or TMDb lists. **Every music list except Last.fm and Custom depends on the Lidarr metadata server.**

### Notifications (`Core/Notifications/`) — 25
Apprise, Custom Script, Discord, Email, Emby/Jellyfin, Gotify, Join, Kodi, Mailgun, Notifiarr, ntfy, Plex Media Server (music section refresh), Prowl, Pushbullet, Pushcut, Pushover, SendGrid, Signal, Simplepush, Slack, ⚑ **Subsonic** (library scan via Subsonic API; also usable with Navidrome/Airsonic), Synology Indexer, Telegram, **Twitter ⚠️**, Webhook.
- There is no Trakt.
- Events: Grab, ReleaseImport, Upgrade, Rename, ArtistAdd, ArtistDelete, AlbumDelete, HealthIssue, HealthRestored, ApplicationUpdate, DownloadFailure, **ImportFailure**, **TrackRetag** (`INotification.cs:12-37`).
- Join has a deprecated "Device IDs" field (`JoinSettings.cs:29`) and still uses `cdn.rawgit.com` ⚠️ (dead CDN) for icons (`JoinProxy.cs:96`).

### Metadata consumers (`Core/Extras/Metadata/Consumers/`) — 3
- Kodi/XBMC (`artist.nfo`, `album.nfo`, `folder.jpg`/cover/disc images; `XbmcMetadata.cs`).
- Roksbox ⚠️.
- WDTV ⚠️.

### Other external services
- AcoustID (`api.acoustid.org/v2/lookup`).
- MusicBrainz links (Discord embeds link `musicbrainz.org/artist/{mbid}`).
- GitHub API for plugins (`api.github.com/repos/{owner}/{repo}/releases`, `PluginService.cs:52`).
- Sentry (`sentry.servarr.com` projects 15/16/20, `NzbDroneLogger.cs:75-81`).
- `wiki.servarr.com/lidarr/*`.

---

## 5. External APIs / interfaces

- **REST `/api/v1`** (`src/Lidarr.Api.V1`, 68 controllers, `openapi.json` checked in). Sonarr/Radarr use v3 (Sonarr also v5). There is **only one API version**, with no v3/v5 compatibility layer.
- Auth: `X-Api-Key` header or `?apikey=` (`Lidarr.Http/Authentication/AuthenticationBuilderExtensions.cs:55-60`).
- Controllers, grouped:
  - **Library:** `artist`, `artist/editor`, `artist/lookup`, `album`, `album/lookup`, `albumstudio` (bulk monitoring, ≈ Sonarr SeasonPass), `track`, `trackfile` (incl. unmapped), `rename`, **`retag`**, `manualimport`, `mediacover`, `calendar` + iCal feed, `wanted/missing`, `wanted/cutoff`, `search` (combined artist+album lookup).
  - **Pipeline:** `release`, **`release/push`** (autobrr), `queue`, `queue/details`, `queue/status`, `history`, `blocklist`, `command`.
  - **Profiles/config:** `qualityprofile`(+schema), **`metadataprofile`(+schema)**, `delayprofile`(+schema), `releaseprofile`, `customformat`, `qualitydefinition`, `autotagging`, `indexerflag`, `language`, `tag`, `tag/detail`, `rootfolder`, `remotepathmapping`, `customfilter`, `config/{host,ui,naming,mediamanagement,indexer,downloadclient,`**`metadataprovider`**`}`.
  - **Providers:** `indexer`, `downloadclient`, `importlist`, `importlistexclusion`, `notification`, `metadata`. Each has the usual `/schema`, `/test`, `/testall`, `/action/{name}` and bulk endpoints.
  - **System:** `system/status`, `health`, `diskspace`, `filesystem`, `log`, `log/file`, `log/file/update`, `system/backup`, `system/task`, `update`, `localization`, `parse`, **`system/plugins`**.
- **SignalR** `/signalr/messages` (same hub as the other apps).
- **Commands** (music-specific ones in bold):
  - **AlbumSearch, ArtistSearch, MissingAlbumSearch, CutoffUnmetAlbumSearch**
  - **RefreshArtist, RefreshAlbum, BulkRefreshArtist, MoveArtist, BulkMoveArtist, RenameArtist, RenameFiles, RetagArtist, RetagFiles**
  - **DownloadedAlbumsScan** (for nzbToMedia and SAB post-processing), **RescanFolders**, ManualImport
  - **InstallPlugin, UninstallPlugin**
  - Plus the shared set (RssSync, ImportListSync, Backup, CheckHealth, Housekeeping, ApplicationUpdate, …). `CheckForFinishedDownload` is deprecated and logs a warning (`DownloadMonitoringService.cs:162`).
- **Webhook** (`Core/Notifications/Webhook/`):
  - Event types: Test, Grab, Download, DownloadFailure, ImportFailure, Rename, ArtistAdd, ArtistDelete, AlbumDelete, Health, **Retag**, ApplicationUpdate, HealthRestored.
  - Payloads carry `WebhookArtist` (MBID), `WebhookAlbum`, `WebhookRelease`, `WebhookTrack` and `WebhookTrackFile`.
- **Custom script env vars** (`Core/Notifications/CustomScript/CustomScript.cs`), prefix `Lidarr_`:
  - `EventType`, `InstanceName`, `ApplicationUrl`
  - `Artist_{Id,Name,Title,Path,MBId,Type,Genres,Tags,DeletedFiles}`
  - `Album_{Id,Title,MBId,Overview,ReleaseDate}`, `AlbumRelease_MBId`
  - `Release_{Title,Indexer,IndexerFlags,Size,Quality,QualityVersion,ReleaseGroup,CustomFormat,CustomFormatScore,AlbumCount,AlbumMBIds,AlbumTitles,AlbumOverviews,AlbumReleaseDates}`
  - `Download_{Client,Client_Type,Id}`
  - `TrackFile_{Id,Ids,Path,Paths,PreviousPaths,Quality,QualityVersion,ReleaseGroup,SceneName,TrackCount,TrackIds,TrackNumbers,TrackTitles}`, `AddedTrackPaths`, `DeletedPaths`, `DeletedDateAdded`
  - **`Tags_Diff`, `Tags_Scrubbed`** (retag event)
  - `Health_Issue_*`, `Health_Restored_*`, `Update_{Message,NewVersion,PreviousVersion}`

---

## 6. Background jobs

`Core/Jobs/TaskManager.cs:60-127`:

| Task | Interval |
|---|---|
| RefreshMonitoredDownloads | 1 min |
| MessagingCleanup | 5 min |
| **ImportListSync** | 5 min tick; each list has its own `MinRefreshInterval` (`FetchAndParseImportListService.cs:54`) |
| RssSync | config `RssSyncInterval` (default 15 min, `ConfigService.cs:107`) |
| ApplicationUpdateCheck | 6 h |
| CheckHealth | 6 h |
| **RefreshArtist** | 24 h (delta via `recent/*` if the last run was <14 days ago; per-entity ShouldRefresh rules) |
| **RescanFolders** | 24 h (full library tag scan; skips files whose mtime is unchanged) |
| Housekeeping | 24 h (31 housekeepers) |
| Backup | config `BackupInterval` (default 7 days, `ConfigService.cs:441`) |

Event-driven:
- **RootFolderWatchingService** runs a FileSystemWatcher on every root folder when `WatchLibraryForChanges` is set. It debounces changes and pushes `RescanFolders(addNewArtists: true)`. This is Lidarr-only.
- `ArtistAddedHandler` / `AlbumAddedHandler` trigger refresh and search.
- `MonitorNewAlbumService` applies `MonitorNewItems` to albums that appear on refresh.
- Plugin update check runs on `system/plugins` access (`PluginService.cs:133`).

---

## 7. Configuration surface (music-specific parts)

`Core/Configuration/IConfigService.cs`, `ConfigService.cs`:

- **Metadata** (`/config/metadataprovider`):
  - `MetadataSource` (custom metadata server URL; empty means `api.lidarr.audio`).
  - **`WriteAudioTags`**: No / NewFiles / AllFiles / Sync (`WriteAudioTagsType.cs`).
  - **`ScrubAudioTags`** (bool).
  - **`EmbedCoverArt`** (bool).
- **Media management:**
  - **`AllowFingerprinting`**: Never / NewFiles / AllFiles.
  - **`WatchLibraryForChanges`**.
  - `RescanAfterRefresh` (Always/AfterManual/Never).
  - **`AutoUnmonitorPreviouslyDownloadedTracks`**.
  - `CreateEmptyArtistFolders`, `DeleteEmptyFolders`, `ImportExtraFiles` + `ExtraFileExtensions` (e.g. `.lrc`, `.cue`, `.log`).
  - `SkipFreeSpaceCheckWhenImporting`, `MinimumFreeSpaceWhenImporting`, `CopyUsingHardlinks`, `FileDate`, `RecycleBin*`, `SetPermissionsLinux`/`ChmodFolder`/`ChownGroup` (chown was removed in mig 045 but `ChownGroup` remains), `UseScriptImport`/`ScriptImportPath`.
- **UI:** **`ExpandAlbumByDefault`, `ExpandSingleByDefault`, `ExpandEPByDefault`, `ExpandBroadcastByDefault`, `ExpandOtherByDefault`**.
- **Naming:** `RenameTracks`, `StandardTrackFormat`, **`MultiDiscTrackFormat`**, `ArtistFolderFormat`, `ColonReplacementFormat`.
- **Per-entity:**
  - Artist: `MetadataProfileId`, `MonitorNewItems`.
  - Album: `AnyReleaseOk`, the monitored `AlbumRelease`.
  - Root folder: add defaults.
  - Import list: `ShouldMonitor`/`ShouldSearch`/`MonitorNewItems`/`MetadataProfileId`.
- **Indexer:** Newznab/Torznab `TextSearchEngine` / `AudioTextSearchEngine` (raw vs sphinx-style cleaning). Redacted and Gazelle have format/encoding category selectors (`RedactedSettings.cs:68`).
- **Dead or legacy keys:**
  - Rijndael/Hmac passphrase and salt (same as the other apps).
  - `PlexClientIdentifier`.
  - `ChownGroup` (see above).
- **Host config** (`config.xml` / `Lidarr__*` env) is the same as Sonarr/Radarr.

---

## 8. Platform / infrastructure concerns (divergences only)

- **.NET 8**, the same as Radarr. Sonarr is already on .NET 10. Lidarr has **no `RuntimePatches` project** (Sonarr's dead Mono Harmony patches are absent), but otherwise the same Host/Windows/Mono/tray/ServiceHelpers/Update projects exist, including `Host/AccessControl/FirewallAdapter.cs`, `RemoteAccessAdapter.cs` and `src/Libraries/Interop.NetFwTypeLib.dll`.
- **Runtime plugin loading** (Lidarr-only, `src/NzbDrone.Common/Composition/PluginLoader.cs`, `PluginLoadContext.cs`, `NzbDrone.Host/Bootstrap.cs:92-107,204-227`):
  - Arbitrary .NET assemblies from `{AppData}/plugins` are loaded into collectible `AssemblyLoadContext`s and scanned into DI. That means they can register indexers, download clients, import lists, notifications and so on.
  - Installation goes through `InstallPluginCommand` from any GitHub repo URL's release assets (`Core/Plugins/PluginService.cs:52`, `InstallPluginService.cs`).
  - If loading fails, the app restarts without plugins and `PluginCheck` raises an error.
  - This is how the community adds sources Lidarr lacks (e.g. Soulseek/slskd, Tidal/Deezer downloaders and alternative metadata providers). **It is an unsandboxed arbitrary-code-execution surface** that a Go monolith cannot reproduce in the same way.
- **External binary:** `fpcalc` (Chromaprint), shipped per RID in `src/Runtimes/*/fpcalc`. macOS and Windows expect it in the app folder (`FingerprintingService.cs:82-127`). `FpcalcCheck` reports if it is missing.
- **TagLib fork** (`TagLibSharp-Lidarr`): tags are read and written in-process. There is no ffprobe dependency, unlike Sonarr and Radarr.
- **FileSystemWatcher** on library roots, with inotify limits on Linux. Sonarr and Radarr don't watch.
- **TrackFile absolute paths.** Moving an artist rewrites every TrackFile row; Sonarr/Radarr store paths relative to the series or movie folder.
- **Heavier DB footprint.** Tracks and AlbumReleases rows are 10–100× the album count, and all releases of every release group are stored, not just the monitored one. Mig 079 added album statistics indexes for this reason.
- **Frontend** is the most outdated of the PVRs: Redux + webpack + react-router 5 + connected-react-router, about 82% JS, and no react-query.
- Update, Sentry and server notifications use `lidarr.servarr.com` and `sentry.servarr.com` (the same Servarr infrastructure).

---

## 9. Interactions with other *arr apps

- **Prowlarr → Lidarr:**
  - Prowlarr pushes Newznab/Torznab indexers through **`/api/v1/indexer`** (`LidarrV1Proxy`, min version 1.0.2, categories 3000/3010/3030/3040/3050/3060; see the Lidarr row of the Applications table in [prowlarr.md](prowlarr.md)).
  - Lidarr has no Prowlarr-specific code (`grep -ri prowlarr src/` finds nothing outside tests).
  - Prowlarr indexers that matter here are the Gazelle-family trackers (RED, OPS) and music usenet categories.
- **Profilarr:** **not supported.** `clients/lidarr.ts` is an empty stub and `VALID_TYPES=['radarr','sonarr']` ([profilarr.md](profilarr.md), Lidarr row of its clients table). Lidarr's CF engine is also much thinner (4 spec types), so TRaSH-style CF sets don't carry over.
- **Seerr (Overseerr/Jellyseerr):** no Lidarr support (`grep -ri lidarr submodules/seerr` finds 0 hits). Music requests are not part of Seerr.
- **Recyclarr:** no Lidarr support (0 hits).
- **Other consumers of `/api/v1`** found in sibling submodules:
  - **Cleanuparr** (queue records `QueueAlbum`, `command`; `code/backend/Cleanuparr.Domain/Entities/Arr/Queue/QueueAlbum.cs`, `InstanceType.cs`).
  - **Unpackerr** (queue polling; `frontend/src/pages/settings/StarrForm.svelte`).
  - **autobrr** (`/api/v1/release/push`).
  - Bazarr (1 incidental mention).
  - Also nzbToMedia and SAB scripts (`DownloadedAlbumsScan`), LunaSea/nzb360, Homepage and Notifiarr.
- **Lidarr → Lidarr** import list (instance to instance).
- **Hosted services it depends on:**

| Service | Used for | Risk |
|---|---|---|
| `api.lidarr.audio/api/v0.4` (LidarrAPI.Metadata + SkyHook cache over a MusicBrainz replica) | **All** artist/album/track metadata, search, fingerprint→album lookup, recent-changes feed, charts, MB series, Spotify→MBID mapping, cover image URLs | **Single point of failure, with no local fallback.** Search and add fail with `SkyHookException` "Unable to communicate with LidarrAPI… (503)" (`SkyHookProxy.cs:91-93,173-175,239`). *External knowledge, not code-verified:* during 2025 the server was degraded for long periods (MusicBrainz schema and ingest rebuild). Users relied on the `MetadataSource` override and on community mirrors or plugins. |
| `spotify.lidarr.audio/{auth,renew}` | Spotify OAuth relay | Lidarr-hosted |
| `api.acoustid.org` | fingerprint lookup (hard-coded app key) | third party, rate limited |
| `lidarr.servarr.com/v1` | updates, server notifications, time check | Servarr |
| `sentry.servarr.com` | error reporting | Servarr |
| `api.github.com` | plugin discovery | third party |

---

## 10. Outdated / legacy candidates

| Item | Evidence | Why |
|---|---|---|
| Headphones VIP indexer | `Core/Indexers/Headphones/`, default `https://indexer.codeshy.com` | Headphones project dormant; indexer effectively dead |
| Headphones import list | `Core/ImportLists/HeadphonesImport/` | Migration aid from a dead app |
| NZBCat and other stale Newznab presets | `Indexers/Newznab/Newznab.cs:49-58` | Hard-coded presets go stale |
| EZRSS parser | `Indexers/EzrssTorrentRssParser.cs` | ezRSS dead |
| Torcache interceptor, CloudFlare captcha | `Core/Http/` | Same as Sonarr/Radarr |
| uTorrent, Vuze, Hadouken, Pneumatic, qBittorrent API v1 | `Download/Clients/*`, `QBittorrentProxyV1.cs` | Same as comparison §3.1 |
| Twitter notification + OAuth1 lib + `frontend/src/oauth.html` | `Notifications/Twitter/`, `Common/OAuth/` | X API v1.1 gone |
| Join (rawgit icons), Prowl, Pushbullet, Simplepush | `Notifications/Join/JoinProxy.cs:96` (`cdn.rawgit.com`, dead since 2019) | Low value; covered by Apprise/ntfy |
| Roksbox, WDTV metadata | `Extras/Metadata/Consumers/{Roksbox,Wdtv}` | Dead devices |
| Basic auth | `Authentication/AuthenticationType.cs:8` `[Obsolete]` | |
| `CheckForFinishedDownload` command | `DownloadMonitoringService.cs:162` | Deprecated alias |
| Vestigial Languages | `Core/Languages/` (35 langs), `Api/Languages`, mig 034 removed language profiles | No behaviour; UI and RSS only |
| `SecondaryAlbumType.Audiobook` defined but excluded from `All` | `Music/Model/SecondaryAlbumType.cs:73` vs `All` | Half-removed type |
| TV leftovers | `FileNameBuilder.SeasonEpisodePatternRegex` (:46), `SearchCriteriaBase.MonitoredEpisodesOnly`, `SimpleTitleRegex` "TODO Rework this Regex for Music" (`Parser.cs:170`), mig 067 `download_clients_rename_tv_to_music` | Sonarr copy-paste debris |
| `ChownGroup`, Rijndael/Hmac keys, `PlexClientIdentifier` | `IConfigService.cs` | Unused config |
| Firewall COM interop / RegisterUrl / tray / ServiceHelpers | `NzbDrone.Host/AccessControl/`, `src/Libraries/Interop.NetFwTypeLib.dll` | Same Windows legacy as siblings |
| Frontend stack | `package.json`: redux 4, redux-thunk, connected-react-router 6.9, react-router 5.2, webpack 5, `@sentry/browser` 7, jQuery, moment; 865 JS files | Furthest behind of the PVRs |
| .NET 8 | `global.json` | Sonarr already on .NET 10 |
| Hard-coded third-party keys | AcoustID `QANd68ji1L` (`FingerprintingService.cs:36`), Last.fm `204c7664…`, Spotify client id | Shared app keys, fragile |
| Plugin loader | `Common/Composition/PluginLoader.cs` | Not "outdated", but an unsandboxed ACE surface; do not port |

---

## 11. Rewrite notes

**Music-specific logic to preserve**
1. **Four-level model**: Artist(+ArtistMetadata) → Album (release group) → AlbumRelease (edition, one monitored, `AnyReleaseOk`) → Track (track id + recording id) ← TrackFile.
   - Keep the metadata/owned-state split. It lets credited artists and non-library artists exist.
   - Keep `Old*Ids` so MusicBrainz merges don't orphan files.
   - A file may map to several tracks, and unmapped files are first-class.
2. **Metadata profiles** (primary types × secondary types × release statuses, "no secondary = Studio", reserved "None" profile for album-only adds) filter the *catalogue*. This is a concept that does not exist for TV or movies.
3. **The identification engine** is the crown jewel. It should be ported with its fixtures (`Core.Test/Files/Identification/*.json` and the `IdentificationServiceTests`):
   - tag read → filename aggregation → grouping (dir/disc folders/VA) → candidates (MBIDs, fuzzy DB, AcoustID) → weighted distance (weights and thresholds in §3.3) → Munkres assignment → accept at ≤0.20 album / ≤0.40 track;
   - release switching; all-or-nothing album import; album-level upgrade.
   - Go options: `go-taglib` / `dhowden/tag` (read-only) plus a writer (e.g. a taglib cgo binding). Munkres is small. fpcalc can stay as an external binary.
4. **Tag writing and retagging**: the MusicBrainz Picard tag mapping, No/New/All/Sync modes, scrub, embedded art, retag preview diff, and `Tags_Diff` in events.
5. **Music quality model**: codec × bitrate/VBR level × bit depth, detected from the release title *and* from file properties. Lossless/lossy tier groups. It does not fit Radarr's Source×Resolution×Modifier, so the unified quality model needs a per-media-kind quality axis.
6. **Parsing**: album/discography regex list + the fuzzy "find known artist/album inside the title" matcher for search results (`ParseAlbumTitleWithSearchCriteria`, bitap `FuzzyContains`), ±3-year matching, and discography handling (one grab covers N albums; reject if any album is unreleased).
7. **Search**: Newznab `t=music&artist=&album=` with caps detection and fallback to `q=`; Gazelle/Redacted/Orpheus JSON APIs (these can live in the Prowlarr-style indexer layer as long as they expose artist/album/year/format params).
8. **Naming**: multi-disc format, `{Medium …}`, `{Track Artist…}`, MBID tokens; album folder embedded in the track format.
9. **Library watching + auto-add from disk** using root-folder defaults.
10. **Contracts**: the `/api/v1` shape (Prowlarr sync, Cleanuparr, Unpackerr, autobrr), `Lidarr_*` env vars and webhook payloads, and the `DownloadedAlbumsScan` command.

**Shared with the unified pipeline (build once)**
- Indexer layer, download clients, tracked downloads / CDH / failed handling, queue, blocklist, pending/delay profiles, release profiles, custom formats (Lidarr's 4 specs are a subset), history, notifications, root folders, remote path mappings, tags, auto-tagging, recycle bin, housekeeping, health, backup, auth, API key, SignalR/events, update.
- The **import-list framework** (lists yield MBIDs instead of TVDB/TMDb ids).
- The **metadata-source abstraction** with a provider per media kind (TVDB proxy, TMDb proxy, MusicBrainz).
- **Decision:** whether to keep depending on `api.lidarr.audio` or talk to MusicBrainz directly (1 req/s rate limit, so a local cache or mirror is needed) plus Cover Art Archive. Given the server's reliability history, the monolith should make the music metadata provider pluggable and cache aggressively.

### Divergence from Sonarr/Radarr
- **Domain:** artist/release-group/release/track with MBIDs, against series/season/episode (TVDB) or movie (TMDb). The shared metadata tables (ArtistMetadata) resemble Radarr's MovieMetadata split but go further: albums hang off metadata.
- **Extra profile type:** MetadataProfile. There is no language profile or language CF: languages are vestigial.
- **Quality:** 38 audio qualities with tier groups, and file-property-based detection through TagLib (no ffprobe).
- **CF engine:** only IndexerFlag / ReleaseGroup / ReleaseTitle / Size. There is no `MinUpgradeFormatScore`.
- **Import:** probabilistic identification (distance, Hungarian assignment, AcoustID) against deterministic filename parsing in Sonarr/Radarr; album-atomic import; release switching; auto-add from disk; FileSystemWatcher; tag writing and retag.
- **Storage:** absolute TrackFile paths; many-tracks-per-file; unmapped files.
- **Root folders** carry add defaults. **Import lists** have SpecificAlbum/EntireArtist monitoring and search-on-add.
- **Indexers:** Gazelle/Redacted/Headphones added; Newznab `t=music`; no BTN/PTP/HDBits. **Clients:** no RQBit/Tribler.
- **Notifications:** adds Subsonic, ImportFailure and TrackRetag events; no Trakt.
- **API:** `/api/v1` only (no v3/v5); adds `metadataprofile`, `albumstudio`, `retag`, `system/plugins`, `config/metadataprovider`.
- **Plugins:** runtime assembly loading from GitHub. Unique among the three PVRs.
- **Migrations:** its own 81-step lineage starting in 2017, not a Sonarr squash.
- **Lagging copies of shared code:**
  - .NET 8 against Sonarr's .NET 10.
  - The frontend is still Redux/webpack/react-router 5 (~82% JS), against Sonarr's TS/Vite/react-query/zustand and Radarr's half-migrated TS.
  - No OIDC and no v5-style API.
  - The same dead clients and notifications as the siblings.
- **Music-specific hosted dependencies:** `api.lidarr.audio`, `spotify.lidarr.audio` and AcoustID, on top of the shared Servarr infrastructure.
