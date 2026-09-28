# 08. Metadata

This file specifies how bonnyread obtains, caches, refreshes and identifies metadata for every media kind. It covers the per-kind provider interface and Assemblers, the structure authority for each kind, the runtime-downloaded mapping datasets, the document cache and each source's terms, external-id states with merge and deletion handling, the refresh engine, the shared rate limiter and its lanes, key management, attribution, discovery for requests, and HA placement. It hands canonical documents to the library through one port. The library tables themselves (Entity, Item, ExternalID, AltTitle and so on) are owned by `02-domain-model.md`. Subtitle search is in `09-subtitles.md`.

Phases: **P1** for TV, movies, discovery, the shared machinery and the A17 groundwork. **P2** for music. **P3** for books, adult and comics.

---

## Concepts

| Term | Definition |
|---|---|
| **Source** | One external metadata service (TVDB, TMDb, TheXEM, MusicBrainz, Cover Art Archive, AcoustID, Hardcover, OpenLibrary, Audnexus/AudiobookDB, StashDB, ThePornDB, ComicVine, Metron). Each is a native Go client in the core. |
| **SourceInfo** | A source's self-description: attribution, rate specification, credential specifications, cache policy (hard max age from its terms, default soft TTL) and whether lookups of merged ids redirect. |
| **SourceClient** | The wrapper around every outbound call to one source. It enforces the SourceInfo: rate limits, quotas, cooldown, User-Agent, authentication and conditional requests. |
| **Assembler** | One per media kind (TV, Movie, Music, Book, Adult, Comic). It composes sources into a canonical document and enforces the kind's structure authority. |
| **Structure authority** | The single source that may create, delete or renumber an entity's children for a kind. |
| **Enrichment source** | A source that may only fill named field groups on a document whose structure comes from the authority. |
| **Canonical document (Doc)** | A normalised, source-independent struct (`SeriesDoc`, `MovieDoc`, `ArtistDoc`, `ReleaseGroupDoc`, `ReleaseDoc`, `AuthorDoc`, `WorkDoc`, `EditionDoc`, `SceneDoc`, `PerformerDoc`, `StudioDoc`, `VolumeDoc`, `IssueDoc`) with an external-id set, localised titles and aliases, status, dates, images, per-source ratings and typed children. |
| **Provenance** | The record of which source supplied each field group of a Doc, used for attribution and debugging. |
| **Document cache** | The persistent store of normalised per-source documents (`metadata_doc`), with a hard expiry derived from each source's terms. |
| **Terms deadline** | The earliest hard expiry among the cached documents and library copies a refresh root depends on. |
| **Refresh root** | A node refreshed as a unit: series, movie, artist, release group, author, comic volume, performer or studio. |
| **Change feed** | A source endpoint listing ids changed since a timestamp. |
| **Lane** | A traffic class with its own share of a source's rate budget: `interactive`, `add`, `change`, `import`, `refresh`. |
| **Project credential / user credential** | A key registered by the bonnyread project and shipped in the binary (D3b, D16), or one entered by an administrator. Both live in the credential store [OPN-201]. |
| **Mapping dataset** | A community dataset downloaded at runtime: scene-name mappings with the daily-series list, and anime id lists. TheXEM is a live source, not a dataset. |
| **Catalog** | The metadata-owned part of the domain model (`02-domain-model.md`). |

**Aliases.** The design document's `external_id` table is the domain model's **ExternalID** (`02-domain-model.md` DOM-027–DOM-029), with `state` `active`, `merged` (with `merged_into_value`) or `deleted`; `design/library.md`'s `status current|alias` means `active|merged`. When an id becomes `deleted`, its catalog node gets `catalog_state = tombstoned` (DOM-034); the design's "upstream deletion" flag is that tombstone. "Per-user key" (D3b) means a credential for the operator's own account, set by an administrator for the installation.

---

## Requirements

### Provider interface and Assemblers

**MET-001** (MUST) Metadata sources **MUST** be implemented as native Go clients in the core, not as WASM plug-ins. The D15 plug-in boundary does not apply to metadata sources. _Source: design/metadata-and-subtitles.md §A1._

**MET-002** (MUST) Every source **MUST** implement `Source{ID(), Info() SourceInfo}` and **MUST** implement only the capability interfaces it supports: `Searcher`, `Fetcher`, `BatchFetcher`, `ChangeFeed`, `IDResolver`, `Discoverer`, `Fingerprinter`. _Source: design/metadata-and-subtitles.md §A3._

**MET-003** (MUST) Sources **MUST** report failures with these typed errors, and the refresh engine **MUST** branch on them:

| Error | Meaning | Handling |
|---|---|---|
| `ErrNotFound` | The id does not exist upstream | Deletion path (MET-060) |
| `RedirectedError{To}` | The id was merged into another | Merge path (MET-056) |
| `ThrottledError{RetryAfter}` | 429/503 or `Retry-After` | Shared cooldown (MET-083) |
| `QuotaError{ResetAt}` | A window budget is exhausted | Requeue at `ResetAt` |
| `AuthError{Credential}` | The credential was rejected | Source degraded, health issue (MET-091) |

_Source: design/metadata-and-subtitles.md §A3._

**MET-004** (MUST) There **MUST** be exactly one Assembler per media kind: TV, Movie, Music, Book, Adult, Comic. An Assembler **MUST** expose `Assemble(root) → (Doc, Provenance, error)` and `Search(query) → []Candidate`. Search results **MUST** be merged across sources and de-duplicated by external id. _Source: design/metadata-and-subtitles.md §A3._

**MET-005** (MUST) Only the kind's structure authority **MAY** create, delete or renumber children in a Doc. An enrichment source **MUST** only fill the field groups the kind's precedence table allows it to fill. _Source: design/metadata-and-subtitles.md §A2, §A4._

**MET-006** (MUST) The field-group precedence per kind **MUST** be a table in code. Users **MAY** override only the image source preference and the title language. There **MUST NOT** be a general engine that merges any field from any source. _Source: design/metadata-and-subtitles.md §A4._

**MET-007** (MUST) The metadata service **MUST** hand results to the library only through `LibraryPort.Apply(ChangeSet)`, where a ChangeSet carries the root reference, the canonical Doc, confirmed merges (old reference → new reference, children included), confirmed removals and `fetched_at`. `Apply` **MUST** be transactional. The library diffs children by ExternalID and emits the resulting `CatalogDelta` events (`02-domain-model.md`). _Source: design/metadata-and-subtitles.md §A3; design/library.md §3.9._

**MET-008** (MUST) Every Doc **MUST** carry Provenance at field-group granularity. _Source: design/metadata-and-subtitles.md §A3._

**MET-009** (MUST) Interactive search **MUST** accept id prefixes (`tvdb:`, `tmdb:`, `imdb:`, `mbid:`, `stash:`, `tpdb:`, `hardcover:`, `isbn:`, `asin:`, `cv:`) and source URLs, and resolve them to a direct lookup instead of a text search. _Source: design/metadata-and-subtitles.md §1.1 M12._

**MET-010** (MUST) A hosted cache, if added later, **MUST** be one more `Source` implementation behind the same interfaces. v1 **MUST NOT** depend on any hosted service run by the project. _Source: D3c._

### Structure authority per kind

**MET-011** (MUST) Sources **MUST** play these roles:

| Kind | Phase | Structure authority | Enrichment | Discovery / search | Change feed |
|---|---|---|---|---|---|
| TV | P1 | **TVDB v4** (series, seasons, episodes in default order, absolute order, aliases, translations) | TMDb (images, rating, certification, cross-ids); TheXEM (scene numbering, scene names); scene-mapping dataset; anime-id dataset | TVDB search; TMDb discovery for requests (MET-100) | TVDB `/updates?since=`; TMDb `/tv/changes` for enrichment |
| Movie | P1 | **TMDb** | TMDb translations, collections, credits, IMDb id | TMDb | TMDb `/movie/changes` |
| Music | P2 | **MusicBrainz** (artist, release group, release, medium, track, recording) | Cover Art Archive (covers); AcoustID (fingerprint → recording ids, import only) | MusicBrainz | none (age-based) |
| Book | P3 | **Hardcover** | OpenLibrary (ISBN lookup, missing editions, keyless fallback); Audnexus/AudiobookDB (audiobook fields) | Hardcover, then OpenLibrary | Hardcover `updated_at` (to verify, MET-070) |
| Adult | P3 | **StashDB** (scenes, performers, studios) | ThePornDB; TMDb (full-length movies) | StashDB, ThePornDB | StashDB `updated_at` (to verify) |
| Comic | P3 | **ComicVine** | Metron (optional) | ComicVine | ComicVine `date_last_updated` (to verify) |

_Source: D3, D3a, D3d; design/metadata-and-subtitles.md §A2._

**MET-012** (MUST) The system **MUST** store one ExternalID per provider per known identifier, and **MUST NOT** model any entity with a single "foreign id". _Source: D3d; design/metadata-and-subtitles.md M9._

### TV

**MET-013** (MUST) TV episode identity and ordering **MUST** follow TVDB's default order. A TV Item's identity **MUST** be its TVDB episode ExternalID, never its numbers. TMDb **MUST NOT** decide TV structure. _Source: D3a; design/metadata-and-subtitles.md M1; design/library.md §3.3._

**MET-014** (MUST) The TV Assembler **MUST** take from TVDB: series identity, seasons, episodes in default order, absolute numbers (TVDB "absolute" season type), air dates converted to UTC from the TVDB air date and the network time zone, specials placement (`airsBefore*`/`airsAfter*`), aliases and translations. _Source: design/metadata-and-subtitles.md §A4._

**MET-015** (MUST) The TV Assembler **MUST** take from TMDb only posters, fanart, logos, TMDb rating, certification and `external_ids` (IMDb, and TVDB as a cross-check). The image preference **MUST** be a setting, defaulting to TMDb then TVDB. _Source: design/metadata-and-subtitles.md §A4._

**MET-016** (MUST) The TVDB client **MUST** authenticate with the project key and, when the administrator supplies one, the user's subscriber PIN (`/login {apikey, pin}`). The bearer token **MUST** be cached in the credential store and refreshed on 401. _Source: D3a; verification/metadata-services.md E1, F1._

**MET-017** (MUST) The TV Assembler **MUST** be written so that the structure-authority source is a single binding, and nothing outside the TV Assembler and the mapping layer assumes that TV structure comes from TVDB. This keeps the D20 fallback (MET-018) possible without a schema change. _Source: D20 (D3a risk)._

**MET-018** (MUST) The project **MUST** apply for TVDB's free/OSS licensed tier before phase 1 ships. If the application is refused, TV **MUST** fall back to **TMDb as the TV structure authority** with an **episode-order mapping layer** that produces TVDB-equivalent season, episode and absolute numbering wherever TheXEM, the scene-mapping dataset or anime numbering need it (sources: TMDb episode groups, TMDb `external_ids.tvdb_id`, XEM's TVDB origin). In that mode, TVDB with a user PIN **MAY** remain an optional structure authority per installation. The design of the mapping layer is an open item. [OPN-131] _Source: D20; open-issues B9; verification/metadata-services.md F2._

**MET-019** (MUST) Episode numbers for parsing and search **MUST** be used in this order: XEM scene numbering if present, else TVDB numbering. Anime series **MUST** also use absolute numbers, scene absolute first. This is Sonarr's `ParsingService` behaviour, kept as-is. _Source: design/metadata-and-subtitles.md §A12._

**MET-020** (MUST) Only TVDB's default (aired) order is supported in v1. Per-title alternate TVDB orderings (DVD, absolute, alternate) come after v1. [OPN-044] _Source: open-issues Group C (Library); design/library.md Q7._

**MET-021** (MUST) The series type `anime` **MUST** be suggested, never set silently, when TVDB genres include Anime, or when the original language is ja, zh or ko and a genre is Animation. The user confirms it at add time. The series type `daily` **MUST** come from the daily-series list (MET-030) or the daily heuristic (MET-031). The user's choice is stored on the Title and is never overwritten by refresh. _Source: design/metadata-and-subtitles.md §A12; design/library.md §3.9._

### TheXEM

**MET-022** (MUST) The system **MUST** call TheXEM (`https://thexem.info/map/`) directly, with `origin=tvdb`:
- `havemap?origin=tvdb` once a day, to learn which series have maps;
- `all?id=<tvdb>&origin=tvdb` on add and on each refresh of a mapped series;
- `allNames?origin=tvdb&seasonNumbers=1` once a day, for scene names.

_Source: design/metadata-and-subtitles.md §A12; verification/metadata-services.md C18, E14._

**MET-023** (MUST) XEM scene season, episode and absolute numbers **MUST** be stored per episode (`EpisodeInfo.scene_*`, `02-domain-model.md`). Episodes that XEM has not mapped yet **MUST** get extrapolated scene numbers flagged `unverified_scene = true`, following Sonarr's `XemService` extrapolation. Series without a map **MUST** fall back to TVDB numbering. _Source: design/metadata-and-subtitles.md M2._

**MET-024** (MUST) XEM scene names **MUST** be projected into `AltTitle(source=xem)` with their season scope. _Source: design/library.md §3.2 (AltTitle)._

### Mapping datasets (scene mappings, daily series, anime ids)

**MET-025** (MUST) Scene-name mappings **MUST** come from a **community dataset in its own git repository** (`scene-mappings`): JSON records in the shape of Sonarr's `SceneMapping` (`tvdb_id`, `title`, `parse_term`, `search_term`, `season`, `scene_season`, `filter_regex`, `search_mode`), plus a `daily` list of TVDB ids. _Source: D3c; design/metadata-and-subtitles.md §A12._

**MET-026** (MUST NOT) The system **MUST NOT** scrape, proxy or copy data from `services.sonarr.tv`. _Source: design/metadata-and-subtitles.md §6.1 item 4._

**MET-027** (MUST) Anime ids (AniDB, MyAnimeList, AniList, TMDb) **MUST** come from the Fribb `anime-lists` and Anime-Lists `anime-list.xml` datasets, loaded into `anime_id_map`. No AniDB HTTP API client is built. _Source: design/metadata-and-subtitles.md §A12._

**MET-028** (MUST) Every mapping dataset **MUST** be **downloaded at runtime**, pinned by git commit, verified by sha256, and stored in the database (A2), never embedded in the binary or shipped in release artefacts. The default refresh interval **MUST** be 6 h. _Source: D5b reasoning; A2; design/metadata-and-subtitles.md §A12._

**MET-029** (MUST) The mapping datasets (scene mappings, anime id lists) **MUST** be in scope of the D5b legal review before the first public release, including the fact that they end up in users' databases and backups. _Source: D20 (legal review scope); open-issues B12; A2._

**MET-030** (MUST) Scene mappings **MUST** be projected into `AltTitle(source=scene_mapping)` with `usage` from `search_mode` and `season_scope` from `season`. The daily list **MUST** set the suggested `series_type=daily`. _Source: design/library.md §3.2._

**MET-031** (SHOULD) Until the community dataset covers a series, the system **SHOULD** derive seed entries from XEM `allNames`, TVDB aliases, and a daily-type heuristic (TVDB seasons numbered by year with more than 100 episodes). Seeded entries **MUST** be marked with their origin. _Source: design/metadata-and-subtitles.md §A12._

**MET-032** (MUST) Users **MUST** be able to add local mappings, layered above the dataset. The UI **SHOULD** offer a one-click upstream contribution (a pull-request link), mirroring D9a's authoring model. _Source: design/metadata-and-subtitles.md §A12._

**MET-033** (MUST) A health issue **MUST** be raised when any mapping dataset is more than 7 days old. _Source: design/metadata-and-subtitles.md §A17._

### Movies

**MET-034** (MUST) TMDb **MUST** supply all movie structure and descriptive data. A movie collection **MUST** be a separate `Collection` Doc. _Source: D3; design/metadata-and-subtitles.md §A4._

### Music (P2)

**MET-035** (MUST) An album **MUST** be a MusicBrainz release group, with one selected release (the ItemSlot's target edition) and tracks keyed by recording id. _Source: design/metadata-and-subtitles.md M8; design/library.md §3.3._

**MET-036** (MUST) An artist Doc **MUST** be built from `artist/{mbid}?inc=aliases+url-rels+genres` plus release-group browse pages (`release-group?artist={mbid}&limit=100`). The metadata profile's primary types, secondary types and statuses **MUST** be pushed into the query where MusicBrainz supports the filter. Artist Docs **MUST** contain release-group summaries only. _Source: design/metadata-and-subtitles.md §A13._

**MET-037** (MUST) Release-group detail (releases, media, tracks, recordings) **MUST** be fetched lazily: when the album becomes monitored, when a search needs track counts, or when import needs candidates. It **MUST** use one `release?release-group={rg}&inc=media+recordings+artist-credits+labels` browse per release group, paged as needed. _Source: design/metadata-and-subtitles.md §A13._

**MET-038** (MUST) Metadata profiles **MUST** filter which members are admitted to the library, and the Music and Book Assemblers **MUST** also use them to skip fetches where the source supports filtering. Changing a profile **MUST NOT** delete catalog rows or files. [OPN-042] _Source: open-issues Group C (Library); design/library.md §3.9, Q5._

**MET-039** (MUST) Identification at import **MUST** prefer MBIDs already in file tags (one `release/{mbid}` lookup). Otherwise it uses `release?query=` plus candidate lookups. AcoustID **MUST** be used only for files that remain ambiguous, and its results **MUST** be applied to files at import, not stored as cached Docs. The distance matcher is owned by `06-import.md`. _Source: design/metadata-and-subtitles.md §A13, §A7._

**MET-040** (MUST) Cover Art Archive covers **MUST** be fetched on demand by the media-cover service, with a per-node limit of 5 requests/second. _Source: design/metadata-and-subtitles.md §A13; verification/metadata-services.md E6._

**MET-041** (MUST) The MusicBrainz base URL **MUST** be configurable, to point at a user-run mirror. When the base URL is not `musicbrainz.org`, the administrator **MUST** be able to set the rate, which replaces the 1 request/second default. _Source: design/metadata-and-subtitles.md §A9, §A13; verification/metadata-services.md F4._

**MET-042** (MUST) Every MusicBrainz request **MUST** carry the platform User-Agent `bonnyread/<version> (+<project URL>)` (`12-platform.md` PLT-122), which names the application and a contact URL as MusicBrainz requires. The same User-Agent **MUST** be used for every source. _Source: verification/metadata-services.md E4._

### Books (P3)

**MET-043** (MUST) Hardcover **MUST** be accessed only through the **OAuth device-authorization grant** with the project's registered client id. The system **MUST NOT** ask users to paste a personal access token. [OPN-054] _Source: D3d; verification/metadata-services.md E7, F5; open-issues Group C._

**MET-044** (MUST) The Hardcover client **MUST** batch up to 5 top-level queries per request and implement `BatchFetcher`. It **MUST** respect 60 requests/minute (burst 10) and 5,000/day. A bulk import that exceeds the daily budget **MUST** be spread across days by the planner, while OpenLibrary fills ISBN lookups so items appear with partial data. _Source: design/metadata-and-subtitles.md §A14; verification/metadata-services.md E7._

**MET-045** (MUST) OpenLibrary **MUST** be used keyless, at 1 request/second without an identifying User-Agent and at most 3 requests/second with one, and **MUST NOT** be used for bulk download. _Source: verification/metadata-services.md E8._

**MET-046** (MUST) Audiobook fields (narrators, ASIN, chapters) **MUST** come from Audnexus behind the provider interface, so that AudiobookDB, its announced successor, can replace it without changes outside the source client. _Source: D3d; verification/metadata-services.md E9._

**MET-047** (MUST) Every book **MUST** have ExternalIDs for each id it is known by: Hardcover, OpenLibrary work and edition, ISBN-10, ISBN-13, ASIN, and Goodreads (legacy, used only to match old tags). rreading-glasses **MUST NOT** be a default source. _Source: design/metadata-and-subtitles.md §A14; D3._

**MET-048** (MUST) Hardcover's clause that commercial products may not use user-owned data **MUST** be in scope of the D5b legal review. _Source: D20; open-issues B12._

### Adult (P3)

**MET-049** (MUST) StashDB **MUST** be the structure authority for scenes, performers and studios. ThePornDB and TMDb ids **MUST** be linked from the cross-references StashDB carries, or by the user. Full-length movies come from TMDb or ThePornDB. _Source: D10; design/metadata-and-subtitles.md §A4, §A14._

**MET-050** (MUST) Product codes **MUST** be stored as non-unique alternate identifiers (the `ProductCode` table with its non-unique index, `02-domain-model.md` DOM-031), not in the unique ExternalID map. _Source: A14; D10._

**MET-051** (MUST) Without StashDB or ThePornDB keys, the adult kind **MUST** remain usable for manually added TMDb movies, and the UI **MUST** explain how to obtain each key (including StashDB's rotating invite code). _Source: D10; verification/metadata-services.md E11, F6._

### Comics (P3)

**MET-052** (MUST) The ComicVine client **MUST** implement `BatchFetcher` with `volumes/?filter=id:a|b|…` (up to 100 ids per call), and **MUST** respect 200 requests per resource per hour, keyed per resource. Issue lists **MUST** be fetched only when `count_of_issues` changes or after 30 days. _Source: design/metadata-and-subtitles.md §A14; verification/metadata-services.md E12._

**MET-053** (MAY) Metron **MAY** be configured as an optional secondary source that fills missing issues, linked by the ComicVine id Metron stores. It **MUST** respect 20 requests/minute and 5,000/day per account. _Source: design/metadata-and-subtitles.md §A4, §6.1 item 8; verification/metadata-services.md E13._

### External ids, merges and deletions

**MET-054** (MUST) Every catalog node the metadata service creates **MUST** get one ExternalID row per identifier it is known by. Lookups by any known id (an indexer's `tvdbid`, a request's TMDb id, a media-server GUID, a file tag) **MUST** resolve through ExternalID in every state; a `merged` id resolves to the node it points at (`02-domain-model.md` DOM-029). _Source: design/metadata-and-subtitles.md §A6; design/library.md §3.2._

**MET-055** (MUST) For each (provider, namespace) at most one ExternalID per node **MUST** be `active` (DOM-028). A superseded id **MUST** be kept as `merged`, with `merged_into_value` set, on the surviving node, so that history rows, tags written into files and indexer results that carry it still resolve. _Source: design/library.md §3.2, O17._

**MET-056** (MUST) Merges **MUST** be detected deterministically:
- for sources with `Redirects = true` (MusicBrainz, StashDB), a lookup of id X that returns an entity with id Y ≠ X is a merge X → Y;
- for TVDB, a series fetch that returns a different series id is a merge;
- when an authority's child list no longer contains child X, the engine **MUST** look X up directly before concluding anything: a redirect is a merge, `ErrNotFound` is a deletion.

The engine **MUST NOT** infer merges from titles. _Source: design/metadata-and-subtitles.md §A6._

**MET-057** (MUST) Applying a merge X → Y when no local node has Y **MUST** re-key the node as specified in `02-domain-model.md` DOM-036 (1): ExternalID(X) becomes `merged` with `merged_into_value = Y`, and ExternalID(Y) is inserted as `active`. Files, monitoring, history and slots **MUST** be untouched. _Source: design/metadata-and-subtitles.md §A6._

**MET-058** (MUST) Applying a merge X → Y when a local node already has Y **MUST** fold X into Y in one transaction, following `02-domain-model.md` DOM-036 (2) and DOM-037: children, ExternalIDs, AltTitles, ItemLinks, ItemSlots, FileLinks, Intents, Follows and history move to Y; on a unique-key collision the older row survives with the union of monitoring and recomputed coverage [OPN-126]; X's node is deleted; X's id stays on Y as `merged` with `merged_into_value = Y`. The library **MUST** emit `EntityMerged{from, to}`. _Source: design/metadata-and-subtitles.md §A6; design/library.md §3.6._

**MET-059** (MUST NOT) A merge, deletion, re-link or refresh **MUST NOT** delete, move or rename a user's media file. Renames that follow renumbering or merges are proposed by `06-import.md`, never executed by the metadata service. _Source: design/metadata-and-subtitles.md M4, M5._

**MET-060** (MUST) `ErrNotFound` on a refresh root **MUST** tombstone the node as in `02-domain-model.md` DOM-034 (`catalog_state = tombstoned`, its ExternalIDs `state = deleted`), keep all of its rows, stop refreshing it, raise a health issue ("removed from <source>") and offer "re-link to another id" in the UI. _Source: design/metadata-and-subtitles.md §A6; design/library.md O19._

**MET-061** (MUST) A child confirmed deleted upstream **MUST** be removed only if it has no FileLink, Intent or Follow referencing it. A referenced child **MUST** be tombstoned, kept and surfaced in the health check. _Source: design/metadata-and-subtitles.md §A6; design/library.md §3.6._

**MET-062** (MUST) Re-linking a tombstoned node to another id **MUST** behave as a merge from the old id to the chosen id (MET-057/058). _Source: design/metadata-and-subtitles.md §A6._

### Document cache and source terms

**MET-063** (MUST) Normalised per-source documents **MUST** be stored in `metadata_doc` with `fetched_at`, `hard_expires_at = fetched_at + SourceInfo.Cache.HardMaxAge`, a content hash, and `etag`/`last_modified` where the source provides them. Raw HTTP bodies **MUST NOT** be stored. _Source: design/metadata-and-subtitles.md §A5._

**MET-064** (MUST) Per-source cache limits and defaults **MUST** be:

| Source | Hard max age (terms) | Default soft TTL |
|---|---|---|
| TMDb | **180 days** | policy-driven (MET-072) |
| TVDB | none found | policy-driven |
| MusicBrainz, Cover Art Archive | none (CC0 core data) | 30 d artist, 60 d release group |
| AcoustID | not cached | — |
| Hardcover, OpenLibrary, Audnexus/AudiobookDB | none found | 30 d |
| StashDB, ThePornDB | none found | change feed, else 30 d |
| ComicVine, Metron | none (ComicVine forbids redistributing altered data; a local cache is allowed) | 24 h volume, 30 d issues |
| TheXEM | unknown | `havemap` daily; per series on refresh |
| TMDb discovery pages | not persisted | 15 min lists, 6 h detail pages, per-node LRU |

_Source: design/metadata-and-subtitles.md §A7; verification/metadata-services.md E2, E12, E14._

**MET-065** (MUST) The TMDb 180-day limit **MUST** apply to `metadata_doc`, to TMDb-derived fields copied into library rows, and to TMDb images in the media-cover cache. Each refresh root's `terms_deadline` **MUST** be the minimum hard expiry over everything it depends on, and the planner **MUST** schedule a refresh no later than `terms_deadline − 7 days`. _Source: D20; open-issues B11; design/metadata-and-subtitles.md §A7._

**MET-066** (MUST) If TMDb data cannot be refreshed before its deadline (TMDb unreachable, key revoked, source disabled), a daily terms sweep **MUST** **purge the TMDb descriptions and the cached TMDb images** for the affected rows, and **MUST** **keep TMDb ids** and all user state. The purge covers overview, tagline, TMDb ratings, certification text and every cached TMDb image file. A health issue **MUST** report the purge with a count. _Source: D20 ("TMDb 180-day rule"); open-issues B11._

**MET-067** (MUST) The purge in MET-066 **MUST** keep the fields the library needs to operate: titles, original title, year and release dates. Whether these count as purgeable TMDb data is an open item; the default keeps them, like ids. [OPN-132] _Source: D20; design/metadata-and-subtitles.md §6.1 item 3._

**MET-068** (MUST) Where a source returns `ETag` or `Last-Modified`, the SourceClient **MUST** send conditional requests. A 304 **MUST** extend `fetched_at` without re-parsing. A 304 **MUST** reset the TMDb 180-day clock only if TMDb itself answered. _Source: design/metadata-and-subtitles.md §A7._

**MET-069** (MUST) Library entities **MAY** carry denormalised display fields projected from Docs. Those copies **MUST** be covered by MET-065 and MET-066. _Source: design/metadata-and-subtitles.md §A5._

### Refresh

**MET-070** (MUST) Refresh **MUST** be change-driven with adaptive age ceilings. For every source with a `ChangeFeed` and an active credential, the planner **MUST** poll `Changes(kind, cursor − 1 h)`, join the ids against current ExternalIDs, mark matching refresh roots due now with priority `change`, and advance the cursor. Where a feed is unavailable, the kind **MUST** fall back to its age-based policy. The change-feed semantics for TVDB `/updates`, the TMDb 14-day `/changes` window, MusicBrainz redirects, StashDB, ComicVine and Hardcover **MUST** be confirmed by a non-blocking spike. [OPN-053] _Source: design/metadata-and-subtitles.md §A8, §6.2 item 8; open-issues Group C._

**MET-071** (MUST) If the TMDb cursor is older than 14 days, the planner **MUST NOT** page `/changes`. It **MUST** instead mark every TMDb-dependent root due, spread evenly over the next 24 h. _Source: design/metadata-and-subtitles.md §A8._

**MET-072** (MUST) Each refresh root **MUST** have one of these policies:

| Policy | Rule |
|---|---|
| `tv_continuing` | change feed; else every 24 h; 6 h floor; refresh 6 h after an aired episode is still titled "TBA"/"TBD" |
| `tv_ended` | change feed; else 30 d; 7 d if the last episode aired less than 30 d ago |
| `movie` | change feed; else 30 d if any release date is within ±30 d, otherwise 90 d |
| `music_artist` | 30 d ceiling, 12 h floor; 7 d if a release group is dated within ±30 d |
| `music_release_group` | only if monitored or needed for identification; 60 d, or 7 d if released less than 30 d ago |
| `book_author` | 30 d; 2 d if continuing with a book less than 30 d old; 12 h floor |
| `comic_volume` | 24 h; issues only when `count_of_issues` changes or after 30 d |
| `adult_followed` | change feed; else 7 d for followed performers and studios, 30 d otherwise |

Every policy **MUST** be capped by the terms deadline (MET-065). _Source: design/metadata-and-subtitles.md §A8 (derived from Sonarr, Radarr, Lidarr, Bookshelf, Kapowarr, Whisparr v3)._

**MET-073** (MUST) `next_due_at` **MUST** be computed from the policy with ±10 % jitter, and capped at `terms_deadline − 7 d`. _Source: design/metadata-and-subtitles.md §A8._

**MET-074** (MUST) The planner **MUST** enqueue no more refresh jobs per source than `forecast_budget(source, next 5 min)` can serve, ordered by priority then `next_due_at`. Refresh jobs **MUST** use the dedupe key `metadata:refresh:<root_ref>`. _Source: design/metadata-and-subtitles.md §A8._

**MET-075** (MUST) A refresh worker **MUST** assemble the Doc, compare its hash with the stored hash, and on equality only touch `fetched_at`, with no library writes. On inequality it **MUST** call `LibraryPort.Apply`. _Source: design/metadata-and-subtitles.md §A8._

**MET-076** (MUST NOT) Metadata refresh **MUST NOT** change any existing `ItemSlot.monitored` flag or any user override (`Title.series_type`, `Title.overrides`, `AltTitle(source=user)`). _Source: design/library.md Invariant M, O7, §3.9._

**MET-077** (MUST) Refresh **MUST** also be triggered by: Title creation (stub → complete), Follow creation, fulfilment of an Intent with an unknown reference, and manual refresh. Manual refresh uses the `interactive` lane. _Source: design/library.md §3.9._

### Rate limiting and lanes

**MET-078** (MUST) Every outbound metadata call **MUST** go through the source's SourceClient, which **MUST** enforce: the token bucket, window budgets, the shared cooldown, the lanes, the User-Agent (MET-042), gzip, HTTP/2 where offered, and request coalescing (single-flight per reference per node). _Source: design/metadata-and-subtitles.md §A9._

**MET-079** (MUST) Sources with a global limit (MusicBrainz 1/s, AcoustID 3/s, ComicVine, Metron, Hardcover) **MUST** use the **DB-backed shared rate limiter** defined in `12-platform.md` ("next allowed at" tokens plus quota buckets), so one budget covers all nodes. _Source: A3; D20 (D14 amended)._

**MET-080** (MUST) Sources whose limit is per IP (TMDb about 40/s, TVDB, Cover Art Archive, OpenLibrary) **MAY** use an in-memory bucket per node. When all nodes share one egress IP, the administrator **MUST** be able to set the per-node rate to the limit divided by the node count. _Source: design/metadata-and-subtitles.md §A9, §A16._

**MET-081** (MUST) Window budgets **MUST** be persisted in the shared limiter: ComicVine 200 per resource per hour (key `comicvine:<resource>`), Hardcover 60/minute and 5,000/day, Metron 20/minute and 5,000/day. _Source: design/metadata-and-subtitles.md §A9; verification/metadata-services.md E7, E12, E13._

**MET-082** (MUST) Each source's budget **MUST** be divided into lanes: `interactive` (user search, lookup and add; waits synchronously), `add` (populating a newly added item), `change`, `import` (bulk library import) and `refresh`. For sources with a strict budget, 25 % **MUST** be reserved for `interactive`. Background lanes **MUST NOT** starve `interactive`. _Source: A3; design/metadata-and-subtitles.md §A9._

**MET-083** (MUST) A 429 or 503 response, or a `Retry-After` header, **MUST** set a cooldown for that source that every node observes. _Source: design/metadata-and-subtitles.md §A9._

**MET-084** (MUST) Rate specifications **MUST** be per credential where the source limits per key (ComicVine, Metron, Hardcover), and per source otherwise. _Source: design/metadata-and-subtitles.md §A5 (source_state)._

### Keys and credentials

**MET-085** (MUST) Credential resolution per source **MUST** be: user credential, then project credential, then none (the source is disabled or degraded). _Source: D3b; design/metadata-and-subtitles.md §A10._

**MET-086** (MUST) Project keys (TVDB, TMDb, AcoustID, the Hardcover OAuth client id, and the subtitle-provider application keys of `09-subtitles.md`) **MUST** be registered by the project, injected at build time into a `projectkeys` package, clearly attributed, and shown in settings as "project key (override)". Every project key **MUST** be user-overridable. _Source: D3b; D16; design/metadata-and-subtitles.md §A10, §6.1 item 2._

**MET-087** (MUST NOT) The binary **MUST NOT** contain keys copied from other applications (Seerr's TMDb, TVDB and Algolia keys; Radarr's and Whisparr v3's TMDb bearer; Lidarr's AcoustID, Last.fm and Headphones keys; Bazarr's OpenSubtitles key and GA4 secret). _Source: D3b; verification/metadata-services.md F9._

**MET-088** (MUST) Keys **MUST** be held by the host. WASM plug-ins **MUST NOT** contain keys; a plug-in receives the credentials it declared from the host at configure time (`09-subtitles.md`, `12-platform.md`). _Source: D3b extension; design/metadata-and-subtitles.md §6.1 item 2._

**MET-089** (MUST) StashDB, ThePornDB, ComicVine and Metron **MUST** take user keys only. Keys are installation-scoped and set by an administrator (`10-identity-and-access.md` roles), not per request user. _Source: D3b; design/metadata-and-subtitles.md §A10._

**MET-090** (MUST) The Hardcover refresh token **MUST** be stored in the credential store. Hardcover's yearly key expiry (1 January) and unannounced token resets **MUST** surface as a health issue with a "re-authorise" action. _Source: verification/metadata-services.md E7; design/metadata-and-subtitles.md §A10._

**MET-091** (MUST) Every credential **MUST** have a `Test()` action and a periodic validation job. An `AuthError` **MUST** mark the source degraded and raise a health issue, and **MUST NOT** cause a retry loop. _Source: design/metadata-and-subtitles.md §A10._

**MET-092** (MUST) Credentials are stored in the database (D16). The use of AcoustID's non-commercial application key under AGPL **MUST** be in scope of the D5b legal review. _Source: D16; D20; open-issues B12._

### Attribution

**MET-093** (MUST) `SourceInfo` **MUST** include `Attribution{Name, URL, LogoAsset, Notice, Placement}`. _Source: design/metadata-and-subtitles.md §A11._

**MET-094** (MUST) The UI **MUST** have an "About data sources" page showing the TMDb logo and the notice "This product uses the TMDB API but is not endorsed or certified by TMDB", plus attribution links for TVDB, MusicBrainz, ComicVine (link-back), StashDB and ThePornDB, and TVMaze if it is ever used. _Source: verification/metadata-services.md E1–E3, E12, F3._

**MET-095** (MUST) Item pages **MUST** show a provenance footer naming the sources of their data (for example "Episode data: TheTVDB · Images: TMDB"). _Source: design/metadata-and-subtitles.md §A11._

**MET-096** (MUST) The API **MUST** expose `GET /api/v1/metadata/sources` with each enabled source's attribution, key origin (project or user), health and quota use. The endpoint shape is in `11-api-and-clients.md`. _Source: design/metadata-and-subtitles.md §A11._

### Ratings

**MET-097** (MUST) Ratings **MUST** come only from the metadata sources in MET-011 (TMDb for movies and TV; others where the source provides them). IMDb and Rotten Tomatoes ratings are dropped in v1. [OPN-048] _Source: open-issues Group C; design/metadata-and-subtitles.md §6.2 item 3._

### Health and observability

**MET-098** (MUST) Health issues **MUST** be raised for: a degraded source (auth failure, quota exhausted, cooldown longer than 1 h); a missing user key for an enabled kind; mapping datasets older than 7 days; TMDb terms purges; catalog nodes tombstoned upstream; unresolved TMDb → TVDB mappings; a MusicBrainz import backlog with an ETA over 24 h (informational, with a "use a mirror" hint). _Source: design/metadata-and-subtitles.md §A17._

**MET-099** (MUST) Metrics **MUST** include, per source: calls, 429 responses, budget used and remaining per window, queue depth per lane, document-cache hit rate and refresh lag. _Source: D7; design/metadata-and-subtitles.md §A17._

### Discovery for requests

**MET-100** (MUST) In phase 1, TMDb **MUST** implement `Discoverer`: trending, popular, upcoming, `/discover/{movie,tv}` with filters, recommendations and similar titles, person and collection pages, and keyword, company and network lookups. This is A17 phase-1 groundwork. _Source: A17; design/metadata-and-subtitles.md §A15, M11._

**MET-101** (MUST) Discovery results **MUST** be cached per node in memory (15 min for lists, 6 h for detail pages) and **MUST NOT** be persisted. _Source: design/metadata-and-subtitles.md §A15._

**MET-102** (MUST) Discovery results **MUST** carry TMDb ids, and library status per slot **MUST** be computed by joining them against ExternalID (provider `tmdb`), as defined by the availability query in `02-domain-model.md`. There **MUST NOT** be availability scanners. _Source: design/metadata-and-subtitles.md §A15; design/library.md §3.8._

**MET-103** (MUST) A TV request carries a TMDb TV id. On request, the TV Assembler **MUST** resolve it to TVDB through `IDResolver`, in this order: TMDb `external_ids.tvdb_id`; a TVDB remote-id search by IMDb id; a manual pick. The library node **MUST** always be keyed by TVDB (MET-013). An unresolved mapping **MUST** block auto-approval and ask an administrator to pick. This resolution is A17 phase-1 groundwork. _Source: A17; design/metadata-and-subtitles.md §A15._

### HA placement

**MET-104** (MUST) Metadata work **MUST** be placed as follows. No metadata component uses a lease or leader election (A1).

| Component | Placement |
|---|---|
| Planner tick (change feeds, due selection), terms sweep, dataset refresh, XEM `havemap`/`allNames` | Schedule rows claimed by any node with `SKIP LOCKED` (`12-platform.md`), dedupe policy `WhilePendingOrRunning` so at most one runs at a time. Planner tick every 5 min; terms sweep daily; datasets every 6 h |
| Refresh, add and import fetch jobs | Durable jobs on the `metadata` queue, any node |
| Interactive search, lookup, discovery | Any API node, directly through the SourceClient on the `interactive` lane |
| Global buckets, window budgets, cooldowns | Shared DB-backed rate limiter (MET-079) |
| Per-IP buckets (TMDb, TVDB, Cover Art Archive, OpenLibrary) | In memory per node (MET-080) |
| Discovery LRU, single-flight | Per node; short TTLs, no cross-node invalidation |
| `metadata_doc`, ExternalIDs, datasets, source state | Database |

_Source: A1, A2, A3; D20 (D14 amended); design/metadata-and-subtitles.md §A16._

**MET-105** (MUST) On single-node SQLite deployments the same components **MUST** run, with the shared limiter and the lanes behaving identically. _Source: D14; D20 (D13 amended)._

---

## Data model

Shared entities (Entity, Item, EpisodeInfo, SceneInfo, ExternalID, AltTitle, ProductCode, Title) are owned by `02-domain-model.md`. The owner of the credential store for source credentials is an open item [OPN-201]. This file defines only the metadata-owned tables.

```go
// Persistent cache of normalised per-source documents (MET-063).
MetadataDoc {
  source        enum(tvdb, tmdb, thexem, musicbrainz, caa, hardcover, openlibrary, audnexus,
                     stashdb, tpdb, comicvine, metron)
  kind          enum(series, episode, movie, collection, person, artist, release_group, release,
                     recording, work, edition, author, scene, performer, studio, volume, issue)
  source_id     string
  doc           json          // normalised, never raw HTTP
  doc_hash      bytes
  fetched_at, hard_expires_at time          // hard_expires_at null when the source has no limit
  etag, last_modified string?
  status        enum(ok, not_found, redirected)
  redirect_source, redirect_kind, redirect_id string?
  unique (source, kind, source_id)
  index (source, hard_expires_at)
}

// One row per refresh root (MET-072).
MetadataRefresh {
  root_source, root_kind, root_id string
  edges: exactly one of entity -> Entity | item -> Item
  policy        enum(tv_continuing, tv_ended, movie, music_artist, music_release_group,
                     book_author, comic_volume, adult_followed)
  priority      enum(interactive, change, due, background)
  next_due_at, terms_deadline time?
  last_success_at, last_attempt_at time?
  failures      int
  unique (root_source, root_kind, root_id)
  index (next_due_at, priority); (terms_deadline)
}

// Per source (and per credential where limits are per key). Buckets and cooldowns live in the
// shared rate limiter of 12-platform.md; this row holds metadata-specific state only.
MetadataSourceState {
  source        string
  credential_id int?
  change_cursor time?          // last change-feed timestamp polled (MET-070)
  health        enum(ok, degraded, disabled)
  degraded_reason string?
  unique (source, credential_id)
}

// Runtime-downloaded datasets (MET-028).
MappingDataset {
  name          enum(scene_mappings, anime_lists_fribb, anime_list_xml)
  repo_url      string
  commit        string
  sha256        bytes
  fetched_at    time
  unique (name)
}
SceneMapping {
  tvdb_id int; title, parse_term, search_term string
  season, scene_season int?; filter_regex string?; search_mode enum(title, search_term, both)?
  origin enum(dataset, seed_xem, seed_tvdb_alias, seed_heuristic, user)
  dataset -> MappingDataset?
  index (tvdb_id); (parse_term)
}
XemEpisode {
  tvdb_series_id, tvdb_season, tvdb_episode int; tvdb_absolute int?
  scene_season, scene_episode, scene_absolute int?
  fetched_at time
  unique (tvdb_series_id, tvdb_season, tvdb_episode)
}
AnimeIDMap {
  tvdb_id int; tvdb_season int?; episode_offset int?
  anidb_id, mal_id, anilist_id, tmdb_id int?
  dataset -> MappingDataset
  index (tvdb_id); (anidb_id); (mal_id); (anilist_id)
}
```

Projections into shared entities: `XemEpisode` → `EpisodeInfo.scene_*` and `unverified_scene`; `SceneMapping` and XEM names → `AltTitle(source = scene_mapping | xem)`; `AnimeIDMap` → ExternalIDs for `anidb`, `mal`, `anilist`.

---

## Flows

### F1. Adding a series (TVDB + TMDb + XEM)

1. Interactive search (any API node, `interactive` lane): TVDB `/search?query=…&type=series`. Candidates with a TMDb id get the cached or fetched TMDb poster; otherwise the TVDB image. Attribution for both is shown.
2. Add: the TV Assembler fetches TVDB `/series/{id}/extended?meta=episodes&short=true` and the absolute-order list; TMDb `/tv/{tmdbId}?append_to_response=images,external_ids,content_ratings` (with `hard_expires_at = now + 180 d`); XEM `all?id=<tvdb>&origin=tvdb` if `havemap` lists the series; `anime_id_map` and `scene_mapping` rows.
3. The user confirms the suggested series type (MET-021).
4. The Assembler builds `SeriesDoc` with Provenance: TVDB (structure), TMDb (images, rating), XEM (scene numbers, extrapolated rows `unverified`).
5. `LibraryPort.Apply` creates the nodes, ExternalIDs (tvdb, tmdb, imdb, anidb, mal, anilist) and a `MetadataRefresh` row with policy `tv_continuing`, capped by the TMDb terms deadline.
6. Failure paths: TVDB `AuthError` → the add fails with a clear message and a health issue; TMDb unavailable → the add completes with TVDB images, and TMDb enrichment is retried on the next refresh; XEM unreachable → TVDB numbering is used and the XEM fetch is retried with the next refresh.

### F2. Change-driven refresh

1. The planner tick (schedule row, any node) polls each change feed with a 1 h overlap, marks matching roots due with priority `change`, and advances the cursor.
2. It selects due roots up to the forecast budget and enqueues refresh jobs (dedupe `metadata:refresh:<root>`).
3. A worker assembles the Doc. Outcomes: hash unchanged → touch `fetched_at`; changed → `Apply`; `ErrNotFound` → MET-060; `RedirectedError` → MET-056/058; `ThrottledError`/`QuotaError` → job snoozed until the reset without consuming an attempt; `AuthError` → source degraded (MET-091).
4. `next_due_at` is set from the policy with jitter, capped at `terms_deadline − 7 d`.

### F3. TMDb terms purge

1. The daily terms sweep selects TMDb-dependent rows with `terms_deadline < now`.
2. For each, it clears overview, tagline, TMDb ratings and certification text, evicts cached TMDb images, and keeps ids, titles, years, release dates and user state (MET-066, MET-067).
3. It raises one health issue with the count and the cause (source disabled, key revoked, unreachable).
4. When TMDb becomes reachable again, the next refresh restores the purged fields.

### F4. Merge of a MusicBrainz release group (RG1 → RG2)

1. The artist refresh finds RG1 missing from the browse.
2. A direct lookup of RG1 returns RG2: `RedirectedError{To: RG2}`.
3. The ChangeSet carries `Merged: [RG1 → RG2]`.
4. RG2 exists locally, so `Apply` folds RG1 into RG2 in one transaction (MET-058). Files are untouched; `EntityMerged` lets import re-tag files if tag writing is on.
5. Failure path: the lookup returns 404, so RG1 is tombstoned if it has files (MET-061), or removed if it has none.

### F5. TV request from discovery

1. A user browses TMDb discovery (per-node LRU; status per slot from the ExternalID join).
2. The request carries a TMDb TV id. The TV Assembler resolves it to TVDB (MET-103).
3. Resolved: the request continues under the approval rules of `10-identity-and-access.md` and `02-domain-model.md` (Intent). Unresolved: auto-approval is blocked and an administrator picks the TVDB series.

### F6. First import of a large music library (P2)

1. Import (`06-import.md`) reads tags with no network calls.
2. Tagged albums cost one `release/{mbid}` call each; untagged albums cost a search plus about 1.5 candidate lookups; ambiguous files go to AcoustID on its own budget.
3. Artists cost one lookup plus release-group summary pages; release detail for unowned albums is not fetched.
4. All calls use the `import` lane; 25 % of the MusicBrainz budget stays reserved for `interactive`.
5. An ETA metric is exposed; above 24 h a health hint suggests a mirror. A 503 sets the shared cooldown for every node.

---

## Non-goals

| Excluded | Reason |
|---|---|
| Hosted metadata proxies (SkyHook, RadarrAPI, LidarrAPI, Whisparr API, bookinfo.pro) and any project-run cache in v1 | D3c |
| `services.sonarr.tv` scene mappings and daily list | No licence; replaced by the community dataset (MET-025, MET-026) |
| IMDb and Rotten Tomatoes ratings | No legitimate direct source; open item (MET-097) |
| Full-tree refetch on every refresh | Replaced by change feeds, lazy release-group detail and hash compare (D19) |
| A general any-field merge engine | Would make structure authority unenforceable (MET-006) |
| Seerr's own TMDb client, `Media` table, availability scanners and 4K columns | Availability is a query; slots replace 4K duplication (D4) |
| AniDB HTTP API client | Anime ids come from datasets (MET-027) |
| rreading-glasses as a default source; Goodreads as a live source | Relies on unofficial access to a retired API (verification/metadata-services.md E16, E17) |
| Metadata sources as WASM plug-ins | Few, first-party, on the data model's critical path (MET-001) |
| Per-title alternate TVDB orderings in v1 | Open item (MET-020) |
| Telemetry | Absent (D3) |

---

## Traceability

| Reference | Preserved | Changed (D19) |
|---|---|---|
| Sonarr | TVDB default order as identity; XEM scene numbering with extrapolation and `unverified`; scene mappings and daily list as parse/search inputs; parse order scene → TVDB → absolute; adaptive refresh (`ShouldRefreshSeries`); deletion never removes files; TVDB id replacement followed | SkyHook replaced by direct TVDB; `services.sonarr.tv` replaced by a community dataset; change feed instead of a 12 h sweep |
| Radarr | TMDb as movie authority; `changed?since` delta refresh; 180 d ceiling; deleted-upstream status | Direct TMDb instead of `api.radarr.video`; no hard-coded bearer |
| Lidarr | Album = release group; one selected release; recordings as track identity; metadata profiles; MBID merge handling (`Old*Ids`) | Direct MusicBrainz with lazy release detail; metadata profiles also filter fetches; one ExternalID table instead of per-table `Old*Ids` |
| Bookshelf | Per-provider id map (fixes the Goodreads/Hardcover split) | Hardcover via OAuth device grant, OpenLibrary fallback, Audnexus/AudiobookDB |
| Whisparr v3 | Multi-provider ids; StashDB authority; performer merges | Direct StashDB/ThePornDB/TMDb with user keys; product codes restored as non-unique ids (D10) |
| Kapowarr | ComicVine per-user key; batched volume fetch; issue refresh only on count change | Shared DB-backed budget across nodes; optional Metron |
| Seerr | TMDb discovery endpoint set; TMDb → TVDB resolution for TV requests | Availability as an ExternalID join; per-node LRU; no borrowed IMDb/RT ratings |
| Bazarr | Anime id datasets | One `anime_id_map` shared by all consumers |
