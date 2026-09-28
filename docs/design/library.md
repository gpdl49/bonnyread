# Library model and monitoring

Status: **proposal** (2026-09-28). Scope: the media kinds and their hierarchies, version slots (D4), files and how they relate to items, external ids, followed entities (D10), monitoring and "wanted", import lists and exclusions, requests as intent, availability, how metadata refresh touches the library, library scan at the model level, root folders and tags.

Other design areas, referenced by name and not designed here:
- **acquisition** (indexers, search, RSS/IRC);
- **release evaluation** (profiles, custom formats, cutoff, the decision engine);
- **download lifecycle** (tracked downloads, queue);
- **import & file management** (parsing, identification, import, rename, move);
- **metadata providers**;
- **runtime** (jobs, events, HA);
- **auth/API/clients**;
- **subtitles**.

Paths are relative to the repo root. `S/` = `submodules/sonarr/src/NzbDrone.Core`, `R/` = `submodules/radarr/src/NzbDrone.Core`, `L/` = `submodules/lidarr/src/NzbDrone.Core`, `B/` = `submodules/bookshelf/src/NzbDrone.Core`, `W3/` = `submodules/whisparr-eros/src/NzbDrone.Core`, `K/` = `submodules/kapowarr/backend`, `SE/` = `submodules/seerr/server`.

---

## 1. Outcomes to preserve

These are user-visible behaviours and correctness invariants. Each one names the reference that establishes it. Section 3 shows how the new model delivers each one (the O-numbers are cited there).

| # | Outcome / invariant | Evidence |
|---|---|---|
| O1 | Automatic acquisition (RSS, backlog) only targets **monitored** items under a **monitored** parent. Interactive search ignores monitoring. | `S/DecisionEngine/Specifications/RssSync/MonitoredEpisodeSpecification.cs:21-27` (skips the check when `MonitoredEpisodesOnly: false`; rejects if `Series.Monitored` is false) |
| O2 | **Missing** = monitored ∧ parent monitored ∧ released ∧ no file. For TV, "released" means `AirDateUtc + Runtime ≤ now`. Specials are excluded unless asked for. | `S/Tv/EpisodeRepository.cs:291-316` (`EpisodesWithoutFilesBuilder`, `BuildAirDateUtcCutoffWhereClause`); `submodules/sonarr/src/Sonarr.Api.V5/Wanted/MissingController.cs:50-57` |
| O3 | For movies, "released" is `MinimumAvailability`: TBA/Announced = always; InCinemas = cinema date; Released = min(physical, digital), falling back to cinema + 90 days, else never. Plus `AvailabilityDelay`. | `R/Movies/Movie.cs:77-118` |
| O4 | An album is missing if **any track of its monitored release** has no file. The album is the unit of want; the track is the unit of completeness. | `L/Music/Repositories/AlbumRepository.cs:95-105` |
| O5 | **Add-time monitor presets** are one-shot bulk choices: All, Future, Missing, Existing, FirstSeason, LastSeason, Pilot, Recent (90 days), Specials on/off, None, Skip (TV); All/Future/Missing/Existing/Latest/First/None (music, books); all/missing/none (comics). | `S/Tv/EpisodeRepository.cs:194-268`, `S/Tv/MonitoringOptions.cs`; `L/Music/Model` `MonitorTypes`; `K/implementations/volumes.py:625-660` |
| O6 | **New-item policy.** New seasons are monitored only if the series' `MonitorNewItems=All`. New episodes inherit their season's flag. Season 0 is never auto-monitored. New albums follow `None/All/New`, where "New" means newer than the newest existing album. Comics use `monitor_new_issues`. v3 performers and studios use `WhisparrMonitorNewItems`. | `S/Tv/RefreshSeriesService.cs:140-160`, `S/Tv/RefreshEpisodeService.cs:146-155`; `L/Music/Services/MonitorNewAlbumService.cs:22-60`; `K/internals/db.py:398`; whisparr-eros.md §2.3 |
| O7 | **Refresh never re-monitors what the user unmonitored.** As a safety net, old episodes that show up during a refresh of a series with no existing episodes are unmonitored so they don't trigger a surprise mass download. | `S/Tv/RefreshEpisodeService.cs:157-187` (`UnmonitorReaddedEpisodes`) |
| O8 | Optionally **unmonitor an item when its file is deleted**, but not when the reason is an upgrade, a manual override or the file going missing from disk. | `S/Tv/EpisodeService.cs:265-290`; Kapowarr `unmonitor_deleted_issues` (`K/internals/settings.py:112`, `K/implementations/file_matching.py:251`) |
| O9 | File↔item cardinalities: one file may hold several episodes, but an episode has at most one file (`Episode.EpisodeFileId`). A movie has one file. A track has one file, one file may hold many tracks, and unmapped files exist. A book edition may have many files (multi-part audiobooks). Comics are **M:N** (one archive covers #1–6). | verification/decision-premises.md §D4; `L/MediaFiles/TrackFile.cs`; bookshelf.md §2 (`BookFile.Part/PartCount`); `K/internals/db.py` `issues_files` |
| O10 | There is **one target edition per album or book**. With `AnyReleaseOk`/`AnyEditionOk`, import may switch the target to the release it actually identified. | `L/MediaFiles/TrackImport/ImportApprovedTracks.cs:118-131`; `B/Books/Model/Book.cs:42` |
| O11 | **Metadata profiles** decide which albums or books exist for an artist or author (types, statuses, popularity, language, …). A reserved **"None"** profile is used when the user added only one album or book. | lidarr.md §3.5; bookshelf.md §2 (`MetadataProfileService.FilterBooks`) |
| O12 | A monitored **collection** auto-adds its released or in-cinema movies with the collection's profile, root and tags, skipping exclusions. | `R/Movies/RefreshCollectionService.cs:124-157` |
| O13 | **Followed performers and studios** expand into scenes and movies. Adds are gated by entity exclusions, tag exclusions and **AfterDate** cut-offs, either rejecting the item or adding it unmonitored with a tag. | `W3/Movies/AddMovieService.cs:265-420`; whisparr-eros.md §2.3 |
| O14 | **Import lists** add items with monitor and search options and respect **exclusions**. Cleanup levels are LogOnly / KeepAndUnmonitor / KeepAndTag / RemoveAndKeep / RemoveAndDelete. Cleanup runs **only when every list fetched successfully**. | `S/ImportLists/ImportListSyncService.cs:348-400`; `R/ImportLists/ImportListSyncService.cs:44-63,183-228` |
| O15 | **Requests** go PENDING → APPROVED/DECLINED, can be per season, can ask for 4K, and can be auto-approved. Users see Available / Partially available / Processing, and requesters are notified when the item becomes available. | seerr.md §2–§3; `SE/lib/scanners/baseScanner.ts:357-410` |
| O16 | Seerr's "Available" for a season means **every** episode, aired or not, has a file (`totalEpisodeCount`), so an airing season shows as *partially available*. | `SE/lib/scanners/sonarr/index.ts:188-221`; `S/SeriesStats/SeriesStatisticsRepository.cs:80-84` |
| O17 | **Multiple external ids per thing, surviving upstream merges**: TVDB/TMDb/IMDb/TVMaze/MAL/AniList for series; MBID `Old*Ids`; StashDB `MergedIntoId`; StashDB/TPDB/TMDb/IMDb for adult content. | sonarr.md §2; lidarr.md §2; `W3/Movies/MovieMetadata.cs` |
| O18 | TV numbering per episode: TVDB season/episode, **absolute**, **scene season/episode/absolute** (XEM), `UnverifiedSceneNumbering`, and **specials placement** (`AiredBefore/After*`). Series types are Standard/Daily/Anime. | `S/Tv/Episode.cs`, `S/Tv/Series.cs` |
| O19 | Items deleted upstream get a **Deleted** status and a health warning. They are not silently removed. | `R/Movies/RefreshMovieService.cs:87-91`; `RemovedSeriesCheck`/`RemovedMovieCheck` |
| O20 | Adult **product codes** work as (non-unique) alternate ids for matching. | D10; `submodules/whisparr/.../Migration/221_add_episode_external_id_index.cs` |
| O21 | Comic **special versions** (TPB, one-shot, HC, omnibus, volume-as-issue) are inferred per volume, and the user can override and lock the value. A TPB file never matches a normal volume. Issue numbers are text with a sortable float. | `K/implementations/volumes.py:1276-1340`; `K/implementations/matching.py:154-221` |
| O22 | **Root folders** carry add defaults (Lidarr/Bookshelf), report free space, and list unmapped folders for library import. | `L/RootFolders/RootFolder.cs`; sonarr.md §3 RootFolders |
| O23 | **Tags** on titles scope indexers, clients, notifications and delay/release profiles. Auto-tagging rules add and remove tags. | sonarr.md §2 AutoTagging, §3 Tags |

---

## 2. How the references do it, and what is accidental

### 2.1 Per-app shape

| App | Hierarchy | Monitoring state | Files |
|---|---|---|---|
| Sonarr | Series → Season (embedded JSON) → Episode | Three stored flags: `Series.Monitored`, `Season.Monitored` and `Episode.Monitored`. `EpisodeMonitoredService` keeps them loosely in sync: a season is monitored if it has monitored episodes (`S/Tv/EpisodeMonitoredService.cs:60-90`). Add presets are applied once via `AddOptions`. | `EpisodeFile` holds N episodes; each episode has an `EpisodeFileId` back-pointer |
| Radarr | Movie (user) + MovieMetadata (shared, mig 207); Collection linked by `CollectionTmdbId`, not an FK | `Movie.Monitored`, `MinimumAvailability`; `Collection.Monitored` drives auto-add | One `MovieFileId` |
| Lidarr | Artist + ArtistMetadata → Album (release group) → AlbumRelease (one `Monitored`) → Track | `Artist.Monitored`, `MonitorNewItems`, `Album.Monitored`, `AlbumRelease.Monitored`, `AnyReleaseOk` | `TrackFile` with an absolute path, N tracks per file; `AlbumId=0` means unmapped |
| Bookshelf | Author + AuthorMetadata → Book (work) → Edition (one `Monitored`); Series is metadata only | Same as Lidarr, plus `AnyEditionOk` | `BookFile` → Edition, `Part/PartCount` |
| Whisparr v2 | Site = Series, Year = Season, Scene = Episode; TPDB ids stored in `TvdbId` | Sonarr's | Sonarr's |
| Whisparr v3 | Movie row with `ItemType` Scene/Movie; Studio and Performer tables (auto-created, unmonitored); performers linked through a JSON id list plus Credits, with no join table | `Studio/Performer.{Monitored, MoviesMonitored, WhisparrMonitorNewItems, AfterDate}` | One file; `ImportFiles` for unmapped files |
| Kapowarr | Volume → Issue | `volumes.monitored`, `monitor_new_issues`, `issues.monitored` | `files` + `issues_files` (M:N) + `volume_files` |
| Seerr | Media (tmdbId, type) → Season; MediaRequest → SeasonRequest | None of its own. It mirrors Arr state as `status`/`status4k` and pushes monitoring into the Arrs over HTTP. | None; it infers availability by scanning the Arrs and media servers |

### 2.2 Accidental complexity

1. **Redundant monitored flags at several levels.** Sonarr stores three flags. The decision engine reads only series and episode (O1). `Season.Monitored` really means "monitor new episodes in this season", plus a UI aggregate, and it is kept consistent by the heuristics in `EpisodeMonitoredService.cs:60-140`. Lidarr likewise stores `Album.Monitored` and a separate "exactly one monitored release".
2. **Catalog and library are tangled together.** Radarr (`MovieMetadata`, mig 207), Lidarr (`ArtistMetadata`) and Bookshelf (`AuthorMetadata`) each split "metadata" from "library" late and differently, and leave pass-through properties behind (radarr.md §11 "Accidental complexity"). Whisparr v3 auto-creates **unmonitored** Studio/Performer rows for every entity it sees (`RefreshMovieService.cs:162-225`), so the library tables double as a metadata cache.
3. **Import-list cleanup doesn't know why an item is in the library.** Sonarr's and Radarr's `CleanLibrary` checks *every* series or movie against the union of list items (`S/ImportLists/ImportListSyncService.cs:361-400`, `R/ImportLists/ImportListSyncService.cs:183-228`). A manually added series that isn't on a list gets unmonitored or tagged, and with Radarr's `removeAndDelete` a manually added movie is **deleted with its files**. The Arrs don't record which list, request or user added an item. Seerr patches this with per-user Arr tags `"{id}-{name}"` (seerr.md §9.7).
4. **Derived exclusions are stored as rows.** Whisparr v3 writes a per-scene exclusion row with `Reason = StudioExclusion/PerformerAfterDate/…` when a gate fires (`W3/Movies/AddMovieService.cs:267-380`). Removing the studio exclusion later leaves the scene rows behind. The gate should be a predicate evaluated at add time.
5. **Metadata profiles delete catalog rows.** In Lidarr and Bookshelf, changing a metadata profile and refreshing deletes albums or books from the database, and `CleanupOrphanedAlbums` then tidies up (lidarr.md §3.5). A *view* choice ends up destroying cached metadata.
6. **Provider id columns tied to one provider.** Bookshelf puts Goodreads *or* Hardcover ids in the same `Foreign*` columns, so switching needs a fresh database (bookshelf.md §2). Whisparr v2 stores TPDB ids in `TvdbId`. Sonarr's `TvRageId` is dead. Each app has its own exclusion table keyed on its own id type (int TVDB, int TMDb, v3 `ForeignId+Type`).
7. **Availability is mirrored, not queried.** Seerr keeps `Media.status/status4k` and `Season.status/status4k`, filled by four scanners and a daily `availabilitySync`, and marks media DELETED only when it is absent from both the Arr and the media server (seerr.md §3, §9.6). That is about 5k lines to approximate a join the Arr can answer exactly.
8. **Versions are implemented as extra instances.** 4K is a second Radarr/Sonarr plus Seerr's `is4k` servers, with every status field duplicated (`SE/entity/Media.ts:140-233`). Ebook and audiobook need two Bookshelf instances (`submodules/bookshelf/README.md:9-11`). Instances are linked by instance-to-instance import lists.
9. **One concept, many "follow" mechanisms.** Series `MonitorNewItems`, per-season flags, artist and author `MonitorNewItems`, Radarr collections, v3 studio/performer works sync, Kapowarr `monitor_new_issues`, TMDb Person/Company/Keyword import lists, StashDB performer/studio lists and MusicBrainz Series lists all mean "keep this set of items in sync with an entity upstream".
10. **Seven copies of the Plex watchlist.** It exists as a Plex API list and a Plex RSS list in each of Sonarr, Radarr and Whisparr v2, plus Seerr's auto-request sync (comparison.md §2.2), each with its own semantics.
11. **FAILED request state.** Seerr's FAILED status and retry exist because approval → Arr POST is a network call made from a detached promise (seerr.md §9.7).
12. **Unmapped files handled three ways**: Lidarr `AlbumId=0`, v3 `ImportFiles` (mig 022), and Sonarr/Radarr "unmapped folders" on root folders.
13. **Name overloading.** "Available" in Radarr means *released enough to search* (`IsAvailable`). In Seerr it means *has files*. This design keeps the two apart: **released** versus **availability**.

---

## 3. Proposed design

### 3.1 Concepts

- **Catalog.** Metadata-owned, shared and refreshable. It holds no user decisions. It contains every entity and item the metadata providers told us about, whether or not the user wants it. It replaces `*Metadata` tables, auto-created v3 performers and studios, collection member movies, and list "discover" caches.
- **Entity.** A catalog node that groups or credits items: series, season, artist, author, comic volume, studio, network, performer, person, movie collection, book series, label, publisher.
- **Item.** The **unit of want**: the smallest thing a user monitors and that acquisition targets. Kinds: `episode`, `movie`, `album`, `book`, `scene`, `adult_movie`, `issue`.
- **Edition and track.** Variants and components of an item, for music (release → tracks) and books (edition). They define what "complete" means but are not units of want.
- **Slot (D4).** A named version kind, defined once per media kind, for example movie {`HD`, `UHD`}, book {`Ebook`, `Audiobook`}, music {`Lossless`, `Lossy`}. Every item is monitored, covered and available **per slot**.
- **Title.** The library root the user added and that owns folders. It points at an entity for series, artist, author and comic volume, and at an item for movie, scene and adult movie. Per slot it has a root folder, path, quality profile and pause flag (**TitleSlot**).
- **ItemSlot.** Per (item, slot): the **monitored** flag, the target edition, and a coverage summary. This is the only stored monitoring state.
- **Follow (D10, generalised).** A durable policy "keep the members of entity E in the library for slot S": which new members to add, which to monitor, date cut-offs and defaults. Series `MonitorNewItems`, season monitoring, artist and author new items, collections, performers, studios, persons, book series and comic `monitor_new_issues` are all follows.
- **Intent.** A recorded *reason* for something to be in the library: a manual add, a user request, an import-list entry, a watchlist, a follow expansion or a disk import. Seerr requests are intents with an approval workflow. Intents give provenance, drive fulfilment, and scope retraction.
- **Exclusion.** "Never add this automatically" (or "never let users request it"), keyed by external id or catalog node.
- **MediaFile / FileLink.** Every media file under a root folder. FileLink is the M:N "this file contains this item (and track or edition)" relation. A file belongs to exactly one slot.
- **Wanted, missing, cutoff-unmet and availability are queries**, never stored state (§3.5, §3.8).

**Invariant M (monitoring mutations).** `ItemSlot.monitored` changes only through:
- (a) a user action;
- (b) fulfilment or retraction of an intent (a one-shot mutation);
- (c) a follow's policy applied to **newly appearing** members;
- (d) the auto-unmonitor-on-delete setting (O8).

Metadata refresh never flips an existing flag (O7).

### 3.2 Schema sketch (ent style)

Conventions:
- Every table has `id` (int64) and `created_at`/`updated_at`.
- `enum` fields are Go-typed.
- "exactly one of" edges are enforced with a CHECK constraint (`entsql.Annotation{Checks: …}`) and in the service layer.
- There are **no partial unique indexes** (MySQL has none, D13). Uniqueness uses NULL-tolerant composite indexes.

#### Catalog

```go
// Entity: a catalog node that groups or credits items.
Entity {
  kind          enum(series, season, artist, author, comic_volume, studio, network,
                     performer, person, movie_collection, book_series, label, publisher)
  title, sort_title, clean_title  string
  status        enum(continuing, ended, upcoming, active, inactive, unknown)
  year          int?         // series/volume start year
  number        int?         // season number; volume number
  attrs         json         // kind-specific non-queried data (overview, images, genres, ratings,
                             // gender/birthdate for performers, special_version for comic_volume…)
  catalog_state enum(stub, complete, tombstoned)   // stub = ids+name only, still being fetched
  refreshed_at  time?
  edges: parent -> Entity (season→series, studio→network)          // strict tree
         superseded_by -> Entity                                   // upstream merges (MBID, StashDB MergedIntoId)
  index: (kind, clean_title); (parent_id, number)
}

// Item: the unit of want.
Item {
  kind          enum(episode, movie, album, book, scene, adult_movie, issue)
  title, clean_title string
  release_at    time?        // UTC instant; episodes = air date/time
  release_precision enum(instant, day, month, year, unknown)
  runtime_min   int?
  sort_key      float        // episode number / issue calculated number / position
  catalog_state enum(active, tombstoned)
  edges: container -> Entity  (episode→season, album→artist, book→author(primary),
                               issue→comic_volume, scene→studio; movie/adult_movie: none)
         superseded_by -> Item
         editions <- Edition; ext 1:1 per kind (below)
  index: (container_id, sort_key); (kind, release_at); (kind, clean_title)
}

// Kind extensions, 1:1 with Item (typed columns the pipeline queries).
EpisodeInfo { season_number, episode_number int; absolute_number int?
              scene_season, scene_episode, scene_absolute int?; unverified_scene bool
              aired_before_season, aired_before_episode, aired_after_season int?   // specials
              finale enum? }                 index (season_number, episode_number), (absolute_number)
MovieInfo   { in_cinemas_at, digital_at, physical_at time?; original_language string
              status enum(tba, announced, in_cinemas, released) }
AlbumInfo   { primary_type enum; secondary_types []enum }
BookInfo    { }                               // (work-level; most data lives on Edition)
IssueInfo   { number_text string /* "1", "½", "1.MU" */ ; is_annual bool }
SceneInfo   { code string? }                  index (code) — NON-unique (D10 restore #1, O20)

// Edition: a variant of an item.
Edition {
  kind       enum(album_release, book_edition)
  title, format string; language string?; release_at time?
  media      json            // [{number, format:"CD"/"Vinyl"/"Digital", name}] for albums
  is_audio   bool            // books: audiobook edition vs text
  status     enum(official, promotion, bootleg, pseudo)?   // albums
  track_count int
  edges: item -> Item (required); tracks <- Track
}
Track { medium int; position int; number_text string /* "A1" */; title string
        duration_ms int; edges: edition -> Edition; artist -> Entity (credit, may differ from album artist) }
        unique (edition_id, medium, position)

// Association (credit/membership), M:N, not containment.
ItemLink   { role enum(performer, cast, crew, collection_member, series_entry, co_author, track_artist)
             position_text string?; position float?; attrs json
             edges: entity -> Entity; item -> Item }       unique (entity_id, item_id, role)
EntityLink { role enum(member_of, alias_of, …); edges: from, to -> Entity }   unique (from_id, to_id, role)
ItemRelation { role enum(collects); edges: from, to -> Item }  // comic TPB → issues (§4.4); unique (from,to,role)

// External ids: one map for every provider (D3d), for every node type.
ExternalID {
  provider   enum(tvdb, tmdb, imdb, tvmaze, mal, anilist, anidb, musicbrainz, acoustid, hardcover,
                  openlibrary, isbn, asin, goodreads, stashdb, tpdb, comicvine, metron, …)
  namespace  string          // e.g. musicbrainz: artist|release_group|release|recording|track
  value      string
  status     enum(current, alias)          // alias = superseded id kept so old tags/lists still resolve (O17)
  edges: exactly one of entity|item|edition|track
  unique (provider, namespace, value)
}

// Alternate titles: display, search, parse (scene mappings, XEM names, TMDb alt titles,
// translations, studio aliases, user search overrides).
AltTitle { title, clean_title string; language string?
           source enum(metadata, scene_mapping, xem, user); usage enum(display, search, parse, all)
           season_scope int?          // scene titles valid only for one season
           edges: exactly one of entity|item }
```

#### Library

```go
Slot {           // D4; defined per kind, a small stable set
  kind  enum(tv, movie, music, book, adult, comic); name string; position int
  attrs json     // e.g. books: {"format":"audio"}; used by identification bias
  default_quality_profile -> QualityProfile; default_root_folder -> RootFolder
  unique (kind, name)
}
RootFolder {
  path string unique; kind enum; edges: slot -> Slot?   // null = any slot of the kind
  add_defaults json      // Lidarr-style defaults for disk auto-add (O22)
}
MetadataProfile { kind enum(music, book); rules json; reserved_none bool }   // O11

Title {
  kind  enum(series, movie, artist, author, comic_volume, scene, adult_movie)
  edges: exactly one of entity -> Entity | item -> Item   ; unique(entity_id), unique(item_id)
  series_type enum(standard, daily, anime)?; use_scene_numbering bool?   // TV user overrides
  overrides json                                   // e.g. comic special_version + locked (O21)
  added_by -> User?; tags M2M Tag via TitleTag
}
TitleSlot {
  edges: title -> Title; slot -> Slot; root_folder -> RootFolder; quality_profile -> QualityProfile
  path string                 // title folder for this slot, relative to root_folder (see §6 I1)
  monitored bool              // pause/resume: the "parent monitored" gate of O1
  minimum_availability enum(announced, in_cinemas, released)?   // movies (O3), per slot
  unique (title_id, slot_id)
}
ItemSlot {
  edges: item -> Item; slot -> Slot; title_slot -> TitleSlot
  monitored bool
  coverage enum(none, partial, complete)    // maintained transactionally by FileLink writes
  target_edition -> Edition?; edition_locked bool   // O10: locked = !AnyReleaseOk/AnyEditionOk
  last_search_at time?
  unique (item_id, slot_id)
  index (slot_id, monitored, coverage)
}

Follow {
  edges: entity -> Entity; slot -> Slot
  add    enum(none, all, released, profile)   + metadata_profile -> MetadataProfile?
  monitor enum(none, all, from_date)          + from_date time?
  after_date time?          // gate: members released before this are not admitted (v3 AfterDate)
  gate_action enum(skip, add_unmonitored)   + gate_tag -> Tag?   // v3 "always exclude … tag"
  item_kinds []enum         // e.g. performer: [scene] and/or [adult_movie] (v3 Monitored vs MoviesMonitored)
  search_on_add bool
  defaults json             // quality profile / root folder for Titles this follow creates
  tags M2M Tag
  unique (entity_id, slot_id)
}

Intent {
  origin   enum(manual, user_request, import_list, watchlist, follow, disk_import)
  status   enum(pending, approved, declined, withdrawn)
  edges: target -> exactly one of Entity | Item      // series/artist/author/volume/performer or movie/album/book/scene
         scope_entities M2M Entity                   // requested seasons; empty = whole target
         slot -> Slot
         title -> Title?                             // set on fulfilment
         requested_by -> User?; decided_by -> User?
         import_list -> ImportList?; follow -> Follow?
  preset   enum(all, future, missing, existing, first_group, last_group, pilot, recent, none, skip)?
  include_specials bool
  overrides json            // quality profile / root folder / tags chosen by an advanced request
  search_on_fulfil bool; ignore_quota bool; is_auto bool
  fulfilled_at time?        // preset applied + monitoring set (one-shot)
  index (target_*, slot_id, status); (import_list_id, status)
}

ImportList {
  provider enum(trakt, plex_watchlist, imdb, tmdb_list, anilist, mal, simkl, lastfm, spotify,
                hardcover, musicbrainz_series, stashdb_filter, custom_json, rss, …)
  kind enum; settings json; enabled bool
  slot_templates json      // per slot: quality profile, root folder
  preset enum; follow_policy json?   // e.g. Lidarr EntireArtist → create Follow(artist)
  approval enum(auto, manual)
  retraction enum(none, log, unmonitor, remove_keep_files, remove_delete_files)   // O14, scoped (§3.7)
  owner -> User?           // per-user lists (Plex watchlist)
  tags M2M Tag
}

Exclusion {
  provider, namespace, value string?          // external ref, so it works before the catalog has the node
  edges: entity -> Entity? | item -> Item?    // or a catalog node
  applies_to enum(auto_adds, user_requests, both)
  reason enum(manual, removed_with_exclusion, keyword_rule); note string
  created_by -> User?
  unique (provider, namespace, value)
}

MediaFile {
  path string unique (see §6 I1); edges: root_folder -> RootFolder; slot -> Slot?; title -> Title?
  size int64; mtime time; state enum(present, missing)
  quality json; languages []string; release_group, scene_name string?; media_info json
  part, part_count int?        // multi-file audiobooks
  last_identify_at time?       // Bookshelf LastRemoteSearchTime: don't re-query unmatched files
}
FileLink {
  edges: file -> MediaFile; item -> Item; edition -> Edition?; track -> Track?
  source enum(parsed, identified, manual)
  unique (file_id, item_id, track_id)
}

Tag { label unique }   TitleTag { edges: title, tag; source enum(manual, rule); rule -> AutoTagRule? }
```

Uniqueness and integrity rules the service enforces in a transaction (with a row lock on `ItemSlot`):
- **FileLink slot consistency:** `file.slot == item_slot.slot` for each link, and `file.title` is the item's title.
- **Coverage exclusivity:** in `exclusive` mode (TV, movie, music, book, adult), every (item, slot, track) has at most one present covering file. Importing an overlapping file replaces the old file. The same-episodes grab and import rules (Sonarr `SameEpisodes*`) are the release-evaluation and import areas' job. In `overlapping` mode (comics), overlap is allowed (O9, O21).
- **Container membership:** an item joins the library only under the title of its layout root. That is the item itself for movie, scene and adult movie. For the other kinds it is the nearest ancestor entity of the title's kind (episode → season → **series**). If that title doesn't exist, it is created with a `Follow(add=none)`, which reproduces Lidarr's and Bookshelf's "None" profile for single-album or single-book adds (O11).

### 3.3 How each media kind maps

| Kind | Title → | Entities | Item (unit of want) | Edition / track | Coverage rule | Typical follows |
|---|---|---|---|---|---|---|
| TV | series entity | series; season (child; S0 = specials); network | episode (`EpisodeInfo`) | – | ≥1 present file linked (exclusive; multi-episode files link N episodes) | series (new seasons), each season (new episodes) |
| Movie | movie item | movie_collection, person | movie (`MovieInfo`) | – (edition names are a *file* attribute, handled by release evaluation) | ≥1 file | collection, person (cast/crew), keyword/company lists |
| Music | artist entity | artist; label | album = release group (`AlbumInfo`) | release (`Edition`) → `Track` | every track of `target_edition` (or of some edition if unlocked) has a linked file (exclusive per track) | artist (+ metadata profile), MusicBrainz series |
| Book | author entity (primary) | author; book_series; publisher | book = work | edition (`is_audio`); files carry part/part_count | a file set for one edition matching the slot's format, all parts present | author (+ metadata profile), book_series (**new**: Bookshelf can't monitor series) |
| Adult | scene or adult_movie item | studio (→ network), performer, movie_collection | scene / adult_movie (`SceneInfo.code`) | – | ≥1 file | studio, performer (with after_date, item_kinds) |
| Comic | comic_volume entity | comic_volume (special_version in attrs, user override in `Title.overrides`), publisher | issue (`IssueInfo`, `sort_key` = calculated number) | – | ≥1 linked file (overlapping mode) | volume (`monitor_new_issues`) |

Kind-specific notes:
- **Anime and scene numbering** are data on `EpisodeInfo`, filled by the metadata providers' TVDB/XEM/anime-list mapping (D3a). The item's identity is its **TVDB episode id** (`ExternalID`), never its numbers. Renumbering upstream updates `EpisodeInfo` in place, so FileLinks survive and a rename follows (§4.1). `AltTitle(season_scope)` holds per-season scene titles.
- **Daily series** are `series_type=daily`. Items are matched by `release_at` date by the import area. Nothing changes in the model.
- **Specials** are episodes in season 0 with placement fields. Season 0 is never auto-followed (O6), and specials are excluded from "missing" unless `include_specials` is set (O2).
- **v3's scenes versus full movies** is `Item.kind`, not a flag on a shared row. A studio's scenes contain `container → studio`. Performers are `ItemLink(role=performer)`.
- **Special versions for comics.** In ComicVine a TPB or omnibus is its own volume. It is a `comic_volume` entity with `attrs.special_version=tpb`, holding one issue item. The optional `ItemRelation(collects)` is described in §4.4.

### 3.4 Monitoring: presets, follows, new members

**Presets (O5)** are pure predicates over a title's current members for one slot, applied once by `Intent` fulfilment (or a user bulk action). Each sets `monitored = P(item)` for every in-scope `ItemSlot`:

| Preset | Predicate P(item) | Also creates follows |
|---|---|---|
| all | not a special (unless `include_specials`) | series + every non-zero season (TV); `Follow(add=profile, monitor=all)` (artist/author/volume) |
| future | `release_at` is null or ≥ now | series (monitor new seasons only if the title is continuing or upcoming), the last season; `monitor=from_date(now)` for artist/author/volume |
| missing / existing | coverage = none / ≠ none in this slot | none added |
| first_group / last_group | season = first/last (TV); album/book = first/latest (Lidarr "First/Latest") | the last season, for last_group |
| pilot | S(first)E1 | – |
| recent | release_at ≥ now − 90 days | as for future |
| none | false | none |
| skip | leave existing flags alone | – |

Sonarr's rule that "season X is monitored" ⇔ "Follow(season X, slot) with monitor=all" is set at the same time as its episodes, so presets and follows can't disagree.

**Follow expansion.** This runs when the catalog reports new members of E, when a follow is created, or when a follow's policy is edited. Editing affects only future members; a separate "apply to existing" action runs a preset. For each new member m of E, for the follow's slot:

```
admit(m, F) :=  m.kind ∈ F.item_kinds
             ∧ add-policy(F.add, m)          // all | released(m) | metadata_profile.accepts(m)
             ∧ ¬excluded_auto(m)             // direct exclusion, or exclusion of any linked entity:
                                             //   container, studio, performer, collection, book series
             ∧ m.release ≥ F.after_date
             ∧ ∀ performer p credited on m with a follow Fp: m.release ≥ Fp.after_date     // v3 rule (O13)
if admit fails on a date/entity gate and F.gate_action = add_unmonitored: admit with monitored=false + F.gate_tag
if admitted:
   ensure Title(layout root of m) [+ Intent(origin=follow, follow=F)]
   ensure TitleSlot(title, F.slot) using F.defaults (or the slot's defaults)
   ensure ItemSlot(m, F.slot) with monitored = monitor-policy(F.monitor, m)
   if m is a season (TV): ensure Follow(season, slot, monitor = (series follow monitor ≠ none && m.number ≠ 0))
   if F.search_on_add ∧ monitored ∧ released: enqueue search job (acquisition area)
```

This reproduces each reference behaviour:

| Reference behaviour | Follow configuration |
|---|---|
| Sonarr `MonitorNewItems=All/None` | Follow(series, add=all, monitor=all/none). New seasons create season follows. Season 0 gets monitor=none (O6). |
| Sonarr episode inherits `Season.Monitored` | Follow(season, add=all, monitor=all/none) |
| Lidarr `MonitorNewItems=All/None/New` | Follow(artist, add=profile, monitor=all/none/from_date(follow time)). See §6 Q6 for how this differs from "New". |
| Radarr collection | Follow(collection, add=released, monitor=all) (O12) |
| v3 performer/studio | Follow(entity, add=all, monitor=all/none, after_date, item_kinds, gate_action) |
| Kapowarr `monitor_new_issues` | Follow(volume, add=all, monitor=all/none) |
| TMDb Person/Company/Keyword lists, StashDB performer/studio lists, MusicBrainz Series lists | Follows on catalog entities, whenever the entity type exists in the catalog. Lists stay for curated external lists. |

**O7 safety.** For containment follows (series, artist, author, volume), when a refresh brings in members whose `release_at < now − 14d` and the title already had members:
- `monitor=all` still applies, which matches Sonarr's behaviour for series with existing episodes;
- the library also emits a `LateMembersAppeared` event (notification and health);
- auto-search for those members is suppressed until the user confirms.

On a title's **first** expansion the preset governs instead, which covers Sonarr's "no existing episodes" branch. This keeps Sonarr's intent (no surprise mass downloads) without its heuristic mutation.

### 3.5 Wanted computation (per slot)

These are queries over `ItemSlot ⋈ Item ⋈ TitleSlot [⋈ kind info]`. The acquisition and release-evaluation areas consume them.

```
monitored(i,s)  := ItemSlot(i,s).monitored ∧ TitleSlot(title(i),s).monitored            -- O1
released(i,s)   := kind switch
                    episode:      release_at ≠ null ∧ release_at + runtime ≤ now          -- O2
                    movie:        is_available(MovieInfo, TitleSlot.minimum_availability, delay)  -- O3
                    album|book|issue|scene|adult_movie:
                                  release_at ≠ null ∧ date(release_at) ≤ today(UTC)
missing(i,s)    := monitored ∧ released ∧ coverage = none|partial ∧ (i not special ∨ include_specials)
cutoff_unmet(i,s):= monitored ∧ coverage = complete ∧ profile(TitleSlot).upgradeable(current files)
                                                  -- evaluated by release evaluation; not stored
grabbable(i,s)  := monitored ∧ kind early-grab policy (e.g. TV accepts unaired monitored episodes from RSS;
                    movies require released; albums: EarlyRelease rule)  -- owned by release evaluation
```

Notes:
- `coverage` is stored on `ItemSlot` (none/partial/complete) because deriving it for albums means counting tracks per edition on every wanted query. It changes **only** inside the import, delete and rescan transactions that write FileLinks, with the ItemSlot row locked. This is the same trade Sonarr makes with `EpisodeFileId`, generalised.
- `partial` exists only for kinds with components: an album missing tracks, an audiobook missing parts. Partial counts as missing (O4).
- Cutoff-unmet depends on profiles that change, so it is never stored.
- Cleanuparr's Seeker and Profilarr's upgrade selector recompute these today from mirrors. Now they are the backlog scheduler's input query (acquisition area).

### 3.6 State transitions

**ItemSlot** (derived display state, per slot):

```
            ┌───────────── user/intent/follow sets monitored=false ─────────────┐
            ▼                                                                   │
  [unmonitored] ──monitor──> [monitored, unreleased] ──release passes──> [missing] ──grab──> [queued]*
        ▲                         │                                          ▲  │              │
        │                         └────── early grab (RSS) ────────────────────┼──┘  import ok  ▼
        │                                                                    │         [covered]
        │           file deleted (reason ∉ {upgrade, manual, missing-from-disk}) ∧ auto-unmonitor│
        └────────────────────────────────────────────────────────────────────┼──────────────────┤
                                               file deleted/missing, no auto-unmonitor ─────────┘
  [covered] ── cutoff unmet ∧ monitored ──> [upgrading]  (still covered)
```
\* `queued` comes from the download-lifecycle area's tracked download ↔ (item, slot) link.

**Intent**: `pending → approved → (fulfilled_at set)`, `pending → declined`, `pending|approved → withdrawn`. There is **no FAILED state**. Fulfilment happens in-process, in the approval transaction, as soon as the target's catalog node is `complete`. Otherwise a durable job (runtime area) fulfils it when the stub completes. Errors retry; they are not surfaced as request failures.

**Catalog node**: `stub → complete → tombstoned → purged`.
- A tombstone is set when the provider reports deletion (O19). It stays while referenced by a Title, FileLink or Intent, and surfaces as a health issue.
- Purge happens only when unreferenced.
- Merges (`superseded_by`): the survivor takes over ExternalIDs (as `alias`), ItemSlots, FileLinks, Intents and Follows. Unique collisions resolve in favour of the older ItemSlot.

**MediaFile**: `present ↔ missing` (library scan), with links deleted when a file is removed. A file with no links is **unmapped**. This one representation replaces Lidarr's `AlbumId=0`, v3's `ImportFiles` and Sonarr's unmapped folders.

### 3.7 Requests, lists and follows feed monitoring through Intents

Every path that adds to the library creates an `Intent`, so every Title has at least one reason to exist:

| Source | Intent | Fulfilment |
|---|---|---|
| Manual add (UI/API) | origin=manual, approved | Title + TitleSlot(s) + preset + follow(s) |
| User request (Seerr) | origin=user_request, pending or auto-approved per permissions (auth area), slot = the requested version (**replaces `is4k`**), scope = seasons | Ensure Title and TitleSlot(slot) using routing: slot defaults → routing rules (Seerr override rules) → request overrides. Then Follow(season, slot) for each scoped season, or series + all seasons for a whole-series request, the preset (default `all` in scope), and `search_on_fulfil`. |
| Watchlist (Plex/Jellyfin, per user) | origin=watchlist, `is_auto`, requested_by = list owner; approval per permissions | Same as a request. **One implementation** replaces the seven copies (§2.2 #10). |
| Import list | origin=import_list, approval = the list's `approval` | Same as manual, using the list's slot templates, preset and follow_policy (for example Lidarr `EntireArtist` → Follow(artist)) |
| Follow expansion | origin=follow | §3.4 |
| Disk import / library import | origin=disk_import | Title linked to existing files; preset from root-folder defaults (O22) |

**Retraction (O14, fixing §2.2 #3).** A list sync that completes with **every enabled list fetched successfully** (the guard from Sonarr and Radarr) marks each `import_list` intent whose ref is gone as `withdrawn`. Then, for each affected title, the list's `retraction` policy applies **only if every intent on the title is withdrawn or declined** and none has origin `manual`:
- `unmonitor` sets the title's ItemSlots in the withdrawn intents' scope and slots to monitored=false;
- `remove_*` deletes the title, and its files if configured.

A manually added series that also appeared on a list is therefore never touched. Sonarr's `KeepAndTag` becomes a derived filter ("titles with no active intent"), not a tag.

User cancellation of a request withdraws the intent but does not unmonitor by default, which matches Seerr: deleting a request leaves the Arr alone.

**Exclusions.**
- `applies_to=auto_adds` blocks follows, lists and collections, but a user can still add manually, which matches the Arrs.
- `user_requests` blocks requests (Seerr blocklist, O15).
- `both` is the default when a title is "removed with exclusion".
- Keyword-based auto-blocklisting (Seerr `blocklistedTags`) creates `reason=keyword_rule` rows.
- Exclusions of *entities* are evaluated as predicates on each candidate (§3.4 `excluded_auto`). They are never materialised per item (§2.2 #4).

**Requester notifications and quotas.** "Who asked for this" is `Intent.requested_by`, a foreign key. Per-user Arr tags go away. Quota counting (TV counts seasons) counts `Intent(origin=user_request)` rows. The quota rules themselves belong to the auth area.

### 3.8 Availability is a query

```
Availability(target, scope, slot) -> {total, released, covered, status}
  items   := members of target within scope (episodes of the scoped seasons; the movie; the album…)
  total   := |items| (specials only if include_specials)
  released:= |{i ∈ items : released(i, slot)}|
  covered := |{i ∈ items : ItemSlot(i,slot).coverage = complete}|
  status  := none      if covered = 0
             complete  if covered = total                      -- Seerr "Available" (O16)
             caught_up if covered = released < total           -- new: all aired episodes present
             partial   otherwise                               -- Seerr "Partially available"
Request display = pending | declined | approved+{processing (monitored, not complete) | status above}
```

- **Transitions come from events, not polling.** Coverage changes only in import, delete and rescan transactions, and totals change only in catalog refresh transactions. Those transactions recompute availability for the approved intents that touch the changed items, and emit `AvailabilityChanged(intent, old, new)`. The notification area turns `→ complete` into "available" (and optionally `→ caught_up`; §6 Q2).
- **Seerr's DELETED** becomes a transition from complete or caught_up back to none or partial.
- **"Visible in Plex/Jellyfin"** is an optional *separate* signal owned by the media-server integration. It never overrides library truth.
- **Per slot by construction**, so every `*4k` field goes away.

### 3.9 Metadata refresh: how it touches the library

The metadata-provider area owns fetching. The library's contract is:
1. **Triggers:**
   - Title creation (fetch the entity or item: stub → complete);
   - Follow creation (expand works for performers, studios, collections and persons);
   - intent fulfilment of an unknown ref;
   - scheduled staleness, where the provider area decides cadence (for example Sonarr's `ShouldRefreshSeries` skip for ended series, Radarr's 14-day changed-feed rule, Kapowarr's issue-count diff);
   - manual refresh.
2. **One transaction per refreshed node.** A refresh upserts catalog rows by `ExternalID` and returns a `CatalogDelta{added, updated, renumbered, tombstoned, merged}`. The library applies these reactions in the same transaction, or as one durable job keyed by entity id (runtime area):
   - `added` → follow expansion (§3.4); `ItemSlot`s are created for titles whose follows admit the member.
   - `renumbered` → nothing in the library; rename is proposed by the import and file-management area.
   - `tombstoned` → keep while referenced, and raise health (O19). No auto-delete. Profilarr's text-matching deletion is gone.
   - `merged` → §3.6 merge rule.
   - `updated` release dates → availability and wanted change implicitly (they are queries).
3. **Metadata profiles filter membership, not the catalog.** Changing a profile re-runs admission for *new* members and offers "apply to existing". It never deletes catalog rows or files (fixing §2.2 #5). The provider area may use the profile to skip fetches, for example not fetching disallowed-language editions (§6 Q5).
4. **User overrides live in the library** (`Title.series_type`, `Title.overrides`, `AltTitle(source=user)`), so a refresh never overwrites them.

### 3.10 Library scan and existing files (model level)

- A scan (import and file-management area) walks each root folder and upserts `MediaFile` by path. It stats size and mtime, sets `state=missing` for vanished files, and deletes their FileLinks with reason `missing_from_disk`. That reason never auto-unmonitors (O8).
- Matching an unmapped file creates FileLinks and updates `ItemSlot.coverage`.
- The **slot** of a scanned file is taken from `RootFolder.slot`. If the root is shared by several slots, the import area assigns it by fit against each TitleSlot's profile (§6 Q9).
- Unmatched top-level folders are **library-import candidates**. Accepting one creates a Title plus `Intent(origin=disk_import)`.
- Lidarr's and Bookshelf's automatic "add unknown artists found on disk" becomes `RootFolder.add_defaults.auto_add=true`, available to every kind (§6 Q8).

### 3.11 Root folders and tags

- **RootFolder** has a kind, an optional slot, and add defaults. Free space and unmapped folders are derived at read time. A TitleSlot's root must match the kind, and match the slot when the root has one.
- **Invariant P.** If two TitleSlots of the same title resolve to the **same folder**, the naming template must contain the slot discriminator, so files cannot collide. Validation lives in the naming engine (import and file-management area).
- **Tags** attach to Titles. Follows, ImportLists and Intents carry tags to stamp on the Titles they create.
- `TitleTag.source` separates manual tags from tags added by an auto-tag rule. A rule only ever removes its own tags.
- Per-slot scoping (for example a longer delay for UHD) uses the **slot** as a selector in the release-evaluation area rather than tags.

---

## 4. Worked examples

### 4.1 TV: an anime with scene numbering and specials

*Kaiju Academy* (hypothetical). TVDB has **S01E01–E24**, with absolute numbers 1–24. Fansub and scene groups release the second cour as "Season 2", E01–E12. TVDB also has specials: **S00E01** "Recap", aired between S01E12 and S01E13 (`aired_before_season=1, aired_before_episode=13`), and **S00E02** "OVA", aired after season 1 (`aired_after_season=1`).

Catalog after the first fetch:
- `Entity(series)` with `ExternalID(tvdb, series, 400001)` plus aliases `(anilist, …)` and `(mal, …)`.
- Seasons `Entity(season, number 0)` and `Entity(season, number 1)`.
- 26 `Item(episode)` rows.
- XEM fills `EpisodeInfo.scene_season=2, scene_episode=1..12` for S01E13–24.
- `AltTitle("Kaiju Academy 2nd Season", source=xem, season_scope=2)`.

Library:
- `Title(series, series_type=anime, use_scene_numbering=true)`.
- One slot, `TV/HD`: `TitleSlot(HD, root /tv, path "Kaiju Academy")`.
- `Intent(manual, preset=all)` creates:
  - `Follow(series, HD, add=all, monitor=all)`;
  - `Follow(S1, HD, monitor=all)`;
  - `Follow(S0, HD, monitor=none)`.
- 26 `ItemSlot`s: S01E01–24 monitored, S00E01–02 not (O6).
- The user then monitors the OVA by hand.

Wanted, on a date when E01–E16 have aired and E01–E12 have files:
- **missing** = S01E13–16, plus S00E02 when `include_specials` is set (O2);
- E17–24 are monitored but unreleased, so `grabbable` from RSS but not missing;
- the Recap is never wanted.

Grabs and files:
- `[Grp] Kaiju Academy - 15 [1080p]` (absolute 15) and `Kaiju Academy S02E03 1080p` (scene) both resolve to S01E15. Resolution is the parser's job; the model supplies absolute and scene numbers.
- `Kaiju Academy - 13-14` imports as **one MediaFile with two FileLinks** (exclusive coverage). A later single-episode upgrade of E13 is subject to the SameEpisodes rules (release-evaluation area). Once imported it replaces the E13–14 file, and E14 becomes missing again, as in Sonarr.

TVDB later splits the show into S01 (1–12) and S02 (1–12). The delta is `renumbered` for 12 episodes, plus `added` for the season-2 entity:
- episode identities (TVDB episode ids) are unchanged, so FileLinks and monitored flags are untouched (Invariant M);
- `Follow(series, monitor=all)` admits the new S2 entity and creates `Follow(S2, HD, monitor=all)`. That only matters for *future* S2 episodes;
- a rename to `S02E0x` is proposed.

A Seerr-style request by another user for "Season 2" at this point is `Intent(user_request, scope=[S2], slot=HD)`. Everything is already monitored, so fulfilment is a no-op apart from recording the requester. `Availability(series, [S2], HD)` reports `partial` (E13–16 of the old numbering are now S02E01–04, some missing). It moves to `complete` when all 12 are covered, and the requester is notified then.

### 4.2 Movie with UHD and HD slots

*Example Film (2025)*. Movie slots are `HD` (profile "HD-1080p", root `/movies`) and `UHD` (profile "UHD-2160p", root `/movies-uhd`).

1. User A requests it with the default version, HD, and is auto-approved:
   - `Intent(user_request, slot=HD, approved)` → `Title(movie item)`;
   - `TitleSlot(HD, /movies, "Example Film (2025)", minimum_availability=released)`;
   - `ItemSlot(movie, HD, monitored)`.
2. User B requests `UHD`. B lacks auto-approve for UHD (auth area), so the intent stays **pending**. On approval, in one transaction: `TitleSlot(UHD, /movies-uhd, …, minimum_availability=released)` and `ItemSlot(movie, UHD, monitored)`. No second instance and no `status4k`.
3. The digital release date passes. `released` is true for both slots and both are **missing**. Each slot is searched and graded against its own profile (acquisition and release-evaluation areas).
4. An HD WEB-DL is imported: `MediaFile(slot=HD)`, and `ItemSlot(HD).coverage=complete`. `Availability(movie, HD)=complete` notifies A. B's UHD intent still shows *processing*.
5. A UHD remux is imported into the UHD slot, and B is notified. Upgrading HD (say to a better encode) only ever replaces the HD file. Exclusive coverage is per (item, **slot**).
6. If the admin later points both slots at `/movies`, Invariant P requires `{Slot}` or quality in the file name, for example `Example Film (2025) - 2160p.mkv` next to `… - 1080p.mkv`. Media servers read that layout as multiple versions (naming area).
7. The user deletes the HD file with auto-unmonitor on: `ItemSlot(HD).monitored=false`. UHD is unaffected.

### 4.3 Music: an album with a multi-disc release

Artist *The Examples* has `Follow(artist, Lossless, add=profile("Studio albums"), monitor=from_date(2026-09-28))`. Album *Twin Peaks of Sound* (release group) has three releases (Editions):
- **R1**: 1×CD, 12 tracks;
- **R2**: 2×CD deluxe, disc 1 = 12 tracks, disc 2 = 10 bonus tracks;
- **R3**: Digital, 12 tracks.

The album's release date is after the follow date, so it is admitted and monitored. Slots:
- **Lossless**: the user picks the deluxe edition, so `target_edition=R2, edition_locked=true`;
- **Lossy**: `target_edition=null`, unlocked (Lidarr `AnyReleaseOk`).

Imports:
- **Lossless, 22 FLAC files.** Identification (import area) matches R2. That gives 22 `FileLink(file, album, edition=R2, track)`, and coverage = complete once 22 of 22 tracks are linked.
- **Only disc 1 arrives.** 12 links give `coverage=partial`, which is still **missing** (O4). The album-atomic import rule may instead reject the partial set (import area); either way the model is consistent.
- **FLAC + CUE images, one file per disc.** File 1 links 12 tracks and file 2 links 10. Coverage is the same, and the file:track relation is 1:N (O9).
- **Lossy, 12 MP3s identified as R3.** Because the slot is unlocked, import sets `target_edition=R3` (O10) and coverage becomes complete. The lossless slot keeps R2. **The target edition is per slot**, which Lidarr's single `AlbumRelease.Monitored` could not express.
- **MusicBrainz merges R2 into R2′.** R2's MBID becomes `ExternalID(status=alias)` on R2′. Tracks, FileLinks and `target_edition` move with the merge, and tagged files carrying the old MBID still resolve.

### 4.4 Comics: a TPB covering issues #1–6

Two ComicVine volumes:
- **V1** *Star Rangers (2021)*: issues #½, #1–#12, with `sort_key` 0.5, 1…12;
- **V2** *Star Rangers Vol. 1: First Light*: `special_version=tpb`, holding one issue item.

Title V1 has slot `Digital` and `Follow(V1, add=all, monitor=all)`.

1. **A pack of single issues.** The release `Star Rangers 001-006 (2021) (Digital)` is imported into V1 as **one MediaFile with FileLinks to issues #1–#6** (overlapping mode). All six ItemSlots become covered. A later separate `Star Rangers 003 (2021)` may also be imported: #3 then has two covering files, which is allowed in comics. The release-evaluation area uses the best file for cutoff.
2. **The actual TPB.** `Star Rangers Vol. 1 TPB` matches **V2**, not V1, because a TPB file never matches a normal volume (O21, `K/implementations/matching.py:154-221`). If the user adds V2 as a title, the file covers V2's single item.
3. **Optional (new): collected editions satisfy singles.** When metadata (Metron `reprints`) or the user asserts `ItemRelation(V2#1 collects V1#1..#6)`, and V1's TitleSlot enables `accept_collected`, the wanted query treats V1#1–#6 as covered if V2#1 is covered in the same slot:
   ```
   covered'(i,s) := coverage(i,s)=complete ∨ ∃ c: ItemRelation(c collects i) ∧ coverage(c,s)=complete
   ```
   This is **derived, not a FileLink**, so the TPB file stays owned by V2's folder and naming. It is **off by default**, which keeps Kapowarr parity (§6 Q1).

---

## 5. Simplifications versus the references

| Reference behaviour | New design | Why it's safe |
|---|---|---|
| Sonarr stores series, season and episode monitored flags and re-syncs them heuristically (`S/Tv/EpisodeMonitoredService.cs:60-140`) | `TitleSlot.monitored` (pause) + `ItemSlot.monitored`; the season "flag" is `Follow(season)` | The decision engine reads only series ∧ episode (O1). Season flags only governed new episodes, which is exactly what a Follow does. |
| Presets stored in `AddOptions` and consumed on first refresh | `Intent.preset`, applied once at fulfilment | Same one-shot semantics (O5), made explicit and retry-safe |
| `Movie/MovieMetadata`, `ArtistMetadata`, `AuthorMetadata` splits; v3 auto-creates unmonitored Studio/Performer rows | One catalog layer for every kind, and a library layer | Same capability (non-library credits, collection members, discover) without pass-through properties |
| `CleanLibrary` checks *all* titles against list items (`R/ImportLists/ImportListSyncService.cs:183-228`) | Retraction only for titles whose every intent is withdrawn, and never ones with a manual intent | Strictly less destructive. It keeps the "all lists fetched OK" guard. |
| Sonarr `KeepAndTag` tag | Derived filter: "no active intent" | The same information, without a tag that can go stale |
| Seerr `Media.status/status4k`, 4 scanners, `availabilitySync`, download poller | `Availability()` query + events at coverage and catalog changes | The library is the source of truth; nothing can drift. Plex/Jellyfin presence stays a separate optional signal. |
| 4K Arr instances, `is4k` servers and permissions, instance-to-instance lists, ebook/audiobook Bookshelf pairs | Slots (D4): TitleSlot and ItemSlot per slot | Each slot keeps its own profile, root, monitoring and coverage, so every workaround behaviour is still expressible |
| Seerr FAILED requests and retry | Durable in-process fulfilment job; no FAILED state | No network hop between request and library |
| Per-user Arr tags for "requested by" | `Intent.requested_by` FK | Exact, and survives tag renames |
| Seven Plex-watchlist implementations | One watchlist import list producing per-user `Intent(origin=watchlist)` | Keeps both Arr auto-add (approval=auto) and Seerr auto-request (approval per permission) |
| v3 materialises gate outcomes as exclusion rows with reasons | Gates evaluated as predicates at admission; only user exclusions are stored | Un-excluding a studio now takes effect, and there is no stale derived state |
| Per-app exclusion tables keyed by TVDB int, TMDb int or v3 `ForeignId+Type`; Seerr blocklist | One `Exclusion` keyed by `(provider, namespace, value)` or a catalog node, with `applies_to` | Covers both "no auto-add" and "no requests" semantics |
| Radarr collections, v3 works sync, `MonitorNewItems` ×4 apps, Kapowarr `monitor_new_issues`, entity-type import lists | `Follow(entity, slot)` with add, monitor and gate policies | Every reference behaviour is a policy setting (§3.4 table) |
| Lidarr and Bookshelf metadata profiles delete albums or books on refresh | The profile is a membership predicate; the catalog is untouched | A view change can no longer destroy data |
| `Album.Monitored` + exactly one `AlbumRelease.Monitored` + `AnyReleaseOk`; the same for Edition/`AnyEditionOk` | `ItemSlot.target_edition` + `edition_locked` | Same semantics per slot. The one-per-album constraint becomes one per (album, slot), which D4 needs anyway. |
| Provider-typed id columns (`TvdbId`, `ForeignAuthorId`, `StashId`/`TpdbId`/…; `Old*Ids`) | `ExternalID` map with `status=alias` | D3d; switching providers or handling merges needs no schema change |
| Scene mappings, alt titles, translations, studio aliases and `SearchTitle` in separate tables | `AltTitle` with source, usage and season scope | One lookup path for search and parse |
| Unmapped files as `AlbumId=0`, `ImportFiles` or root-folder unmapped folders | A `MediaFile` without FileLinks | One scan model for every kind |
| Sonarr `UnmonitorReaddedEpisodes` mutates flags heuristically | The preset governs the first expansion; later late members emit an event and suppress auto-search | Same protection against surprise mass downloads, with no silent flag changes |

---

## 6. Issues with decisions, and open questions

### 6.1 Issues with decisions

- **I1: D2 "absolute paths".** D2 lists absolute paths as a model requirement, because Lidarr and Bookshelf need files that sit outside a title folder. That need is fully met by storing **`root_folder_id` + a path relative to the root**. All library files live under a root folder, and files still in download folders are the download area's concern. Root-relative storage makes moving or remapping a root one row update rather than rewriting every file row. It is also safe under D14 (paths are identical on every node either way). **Recommendation:** read D2 as "not relative to the title folder" and store root-relative paths. I have kept `MediaFile.path unique` in the sketch until this is confirmed.
- **I2: D4 does not say where slots are defined.** D4 says "each item can have several named slots, each with its own quality profile and root folder". Root folders can't sensibly differ per *episode*, so this design puts slot *configuration* at the title level (TitleSlot) and *monitoring and coverage* per item (ItemSlot). Slot *definitions* are global per media kind, because requests, permissions, lists, availability and naming tokens need stable slot identities across titles. This is an interpretation, not a deviation, but it should be confirmed.
- **I3: D4 with overlapping profiles.** D4 has each slot "grabbed … independently". If two slots' profiles both accept a release (for example a 1080p WEB-DL acceptable to "HD" and to an "Any" slot), the same release could be grabbed twice. The model is fine with it, since each slot's coverage is separate. The release-evaluation and download-lifecycle areas need a rule: either reject overlapping slot profiles at configuration time, or let one download import into several slots (hardlinks). Flagged for those areas.
- **I4: D10 product codes.** D10 restores product codes "as indexed alternate ids". They are **non-unique** (verification/decision-premises.md §D10), so they cannot sit in the unique `ExternalID` map. They live in `SceneInfo.code` with a non-unique index. Consistent with D10, but worth recording.
- **I5: D3a TV ordering.** TVDB is canonical for aired order, which the model follows. TVDB also offers DVD, absolute and alternate orderings, which some anime and some shows' releases follow. Neither D3 nor the references (Sonarr uses the default order only) support choosing one per title. Not a flaw in D3, but a gap to decide (Q7).

### 6.2 Open questions for the user

1. **Q1, collected editions (comics):** should a TPB or omnibus be able to satisfy the singles it collects (§4.4 step 3)? The proposal is off by default for Kapowarr parity, as a per-title-slot opt-in.
2. **Q2, "available" for airing seasons:** Seerr only calls a season Available when *every* episode, aired or not, is present (O16). Keep that as the "available" notification trigger, or also notify at `caught_up` (all aired episodes present)?
3. **Q3, list retraction options:** keep Radarr's `remove_delete_files` (now scoped to list-only titles), or cap retraction at `unmonitor`/`remove_keep_files`?
4. **Q4, list approval:** allow `approval=manual` on import lists, so list items arrive as pending requests? This is new, but falls out of the Intent model.
5. **Q5, metadata profiles:** a pure membership filter (the catalog stores everything fetched), or also a fetch filter (fewer provider calls; relevant for Hardcover and MusicBrainz quotas)? This is a joint decision with the metadata-provider area.
6. **Q6, Lidarr "New" semantics:** the proposal replaces "newer than the newest existing album" with a fixed `from_date`, defaulting to the follow's creation time. They differ only when metadata back-fills an album dated between the newest known album and now. Acceptable?
7. **Q7, TV orderings:** support per-title alternate TVDB orderings (DVD/absolute) in v1, or keep aired order only, as Sonarr does?
8. **Q8, disk auto-add:** offer Lidarr's "auto-add unknown titles found in a root folder" for every kind, or only music and books?
9. **Q9, shared roots across slots:** when several slots share one root folder, should a scanned file's slot be inferred by profile fit (proposal), or should roots be required to belong to one slot?
10. **Q10, specials and requests:** Seerr's `enableSpecialEpisodes` is global. Should `include_specials` be per request and per title (proposal), or global?
