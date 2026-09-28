# Design: metadata providers and subtitles

Status: draft for review (2026-09-28). Scope: **A.** metadata providers (D3), all media kinds; **B.** subtitles (phase 2, D2).

Paths without a prefix are relative to `submodules/`. "§" references point to the section in this document unless another doc is named. Binding decisions are in [../decisions.md](../decisions.md); verified facts about external services are in [../verification/metadata-services.md](../verification/metadata-services.md) (cited as *MS C1…E17, F1…F9*).

**Boundaries.** This document defines metadata *documents* and how they are fetched, cached, refreshed and mapped to ids, plus the subtitle domain (wanted computation, search, scoring, plug-ins, post-processing). It does **not** define:
- the library tables that hold series/episodes/albums and so on (the library-model design); this document hands them canonical documents and change sets through a port;
- where subtitle files are written or renamed (the import design); this document asks import to place a file and receives the resulting row;
- the job queue, leader election, events and plug-in runtime plumbing (the runtime/HA design); this document states what it needs from them;
- auth, API shape and deployment.

---

## 1. Outcomes and invariants to preserve

D19: the reference apps define outcomes. These are the outcomes and correctness rules this design keeps. Everything in §2's "accidental" lists is free to change.

### 1.1 Metadata

| # | Invariant / outcome | Reference evidence |
|---|---|---|
| M1 | **TV episode identity and ordering follow TVDB's default order.** Scene numbering (XEM), anime absolute numbering and scene mappings are all keyed to TVDB ids and TVDB season/episode numbers. TMDb never decides TV structure. | `sonarr/src/NzbDrone.Core/Tv/Episode.cs` (Scene*/Absolute fields), `DataAugmentation/Xem/XemProxy.cs:35` (`origin=tvdb`), `DataAugmentation/Scene/SceneMapping.cs:15` (`TvdbId`); MS F2; D3a |
| M2 | **Scene numbering from XEM, with extrapolation for episodes XEM has not mapped yet, flagged "unverified".** Series without mappings fall back to TVDB numbering. | `sonarr/src/NzbDrone.Core/DataAugmentation/Xem/XemService.cs:34-89` (apply), `:100-200` (extrapolate, `UnverifiedSceneNumbering`) |
| M3 | **Scene-name mappings** (alternate titles, per-season scene titles, parse/search terms, `FilterRegex`, `SearchMode`) and the **daily-series list** exist and drive parsing and searching. | `sonarr/src/NzbDrone.Core/DataAugmentation/Scene/SceneMapping.cs:8-28`, `DailySeries/DailySeriesDataProxy.cs:24,33`; refreshed every 3 h (`Jobs/TaskManager.cs:90-91`) |
| M4 | **Upstream deletion never deletes the user's files.** A 404 marks the item deleted-upstream and raises a health warning. | Sonarr `Tv/RefreshSeriesService.cs:69-80`; Radarr `Movies/RefreshMovieService.cs:87-91`; Lidarr `RemovedArtistCheck` (lidarr.md §3.1) |
| M5 | **Upstream id replacement and merges are followed without losing files** (TVDB duplicate replaced; MusicBrainz MBID merges via `Old*Ids`). | Sonarr `Tv/RefreshSeriesService.cs:83-86`; Lidarr `Music/Services/RefreshEntityServiceBase.cs:150-163,226-267`, `RefreshAlbumService.cs:82`; Eros `Performer.MergedIntoId` (whisparr-eros.md §2) |
| M6 | **Refresh is adaptive, not uniform:** recently aired/released or continuing items refresh often, ended/old ones rarely, with a floor and a ceiling. | Sonarr `Tv/ShouldRefreshSeries.cs:28-70` (30 d ceiling, 6 h floor, TBA titles); Radarr `Movies/ShouldRefreshMovie.cs:30-50` (180 d ceiling, 12 h floor); Lidarr `Music/Utilities/ShouldRefreshArtist.cs`, `ShouldRefreshAlbum.cs` (30 d / 60 d); Bookshelf `Books/Utilities/ShouldRefreshAuthor.cs` |
| M7 | **Delta refresh where the source offers it.** | Radarr `Movies/RefreshMovieService.cs:263-265` and Lidarr `Music/Services/RefreshArtistService.cs:417-419` (`changed?since` if last run < 14 days ago); Eros `RefreshMovieService.cs:405-420` |
| M8 | **Music: an album is a MusicBrainz release group; one release is selected; tracks keyed by recording id** (AcoustID returns recordings). Metadata profiles filter which release groups exist at all. | lidarr.md §2 (`Album`, `AlbumRelease`, `Track.ForeignRecordingId`), §3.5 |
| M9 | **Per-provider external-id map, never one "foreign id".** | D3d; Bookshelf's Goodreads-vs-Hardcover split (bookshelf.md §2, `ForeignAuthorId` meaning depends on image); Eros multi-id model (whisparr-eros.md §2) |
| M10 | **Source terms respected:** TMDb data cached ≤ 6 months, TMDb logo + disclaimer; attribution links for TVDB, TVMaze, ComicVine; MusicBrainz ≈1 req/s with a meaningful User-Agent; ComicVine 200 req/resource/hour; Hardcover 60/min and 5,000/day via OAuth; Metron 20/min, 5,000/day; AcoustID 3 req/s. | MS E2-E13, F3-F7 |
| M11 | **Requests discovery** (trending, popular, upcoming, by genre/studio/network/keyword/watch-provider, recommendations, person, collection) with "already available / requested" annotation. | seerr.md §3 ("Discovery (TMDb)"), `seerr/server/routes/discover.ts` |
| M12 | **Search by id prefix** (`tvdb:`, `tmdb:`, `imdb:`, `mbid:`, `stash:`, …) and by URL. | Sonarr `MetadataSource/SkyHook/SkyHookProxy.cs`; Radarr and Eros SkyHook (radarr.md §4, whisparr-eros.md §5); `seerr/server/lib/search.ts` |

### 1.2 Subtitles

| # | Invariant / outcome | Reference evidence |
|---|---|---|
| S1 | **Scoring weights**: episode max 360 (hash 359, series 160, year 90, season 30, episode 30, source 25, release_group 20, audio_codec/resolution/video_codec/hearing_impaired/streaming_service 1); movie max **180** (hash 179, title 60, year 40, source 30, edition 30, release_group 15, the five 1-pointers). Hash is always max − 1. | `bazarr/custom_libs/subliminal_patch/score.py:31-64` |
| S2 | **Hash trust rule**: a hash match counts only if {series, season, episode, source} (episode) or {video_codec, source} (movie) also match; never for TV specials; when trusted, all other matches are discarded. | `score.py:101-130` |
| S3 | **Match equivalences**: `imdb_id` ⇒ series+year+season+episode (episode) or title+year (movie); `tvdb_id` ⇒ series+year+season+episode+title; `series_imdb_id`/`series_tvdb_id` ⇒ series+year; episode `title` ⇒ episode; specials with title+series+year ⇒ season+episode. | `score.py:138-200` (`_episode_checks`) |
| S4 | **Hearing-impaired semantics**: profile item `hi` ∈ {required ("True"), normal ("False"), excluded}. Required → prefer HI (+1) and **skip** non-HI candidates from providers that can verify HI; excluded → prefer non-HI and skip HI; normal → non-HI gets the +1 but HI is accepted. | `bazarr/bazarr/subtitles/download.py:76-86`; `subliminal_patch/core.py:552,604-609`; `score.py:148-155`; `bazarr/bazarr/constants.py:17` |
| S5 | **Forced semantics**: forced = "foreign parts only", a **separate wanted unit** from the full subtitle of the same language; forced and HI are mutually exclusive (a forced item is never HI). Provider queries are "only forced / also forced / not forced" depending on the wanted set. | `download.py:48-50,191-201`; `subliminal_patch/providers/opensubtitlescom.py:300-301,392-398` |
| S6 | **Selection**: sort by (score, score-without-hash) desc; stop below `minimum_score` (90 % episodes, 70 % movies); for episodes require season+episode and series (or imdb_id) unless the hash is unverifiable; fall back to the next candidate when a download fails. | `subliminal_patch/core.py:580-625`; bazarr.md §3 |
| S7 | **Upgrade rule**: re-search subtitles downloaded in the last `days_to_upgrade_subs` (7) with `score < score_out_of − 3`; replacement must beat the old score; skip if the language or HI requirement is no longer wanted. Scores are stored with their `score_out_of`. | `bazarr/bazarr/subtitles/upgrade.py:260-261,324,406`; bazarr.md §3 |
| S8 | **Adaptive searching**: after 3 weeks of failures for a language, search only weekly. | `bazarr/bazarr/subtitles/adaptive_searching.py` |
| S9 | **Profiles**: ordered items with language, forced, hi, audio exclusion rules; cutoff; must-contain/must-not-contain; original-format; `language_equals` (e.g. `pt`↔`pt-BR`); custom codes (pt-BR, zh-Hant, es-419). | bazarr.md §2 (`table_languages_profiles`), §3; `bazarr/bazarr/languages/custom_lang.py` |
| S10 | **Embedded tracks count as present** (per profile), with PGS/VobSub/ASS ignore switches and a default language for `und` tracks; forced/HI read from stream dispositions. | `bazarr/bazarr/utilities/video_analyzer.py:75-78,198-203`; bazarr.md §3 "Embedded subtitles" |
| S11 | **Filename convention** `name.<lang>[.hi|.cc|.sdh][.forced].<ext>`, single-language mode, HI extension choice. | `subliminal_patch/core.py:1162-1190`; bazarr.md §11 |
| S12 | **Post-download order**: write → mods → sync (score-gated: < 90 % episodes / < 70 % movies; quality checker rejects bad results) → history → notify → media-server refresh. | bazarr.md §3, §11 |
| S13 | **Provider throttling** by error class with per-provider overrides, including "until local midnight" resets; blocklist by (provider, subtitle id). | `bazarr/bazarr/app/get_providers.py:58-141`; `bazarr/bazarr/sonarr/blacklist.py` |

---

## 2. How the references do it, and what is accidental

### 2.1 Metadata today

| App | Source path | Refresh | Notable |
|---|---|---|---|
| Sonarr | SkyHook proxy over TVDB (`NzbDrone.Common/Cloud/SonarrCloudRequestBuilder.cs:18`); TheXEM direct (`DataAugmentation/Xem/XemProxy.cs:21`); scene mappings and daily list from `services.sonarr.tv` (MS C2) | `RefreshSeriesCommand` every 12 h over stale series (`Jobs/TaskManager.cs:102-103`, `Tv/ShouldRefreshSeries.cs`); full refetch per series | SkyHook adds TVMaze, TMDb, IMDb, MAL, AniList ids (sonarr.md §5) |
| Radarr | `api.radarr.video` (TMDb proxy) with `movie/changed?since=` (MS C3) | delta if last run < 14 d, else stale-based (`Movies/RefreshMovieService.cs:263-265`) | `MovieMetadata` split from `Movie` (radarr.md §2); hard-coded TMDb bearer for lists (MS C3) |
| Lidarr | `api.lidarr.audio` only (MS C6, C7); no MusicBrainz fallback | 24 h job; `recent/artist?since=` if < 14 d (`RefreshArtistService.cs:417`) | Server returns whole artist with all release groups/releases/tracks, filtered client-side by metadata profile (lidarr.md §3.1); outage from 2025-05-21 (MS E15) |
| Bookshelf | rreading-glasses hosts `api.bookinfo.pro` / `hardcover.bookinfo.pro` (MS C12, E17) | 24 h; caching deliberately bypassed, polls up to 60× on 202 (bookshelf.md §3) | Foreign ids mean Goodreads *or* Hardcover depending on image |
| Whisparr v2 / v3 | `api.whisparr.com` (closed); v3 also StashDB and TMDb direct for lists (MS C9, C10) | v3 `*/changed?since=` | v3 multi-id per entity; performer merges |
| Kapowarr | ComicVine direct, user key (MS C19) | hourly job over volumes > 24 h old; issues only if count changed or 30 d (`kapowarr/backend/implementations/volumes.py:1356-1440`); 10-request batches with 10 s sleeps (`comicvine.py:421`) | bulk `filter=id:a|b|…` 100 ids per call |
| Seerr | TMDb direct (hard-coded key, 50 rps, `seerr/server/api/themoviedb/index.ts:188-194`); optional TVDB (`api/metadata.ts`, `api/tvdb/index.ts:113-114`); Anime-Lists XML (`api/animelist.ts`); IMDb via Radarr proxy, RT via Algolia (MS C4, C5) | in-memory LRU caches, 5 min default TTL (`seerr/server/lib/cache.ts:15-50`) | its own `Media` table and four availability scanners (seerr.md §3) |
| Bazarr | none of its own; mirrors Sonarr/Radarr; AniDB HTTP API and Fribb/Anime-Lists for anime ids (`bazarr/bazarr/subtitles/refiners/anidb.py:63`, `anilist.py:27`) | — | — |

### 2.2 Subtitles today (Bazarr)

- A **mirror** of Sonarr/Radarr series, episodes and movies, kept current by an hourly full sync, SignalR (three client classes) and webhooks, with path mappings between the Arr's view and Bazarr's (bazarr.md §9).
- A **disk indexer** separate from the Arrs' own `SubtitleFiles`/`ExtraFiles`, so the same files are indexed twice (bazarr.md §9.6, §11).
- A **second media probe** (knowit/ffprobe, pickled into the DB, `bazarr/bazarr/utilities/video_analyzer.py:278,344`).
- ~65 provider modules (22k LOC) in a forked subliminal; each provider has its own guessit-based `get_matches()` (bazarr.md §4.1, §11).
- Throttle state in a stringified dict file (`get_providers.py:575`); an in-memory job queue; APScheduler.
- Sync via ffsubsync only (`bazarr/bazarr/subtitles/tools/subsyncer.py`); mods from subzero (`custom_libs/subzero/modification/mods/*`).
- After each change, `POST command RescanSeries/RescanMovie` so the Arr notices the file (`bazarr/bazarr/sonarr/notify.py:13`).

### 2.3 Accidental complexity (drop)

| Item | Why it exists | Evidence |
|---|---|---|
| Hosted proxies (SkyHook, RadarrAPI, LidarrAPI, Whisparr API, bookinfo.pro) | Servarr aggregates and caches centrally | MS C1-C12; D3c forbids hosted infra in v1 |
| Full refetch of the whole artist/series tree on every refresh | Proxy returns whole trees; diffing done client-side | Lidarr `RefreshEntityServiceBase.cs`; lidarr.md §3.1 |
| Seerr's own TMDb client, `Media` table, four availability scanners, availability sync, 4K duplicate columns | Seerr is a separate app that cannot see the library | seerr.md §3; comparison.md §2.3 "Availability" |
| Bazarr's mirrored DB, three sync paths, path mappings, `Rescan*` round trips, second subtitle index, second probe | Bazarr is a separate app | bazarr.md §9.6, §11 |
| Per-provider guessit matching | Python providers each parse release names | bazarr.md §11 "Porting…"; D1c says reuse the main parser |
| Throttle state in a file; in-memory queue | Single-process Python | `get_providers.py:575`; `bazarr/app/jobs_queue.py` |
| Bookshelf's third "cache" database that it then bypasses | Fork drift | bookshelf.md §2, §8 |
| Seerr's IMDb-via-Radarr and RT-via-Algolia "borrowed" ratings | No official source | MS C4, C5, F9 |
| Hard-coded keys (TMDb bearer, TVDB, OpenSubtitles, AcoustID, GA4) | Convenience | MS F9; `bazarr/bazarr/app/get_providers.py:252` |
| Custom score overrides (already removed upstream), dead providers, legacy Sonarr v2/Radarr v0.2 paths | History | `bazarr/bazarr/app/config.py:630-634`; bazarr.md §10 |

---

## 3. Proposed design

### 3.A Metadata providers

#### A1. Shape

```
             ┌──────────────── library / requests / import (consumers) ────────────────┐
             │  LibraryPort.Apply(ChangeSet)      Discovery API      Identify(candidates) │
             └───────────────▲──────────────────────▲────────────────────▲──────────────┘
                             │                      │                    │
┌────────────────────────────┴──────────────────────┴────────────────────┴──────────────┐
│ metadata service                                                                      │
│  Assemblers (one per kind): TV · Movie · Music · Book · Adult · Comic                 │
│     compose Sources into canonical Docs; own the "structure authority" rule           │
│  Planner (singleton): change-feed polling, due-refresh selection, budget forecasting  │
│  Refresh workers (any node, durable jobs): fetch → normalise → diff → apply           │
│  Doc cache (DB): per-source normalised docs with hard expiry from terms               │
│  External-id map (DB) · mapping datasets (scene, XEM, anime ids)                      │
└───────────────────────────────────┬───────────────────────────────────────────────────┘
                                    │ SourceClient (per source): budgets, cooldown, UA, auth, attribution
        TVDB · TMDb · TheXEM · MusicBrainz · CAA · AcoustID · Hardcover · OpenLibrary ·
        Audnexus/AudiobookDB · StashDB · ThePornDB · ComicVine · Metron
```

Metadata sources are **native Go** in the core, not WASM plug-ins: they are few, first-party, and on the critical path of the data model. D15's boundary is for long-tail extension points (subtitle providers, indexers, DDL hosts, notifications). A hosted cache can later be one more `Source` implementation (D3c).

#### A2. Sources and their roles

| Kind | Structure authority (ids, ordering, children) | Enrichment | Discovery / search | Change feed |
|---|---|---|---|---|
| TV | **TVDB v4** (series, seasons, episodes in default order, absolute order, aliases, translations) | TMDb (images, ratings, cross-ids, certifications); TheXEM (scene numbering, scene names); scene-mapping dataset; anime-id dataset | TVDB search; TMDb discover for requests (mapped to TVDB, §A15) | TVDB `/updates?since=`; TMDb `/tv/changes` for enrichment |
| Movie | **TMDb** | TMDb translations, collections, credits; IMDb id | TMDb | TMDb `/movie/changes` |
| Music | **MusicBrainz** (artist, release group, release, medium, track, recording) | Cover Art Archive (covers); AcoustID (fingerprint → recording ids, import only) | MusicBrainz search | none in WS/2 (age-based; replication with a mirror) |
| Book | **Hardcover** (OAuth device grant) | OpenLibrary (ISBN lookup, keyless fallback); Audnexus/AudiobookDB (audiobook: narrators, ASIN, chapters) | Hardcover, then OpenLibrary | Hardcover `updated_at` filter (to verify) |
| Adult | **StashDB** (scenes, performers, studios; per-user key) | ThePornDB (per-user token), TMDb (full movies) | StashDB / TPDB | StashDB `updated_at` sort/filter (to verify) |
| Comic | **ComicVine** (per-user key) | Metron (optional secondary, per-user account) | ComicVine | ComicVine `date_last_updated` filter (to verify) |

"Structure authority" is a hard rule of the Assembler: only the authority may create, delete or renumber children. Enrichment sources may only fill fields they are allowed to fill (§A4).

#### A3. Go interfaces

```go
package metadata

type SourceID string // "tvdb", "tmdb", "musicbrainz", "caa", "acoustid", "hardcover", "openlibrary",
                     // "audnexus", "stashdb", "tpdb", "comicvine", "metron", "thexem"

type Ref struct {
    Source SourceID
    Kind   EntityKind // Series, Episode, Movie, Collection, Person, Artist, ReleaseGroup, Release,
                      // Recording, Work, Edition, Author, Scene, Performer, Studio, Volume, Issue
    ID     string
}

// Every source describes itself; the SourceClient enforces what it declares.
type Source interface {
    ID() SourceID
    Info() SourceInfo
}

type SourceInfo struct {
    Attribution Attribution     // §A11
    Rate        RateSpec        // steady rate, burst, windows (per hour/day/resource), Scope: Global|PerNode
    Credentials []CredentialSpec // §A10
    Cache       CachePolicy     // HardMaxAge (terms), DefaultTTL
    Redirects   bool            // lookup of a merged id returns the target (MusicBrainz, StashDB)
}

// Capabilities; a source implements the ones it supports.
type Searcher     interface{ Search(ctx context.Context, q Query) ([]Candidate, error) }
type Fetcher      interface{ Fetch(ctx context.Context, ref Ref, opt FetchOpts) (Doc, error) }
type BatchFetcher interface{ FetchMany(ctx context.Context, refs []Ref, opt FetchOpts) (map[Ref]Doc, error) } // ComicVine, GraphQL sources
type ChangeFeed   interface{ Changes(ctx context.Context, kind EntityKind, since time.Time) (ChangeBatch, error) }
type IDResolver   interface{ Resolve(ctx context.Context, known ExternalIDs, want SourceID) (ExternalIDs, error) }
type Discoverer   interface{ Discover(ctx context.Context, q DiscoverQuery) (DiscoverPage, error) }       // TMDb
type Fingerprinter interface{ Lookup(ctx context.Context, fp Fingerprint) ([]RecordingMatch, error) }     // AcoustID

// Typed errors; the refresh engine branches on them.
var ErrNotFound = errors.New("not found upstream")       // → deleted-upstream handling (§A6)
type RedirectedError struct{ To Ref }                    // → merge handling (§A6)
type ThrottledError struct{ RetryAfter time.Duration }   // → source cooldown (§A9)
type QuotaError struct{ ResetAt time.Time }              // → budget exhausted until reset
type AuthError struct{ Credential CredentialRef }        // → health issue, source degraded

// One per kind; composes sources into canonical documents.
type Assembler[D any] interface {
    Assemble(ctx context.Context, root Ref, opt AssembleOpts) (D, Provenance, error)
    Search(ctx context.Context, q Query) ([]Candidate, error) // merged, de-duplicated by external ids
}
```

Canonical documents (`SeriesDoc`, `MovieDoc`, `ArtistDoc`, `ReleaseGroupDoc`, `ReleaseDoc`, `AuthorDoc`, `WorkDoc`, `EditionDoc`, `SceneDoc`, `PerformerDoc`, `StudioDoc`, `VolumeDoc`, `IssueDoc`) are plain structs with an `ExternalIDs` set, localised titles and aliases, status, dates, images (URL + source + dimensions), ratings (per source) and typed children. `Provenance` records which source supplied each field group, for attribution and conflict debugging.

The library consumes documents through one port:

```go
type LibraryPort interface {
    // Apply is transactional: the library diffs children by external id,
    // applies merges/deletions per §A6, and emits domain events.
    Apply(ctx context.Context, cs ChangeSet) error
}
type ChangeSet struct {
    Root      Ref
    Doc       any              // canonical doc
    Merged    []Merge          // old Ref → new Ref (children included)
    Removed   []Ref            // confirmed not-found upstream
    FetchedAt time.Time
}
```

#### A4. Composition rules (per kind)

- **TV.** TVDB supplies series identity, seasons, episodes (default order), absolute numbers (TVDB "absolute" season type), air dates (UTC, from TVDB air date + network time zone), specials placement (`airsBefore*`), aliases and translations. TMDb supplies posters/fanart/logos, TMDb rating, certification and `external_ids` (IMDb, TVDB cross-check). XEM supplies per-episode scene season/episode/absolute and scene names. The scene-mapping dataset supplies parse/search terms. The anime-id dataset supplies AniDB/MAL/AniList ids. Image preference is a setting (default: TMDb, then TVDB).
- **Movie.** TMDb supplies everything; the collection is a separate `Collection` doc.
- **Music.** MusicBrainz supplies everything structural; CAA supplies cover URLs by release (and release group front cover). Artist docs contain release-group *summaries* only; releases and tracks are fetched per release group on demand (§A13).
- **Book.** Hardcover work/edition/author; OpenLibrary fills ISBN-only lookups and missing editions; Audnexus/AudiobookDB fills audiobook fields for editions with an ASIN.
- **Adult (D10).** StashDB scene/performer/studio; ThePornDB and TMDb are linked by the cross-ids StashDB carries (`urls`/remote ids) or by the user; full movies from TMDb or TPDB.
- **Comic.** ComicVine volume/issue; Metron may fill missing issues when configured and is linked by the CV id Metron stores.

A field-group precedence table per kind lives in code with a user override for image source and title language only. There is no general "merge any field from any source" engine; it would make structure authority unenforceable.

#### A5. Data model (metadata-owned tables)

```
metadata_doc            -- persistent cache of normalised per-source documents
  source, kind, source_id           PK
  doc            jsonb              normalised (not raw HTTP)
  doc_hash       bytea              for change detection
  fetched_at, hard_expires_at       hard_expires_at = fetched_at + Info().Cache.HardMaxAge (TMDb 180 d)
  etag, last_modified               conditional requests where supported
  status         ok | not_found | redirected
  redirect_to    (source, kind, id) nullable

external_id             -- one row per id an entity is known by
  entity_type, entity_id            FK into library entity (series, episode, movie, album, …)
  source, id_kind, value            UNIQUE(source, id_kind, value)
  state          active | merged | deleted
  merged_into    value nullable     (the id it was merged into)
  confirmed_at

metadata_refresh        -- one row per refresh root (series, movie, artist, release group, author, volume, performer, studio)
  root_ref, entity_type, entity_id
  policy         enum               (§A8)
  next_due_at, priority, last_success_at, last_attempt_at, failures
  terms_deadline                    min over hard_expires_at of docs this root depends on

source_state            -- per source (and per credential for per-key limits)
  source, credential_id
  tokens, tokens_at                 global token bucket (Scope=Global)
  window_used jsonb                 counters per window (hour/day/per-resource)
  cooldown_until, cooldown_reason   shared back-off after 429/503
  change_cursor                     last change-feed timestamp polled
  health, degraded_reason

credential              -- D3b/D16; stored in DB
  id, source, origin (project | user), kind (api_key | pin | oauth | basic)
  secret, refresh_token, expires_at, status, last_validated_at

mapping_dataset         -- runtime-downloaded datasets (scene mappings, anime ids)
  name, version (git commit), sha256, fetched_at
scene_mapping(tvdb_id, title, parse_term, search_term, season, scene_season, filter_regex, search_mode, origin, dataset)
xem_episode(tvdb_series_id, tvdb_season, tvdb_episode, tvdb_absolute, scene_season, scene_episode, scene_absolute, fetched_at)
anime_id_map(tvdb_id, tvdb_season, episode_offset, anidb_id, mal_id, anilist_id, tmdb_id, dataset)
```

Library entities carry denormalised display fields (title, overview, dates…) projected from the docs. Those copies count as "cached TMDb data" for terms purposes (§A7).

#### A6. Identity, merges and deletions

**External-id map.** Every library entity has one or more `external_id` rows; lookups by any known id (e.g. a Plex GUID, an indexer's `tvdbid`, a request's TMDb id) resolve through this table. One id may be `active` per (source, id_kind); merged ids stay as `merged` rows pointing at the same entity, so old references (tags written into files, indexer results, history) still resolve.

**Detecting merges.**
- Sources with `Redirects=true` (MusicBrainz, StashDB): a lookup of id X returning an entity whose id is Y ≠ X is a merge X→Y. (MusicBrainz WS/2 serves the target for a redirected MBID; to be re-checked in the spike.)
- TVDB: a series fetch that returns a different series id (Sonarr's "replaced because duplicate", `RefreshSeriesService.cs:83-86`).
- Children: when an authority's child list no longer contains child X, the engine first issues a direct lookup of X (if the source redirects) to distinguish **merged** from **deleted**, instead of guessing from titles.

**Applying a merge X→Y** (in `LibraryPort.Apply`, one transaction):
1. If no local entity has Y: re-key the entity — `external_id(X).state=merged, merged_into=Y`; insert `external_id(Y)` active. Files, monitoring, history, version slots are untouched.
2. If a local entity already has Y: **fold** X into Y — move children/files/history from X's entity to Y's, union monitoring (monitored if either was), keep Y's user settings unless X's were set explicitly more recently, delete X's entity, keep X's id as `merged` on Y. Emit `EntityMerged{from, to}` so import can re-tag files (MusicBrainz ids in tags) and search can re-evaluate.

**Deletion (404).** `ErrNotFound` on a root: set the entity's `upstream_status=deleted`, keep everything, raise a health issue ("removed from TVDB/TMDb/MusicBrainz"), stop refreshing it, offer "re-link to another id" in the UI. On a child without files: delete it. On a child with files: keep it, flag it, and surface it in the health check. This preserves M4 and never silently loses a file.

#### A7. Caching within each source's terms

| Source | Hard max age (terms) | Default soft TTL | Notes |
|---|---|---|---|
| TMDb | **180 days** (MS E2) | policy-driven (§A8) | Applies to `metadata_doc` **and** to TMDb-derived fields copied into library rows and to TMDb images in the media-cover cache. `terms_deadline` makes the planner re-fetch before expiry. |
| TVDB | none found | policy-driven | Attribution link required (MS E1) |
| MusicBrainz / CAA | none (CC0 core data) | 30 d artist, 60 d RG | |
| AcoustID | not cached beyond import | — | Results are applied to files at import, not stored as docs |
| Hardcover, OpenLibrary, Audnexus | none found | 30 d | Hardcover quota drives cadence |
| StashDB, TPDB | none found | change-feed or 30 d | |
| ComicVine, Metron | ComicVine forbids redistribution of altered data (MS E12); local cache is fine | 24 h volume / 30 d issues (Kapowarr's cadence) | |
| TheXEM | unknown (MS E14) | `havemap` daily; per-series on series refresh | |
| Discovery pages (TMDb) | 180 d, but no reason to keep | 15 min–6 h in-process LRU | not persisted |

**Terms enforcement.** A daily singleton sweep finds TMDb-derived rows whose `terms_deadline` is past (TMDb unreachable, key revoked, source disabled). It clears TMDb-only descriptive fields (overview, tagline, images, ratings) and keeps identifiers and user state. Ids are facts the user relies on; whether they also count as cached data is an open question (§6).

**Conditional requests.** Where a source returns `ETag`/`Last-Modified` (CAA, OpenLibrary, some TMDb image hosts), the SourceClient sends `If-None-Match`. A 304 extends `fetched_at` without spending the body parse. It does not reset the TMDb 180-day clock unless TMDb itself answered.

#### A8. Refresh algorithm

Replace "refetch every stale tree on a timer" with **change-driven refresh plus adaptive ceilings**.

```
Planner (singleton, every 5 min):
  1. For each source with a ChangeFeed and an active credential:
       ids := source.Changes(kind, cursor - overlap)          # overlap = 1 h to absorb clock skew
       hits := ids ⋈ external_id(active)                       # only entities we track
       UPDATE metadata_refresh SET next_due_at = now(), priority = CHANGE WHERE root in hits
       advance cursor
     TMDb /changes covers at most 14 days per query; if cursor is older, mark all TMDb roots due
     over the next 24 h (spread) instead of paging.
  2. due := SELECT … WHERE next_due_at <= now() ORDER BY priority, next_due_at
            LIMIT forecast_budget(source, next 5 min)            # never enqueue more than the budget can serve
  3. enqueue refresh jobs (dedupe key = root_ref) on the durable queue

Refresh worker (any node):
  doc, prov, err := Assembler.Assemble(root)                     # sources fetched via SourceClient
  switch err: NotFound → deletion path (§A6); Redirected → merge path; Throttled/Quota → requeue at reset
  if hash(doc) == stored hash: touch fetched_at; else LibraryPort.Apply(ChangeSet)
  next_due_at := policy(root, doc) with jitter ±10 %, capped at terms_deadline − 7 d
```

Policies (floors and ceilings are the references' values; change feeds let the ceilings be long without losing freshness):

| Policy | Rule | Derived from |
|---|---|---|
| `tv_continuing` | change feed; else every 24 h; 6 h floor; immediate if an aired episode is titled "TBA"/"TBD" | `ShouldRefreshSeries.cs:28-70` |
| `tv_ended` | change feed; else 30 d; 7 d if the last episode aired < 30 d ago | same |
| `movie` | change feed; else 30 d if any release date within ±30 d, otherwise 90 d; always before the 180 d terms deadline | `ShouldRefreshMovie.cs:30-50` |
| `music_artist` | 30 d ceiling, 12 h floor, 7 d if a release group is dated within ±30 d | `ShouldRefreshArtist.cs` |
| `music_release_group` | only if monitored or needed for identification; 60 d, or 7 d if released < 30 d ago | `ShouldRefreshAlbum.cs` |
| `book_author` | 30 d; 2 d if continuing with a book < 30 d old; 12 h floor | `ShouldRefreshAuthor.cs` |
| `comic_volume` | 24 h; issues only when `count_of_issues` changes or 30 d | `kapowarr/backend/implementations/volumes.py:1356-1440` |
| `adult_followed` (performer/studio) | change feed; else 7 d for followed, 30 d otherwise | whisparr-eros.md §3 |

**Diff-apply, not replace.** The library diffs children by external id (the `Entity` diffing in Lidarr's `RefreshEntityServiceBase.cs` is the reference outcome). Unchanged documents cost one hash compare and no writes.

#### A9. Rate-limit-aware fetch queues

**SourceClient** wraps every outbound call for a source:
- a **token bucket**. `Scope=Global` sources (MusicBrainz 1/s, AcoustID 3/s, ComicVine, Metron, Hardcover) keep the bucket in `source_state`, taken with a single `UPDATE … RETURNING`. At these rates contention is trivial, and one bucket covers all HA nodes. `Scope=PerNode` sources (TMDb ~40/s, TVDB, CAA, OpenLibrary 1-3/s) use an in-memory bucket per node.
- **window budgets** (ComicVine 200/resource/hour, keyed `comicvine:<resource>`; Hardcover 60/min + 5,000/day; Metron 20/min + 5,000/day), persisted in `source_state.window_used`.
- **shared cooldown**: 429/503 or `Retry-After` sets `cooldown_until` for every node.
- **lanes with reserved share**: `interactive` (user search/add; waits synchronously; 25 % of a strict budget reserved), `add` (populate a newly added item), `change`, `import` (bulk library import), `refresh`. The planner's `forecast_budget` keeps background lanes from starving interactive use.
- identifying **User-Agent** (`bonnyread/<version> ( <project URL> )`, required by MusicBrainz, MS E4), gzip, HTTP/2 where offered, and request coalescing (single-flight per `Ref` per node).
- **mirror awareness**: if the MusicBrainz base URL is not `musicbrainz.org`, the user supplies the rate (a self-run mirror has no 1/s limit, MS F4).

#### A10. Keys and credentials (D3b, D16)

Resolution order per source: **user credential → project credential → none** (source disabled or degraded).
- Project keys (TVDB, TMDb, AcoustID, the Hardcover OAuth client id) are registered by the project and injected at build time (`-ldflags` into a `projectkeys` package), clearly attributed, and shown in settings as "project key (override)". Nothing is copied from other apps (MS F9).
- TVDB: project key plus an optional user **PIN** (`/login {apikey, pin}`); the bearer token (valid ~1 month) is cached in `credential.secret` and refreshed on 401 (MS E1, F1).
- Hardcover: **OAuth device-authorization grant** with the project's client id; the refresh token is stored; the 1 January key expiry and token resets become health issues with a "re-authorise" action (MS E7, F5).
- StashDB, ThePornDB, ComicVine, Metron: user keys only; the kind degrades gracefully without them (the UI explains how to obtain one; StashDB needs the rotating invite, MS E11).
- Keys are **installation-scoped**, set by an admin (D17 roles), not per request-user. "Per-user key" in D3b means "the operator's own account".
- Every credential has a `Test()` path and a periodic validation job. `AuthError` marks the source degraded and raises a health issue, never a crash loop.

#### A11. Attribution

`Attribution{Name, URL, LogoAsset, Notice, Placement}` is part of `SourceInfo`. The API exposes `/api/v1/metadata/sources` (enabled sources, attribution, key origin, health, quota use). The UI shows:
- an "About data sources" page with the TMDb logo and the notice "This product uses the TMDB API but is not endorsed or certified by TMDB" (MS E2), plus TVDB, MusicBrainz, ComicVine (link-back), TVMaze (if ever used) and StashDB/TPDB credits;
- a per-item provenance footer ("Episode data: TheTVDB · Images: TMDB").

#### A12. TV specifics

- **Episode orders.** Store TVDB default numbering as canonical. Also store TVDB absolute numbers (TVDB "absolute" season type), then XEM scene numbering. Parsing and search use: scene numbering if present, else TVDB; anime additionally uses absolute (scene absolute first). This is Sonarr's `ParsingService` behaviour (sonarr.md §3), kept as-is.
- **XEM** (`https://thexem.info/map/`, MS C18, E14): `havemap?origin=tvdb` daily (one call, cached) to learn which series have maps; `all?id=&origin=tvdb` on series add/refresh for mapped series; `allNames?origin=tvdb&seasonNumbers=1` daily for scene names. Extrapolation and the `unverified` flag follow `XemService.cs:100-200`.
- **Scene mappings and the daily list (D3c).** No public replacement for `services.sonarr.tv` exists (MS F2). Plan: a **community dataset in its own git repository** (`scene-mappings`: JSON records in Sonarr's `SceneMapping` shape, plus a `daily` list), downloaded at runtime into the data directory, pinned by commit and verified by sha256, refreshed every 6 h. It is not embedded in the binary, following D5b's reasoning. Seed it from XEM `allNames`, TVDB aliases, and a daily-type heuristic (TVDB seasons numbered by year with > 100 episodes). User-local mappings are layered on top and can be contributed upstream with one click (a PR link), mirroring D9a's authoring model.
- **Anime ids.** AniDB/MAL/AniList ids come from the Fribb `anime-lists` and Anime-Lists `anime-list.xml` datasets (already used by Bazarr and Seerr, `bazarr/bazarr/subtitles/refiners/anilist.py:27`, `anidb.py:63`, `seerr/server/api/animelist.ts`). They are downloaded at runtime the same way. No AniDB HTTP API client is needed.
- **Series type.** `anime` is suggested when TVDB genres include Anime (or original language is ja/zh/ko and genre Animation) and confirmed by the user; `daily` comes from the dataset or the heuristic.

#### A13. Music specifics (MusicBrainz + CAA + AcoustID)

Direct MusicBrainz at 1 req/s works only if the design **never asks for trees it does not need**:
- **Artist doc** = `artist/{mbid}?inc=aliases+url-rels+genres` (1 call) + browse `release-group?artist={mbid}&limit=100&type=…` pages (≈1–3 calls). The metadata profile's primary/secondary types and statuses are pushed **into the query** where MusicBrainz supports filtering by type, instead of fetching everything and filtering locally (lidarr.md §3.5).
- **Release-group detail** (releases, media, tracks, recordings) is fetched **lazily**: when the album becomes monitored, when a search needs track counts, or when import needs candidates. That is one `release?release-group={rg}&inc=media+recordings+artist-credits+labels` browse per release group (paged if it has many releases).
- **Identification at import** prefers MBIDs already in tags (Picard-tagged libraries): `release/{mbid}?inc=…` directly (1 call). Otherwise search `release?query=` (1 call) plus the candidate lookups. **AcoustID** (3 req/s, own budget) is used only for files that remain ambiguous, and returns recording ids that feed Lidarr's distance matcher (the identification design owns that matcher).
- **CAA** covers are fetched by the media-cover service on demand (no rate limit is published, MS E6; a polite per-node limit of 5/s is used anyway).
- **Optional mirror**: a user-run MusicBrainz mirror (musicbrainz-docker with replication) is a base-URL setting plus a rate override (§A9).

#### A14. Books, adult, comics (phase 3)

- **Books.** The Hardcover GraphQL client batches up to 5 top-level queries per request (MS E7) and implements `BatchFetcher`. With 5,000/day, a bulk import of 2,000 authors is split across days by the budget planner; OpenLibrary (keyless, 1–3 req/s, MS E8) fills ISBN lookups meanwhile, so items show immediately with partial data. Every book has an `external_id` for Hardcover, OpenLibrary work/edition, ISBN-10/13, ASIN and Goodreads (legacy, for matching old tags only).
- **Adult (D10).** StashDB is the authority for scenes, performers and studios; TPDB and TMDb ids are linked from StashDB's cross-references. Product codes are stored as `external_id(source="code")` (D10 restore item 1). Performer merges use StashDB redirects (Eros `MergedIntoId`). Without keys, the kind stays usable for manually added TMDb movies only.
- **Comics.** ComicVine `volumes/?filter=id:a|b|…` (100 ids per call) implements `BatchFetcher`, so a 1,000-volume refresh costs 10 calls of the `volumes` resource budget. Issue lists are fetched only when counts change (Kapowarr's rule).

#### A15. Discovery for requests (replaces Seerr's stack)

- `Discoverer` on TMDb serves trending, popular, upcoming, `/discover/{movie,tv}` with filters, recommendations/similar, person and collection pages, keyword/company/network lookups (the endpoint set in seerr.md §3). Results are cached per node in an LRU (15 min for lists, 6 h for detail pages). Nothing is persisted, so the 180-day rule is satisfied trivially.
- **Availability** is not a scanner: results carry TMDb ids, which are joined against `external_id(source=tmdb)` to return library status per version slot (D4). This is one query, owned by the library/requests design.
- **TV requests** carry a TMDb TV id. On request the TV Assembler resolves it to TVDB through `IDResolver`: TMDb `external_ids.tvdb_id`, else a TVDB remote-id search by IMDb id, else a manual pick. The library entity is always TVDB-keyed (M1). An unresolved mapping blocks auto-approval and asks an admin to pick.
- Seerr's keyword blocklist, override rules and so on belong to the requests design; they call `Discoverer` too.

#### A16. HA placement (D14)

| Component | Placement |
|---|---|
| Planner (change feeds, due selection, terms sweep, dataset refresh, XEM `havemap`) | **Singleton** under leader election |
| Refresh / add / import fetch jobs | **Any node**, via the durable queue |
| Interactive search/lookup, discovery | **Any API node**, directly through SourceClient (interactive lane) |
| Global buckets, window budgets, cooldowns | **DB rows** (`source_state`); single-node SQLite keeps them in memory with periodic persistence |
| Per-node buckets (TMDb, TVDB, CAA, OpenLibrary) | In memory per node, because these limits are per IP; if all nodes share one egress IP, set the per-node rate to limit ÷ node count |
| Discovery LRU, single-flight | Per node; no cross-node invalidation needed (short TTL) |
| `metadata_doc`, external ids, datasets | DB (shared) |

#### A17. Failure modes and health

Health issues are raised for: a source degraded (auth, quota exhausted, cooldown > 1 h); a missing user key for an enabled kind; datasets older than 7 days; terms-deadline purges; items deleted upstream; unresolved TMDb→TVDB mappings; MusicBrainz import backlog ETA > 24 h (informational, with a "use a mirror" hint). Metrics: calls, 429s, budget used and remaining per window, queue depth per lane, doc cache hit rate, refresh lag (D7 observability).

### 3.B Subtitles (phase 2)

#### B1. Flow

```
import: MediaFileImported / Upgraded / Deleted ──► subtitle service
probe (shared, from import): streams incl. subtitle codec, language, forced/hearing_impaired dispositions
                                   │
                    Wanted(file) = profile − present   (pure function, B3)
                                   │
    Search job (any node): build VideoFacts → fan out to enabled provider plug-ins (B10)
                                   │  candidates
    Host matching (shared release parser, D1c) → scoring (S1-S3) → per-want-key selection (S4-S6)
                                   │  best candidate per want key
    Download (plug-in) → decode archive (host) → UTF-8 → format → mods → sync (gated) 
                                   │
    import.PlaceSubtitle(file, bytes, lang, hi, forced, format) → subtitle row → events
```

No mirror, no path mapping, no rescan round-trip: the subtitle domain reads the library directly and FKs to the media file.

#### B2. Data model

```
subtitle                              -- THE extra-files subtitle table (shared with import; one index, not two)
  id, media_file_id FK (ON DELETE CASCADE)
  origin        embedded | external | downloaded | uploaded | extracted | translated
  language      text   -- BCP 47-style: ISO 639-3/1 + optional region/script (pt-BR, zh-Hant, es-419)
  hi, forced    bool
  format        srt | ass | ssa | vtt | sub | pgs | vobsub | …
  path          nullable (embedded → null)
  stream_index  nullable (embedded only)
  provider, provider_sub_id, release_info, uploader     nullable
  score, score_max, matches text[]                      nullable (downloaded only; S7)
  sync          jsonb nullable  {tool, reference, offset_ms, framerate_fixed, checker_score}
  mods          text[]
  created_at

subtitle_profile(id, name, cutoff_item_id nullable, must_contain text[], must_not_contain text[], original_format bool)
subtitle_profile_item(id, profile_id, ord, language, forced bool,
                      hi enum(required, normal, excluded),
                      audio_rule enum(always, skip_if_audio_is, only_if_audio_is))
language_equivalence(from_lang, to_lang)               -- global `language_equals`
subtitle_search_state(media_file_id, want_key, first_missing_at, attempts, last_attempt_at, next_attempt_at)
subtitle_blocklist(item_id, provider, provider_sub_id, created_at, reason)
subtitle_provider(id, plugin_id, enabled, priority, settings jsonb, credential_ids)
subtitle_provider_status(provider_id, throttled_until, reason, failures, last_error, daily_used, daily_reset_at)
```

The profile is assigned on the item (series/movie) with root-folder/tag defaults (library design) and applies to every file of every version slot. History uses the shared history/event table with subtitle event types (downloaded, upgraded, manual, uploaded, synced, modified, deleted, extracted, blocklisted). Bazarr's numeric action codes are not kept (D7).

#### B3. Language profiles and the wanted computation

A **want key** is `(language, forced, hi-mode)` from a profile item. Wanted is computed on demand, not stored (Bazarr's stringified `missing_subtitles` column goes away):

```
Wanted(file, profile, present, audioLangs):
  for item in profile.items (ordered):
    if item.audio_rule excludes this file given audioLangs: continue
    if satisfied(item, present): 
        if item.id == profile.cutoff_item_id: return ∅       # cutoff reached: nothing more wanted
        continue
    emit item
satisfied(item, p ∈ present) ⇔ equiv(p.language, item.language)
                              ∧ p.forced == item.forced
                              ∧ (item.hi == required ⇒ p.hi) ∧ (item.hi == excluded ⇒ ¬p.hi)
                              ∧ p counts (embedded image formats only if not ignored; B4)
```

Forced items never carry `hi=required` (S5); the profile editor enforces this. "Normal" accepts either HI or non-HI; that matches Bazarr's collapse of "only/also/never" (`bazarr/bazarr/constants.py:14-17`).

#### B4. Embedded and external detection

- **Embedded.** The shared probe (one ffprobe per file, owned by import; comparison.md §2.3 "Media probing") records every subtitle stream: codec, language tag, title, and dispositions `forced` and `hearing_impaired`. The subtitle service maps them to `subtitle(origin=embedded)` rows. Title heuristics ("SDH", "CC", "Forced", "Signs") fill missing dispositions. Settings: ignore PGS / VobSub / ASS, and a default language for `und`. Bazarr's `embeddedsubtitles` provider disappears: embedded tracks are already "present".
- **External.** Import indexes sidecars when it imports or rescans (`name.<lang>[.hi|.cc|.sdh][.forced].<ext>`, subfolder modes). Content-based language detection (lingua-go) is used only when the filename has no language.
- **Extraction** (embedded text track → external file) is an explicit action or a profile option. It runs `ffmpeg` as a tool job (B12).

#### B5. Search orchestration and triggers

| Trigger | Action |
|---|---|
| `MediaFileImported` (new or upgrade) | index embedded streams; compute wanted; enqueue a search job (priority high). Release info (scene name, group, source, streaming service) comes from the grab/import record in-process, replacing Bazarr's `use_scenename` and the AvistaZ `nzbInfoUrl` history call (`bazarr/bazarr/subtitles/refiners/arr_history.py`) |
| `MediaFileReplaced` | subtitles of the old file with `origin=downloaded` are dropped (they are timed to the old encode); import deletes the sidecars; the new file is searched |
| `MediaFileDeleted` | rows cascade; import removes sidecars |
| Profile edited / item re-assigned | recompute wanted for affected files; batch-enqueue searches at low priority |
| Wanted sweep (singleton, 6 h default) | enqueue files with wanted keys whose `next_attempt_at` ≤ now |
| Adaptive back-off (S8) | after a key has been missing for 3 weeks, `next_attempt_at` = +1 week |
| Upgrade sweep (singleton, 12 h) | B8 |
| Manual / interactive search | synchronous fan-out; returns every candidate with score and matched/unmatched attributes |
| Playback started (optional, from media-server integration) | search-on-play for that file |

**One provider call per (file, provider).** The call passes all wanted languages with a `forced_mode` of `only`, `also` or `none` (from the want keys, as Bazarr's `_set_forced_providers` does, `download.py:48-50`). Selection then runs per want key. Providers are called concurrently with a per-call deadline; throttled providers are skipped.

**VideoFacts** given to providers: kind; titles (primary, original, alternates, scene titles from §A12); year; ids (IMDb, TMDb, TVDB series/episode, AniDB, MAL, AniList); season/episode in TVDB **and** scene numbering, absolute number, special flag; parsed release (group, source, resolution, codecs, streaming service, edition); file size, duration, fps; hashes the provider's manifest asks for (OpenSubtitles hash, napiprojekt MD5 of the first 10 MB), computed by the host once per file.

#### B6. Matching, scoring and selection (preserved)

- **Matching moves to the host.** Plug-ins return candidate facts (release names, season/episode, year, ids they matched by, hash-matched flag, HI/forced flags, format). The host parses release names with the shared parser (D1c) and compares them with VideoFacts to produce the match set. Title comparison uses normalised titles plus aliases and scene titles. Provider "claims" (`hash`, `imdb_id`, `tvdb_id`, `series_imdb_id`, `series_tvdb_id`) are accepted only if the manifest declares them.
- **Scoring**: exactly S1–S3, including the hash-trust guard and equivalences, and the `hearing_impaired` +1 as in S4. Scores are stored with `score_max`.
- **Selection per want key**: keep candidates whose language is equivalent and whose forced flag equals the key's; apply `must_contain`/`must_not_contain` to release info; drop blocklisted (provider, id); for `hi=required|excluded`, skip mismatches from HI-verifiable providers (S4); sort by (score, score-without-hash); stop below `minimum_score` (90 % / 70 %, configurable); for episodes require series+season+episode unless the hash is unverifiable (S6); try the next candidate on download failure.
- A **golden test suite** ports Bazarr's scoring and selection tests plus recorded provider fixtures; it is the acceptance gate for S1–S6.

#### B7. Post-processing

1. **Decode.** If the plug-in returns an archive, the host extracts it (zip, RAR via pure-Go `rardecode`, 7z via `sevenzip`, with size and entry limits) and picks the entry by episode match (the parser again).
2. **Normalise.** Detect the encoding and re-encode to UTF-8 (unless "keep encoding"); convert to SRT unless the profile's `original_format` is set (go-astisub or a small converter).
3. **Mods.** A native Go port of subzero's mods: common, remove_tags, reverse_rtl, fix_uppercase, emoji, OCR_fixes (with the per-language dictionaries), remove_HI, color, shift_offset, change_FPS, two_point_fit. **remove_HI is never applied to a `hi=required` subtitle**, which would defeat the want key.
4. **Sync** (gated, S12): runs when the score is below the sync threshold, or on demand. `Syncer` interface with two external-tool implementations:
   - **alass** (single static binary; subtitle-to-subtitle or video reference via ffmpeg audio): the default when present.
   - **ffsubsync** (Python; Bazarr's tool): supported for parity when installed.
   - Reference preference: an embedded text subtitle of any language (fast, precise), else the audio track.
   - **Quality checker** (kept from Bazarr): reject if |offset| > `max_offset` (60 s), if a framerate change was applied and not allowed, or if the tool's confidence is below the minimum. A rejected sync keeps the unsynced file and records why.
   - Native Go alignment is a later option, not a phase-2 goal.
5. **Place.** `import.PlaceSubtitle(file, bytes, lang, hi, forced, format)` names and writes the sidecar (HI suffix `hi|cc|sdh`, single-language mode, subfolder mode, chmod), replaces an upgraded predecessor atomically, and returns the `subtitle` row. Then history and events follow (notifications and media-server refresh are other designs).

Translation (Google/Gemini/Lingarr) and Whisper generation are **not in the first subtitle release**. When added, they are host-side "generators", not providers, because Whisper needs audio extraction. They produce `origin=translated|generated` rows at a fixed default score (Bazarr: 50).

#### B8. Upgrade logic

The upgrade sweep (singleton, every 12 h) selects `subtitle` rows with `origin=downloaded` (plus `manual`/`uploaded`/`translated` if "upgrade manual" is on), `created_at` within `days_to_upgrade` (7), and `score < score_max − 3`, whose want key is still wanted with the same HI mode (S7). It enqueues a search for that one want key with `min_score = old score + 1` (Bazarr's `forced_minimum_score`, `download.py:66-68`). A winner replaces the old file atomically via `PlaceSubtitle`. The upgrade chain is kept in history (`upgraded_from`).

#### B9. Provider throttling and blocklist

- Plug-ins return **typed errors** (B10). The host maps them to back-off durations: the defaults are Bazarr's table (`get_providers.py:69-86`: too-many-requests 1 h, download limit 3 h, unavailable 20 min, API throttled 10 min, parse error 6 h, timeout 1 h, auth/config 12 h), overridable per plug-in in its manifest, including `reset: {at: "00:00", tz: "Europe/Prague"}` for daily limits (titulky, legendasdivx, subdl GMT).
- State lives in `subtitle_provider_status` (DB), so every HA node sees the same throttle. A `blocklist_this` error blocklists (provider, subtitle id) automatically.
- Provider status is the generic provider-status/back-off service shared with indexers and download clients (comparison.md §2.3 "Rate limits"), not a subtitle-specific mechanism.
- Manual blocklist deletes the file, records (provider, id) for the item, and re-searches.

#### B10. Plug-in contract (WASM, D15)

**Runtime.** wazero, in-process. The module is compiled once per node (on-disk compilation cache); each call runs in a fresh instance (or a pooled one reset between calls), with a memory cap (default 64 MiB), a wall-clock deadline via context (search 20 s, download 60 s) and WASI preview 1 with **no preopened directories, no environment, no sockets**. Guests are written in Go (≥ 1.24, `GOOS=wasip1`, `//go:wasmexport`), TinyGo or Rust. The byte-level ABI (below) is shared with every D15 plug-in kind; only the message schemas are subtitle-specific.

**Guest exports** (ABI v1; buffers are JSON in guest memory; a `u64` return packs `ptr<<32 | len`):

| Export | Purpose |
|---|---|
| `bonny_abi() -> u32` | ABI version (1) |
| `bonny_alloc(len u32) -> u32`, `bonny_free(ptr, len)` | host writes requests into guest memory |
| `bonny_describe() -> u64` | returns the **manifest** |
| `bonny_configure(ptr, len) -> u64` | settings + credentials → `{ok, field_errors}` |
| `bonny_search(ptr, len) -> u64` | `SearchRequest` → `SearchResponse` |
| `bonny_download(ptr, len) -> u64` | `DownloadRequest` → `DownloadResponse` |
| `bonny_test(ptr, len) -> u64` (optional) | connectivity/credential test |

**Manifest** (validated on install; the admin approves the declared hosts and capabilities):

```yaml
id: opensubtitlescom
abi: 1
version: 1.2.0
license: GPL-3.0-or-later
kind: subtitle-provider
media: [movie, episode]
languages: [eng, fra, por-BR, zho-Hant, …]           # or "any"
hosts: [api.opensubtitles.com, "*.opensubtitles.com"] # egress allowlist
capabilities:
  hashes: [opensubtitles]           # host computes and sends these
  claims: [hash, imdb_id, series_imdb_id, tmdb_id]
  hi_verifiable: true
  forced_verifiable: true
  archives: false                   # may return archives for host extraction
  cloudflare: false                 # may request the FlareSolverr path
  captcha: false                    # may call captcha_solve
rate: {rps: 5, burst: 5}
quotas: [{name: downloads, per: day, limit_from_setting: daily_downloads, reset: {at: "00:00", tz: UTC}}]
throttle: {too_many_requests: 1m, download_limit: 6h, provider_error: 1m}
credentials:
  - {name: api_key, kind: api_key, project_default: true}   # host supplies (D3b); never inside the module
  - {name: username, kind: text}
  - {name: password, kind: secret}
settings_schema: { …JSON Schema for the UI form… }
flags: {ai_translated_opt_in: true, machine_translated_opt_in: true}
```

**Messages** (abridged):

```go
type SearchRequest struct {
    Video     VideoFacts  // B5
    Languages []LangQuery // {Code, ForcedMode: "none"|"also"|"only", HIPreference: "required"|"normal"|"excluded"}
    Limit     int
}
type Candidate struct {
    ID            string   // provider subtitle id (blocklist key)
    Language      string
    HI, Forced    *bool    // nil = unknown (then not verifiable for this candidate)
    Format        string
    ReleaseNames  []string // host parses these (D1c)
    Season, Episode, Year *int
    FPS           *float64
    Claims        []string // subset of manifest.capabilities.claims
    Uploader, PageURL string
    AITranslated, MachineTranslated bool
    DownloadToken []byte   // opaque; echoed back in DownloadRequest
}
type SearchResponse   struct{ Candidates []Candidate; Error *PluginError }
type DownloadRequest  struct{ Candidate Candidate }
type DownloadResponse struct{ Filename string; Content []byte; IsArchive bool; Error *PluginError }
type PluginError struct {
    Code       string // too_many_requests | download_limit | api_throttled | unavailable | parse |
                      // auth | config | not_found | ip_blocked | blocklist_this
    RetryAfter *int64 // seconds
    ResetAt    *int64 // unix
    Message    string
}
```

**Host functions** (import module `bonny`; every call is checked against the manifest):

| Function | Behaviour |
|---|---|
| `http_do(req) -> resp` | Through the core HTTP client: allowlist, per-host rate limit and plug-in budget, proxy settings, TLS verification always on, redirects within the allowlist, 20 MiB response cap, per-instance cookie jar persisted via kv, optional FlareSolverr path if `cloudflare: true` and configured (the same capability the indexer layer uses) |
| `kv_get / kv_put(key, value, ttl) / kv_delete` | Namespaced per plug-in instance, stored in the DB (shared across nodes): session tokens, login cookies, short result caches |
| `archive_list / archive_read(bytes)` | Optional helper so a plug-in can pick an entry itself; the host otherwise extracts after download |
| `captcha_solve(kind, site_key, page_url) -> token` | Only with `captcha: true` and a configured solver (anti-captcha, etc.) |
| `log(level, msg)` | Structured, tagged with plug-in id |
| `now_ms()` | Wall clock; WASI random is available for nonces |

**Credentials** are delivered in `bonny_configure`, never compiled into modules. Project defaults (e.g. an OpenSubtitles.com application key) are held by the host under D3b and are user-overridable.

**Distribution.**
- First-party plug-ins live in the repo (`plugins/subtitles/<id>`). They are built to `.wasm` in CI and embedded in the binary, so the ported set works offline and is versioned with the core.
- Third-party and updated plug-ins come from a plug-in index (a git repository, like the Cardigann definitions), pinned by sha256. The manifest's hosts and capabilities are shown for consent on install and on any change.
- A per-plug-in conformance kit (recorded HTTP fixtures and expected candidates) runs in CI; this is also the regression net for scraping providers.

#### B11. Which providers to port first

Bazarr offers ~55 providers in the UI, ~50 of them live remote providers (bazarr.md §4.1). Criteria: usage, API stability (JSON over scraping), language coverage, and whether they need Cloudflare or captcha capabilities.

| Wave | Providers | Why |
|---|---|---|
| **Built-in (host, not plug-ins)** | embedded tracks (B4) | Local; replaces the `embeddedsubtitles` provider |
| **1 — launch of phase 2** | opensubtitlescom, subdl, subsource, gestdown (Addic7ed data without scraping), jimaku, animetosho, napiprojekt (hash) | JSON APIs covering most users and anime; hashes give the highest-confidence matches |
| **1b — JSON regional** | subsro, regielive (RO), subtis, subx (ES), assrt (ZH), betaseries (FR), bayflix, vladoonmooo (BG), tsukihime (anime), hdbits (private-tracker API), subsarr / subsdump (self-hosted Subscene archives) | Small (150–650 LOC each, bazarr.md §11), APIs, regional coverage |
| **2 — simple HTML scrapers** | yifysubtitles, subf2m, animesubinfo, napisy24, legendasnet, subtitulamostv, subclub, subssabbz, subsunacs, yavkanet, titrari, subtitrarinoi, karagarga, ktuvit, xsubs, pipocas, animekalesi, subtitriid, nekur, wizdom, subsynchro, subtitlecat (opt-in, machine translations) | Need the archive helper and fixtures; brittle, so community maintenance via the plug-in index matters |
| **3 — need Cloudflare/captcha capabilities** | addic7ed (direct, VIP), titlovi, zimuku, subs4series, greeksubs, prijevodionline, soustitreseu, legendasdivx, turkcealtyaziorg, titulky, avistaz/cinemaz | Depend on FlareSolverr or solvers; port once those host capabilities are proven |
| **Later (host generators)** | whisperai, translation backends | Need audio extraction; B7 |
| **Do not port** | bsplayer, greeksubtitles, subs4free, hosszupuska, shooter, subscenter, argenteam, legendastv, thesubdb, podnapisi (removed upstream; its config section is deleted at startup, `bazarr/bazarr/app/config.py:636`); hidden supersubtitles/tvsubtitles unless someone asks | Dead, hidden or removed (bazarr.md §4.1, §10) |

#### B12. HA placement

| Component | Placement |
|---|---|
| Wanted sweep, upgrade sweep, plug-in index refresh | **Singleton** |
| Search / download jobs, interactive search | **Any node**; plug-ins compiled per node |
| Provider status, quotas, kv store, blocklist | **DB** (shared) |
| Sync, extraction (ffmpeg, alass, ffsubsync) | **Tool jobs** routed to nodes that advertise the tool; media paths are shared storage (D14) |
| Placement of files | import's port (same shared storage) |

---

## 4. Walkthroughs

### 4.1 Adding an anime series (TVDB + XEM + TMDb images)

The user searches "Frieren".
1. **Search** (interactive lane, any API node): `TVDB /search?query=Frieren&type=series` returns candidates with remote ids. For each candidate with a TMDb id, the cached or fetched TMDb poster is attached (TVDB image as fallback). Attribution for both is shown.
2. **Add.** The TV Assembler fetches:
   - TVDB `/series/{id}/extended?meta=episodes&short=true` (default order, aliases, translations, remote ids) and the absolute-order episode list;
   - TMDb `/tv/{tmdbId}?append_to_response=images,external_ids,content_ratings` (images, rating, cross-check of `tvdb_id`), `hard_expires_at = now + 180 d`;
   - XEM: the daily-cached `havemap` shows the series is mapped, so `map/all?id=<tvdb>&origin=tvdb` returns scene numbering;
   - datasets: `anime_id_map` for AniDB/MAL/AniList ids; `scene_mapping` for scene titles such as "Sousou no Frieren".
3. **Series type.** TVDB genre "Anime" suggests `anime`; the user confirms it in the add dialog.
4. **Build `SeriesDoc`.** Episodes carry TVDB season/episode, absolute number, scene season/episode/absolute (XEM; extrapolated rows flagged `unverified`), UTC air dates, specials placement. `Provenance` records TVDB (structure), TMDb (images, rating), XEM (scene numbers).
5. **Apply.** `LibraryPort.Apply` creates the series and episodes, writes `external_id` rows (tvdb, tmdb, imdb, anidb, mal, anilist) and a `metadata_refresh` row with policy `tv_continuing`, due in 24 h ± jitter and capped by the TMDb terms deadline. It emits `SeriesAdded`; the acquisition design takes over from there.
6. **Later.**
   - A new episode appears on TVDB: the planner's `/updates?since=` poll hits this series, the refresh job re-assembles it, the doc hash differs, and the episode is added. XEM `all` is re-read in the same job, so the new episode gets extrapolated scene numbering until XEM maps it.
   - An aired episode still titled "TBA" forces a refresh 6 h later.
   - The daily `allNames` refresh adds new scene names.

API cost: ~5 calls, all on per-node budgets. No hosted service is involved.

### 4.2 A MusicBrainz release-group merge

The library has artist A with albums RG1 (files present, monitored) and RG2 (no files, monitored). MusicBrainz editors merge RG1 into RG2.
1. The `music_artist` refresh for A browses A's release groups: RG2 is present, RG1 is missing.
2. For the missing RG1, the engine looks it up directly: `release-group/RG1` returns an entity with id RG2, so `RedirectedError{To: RG2}`. It does not guess from titles.
3. The ChangeSet carries `Merged: [RG1 → RG2]`. The releases and recordings under RG1 are checked the same way when RG2's detail is fetched (lazy fetch, triggered because a monitored album changed).
4. `LibraryPort.Apply`, in one transaction:
   - RG2 already exists locally, so the albums are **folded**: RG1's release rows, track-file links and history move to album RG2. Monitored = RG1 ∨ RG2 = true. The selected release stays the one that has files, since its release MBID is unchanged or also redirected.
   - `external_id(RG1)` becomes `merged → RG2` on album RG2, and the RG1 album row is deleted.
   - `EntityMerged{RG1 → RG2}` is emitted. Import/tagging re-writes the MusicBrainz release-group id in the file tags if tag writing is enabled. Wanted/search recomputes, and RG2 is now satisfied by the moved files, so no needless search runs.
5. If step 2 had returned 404: RG1 has files, so it stays, flagged "removed from MusicBrainz", and a health issue offers "re-link". If RG1 had had no files, it would simply be deleted.

Old references (a history row, a tag in a file, an indexer result carrying RG1) still resolve through the `merged` row.

### 4.3 First import of a 5,000-album library under MusicBrainz limits

Assume 5,000 album folders by ~800 artists; 60 % already Picard-tagged with MBIDs; MusicBrainz at 1 req/s (≈3,600 calls/h); AcoustID at 3 req/s.
1. **Scan (import design)** reads tags and paths for all files. There are no network calls.
2. **Tagged albums (3,000).** One `release/{mbid}?inc=media+recordings+artist-credits+release-groups+labels` per album gives 3,000 calls. The release group and the artist ids come back in the same response.
3. **Untagged albums (2,000).** One `release?query=` search each, plus on average 1.5 candidate lookups, gives ≈5,000 calls. About 10 % stay ambiguous and go to AcoustID fingerprint lookups (≈2,000 files at 3 req/s ≈ 11 min, on a separate budget).
4. **Artists (800).** `artist/{mbid}` plus ~1.5 release-group browse pages each gives ≈2,000 calls. Only release-group *summaries* are fetched. Release and track detail for albums the user does not own are **not** fetched until they are monitored (§A13).
5. **Total** ≈ 10,000 MusicBrainz calls ≈ **2.8 h** at 1 req/s. Lidarr-style full trees would need every release of every release group of 800 artists, often 50k+ calls (> 14 h).
6. **Ordering and UX.**
   - Import jobs use the `import` lane. The `interactive` lane keeps 25 % of the bucket, so the UI's own searches stay responsive.
   - Albums appear as they are identified: tagged ones first, since they need 1 call each.
   - A progress/ETA metric is exposed. If the ETA exceeds 24 h, a hint suggests pointing at a MusicBrainz mirror.
   - CAA covers load lazily as album pages are viewed or the cover cache warms at a low rate.
7. **Afterwards.** The refresh rows get `music_artist` / `music_release_group` policies with jitter spread over their ceilings. Steady-state cost: 800 artists / 30 d ≈ 27 artist refreshes per day plus recently released groups, a few hundred calls a day.
8. **HA.** Jobs run on any node; the single DB-backed bucket keeps the cluster at 1 req/s in total. A 503 from MusicBrainz sets `cooldown_until` for everyone.

### 4.4 Subtitle search for a newly imported episode requiring HI + forced

Profile "English (SDH + forced)": item 1 `eng, forced=false, hi=required`; item 2 `eng, forced=true, hi=normal`, `audio_rule=always`; no cutoff; minimum score 90 %.
1. **Import** emits `MediaFileImported{file 812, episode S02E05, release "Show.S02E05.1080p.AMZN.WEB-DL.DDP5.1.H.264-GRP"}`. The shared probe shows one embedded English SubRip track with no dispositions and the title "English".
2. **Wanted.** The embedded track is `eng, hi=false, forced=false`. It does not satisfy item 1 (HI required) or item 2 (forced), so wanted = {`eng:hi`, `eng:forced`}.
3. **Search job.**
   - VideoFacts: series titles + aliases + scene titles; TVDB S02E05 plus the scene numbering if different; year; IMDb/TVDB/TMDb ids; release group GRP, source WEB-DL, streaming AMZN, 1080p, H.264, DDP; file size, fps; OpenSubtitles hash (computed once).
   - Each enabled, non-throttled provider gets one call with `Languages: [{eng, ForcedMode: "also"}]`.
4. **Candidates.** For example:
   - OpenSubtitles: `A` (HI, hash claim, release "…AMZN.WEB-DL…-GRP"), `B` (non-HI, same release), `C` (forced, "foreign parts only", non-HI);
   - subdl: `D` (HI, release "…WEB-DL…-OTHER").
5. **Matching and scoring** (host). A: the hash claim is trusted only if series+season+episode+source match (they do), so it scores 359, plus +1 hearing_impaired because the key wants HI = **360**. B: series 160 + season 30 + episode 30 + year 90 (via the imdb equivalence) + source 25 + group 20 + codecs/resolution/streaming 4 = 359; no HI match. D: 160+30+30+90+25+4, +1 HI = 340.
6. **Selection.**
   - `eng:hi`: B is skipped because it is not HI and the provider verifies HI. A (360) is the best candidate and above 324 (90 % of 360).
   - `eng:forced`: only C carries `forced=true`; A, B and D are excluded because they are full subtitles. C matches the same release (359) and gets the +1 because the forced key's HI mode is "normal" and C is non-HI, so it scores 360 and is selected.
7. **Download and post-process.** A and C are downloaded. Both are UTF-8 SRT; mods run without `remove_HI` for A. Both score ≥ the sync threshold, so neither is synced.
8. **Place.** `PlaceSubtitle` writes `Show - S02E05.en.sdh.srt` (HI extension setting "sdh") and `Show - S02E05.en.forced.srt`. Two `subtitle` rows are created (origin downloaded, scores 360/360), history is written and events fire.
9. **If no forced subtitle existed.** `eng:forced` stays wanted and `subtitle_search_state` records the attempt. The 6-hourly sweep retries it; after 3 weeks it is retried weekly. If the user later sets `audio_rule=only_if_audio_is ≠ eng` on the forced item, it stops being wanted for English-audio files.
10. **Upgrade.** Had A scored 340, it would be eligible for 7 days (340 < 357) and re-searched every 12 h with `min_score = 341`.

---

## 5. Simplifications compared with the references

| Reference behaviour | This design | Why it is safe |
|---|---|---|
| Five hosted metadata proxies (SkyHook, RadarrAPI, LidarrAPI, Whisparr API, bookinfo.pro) | Direct calls behind per-kind Assemblers | D3c; every source permits direct self-hosted use (MS §3) |
| Full-tree refetch per refresh (Lidarr artist → all RGs → releases → tracks) | Change feeds + adaptive ceilings + lazy release-group detail + hash-compare no-op | Same freshness outcome (M6, M7); ~5× fewer MusicBrainz calls on import (§4.3) |
| One "foreign id" per entity; Bookshelf's two incompatible DB flavours; Lidarr's `Old*Ids` columns per table | One `external_id` table with `active/merged/deleted` states | M5, M9 kept; merges handled once, for every kind |
| Merge detection by title/diff heuristics | Direct lookup of vanished ids (redirect ⇒ merge, 404 ⇒ delete) | Deterministic |
| Seerr's TMDb client, `Media` table, 4 scanners, availability sync, 4K columns, IMDb/RT borrowed keys | `Discoverer` + availability as a join on `external_id`; ratings from TMDb only | M11 kept; D4 slots replace 4K duplication |
| Sonarr's `services.sonarr.tv` scene mappings and daily list | Runtime-downloaded community dataset + XEM names + local overrides | D3c; coverage gap is flagged (§6) |
| AniDB HTTP client (Bazarr) and Anime-Lists copies in Seerr | One `anime_id_map` from runtime-downloaded datasets | Same ids, one place |
| Per-app hard-coded keys | Project keys at build time, user-overridable; per-user keys where required | D3b |
| Bazarr mirror DB, SignalR ×3, webhooks, hourly full sync, path mappings, `Rescan*` round-trips | Subtitle rows FK the media file; triggers are in-process events | comparison.md §4 |
| Two subtitle indexes (Arr `SubtitleFiles` + Bazarr) | One `subtitle` table, shared with import | comparison.md §2.3 |
| Second media probe (pickled knowit cache) | The shared probe's stream dispositions | One probe per file |
| Per-provider guessit `get_matches()` | Host-side matching with the shared parser; plug-ins return facts | D1c; smaller plug-ins; consistent scores |
| `embeddedsubtitles` as a "provider" | Embedded tracks are indexed as present | Same outcome (S10) |
| Stored `missing_subtitles` strings | `Wanted()` computed on demand | No drift between profile and cache |
| Throttle dict in a file; in-memory job queue | DB-backed provider status (shared with indexers), durable jobs | HA-safe (D14) |
| Python providers in a forked subliminal | Sandboxed WASM plug-ins with a declared egress allowlist | D15; community long tail without recompiling |
| unrar / 7z binaries downloaded from GitHub at runtime | Pure-Go archive readers in the host | No native downloads |
| ffsubsync only | `Syncer` with alass (default) or ffsubsync; same quality checker | S12 outcome kept |
| Custom score sections, legacy Arr API paths, dead providers | Not carried | Already dead upstream |

---

## 6. Issues with decisions, and open questions

### 6.1 Issues with decisions

1. **D3a: the TVDB licence tier decides whether TV is free to use.** TVDB requires every key to have "either a commercial license, or have subscriptions enabled for end users" (MS E1). If the project key is registered in *user-supported* mode, every user needs a paid PIN and TV does not work without one. D3a works as intended only if the project obtains the free/OSS **licensed** tier (< $50k revenue, attribution link). *Action:* confirm the tier with TVDB before phase 1. Otherwise D3a needs a fallback (TMDb TV plus an episode-order mapping layer, which MS F2 calls a correctness risk).
2. **D3b's scope should cover non-metadata provider keys.** OpenSubtitles.com needs a per-application API key (Bazarr hard-codes one, `bazarr/bazarr/app/get_providers.py:252`), and AcoustID needs an application key. The proposal extends D3b as follows: project-registered, attributed, user-overridable keys for subtitle providers too, **held by the host and passed to plug-ins at configure time, never compiled into WASM modules**. It also clarifies that "per-user key" means an installation-level credential set by an admin (D17), not per request-user.
3. **D3/D3c: the TMDb 6-month limit reaches beyond "the cache".** Library rows copied from TMDb (titles, overviews, images, ratings) are cached TMDb data too. This design enforces a hard refresh deadline and purges TMDb-only descriptive fields when a refresh cannot happen (§A7). Whether **identifiers** (TMDb ids) must also be purged is unresolved; the design keeps them. This needs sign-off, because a revoked key would blank descriptions across the library after 180 days.
4. **D3c: the scene-mapping gap is real and has no seed.** `services.sonarr.tv` data has no stated licence and must not be scraped or copied. The community dataset starts nearly empty (XEM names + TVDB aliases + heuristics), so anime and scene-titled searches will regress compared with Sonarr until it matures. The Fribb and Anime-Lists datasets also have no verified licence. Recommendation, following D5b's logic: download all such datasets at runtime and never embed them. Add them to the D5b legal review.
5. **D14: two primitives are missing.** (a) **Capability-routed jobs:** sync and extraction need ffmpeg/alass/ffsubsync, which may not be on every node; D14's `SKIP LOCKED` queue has no "requires tool X" routing. (b) **A DB-backed shared rate limiter** for per-key or per-IP sources (MusicBrainz, ComicVine, Hardcover, Metron, AcoustID). Also, MusicBrainz's limit is per IP: nodes with different egress IPs could go faster, but a global bucket is the safe default.
6. **D15 + D5: licensing of third-party plug-ins.** WASM modules loaded in-process may be treated as a combined work under the GPL (the FSF plug-in FAQ). First-party plug-ins ported from Bazarr are GPL-3.0-derived and fine under AGPL-3.0. What the plug-in index may host under other licences needs the same legal review as D5b.
7. **D2 phasing: phase-2 features depend on phase-1 work.**
   - Import (phase 1) must already index subtitle sidecars into the shared `subtitle` table with lang/HI/forced.
   - The shared probe must record subtitle stream dispositions.
   - Requests (phase 1) needs TMDb discovery **and** the TMDb→TVDB resolution (§A15).
   
   None of these conflicts with D2, but phase-1 designs must include them to avoid the rescans and schema changes D2 wants to avoid.
8. **D3 wording, comics.** D3 names ComicVine only. This design treats Metron as an optional secondary source behind the same interface. It is not a deviation, but D3 could say so.
9. **Bazarr's API and history codes** are listed as "contracts" in bazarr.md §11. D7 drops external compatibility, so they are not kept. This is noted for completeness, not a conflict.

### 6.2 Open questions

1. **Episode order override.** Should a user be able to choose a TVDB alternate order (for example "absolute" or "DVD") per series? Sonarr only uses default order; D3a keys invariants to TVDB default. The proposal is not in phase 1.
2. **Image caching under TMDb terms.** Do cached poster files count as "TMDb data" for the 180-day rule? The proposal is to treat them as such (the media-cover cache re-fetches or evicts).
3. **Ratings.** Drop IMDb/RT ratings (no legitimate direct source), or offer IMDb's non-commercial datasets as an opt-in download? The proposal is to drop them in v1.
4. **Subtitle profile per version slot (D4).** Is one profile per item applied to all slots enough, or do 4K slots need their own? The proposal is per item, with a per-slot override later if asked for.
5. **Subtitles on media upgrade.** Is dropping downloaded subtitles of the replaced file (§B5) right, or should they be kept and re-synced when the release group matches? The proposal is to drop them and re-search, and to keep `uploaded`/`manual` ones re-synced.
6. **Default sync tool.** alass is proposed as default (single binary, fast), though Bazarr users are used to ffsubsync results. Is a comparison on a sample corpus needed before choosing?
7. **subzero OCR dictionaries** have no licence file in `custom_libs/subzero/`. Their provenance must be checked before porting the OCR_fixes mod.
8. **Change-feed availability to verify in a spike:**
   - TVDB `/updates` semantics;
   - the TMDb `/changes` 14-day window;
   - MusicBrainz WS/2 redirect behaviour for merged MBIDs;
   - StashDB `updated_at` filtering;
   - ComicVine `date_last_updated` filters;
   - Hardcover `updated_at` where-clauses.
   
   The refresh engine degrades to age-based policies where a feed is missing, so none of these is blocking.
9. **Hardcover OAuth client.** The project must register it (device grant), and the implications of Hardcover's "commercial products may not use user-owned data" clause (MS E7) should be recorded.
10. **Plug-in ABI encoding.** JSON (debuggable) is proposed for v1, with CBOR later under a new ABI version. The runtime design owns the final choice, since the ABI is shared across all D15 plug-in kinds.
