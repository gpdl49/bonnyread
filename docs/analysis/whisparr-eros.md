# Whisparr v3 "Eros" — Codebase Analysis

Source: `submodules/whisparr-eros`, branch `eros-develop`, HEAD `a99b7b5e` (2026-09-26, "New: Fall back to alternate download clients on failure (#925)", released as `v3.6.3-develop.1745`). The working tree is a shallow clone, so history facts come from `gh api repos/Whisparr/Whisparr-Eros` and `repos/Radarr/Radarr`. Paths are relative to `submodules/whisparr-eros/`, and `Core/` means `src/NzbDrone.Core/`. File-level comparisons against Radarr use `submodules/radarr` @ `c90668a5`, the same checkout that `radarr.md` analyses. v2 references point at `whisparr.md`.

**Summary.** Eros is a **Radarr fork** whose backend tracks Radarr `develop` **by cherry-pick with a written disposition for every upstream commit**. Its frontend tracks **Sonarr** `v5-develop` the same way. It is caught up with Radarr: its high-water mark is `c90668a5`, which is exactly our Radarr submodule. About 60% of Core files are byte-identical to Radarr, and about 80% of Core lines match Radarr.

The domain model is **"everything is a Movie row"**:
- `MovieMetadata.ItemType` is `Scene` or `Movie`.
- Scenes come from **StashDB**. Full-length movies come from **TMDb** or **ThePornDB** (a per-install choice).
- **Studio** and **Performer** are first-class tables with their own monitoring, quality profile, root folder, tags and an `AfterDate` cut-off. Monitoring a studio or performer pulls its "works" into the library as monitored items.

All metadata still goes through the closed-source Whisparr proxy, `api.whisparr.com/v4`. The v2 parsing and matching ideas were carried over and extended: date-first regexes, the same-date title/performer matcher, and codes. Some v2 safety properties were lost:
- Fuzzy matching picks the top score without checking for ambiguity.
- Product codes have no ID index.
- Code-based matching was removed from completed-download tracking.
- Search toggles are global, not per indexer.

There is **no v2→v3 migration path**. v2 is still maintained and is not formally deprecated.

---

## 1. Overview

### 1.1 Stack and versions
- **Backend:** C# on **.NET 10**:
  - `global.json` SDK 10.0.101. 23 projects target `net10.0`, and the tray app targets `net10.0-windows`.
  - Package set: Dapper 2.1.89, DryIoc 5.4.3, FluentMigrator 8.0.1, FluentValidation 12.1, Npgsql 10.0.3, System.Data.SQLite 2.0.4 + SourceGear.sqlite3 3.53.4, NLog 6.2, Sentry 6.11, Polly 8.8, MailKit 4.18, Swashbuckle 10.2.3, SixLabors.ImageSharp 3.1.12, MonoTorrent 3.0.2.
  - The FFmpeg bindings are `Openur.FFMpegCore`/`Openur.FFprobeStatic`, not Servarr's packages.
  - **`FuzzySharp` 2.0.2** is an addition, used for Levenshtein scene matching.
  - The package set is newer than our Radarr checkout (.NET 8, FluentMigrator 6.2).
- **Frontend: follows Sonarr, not Radarr**, and is ahead of both Radarr and v2:
  - React **19.3**, react-router **7**, `@tanstack/react-query` 5.103, **zustand** 5, TypeScript 5.9, `@microsoft/signalr` 10. It still builds with **webpack** 5.111.
  - There are **1,533 .ts/.tsx files, 0 .js files and 0 Redux**, about 108k LOC. `REDUX_MIGRATION.md` documents the migration as complete.
  - `AGENTS.md` still says "React 19 / Redux", which is stale.
- **Versioning:**
  - `WHISPARR_MAJOR_VERSION: 3` and `VERSION_FALLBACK: 3.3.4` (`.github/workflows/build_v3.yml:28-32`). The version is derived from the reachable `vX.Y.Z` tag.
  - `package.json` says `3.1.0`. The current tags are `v3.6.2-release.1727` and `v3.6.3-develop.1745`.
- **Database:**
  - SQLite `whisparr3.db` (`src/NzbDrone.Common/Extensions/PathExtensions.cs:15`) or Postgres (`whisparr-main`/`whisparr-log`), plus `logs.db`.
  - Default port **6969**, the same as v2 (`Core/Configuration/ConfigFileProvider.cs:221`).
  - The app-data folder is `Whisparr`, the same as v2 (`NzbDrone.Common/EnvironmentInfo/AppFolderInfo.cs:36`). v2 and v3 can share a data directory because the DB file names differ, but they collide on the default port.
- **Tooling:**
  - `AGENTS.md` (symlinked as `CLAUDE.md`) and `.claude/skills/` exist, so the repo is agent-assisted.
  - `scripts/upstream-sync.mjs` plus `.github/upstream/state.json` drive the upstream sync (§1.3).

### 1.2 Size (LOC of .cs files incl. blanks / file count)

| Project | Eros LOC | Files | Radarr LOC (ref.) |
|---|---:|---:|---:|
| NzbDrone.Core | 105,280 | 1,469 | 101,133 / 1,556 |
| Whisparr.Api.V3 | 16,077 | 195 | 11,191 / 160 |
| Whisparr.Http | 3,532 | 73 | 3,354 |
| NzbDrone.Common | 13,695 | 170 | 13,559 |
| NzbDrone.Host | 1,646 | 20 | 1,624 |
| NzbDrone.Update / Mono / Windows | 663 / 1,411 / 302 | 10 / 14 / 3 | same |
| NzbDrone.Core.Test | 54,847 | 384 | 49,288 |
| NzbDrone.Api.Test | 2,706 | 23 | 384 |
| frontend/src (ts/tsx) | ~108,100 | 1,533 | ~100,600 |

**Duplication against Radarr, measured file by file.** The comparison was run against `submodules/radarr` after substituting `Radarr→Whisparr`:
- Core has 1,469 files. **1,312 share a path with Radarr, and 878 of those (67%) are byte-identical**, which is **60% of all Core files**. The 434 that differ account for about 17.8k diff lines.
- **At line level, 83.8k of 105.3k Core lines (80%) match a line in the same-path Radarr file** (SequenceMatcher).
- **157 files are Eros-only:**
  - migrations 002–026 plus 243–245, and `Datastore/PagedFilters/*` (server-side paged filtering)
  - `Movies/Performers/*` and `Movies/Studios/*` (27 files, 2.2k LOC)
  - `ImportLists/StashDB/*` (25 files) and `ImportLists/Whisparr/*`
  - `Statistics/*` (9, ported from Sonarr and extended) and `LibrarySearch/*`
  - `IndexerSearch/{SceneSearchCriteria, Performers/StudiosSearch*}`
  - `MediaFiles/SceneDiskScanService.cs`, `MediaFiles/MovieImport/Identification/IdentificationService.cs`, `RootFolders/ImportFile*`
  - `Notifications/Stash/*`
  - `Profiles/Qualities/QualityProfileQualityRank*` (ported from Sonarr)
  - `Update/GithubUpdatePackageProvider.cs`
  - `Configuration/{MovieMetadataType,SearchDateFormatType,SearchStudioFormatType}.cs`
- **244 Radarr files are absent.** 134 are migrations. The rest are deliberate removals (§11.3).
- **Largest diffs:**
  - `MetadataSource/SkyHook/SkyHookProxy.cs` (1,754 diff lines)
  - `MediaCover/MediaCoverService.cs` (1,118)
  - `Movies/MovieService.cs` (1,061)
  - `Parser/Parser.cs` (943)
  - `Download/Clients/Sabnzbd/Sabnzbd.cs` (791)
  - `Datastore/BasicRepository.cs` (723, paged filters)
  - `Movies/MovieRepository.cs` (647)
  - `Organizer/FileNameBuilder.cs` (451)
  - `Movies/AddMovieService.cs` (367)
  - `ImportLists/ImportListSyncService.cs` (289)
  - `Movies/RefreshMovieService.cs` (284)
  - `MediaFiles/DiskScanService.cs` (278)
- **Diff lines by folder:** Movies 3.0k, Download 2.0k, MetadataSource 1.9k, MediaFiles 1.7k, Parser 1.6k, Datastore 1.4k, MediaCover 1.1k.
- Download (Sabnzbd, Flood, TorrentClientBase, DownloadClientProvider) and MediaCover are **not adult logic**. They are Eros-local fixes and Sonarr ports, such as the download-client fallback in #925.

### 1.3 Lineage and upstream lag
- **Upstream is Radarr.** `AGENTS.md`: "a **fork of Radarr**". `Whisparr/Whisparr` README: "Version 3.x (based on Radarr), focuses on scenes, but also supports movies."
- **History:**
  - The repo contains Radarr's full history: 13,435 commits on `eros-develop`, and Radarr tags up to `v6.0.3.10276` were imported.
  - The **merge base with Radarr `develop` is `fa62f3f66a` (Radarr, committed 2023-12-12, just before `v5.2.6.8376`)**, found with `gh api repos/Whisparr/Whisparr-Eros/compare/ef9836d71d...eros-develop` and `repos/Radarr/Radarr/compare`.
  - After that, no Radarr merges happened. Everything since (1,351 Radarr commits up to `c90668a5`) came in as cherry-picks: first as "January Radarr upstream N" batches (e.g. #896, 2026-01-05), then as monthly "Chore: Sync Radarr YYYY-MM" PRs.
  - v3 lived on the `eros` branch of `Whisparr/Whisparr` until the separate repo was created on **2026-01-15**.
- **The sync mechanism is cherry-pick plus disposition ledger, not merge:**
  - **Two upstreams, two clocks: backend follows Radarr `develop`, frontend follows Sonarr `v5-develop`** (`UPSTREAM_SYNC.md`, `.github/upstream/state.json`).
  - Every upstream commit gets a disposition, `pick | adapt | have | skip | defer`, with a written reason.
  - The weekly workflow `.github/workflows/upstream-watch.yml` (cron Monday 06:00) runs `scripts/upstream-sync.mjs --attempt`. It cherry-picks the ungated backlog into a draft PR and keeps a rolling tracking issue. `upstream-check.yml` runs `--check` in CI.
  - PRs "need a merge commit, not a squash" so that `(cherry picked from commit radarr/radarr@…)` trailers survive (#909 body).
  - The dispositions start in 2025-10 (earlier cherry-picks were untracked):
    - **Radarr:** 125 = 55 skip, 28 have, 21 pick, 19 adapt, 2 defer.
    - **Sonarr:** 578 = 325 skip, 138 have, 73 adapt, 40 pick, 2 defer.
  - Radarr **frontend** commits are skipped by default (the "radarr-frontend gate"), and so are **parser** changes ("skip-by-default rule", e.g. GiLG group).
- **Current lag:**
  - **Radarr:** high-water `c90668a5` (2026-09-12, the tip of `develop`). **0 commits behind.** Deferred: `e80abdd60` (restart lifetime, upstream half-wired) and `c52c7f438` (file-quality filter, a product decision).
  - **Sonarr (frontend):** high-water `76c684e0` (2026-09-16). **24 commits behind** `v5-develop`. Deferred: `aea7ea743` (cert chain) and **`9f0cb9284` OIDC** (a product decision; v2 already took it).
  - Commit-level lag is ~0. Feature-level divergence is deliberate (§11.3).
- **Schema:**
  - `001_initial_setup.cs` is a **fresh squashed baseline** that already contains `Studios`, `Performers` and `MovieMetadata`.
  - `002–026` are Eros-own.
  - After that, it **jumps to upstream numbering**: `236_clear_last_rss_releases_info`, `237_add_indexes`, `239_…min_upgrade_format_score`, `240_drop_multi_episode_style`, `241_6K3384_Quality`, `242_release_profile_indexer_ids`.
  - It then continues with its own numbers **beyond Radarr's current max (242)**: `243_performer_after_date`, `244_quality_profile_quality_ranks` (Sonarr's mig 233), `245_add_tag_existing_to_importlists` (Sonarr's mig 231).
  - Radarr's next migrations (243+) will **collide by number**. The schema is neither Radarr- nor v2-compatible.

### 1.4 Maturity and activity
- **Releases:** ~480 GitHub releases since 2026-01-15, of which **21 are stable** (2026-01-15 to 2026-09-24, per `gh release list`), ending with `v3.6.2-release.1727` (2026-09-24). They are not weekly overall (there is a gap from 2026-03-26 to 2026-06-19), but have come about weekly since September (3.5.0 on 09-09, 3.6.0 on 09-17, 3.6.1 on 09-20, 3.6.2 on 09-24). The updater reads GitHub Releases, and uses `/releases/latest` for the stable branch (`Common/Cloud/WhisparrCloudRequestBuilder.cs:34`).
- **Activity:**
  - 726 commits in 2026.
  - Of the last 100 commits (since 2026-08-27), **72 are by one maintainer (`plz12345`)**. Upstream maintainers (Mark McDowall, Bogdan) appear via cherry-picks. **The bus factor is about 1.**
  - Repo stats: 54 stars, 14 forks, GPL-3.0, not archived.
- **Issues:** 213 issues and 712 PRs in total, with **0 open issues and 0 open PRs** (#880 moved issue tracking to this repo, 2026-09-17). Issues are typically opened and closed by the maintainer on the same day. `Whisparr/Whisparr` has 38 open issues (v2 and legacy).
- **Distribution:** the Docker image is `hotio/whisparr:v3` (README of `Whisparr/Whisparr`). Direct installs come from Eros Releases.
- **Assessment:** active, stable channel exists, well-tested (Core.Test 54.8k LOC, 4,563 tests per #909), but young as a separate product and single-maintainer. It is **not** a drop-in successor to v2: there is no migration and the model is different.

---

## 2. Domain model

Tables are mapped in `Core/Datastore/TableMapping.cs:66-196`. All Radarr tables are kept except `MovieTranslations`. New tables: `Performers`, `Studios`, `ImportFiles`, `QualityProfileQualityRanks`.

| Entity (table) | Key fields / notes | File |
|---|---|---|
| **Movie** (`Movies`) | Radarr's per-library row, unchanged in shape: `MovieMetadataId`, `Monitored`, `QualityProfileId`, `Path`, `Added`, `Tags`, `AddOptions` (`AddMethod = Manual\|List\|Performer\|Studio`), `LastSearchTime`, **`MovieFileId` (single file)**. Pass-through props `ForeignId/TpdbId/TmdbId/ImdbId/Year`. `IsAvailable()` ignores the delay: Released ⇔ release date has passed. **Represents both scenes and movies.** | `Core/Movies/Movie.cs`, `AddMovieOptions.cs` |
| **MovieMetadata** (`MovieMetadata`) | **`ItemType` (`Movie, Scene, Studio, Performer`)** and **`MetadataSource` (`Tmdb, Stash, TpdbMovie`)**. **`ForeignId`** is the primary key into the source: the StashDB UUID for scenes, the TMDb int as a string, or `tpdbId:<id>`. **Multi-ID columns: `StashId`, `TpdbId`, `TmdbId`, `ImdbId`.** **`Code`** (studio code / product code). `Title/CleanTitle/SortTitle`, `ReleaseDate` (string `yyyy-MM-dd`) + `ReleaseDateUtc`, `Year`, `Runtime`, `Overview`, `Website`, `Genres`, `Images`, `Ratings`, `Status`, `OriginalLanguage`. **`StudioForeignId`**, `StudioTitle`, `Studio` (embedded `StudioResource` JSON). **`PerformerForeignIds`**, **`PerformerNames`** (JSON lists). `CollectionTmdbId`/`CollectionTitle` (TMDb movies only). `TagIds` (StashDB tag UUIDs; *not persisted*, used at add-time for exclusions). Radarr fields dropped: `InCinemas/PhysicalRelease/DigitalRelease`, `Certification`, `Keywords`, `SecondaryYear`, `YouTubeTrailerId`, `Translations`, `Popularity`. | `Core/Movies/MovieMetadata.cs` |
| **Studio** (`Studios`) | `ForeignId` (StashDB UUID), `TpdbId`, `TmdbId`, `Title/SortTitle/CleanTitle`, **`Aliases`**, **`SearchTitle`/`CleanSearchTitle`** (user override for indexer queries), `Website`, `Network` (parent), `Images`, `Status` (Active/Inactive/Unknown/Deleted). User state: **`Monitored`** (scenes), **`MoviesMonitored`** (full movies), **`WhisparrMonitorNewItems`**, **`AfterDate`**, `QualityProfileId`, `RootFolderPath`, `SearchOnAdd`, `Tags`. Counters: `MovieCount/SceneCount/TotalMovieCount/TotalSceneCount/SizeOnDisk`. | `Core/Movies/Studios/Studio.cs` |
| **Performer** (`Performers`) | `ForeignId` (StashDB UUID), `TmdbId`, `TpdbId`, `MergedIntoId` (not persisted; follows StashDB merges), `Name`, `Disambiguation`, `Aliases`, `SortName/CleanName`, `Gender` (Male, Female, TransMale, TransFemale, Intersex, NonBinary), birth/death dates, `Country`, `CareerStart/End`, descriptive physical attributes (enums/ints mirroring StashDB's performer schema), `Status`. The same user-state block as Studio: `Monitored`, `MoviesMonitored`, `WhisparrMonitorNewItems`, `AfterDate`, `QualityProfileId`, `RootFolderPath`, `SearchOnAdd`, `Tags`. Counters (mig 020). | `Core/Movies/Performers/Performer.cs` |
| **Credit** (`Credits`) | `MovieMetadataId`, **`PerformerForeignId`** (may be empty; then `UniqueId` falls back to `PersonName`), `PersonName`, `Character`, `Job`, `Order`, `Type`, `Images`, embedded `CreditPerformer {Name, Disambiguation, ForeignId, TmdbId, TpdbId, Gender, Country, Images, Monitored, MoviesMonitored}` | `Core/Movies/Credits/Credit.cs` |
| **MovieCollection** (`Collections`) | Radarr's TMDb collection, kept for full movies only; gains `WhisparrMonitorNewItems` (mig 026) | `Core/Movies/Collections/` |
| **ImportListExclusion** (`ImportExclusions`) | Re-keyed from int TmdbId to **`ForeignId` string + `Type` (`Scene, Movie, Studio, Performer, Tag, Collection`) + `Reason`** (`Manual, DuringDelete, PerformerExclusion, StudioExclusion, StudioAfterDate, TagExclusion, CollectionExclusion, PerformerAfterDate`) | `Core/ImportLists/ImportExclusions/` |
| **ImportFile** (`ImportFiles`) | Unmapped files found under root folders (mig 022): `RootFolderId`, `Path`, `RelativePath`, `Name` | `Core/RootFolders/ImportFile.cs` |
| **QualityProfileQualityRank** | Sonarr's per-profile quality ranks (mig 244) | `Core/Profiles/Qualities/` |
| **Quality** | Sonarr-style `QualitySource` enum (`Unknown, Web, WebRip, DVD, DVDRaw, Television, TelevisionRaw, Bluray, BlurayRaw, BlurayDisk, VR`). **Radarr's `Modifier` enum and CAM/TELESYNC/TELECINE/WORKPRINT/DVDSCR/REGIONAL are removed.** Added `WEBDL-2880p/3160p/3384p/4320p` (ids 33–36, the 5K/6K/8K VR resolutions) and **`VR`** (id 32) | `Core/Qualities/Quality.cs`, `QualitySource.cs` |
| **NamingConfig** | Separate movie and scene formats (see §3.4) | `Core/Organizer/NamingConfig.cs` |
| Everything else | Same as Radarr: MovieFile, History, DownloadHistory, Blocklist, PendingRelease, ImportListMovies, AlternativeTitles, provider tables + `*Status`, Tags, AutoTagging, CustomFormats, Release/Delay profiles, Users, Commands, CustomFilters, Config | |

### 2.1 Relationships
- `Movie 1–1 MovieMetadata`, as in Radarr.
- `MovieMetadata → Studio` goes through **`StudioForeignId` (a string, no FK)**.
- `MovieMetadata ↔ Performer` is **M:N through the `PerformerForeignIds` JSON list plus the `Credits` rows** (`Credits.PerformerForeignId`). There is no join table. Lookups use `MovieRepository.GetByPerformerForeignId` (JSON contains).
- `MovieMetadata → Collection` goes through `CollectionTmdbId`.
- `Movie 1–0..1 MovieFile`.
- **There is no link between a full movie and the scenes cut from it.** They are independent items.

### 2.2 Scenes vs full movies
- `ItemType` is set from the metadata resource (`SkyHookProxy.MapMovie`, `SkyHookProxy.cs:94-185`).
- `MetadataSource` is derived from it (`MapMetadataSource`, `:187-201`): a Movie with a `TpdbId` is `TpdbMovie`, any other Movie is `Tmdb`, and a scene is always `Stash`.
- The same `Movies` table and `/api/v3/movie` resource serve both. Behaviour branches on `ItemType` in these places:
  - parsing and matching (`ParsingService.FindMovie`, §3.2);
  - search (`ReleaseSearchService.MovieSearch`: `MovieSearchCriteria` vs `SceneSearchCriteria`);
  - naming (`StandardMovieFormat` vs `StandardSceneFormat`, `MovieFolderFormat` vs `SceneFolderFormat`);
  - `Movie.ToFormattedString()`, which gives `Studio - Date - Title` for scenes and `Title (Year)` for movies;
  - the exclusion type (Scene/Movie).
- **Global setting `WhisparrMovieMetadataSource` (`NONE|TMDB|TPDB`, default TMDB)** picks the provider for full movies (`Core/Configuration/MovieMetadataType.cs`, `ConfigService.cs:612`).

### 2.3 Studio- and performer-driven acquisition
- **Auto-creation.** Refreshing any scene or movie auto-creates its Studio and all credited Performers as **unmonitored** rows (`RefreshMovieService.cs:162-225`). The Studio/Performer tables therefore hold every entity seen, like Radarr's Collections.
- **Monitoring a performer or studio.** When a performer or studio is monitored, the `*AddedHandler` queues a `RefreshPerformersCommand`/`RefreshStudiosCommand`. `SyncPerformerItems` (`RefreshPerformerService.cs:139-285`) then works as follows:
  - It calls `GET performer/{stashId}/works` → `(StashdbIds, TpdbIds, TmdbIds)`.
  - If **`Monitored`**, it adds every StashDB scene not already present and not excluded.
  - If **`MoviesMonitored`**, it adds every TMDb or TPDB movie, depending on `WhisparrMovieMetadataSource`.
  - Items are added in chunks of 10, with `AddMethod = Performer`.
  - Added items are **monitored** only if `WhisparrMonitorNewItems` is on both globally and on the entity, and searched only if `SearchOnAdd` is also on.
- Studios work the same way through `site/{stashId}/works` (`RefreshStudioService.cs:121`).
- **Gates at add time** (`AddMovieService.cs:265-420`). A new item is either **excluded** (an exclusion row with a reason, and the add is aborted), or, when the matching "always exclude … tag" setting is set, **added unmonitored with that tag**. The gates are:
  - an excluded studio, collection, performer or StashDB tag;
  - a release date earlier than the studio's `AfterDate`;
  - a release date earlier than *any* credited performer's `AfterDate`.
- **Compared with v2:** v2 had no performer entity, only a TPDb Performer *import list* producing (site, scene) pairs with per-episode monitoring (whisparr.md §4.4). v3 makes performer and studio **first-class monitorable sources**. It also keeps StashDB import lists (§4.4) for filtered feeds (performer + studio + tag filters, favourites, after-date).

### 2.4 Tags and collections
- **Local tags** are Radarr's int tag sets on Movie, Studio and Performer, used for the usual profile/indexer/notification scoping.
- **StashDB tags** exist only as `MovieMetadata.TagIds`, which is transient. They drive tag exclusions and the StashDB Tags import list.
- **Genres** are a string list.
- **Collections:** Radarr's TMDb collections are reused unchanged, for full movies only. **Studio is a new container type, not a reuse of Collection.**

### 2.5 Migrations (`Core/Datastore/Migration/`, 37 files)
- `000`, `001_initial_setup` (baseline).
- **Own:**
  - `002_postgres_update_timestamp…`, `003_movie_last_searched_time`, `004_email_encryption`
  - `005_import_exclusion_type`, `006_5K_6K_8K_Qualities`, `007_add_import_folder_config`, `008_add_credits`, `009_studio_alases`, `010_add_filelength_config`
  - **`011_add_tpdb`**, `012_studio_status`, **`013_add_tmdb`**, `014_import_exclusion_reason`, `015_add_collectioms`, `016_credit_personname`, `017_migrate_stashdb_enum_fields`, `018_remove_credits_from_moviemetadata`
  - `019_extend_performer`, `020_add_performer_totals`, `021_qualityprofile_fallback`, `022_add_unmappedfiles`, `023_add_missing_studio_columns`, `024_movie_metadata_paging_indexes`, `025_afterDate`, `026_add_monitor_new_items`
- **Upstream-numbered:** 236, 237, 239–242 (Radarr), and 243–245 (own/Sonarr ports) (§1.3).

---

## 3. Feature areas

Top-level `Core/` folders. **R** = shared with Radarr (identical or near-identical); **A** = adult-specific changes; **R−** = shared but with parts removed; **S** = ported from Sonarr.

- **Analytics** (R).
- **Annotations** (R).
- **Authentication** (R): None / Basic (`[Obsolete]`) / Forms / External. **No OIDC** (deferred; v2 has it).
- **AutoTagging** (R−): Genre, Monitored, OriginalLanguage, QualityProfile, RootFolder, Runtime, Status, Tag, Year. Radarr's Keyword and Studio specs are removed.
- **Backup** (R).
- **Blocklisting** (R).
- **Configuration** (A): ~27 new DB keys (§7), the `WhisparrMetadata` and `GithubOwnerRepo` config.xml keys, and the enums `MovieMetadataType`, `SearchDateFormatType` and `SearchStudioFormatType`. `CertificationCountry` and `MovieInfoLanguage` are removed.
- **CustomFilters** (R).
- **CustomFormats** (R−): QualityModifier spec removed (there is no Modifier).
- **Datastore** (A):
  - own migration chain;
  - **`PagedFilters/*`**: 15 typed filter classes for server-side paging of the movie, performer and studio indexes, needed for libraries of 100k+ scenes (`BasicRepository.cs` +723 lines);
  - paging indexes (mig 024).
- **DecisionEngine** (R−): Radarr's `AvailabilitySpecification` (minimum availability) and `PendingSpecification` are removed. The rest are shared. `DownloadDecisionMaker.cs:81-126` accepts scene parses that have a studio or a code.
- **DiskSpace** (R).
- **Download** (R + local):
  - client fallback on grab failure, a Sonarr port (#925, `DownloadClientProvider.cs`);
  - large Sabnzbd and Flood rewrites (local fixes);
  - **no code-based completed-download fallback**, which v2 had (whisparr.md §3 Download);
  - RQBit client not ported.
- **Exceptions** (A): `ExcludedException`.
- **Extras** (R + A): the Xbmc NFO formatter adds `<studio>` and `<actor>`, but **only writes `tmdb`/`imdb` uniqueids**, so a scene gets `<uniqueid type="tmdb">0` and no StashDB id (`XbmcMetadata.cs:245-257`). Kometa, MediaBrowser, Roksbox and Wdtv are as in Radarr.
- **HealthCheck** (R−/A): 33 checks. `MetadataUrlCheck` warns when `WhisparrMetadata` still points at v2's `api.whisparr.com/v3`. Radarr's `MetadataCheck` and `PTPOldSettingsCheck` are removed.
- **History** (R + A): `MovieFileScannedImportHistoryHandler`.
- **Housekeeping** (R−/A): 35 housekeepers:
  - added `AssignDefaultQualityProfiles`, `CleanupExtraFilesInExcludedFolders`;
  - removed the orphaned-collections/credits/files/translations cleanups.
- **Http** (R).
- **ImportLists** (A): StashDB ×4, TMDb ×6, RSS, Whisparr; Radarr's other lists are removed (§4.4). Exclusions are re-keyed (§2).
- **IndexerSearch** (A): `SceneSearchCriteria`, `PerformersSearchService`, `StudiosSearchService` (search every monitored missing item of a performer or studio), and the scene query builder in `ReleaseSearchService` (§3.3).
- **Indexers** (R−/A): Nyaa, PassThePopcorn and TorrentPotato are removed. Newznab/Torznab/HDBits/FileList gain `GetSearchRequests(SceneSearchCriteria)`. There are no per-indexer adult toggles, only `RemoveYear` (§3.3).
- **Instrumentation** (R): Sentry `sentry.servarr.com/9|10|11`.
- **Jobs** (A): `RefreshPerformers` and `RefreshStudios` tasks (§6).
- **Languages** (R).
- **LibrarySearch** (A): ranked local search across scenes, movies, performers and studios (`LibrarySearchRanker.cs`), backing `/api/v3/library/search`.
- **Lifecycle** (R).
- **Localization** (R/A): 45 locale files.
- **MediaCover** (R + A): heavily reworked (+1.1k diff lines). Covers for performers and studios, a background download queue, and cacheability checks (#909).
- **MediaFiles** (A):
  - `SceneDiskScanService` + `RescanScenesCommand` + `CleanUnmappedFiles` scan root folders for unmapped files into `ImportFiles`;
  - `Identification/IdentificationService` identifies a loose file against the metadata server and moves it into place (§3.2);
  - `WhisparrCorruptFileDetection` and `WhisparrValidateRuntime` import checks;
  - `FFMpegCoreSideDataTypes` (VR/stereo side data).
- **Messaging** (R).
- **MetadataSource** (A): a single `SkyHookProxy` against `api.whisparr.com/v4` (§3.1).
- **Movies** (A): the domain model, performers, studios, `FindScene`/`MatchMovies`, fuzzy matching, and add-time exclusion gates.
- **MovieStats** (R + A): `MovieIndexOverview`.
- **Notifications** (R + A): **Stash** added. Radarr's list is otherwise kept, including Trakt, Twitter, Join and Prowl.
- **Organizer** (A): scene tokens and formats (§3.4).
- **Parser** (A): scene, date, code and StashId regexes ahead of Radarr's movie cascade (§3.2).
- **Profiles** (R + S): quality ranks (Sonarr), `QualityProfileInUse`, and a quality-profile fallback (mig 021).
- **ProgressMessaging** (R).
- **Qualities** (A): Sonarr-style source enum, VR and 5K–8K (§2).
- **Queue** (R).
- **RemotePathMappings** (R).
- **RootFolders** (A): `ImportFile` repository and service; `WhisparrFolderLimit`.
- **Security** (R).
- **Statistics** (S + A): Sonarr's library statistics, extended with per-performer, per-studio, per-tag and per-quality stats (`/api/v3/statistics`).
- **Tags** (R).
- **ThingiProvider** (R).
- **Update** (A): `GithubUpdatePackageProvider` (GitHub Releases of `whisparr/whisparr-eros`, branch `eros`).
- **Validation** (R + A).

### 3.1 Metadata source (deep dive)
- **Endpoints** (`src/NzbDrone.Common/Cloud/WhisparrCloudRequestBuilder.cs`):
  - `WhisparrMetadata = https://api.whisparr.com/v4/{route}`. The default is repeated in `ConfigFileProvider.cs:193` and `SkyHookProxy.cs:59`. It can be overridden in config.xml as `WhisparrMetadata`, and v2's `/v3/` URL is silently replaced (`:197`).
  - `StashDB = https://stashdb.org/graphql`, used **only by the StashDB import lists** with the user's API key.
  - `TMDB = https://api.themoviedb.org/{api}/{route}/{id}{secondaryRoute}` with a **hard-coded bearer token identical to Radarr's** (`AuthToken`, same value as `RadarrCloudRequestBuilder`), used only by the TMDb import lists.
  - `Services = https://whisparr.servarr.com/v1/`.
  - GitHub Releases.
- **All entity metadata goes through the proxy.** Routes (`SkyHookProxy.cs`):

| Route | Use |
|---|---|
| `scene/{stashId}`, `scene/search?q=[&studio=<stashId>&date=<yyyy-MM-dd>]`, `scene/changed?since=` | scenes (StashDB-backed) |
| `site/{stashId}`, `site/search?q=`, `site/{id}/scenes`, `site/{id}/works`, `site/changed` | studios |
| `performer/{stashId}`, `performer/search?q=`, `performer/{id}/works`, `performer/changed` | performers |
| `movie/{tmdbId}`, `movie/bulk`, `movie/imdb/{id}`, `movie/search?q=&year=`, `movie/collection/{tmdbId}`, `movie/changed` | TMDb movies |
| `tpdb/movie/{id}`, `tpdb/movie/bulk`, `tpdb/movie/search`, `tpdb/movie/changed` | ThePornDB movies |

- **Resource shape** (`MetadataSource/SkyHook/Resource/`):
  - `MovieResource {ForeignIds {StashId, TmdbId, TpdbId}, ItemType, Title, Code, ReleaseDate, ReleaseDateUtc, Year, Duration, Overview, Homepage, Images, Genres, Tags[{ForeignIds}], Credits[], Studio {ForeignIds, Title, …}, Collection, AlternativeTitles, Ratings}`.
  - `PerformerResource`, `StudioResource`, and `*WorksResource` (lists of stash/tpdb/tmdb ids).
- **Lookup syntax** (`SearchForNewEntity`, `:780-843`):
  - `stash:`, `stashid:`, `tpdb:`, `tpdbid:`, `tmdb:`, `tmdbid:`, `imdb:` prefixes;
  - full `https://stashdb.org/{scenes|performers|studios}/<uuid>`, `https://theporndb.net/movies/<id>` and `https://www.themoviedb.org/movie/<id>` URLs are rewritten to ids.
- **IDs stored:** a **multi-provider ID set per entity** (`StashId`, `TpdbId`, `TmdbId`, `ImdbId` on metadata; `ForeignId`/`TpdbId`/`TmdbId` on Studio and Performer) **plus one canonical `ForeignId`** whose meaning depends on `MetadataSource`. Studio and Performer identity is always the StashDB UUID. **ThePornDB scene ids are not stored**, so v2's TPDB scene ids cannot be mapped directly.
- **Refresh is delta-based:**
  - `RefreshMovieService.cs:405-420` asks `*/changed?since=<last run>` and refreshes only changed items;
  - `ShouldRefreshMovie` forces a refresh after 60 days, skips if synced within 12h, and refreshes recently released items;
  - performers and studios do the same.
  - This is a clear improvement over v2's full site refetch every 12h (whisparr.md §3.1).
- **Opt-in response caching:** the `WhisparrCache{Movie,Performer,Studio,Exclusion}API` keys (§7).
- **No API key is needed** for the proxy, which is anonymous. StashDB lists need a user StashDB key. The TMDb token is embedded.

### 3.2 Parsing and matching (deep dive)

**`ReportTitleRegex`** (`Core/Parser/Parser.cs:38-179`) is an ordered list. **Scene/date patterns come first**, then Radarr's movie cascade, then ID patterns:
1. `Site - Performers - Title (Month D, YYYY)` (long month).
2. `[Site] Title - Performers (YYYY-MM-DD)`.
3. `[Site] - YYYY-MM-DD - …`.
4. `[Site]` or `(Site)` `… - YYYY-MM-DD` (date after title/performer).
5. `Site.YY.MM.DD.Performer.Title` (captures `performer` and `title` separately).
6. `[Site] DD-MM-YYYY`.
7. `[Site] … YYYY-MM-DD` (variant).
8. `Site YYYYMMDD`.
9. `[Site] … DD.MM.YYYY)`.
10. `Studio-Performer-ID.429.[DD.MM.YYYY]` (bracketed day-first).
11. `Site YY.MM.DD` (bare).
12. `YYYY-MM-DD…` at the start (1960+).
13. `[Site] … [DD.MM.YYYY`.
14. `[Site] … YY(YY)-MM-DD`.
15. **`[Site] (E1234)`**: studio episode numbers → `Episode`.

Then come the Radarr movie patterns (German/French, editions), which are **adult-specific**:
- `Studio.2024.Title-VERIFIED`
- `[Studio] Title [2024]`
- `Title (Studio) XXX … 2018`

After the normal movie patterns come `Site E1234` and a `…XXX` fallback. Last are the ID patterns:
- **StashId** (a UUID anywhere);
- **JAV** codes: `^[A-Z]{2,5}[- ][0-9]{3,5}`, `FC2…PPV…\d{4,7}`, a code after leading `[tag]` blocks, and `[ABC-123]`/`(ABC_123)`/`{ABC.123}`;
- a final "`…1080p` = it is a video" pattern.

Pre-processing is Radarr's plus v2-style additions:
- `SixDigitAirDateRegex`;
- a studio-domain suffix strip (`.com|.xxx|…`);
- a `RequestInfoRegex` prefix strip;
- `ReleaseTagBoundaryRegex` for cutting release tokens;
- `CleanQualityBracketsRegex`.

**`ParsedMovieInfo`** (`Parser/Model/ParsedMovieInfo.cs`) adds `StudioTitle`, `ReleaseDate`, `ReleaseTokens`, `FirstPerformer`, `Episode`, `Code`, `StashId` to Radarr's fields. **`IsScene`** means a date, episode, code or StashId is present.

**Release → item** (`Parser/ParsingService.cs:135-179, 286-327`):
- A non-scene parse uses Radarr's movie path, restricted to `ItemType.Movie`: TMDb id, then search criteria, then title/year, then a new **`TryGetMovieByFuzzyYear`** (Levenshtein with a margin over the runner-up, `MovieService.cs:1098`).
- A scene parse goes to `MovieService.FindScene` (`MovieService.cs:700-755`):
  1. **StashId** in the release title: look up by `ForeignId`.
  2. **Code:** `FindByTitle(code)`. This is a *title* match. There is no index or lookup on `MovieMetadata.Code`.
  3. **Studio + date:**
     - `StudioService.FindAllByTitle(studioTitle)`, which matches title, clean title and aliases.
     - For each studio, `FindByStudioAndReleaseDate` (`:1228-1384`):
       - query `(studio, date)`;
       - **fuzzy pass first**: if `WhisparrFuzzyTitleMatchingThreshold ≥ 70` (default **80**), score `FuzzySharp.Fuzz.Ratio(releaseTokens minus performer/character names, cleanTitle)` for each same-date scene and **return the highest scorer ≥ threshold, with no ambiguity check**;
       - date widening: a `…-01` date also queries the whole month, and `…-01-01` the whole year;
       - if there are no rows, fall back to *all* scenes of the studio with `verifyDate=true`;
       - **`WhisparrAutoMatchOnDate`** (default on): a single scene on that studio and date is accepted outright;
       - without a date, fall back to studio-wide with `verifyEpisode=true`;
       - then call **`MatchMovies`**.
     - The result is accepted only if exactly one studio produced a match.
- **`MatchMovies`** (`MovieService.cs:771-1012`) is the successor of v2's `EpisodeService.MatchEpisodes`. Each candidate gets the *first* level it satisfies, in the order of the `MovieParseMatchType` enum (`Movies/MovieParseMatchType.cs`):
  1. StashId
  2. Title (exact, or equal ignoring spaces)
  3. Episode/Code (equal, or numerically equal; or both code and title contained)
  4. PerformersTitle (tokens == a performer name)
  5. CharactersTitle
  6. Performers (all female performers contained)
  7. Characters
  8. PerformerTitle (a performer *and* the title contained)
  9. CharacterTitle
  10. PerformersNotTitle (all performers, title absent)
  11. CharactersNotTitle
  12. ParsedTitleContainsCleanTitle
- **Disambiguation:** if more than one candidate matches, it walks the levels and keeps the set strictly better than level *k* once that set has size 1. If no level isolates one candidate, **no match** (the reject-if-ambiguous policy is retained here). `verifyDate` and `verifyEpisode` then discard a lone match whose date or code disagrees.
- **Search-time guard:** when a search criterion is present, the found scene must equal the searched item by `ForeignId` or `TmdbId` (`ParsingService.cs:299-314`).
- **File identification** (`IdentificationService.Identify`, `MediaFiles/MovieImport/Identification/IdentificationService.cs:75+`) handles loose files:
  - StashId in the name: direct lookup.
  - Otherwise, if the parent folder has a date, search the server with the folder name; else search `"<studio> <date> <first performer>"`.
  - Then run `MatchMovies` over the *remote* results, and move the file into the scene folder.
  - `SearchForNewEntity` with a parsed release uses the new server-side `scene/search?studio=&date=` filter and drops cross-studio hits (`SkyHookProxy.cs:1649-1690`).

**Comparison with v2 (whisparr.md §3.2, §11.1):**

| Algorithm | v2 | v3 Eros | Verdict |
|---|---|---|---|
| Date-first scene regexes | 7 patterns, date-only | ~15 scene patterns + movie cascade + ID patterns | **Improved** (more formats, `[Site]`/`(Site)`, bracketed dates, episode numbers) |
| Day/month swap, future-date rejection, 2-digit year | In `ParseMatchCollection` | Radarr's year handling + date parse; month/year widening on `-01` | Equivalent / different |
| Same-date disambiguation | Graded matcher, recursion to stricter level, **reject if ambiguous** | Same levels + characters + studio episode code, precedence by enum, reject if ambiguous; **but a fuzzy Levenshtein pass runs first and takes the top score without a margin** | **Mixed:** richer, but the fuzzy pass can mis-assign when two same-date scenes both score ≥80 |
| Single scene on date | Accept | Accept (`WhisparrAutoMatchOnDate`, toggleable) | Equivalent |
| Studio lookup | Clean title → all titles → slug → TPDB id → ExternalId owner | Title/clean title/**aliases**; the user `SearchTitle` is used for queries only | Equivalent; no "owner of this code" fallback |
| Product code / JAV | 10 regex variants, `Episode.ExternalId` index (plain, not unique: `Migration/221_add_episode_external_id_index.cs:12`), used in decisions, **completed-download fallback**, import aggregation, folder→site mapping; `JavEpisodeFormat` naming | 4 JAV regexes, `MovieMetadata.Code` (no index), matched via `FindByTitle(code)` or as a scene "episode"; `{Scene Code}` token, but no JAV-specific default format | **Regression** |
| StashId in release name | n/a | Direct match | **New** |
| Full-movie matching | n/a (v2 has no movies) | Radarr title/year + fuzzy-year fallback | **New** |
| Library import of unknown files | Via site folders | Unmapped-file scan + remote identification | **Improved** |

### 3.3 Search
- **Criteria:** `SceneSearchCriteria {SiteTitle, ReleaseDate (DateOnly), Performer}`. **`Performer` is never populated or used**; this is dead (`IndexerSearch/Definitions/SceneSearchCriteria.cs`). `MovieSearchCriteria` is Radarr's.
- **Query templates are built in `ReleaseSearchService.cs:59-199`**, not in the indexer (v2 built them in `NewznabRequestGenerator`). All toggles are **global DB config**, not per indexer:
  - dates: `SearchDateFormat` = `YYMMDD` → `yy.MM.dd`, `DDMMYYYY` → `dd.MM.yyyy`, `BOTH`. v2 used `dd.MM.yy`.
  - `SearchTitleOnly` → `q=<scene title>`.
  - `SearchTitleDate` → `q=<scene title> <date>`.
  - For each studio matching `SiteTitle`, and for each of its **aliases**, using `SearchTitle` if set, otherwise `Title` and/or `CleanTitle` per `SearchStudioFormat` (`ORIGINAL|CLEAN|BOTH`, default CLEAN):
    - `SearchStudioDate` (default **on**) → `q=<studio> <date>`;
    - `SearchStudioTitle` → `q=<studio> <scene title>`;
    - `SearchStudioCode` → `q=<studio> <code>`.
  - Full movies: `q=<title>` and `q=<title> <year>`, plus `t=movie&tmdbid=` when the indexer supports it and the item has a TmdbId (`NewznabRequestGenerator.cs:127-172`).
- **Newznab/Torznab** (`Indexers/Newznab/NewznabRequestGenerator.cs:96-125`): each scene title becomes `t=search&q=…`. In non-`raw` text-search engines the dots are stripped from dates, then restored. There is no `tvsearch` and no id search for scenes.
- **ID searches:** only `tmdbid` for TMDb movies. There is no StashDB, TPDB or code id parameter; codes are free text.
- **Categories:**
  - Newznab/Torznab default to **`6000,6010,6020,6030,6040,6045,6050,6070,6080,6090`** (the XXX range; `NewznabSettings.cs:59`).
  - **FileList defaults to Radarr's movie categories (SD/HD/HDRO/4K), not XXX**, which is a regression against v2's `XXX=7` (`FileList/FileListSettings.cs:35`).
  - HDBits: `<title> <yy.MM.dd>` (`HDBits/HDBitsRequestGenerator.cs:37-52`).
  - FileList: `type=name&query=<each scene title>`.
- **Bulk searches:** `PerformersSearchCommand` and `StudiosSearchCommand` search all monitored missing items of the given performers or studios.

### 3.4 Naming (`Core/Organizer/`)
- **Defaults** (`NamingConfig.cs`), with separate rename flags `RenameMovies`/`RenameScenes` (both false):
  - `SceneFolderFormat = scenes/{Studio CleanNetwork}/{Studio CleanTitle}/{Release Date} - {Scene CleanTitle} {[StashId]}`
  - `StandardSceneFormat = {Studio CleanTitleSlug} - {Release-Date} - {Scene CleanTitle} {[Quality Title]}`
  - `MovieFolderFormat = movies/{Studio CleanNetwork}/{Studio CleanTitle}/{Movie Title} {(Release Year)}`
  - `StandardMovieFormat`
  - `SceneImportFolderFormat = import/`
- **One folder per scene** (Radarr style). v2 used a site folder holding flat files.
- **Tokens:**
  - `{Studio Title|CleanTitle|TitleSlug|CleanTitleSlug|TitleThe|TitleFirstCharacter|Network|CleanNetwork|CleanNetworkSlug}`
  - `{Scene Title|CleanTitle|TitleThe|TitleFirstCharacter|CleanTitleNoSeasonEpisode|Code}`
  - `{Scene [Clean]Performers[Female|Male|Other][Alias]}`
  - `{Release Date|ShortDate|Year}`, `{StashId}`, `{TmdbId}`, `{ImdbId}`
  - `{Movie Title|CleanTitle|TitleThe|TitleFirstCharacter|Collection|CleanTitleThe}`
  - plus Radarr's quality, mediainfo, edition and custom-format tokens.
  - `{Movie Collection}` is advertised in the UI but has no handler (noted in `state.json` disposition `4c007291`).
- `WhisparrFolderLimit` (default 100) caps folder enumeration.

---

## 4. Integrations (full lists)

### 4.1 Indexers (`Core/Indexers/`) — 6
Newznab, Torznab, TorrentRss, IPTorrents, HDBits, FileList.
- Removed from Radarr: Nyaa, PassThePopcorn, TorrentPotato.
- **HDBits** is obsolete for this content (as in v2).
- Newznab presets still list general indexers (DOGnzb, DrunkenSlug, **Nzb.su (defunct)**, NZBCat, NZBFinder, NZBgeek, nzbplanet, SimplyNZBs, Tabula Rasa, **Usenet Crawler**) (`Newznab.cs:49-58`).

### 4.2 Download clients (`Core/Download/Clients/`) — 17
Aria2, Blackhole (torrent/usenet), Deluge, DownloadStation (torrent/usenet), Flood, FreeboxDownload, **Hadouken** (obsolete), NzbGet, **NzbVortex** (obsolete), **Pneumatic** (obsolete), QBittorrent, rTorrent, Sabnzbd, Transmission, uTorrent, **Vuze** (obsolete).
- **RQBit** (present in Radarr) and **Tribler** (Sonarr/v2) are absent.
- Grab failures now fall back to another client (#925).

### 4.3 Notifications (`Core/Notifications/`) — 27
- Apprise, CustomScript, Discord, Email, Gotify, **Join** (obsolete), Mailgun, MediaBrowser, Notifiarr, Ntfy, Plex, **Prowl** (obsolete), PushBullet, Pushcut, Pushover, Pushsafer, SendGrid, Signal, Simplepush, Slack, **Stash**, Synology Indexer, Telegram, **Trakt** (irrelevant here), **Twitter** (obsolete), Webhook, Xbmc.
- Discord embeds link to `stashdb.org/scenes/{id}` or TMDb (`Discord.cs:620-630`). The icon URL still points at `Whisparr/Whisparr/eros/Logo`.
- **Stash** (`Notifications/Stash/`) is the v2 notifier, extended:
  - It triggers on download, rename, file delete and movie delete.
  - It runs `metadataScan`, and optionally `metadataClean` and `metadataIdentify`.
  - **The default `StashBoxEndpoint` is `https://stashdb.org/graphql`** (v2 hard-coded `theporndb.net/graphql`).
  - New settings: Set Cover Image, Skip Multiple Matches, and Skip Multiple Match Tag ID.
  - `CheckForError` is still a TODO (`StashProxy.cs:138-142`).

### 4.4 Import lists (`Core/ImportLists/`) — 12, plus exclusions
- **StashDB** (`ImportLists/StashDB/`, direct GraphQL with the user's API key):
  - Favorite, Performer (performer UUIDs + optional studio/tag filters with include/exclude modifiers, "only favourite studios"), Studio, Tags.
  - Common settings: Limit (≤1000), Sort, **"Scenes released after"** (`StashDBSettingsBase.cs:36-46`).
- **TMDb**: Company, Keyword, List, Person, Popular, User (Radarr's; uses the embedded token).
- **RSS** (`RSSImport`, `Rss/RssImportBase`).
- **Whisparr**: another *v3* instance via `/api/v3/movie|qualityprofile|tag` (`WhisparrV3Proxy.cs:34-53`); it cannot read v2.
- Removed from Radarr: CouchPotato, Plex watchlist and Plex RSS, Radarr list/RadarrList2 (IMDb, StevenLu), Simkl, StevenLu, Trakt.

### 4.5 Metadata consumers
Kodi/XBMC (customised, with the id bug in §3), Kometa, MediaBrowser, Roksbox, Wdtv.

### 4.6 Metadata sources
`api.whisparr.com/v4` (all entities), StashDB GraphQL (lists only), TMDb API (lists only). There is **no direct TPDB API**; TPDB data arrives through the proxy.

---

## 5. External APIs / interfaces
- **REST API v3 only** (`src/Whisparr.Api.V3`, 72 controllers). It keeps Radarr's `/api/v3/movie…` surface (movie, movie/lookup, movie/import, movie/editor, moviefile, calendar, queue, release, release/push, wanted/missing|cutoff, collection, credit, alttitle, rename, etc.) and adds:
  - `/api/v3/performer` (GET list, `{id}`, `{foreignId}`, `{performerForeignId}/works`, POST `list`, POST `paged`), `/performer/editor` (PUT/DELETE).
  - `/api/v3/studio` (the same shape) and `/studio/editor`.
  - `/api/v3/lookup/{scene|movie|studio|performer}` (remote search).
  - `/api/v3/library/search[/scene|movie|performer|studio]` (local ranked search).
  - `/api/v3/statistics`.
  - `/api/v3/movie/lookup/{tmdb|tpdb|imdb}`, `/movie/listByPerformerForeignId`, `/movie/listByStudioForeignId`, `/movie/paged`, `/movie/stats`, `/movie/bulk`.
- `MovieResource` adds `Code`, `StudioTitle`, `StudioForeignId`, `TpdbId`, `ForeignId`, `StashId`, `PerformerForeignIds`, `PerformerNames`, `ItemType`, `IsExcluded`, and `Grabbed` (`Movies/MovieResource.cs`).
- **Upstream API v5 is not ported** (Radarr has none either).
- **OpenAPI:** served via Swashbuckle. No `openapi.json` is checked in (Radarr checks one in). #904 fixed 50 mis-described operations.
- **SignalR** at `/signalr/messages`.
- **Webhooks:** Radarr's payload with `WebhookMovie` extended by `ReleaseDate`, `StashId`, `TpdbId`, `ItemType`, `StudioTitle`, `Network` and `Performers[]` (`Notifications/Webhook/WebhookMovie.cs`). v2 had no performer fields.
- **Custom-script env vars:** prefix **`Whisparr_`**, Radarr's set:
  - `Whisparr_EventType`, `InstanceName`, `ApplicationUrl`
  - `Whisparr_Movie_{Id,Title,Year,Path,TmdbId,ImdbId,ForeignId,Release_Date,Genres,Tags,Overview,OriginalLanguage,AddMethod,DeletedFiles,Folder_Size}`
  - `Whisparr_Release_{Title,Indexer,Size,Quality,QualityVersion,ReleaseGroup,CustomFormat(Score)}`, `Whisparr_IndexerFlags`
  - `Whisparr_MovieFile_*` (Id(s), Path(s), RelativePath(s), PreviousPaths, Quality, ReleaseGroup, SceneName, SourcePath/Folder, Size, MediaInfo_*, CustomFormat(Score), DeleteReason)
  - `Whisparr_Download_*`, `Whisparr_IsUpgrade`, `Whisparr_Deleted*`, `Whisparr_Health_{Issue,Restored}_*`, `Whisparr_Update_*`
  - Source: `Notifications/CustomScript/CustomScript.cs`. **There are no studio, performer, StashId or code variables** (only `ForeignId`).
- **iCal** calendar feed (Radarr's controller).

---

## 6. Background jobs
`Core/Jobs/TaskManager.cs`:

| Command | Interval |
|---|---|
| RefreshMonitoredDownloads | configurable (1 min default) |
| MessagingCleanup | 5 min |
| ImportListSync | 5 min (per-list `MinRefreshInterval`) |
| ApplicationCheckUpdate | 6 h (GitHub Releases) |
| CheckHealth | 6 h |
| RefreshMovie (delta via `scene/changed`, `movie/changed`, `tpdb/movie/changed`) | 24 h |
| **RefreshPerformers** (metadata + works sync for monitored performers) | 24 h |
| **RefreshStudios** (the same for studios) | 24 h |
| RefreshCollections | 24 h |
| Housekeeping | 24 h |
| CleanUpRecycleBin | 24 h |
| Backup | configurable |
| RssSync | configurable |

- Differences from Radarr: **+RefreshPerformers, +RefreshStudios**.
- On-demand commands: `PerformersSearch`, `StudiosSearch`, `RescanScenes`, `CleanUnmappedFiles`, `AddMovies` (bulk add), plus Radarr's set.
- **Event-driven:** adding or monitoring a performer or studio queues its refresh (`PerformerAddedHandler.cs`, `StudioAddedHandler.cs`). Refreshing a movie auto-adds its studio and performers (unmonitored).

---

## 7. Configuration surface (specific parts only)
- **config.xml** (`ConfigFileProvider.cs`):
  - `WhisparrMetadata` (default `https://api.whisparr.com/v4/{route}`; the v2 `/v3/` value is rewritten).
  - `GithubOwnerRepo` (default `whisparr/whisparr-eros`).
  - `Branch` default `eros`.
  - Port default 6969.
  - **`Port` and `Branch` ignore the env/`ServerOptions`/`UpdateOptions` overrides** that Radarr honours (`:221`, `:283`), which is a divergence.
- **DB config**, added against Radarr (`ConfigService.cs`):
  - **Search:** `SearchDateFormat` (YYMMDD|DDMMYYYY|BOTH), `SearchStudioFormat` (ORIGINAL|CLEAN|BOTH), `SearchStudioDate` (default true), `SearchStudioTitle`, `SearchStudioCode`, `SearchTitleOnly`, `SearchTitleDate`.
  - **Matching:** `WhisparrAutoMatchOnDate` (true), `WhisparrFuzzyTitleMatchingThreshold` (80; below 70 disables it).
  - **Metadata:** `WhisparrMovieMetadataSource` (TMDB|TPDB|NONE), `WhisparrCache{Movie,Performer,Studio,Exclusion}API`.
  - **Monitoring:** `WhisparrMonitorNewItems` (the global master switch).
  - **Exclusion → tag instead of exclude:** `WhisparrAlwaysExclude{Studios,StudiosAfter,Performers,PerformersAfter,Collections,Tags}Tag`.
  - **Import:** `WhisparrValidateRuntime` + `WhisparrValidateRuntimeLimit` (1), `WhisparrCorruptFileDetection`, `WhisparrFolderLimit` (100), `SkipFreeSpaceCheckWhenGrabbing`.
  - `TimeZone` (Sonarr port).
  - Removed: `CertificationCountry`, `MovieInfoLanguage`.
- **Per Studio and Performer:** `Monitored`, `MoviesMonitored`, `WhisparrMonitorNewItems`, `AfterDate`, `QualityProfileId`, `RootFolderPath`, `SearchOnAdd`, `Tags`. Studio also has a `SearchTitle` query override.
- **Per indexer:** only Radarr's settings plus `RemoveYear`. v2's per-indexer `SearchTitleOnly`/`SearchSiteTitleOnly`/`SearchExternalIdOnly`/`DateSearchFormat`/`SeriesNameSource` are gone, moved to global.
- **StashDB list:** ApiKey, Limit, Sort, AfterDate, plus performer/studio/tag ids and filter modifiers.
- **Stash notifier:** Host, Port 9999, SSL, ApiKey, scan generate options, Identify (+StashBoxEndpoint, Builtin Autotag, Include Male Performers, Set Cover Image, Skip Multiple Matches, Skip Multiple Match Tag, Set Organized), MapFrom/MapTo.
- **Naming:** `RenameScenes`, `StandardSceneFormat`, `SceneFolderFormat`, `SceneImportFolderFormat`.

---

## 8. Platform/infrastructure concerns (divergences only)
- **Cloud endpoints:** `api.whisparr.com/v4` (metadata, closed source), `whisparr.servarr.com/v1` (services, server-side notifications), `stashdb.org/graphql`, `api.themoviedb.org` (**Radarr's embedded TMDb token**), `api.github.com/repos/whisparr/whisparr-eros/releases[/latest]`, and Sentry `sentry.servarr.com` projects 9/10/11 (`NzbDroneLogger.cs:75-81`).
- **Updates come from GitHub Releases** (`Update/GithubUpdatePackageProvider.cs`). Branches are `eros` (stable, `/releases/latest`) and develop (prereleases). Tag format: `v3.6.2-release.1727`.
- **CI:** GitHub Actions only (`build_v3.yml`, `deploy.yml`, `codeql.yml`, `sonarqube.yml`, `upstream-check.yml`, `upstream-watch.yml`). There is no azure-pipelines file (unlike Radarr and v2).
- **Schema numbering** collides with future Radarr migrations 243+ (§1.3).
- **DB file** `whisparr3.db`, in the same app-data folder name and on the same default port as v2 (§1.1).
- `DryIoc`, `NzbDrone.*` folder names, NLog, SQLite/Postgres and the Mono/Windows shims are identical to Radarr.

---

## 9. Interactions with other *arr apps
- **Prowlarr:**
  - The Whisparr app type (`submodules/prowlarr/src/NzbDrone.Core/Applications/Whisparr/WhisparrV3Proxy.cs:27`) talks to `/api/v3` and syncs `/api/v3/indexer`, so **v2 and v3 share one Prowlarr contract**. The default sync categories are the XXX 6000 range.
  - v2's per-indexer adult fields no longer exist in v3, so Prowlarr's field-preservation logic has nothing to keep.
- **autobrr:** `DownloaderTypeWhisparr` and `DownloaderTypeWhisparrV3` share one client (`submodules/autobrr/internal/downloader/service.go:141,535`) that pushes to `/api/v3/release/push`, which is Radarr's `ReleasePushController`, present in Eros.
- **Cleanuparr:** models v3 as **Radarr-shaped** (`Cleanuparr.Domain/Entities/Radarr/Movie.cs`: "A movie in Radarr or Whisparr v3"), v2 as Sonarr-shaped, and has a `WhisparrV3Command`. Queue and manual-import records reuse the Radarr path.
- **Whisparr → Whisparr:** v3 to v3 only, via the "Whisparr" import list. **There is no v2 → v3 path** (§10).
- **Stash:** push-only GraphQL (scan, clean, identify against a stash-box, StashDB by default). StashDB UUIDs are the shared identity, so a Stash library identified against StashDB lines up with Eros scene ids.
- **Overseerr/Jellyseerr/Seerr:** no support.
- **Hosted services:** `api.whisparr.com` (closed source; aggregates StashDB, TPDB and TMDb, and computes performer/studio "works" and change feeds), `whisparr.servarr.com`, StashDB (user key), TMDb (embedded key), GitHub.

---

## 10. Outdated / legacy candidates
- **Indexers:** HDBits (irrelevant here). The Newznab presets Nzb.su and Usenet Crawler are defunct.
- **Download clients:** Hadouken, Vuze, Pneumatic, NzbVortex.
- **Notifications:** Twitter, Join, Prowl, and **Trakt** (a movie scrobbler, meaningless for scenes).
- **Parser leftovers from Radarr:**
  - German/TrueFrench tracker patterns;
  - the "PassThePopcorn torrent names" pattern (PTP itself is removed);
  - `ReportImdbId`/`ReportTmdbId` for scenes;
  - `SpecialEpisodeTitleRegex`/`SpecialEpisodeWordRegex`;
  - `Scene CleanTitleNoSeasonEpisode`.
- **Model:**
  - `MovieStatusType` `InCinemas`/`TBA` (meaningless for scenes);
  - `IsAvailable(delay)` ignores the delay;
  - `ItemType.Studio`/`ItemType.Performer` enum members on `MovieMetadata` (unused as item types);
  - `SceneSearchCriteria.Performer` (never set);
  - `AuthenticationType.Basic` (`[Obsolete]`).
- **Bugs spotted:**
  - Xbmc NFO writes the `tmdb` uniqueid (0) for scenes and never writes a StashDB/TPDB id (`XbmcMetadata.cs:245-257`).
  - Fuzzy scene matching returns the top score with no margin or tie check (`MovieService.cs:1294-1299`).
  - The collection exclusion message says "Studio: …" and uses the studio title (`AddMovieService.cs:334`).
  - The `CollectionTmdbId != null` check is always true for an `int` (`AddMovieService.cs:324`).
  - The Stash `CheckForError` TODO.
  - `{Movie Collection}` token has no handler.
  - Port and Branch env overrides are ignored.
  - FileList defaults to non-XXX categories.
  - `AGENTS.md` is stale (Redux, issue tracker location).
- **Removed v2 capability:** code-based completed-download tracking, the code index, the JAV naming template, and per-indexer query toggles.

---

## 11. Rewrite notes

### 11.1 What is genuinely unique and what is configuration
Almost all of Eros is **Radarr's movie pipeline with a second item type**. What is genuinely unique:

1. **Entity graph:** Scene/Movie items, **Studio** (container, optional parent network) and **Performer** (M:N with items), each a *monitorable source* with profile, root folder, tags, `AfterDate` and "monitor new items". In a unified model:
   - studio and performer are **"followable entities that expand into items"**, the same concept as Radarr collections, Lidarr artists, Sonarr series and Bookshelf authors;
   - add-time gates (entity exclusion, after-date, tag exclusion, with "exclude" or "add unmonitored + tag") are a generic **acquisition policy**.
2. **Metadata adapter:**
   - scenes, studios and performers from StashDB;
   - full movies from TMDb or TPDB;
   - "works" lists per performer and studio, and change feeds.
   - Today this is all one closed proxy (`api.whisparr.com/v4`). A direct implementation needs:
     - StashDB GraphQL (`findScene`, `queryScenes` by performer/studio/date, `findPerformer`, `findStudio`, and merges via `MergedIntoId`);
     - TPDB REST (key);
     - TMDb (key; adult movies).
   - Change feeds have to be replaced by `updated_at` polling.
3. **Scene parser:** ~15 date-first patterns + StashId + JAV/FC2 codes + studio episode numbers, taken ahead of the movie cascade. `ParsedMovieInfo.IsScene` decides the path.
4. **Scene matcher:**
   - studio (title + aliases) → `(studio, date)` candidate set with month/year widening;
   - `MatchMovies` graded precedence (title, code, performers incl. female-only, characters, title+performer, performers-not-title, contains);
   - verify-date/verify-code for studio-wide fallbacks;
   - optional fuzzy scoring.
   - Port it **with v2's rule of rejecting ambiguity applied to the fuzzy pass too** (require a margin, as `FindFuzzyMovieByYear` already does).
5. **Query templates:** `{studio|studioAlias|studioSearchTitle} × {yy.MM.dd | dd.MM.yyyy}`, `{studio} {title}`, `{studio} {code}`, `{title}`, `{title} {date}`, plus `tmdbid` for movies. This is a data-driven template list per media kind. Make the toggles **per indexer** (as v2 did), not global.
6. **Loose-file identification:** unmapped-file scan → remote search by `studio date performer` → match → move. It generalises to "identify unknown files against the metadata provider", which Lidarr and Bookshelf need too.
7. **Naming tokens:** Studio/Network, Release Date, Performers (all/female/male/other, alias variants), Code, StashId. Separate item-type formats.
8. **Stash integration:** a media-server-style notifier (scan/clean/identify) and a Stash-flavoured NFO. The NFO must carry the StashDB id.
9. **Qualities:** `VR` source/quality plus 5K–8K web resolutions.

**Expressible as pure configuration of the unified pipeline:**
- Radarr's download, queue, history, blocklist, custom formats, quality/delay/release profiles, import/rename, notifications, backups, health and auth.
- TMDb collections for full movies.

### 11.2 v2 vs v3 for the monolith

| Aspect | v2 (Sonarr-based) | v3 Eros (Radarr-based) | Better for the monolith |
|---|---|---|---|
| Item model | Site=Series, Year=Season, Scene=Episode | Scene or Movie = one item; `ItemType` flag | **v3**: no fake seasons; full movies supported |
| Performer | Embedded JSON on episodes; list source only | First-class table, M:N via credits, monitorable, after-date | **v3** |
| Studio | Series row (TPDB site id in `TvdbId`) | First-class, aliases, search-title override, parent network, monitorable | **v3** |
| IDs | TPDB ints stuffed in `TvdbId` fields | StashId/TpdbId/TmdbId/ImdbId columns + canonical ForeignId (StashDB for scenes) | **v3** (closer to D3's "external-id map per provider") |
| Metadata source | Proxy `api.whisparr.com/v3` (TPDB) | Proxy `api.whisparr.com/v4` (StashDB + TPDB + TMDb); StashDB direct for lists | **v3** (StashDB is open data); both proxy-dependent |
| Refresh | Full site refetch every 12h | Delta (`/changed`) + staleness rules, 24h | **v3** |
| Parser | 7 date patterns + 10 code variants | ~15 date patterns + StashId + 4 code patterns + episode codes + adult movie patterns | **v3**, but port v2's code variants |
| Same-date matcher | Graded, reject-if-ambiguous | Graded + characters/codes, reject-if-ambiguous, **but an unguarded fuzzy pre-pass** | v3 logic plus v2's strictness |
| Product codes / JAV | Indexed `ExternalId`; decisions, completed-download, import, JAV naming | `Code` column, no index, weaker use | **v2** |
| Query templates | Per-indexer toggles, `t=search` | Global toggles, studio aliases, code queries | Mixed: v3 templates with v2's per-indexer scope |
| Performer-driven acquisition | TPDb Performer import list → specific-episode monitoring | Performer/studio works sync + StashDB filtered lists + after-date gates | **v3** |
| File layout | Site folder, flat files | Folder per scene under studio/network | Neutral (template) |
| Schema compatibility | Sonarr-incompatible numbering | Radarr-incompatible numbering; no v2 importer | Neither; needs a one-time importer per version |
| Upstream hygiene | Daily cherry-pick bot vs Sonarr; ~1 month lag | Disposition ledger vs Radarr (0 behind) + Sonarr frontend | **v3** |
| Maturity | v2.2 stable (2026-08), 1k stars, still maintained | v3.6.2 stable (2026-09), 21 stable releases (irregular; weekly since September), single main maintainer, 0 open issues | v2 more established; v3 is where the domain work goes |
| Ecosystem | Prowlarr/autobrr (`/api/v3`), Cleanuparr (Sonarr-shaped) | The same, Radarr-shaped | Neutral |

**Recommendation for D10: close it as Option B (model on v3), with four v2 corrections.**

The adult kind in the monolith should be:
- an **item** (`kind=adult`, `subtype=scene|movie`, release date, code, runtime);
- an optional **Studio** container (with parent network, aliases and search-title);
- **M:N Performers**;
- a **per-provider external-ID map** (StashDB, TPDB, TMDb, IMDb; code as an indexed alternate identifier);
- Studio and Performer as **followable sources** that expand into items, with the generic acquisition-policy gates (exclusion, after-date, monitor-new-items).

It should reuse the movie pipeline (single file per item, Radarr decisioning) and the collection concept.

Take from v2:
1. **Product-code identity:** v2's 10 regex variants, a (non-unique) index, and use in decisions, completed-download tracking and import.
2. **Strict ambiguity rejection** everywhere, including fuzzy scoring (require a margin).
3. **Per-indexer query-template toggles and date formats.**
4. **XXX default categories for every indexer type.**

The metadata adapter should go **direct to StashDB (primary, scenes/studios/performers) with TPDB and TMDb for full movies**, per D3. The proxy's "works" and change-feed endpoints must be re-implemented locally.

A v2 importer is needed anyway, because no upstream path exists. Map:
- Series → Studio (TPDB site id → StashDB via TPDB's stash links, or by name);
- Episode → scene item (keep TPDB scene id, date, `ExternalId` → code; the StashDB id is resolved lazily);
- Actors → Performers.

Keep this import as a Phase 3 item (D2).

### 11.3 Divergence from Radarr (develop @ `c90668a5`)
- **Tracking:** backend cherry-picks from Radarr `develop`, with a disposition per commit. High-water `c90668a5`, **0 commits behind**. Deferred: restart-lifetime and the file-quality filter. Radarr frontend commits and parser commits are skipped by policy. The history merge base is `fa62f3f6` (Dec 2023, ~v5.2).
- **Frontend:** not Radarr's. It follows Sonarr `v5-develop` (24 commits behind): React 19, react-router 7, react-query + zustand, no Redux, no `.js` files.
- **Deliberately removed:**
  - `MovieTranslations`, certification country, movie-info language, keywords, in-cinema/physical/digital release dates, minimum availability (`AvailabilitySpecification`), `PendingSpecification`;
  - the Keyword and Studio auto-tag specs, and the `Modifier` enum with the CAM/TS/TC/WORKPRINT/DVDSCR/REGIONAL qualities and QualityModifier CF spec;
  - indexers PTP, Nyaa and TorrentPotato; import lists CouchPotato, Plex (both), RadarrList/IMDb/StevenLu, Simkl and Trakt;
  - the RQBit client;
  - `MetadataCheck` and `PTPOldSettingsCheck`;
  - the orphan-cleanup housekeepers for translations, credits and collections;
  - azure-pipelines;
  - the env overrides for Port/Branch.
- **Ported from Sonarr instead:**
  - `QualitySource` enum shape and quality ranks (mig 244);
  - tag-existing import lists (mig 245);
  - `Statistics/`, `TimeZone`;
  - download-client fallback (#925);
  - the whole frontend.
- **Adult additions:**
  - `ItemType`/`MetadataSource`, `Code`, multi-id columns;
  - Studio and Performer entities with services, refresh, works sync and editors;
  - Credits keyed by performer foreign id;
  - scene parser, matcher and fuzzy matching; scene search criteria and query templates;
  - StashDB import lists and the Stash notifier;
  - unmapped-file scan and remote identification;
  - paged filters and library search;
  - VR and 5K–8K qualities;
  - re-keyed exclusions with type and reason;
  - add-time exclusion gates;
  - GitHub updater;
  - `Whisparr_*` env prefix, Webhook performer/studio fields.
- **Schema:** fresh baseline + own 002–026 + upstream-numbered 236–242 + own 243–245. **Not upgrade-compatible with Radarr or v2.**
- **For the Go monolith:** porting Radarr's pipeline once and adding the following reproduces Eros with no other forked code:
  1. Studio/Performer followable entities;
  2. the StashDB/TPDB/TMDb adapter;
  3. the scene parser and matcher (§11.1 #3–4);
  4. query templates;
  5. loose-file identification;
  6. the Stash notifier.

  That covers about 17.8k diff lines plus ~9k Eros-only lines. The adult-specific logic is roughly 6–8k of that: Movies/Performers/Studios 2.2k, the SkyHook adapter ~1.8k, the matcher ~0.7k, the parser ~0.9k, search ~0.3k, import lists ~1.7k and identification ~0.4k. The rest is paging, UI support, local fixes and Sonarr ports.
