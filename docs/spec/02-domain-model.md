# 02. Domain model

This file defines the shared domain entities of bonnyread: the metadata-owned **catalog**, the user-owned **library**, the media kinds and their hierarchies, external identifiers, version slots, media files and their links to items, intents, follows, monitoring, the wanted and availability queries, import lists and exclusions, root folders, tags, media-server connections, the shared subtitle record, and the per-kind extension points. Other spec files reference these entities by name and requirement ID and define only their own tables. Users, roles and credentials are owned by `10-identity-and-access.md`; downloads, download targets and the blocklist by `05-downloads.md`; quality profiles by `07-profiles.md`; naming schemes, probes and extra files by `06-import.md`; metadata documents and refresh state by `08-metadata.md`; subtitle profiles, subtitle search state and the subtitle blocklist by `09-subtitles.md` (the shared `Subtitle` record itself is defined here, DOM-110).

Phases: the schema applies from **P1** for every media kind (D2, A17). TV and movies are populated in P1, music in P2, books, adult and comics in P3.

---

## 1. Concepts

| Term | Definition |
|---|---|
| **Catalog** | Metadata-owned, refreshable rows describing everything the metadata sources reported, whether or not the user wants it. It holds no user decisions. Tables: `Entity`, `Item`, kind-info tables, `Edition`, `Track`, `ItemLink`, `EntityLink`, `ItemRelation`, `ExternalID`, `AltTitle`, `ProductCode`. |
| **Library** | User-owned rows: what the user keeps, where, in which versions, and why. Tables: `Title`, `TitleSlot`, `ItemSlot`, `SlotDefinition`, `Follow`, `Intent`, `ImportList`, `Exclusion`, `RootFolder`, `MetadataProfile`, `MediaFile`, `FileLink`, `Tag`, `TitleTag`, `AutoTagRule`. |
| **Media kind** | One of `tv`, `movie`, `music`, `book`, `adult`, `comic`. |
| **Entity** | A catalog node that groups or credits items: series, season, artist, author, comic volume, studio, network, performer, person, movie collection, book series, label, publisher. |
| **Item** | The **unit of want**: the smallest thing a user monitors and acquisition targets. Kinds: `episode`, `movie`, `album`, `book`, `scene`, `adult_movie`, `issue`. |
| **Edition** | A variant of an item: a music release (`album_release`) or a book edition (`book_edition`). Defines what "complete" means; not a unit of want. |
| **Track** | A component of a music release. |
| **Title** | The library root the user added. It owns folders and points at an entity (series, artist, author, comic volume) or an item (movie, scene, adult movie). |
| **SlotDefinition** | A named version kind, defined globally per media kind (A13), for example movie {`HD`, `UHD`}, book {`Ebook`, `Audiobook`}. Alias: "Slot" in `design/library.md`, "version slot" in D4. |
| **TitleSlot** | Per (title, slot definition) configuration: root folder, folder path, quality profile, naming scheme, pause flag (A13). |
| **ItemSlot** | Per (item, slot definition) state: monitored flag, coverage, target edition (A13). The **only** stored monitoring state. |
| **MediaFile** | A media file under a root folder, stored as root id plus root-relative path (A12). Belongs to exactly one slot. |
| **FileLink** | The M:N relation "this file contains this item (and this track, part or issue range)". |
| **Coverage** | Per ItemSlot: `none`, `partial` or `complete`, derived from present files' FileLinks by the kind's coverage rule. |
| **Coverage mode** | `exclusive` (at most one present covering file per unit per slot: TV, movie, music, book, adult) or `overlapping` (several files may cover the same item: comics). |
| **Intent** | A recorded reason for something to be in the library. Origins: `manual`, `user_request`, `import_list`, `watchlist`, `follow`, `disk_import`. Aliases: "request" = `user_request`, "list" = `import_list`, "disk" = `disk_import`. A request (Seerr sense) is an Intent with an approval workflow. |
| **Follow** | A durable policy "keep the members of entity E in the library for slot S" (D10, generalised): which new members to admit, which to monitor, date gates, defaults. |
| **Exclusion** | "Never add this automatically" and/or "never let users request it", keyed by external id or catalog node. |
| **Released** | An item has reached the kind's release condition; the precondition for "missing". Distinct from availability. |
| **Availability** | Whether the library holds files for a target in a slot (`none`, `partial`, `caught_up`, `complete`). Always a query. |
| **Wanted** | The umbrella for the `missing` and `cutoff_unmet` queries. |
| **Layout root** | The node a Title is created for: the item itself for movie, scene and adult movie; otherwise the nearest ancestor entity of the title's kind (episode → series, album → artist, book → primary author, issue → comic volume). |
| **Stub** | A catalog node known by id and name only, still being fetched. |
| **Tombstone** | A catalog node its structure authority reported deleted, kept while referenced (O19). |

Term aliases kept once: `library.md` `ExternalID.status current|alias` is written here as `state active|merged|deleted` (following `metadata-and-subtitles.md` §A5–A6; `08-metadata.md` uses the same states); `import.md` `RootID/RelPath` is `root_folder_id/rel_path`.

---

## 2. Requirements

### 2.1 Schema conventions

**DOM-001** (MUST) Every domain table **MUST** have an `id` (int64, auto-increment), `created_at` and `updated_at` (UTC). Rows owned by other files **MAY** use other id types (for example `Download.id` is a UUID in `05-downloads.md`). _Source: design/library.md §3.2._

**DOM-002** (MUST) Enumerated fields **MUST** be Go-typed enums stored as strings, so that adding a value is an additive change. Removing or renaming a value **MUST** follow the expand/contract migration rule (A7). _Source: A7; D2._

**DOM-003** (MUST) "Exactly one of" edges **MUST** be enforced both by a CHECK constraint (`entsql.Annotation{Checks: …}`) and in the service layer. _Source: design/library.md §3.2._

**DOM-004** (MAY) Because MySQL/MariaDB is dropped (D20), uniqueness **MAY** use partial unique indexes (`entsql.IndexWhere`), which PostgreSQL ≥ 15 and SQLite both support. Where a NULL-able column takes part in uniqueness, the schema **MUST** use either a partial index or a NOT NULL surrogate key, never rely on NULL-distinct semantics. _Source: D20 (D13 amended); supersedes design/library.md §3.2 "no partial unique indexes"._

**DOM-005** (MUST) Every write that changes coverage, monitoring or catalog membership **MUST** run in one database transaction together with the outbox events it causes. Events that must be delivered **MUST** go through the transactional job queue; NOTIFY is only a doorbell (A4). _Source: A4; design/library.md §3.8._

**DOM-006** (MUST) External provider names (`ExternalID.provider`, `Exclusion.provider`) **MUST** be strings validated against a compiled-in provider registry rather than a closed database enum, so that adding a provider needs no migration. _Source: D2; design/library.md §2.2 #6._

### 2.2 Catalog and library split

**DOM-007** (MUST) Catalog tables **MUST** hold only data supplied by metadata sources or derived from them. They **MUST NOT** hold user decisions (monitoring, profiles, folders, overrides, tags). _Source: design/library.md §3.1, §2.2 #2._

**DOM-008** (MUST) User overrides of catalog facts (series type, scene numbering use, comic special version, user search titles) **MUST** live in library rows (`Title` fields, `AltTitle(source=user)`), so that a metadata refresh never overwrites them. _Source: design/library.md §3.9 (4)._

**DOM-009** (MUST) The catalog **MAY** contain nodes that no Title references (collection members, performer credits, an artist's unmonitored albums, discovery stubs). Their presence **MUST NOT** imply any library state. _Source: design/library.md §3.1._

**DOM-010** (MUST) Only the kind's structure-authority source (defined in `08-metadata.md`) **MAY** create, delete or renumber catalog children. Catalog writes **MUST** arrive as a `CatalogDelta{added, updated, renumbered, tombstoned, merged}` applied by the library in one transaction per refreshed node (Flow 4.3). _Source: design/metadata-and-subtitles.md §A2; design/library.md §3.9 (2)._

**DOM-011** (MUST) Catalog fields copied from TMDb count as cached TMDb data. When they cannot be refreshed within 180 days, TMDb-only descriptive fields (overview, tagline, images, ratings) **MUST** be purged and TMDb ids **MUST** be kept; enforcement is specified in `08-metadata.md`. _Source: D20 (TMDb 180-day rule); B11._

### 2.3 Media kinds and hierarchies

**DOM-012** (MUST) The media kinds, their Title kinds, entities, items and editions **MUST** be as follows:

| Media kind | Title → | Entity kinds | Item kind | Edition / track | Coverage mode |
|---|---|---|---|---|---|
| `tv` | `series` entity | `series`, `season` (child of series; number 0 = specials), `network` | `episode` (`EpisodeInfo`) | – | exclusive |
| `movie` | `movie` item | `movie_collection`, `person` | `movie` (`MovieInfo`) | – | exclusive |
| `music` | `artist` entity | `artist`, `label` | `album` = release group (`AlbumInfo`) | `album_release` → `Track` | exclusive per track |
| `book` | `author` entity (primary author) | `author`, `book_series`, `publisher` | `book` = work (`BookInfo`) | `book_edition` (text or audio) | exclusive per edition file set |
| `adult` | `scene` or `adult_movie` item | `studio` (child of `network`), `performer`, `movie_collection` | `scene`, `adult_movie` (`SceneInfo`) | – | exclusive |
| `comic` | `comic_volume` entity | `comic_volume` (`ComicVolumeInfo`), `publisher` | `issue` (`IssueInfo`) | – | overlapping |

_Source: design/library.md §3.3; D10._

**DOM-013** (MUST) Containment (a strict tree) **MUST** be expressed by `Entity.parent` (season → series, studio → network) and `Item.container` (episode → season, album → artist, book → primary author, issue → comic volume, scene → studio; movie and adult movie have none). Credits and memberships (cast, performers, co-authors, collection members, book-series entries, track artists) **MUST** be `ItemLink` rows, never containment. _Source: design/library.md §3.2._

#### TV

**DOM-014** (MUST) An episode's identity **MUST** be its TVDB episode id (`ExternalID(tvdb, episode, …)`), never its numbers. `EpisodeInfo` **MUST** store the TVDB default-order season and episode number, the TVDB absolute number, the XEM scene season, episode and absolute numbers, and `unverified_scene`. An upstream renumbering **MUST** update `EpisodeInfo` in place so FileLinks and ItemSlots survive. _Source: O18; design/library.md §3.3; design/metadata-and-subtitles.md §A12._

**DOM-015** (MUST) Specials **MUST** be episodes in the season with number 0. Their placement **MUST** be stored as `aired_before_season`, `aired_before_episode` and `aired_after_season` on `EpisodeInfo`. _Source: O18._

**DOM-016** (MUST) `Title.series_type` (`standard`, `daily`, `anime`) and `Title.use_scene_numbering` **MUST** be library-owned. A metadata-suggested series type **MUST** be stored in the catalog (`SeriesInfo.suggested_series_type`) and applied to the Title only on creation or user confirmation. _Source: design/metadata-and-subtitles.md §A12; design/library.md §3.9 (4)._

**DOM-017** (MUST) Only the TVDB default (aired) ordering **MUST** be modelled in v1. Alternate TVDB orderings (DVD, absolute-as-order) **MUST** be addable later as additive columns or rows without changing item identity. [OPN-044] _Source: design/library.md §6 Q7; Group C (Library)._

**DOM-018** (MUST) Per-season scene titles **MUST** be `AltTitle(source=xem|scene_mapping, season_scope=N)`. _Source: design/library.md §3.3._

#### Movies

**DOM-019** (MUST) A movie is an `Item(movie)` with `MovieInfo` holding `in_cinemas_at`, `digital_at`, `physical_at`, `status` (`tba`, `announced`, `in_cinemas`, `released`) and original language. Collections **MUST** be `Entity(movie_collection)` linked by `ItemLink(role=collection_member)`; persons by `ItemLink(role=cast|crew)`. Movie edition names ("Final Cut") are a **file** attribute, not a catalog Edition. _Source: design/library.md §3.3; O3; O12._

#### Music

**DOM-020** (MUST) An album is an `Item(album)` for a MusicBrainz release group. Each release is an `Edition(album_release)` with a `media` list (medium number, format, name) and `Track` rows keyed by (edition, medium, position). Track artist credits that differ from the album artist **MUST** be `Track.artist` edges. Recording and track MBIDs **MUST** be `ExternalID`s on the `Track`. _Source: design/library.md §3.2–3.3; O4; O9._

#### Books

**DOM-021** (MUST) A book is an `Item(book)` for a work, contained by its primary author; co-authors are `ItemLink(role=co_author)`. Editions are `Edition(book_edition)` with `format`, `language` and `is_audio`. ISBN-10/13, ASIN, Hardcover, OpenLibrary work/edition and Goodreads ids **MUST** be `ExternalID`s on the edition or work. Book series **MUST** be `Entity(book_series)` with `ItemLink(role=series_entry, position_text, position)`, and **MUST** be followable. _Source: design/library.md §3.3; design/metadata-and-subtitles.md §A14._

#### Adult (D10)

**DOM-022** (MUST) Adult content **MUST** follow the Whisparr v3 model: `Item(scene)` and `Item(adult_movie)` are distinct item kinds (not a flag); `Entity(studio)` (optionally under `Entity(network)`) contains scenes; `Entity(performer)` is credited by `ItemLink(role=performer)`. Studios and performers **MUST** be followable with an `after_date` gate (DOM-078). StashDB, ThePornDB, TMDb and IMDb ids **MUST** be stored in `ExternalID`. _Source: D10; O13; O17._

**DOM-023** (MUST) The catalog **MUST NOT** auto-create library rows for performers or studios it encounters; they exist only as catalog entities until followed. _Source: design/library.md §2.2 #2._

#### Comics

**DOM-024** (MUST) A comic volume is `Entity(comic_volume)` with `ComicVolumeInfo.special_version` (`none`, `tpb`, `one_shot`, `hard_cover`, `omnibus`, `volume_as_issue`) as inferred from metadata. A user override **MUST** be stored as `Title.special_version_override` with `special_version_locked`; the effective value is the override when locked, else the catalog value. _Source: O21; design/library.md §3.3._

**DOM-025** (MUST) An issue is `Item(issue)` with `IssueInfo.number_text` (text, for example `"½"`, `"1.MU"`) and `Item.sort_key` holding the calculated sortable float. A TPB or omnibus in ComicVine is its own `comic_volume` holding one issue item. A file identified as a special version **MUST NOT** be matched to a volume whose effective special version is `none` (import rule, `06-import.md`). _Source: O21; design/library.md §3.3, §4.4._

**DOM-026** (MUST) `ItemRelation(role=collects)` **MUST** express that a collected edition's item collects other items (for example V2#1 collects V1#1–#6), sourced from metadata (Metron reprints) or the user. It **MUST NOT** create FileLinks. _Source: design/library.md §4.4._

### 2.4 External identifiers, alternate titles and product codes

**DOM-027** (MUST) Every catalog node (entity, item, edition, track) **MUST** be addressable by one or more `ExternalID(provider, namespace, value)` rows, unique across the whole map. `namespace` **MUST** distinguish id spaces within a provider (for example `tmdb: movie|tv|person|collection`, `musicbrainz: artist|release_group|release|recording|track`). _Source: D3d; O17; design/library.md §3.2._

**DOM-028** (MUST) `ExternalID.state` **MUST** be one of:

| State | Meaning |
|---|---|
| `active` | The node's current id at that provider. At most one `active` row per (node, provider, namespace). |
| `merged` | A superseded id that upstream merged into another; `merged_into_value` records the target. Kept so that old file tags, list entries, indexer results and history still resolve. |
| `deleted` | Upstream reports the id no longer exists; the node is tombstoned (DOM-034). |

_Source: design/metadata-and-subtitles.md §A5–A6; O17._

**DOM-029** (MUST) Every lookup by an external reference (indexer `tvdbid`, list entry, request TMDb id, file tag MBID, media-server GUID) **MUST** resolve through the `ExternalID` map regardless of state, and a `merged` hit **MUST** resolve to the node the row points at. _Source: design/metadata-and-subtitles.md §A6._

**DOM-030** (MUST) Alternate titles (metadata aliases, translations, scene mappings, XEM names, studio aliases, user search overrides) **MUST** be `AltTitle` rows with `source` (`metadata`, `scene_mapping`, `xem`, `user`), `usage` (`display`, `search`, `parse`, `all`), optional `language` and optional `season_scope`, attached to exactly one entity or item. Metadata refresh **MUST NOT** modify rows with `source=user`. _Source: design/library.md §3.2, §5._

**DOM-031** (MUST) Product codes (adult) **MUST** be stored in a separate `ProductCode` table with a **non-unique** index on the normalised code, not in the unique `ExternalID` map. An item **MAY** have several codes. Code lookups returning more than one item **MUST** be treated as ambiguous by matching (`04-evaluation.md`, `06-import.md`). _Source: A14; D10 restore #1; O20._

### 2.5 Catalog lifecycle, deletion and merges

**DOM-032** (MUST) A catalog node **MUST** move through `stub → complete → tombstoned → purged`. A stub **MUST** be created with its `ExternalID` as soon as a reference is needed (intent, follow, list entry), so that library rows can point at it before the fetch completes. _Source: design/library.md §3.6._

**DOM-033** (MUST) Metadata refresh **MUST NOT** delete catalog rows because of a metadata-profile change; profiles filter library membership only (DOM-104). _Source: design/library.md §2.2 #5, §3.9 (3)._

**DOM-034** (MUST) When the structure authority reports a node deleted, the node **MUST** be tombstoned, its ids set to `state=deleted`, and a health issue raised while it is referenced by a `Title`, a `FileLink` or an `Intent`. It **MUST NOT** be silently removed. An unreferenced tombstoned node **MUST** be purged; its ItemSlots cascade. _Source: O19; design/library.md §3.6; design/metadata-and-subtitles.md §A6._

**DOM-035** (MUST) Before tombstoning a child that disappeared from its parent's list, the metadata service **MUST** distinguish merge from deletion by a direct lookup where the source redirects (`08-metadata.md`). _Source: design/metadata-and-subtitles.md §A6._

**DOM-036** (MUST) A merge X → Y **MUST** be applied in one transaction:
1. If no local node has Y: re-key in place (X's `ExternalID` becomes `merged` with `merged_into_value=Y`, a new `active` row for Y is inserted). Nothing else changes.
2. If a local node already has Y: **fold** X into Y. Children, `ExternalID`s (as `merged`), `AltTitle`s, `ItemLink`s, ItemSlots, FileLinks, Intents, Follows and history move to Y; X is deleted; `EntityMerged{from, to}` is emitted.

_Source: design/metadata-and-subtitles.md §A6; design/library.md §3.6._

**DOM-037** (MUST) When a fold collides on a unique key, the survivor **MUST** be the **older** row, except that the survivor's `ItemSlot.monitored` **MUST** be `true` if either row was monitored, and its coverage **MUST** be recomputed from the moved FileLinks. [OPN-126] _Source: design/library.md §3.6 ("older ItemSlot"); design/metadata-and-subtitles.md §A6 ("union monitoring") — reconciled._

### 2.6 Slots (D4, A13, D20)

**DOM-038** (MUST) `SlotDefinition` rows **MUST** be global per media kind, with a unique `(media_kind, name)`. Each has a display `position`, a `quality_family` (`video`, `audio`, `text`, `archive`), a `score_domain` (a score domain of `07-profiles.md` PRF-080/PRF-081, for example `movie`, `book`, `audiobook`), a naming discriminator value (the `{Slot}` token), default quality profile, default root folder, default naming scheme, an optional delay-profile override (`07-profiles.md` PRF-073) and the `prefer_indexer_flags` ranking switch (`04-evaluation.md` EVL-081, default off). _Source: A13; D4; design/library.md §3.2; design/evaluation.md §3.6._

**DOM-039** (MUST) Each media kind **MUST** have at least one SlotDefinition. On first boot the server **MUST** create one default slot per kind: `tv/HD`, `movie/HD`, `music/Lossless`, `book/Ebook`, `adult/HD`, `comic/Digital`; users **MAY** add, rename and reorder slots. [OPN-127] _Source: D4; design/library.md §4 examples._

**DOM-040** (MUST) A quality profile attached to a SlotDefinition or TitleSlot **MUST** belong to the slot's `quality_family`; a mismatch **MUST** be rejected with `quality_family_mismatch`. Book slots **MUST** distinguish text and audio families, with separate ebook and audiobook score domains. [OPN-059] _Source: design/evaluation.md §3.6; design/profiles.md §3.2; Group C (Profiles)._

**DOM-041** (MUST) SlotDefinition **MUST** carry request policy: which roles may request the slot and which roles are auto-approved for it (role semantics in `10-identity-and-access.md`), and whether it is the default slot for requests of its kind (exactly one per kind). This replaces Seerr's `is4k` flags. _Source: design/identity-api-deployment.md §3.4 ("Slots instead of 4K flags"); O15._

**DOM-042** (MUST) Deleting a SlotDefinition that has TitleSlots **MUST** be refused with `slot_in_use`. _New; protects D4 per-slot history._

**DOM-043** (MUST) `TitleSlot` **MUST** be unique per `(title, slot_definition)` and **MUST** hold: root folder, folder `path` relative to that root, quality profile, optional naming scheme, `monitored` (the pause flag and the "parent monitored" gate of O1), an optional delay-profile override (`07-profiles.md` PRF-073), an optional season-pack policy override (`04-evaluation.md` EVL-071), and kind-specific settings (`minimum_availability` for movies, `accept_collected` for comics). Unset values default from the SlotDefinition. _Source: A13; design/library.md §3.2._

**DOM-044** (MUST) The effective naming scheme for a TitleSlot **MUST** resolve in this order: `TitleSlot.naming_scheme` → `SlotDefinition.default_naming_scheme` → `RootFolder.default_naming_scheme` → the kind's built-in scheme. _Source: design/import.md §3.4 ("chosen per slot, defaulting per root folder"); task scope (TitleSlot naming)._

**DOM-045** (MUST) A TitleSlot's root folder **MUST** have the title's media kind, and, if the root folder names a slot, that slot. Violations **MUST** be rejected with `root_kind_mismatch` or `root_slot_mismatch`. _Source: design/library.md §3.11._

**DOM-046** (MUST) If two TitleSlots of the same title resolve to the same folder, the effective naming templates **MUST** contain the slot discriminator or a quality token, so that files cannot collide (Invariant P). The naming engine (`06-import.md`) validates this and the configuration **MUST** be rejected with `naming_collision` otherwise. _Source: design/library.md §3.11._

**DOM-047** (MUST) `ItemSlot` **MUST** be unique per `(item, slot_definition)` and **MUST** hold `monitored`, `coverage`, `target_edition` and `edition_locked`, a reference to its TitleSlot, and a `version` counter. `version` **MUST** be incremented in the same transaction as every change to the ItemSlot's incumbents (files and in-flight download targets), its monitoring, its TitleSlot's profile binding, and every grab transaction that touches it; the grab transaction of `04-evaluation.md` (EVL-060, EVL-093) compares it for optimistic concurrency. `ItemSlot.slot_definition` **MUST** equal `title_slot.slot_definition`, and the item's layout root **MUST** be the TitleSlot's title. _Source: A13; D20; design/library.md §3.2._

**DOM-048** (MUST) The target edition (books) and target release (music) **MUST** belong to the ItemSlot, not the Item, so that different slots of one album or book can target different editions. `edition_locked=false` reproduces `AnyReleaseOk`/`AnyEditionOk`: import **MAY** switch `target_edition` to the edition it identified. `edition_locked=true` **MUST** prevent that. _Source: A13; D20; O10; design/import.md §6.1 (1)._

**DOM-049** (MUST) Every read the pipeline makes about monitoring, wanted state, grabs, queue, history, naming and availability **MUST** be keyed by (item, slot). No status field **MAY** exist per "4K" variant outside the slot model. _Source: D4; design/library.md §3.8._

### 2.7 Titles

**DOM-050** (MUST) A `Title` **MUST** point at exactly one layout-root node: an entity (for `series`, `artist`, `author`, `comic_volume`) or an item (for `movie`, `scene`, `adult_movie`). At most one Title **MAY** exist per node; adding an existing one **MUST** fail with `title_exists`. _Source: design/library.md §3.2._

**DOM-051** (MUST) An item **MUST** join the library only under the Title of its layout root. If that Title does not exist when an item is added (for example a single album or book), it **MUST** be created with a `Follow(add=none)` so that no other members are admitted, reproducing the "None" metadata profile. _Source: O11; design/library.md §3.2 ("Container membership")._

**DOM-052** (MUST) Every Title **MUST** have at least one Intent (DOM-065). A Title with no Intent in `pending` or `approved` state is shown by the derived filter "no active intent", which replaces Sonarr's `KeepAndTag`. _Source: design/library.md §3.7._

**DOM-053** (MUST) `Title.include_specials` (default `false`) **MUST** govern, for TV: whether presets monitor season-0 episodes, whether specials count in `missing` and in availability totals. The same flag **MUST** be settable per request (`Intent.include_specials`), and fulfilling an intent with `include_specials=true` **MUST** set it on the Title. [OPN-047] _Source: O2; design/library.md §6 Q10; Group C (Library)._

**DOM-054** (MUST) A special monitored by hand on a title with `include_specials=false` **MUST** be grabbable by RSS and interactive search, but **MUST NOT** appear in `missing` or be backlog-searched. [OPN-128] _Source: design/library.md §4.1 (reconciled with O2); new._

**DOM-055** (MUST) Removing a Title **MUST** delete its TitleSlots, ItemSlots, Follows and non-request Intents, withdraw its request Intents, and either delete its files or keep them as unmapped MediaFiles (FileLinks removed, `title` cleared), as the user chooses. The user **MAY** add an `Exclusion(applies_to=both)` in the same action. _Source: design/library.md §3.7 ("removed with exclusion")._

### 2.8 Media files and links

**DOM-056** (MUST) A `MediaFile` **MUST** be stored as `(root_folder_id, rel_path)`, unique together, where `rel_path` is relative to the root folder. Absolute paths **MUST NOT** be stored. Moving or remapping a root is one RootFolder row update. _Source: A12; design/import.md §3.3._

**DOM-057** (MUST) Every MediaFile that has at least one FileLink **MUST** belong to exactly one SlotDefinition and one Title. An unmapped MediaFile (no FileLinks) **MAY** have no slot and no title. This single representation replaces Lidarr `AlbumId=0`, Whisparr v3 `ImportFiles` and root-folder unmapped folders. _Source: design/library.md §3.6; D20._

**DOM-058** (MUST) `FileLink` **MUST** relate one MediaFile to one Item and optionally one Edition, one Track, a `part` index (audiobook part or music disc) and a `coverage` descriptor (comic issue range or special version). For every link, `file.slot_definition` **MUST** equal the ItemSlot's slot and `file.title` **MUST** be the item's Title. _Source: O9; design/library.md §3.2; design/import.md §3.3._

**DOM-059** (MUST) Coverage **MUST** be computed per (item, slot) from **present** files only, by the kind's rule:

| Kind | `complete` when | `partial` when |
|---|---|---|
| episode, movie, scene, adult_movie | ≥ 1 present linked file | – |
| album | every track of `target_edition` has a present linked file | some but not all tracks linked |
| book | a present file set for one edition of the slot's medium has all parts (`part_count`) | some parts present |
| issue | ≥ 1 present linked file | – |

Partial coverage counts as missing (O4). If `target_edition` is null, album coverage is `none`. _Source: O4; O9; design/library.md §3.3, §3.5._

**DOM-060** (MUST) In `exclusive` mode, every (item, slot, track-or-part) **MUST** have at most one present covering file; importing an overlapping file into the same slot **MUST** retire the old file (`06-import.md` retire set). In `overlapping` mode (comics), several present files **MAY** cover the same item. _Source: design/library.md §3.2; O9; O21._

**DOM-061** (MUST) `ItemSlot.coverage` is stored and **MUST** change only inside the import, delete, rescan and merge transactions that write FileLinks or MediaFile state, with the ItemSlot row locked (`SELECT … FOR UPDATE` on PostgreSQL; the single writer on SQLite). _Source: design/library.md §3.5 notes._

**DOM-062** (MUST) **One download **MAY** feed several slots** (D20). When a download's targets name several (item, slot) pairs that accept its content, import **MUST** create a **separate MediaFile per slot**, each under that slot's TitleSlot root, path and naming scheme, transferred by reflink, then hardlink, then copy (`06-import.md`). Slot files **MAY** share an inode. Each slot's file **MUST** then be upgraded, retired, renamed and deleted independently: retiring one slot's file **MUST NOT** touch another slot's file. A slot whose import is rejected (for example `NotQualityUpgrade`) **MUST NOT** block the other slots. _Source: D20; B1(a); design/evaluation.md §4.2 step 7._

**DOM-063** (MUST) A file on disk that was not delivered by a multi-slot download (library import, rescan) **MUST** be assigned to exactly one slot by the import slot-selection rule. When several slots share the root folder, the slot **MUST** be inferred by profile fit; a tie is `SlotAmbiguous`. [OPN-046] _Source: design/import.md §3.2 stage 6; design/library.md §3.10; Group C (Library)._

**DOM-064** (MUST) A MediaFile **MUST** have `state` `present` or `missing`. A scan that no longer finds the file **MUST** set `missing` and `missing_since`; missing files **MUST NOT** count toward coverage; after the grace period (default 7 days) the row and its FileLinks **MUST** be deleted with reason `missing_from_disk`. A reappearing file within the grace period **MUST** return to `present` with its links. [OPN-033] _Source: design/import.md §3.7; design/library.md §3.10; Group C (Import)._

### 2.9 Intents

**DOM-065** (MUST) Every path that adds to the library **MUST** create an `Intent` with one origin:

| Origin | Created by | Initial status |
|---|---|---|
| `manual` | UI/API add | `approved` |
| `user_request` | a user's request | `pending`, or `approved` if the requester is auto-approved for the slot (DOM-041) |
| `watchlist` | a per-user watchlist import list (`is_auto=true`, `requested_by` = list owner) | as `user_request` |
| `import_list` | an import list entry | `approved`, or `pending` when the list's `approval=manual` |
| `follow` | follow expansion | `approved` |
| `disk_import` | library import of existing files | `approved` |

_Source: design/library.md §3.7; O15._

**DOM-066** (MUST) An Intent **MUST** target exactly one catalog node (entity or item), one SlotDefinition, and an optional scope of child entities (requested seasons; empty = whole target). It **MUST** record `requested_by` and `decided_by` (users, `10-identity-and-access.md`), the originating `import_list` or `follow`, `preset`, `include_specials`, `search_on_fulfil`, `ignore_quota`, and per-request overrides (quality profile, root folder, tags). _Source: design/library.md §3.2._

**DOM-067** (MUST) Intent status **MUST** follow `pending → approved | declined`, `pending | approved → withdrawn`. There **MUST NOT** be a failed state. `fulfilled_at` records when the preset and monitoring were applied. _Source: design/library.md §3.6; §2.2 #11._

**DOM-068** (MUST) Fulfilment **MUST** run in the approval transaction when the target is `complete`, otherwise as a durable job keyed by intent id that runs when the stub completes. Fulfilment **MUST**: ensure the Title and TitleSlot (resolving root folder and profile as slot defaults → routing rules → intent overrides), create the follows and ItemSlots, apply the preset once, and enqueue a search if `search_on_fulfil`. Errors **MUST** be retried, not surfaced as a request failure. _Source: design/library.md §3.6–3.7._

**DOM-069** (MUST) At most one `import_list` or `watchlist` Intent **MAY** exist per (import list, target, slot); a re-appearing list entry **MUST** re-activate the withdrawn row. _New; needed for retraction (DOM-094)._

**DOM-070** (MUST) Each `user_request` Intent **MUST** store `quota_units`, computed at creation: TV = number of requested seasons (all seasons for a whole-series request), movie = 1, music = albums, book = books, adult = items, comic = volumes. Quota rules are owned by `10-identity-and-access.md` and count these rows. [OPN-118] _Source: design/identity-api-deployment.md §3.4 ("as the domain design decides"); O15._

**DOM-071** (MUST) Withdrawing a user request **MUST NOT** unmonitor anything by default. _Source: design/library.md §3.7 (Seerr parity)._

**DOM-072** (MUST) TV requests arriving with a TMDb TV id **MUST** be resolved to a TVDB-keyed series before fulfilment (`08-metadata.md` resolution). An unresolved mapping **MUST** keep the intent `pending` and block auto-approval. _Source: A17; design/metadata-and-subtitles.md §A15._

### 2.10 Follows

**DOM-073** (MUST) A `Follow` **MUST** be unique per `(entity, slot_definition)` and **MUST** hold: `add` (`none`, `all`, `released`, `profile` + metadata profile), `monitor` (`none`, `all`, `from_date` + `from_date`), `after_date`, `gate_action` (`skip`, `add_unmonitored`) + `gate_tag`, `item_kinds`, `search_on_add`, defaults for Titles it creates (quality profile, root folder, naming scheme), and tags. _Source: D10; design/library.md §3.2._

**DOM-074** (MUST) Every "keep in sync with upstream" behaviour **MUST** be a Follow:

| Reference behaviour | Follow |
|---|---|
| Sonarr `MonitorNewItems=All/None` | `Follow(series, add=all, monitor=all|none)`; new seasons get season follows |
| Sonarr episode inherits `Season.Monitored` | `Follow(season, add=all, monitor=all|none)` |
| Lidarr/Bookshelf `MonitorNewItems=All/None/New` | `Follow(artist|author, add=profile, monitor=all|none|from_date)` |
| Radarr monitored collection | `Follow(movie_collection, add=released, monitor=all)` |
| Whisparr v3 performer/studio | `Follow(performer|studio, add=all, monitor, after_date, item_kinds, gate_action)` |
| Kapowarr `monitor_new_issues` | `Follow(comic_volume, add=all, monitor=all|none)` |
| Book series (new) | `Follow(book_series, add=profile, monitor)` |
| TMDb person/company, StashDB performer/studio, MusicBrainz series lists | Follows on the corresponding catalog entity where that entity type exists |

_Source: design/library.md §3.4; O6; O12; O13._

**DOM-075** (MUST) Lidarr's "New" **MUST** be expressed as `monitor=from_date` with `from_date` defaulting to the follow's creation time, not "newer than the newest existing album". [OPN-043] _Source: design/library.md §6 Q6; Group C (Library)._

**DOM-076** (MUST) A follow on `Entity(season)` with number 0 **MUST** default to `monitor=none` and **MUST** never be set to `monitor=all` by a preset or series follow unless `include_specials` is set. _Source: O6._

**DOM-077** (MUST) Editing a Follow's policy **MUST** affect only future members. Applying a policy to existing members **MUST** be a separate explicit action that runs a preset. _Source: design/library.md §3.4._

**DOM-078** (MUST) Follow expansion **MUST** run when the catalog reports new members of the entity, when the follow is created, and when its policy is edited; it **MUST** admit member `m` for slot `F.slot` iff:
`m.kind ∈ F.item_kinds` ∧ add-policy(`F.add`, m) ∧ ¬excluded_auto(m) ∧ `m.release_at ≥ F.after_date` ∧ for every performer p credited on m with a follow Fp: `m.release_at ≥ Fp.after_date`.
`excluded_auto(m)` **MUST** be true if m, or any entity linked to m (container, studio, performer, collection, book series), has an Exclusion with `applies_to ∈ {auto_adds, both}`. _Source: O13; design/library.md §3.4._

**DOM-079** (MUST) When admission fails only on a date or entity gate and `F.gate_action=add_unmonitored`, m **MUST** be admitted with `monitored=false` and `F.gate_tag` added to its Title. Gate outcomes **MUST NOT** be stored as Exclusion rows. _Source: O13; design/library.md §2.2 #4._

**DOM-080** (MUST) On admission, expansion **MUST** ensure the Title of m's layout root (with an `Intent(origin=follow)` when newly created), the TitleSlot (from `F` defaults, else slot defaults), and the ItemSlot with `monitored = monitor-policy(F.monitor, m)`; for a new season it **MUST** ensure `Follow(season, slot)` per DOM-076. If `F.search_on_add` and the ItemSlot is monitored and released, it **MUST** enqueue a search (`03-acquisition.md`). _Source: design/library.md §3.4._

### 2.11 Monitoring rules

**DOM-081** (MUST) **Invariant M.** `ItemSlot.monitored` **MUST** change only through: (a) a user action; (b) fulfilment of an intent (one-shot preset) or list retraction (DOM-094); (c) a follow's policy applied to **newly appearing** members; (d) auto-unmonitor on file deletion (DOM-085). Metadata refresh **MUST NOT** change an existing flag. Each change **MUST** emit `ItemSlotMonitoringChanged{item, slot, old, new, cause}`. _Source: O7; design/library.md §3.1._

**DOM-082** (MUST) Add-time presets **MUST** be one-shot predicates over the title's current members for one slot, applied by intent fulfilment or a user bulk action:

| Preset | Predicate P(item) | Follows created |
|---|---|---|
| `all` | not a special unless `include_specials` | series + each non-zero season (TV); `Follow(add=profile, monitor=all)` for artist/author/volume |
| `future` | `release_at` null or ≥ now | series (monitor new seasons only if continuing or upcoming), last season; `monitor=from_date(now)` for artist/author/volume |
| `missing` | coverage = none in this slot | none |
| `existing` | coverage ≠ none in this slot | none |
| `first_group` | first season (TV); first album/book | – |
| `last_group` | last season (TV); latest album/book | last season |
| `pilot` | first season, episode 1 | – |
| `recent` | `release_at` ≥ now − 90 days | as `future` |
| `none` | false | none |
| `skip` | leave existing flags unchanged | – |

Comics support `all`, `missing`, `none`. Setting a season follow and its episodes' flags **MUST** happen in the same transaction. _Source: O5; design/library.md §3.4._

**DOM-083** (MUST) On a title's first expansion the preset **MUST** govern. On later refreshes of a containment follow (series, artist, author, volume), members with `release_at < now − 14 days` appearing while the title already has members **MUST** get monitoring from the follow as usual, but **MUST** have `search_hold=true` (auto-search suppressed until the user confirms), and a `LateMembersAppeared` event **MUST** be emitted. _Source: O7; design/library.md §3.4._

**DOM-084** (MUST) Unmonitoring an ItemSlot **MUST NOT** delete files. Consumers of `ItemSlotMonitoringChanged` (pending grabs in `04-evaluation.md`, active downloads in `05-downloads.md`) decide their own reaction. _New; keeps monitoring a pure want flag._

**DOM-085** (MUST) When the setting `library.unmonitor_on_delete` is on, deleting the last covering file of an ItemSlot **MUST** set `monitored=false`, except when the deletion reason is `upgrade`, `manual_override` or `missing_from_disk`. The setting **MUST** default to off. [OPN-129] _Source: O8._

**DOM-086** (MUST) `TitleSlot.monitored=false` (pause) **MUST** exclude every ItemSlot of that TitleSlot from automatic acquisition without changing any `ItemSlot.monitored`. Interactive search **MUST** ignore monitoring. _Source: O1._

### 2.12 Wanted, missing, cutoff-unmet and availability (queries)

**DOM-087** (MUST) Wanted, missing, cutoff-unmet and availability **MUST** be computed by queries over `ItemSlot ⋈ Item ⋈ TitleSlot [⋈ kind info]` and **MUST NOT** be stored, except for `ItemSlot.coverage` (DOM-061). _Source: design/library.md §3.1, §3.5._

**DOM-088** (MUST) The predicates **MUST** be:

```
monitored(i,s)    := ItemSlot(i,s).monitored ∧ TitleSlot(title(i),s).monitored                    -- O1
released(i,s)     := episode:     release_at ≠ null ∧ release_at + runtime ≤ now                   -- O2
                     movie:       is_available(MovieInfo, TitleSlot.minimum_availability, delay)   -- O3
                     otherwise:   release_at ≠ null ∧ date(release_at) ≤ today(UTC)
missing(i,s)      := monitored ∧ released ∧ coverage ∈ {none, partial} ∧ ¬search_hold
                     ∧ (¬special(i) ∨ Title.include_specials)
cutoff_unmet(i,s) := monitored ∧ coverage = complete ∧ upgradeable(profile(TitleSlot), files(i,s))  -- 04-evaluation.md
```

`grabbable(i,s)` (early grabs such as TV RSS for unaired monitored episodes) is owned by `04-evaluation.md`. _Source: design/library.md §3.5._

**DOM-089** (MUST) Movie `is_available` **MUST** follow Radarr: `announced` = always (TBA included); `in_cinemas` = `in_cinemas_at ≤ now`; `released` = min(`physical_at`, `digital_at`) ≤ now, else `in_cinemas_at + 90 days` ≤ now, else never; each shifted by the availability delay. The delay **MUST** be a global library setting `library.movie_availability_delay_days` (default 0). [OPN-130] _Source: O3; `R/Movies/Movie.cs:77-118`._

**DOM-090** (MUST) `Availability(target, scope, slot)` **MUST** return `{total, released, covered, status}` where `total` counts the target's members in scope (specials only with `include_specials`), `covered` counts ItemSlots with coverage `complete`, and `status` is `none` (covered = 0), `complete` (covered = total), `caught_up` (covered = released < total), else `partial`. For a request, display state is `pending`, `declined`, or `approved` + (`processing` while monitored and not complete | the availability status). _Source: O15; O16; design/library.md §3.8._

**DOM-091** (MUST) Transactions that change coverage or catalog membership **MUST** recompute availability for the approved intents touching the changed items and emit `AvailabilityChanged{intent, slot, old, new}`. Requesters **MUST** be notified on the transition to `complete`; notification on `caught_up` **MUST** be an option (default off). A transition from `complete`/`caught_up` back to `none`/`partial` replaces Seerr's DELETED. [OPN-039] _Source: design/library.md §3.8; Group C (Library)._

**DOM-092** (MUST) Media-server presence ("visible in Plex/Jellyfin") **MAY** be recorded as a separate signal but **MUST NOT** change library availability. _Source: design/library.md §3.8._

**DOM-093** (MUST) When `TitleSlot.accept_collected=true` (comics; default `false`), `covered'(i,s) := coverage(i,s)=complete ∨ ∃c: ItemRelation(c collects i) ∧ coverage(c,s)=complete` **MUST** replace coverage in `missing` and availability for that TitleSlot. This is derived; it **MUST NOT** create FileLinks. [OPN-038] _Source: design/library.md §4.4, §6 Q1; Group C (Library)._

### 2.13 Import lists, exclusions and retraction

**DOM-094** (MUST) A list sync **MUST** perform retraction only when **every enabled import list fetched successfully** in that run. It **MUST** then mark each `import_list`/`watchlist` intent whose entry is gone as `withdrawn`, and apply the list's `retraction` policy to a Title only if every intent on that Title is `withdrawn` or `declined` and none has origin `manual`:

| Policy | Effect |
|---|---|
| `none` | nothing |
| `log` | event only |
| `unmonitor` | ItemSlots in the withdrawn intents' scope and slots set `monitored=false` |
| `remove_keep_files` | Title removed, files kept as unmapped |
| `remove_delete_files` | Title and its files removed |

A title added manually that also appeared on a list **MUST** never be affected. [OPN-040] _Source: O14; design/library.md §3.7, §2.2 #3; Group C (Library)._

**DOM-095** (MUST) Import lists **MUST** support `approval=auto|manual`; `manual` creates `pending` intents shown in the request queue. [OPN-041] _Source: design/library.md §6 Q4; Group C (Library)._

**DOM-096** (MUST) An ImportList **MUST** carry per-slot templates (quality profile, root folder), a preset, an optional follow policy (for example "entire artist" → `Follow(artist)`), tags, an optional owner (per-user watchlists), and sync status (`last_fetch_at`, `last_fetch_ok`). Plex/Jellyfin watchlists **MUST** be one import-list provider producing per-user `Intent(origin=watchlist)`. _Source: design/library.md §3.2, §2.2 #10._

**DOM-097** (MUST) An `Exclusion` **MUST** be keyed by an external reference `(provider, namespace, value)` or by a catalog node, and carry `applies_to` (`auto_adds`, `user_requests`, `both`), `reason` (`manual`, `removed_with_exclusion`, `keyword_rule`), a note and creator. `auto_adds` **MUST** block follows, lists and collections but not manual adds; `user_requests` **MUST** block requests; "remove with exclusion" **MUST** default to `both`. Entity exclusions **MUST** be evaluated as predicates at admission (DOM-078), never materialised per item. _Source: O14; O15; design/library.md §3.7._

### 2.14 Root folders

**DOM-098** (MUST) A `RootFolder` **MUST** have an absolute server path (identical on every node, D14), a media kind, an optional SlotDefinition (null = any slot of the kind), a marker UUID written to `.bonnyread-root` at creation (`12-platform.md` health check), add defaults, a default naming scheme, and import options (hardlink/reflink allowed, hash verification, permissions, file date, recycle bin; semantics in `06-import.md`). Free space and unmapped folders **MUST** be derived at read time. _Source: O22; design/library.md §3.11; design/runtime.md mount verification; design/import.md §3.5._

**DOM-099** (MUST) Deleting a RootFolder referenced by a TitleSlot or MediaFile **MUST** be refused with `root_folder_in_use`. Changing its path **MUST** be a single row update (DOM-056). _Source: A12._

**DOM-100** (MUST) `RootFolder.add_defaults.auto_add` (default `false`) **MUST** allow, for every media kind, creating Titles with `Intent(origin=disk_import)` for unknown titles found on disk. [OPN-045] _Source: design/library.md §3.10, §6 Q8; Group C (Library, Import)._

### 2.15 Tags

**DOM-101** (MUST) `Tag` labels **MUST** be unique case-insensitively. Tags attach to Titles through `TitleTag` with `source` `manual` or `rule`. Follows, ImportLists and Intents **MUST** carry tags to stamp on the Titles they create. _Source: O23; design/library.md §3.11._

**DOM-102** (MUST) An `AutoTagRule` **MUST** only add and remove its own `TitleTag(source=rule, rule=R)` rows; it **MUST NOT** remove manual tags. _Source: O23; design/library.md §3.11._

**DOM-103** (MUST) Tags scope indexers, download clients, notifications and profiles through join tables owned by those files. Per-slot scoping **MUST** use the SlotDefinition as a selector, not tags. _Source: design/library.md §3.11._

### 2.16 Metadata profiles

**DOM-104** (MUST) A `MetadataProfile` (music, book) **MUST** be a membership predicate over catalog members (types, statuses, language, popularity), used by `Follow(add=profile)`. A reserved "None" profile per kind **MUST** exist. Changing a profile **MUST** re-run admission for new members only and offer "apply to existing"; it **MUST NOT** delete catalog rows or files. The metadata service **MAY** also use the profile to skip fetches where the provider supports filtering (`08-metadata.md`). [OPN-042] _Source: O11; design/library.md §3.9 (3); Group C (Library)._

### 2.17 Per-kind extension points

**DOM-105** (MUST) Each media kind **MUST** be implemented as a compiled-in `KindModule` (native Go, not a WASM plug-in) that registers:

| Extension point | Provides | Consumer (owning file) |
|---|---|---|
| Hierarchy | entity kinds, item kind, edition/track use, title kind, layout-root rule, coverage mode and rule, `released` predicate, supported presets, default slot | this file |
| Sub-parser | the kind's `Structure` from release names | `04-evaluation.md` |
| Identifier (+ Grouper, Prober) | file → items (+ slot) identification | `06-import.md` |
| Naming token set | typed tokens and template slots | `06-import.md` |
| Metadata assembler | composing sources into catalog deltas under the structure-authority rule | `08-metadata.md` |
| Quality family | the axes of the kind's slots | `04-evaluation.md`, `07-profiles.md` |

_Source: design/import.md §3.1; design/metadata-and-subtitles.md §A1; design/evaluation.md §3.6._

**DOM-106** (MUST) Adding a media kind **MUST** require only: new enum values, new 1:1 kind-info tables, and a KindModule. It **MUST NOT** require changing columns of `Entity`, `Item`, `ItemSlot`, `TitleSlot`, `MediaFile` or `FileLink`. _Source: D2._

### 2.18 Phase-1 groundwork (A17)

**DOM-107** (MUST) The P1 schema **MUST** include, for all kinds: every `Entity`, `Item`, `Title` and `media_kind` enum value; `Edition` and `Track` (referenced by `FileLink` and `ItemSlot`); `ItemLink`, `EntityLink`, `ItemRelation`; `FileLink.edition`, `.track`, `.part`, `.coverage`; `ItemSlot.target_edition`, `.edition_locked`; `MediaFile.part`, `.part_count`, `.quality_family`; and `SlotDefinition.quality_family` and `score_domain`. _Source: A17; D2._

**DOM-108** (MAY) Kind-info tables used only by later kinds (`AlbumInfo`, `BookInfo`, `SceneInfo`, `IssueInfo`, `ComicVolumeInfo`, `ProductCode`) **MAY** be added by additive migrations in their phase, since they reference `Item`/`Entity` without changing them. _Source: A7; D2._

**DOM-109** (MUST) P1 **MUST** also deliver the other A17 groundwork owned elsewhere: subtitle sidecar indexing and stream flags in the shared probe (`06-import.md`, `09-subtitles.md`), and TMDb discovery with TMDb → TVDB resolution (`08-metadata.md`). The subtitle profile **MUST** be assignable per Title (one per item, applied to all slots). [OPN-049] _Source: A17; design/metadata-and-subtitles.md §B2, §6.2 Q4; Group C (Metadata and subtitles)._

### 2.19 Shared records used by several areas

**DOM-110** (MUST) Subtitles **MUST** be recorded in one shared `Subtitle` table defined here and used by `06-import.md` (sidecar indexing, P1 groundwork) and `09-subtitles.md` (P2). A `Subtitle` row **MUST** reference exactly one MediaFile (cascade on delete) and **MUST** carry the fields required by `09-subtitles.md` SUB-002: `origin` (`embedded`, `external`, `downloaded`, `uploaded`, `manual`, `extracted`, `translated`, `generated`), `language` (ISO 639 with optional region or script), `hi`, `forced`, `format`, `stream_index` (embedded only), for downloaded subtitles `provider`, `provider_sub_id`, `release_info`, `uploader`, `score`, `score_max` and `matches`, the `sync` record and the applied `mods`. A sidecar subtitle file **MUST** additionally reference its `ExtraFile{type=subtitle}` row (`06-import.md` IMP-072, IMP-077), which owns the file's path; an embedded track has no ExtraFile. The subtitle profile is assigned by the edge `Title.subtitle_profile` → `SubtitleProfile` (`09-subtitles.md`, DOM-109). _Source: design/metadata-and-subtitles.md §B2; design/import.md §3.6; A17._

**DOM-111** (MUST) A `MediaServer` row **MUST** describe one media-server connection (`kind` `plex`, `jellyfin`, `emby`, `kodi`, `subsonic`, `audiobookshelf`, `kavita`, `komga` or `stash`; name; URL; write-only credential; enabled flag). It is the single connection record used by media-server refresh (`06-import.md` IMP-078), media-server sign-in (`10-identity-and-access.md` IAM-040, `MediaServerSignInPolicy`) and media-server watchlist import lists (DOM-096). The refresh implementation per kind is registered as a provider in the notification-target registry (`12-platform.md` PLT-079). _Source: design/import.md §3.9; design/identity-api-deployment.md §3.2; D17._

---

## 3. Data model

Notation is ent-style. `edge.To`/`edge.From` name the target; `?` means optional; indexes and CHECKs follow each entity. Timestamps and `id` (DOM-001) are omitted.

### 3.1 Catalog

```go
// Entity — catalog node that groups or credits items.
field.Enum("kind").Values("series","season","artist","author","comic_volume","studio","network",
    "performer","person","movie_collection","book_series","label","publisher")
field.String("title"); field.String("sort_title"); field.String("clean_title")
field.Enum("status").Values("continuing","ended","upcoming","active","inactive","unknown")
field.Int("year").Optional().Nillable()        // series/volume start year
field.Int("number").Optional().Nillable()      // season number; volume number
field.JSON("attrs", EntityAttrs{})             // non-queried display data: overview, images, genres, ratings, gender…
field.Enum("catalog_state").Values("stub","complete","tombstoned")
field.Time("refreshed_at").Optional().Nillable()
edge.To("parent", Entity).Unique()?            // season→series, studio→network (strict tree)
index: (kind, clean_title); (parent_id, number)

// SeriesInfo (1:1 Entity kind=series)
field.String("original_language"); field.Enum("suggested_series_type").Values("standard","daily","anime")

// ComicVolumeInfo (1:1 Entity kind=comic_volume)   — P3 (DOM-108)
field.Enum("special_version").Values("none","tpb","one_shot","hard_cover","omnibus","volume_as_issue")
field.Int("issue_count")

// Item — unit of want.
field.Enum("kind").Values("episode","movie","album","book","scene","adult_movie","issue")
field.String("title"); field.String("clean_title")
field.Time("release_at").Optional().Nillable()   // UTC instant; episodes: air date+time from 08-metadata
field.Enum("release_precision").Values("instant","day","month","year","unknown")
field.Int("runtime_min").Optional().Nillable()
field.Float("sort_key")                           // episode number / issue calculated number / position
field.Enum("catalog_state").Values("stub","complete","tombstoned")
edge.To("container", Entity).Unique()?           // episode→season, album→artist, book→author, issue→volume, scene→studio
index: (container_id, sort_key); (kind, release_at); (kind, clean_title)
check: kind IN (movie, adult_movie) OR container_id IS NOT NULL

// Kind-info tables, 1:1 with Item (item_id is PK and FK).
EpisodeInfo {  season_number, episode_number int; absolute_number int?
               scene_season, scene_episode, scene_absolute int?; unverified_scene bool
               aired_before_season, aired_before_episode, aired_after_season int?
               finale enum(series, season, midseason)?
               index (season_number, episode_number); (absolute_number); (scene_season, scene_episode) }
MovieInfo   {  in_cinemas_at, digital_at, physical_at time?; original_language string
               status enum(tba, announced, in_cinemas, released) }
AlbumInfo   {  primary_type enum; secondary_types []enum }                              // P2
BookInfo    {  original_language string? }                                              // P3
SceneInfo   {  release_date_precision enum(day, month, year) }                          // P3
IssueInfo   {  number_text string; is_annual bool }                                     // P3

// Edition — variant of an item.
field.Enum("kind").Values("album_release","book_edition")
field.String("title"); field.String("format"); field.String("language").Optional()
field.Time("release_at").Optional().Nillable()
field.JSON("media", []Medium{})            // albums: [{number, format:"CD"|"Vinyl"|"Digital", name}]
field.Bool("is_audio")                     // books: audio edition vs text
field.Enum("status").Values("official","promotion","bootleg","pseudo").Optional()   // albums
field.Int("track_count")
edge.From("item", Item).Required().Unique()
index: (item_id)

// Track — component of an album release.
field.Int("medium"); field.Int("position"); field.String("number_text")  // "A1"
field.String("title"); field.Int("duration_ms")
edge.From("edition", Edition).Required().Unique(); edge.To("artist", Entity).Unique()?
unique: (edition_id, medium, position)

// ItemLink — credit/membership (M:N, not containment).
field.Enum("role").Values("performer","cast","crew","collection_member","series_entry","co_author","track_artist")
field.String("position_text").Optional(); field.Float("position").Optional(); field.JSON("attrs", LinkAttrs{})
edge.To("entity", Entity).Required(); edge.To("item", Item).Required()
unique: (entity_id, item_id, role); index: (item_id, role)

// EntityLink — relation between entities.
field.Enum("role").Values("member_of","alias_of","network_of")
edge.To("from", Entity).Required(); edge.To("to", Entity).Required()
unique: (from_id, to_id, role)

// ItemRelation — item-to-item (collected editions).
field.Enum("role").Values("collects"); field.Enum("source").Values("metadata","user")
edge.To("from", Item).Required(); edge.To("to", Item).Required()
unique: (from_id, to_id, role); index: (to_id, role)

// ExternalID — one map for every provider and node type.
field.String("provider")        // registry-validated (DOM-006): tvdb, tmdb, imdb, tvmaze, mal, anilist, anidb,
                                // musicbrainz, hardcover, openlibrary, isbn, asin, goodreads, stashdb, tpdb,
                                // comicvine, metron, …
field.String("namespace")       // e.g. tmdb: movie|tv|person|collection; musicbrainz: artist|release_group|release|recording|track
field.String("value")
field.Enum("state").Values("active","merged","deleted")
field.String("merged_into_value").Optional()
field.Time("confirmed_at").Optional().Nillable()
edge.To("entity", Entity)?; edge.To("item", Item)?; edge.To("edition", Edition)?; edge.To("track", Track)?
check: exactly one of entity_id, item_id, edition_id, track_id
unique: (provider, namespace, value)
unique partial: (entity_id, provider, namespace) WHERE state='active' AND entity_id IS NOT NULL
               — likewise for item_id, edition_id, track_id

// AltTitle
field.String("title"); field.String("clean_title"); field.String("language").Optional()
field.Enum("source").Values("metadata","scene_mapping","xem","user")
field.Enum("usage").Values("display","search","parse","all")
field.Int("season_scope").Optional().Nillable()
edge.To("entity", Entity)?; edge.To("item", Item)?
check: exactly one of entity_id, item_id
index: (clean_title); (entity_id); (item_id)

// ProductCode — non-unique alternate id (A14).   — P3
field.String("code"); field.String("normalized")        // upper-case, separators removed
field.Enum("source").Values("metadata","user")
edge.To("item", Item).Required()
index: (normalized)  — NON-unique;  unique: (item_id, normalized)
```

### 3.2 Library

```go
// SlotDefinition (A13) — global per media kind.
field.Enum("media_kind").Values("tv","movie","music","book","adult","comic")
field.String("name"); field.Int("position")
field.Enum("quality_family").Values("video","audio","text","archive")
field.String("score_domain")              // e.g. "movie", "book.ebook", "book.audio" (07-profiles.md)
field.String("discriminator")             // value of the {Slot} naming token
field.JSON("attrs", SlotAttrs{})          // e.g. books {"medium":"audio"}: identification bias
field.Bool("default_for_requests")
field.Bool("prefer_indexer_flags").Default(false)              // EVL-081
edge.To("default_quality_profile", QualityProfile).Required()   // 07-profiles.md
edge.To("delay_profile_override", DelayProfile)?                // 07-profiles.md PRF-073
edge.To("default_root_folder", RootFolder)?
edge.To("default_naming_scheme", NamingScheme)?                 // 06-import.md
edge.To("requestable_by", Role); edge.To("auto_approve_for", Role)   // 10-identity-and-access.md
unique: (media_kind, name)
unique partial: (media_kind) WHERE default_for_requests

// RootFolder
field.String("path").Unique()             // absolute server path, identical on all nodes (D14)
field.Enum("media_kind"); field.UUID("marker")
field.JSON("add_defaults", AddDefaults{})     // {quality_profile_id, preset, metadata_profile_id, auto_add:false}
field.JSON("import_options", RootImportOptions{})   // hardlink/reflink allowed, hash_verify, chmod, file_date,
                                                    // recycle_bin, scan_interval (06-import.md)
edge.To("slot_definition", SlotDefinition)?     // null = any slot of the kind
edge.To("default_naming_scheme", NamingScheme)?

// MetadataProfile
field.Enum("media_kind").Values("music","book"); field.String("name")
field.JSON("rules", MetadataRules{}); field.Bool("reserved_none")
unique: (media_kind, name); unique partial: (media_kind) WHERE reserved_none

// Title
field.Enum("kind").Values("series","movie","artist","author","comic_volume","scene","adult_movie")
field.Enum("media_kind")
field.Enum("series_type").Values("standard","daily","anime").Optional()      // TV
field.Bool("use_scene_numbering").Optional()                                  // TV
field.Bool("include_specials").Default(false)                                 // TV
field.Enum("special_version_override").Optional(); field.Bool("special_version_locked").Default(false) // comics
field.JSON("overrides", TitleOverrides{})        // other kind-specific user overrides
edge.To("entity", Entity)?; edge.To("item", Item)?
edge.To("added_by", User)?                        // 10-identity-and-access.md
edge.To("subtitle_profile", SubtitleProfile)?     // 09-subtitles.md; P2
edge.To("tags", Tag).Through("title_tags", TitleTag)
check: exactly one of entity_id, item_id
unique partial: (entity_id) WHERE entity_id IS NOT NULL;  (item_id) WHERE item_id IS NOT NULL

// TitleSlot (A13)
field.String("path")                               // title folder, relative to root_folder
field.Bool("monitored").Default(true)              // pause gate (O1)
field.Enum("minimum_availability").Values("announced","in_cinemas","released").Optional()  // movies
field.Bool("accept_collected").Default(false)      // comics (DOM-093)
field.JSON("pack_policy_override", PackPolicy{})?  // 04-evaluation.md EVL-071
edge.To("delay_profile_override", DelayProfile)?   // 07-profiles.md PRF-073
edge.To("title", Title).Required(); edge.To("slot_definition", SlotDefinition).Required()
edge.To("root_folder", RootFolder).Required(); edge.To("quality_profile", QualityProfile).Required()
edge.To("naming_scheme", NamingScheme)?
unique: (title_id, slot_definition_id); index: (root_folder_id)

// ItemSlot (A13, D20) — the only stored monitoring state.
field.Bool("monitored")
field.Enum("monitored_cause").Values("user","intent","follow","retraction","auto_unmonitor")
field.Enum("coverage").Values("none","partial","complete").Default("none")
field.Int("covered_units").Default(0); field.Int("expected_units").Default(0)   // tracks/parts; 1 otherwise
field.Bool("edition_locked").Default(false)
field.Bool("search_hold").Default(false)           // DOM-083
field.Int64("version").Default(0)                  // DOM-047; grab-transaction concurrency (EVL-093)
edge.To("item", Item).Required(); edge.To("slot_definition", SlotDefinition).Required()
edge.To("title_slot", TitleSlot).Required(); edge.To("target_edition", Edition)?
unique: (item_id, slot_definition_id)
index: (slot_definition_id, monitored, coverage); (title_slot_id)

// Follow (D10 generalised)
field.Enum("add").Values("none","all","released","profile")
field.Enum("monitor").Values("none","all","from_date"); field.Time("from_date").Optional().Nillable()
field.Time("after_date").Optional().Nillable()
field.Enum("gate_action").Values("skip","add_unmonitored").Default("skip")
field.JSON("item_kinds", []ItemKind{})
field.Bool("search_on_add")
edge.To("entity", Entity).Required(); edge.To("slot_definition", SlotDefinition).Required()
edge.To("metadata_profile", MetadataProfile)?; edge.To("gate_tag", Tag)?
edge.To("default_quality_profile", QualityProfile)?; edge.To("default_root_folder", RootFolder)?
edge.To("default_naming_scheme", NamingScheme)?; edge.To("tags", Tag)
unique: (entity_id, slot_definition_id)

// Intent
field.Enum("origin").Values("manual","user_request","import_list","watchlist","follow","disk_import")
field.Enum("status").Values("pending","approved","declined","withdrawn")
field.String("target_key")                  // "e:<id>" | "i:<id>", for uniqueness
field.Enum("preset").Values("all","future","missing","existing","first_group","last_group",
                            "pilot","recent","none","skip").Optional()
field.Bool("include_specials"); field.JSON("overrides", IntentOverrides{})   // profile, root, tags
field.Bool("search_on_fulfil"); field.Bool("ignore_quota"); field.Bool("is_auto")
field.Int("quota_units").Default(0)
field.String("list_entry_ref").Optional()   // provider:namespace:value the list reported
field.String("decline_reason").Optional()
field.Time("decided_at").Optional().Nillable(); field.Time("fulfilled_at").Optional().Nillable()
edge.To("target_entity", Entity)?; edge.To("target_item", Item)?
edge.To("scope_entities", Entity)             // requested seasons; empty = whole target
edge.To("slot_definition", SlotDefinition).Required()
edge.To("title", Title)?                     // set on fulfilment
edge.To("requested_by", User)?; edge.To("decided_by", User)?
edge.To("import_list", ImportList)?; edge.To("follow", Follow)?
edge.To("tags", Tag)
check: exactly one of target_entity_id, target_item_id
index: (target_key, slot_definition_id, status); (title_id, status); (requested_by_id, origin, created_at)
unique partial: (import_list_id, target_key, slot_definition_id) WHERE import_list_id IS NOT NULL

// ImportList
field.String("provider")        // trakt, plex_watchlist, jellyfin_watchlist, imdb, tmdb_list, anilist, mal, simkl,
                                // lastfm, spotify, hardcover, musicbrainz_series, stashdb_filter, custom_json, rss, …
field.Enum("media_kind"); field.String("name"); field.JSON("settings", ListSettings{})   // secrets per D16
field.Bool("enabled"); field.Int("sync_interval_s")
field.JSON("slot_templates", []SlotTemplate{})   // per slot: quality profile, root folder
field.Enum("preset"); field.JSON("follow_policy", FollowPolicy{})?
field.Enum("approval").Values("auto","manual").Default("auto")
field.Enum("retraction").Values("none","log","unmonitor","remove_keep_files","remove_delete_files").Default("none")
field.Time("last_fetch_at").Optional().Nillable(); field.Bool("last_fetch_ok")
edge.To("owner", User)?; edge.To("tags", Tag)

// Exclusion
field.String("provider").Optional(); field.String("namespace").Optional(); field.String("value").Optional()
field.Enum("applies_to").Values("auto_adds","user_requests","both")
field.Enum("reason").Values("manual","removed_with_exclusion","keyword_rule"); field.String("note").Optional()
edge.To("entity", Entity)?; edge.To("item", Item)?; edge.To("created_by", User)?
check: (provider, namespace, value all set) OR entity_id IS NOT NULL OR item_id IS NOT NULL
unique partial: (provider, namespace, value) WHERE value IS NOT NULL; (entity_id) WHERE …; (item_id) WHERE …

// MediaFile (A12)
field.String("rel_path")                    // relative to root folder
field.Int64("size"); field.Time("mtime")
field.JSON("content_key", ContentKey{})     // dev, inode, size, mtime_ns, sampled hash → probe cache (06-import.md)
field.Enum("class")                         // video|audio|ebook|comic_archive (06-import.md FileClass)
field.Enum("state").Values("present","missing"); field.Time("missing_since").Optional().Nillable()
field.Enum("quality_family").Values("video","audio","text","archive")
field.String("quality_id"); field.JSON("quality", QualityModel{})    // per-family axes + revision (04-evaluation.md)
field.JSON("languages", []Lang{}); field.String("release_group").Optional()
field.String("edition_name").Optional(); field.String("scene_name").Optional(); field.String("original_path").Optional()
field.JSON("indexer_flags", Flags{}); field.JSON("transfer", TransferRecord{})   // method, source inode (06-import.md)
field.Int("part").Optional(); field.Int("part_count").Optional()               // multi-file audiobooks
field.Time("imported_at").Optional(); field.Time("last_identify_at").Optional()
field.UUID("download_id").Optional()        // 05-downloads.md Download
edge.To("root_folder", RootFolder).Required()
edge.To("slot_definition", SlotDefinition)?; edge.To("title", Title)?
unique: (root_folder_id, rel_path)
index: (title_id, slot_definition_id); (state, missing_since); (download_id)

// FileLink — M:N file ↔ item.
field.Int("part").Optional()                // audiobook part / music disc
field.JSON("coverage", LinkCoverage{})?     // comics: {from, to} issue range, special version
field.Enum("source").Values("parsed","identified","manual")
edge.To("file", MediaFile).Required(); edge.To("item", Item).Required()
edge.To("edition", Edition)?; edge.To("track", Track)?
unique partial: (file_id, item_id) WHERE track_id IS NULL
unique partial: (file_id, item_id, track_id) WHERE track_id IS NOT NULL
index: (item_id); (track_id)

// Subtitle (DOM-110) — shared by 06-import.md and 09-subtitles.md.
field.Enum("origin").Values("embedded","external","downloaded","uploaded","manual","extracted","translated","generated")
field.String("language"); field.Bool("hi"); field.Bool("forced"); field.String("format")
field.Int("stream_index").Optional().Nillable()             // embedded only
field.String("provider").Optional(); field.String("provider_sub_id").Optional()
field.String("release_info").Optional(); field.String("uploader").Optional()
field.Int("score").Optional().Nillable(); field.Int("score_max").Optional().Nillable(); field.JSON("matches", []string{})
field.JSON("sync", SyncRecord{})?                            // tool, reference, offset_ms, framerate_fixed, checker result
field.JSON("mods", []string{})
edge.To("media_file", MediaFile).Required()                 // ON DELETE CASCADE
edge.To("extra_file", ExtraFile)?                           // sidecars only (06-import.md)
index: (media_file_id, language, forced)

// MediaServer (DOM-111)
field.Enum("kind").Values("plex","jellyfin","emby","kodi","subsonic","audiobookshelf","kavita","komga","stash")
field.String("name"); field.String("url"); field.String("credential").Sensitive()   // write-only (D16)
field.Bool("enabled"); field.JSON("settings", MediaServerSettings{})                 // library/path mappings
unique: (name)

// Tag / TitleTag / AutoTagRule
Tag         { field.String("label"); unique: (lower(label)) }
TitleTag    { edge title, tag; field.Enum("source").Values("manual","rule"); edge.To("rule", AutoTagRule)?
              unique: (title_id, tag_id, source, rule_id) }
AutoTagRule { field.String("name"); field.JSON("conditions", TagConditions{}); field.Bool("remove_when_unmatched")
              edge.To("tags", Tag) }
```

### 3.3 Cross-file references

| Entity (owner) | Edges into this file |
|---|---|
| `Download`, `DownloadTarget`, `Blocklist` (`05-downloads.md`) | `DownloadTarget(item, slot_definition, media_file?)`; a Download has no single slot column: its slots are its targets' slots (D20, DLD-047); `MediaFile.download_id` |
| `SearchIntent`, `SearchLedger` (`05-downloads.md`) | `(item, slot_definition)` |
| `ExtraFile`, `ProbeResult`, `ImportOp`, `NamingScheme` (`06-import.md`) | `ExtraFile.owner_file → MediaFile`, `owner_item → Item` |
| `QualityProfile`, `DelayProfile` (`07-profiles.md`) | QualityProfile referenced by SlotDefinition, TitleSlot, Follow; DelayProfile by the SlotDefinition and TitleSlot overrides (PRF-073) |
| `metadata_doc`, `metadata_refresh` (`08-metadata.md`) | refresh roots point at `Entity`/`Item` |
| `SubtitleProfile`, `SubtitleSearchState`, `SubtitleBlocklist` (`09-subtitles.md`) | `Title.subtitle_profile`; `SubtitleSearchState.media_file`; the `Subtitle` record is owned here (DOM-110) |
| `User`, `Role`, `MediaServerSignInPolicy` (`10-identity-and-access.md`) | `Intent`, `Title`, `ImportList`, `Exclusion`, `SlotDefinition`; `MediaServerSignInPolicy.media_server → MediaServer` (DOM-111) |

---

## 4. Flows

### 4.1 Intent fulfilment

1. Intent created (DOM-065). If the target has no catalog node, create a stub with its `ExternalID` and enqueue a metadata fetch.
2. If `status=pending`, stop; approval (IAM permission) moves it to `approved` in a transaction that continues at step 3.
3. If the target is still a stub, enqueue a durable job `intent.fulfil:<id>` and stop; the job reruns step 4 when the catalog delta completes the stub.
4. In one transaction: ensure Title (layout root); ensure TitleSlot(slot) resolving root/profile/naming (DOM-068); create follows per preset (DOM-082); create ItemSlots for in-scope members; apply the preset; set `fulfilled_at`; set `Title.include_specials` if requested; stamp tags; emit `TitleAdded` / `ItemSlotMonitoringChanged`; enqueue search if `search_on_fulfil`.
5. Failure: the job retries with backoff; the intent stays `approved` and shows `processing`. No failed state.

### 4.2 Follow expansion

1. Trigger: `CatalogDelta.added` for members of E, follow creation, or policy edit (future members only).
2. For each new member m and each Follow F on E (or on an entity linked to m, such as a performer): evaluate `admit(m, F)` (DOM-078).
3. Gate failure with `gate_action=add_unmonitored` → admit unmonitored with `gate_tag`; other failures → skip (nothing stored).
4. Admitted → DOM-080; for TV seasons create the season follow; apply the O7 rule (DOM-083) if not the title's first expansion.
5. Enqueue searches for monitored, released, non-held members if `search_on_add`.

### 4.3 Applying a catalog delta

1. `08-metadata.md` delivers `CatalogDelta` for one refresh root.
2. One transaction (or one durable job keyed by root id):
   - `added` → upsert catalog rows by `ExternalID`, then Flow 4.2;
   - `updated` → update rows; wanted and availability change implicitly;
   - `renumbered` → update `EpisodeInfo`; emit `ItemRenumbered` (a rename plan is proposed by `06-import.md`); FileLinks and ItemSlots are untouched;
   - `tombstoned` → DOM-034;
   - `merged` → DOM-036/037.
3. Recompute totals for affected approved intents and emit `AvailabilityChanged` where the status changed (DOM-091).

### 4.4 Recording an import into several slots (D20)

1. `05-downloads.md` hands import a completed download with targets `{(item, slot)…}`.
2. `06-import.md` identifies units and, per target slot, decides accept/reject independently (DOM-062).
3. For each accepting slot: plan a destination under that slot's TitleSlot; stage via reflink → hardlink → copy (copy when `Mutates`).
4. Per slot, one Record transaction: lock the ItemSlots; insert the slot's `MediaFile` and `FileLink`s; delete retired rows of that slot only; recompute coverage (DOM-059); emit `file.imported{slot}` and `AvailabilityChanged`. All units of one album or audiobook in one slot share one transaction.
5. Failure of one slot's execution (for example `MinimumFreeSpace` on its root) is retried for that slot alone; completed slots stay recorded.

### 4.5 File deletion and missing files

1. User deletes a file (reason `user_delete`), import retires it (`upgrade`), or scan does not find it (`missing_from_disk`).
2. Retire/delete: remove the MediaFile and its links in one transaction; recompute coverage; apply DOM-085.
3. Missing: set `state=missing` (coverage recomputed without it); after the grace period delete the row with reason `missing_from_disk` (never auto-unmonitors). If a whole root is empty or unreachable, raise health and do not mark files missing (`06-import.md`).

### 4.6 List sync and retraction

1. Fetch every enabled list. For each entry resolve or stub the target; create or re-activate `Intent(import_list|watchlist)` (DOM-069), skipping targets with an `auto_adds` exclusion.
2. If any enabled list failed to fetch, stop (no retraction).
3. Otherwise withdraw intents whose `list_entry_ref` is absent, then apply DOM-094 per affected Title.

### 4.7 Worked example: anime with XEM and specials

*Kaiju Academy* (hypothetical). TVDB: S01E01–E24, absolute 1–24; scene groups release E13–24 as "Season 2" E01–12. Specials: S00E01 "Recap" (`aired_before_season=1, aired_before_episode=13`), S00E02 "OVA" (`aired_after_season=1`).

Catalog: `Entity(series)` + `ExternalID(tvdb, series, 400001, active)` + `(anilist, …)`, `(mal, …)`; `Entity(season 0)`, `Entity(season 1)`; 26 `Item(episode)`; XEM fills `scene_season=2, scene_episode=1..12` on S01E13–24; `AltTitle("Kaiju Academy 2nd Season", source=xem, usage=parse, season_scope=2)`.

Library: `Title(series, series_type=anime, use_scene_numbering=true, include_specials=false)`; `TitleSlot(tv/HD, root /tv, path "Kaiju Academy")`; `Intent(manual, preset=all)` creates `Follow(series, HD, add=all, monitor=all)`, `Follow(S1, HD, monitor=all)`, `Follow(S0, HD, monitor=none)`; 26 ItemSlots, S01E01–24 monitored, S00 not. The user monitors the OVA by hand.

When E01–16 have aired and E01–12 have files: `missing` = S01E13–16. S00E02 is monitored but, with `include_specials=false`, only RSS/interactive grabbable (DOM-054). E17–24 are monitored and unreleased: grabbable from RSS, not missing.

Files: `[Grp] Kaiju Academy - 15` (absolute) and `Kaiju Academy S02E03` (scene) both resolve to S01E15 using `EpisodeInfo`. `Kaiju Academy - 13-14` imports as one MediaFile with two FileLinks (exclusive). A later E13-only upgrade retires that file; E14 becomes missing again.

TVDB later splits into S01 (1–12) and S02 (1–12): the delta is `renumbered` ×12 plus `added` (season 2 entity). Episode TVDB ids are unchanged, so FileLinks and flags are untouched; `Follow(series)` admits S2 and creates `Follow(S2, HD, monitor=all)` for future S2 episodes; a rename is proposed. A later request by another user, `Intent(user_request, scope=[S2], slot=HD)`, is a no-op fulfilment that records the requester; `Availability(series, [S2], HD)` is `partial` until all 12 are covered, then `complete` notifies the requester.

### 4.8 Worked example: movie with UHD and HD slots fed by one download

*Example Film (2025)*. Slots `movie/HD` (profile "HD-1080p": WEBDL-1080p < Bluray-1080p, cutoff Bluray-1080p, root `/movies`) and `movie/UHD` (profile "UHD": Bluray-1080p < WEBDL-2160p < Remux-2160p, cutoff Remux-2160p, i.e. 1080p allowed as a fallback, root `/movies-uhd`). Both roots are on the downloads filesystem.

1. User A requests HD (auto-approved): `Intent(user_request, HD, approved)` → `Title(movie item)`, `TitleSlot(HD, /movies, "Example Film (2025)", minimum_availability=released)`, `ItemSlot(movie, HD, monitored)`.
2. User B requests UHD; not auto-approved for UHD, so `pending`. On approval: `TitleSlot(UHD, /movies-uhd, …)`, `ItemSlot(movie, UHD, monitored)`.
3. The digital date passes; both ItemSlots are `missing`. RSS offers `Example.Film.2025.1080p.BluRay.x264-GRP` (torrent, private). Evaluation accepts it in both slots (cutoff met for HD; allowed but below cutoff for UHD); coverage-aware selection makes **one grab** with `DownloadTarget`s `{(movie, HD), (movie, UHD)}` (`04-evaluation.md`, `05-downloads.md`).
4. Import (Flow 4.4) creates two MediaFiles: `/movies/Example Film (2025)/Example Film (2025) - 1080p.mkv` (slot HD) and `/movies-uhd/Example Film (2025)/… .mkv` (slot UHD), both hardlinked from the seeding download. Both ItemSlots become `complete`. `Availability(movie, HD)=complete` notifies A; `Availability(movie, UHD)=complete` notifies B. UHD remains `cutoff_unmet`.
5. Later a Remux-2160p is grabbed for UHD only (HD rejects `QualityNotAllowed`). Its import retires only the UHD MediaFile (one hardlink name removed); the HD file and its inode stay; the seeding torrent still has library references from HD (`05-downloads.md` `library_refs`).
6. If the admin points both TitleSlots at `/movies`, DOM-046 requires the `{Slot}` or quality token, for example `… - 2160p.mkv` next to `… - 1080p.mkv`.
7. The user deletes the HD file with `unmonitor_on_delete` on: `ItemSlot(HD).monitored=false`; UHD is unaffected.

### 4.9 Worked example: multi-disc album in two slots

Artist *The Examples*: `Title(artist)` with `Follow(artist, music/Lossless, add=profile("Studio albums"), monitor=from_date(2026-09-28))` and `Follow(artist, music/Lossy, add=profile("Studio albums"), monitor=none)`. Album *Twin Peaks of Sound* (release group, released after the follow date) has releases R1 (1×CD, 12 tracks), R2 (2×CD deluxe: 12 + 10 bonus), R3 (Digital, 12). It is admitted in both slots; monitored in Lossless; the user monitors Lossy by hand.

- Lossless: the user picks R2, so `ItemSlot(album, Lossless).target_edition=R2, edition_locked=true`, `expected_units=22`.
- Lossy: `target_edition=null`, unlocked.
- 22 FLAC files identified as R2 → 22 `FileLink(file, album, edition=R2, track)`; coverage `complete` at 22/22. If only disc 1 arrives: 12/22 → `partial`, still missing (or the album-atomic rule rejects the set, `06-import.md`).
- FLAC + CUE, one image per disc: file 1 links 12 tracks, file 2 links 10 (file:track 1:N); same coverage.
- 12 MP3s identified as R3 into Lossy: unlocked, so `target_edition=R3`; coverage `complete`. Lossless keeps R2. A FLAC download is never imported into Lossy because the Lossy profile does not allow FLAC; one download feeds both slots only when both profiles accept it (DOM-062).
- MusicBrainz merges R2 into R2′: R2's MBID becomes `ExternalID(state=merged, merged_into_value=R2′)`; Tracks, FileLinks and `target_edition` follow the fold; files tagged with the old MBID still resolve.

### 4.10 Worked example: comic TPB covering #1–6

ComicVine: V1 *Star Rangers (2021)* with issues #½, #1–#12 (`sort_key` 0.5, 1…12); V2 *Star Rangers Vol. 1: First Light*, `special_version=tpb`, one issue item.

`Title(V1)` has `TitleSlot(comic/Digital)` and `Follow(V1, add=all, monitor=all)`.

1. `Star Rangers 001-006 (2021) (Digital)` imports into V1 as one MediaFile with six FileLinks (`coverage {from:1, to:6}`), overlapping mode; six ItemSlots `complete`. A later single `Star Rangers 003` also imports: #3 has two covering files, which is allowed.
2. `Star Rangers Vol. 1 TPB` matches V2, not V1 (DOM-025). If the user adds `Title(V2)`, it covers V2#1.
3. With `ItemRelation(V2#1 collects V1#1..#6)` and `TitleSlot(V1, Digital).accept_collected=true` (off by default), `missing` and availability treat V1#1–6 as covered when V2#1 is covered in the same slot (DOM-093). No FileLink is created; the TPB file stays in V2's folder and naming.

---

## 5. Non-goals

| Not done | Reason |
|---|---|
| Stored monitored flags at season, series, album or release level | Replaced by `TitleSlot.monitored` and Follows; the decision engine only needs parent ∧ item (O1). |
| Stored availability/status mirrors (Seerr `status`/`status4k`, scanners, `availabilitySync`) | Availability is a query plus events (DOM-087, DOM-091). |
| Multi-instance 4K or ebook/audiobook setups, instance-to-instance lists | Slots (D4, D20). |
| A FAILED request state | Fulfilment is in-process and retried (DOM-067). |
| Library-wide list cleanup (Sonarr/Radarr `CleanLibrary`) | Retraction is scoped to list-only titles (DOM-094). |
| Materialised gate exclusions (Whisparr v3) | Gates are predicates (DOM-079). |
| Metadata profiles deleting catalog rows | Profiles filter membership (DOM-104). |
| Provider-typed id columns (`TvdbId`, `ForeignAuthorId`, `Old*Ids`) | One `ExternalID` map (DOM-027). |
| Product codes in the unique id map | Non-unique (A14, DOM-031). |
| Absolute file paths | Root-relative (A12, DOM-056). |
| One file shared by two slots as a single row | Each slot owns its own MediaFile (D20, DOM-062). |
| Per-title alternate TV orderings in v1 | Deferred (DOM-017). |
| Whisparr v2 site/year/scene model | D10 chose v3. |

---

## 6. Traceability

| Reference | Preserved | Changed (D19) |
|---|---|---|
| Sonarr | Monitoring gates (O1), missing definition (O2), presets (O5), new-season policy and S0 rule (O6), no re-monitor on refresh (O7), auto-unmonitor exceptions (O8), multi-episode files, TVDB/absolute/scene numbering and specials placement (O18) | Season flag → Follow; `UnmonitorReaddedEpisodes` → search hold + event; `CleanLibrary` scoped; `KeepAndTag` → derived filter |
| Radarr | `MinimumAvailability` rule (O3), collections auto-add (O12), Deleted status (O19) | `MovieMetadata` split → catalog/library; 4K instances → slots |
| Lidarr | Album as unit of want, track completeness (O4), one target release with `AnyReleaseOk` (O10), metadata profiles (O11), MBID aliases (O17) | Target release per slot; "New" → from-date; profiles never delete catalog rows |
| Bookshelf | Edition model, multi-part audiobooks (O9), `AnyEditionOk` | Edition per slot (ebook vs audiobook in one instance); book series followable |
| Whisparr v3 | Scene/movie items, followable studios/performers with AfterDate (O13), multi-provider ids | Gates as predicates; no auto-created library rows |
| Whisparr v2 | Non-unique product codes (O20), ambiguity rejection | – |
| Kapowarr | M:N issue files, special versions with override/lock (O21), `monitor_new_issues` | Collected-edition coverage as opt-in derivation |
| Seerr | Request states, per-season requests, availability semantics (O15, O16), requester notification | `is4k` → slot; status mirrors → queries; FAILED dropped; watchlists unified |
