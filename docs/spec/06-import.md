# 06. Import

This file specifies everything from "files are on local disk" to "organised, identified and recorded library files with their extras": the import pipeline (Discover → Probe → Group → Identify → Aggregate → Decide → Plan → Execute → Post-import), the per-kind interfaces, identification per media kind, typed rejections, slot selection including import of one download into several slots (D20), the crash-safe journal, transfer modes, probing, the naming engine, tag writing and content transforms, extras and metadata writers, subtitle sidecar indexing, the media-server refresh, scanning and watching, library import of existing folders, the recycle bin and the filesystem requirements. The download lifecycle that hands files to import is `05-downloads.md`; quality models and custom-format scoring are `04-evaluation.md`; the entities `MediaFile`, `FileLink`, `RootFolder` and the shared `Subtitle` record are owned by `02-domain-model.md`; `ExtraFile`, `ProbeResult`, `NamingScheme` and the import journal are owned here. Phases: **P1** (pipeline, TV, movies, naming, transfer, journal, video probing, subtitle sidecar indexing and stream flags, scanning, library import, Kodi/Plex writers, media-server refresh), **P2** (music identification, TagLib probing and tag writing, fingerprinting), **P3** (books, adult, comics, their writers and transforms).

## Concepts

| Term | Definition |
|---|---|
| ImportRequest | The input to one pipeline run: a source, paths, optional grab context and transfer constraints. |
| Source | Where the request comes from: `download`, `rescan`, `library_import`, `manual`, `watch`. |
| FileClass | The class of a discovered file: `video`, `audio`, `ebook`, `comic_archive`, `subtitle`, `image`, `nfo`, `lyric`, `playlist_cue_log`, `archive`, `executable`, `dangerous`, `other`. |
| ContentKey | `(device, inode, size, mtime_ns)`, with a sampled xxhash fallback; the key of the probe cache. |
| ProbeResult | The cached, schema-versioned output of one prober for one ContentKey. |
| ImportUnit | The atomic unit of identification and decision produced by a Grouper: one video file, one album, one book edition per medium, one comic archive. |
| Identification | An Identifier's result for a unit: `Matched`, `Ambiguous`, `NoMatch` or `Unparseable`, with targets, ranked candidates and evidence. |
| FileFacts | Aggregated per-file facts: quality, revision, languages, release group, edition, scene name, indexer flags, custom-format score. |
| Placement | One accepted `(unit, items, slot)` combination: one unit placed into one slot. One unit **MAY** have several placements (D20). Each placement yields its own MediaFile(s). |
| Rejection | A typed decision against a file, unit, placement or request: `{code, class, scope, benign, params}`. |
| ImportPlan | The pure output of Decide + Plan: placements, destination paths, transfer methods, retire sets, extras, transforms, refreshes. A preview is a plan that is not executed. |
| ImportOp | One journaled filesystem operation of a plan, with a deterministic id and a state machine. |
| Retire set | Existing files (and their extras) in a slot that a placement replaces. |
| TokenSet | A per-kind set of naming tokens and template slots. |
| NamingScheme | A named set of templates plus rename options, chosen per slot. |
| Transfer method | `rename`, `reflink`, `hardlink`, `copy`, `in_place`. |
| Recycle bin | Where retired files go before purge. |

Entity names follow `02-domain-model.md` (`Title`, `Item`, `SlotDefinition`, `TitleSlot`, `ItemSlot`, `MediaFile`, `FileLink`, `RootFolder`, `Edition`, `Track`). "Slot" means a `SlotDefinition` as configured on a `TitleSlot`. Sonarr's "import decision" is a Decide-stage result here; Sonarr's `ImportRejectionReason` values map to Rejection codes (IMP-030).

## Requirements

### 1. Pipeline and interfaces

**IMP-001** (MUST) All media kinds **MUST** share one pipeline with the stages Discover, Probe, Group, Identify, Aggregate, Decide, Plan, Execute and Post-import. Per-kind logic **MUST** live only behind the interfaces `Grouper`, `Identifier`, `Prober` and `TokenSet`, plus kind-specific decision specs registered with the shared decision framework. Discovery, aggregation plumbing, decision framework, planning, transfer, journal, extras, events and refresh **MUST** be kind-neutral. _Source: design/import.md §3.1.1, §3.2._

**IMP-002** (MUST) An `ImportRequest` **MUST** carry:

```go
type ImportRequest struct {
    ID         uuid.UUID
    Source     Source                  // download | rescan | library_import | manual | watch
    Paths      []LocalPath             // local paths only; remote mappings already applied
    DownloadID *uuid.UUID
    Grab       *GrabContext            // targets [](item, slot), release snapshot, parsed facts; download sources only
    Payload    PayloadOwnership        // client_seeding | client_done | owned | in_place (05-downloads.md DLD-083)
    Mode       ImportMode              // auto | move | copy | in_place
    Overrides  []RejectionCode         // codes accepted by the reason policy or the user
    ScopeItem  *ItemRef                // manual import / rescan restricted to one item or container
    AllowRemote bool                   // may call metadata CatalogLookup (library import, rescan)
}
```

The `ImportOutcome` returned to the caller **MUST** carry a status (`complete | partial | blocked | failed`), per-target results (`imported(media_file_ids, transfer methods) | not_wanted | rejected(rejections) | skipped`) and per-file rejections. _Source: design/import.md Boundaries, §3.2 stage 9; design/downloads.md §3.9.1; D20._

**IMP-003** (MUST) The per-kind interfaces **MUST** have these shapes (names normative, signatures indicative):

```go
type Prober interface {
    Kinds() []MediaKind
    Classes() []FileClass
    Tools() []Capability                 // native tools needed, e.g. "ffprobe", "fpcalc" (A6)
    SchemaVersion() int
    Probe(ctx context.Context, path string) (ProbeResult, error)
}
type Grouper interface {
    Kind() MediaKind
    Group(ctx context.Context, files []DiscoveredFile, hints Hints) ([]ImportUnit, error)
}
type Identifier interface {
    Kind() MediaKind
    // Must return Ambiguous rather than guess (IMP-026).
    Identify(ctx context.Context, u *ImportUnit, hints Hints) (Identification, error)
}
type TokenSet interface {
    Kind() MediaKind
    Tokens() []TokenDef                  // IMP-065
    Templates() []TemplateSlot           // IMP-064
}
type Hints struct {
    Grab        *GrabContext
    ScopeItem   *ItemRef
    AllowRemote bool
}
type Identification struct {
    Status     IdentStatus                // Matched | Ambiguous | NoMatch | Unparseable
    Targets    []Target                   // file index → item(s), part, coverage
    Candidates []Candidate                // ranked, with distance/confidence and evidence
    Evidence   []Evidence
}
```

_Source: design/import.md §3.2._

### 2. Discover

**IMP-004** (MUST) Discover **MUST** walk the request's paths and classify every file into a `FileClass` by extension, and by magic bytes for archives and comic archives (correcting wrong extensions). _Source: design/import.md §3.2 stage 1._

**IMP-005** (MUST) The extension registry used by Discover **MUST** be the single registry of file classes for the product; the download content policy (`05-downloads.md` DLD-062) **MUST** use it. _Source: design/downloads.md §3.7; design/import.md §3.2._

**IMP-006** (MUST) Discover **MUST** reject executables with `ExecutableFile`, dangerous files with `DangerousFile`, and extensions on the user's deny list with `UserRejectedExtension`, each at file scope. If no media-class file remains (or only samples remain after Decide), the request **MUST** get a request-scope `NoMediaFiles` rejection with a `samples` count parameter. _Source: design/import.md §3.2 stage 1; design/downloads.md §3.7 item 3._

**IMP-007** (MUST) For `rescan` and `watch` sources, files already recorded with the same `(root_id, rel_path, size, mtime)` **MUST** be skipped. _Source: design/import.md I19._

**IMP-008** (MUST) Discover **MUST** honour configured ignore globs and `.bonnyreadignore` files (gitignore syntax, scoped to their directory). _Source: design/import.md §3.2 stage 1._

**IMP-009** (MUST) For download sources the lifecycle guarantees completion and extraction; the working-folder check (`_UNPACK_`, `_FAILED_` prefixes) **MUST** apply only to disk-sourced requests and **MUST** yield `Unpacking` (transient). _Source: design/import.md §3.2 stage 1 (Sonarr `NotUnpackingSpecification`)._

### 3. Probe

**IMP-010** (MUST) Each media file **MUST** be probed at most once per ContentKey and prober schema version. Results **MUST** be stored in the database probe cache and shared by every consumer: quality augmenters, custom-format evaluation, sample detection, `{MediaInfo …}` tokens, the subtitle subsystem's embedded-track view and the UI. Consumers **MUST NOT** run their own probes. _Source: design/import.md §3.1.3, §3.2 stage 2._

**IMP-011** (MUST) The ContentKey **MUST** be `(device, inode, size, mtime_ns)`. On filesystems without stable inodes the key **MUST** fall back to `(size, mtime_ns, xxhash of the first, middle and last MiB)`. _Source: design/import.md §3.2 stage 2._

**IMP-012** (MUST) A prober schema-version bump **MUST** cause re-probing lazily, when a consumer next reads a stale result or a background job reaches the file, and **MUST NOT** probe the whole library at start-up. _Source: design/import.md §3.2 stage 2._

**IMP-013** (MUST) The probers **MUST** be:

| Prober | Kinds | Implementation | Phase |
|---|---|---|---|
| `VideoProber` | TV, movie, adult | bundled `ffprobe` executable; a deep second pass only when the first yields no streams | P1 |
| `AudioProber` | music, audiobooks | TagLib (read and write); binding chosen by spike | P2 |
| `EbookProber` | books | EPUB OPF (zip + XML), MOBI/AZW3 EXTH (port of Bookshelf's `AzwTag`), PDF Info dictionary | P3 |
| `ComicProber` | comics | archive listing (zip; pure-Go RAR/7z via xtractr dependencies), page count, embedded `ComicInfo.xml` | P3 |
| `Fingerprinter` | music, on escalation only | `fpcalc` (Chromaprint) | P2 |

[OPN-121] _Source: design/import.md §3.2 stage 2; A6._

**IMP-014** (MUST) `VideoProber` output **MUST** include container; video streams (codec, resolution, HDR format, bit depth); audio streams (codec, channels, language); **subtitle streams (language, codec, and the forced, default and hearing-impaired disposition flags)**; runtime; chapters. Subtitle stream flags are phase-1 groundwork for `09-subtitles.md`. _Source: A17; design/import.md §3.2 stage 2._

**IMP-015** (MUST) Probe and fingerprint jobs **MUST** declare the native tools they need, and **MUST** be claimed only by nodes that advertise those tools. Desktop installers and container images **MUST** bundle `ffprobe` and `fpcalc`. If no live node has a required tool, a health issue **MUST** be raised and the affected imports **MUST** wait (transient `ProbeFailed` with a `tool_missing` parameter) rather than fail. _Source: A6._

### 4. Group

**IMP-016** (MUST) Groupers per kind:
- **Video** (TV, movie, adult): one unit per video file, carrying its folder context (season pack or single release).
- **Music** (P2): album units: group by directory, merge multi-disc subfolders (`CD1`, `Disc 2`, …), then split by album tag, following Lidarr `TrackGroupingService`.
- **Books** (P3): one unit per edition **per medium**; ebook files and audio files **MUST NOT** share a unit; a multi-file audiobook is one unit with parts ordered by track number, then disc, then natural filename order.
- **Comics** (P3): one unit per archive, with the volume folder as context.
_Source: design/import.md §3.2 stage 3._

**IMP-017** (MUST) [P2] A FLAC+CUE image **MUST** be split into tracks before grouping (as a staged transform, IMP-071), unless the download lifecycle already split it (`05-downloads.md` DLD-074). _Source: design/import.md §3.6 Other transforms._

### 5. Identify

**IMP-018** (MUST) For download sources, the grab targets **MUST** be authoritative for which items the payload is for, and **MUST** still be checked: a unit that parses to a different item than the grab **MUST** be rejected with `GrabMismatch`, and one whose folder parses differently with `FolderMismatch` (both overridable, `05-downloads.md` DLD-084). _Source: design/import.md I2; design/downloads.md §3.9.1._

**IMP-019** (MUST) Download imports **MUST** identify against the local library only and **MUST NOT** add unknown titles. Rescans and library import **MAY** call metadata `CatalogLookup` (`08-metadata.md`), through the shared rate limiter and cache (A3). _Source: design/import.md §3.2 stage 4._

**IMP-020** (MUST) TV **MUST** use a `ParseIdentifier`: the release parser, then series by external id or scene title, then episodes through scene, XEM, anime absolute and daily air-date numbering, as a port of Sonarr's `ParsingService`. For download sources both the release name and the file name **MUST** be parsed and **MUST** agree. Results **MUST** match the ported Sonarr parsing and import test corpus. _Source: design/import.md §3.2 stage 4, I3; D1a._

**IMP-021** (MUST) Movies **MUST** use a `ParseIdentifier` in Radarr's order: ids in the name, then TMDb/IMDb, then title + year (± secondary year), then title. Editions come from the parser. _Source: design/import.md I3._

**IMP-022** (MUST) [P3] Adult content **MUST** use a `ParseIdentifier` with a graded matcher in the order StashDB id, indexed product code, studio + date with graded levels. The fuzzy pass **MUST** apply the ambiguity rule (IMP-026); a tie at the best level **MUST** be `Ambiguous`. _Source: design/import.md I3; D10._

**IMP-023** (MUST) [P2] Music **MUST** use a `DistanceIdentifier`: candidates from tag MBIDs, then the grab, then local fuzzy artist/album, then AcoustID recordings; weighted normalised distance with Lidarr's weights (including the 10 s track-length grace); an optimal track↔file assignment (Hungarian algorithm); and thresholds album ≤ 0.20, worst track ≤ 0.40. Fingerprinting **MUST** run only when the best distance is above 0.15 or the worst track above 0.40. Results **MUST** pass Lidarr's identification fixtures (`NzbDrone.Core.Test/Files/Identification/*.json`). _Source: design/import.md I4._

**IMP-024** (MUST) [P3] Books **MUST** use a `DistanceIdentifier` with Bookshelf's weight table (ISBN and ASIN weight 10, ebook/audio format bias), candidates in the order ISBN, ASIN, provider-id tags, author + title, title, author, and a match threshold of 0.30. Results **MUST** pass Bookshelf's identification fixtures. [OPN-029] _Source: design/import.md I4, §6.2 Q2; open-issues C (Import)._

**IMP-025** (MUST) [P3] Comics **MUST** use a `CoverageIdentifier`: the comic sub-parser yields volume, issue number(s), range and special version (TPB, one-shot, hardcover, omnibus, volume-as-issue), mapped to issues with `FileLink.coverage`. Results **MUST** pass Kapowarr's filename test corpus (`extract_filename_data`, `refine_special_version`). Whether a collected edition also satisfies the singles it collects is a library setting, off by default (`02-domain-model.md`). [OPN-038] _Source: design/import.md I5; design/library.md §4.4._

**IMP-026** (MUST) If the best two candidates resolve to **different items** and their scores are within the kind's margin, the result **MUST** be `Ambiguous` with rejection `AmbiguousMatch` (needs user). Margins: a distance difference ≤ 0.05 for distance kinds; the same match level for graded matching. Candidates that are different releases or editions of the **same item** **MUST NOT** be treated as ambiguous; the choice between them follows the slot's release/edition lock (`ItemSlot.edition_locked`; Lidarr `AnyReleaseOk`). [OPN-122] _Source: design/import.md §3.2 stage 4, §6.1 item 3; D10; A13._

**IMP-027** (MUST) When a distance identifier chooses a release or edition other than the slot's current target and the slot is not locked, the plan **MUST** switch the slot's target edition/release (Lidarr `ImportApprovedTracks` behaviour), recorded in the same Record transaction. _Source: design/import.md §4.2 step 5; A13 (edition belongs to the slot)._

### 6. Aggregate

**IMP-028** (MUST) Aggregate **MUST** build `FileFacts` per file with quality from confidence-ordered sources: grab release name, file name, folder name, client item name, probe. The probe **MUST** win for resolution and codec; names **MUST** win for source (for example WEB-DL vs Blu-ray). The probe **MUST** win for audio languages. Revision, release group, edition, scene name and indexer flags **MUST** come from the grab when present. The custom-format score **MUST** be computed by the evaluation engine on these facts (`04-evaluation.md`). Attribute detection **MUST** agree with the ported Sonarr/Radarr parser tests (D20 attribute authority). _Source: design/import.md I10, §3.2 stage 5; D20._

### 7. Decide and typed rejections

**IMP-029** (MUST) Every decision spec **MUST** return `Accept` or a `Rejection`:

```go
type Rejection struct {
    Code   RejectionCode      // stable enum; part of the API and event contract
    Class  RejectionClass     // transient | permanent | needs_user
    Scope  RejectionScope     // file | unit | placement | request
    Benign bool               // does not prevent a "complete" outcome (Sample, AlreadyImported)
    Params map[string]string  // e.g. {"missing":"12","disc":"2"}, {"existing":"Bluray-1080p","new":"WEBDL-1080p"}
}
```

User-visible reasons **MUST** be rendered from the code and params through localisation; free-text reasons **MUST NOT** be part of the contract. _Source: design/import.md §3.2 stage 6._

**IMP-030** (MUST) The rejection codes **MUST** include at least (T = transient, P = permanent, NU = needs user; b = benign):

| Group | Codes |
|---|---|
| Source | `FileLocked` (T), `SourceMissing` (P), `SourceNotAccessible` (NU; raises a health issue pointing at path mappings or mounts), `Unpacking` (T), `UnsupportedExtension` (P), `UserRejectedExtension` (P), `DangerousFile` (P), `ExecutableFile` (P), `ArchiveFile` (P), `NoMediaFiles` (P, request), `ProbeFailed` (NU), `NoAudioTrack` (P), `Sample` (P, b), `SampleIndeterminate` (NU) |
| Identification | `Unparseable` (NU), `UnknownItem` (NU), `AmbiguousMatch` (NU), `GrabMismatch` (NU), `FolderMismatch` (NU), `UnverifiedSceneNumbering` (NU), `MissingAbsoluteNumber` (NU), `TitleRequired` (T), `PartialSeason`, `FullSeasonRequired`, `MultiSeason`, `SplitEpisode` (NU), `DistanceTooHigh` (NU), `MissingTracks` (NU), `UnmatchedFiles` (NU), `FewerTracksThanExisting` (NU), `WrongMedium` (P) |
| Slot | `NoSlotAccepts` (P, placement or unit), `SlotAmbiguous` (NU) |
| Upgrade | `AlreadyImported` (P, b), `NotQualityUpgrade`, `NotRevisionUpgrade`, `NotCustomFormatUpgrade` (P, placement), `ExistingFileCoversMoreItems` (P, placement) |
| Destination | `MinimumFreeSpace` (T), `RootFolderMissing` (T, becoming NU after the transient budget), `DestinationOccupied` (NU) |

New codes **MAY** be added (additive API change, A7); existing codes **MUST NOT** change meaning. The download-level reason classes are assigned in `05-downloads.md` DLD-084. _Source: design/import.md §3.2 stage 6 (collapsing Sonarr's 37 `ImportRejectionReason` values)._

**IMP-031** (MUST) Sample detection **MUST** compare the probed runtime against the item's expected runtime using Sonarr's `DetectSample` tiers, and **MUST** pass Sonarr's sample-detection tests. A file whose runtime cannot be read **MUST** get `SampleIndeterminate`. A sample **MUST** be excluded before duplicate-target checks. _Source: design/import.md I6._

**IMP-032** (MUST) A placement **MUST** replace existing files in its slot only if it is better by quality, then revision, then custom-format score, using the evaluation comparator (`04-evaluation.md`). Music **MUST** decide upgrades at album level (`AlbumUpgradeSpecification`, `MoreTracksSpecification`). _Source: design/import.md I7._

**IMP-033** (MUST) [P2] A new download of an album **MUST** be all or nothing: missing or unmatched tracks **MUST** reject the unit (`MissingTracks`, `UnmatchedFiles`) unless the user explicitly allows a partial import in manual import. _Source: design/import.md I8._

**IMP-034** (MUST) TV imports **MUST** support Sonarr's episode-title-required setting (`always | bulk_season_releases | never`): while the episode title is unknown ("TBA"), the unit **MUST** be rejected with `TitleRequired` (transient) until the title appears or a timeout (default 48 h after the first attempt) elapses, after which it imports. [OPN-035] _Source: design/import.md §6.2 Q8; open-issues C (Import)._

**IMP-035** (MUST) Kind-specific specs **MUST** register with the framework rather than fork it: Sonarr's season-pack specs, Lidarr's `MoreTracks` and `NoMissingOrUnmatched`, Bookshelf's medium check. _Source: design/import.md §3.2 stage 6._

### 8. Slot selection and multiple slots (D20)

**IMP-036** (MUST) For download sources, the candidate slots for a unit **MUST** be exactly the slots named in the grab targets for the unit's items. The Decide stage **MUST** evaluate each candidate slot independently (quality allowed in that slot's profile, upgrade against that slot's files, destination checks) and produce one placement per accepting slot, or a placement-scope rejection per rejecting slot. _Source: D20; open-issues B1(a); design/import.md §3.2 Slot selection._

**IMP-037** (MUST) One unit **MAY** be placed into several slots. Each placement **MUST** produce its own MediaFile(s) under that slot's root folder, named with that slot's naming scheme, with `MediaFile.slot` set to that slot. A MediaFile **MUST** belong to exactly one slot. _Source: D20 ("each slot gets its own library file"); design/library.md §3.1 (MediaFile belongs to exactly one slot)._

**IMP-038** (MUST) For non-download sources (rescan, watch, library import, manual without an explicit slot), a file **MUST** be placed into exactly one slot:
1. the slot of the root folder, if the root is bound to a slot;
2. otherwise the item's slots whose profile allows the file's quality (quality axes are per kind, so an EPUB fits only an ebook slot); among these, an empty slot or one where the file is an upgrade; then the slot whose cutoff the file fits most closely;
3. a remaining tie **MUST** give `SlotAmbiguous`.
When several slots share one root folder, this inference by profile fit applies to scanned files. [OPN-046] _Source: design/import.md §3.2 Slot selection; design/library.md §3.10; open-issues C (Library)._

**IMP-039** (MUST) In manual import the user **MUST** be able to choose one or several slots for a unit; each chosen slot is a placement subject to the same specs. _Source: D20._

**IMP-040** (MUST) Two placements of one plan **MUST NOT** resolve to the same destination path. A collision with another placement or an existing unrelated file **MUST** give `DestinationOccupied` for the later placement. Configuration validation **MUST** warn when two TitleSlots of one title share a root folder and their naming schemes can render identical paths. _Source: D20 (consequence of per-slot files); design/import.md §3.2 Destination codes._

### 9. Plan

**IMP-041** (MUST) Deciding and planning **MUST** be a pure function of (files, probes, library state, settings) → `ImportPlan`. For every placement the plan **MUST** contain: the destination path(s) from the naming engine using the slot's root folder and scheme; the transfer method (IMP-049); the retire set (existing files in that slot the placement replaces, and their extras); the extras to carry; the content transforms; the metadata extras to write; and the media servers to refresh. _Source: design/import.md §3.1.2, §3.2 stage 7._

**IMP-042** (MUST) The plan **MUST** be persisted as `ImportOp` rows before any filesystem change. _Source: design/import.md §3.2 stage 7._

**IMP-043** (MUST) Manual import, rename previews and retag previews **MUST** be plans that are not executed. The user **MAY** edit a plan (another item, slot(s), release or edition, partial acceptance, `AllowPartial` on a unit); an edited plan **MUST** re-run Decide and then execute through the same executor. The preview **MUST** show the transfer method and any forced copy. _Source: design/import.md §3.2 stage 7, §4.2 step 3; §5 "Manual import and previews"._

### 10. Execute and the crash-safe journal

**IMP-044** (MUST) Every filesystem mutation (import, rename, retag, move, delete) **MUST** be an `ImportOp` with a deterministic id and the state machine `planned → staged → transformed → retired → committed → recorded → cleaned → done`, with `failed` reachable from any state and triggering compensation:

| Step | Action | Postcondition checked on replay |
|---|---|---|
| Stage | Transfer the source to `<dest dir>/.<name>.br-<opid>.partial` with the chosen method. For a same-filesystem move, create a hardlink at the staging name so both names exist until commit; if hardlinks are unsupported, the rename happens at Commit | staged file exists with expected size |
| Transform | Apply tags, ComicInfo.xml, conversions to the staged copy only | transform marker hash recorded |
| Retire | Rename each retired file and its extras to the recycle bin, or with no recycle bin to `.<name>.br-<opid>.retired` beside it | old paths absent, retired paths recorded |
| Commit | `rename(staged, final)` (atomic on the same filesystem) | final path exists and its inode matches the staged one |
| Record | One DB transaction: upsert MediaFile, FileLink and ExtraFile rows; delete retired rows; update `ItemSlot.coverage`; write history [OPN-203] and outbox events; set op `recorded` | rows exist |
| Clean | Unlink the move source (if not retained), delete `.retired` files, remove empty source directories | source absent |

_Source: design/import.md §3.1.2, §3.8 Journal._

**IMP-045** (MUST) Staging **MUST** complete before any retire step, so that an item never loses its file to a crash or a failed copy. Deletion without a recycle bin **MUST** happen only in Clean, after Record. _Source: design/import.md §3.8, §2.2.6._

**IMP-046** (MUST) All units of one album or one multi-part audiobook **MUST** be recorded in one transaction, so that partial albums never appear. Placements in different slots **MAY** record in separate transactions; their target outcomes are independent. _Source: design/import.md §3.8 Record; D20._

**IMP-047** (MUST) A recovery job **MUST** run at start-up and whenever an import job is reclaimed after a node failure. It **MUST** walk non-`done` ops and roll each forward where its postconditions allow, and otherwise compensate by restoring retired files to their original paths and deleting staged files. Each step **MUST** check its postcondition first, so that replay is idempotent. _Source: design/import.md §3.8 Recovery; A1._

**IMP-048** (MUST) Before writing into a root folder, an import job **MUST** verify the root's `.bonnyread-root` marker file on the executing node; a mismatch **MUST** give `RootFolderMissing`. _Source: design/runtime.md §3.15._

### 11. Transfer modes

**IMP-049** (MUST) The transfer method **MUST** be resolved by trying options in order and falling back on `EXDEV`, `ENOTSUP` or `EPERM`:
- **retain source** (`client_seeding`, or Copy mode): reflink (`FICLONE` on Linux; `clonefile` on macOS), then hardlink, then copy;
- **release source** (`client_done`, `owned`, Move mode): rename on the same filesystem, then reflink plus unlinking the source, then verified copy plus unlinking the source;
- **in place** (library import, rescan): no transfer.
Reflink **MUST** be attempted by trying the clone, never by checking filesystem names. _Source: design/import.md I11, §3.5; §2.2.8._

**IMP-050** (MUST NOT) A hardlink **MUST NOT** be used for a placement whose plan mutates content (tags, ComicInfo.xml, conversion, split) or applies `chmod`/`chgrp` or file dates. On a filesystem without reflink this **MUST** fall back to a real copy, and the plan preview **MUST** show the extra disk use. _Source: design/import.md §3.5 (`Mutates`), §2.2.7._

**IMP-051** (MUST) A retag, transform or permission change of an existing library file whose link count is greater than 1 **MUST** first replace it with an unshared copy (through the journal), so that the change never reaches a client's seeding copy or another slot's file. _Source: design/import.md §3.6 Tag writing; D20._

**IMP-052** (MUST) When one source feeds several placements (D20):
- with a retained source, every placement **MUST** use the retain-source order independently;
- with a released source, all placements but one **MUST** use the retain-source order, and the executor **MUST** perform the release-source transfer **last**, choosing a placement on the source's filesystem when one exists (ties broken by `SlotDefinition.position`);
- the Clean step that unlinks the source **MUST** run only after every placement from that source is `recorded`.
_Source: D20; design/import.md §3.5._

**IMP-053** (MUST) Copies **MUST** use `copy_file_range` where available (via Go's `ReadFrom`) and **MUST** be verified by size. Hash verification **MUST** be a per-root option, on by default for network roots (NFS, SMB/CIFS) and off otherwise. [OPN-034] _Source: design/import.md I12, §3.5, §6.2 Q7; open-issues C (Import)._

**IMP-054** (MUST) A case-only rename **MUST** go through a temporary name. _Source: design/import.md I12._

**IMP-055** (MUST) The method actually used, the source path and the source inode **MUST** be recorded on the MediaFile (`transfer`) and included in `file.imported`, so that the download lifecycle can maintain links without scanning. _Source: design/import.md §3.5; design/downloads.md §3.10._

**IMP-056** (MUST) `chmod`/`chgrp` for new files and directories (default: inherit the umask) and the file date (`none | air_or_release_date | import_time`) **MUST** be per-root options applied after the method is known, and **MUST NOT** be applied to hardlinked destinations (the plan warns instead). Empty source folders **MUST** be removed after a move. _Source: design/import.md §3.5 Permissions and dates._

### 12. Post-import and outcome

**IMP-057** (MUST) After Record, Post-import **MUST** write metadata extras (IMP-074), emit events through the outbox (in the Record transaction), enqueue a debounced media-server refresh, and return the `ImportOutcome`. _Source: design/import.md §3.2 stage 9._

**IMP-058** (MUST) The outcome status **MUST** be `complete` when every expected target is imported or rejected only for benign reasons; `partial` when some targets are imported and others rejected; `blocked` when no target is imported and a needs-user rejection exists; `failed` when no target is imported and only permanent rejections exist. Transient rejections **MUST** be retried by the import job with backoff (for example `FileLocked`, `MinimumFreeSpace`, `TitleRequired`) before the outcome is reported. For download sources the lifecycle's reason policy decides the download's phase (`05-downloads.md` DLD-086). _Source: design/import.md §3.2 stage 9._

### 13. Naming engine

**IMP-059** (MUST) There **MUST** be one naming engine: `Render(template, NamingContext) → (path, []Warning)`, a pure function used by previews, validation, sample output, import and rename. _Source: design/import.md §3.4._

**IMP-060** (MUST) Templates **MUST** follow this grammar (Sonarr's `TitleRegex` contract, I13):

```ebnf
template   = { literal | escape | token } ;
escape     = "{{" | "}}" ;                         (* literal braces *)
token      = "{" prefix name [ ":" args ] suffix "}" ;
prefix     = { " " | "." | "-" | "_" | "[" | "(" } ;
suffix     = { " " | "." | "-" | "_" | "]" | ")" } ;
name       = word { sep word } ;                   (* "Series Title", "series.title", "Release-Date" *)
sep        = " " | "." | "-" | "_" ;
word       = alnum { alnum } ;
args       = argchar { argchar } ;                 (* [ ,a-z0-9+-], case-insensitive; no trailing space or dash *)
literal    = any character except "{" and "}" ;
```

The grammar **MUST** be covered by a fuzz test and by the ported Sonarr and Radarr `FileNameBuilder` test suites, which are the acceptance tests. _Source: design/import.md I13, §3.4 Grammar._

**IMP-061** (MUST) Rendering **MUST** apply, in order:
1. **Lookup:** normalise `name` (case-folded, separators removed) and look it up in the kind's TokenSet, then the common set. Unknown tokens **MUST** be a validation error at save time and render literally with a warning at run time. Aliases **MUST** exist only for names used by published presets (Sonarr and Radarr names).
2. **Value:** a typed value (string, number, date, list, or empty).
3. **Args:** parsed by the token's own arg spec: `:0…0` pads numbers; `:N` truncates text to N characters with an ellipsis; `:EN+DE` / `:-EN` filter language lists; `:Name` selects a custom format; `:DE` selects a translation.
4. **Separator:** a separator inside `name` replaces spaces in the value (`{Series.Title}` → `The.Office`).
5. **Case:** an all-lower-case name gives lower-case output, an all-upper-case name upper-case output; otherwise unchanged.
6. **Affixes:** prefix and suffix are emitted only for a non-empty value.
7. **Clean-up** of the whole string: collapse repeated separators; remove empty bracket pairs; trim separators per path component; colon replacement (IMP-062); replace or delete illegal characters; escape reserved device names; strip trailing dots and spaces per component; enforce **255 UTF-8 bytes per component** by first shortening the longest truncatable text token, then hard-truncating.
8. `/` in a template separates path components; values are sanitised per component, so a `/` inside a value never creates a directory.
Whether `:-N` truncates from the start **MUST** be settled against Sonarr's tests before it is part of the grammar. [OPN-123] _Source: design/import.md §3.4 Rendering rules._

**IMP-062** (MUST) Colon replacement modes **MUST** be `delete`, `dash`, `space_dash`, `space_dash_space`, `smart` and `custom`. _Source: design/import.md I14._

**IMP-063** (MUST) A generic range style **MUST** apply where a file links to several numbered items: TV episodes use Sonarr's `extend`, `duplicate`, `repeat`, `scene`, `range` and `prefixed_range` styles, reproduced from Sonarr's golden tests; comic issue ranges render `#001-006`; audiobook parts use `{Part:00} of {PartCount}`. _Source: design/import.md I14, §3.4._

**IMP-064** (MUST) TokenSets **MUST** provide these template slots:

| Kind | Templates |
|---|---|
| TV | `ContainerFolder` (series), `SubFolder` (season; specials), `File.Standard`, `File.Daily`, `File.Anime` |
| Movie | `ContainerFolder`, `File` |
| Music (P2) | `ContainerFolder` (artist), `File.Standard`, `File.MultiDisc` (album folder is part of the file template) |
| Book (P3) | `ContainerFolder` (author), `File` (may contain the book folder), `File.Part` |
| Comic (P3) | `ContainerFolder` (volume), `File.Issue`, `File.IssueNoTitle`, `File.SpecialVersion`, `File.VolumeAsIssue` |
| Adult (P3) | `ContainerFolder.Scene`, `File.Scene`, `ContainerFolder.Movie`, `File.Movie`, `File.Code` |

_Source: design/import.md §3.4 Template slots._

**IMP-065** (MUST) Each `TokenDef` **MUST** declare its canonical name, aliases, scope (`container | file`), arg spec, whether it is truncatable, and a typed value function. Container templates **MUST** reject file-scoped tokens (quality, media info, release group, custom formats) at save time, so that an upgrade never moves a folder. _Source: design/import.md I15, §3.4._

**IMP-066** (MUST) The common token set **MUST** cover ids, quality (full, title, proper, real), `MediaInfo *`, release group, custom formats, original title and original filename. Per-kind sets **MUST** be the union of the tokens of Sonarr, Radarr, Lidarr, Bookshelf, Whisparr v3 and Kapowarr, without Whisparr's v2/v3 dual set. A `{Slot Name}` token **SHOULD** be available in the common set so that slots sharing a root can be distinguished. [OPN-124] _Source: design/import.md §3.4 Token sets; D10; D20._

**IMP-067** (MUST) A `NamingScheme` **MUST** be a named, per-kind set of templates plus options (rename on/off, colon mode, illegal-character mode, range style). The scheme for a placement **MUST** resolve as specified in `02-domain-model.md` DOM-044: the TitleSlot's override, else the SlotDefinition's default, else the root folder's default, else the kind's built-in scheme. Naming presets from profile databases (`07-profiles.md`) **MUST** import into schemes. _Source: design/import.md §3.4 Configuration; A13; D9._

**IMP-068** (MUST) A rename **MUST** be a plan: `Render` for every file in scope, diffed against current paths, with only changed paths becoming `ImportOp{kind=rename}`. Metadata changes that affect a rendered name (for example an episode title replacing "TBA") **MUST** trigger a rename plan for affected files when rename is on. _Source: design/import.md §3.4 Rename._

### 14. Tags and content transforms

**IMP-069** (MUST) Tag writing **MUST** have the modes `never`, `new_files`, `all_files` and `sync`, plus `scrub` and `embed_cover` options, set per kind and per root, with a retag preview (a plan) and a `file.retagged` event. The default **MUST** be `never`. _Source: design/import.md I17, §3.6._

**IMP-070** (MUST) Tag writers per format: audio (ID3v2, MP4 atoms including m4b chapters, Vorbis comments, FLAC) through TagLib (P2); EPUB through a native OPF rewrite of title, creators, identifiers (ISBN/ASIN), series (`calibre:series` and EPUB 3 `belongs-to-collection`) and language (P3, opt-in per root); MOBI, AZW3 and PDF read-only. [OPN-032] _Source: design/import.md §3.6, §6.2 Q5; open-issues C (Import)._

**IMP-071** (MUST) Tag writing and every content transform (ComicInfo.xml, CBR/CB7 → CBZ conversion, splitting multi-issue archives, FLAC+CUE split) **MUST** run only on a staged copy before commit, never on the source. Comic transforms **MUST** be opt-in and off by default. CBR/RAR writing **MUST NOT** be implemented. [OPN-037] _Source: design/import.md §3.1.5, §3.6, §6.2 Q10; open-issues C (Import)._

### 15. Extras and metadata writers

**IMP-072** (MUST) Subtitles found next to a video (same stem, or in a `Subs/` folder) **MUST** be matched to their video by stem, with language, hearing-impaired and forced flags parsed by the shared language parser, renamed to `<video stem>.<lang>[.hi][.forced].<ext>`, and recorded as `ExtraFile{type=subtitle, owner_file}` with its shared `Subtitle` record (`origin=external`, `02-domain-model.md` DOM-110). Other extras **MUST** be imported only if their extension is in the per-kind allow-list (default: `.lrc`, `.cue`, `.log` for music; none for video). _Source: design/import.md I16, §3.6 Carried extras; D1c._

**IMP-073** (MUST) Extras **MUST** follow their media file: renamed with it, moved with it, and added to the retire set with it. On an upgrade the subtitle subsystem is notified by `file.upgraded` and decides per `09-subtitles.md`. _Source: design/import.md I9, I16, §3.6._

**IMP-074** (MUST) Metadata writers **MUST** implement one interface:

```go
type MetadataWriter interface {
    ID() string                                   // "kodi", "plex", "comicinfo", …
    Kinds() []MediaKind
    ForItem(ctx context.Context, item Item, files []MediaFile) []ExtraSpec   // container level
    ForFile(ctx context.Context, file MediaFile, items []Item) []ExtraSpec   // per file
}
type ExtraSpec struct { RelPath string; Content func() (io.Reader, error); Hash []byte }
```

Writers **MUST** write to a temporary file and rename into place, and **MUST** skip the write when the content hash is unchanged. Unmanaged (adopted) extras **MUST NOT** be overwritten unless the user opts in ("take over existing metadata"). _Source: design/import.md §3.6 Metadata writers, I18._

**IMP-075** (MUST) The writers **MUST** be: Kodi (also read by Jellyfin and Emby) for TV, movie, music and adult: `tvshow.nfo`, episode and movie `.nfo` with `<uniqueid>`, `artist.nfo`/`album.nfo`, `poster`/`fanart`/`folder.jpg` (P1 for TV and movie); Plex for TV and movie: `.plexmatch` and images (P1); ComicInfo for comics: `ComicInfo.xml` inside the CBZ as a staged transform (P3). Audiobookshelf and Stash writers come later. [OPN-031] _Source: design/import.md §3.6 table, §6.2 Q4; open-issues C (Import)._

**IMP-076** (MUST) Calibre **MUST** be a downstream target only (notify or add a copy after import); Calibre **MUST NOT** own file placement for any root. [OPN-028] _Source: design/import.md §6.2 Q1; open-issues C (Import)._

### 16. Subtitle sidecar indexing (phase-1 groundwork)

**IMP-077** (MUST) From phase 1, import, rescan and library import **MUST** index subtitle sidecar files as `ExtraFile{type=subtitle}` with language, forced, hearing-impaired and default flags and an owner MediaFile, each with its `Subtitle` record (`02-domain-model.md` DOM-110), and **MUST** record embedded subtitle streams from the probe as `Subtitle{origin=embedded}` rows, even though subtitle acquisition is phase 2. Subtitle sidecars **MUST** always be managed (`managed=true`) when adopted. The subtitle subsystem **MUST** read and write subtitle files only through this extras service, which emits `extra.written` / `extra.deleted`. _Source: A17; design/import.md §3.6, §4.4 step 4, §5 Subtitle files._

### 17. Media-server refresh

**IMP-078** (MUST) Media-server refresh **MUST** be a durable event subscriber to `file.*` and `extra.*` events that batches per server with a short debounce (default 10 s), deduplicates container paths, and issues path-scoped refreshes where supported: Plex partial scan by path; Jellyfin/Emby `Library/Media/Updated`; Kodi `VideoLibrary.Scan` with a directory; Subsonic/Navidrome `startScan`; Audiobookshelf, Kavita and Komga library scans; Stash `metadataScan` with paths. Media-server connections are the shared `MediaServer` rows of `02-domain-model.md` DOM-111, also used by sign-in and watchlists. A refresh **MUST NOT** be sent when nothing changed on disk (for example an in-place library import). _Source: design/import.md §3.9 Media-server refresh, §4.4 step 6._

### 18. Scanning and watching

**IMP-079** (MUST) Scanning **MUST** be the source of truth for what is on disk. A scan for a scope (root, container or item folder) **MUST** list files and join them with MediaFile/ExtraFile rows by `(root_id, rel_path)`, and:
1. mark missing files `missing` and keep the row for a grace period of 7 days, then delete it with `file.deleted{reason=missing_from_disk}`; a health issue **MUST** fire when a whole root is empty or unreachable, and rows **MUST NOT** be deleted while that issue is open;
2. re-probe changed files (size or mtime) and re-derive facts;
3. run new files inside a known item folder through the pipeline with `source=rescan`, `mode=in_place`, auto-accepting only a `Matched` identification that is not an upgrade conflict, and listing the rest as unmapped files (MediaFiles without FileLinks);
4. treat new folders outside any item folder as library-import proposals (`library.unmapped_found`).
[OPN-033] _Source: design/import.md §3.7; design/library.md §3.6, §3.10._

**IMP-080** (MUST NOT) A file going missing **MUST NOT** change `ItemSlot.monitored`. _Source: design/library.md §3.10 (O8)._

**IMP-081** (MUST) Each root **MUST** have a periodic scan schedule (default every 12 h, with a light mtime-only pass), independent of the watcher. _Source: design/import.md §3.7 Watch._

**IMP-082** (MUST) A filesystem watcher (fsnotify) per root **MAY** be enabled. It **MUST** hold a lease `watch:<root_id>` (A1), **MUST** only schedule debounced targeted scans of the affected container folder, and **MUST NOT** be required for correctness. A node that takes over the lease **MUST** re-arm its watches and schedule a light scan of the root. The watcher **MUST** suppress events for paths the executor touched within a short window (default 30 s). _Source: A1; design/import.md §3.7, §3.8 Singletons._

**IMP-083** (MUST) Automatically adding unknown titles found on disk (artists, authors, and other kinds) **MUST** be a per-root-folder option, off by default. [OPN-030] [OPN-045] _Source: design/import.md §6.2 Q3; design/library.md §3.10; open-issues C (Import, Library)._

### 19. Library import of existing folders

**IMP-084** (MUST) Library import **MUST** let users adopt existing folders without moving anything:
1. a root scan lists unmapped folders;
2. for each folder a proposal is built from the parser and from ids in existing metadata (Kodi NFO `<uniqueid>`, `.plexmatch`, embedded tags), resolved with `CatalogLookup` (`AllowRemote`);
3. an id match **MUST** be pre-selected; a title-only match within the ambiguity margin **MUST** require confirmation;
4. the user confirms item and slot (default inferred by IMP-038);
5. the import runs with `source=library_import`, `mode=in_place`, recording MediaFiles with `transfer=in_place`.
_Source: design/import.md I18, §4.4._

**IMP-085** (MUST) Library import **MUST** keep existing paths by default; renaming to the scheme **MUST** be an explicit option that shows the rename plan. The title's `TitleSlot.path` **MUST** be set to the existing folder. [OPN-036] _Source: design/import.md §6.2 Q9; open-issues C (Import)._

**IMP-086** (MUST) Existing extras **MUST** be adopted: subtitles as managed `ExtraFile`s; NFO and images as `managed=false`. _Source: design/import.md §4.4 step 4, I18._

### 20. Recycle bin

**IMP-087** (MUST) A recycle bin **MUST** be configurable per root folder or globally. Retired files and their extras **MUST** keep their root-relative directory; a name collision **MUST** get a timestamp suffix. The recycle bin **SHOULD** be on the same filesystem as the root so that retiring is a rename. _Source: design/import.md I9, §3.5 Recycle bin._

**IMP-088** (MUST) A scheduled job **MUST** purge recycle-bin entries older than the retention (default 7 days) and emit `recycle.purged{root_id, rel_path, content_key}` for each purged file. [OPN-125] _Source: design/import.md §3.5; design/downloads.md §3.10 (hardlink mode needs the purge event)._

### 21. Filesystem requirements (A5)

**IMP-089** (MUST) The extraction staging root (`05-downloads.md` DLD-069) **MUST** be on the same filesystem as each library root it serves. This is a deployment requirement checked by a health check. _Source: A5._

**IMP-090** (MUST) Hardlink and reflink imports require the download paths and the library roots on one filesystem. A health check **MUST** test, per (download client, root folder) pair and on each node, whether a hardlink and a reflink can be created (with test files, at start-up and daily), and **MUST** raise a warning when they cannot. _Source: A5; design/import.md §3.8 Shared storage._

**IMP-091** (MUST) When an import falls back to copy because links are impossible, the fallback **MUST** be visible: in the plan preview, in `file.imported` (transfer method), in a metric, and in the health issue from IMP-090. It **MUST NOT** be silent. _Source: A5._

**IMP-092** (MUST) In HA deployments roots **MUST** be mounted at identical paths on every node; paths are stored as `(root_id, rel_path)` (A12). _Source: D14; A12._

### 22. HA placement

**IMP-093** (MUST) Import, scan, rename, retag, recycle purge and recovery **MUST** run as jobs on the durable queue, claimable by any node, in the `io` queue. Every job that plans or executes file operations for a title **MUST** declare the concurrency key `lib:title:<title_id>` for its whole Plan and Execute, so that two nodes never plan against the same slot state at once (replacing per-container leases, A1). Import jobs for downloads **MUST** use the dedupe key `import:download:<id>:<attempt>`. A root scan **MUST** fan out into per-title child jobs carrying their title's key. Only the watcher uses a lease (IMP-082). _Source: A1; design/import.md §3.8 HA; design/runtime.md §3.4._

**IMP-094** (MUST) Probing, copying and hashing **MUST** run on bounded per-node worker pools, with a global concurrency limit per root (default 2) to avoid saturating network mounts. _Source: design/import.md §3.8 Heavy work._

### 23. Events

**IMP-095** (MUST) Import **MUST** publish, in the Record transaction's outbox:

| Event | Payload (abridged) |
|---|---|
| `import.completed` | request id, download id, status, per-target results, transfer methods |
| `import.rejected` | unit, rejections, candidates (for manual import and needs-user alerts) |
| `file.imported` | media file, items, slot, source, download id, transfer method, source path and inode |
| `file.upgraded` | new file, retired files, recycle paths, slot |
| `file.deleted` | file, reason (`upgrade`, `missing_from_disk`, `user_delete`, `manual_override`, `item_deleted`, `slot_removed`; the first four as in `02-domain-model.md` DOM-085 and §4.5) |
| `file.renamed` / `file.moved` | old and new `(root_id, rel_path)` |
| `file.retagged` | file, tag diff |
| `extra.written` / `extra.deleted` | extra, owner |
| `recycle.purged` | root, relative path, content key |
| `library.scan_completed` | scope, counts (new, changed, missing, unmapped) |
| `library.unmapped_found` | root, folders or files |

_Source: design/import.md §3.9._

## Data model

`MediaFile`, `FileLink`, `RootFolder`, `TitleSlot`, `ItemSlot`, `SlotDefinition` and `Subtitle` are owned by `02-domain-model.md`. Import relies on these fields of theirs:

```go
MediaFile (02)  { root -> RootFolder; rel_path string; unique(root_id, rel_path)   // A12
                  slot -> SlotDefinition (exactly one); size int64; mtime time; content_key bytes
                  class enum; quality json; languages []string; release_group, edition, scene_name, original_path string?
                  indexer_flags json; transfer json {method, source_path, source_inode}; imported_at time
                  download -> Download?; state enum(present, missing); missing_since time? }
FileLink (02)   { file -> MediaFile; item -> Item; edition -> Edition?; track -> Track?; part int?; coverage json?
                  source enum(parsed, identified, manual) }
```

Import owns:

```go
ExtraFile       { owner_file -> MediaFile?; owner_item -> Item?; type enum(subtitle, metadata, image, lyric, other)
                  writer string; lang string?; hi, forced, default bool; root -> RootFolder; rel_path string
                  hash bytes; managed bool }       // type=subtitle rows are referenced by Subtitle (02 DOM-110)
ImportRun { id uuid; source enum(download, rescan, library_import, manual, watch); download -> Download?
            request json; plan json; status enum(planning, planned, executing, complete, partial, blocked, failed)
            outcome json; created_at, finished_at time; created_by -> User? }
ImportOp  { id string (deterministic: run id + placement + file); run -> ImportRun
            kind enum(import, rename, retag, move, delete)
            state enum(planned, staged, transformed, retired, committed, recorded, cleaned, done, failed)
            source_path string?; staged_path string?; final_root -> RootFolder?; final_rel_path string?
            retired json      // [{original, retired_path}]
            method enum(rename, reflink, hardlink, copy, in_place)?; transform_hash bytes?
            error json?; updated_at time
            index(run_id); index(state) }
ProbeResult { content_key bytes; prober string; schema int; data json; probed_at time
              unique(content_key, prober) }
NamingScheme { name string; kind enum; templates json; options json   // rename, colon mode, illegal-char mode, range style
               source_preset json?; unique(kind, name) }
ImportSettings (per kind, per root override) { tag_mode enum; scrub, embed_cover bool; extras_allowlist []string
               episode_title_required enum(always, bulk_season_releases, never); title_required_timeout_s int
               verify_hash bool; chmod_file, chmod_dir string?; chgrp string?; file_date enum
               recycle_path string?; recycle_retention_days int; auto_add_unknown bool; watch bool; scan_interval_s int }
MetadataWriterConfig { writer string; kinds []enum; enabled bool; options json; take_over_existing bool }
```

`TitleSlot.naming_scheme`, `SlotDefinition.default_naming_scheme` and `RootFolder.default_naming_scheme` reference `NamingScheme`; `RootFolder.import_options` carries the per-root settings above (`02-domain-model.md` §3.2).

## Flows

### F1. TV season pack with a sample (P1)

1. The lifecycle sends `ImportRequest{source=download, grab targets=(S02E01–E10, hd), payload=client_seeding}`; the job holds `lib:title:<Show>`.
2. Discover: 11 videos, 10 subtitles, 1 NFO (not in the TV allow-list, ignored).
3. Probe: 11 cache misses; the sample runs 41 s, episodes about 44 min.
4. Group: 11 units, folder context "season pack". Identify: `S02Exx` from each name; the release name agrees.
5. Aggregate: WEBDL-1080p from the release name, confirmed by the probe.
6. Decide: the sample gets `Sample` (benign) and is excluded; E01–E03 upgrade HDTV-720p; E04–E10 fill empty slots.
7. Plan: destinations from the `hd` scheme; retain source → reflink, else hardlink, else copy; retire set = three old files and extras; subtitles renamed to `….en.srt`.
8. Execute: stage 20 files, retire 3 to the recycle bin, commit, record.
9. Post-import: `file.upgraded` ×3, `file.imported` ×7, `import.completed{complete}`; one Plex refresh for the season folder.

*Failure path:* a crash after Retire and before Commit: recovery finds staged files present, rolls forward to Commit; if the staged file is missing, it restores the retired files.

### F2. One movie into two slots (D20)

1. `ImportRequest{targets=[(M, uhd), (M, best)], payload=owned}` with one staged Remux-2160p file.
2. Decide per slot: `uhd` accepts (empty); `best` accepts (upgrade over Bluray-1080p). Two placements.
3. Plan: `uhd` destination `/movies-uhd/M (2023)/M (2023) [Remux-2160p].mkv`, `best` destination `/movies/M (2023)/M (2023) [Remux-2160p].mkv`; retire set for `best` = the old 1080p file. The source is released: the `best` placement uses reflink → hardlink → copy; the `uhd` placement (same filesystem as staging) gets the rename, executed last (IMP-052).
4. Execute and record per slot; Clean runs after both are recorded.
5. Events: `file.imported{slot=uhd}`, `file.upgraded{slot=best}`; the download lifecycle records two links.

*Failure path:* `best` has tag-free video, so no mutation; if `best` had tag writing on, it would get a reflink or copy, never a hardlink.

### F3. Music album with a missing disc (P2)

1. TagLib reads 12 files of a 2×CD release; one album unit.
2. Distance: the 2×CD release 0.09 (12 missing tracks); a 1×CD release of the same album 0.02. Same album → no ambiguity; the slot is not locked → choose the 1×CD release and switch the target.
3. *Variant:* only the 2×CD release exists → `MissingTracks{missing=12}` → unit blocked; in manual import the user sets `AllowPartial`, Decide re-runs, 12 files import in one transaction.
4. Tag writing `new_files` → `Mutates` → reflink or copy.

### F4. Library import of an existing movie folder (P1)

1. A root scan lists `Blade Runner (1982) [Final Cut]/` as unmapped (`library.unmapped_found`).
2. The Kodi NFO gives TMDb id 78; `CatalogLookup(tmdb:78)` matches; the proposal is pre-selected; the user confirms slot `hd` (inferred by profile fit).
3. Import `{source=library_import, mode=in_place}`: probe, identify within `ScopeItem`, Record only. The `.srt` is adopted managed; NFO and poster unmanaged.
4. `file.imported{source=library_import}`; no media-server refresh.

### F5. Transient rejection

1. A download import hits `FileLocked` on one file.
2. The import job retries with backoff; if the file stays locked past the transient budget, the outcome reports the rejection and the lifecycle applies its reason policy (`05-downloads.md` DLD-085).

## Non-goals

| Non-goal | Reason |
|---|---|
| Script import (`ScriptImportDecider`, `Sonarr_*` environment) | D7; transforms are built in, and remux/transcode uses the external processing hold (`05-downloads.md` §11, D20). |
| Calibre owning file placement | Defeats naming, the journal and the recycle bin. [OPN-028] |
| Writing CBR/RAR | Needs the proprietary `rar` binary; CBZ is written instead. |
| Roksbox, WDTV, Emby Legacy and Kometa metadata writers | Dead devices or superseded formats. |
| One file serving two slots (a single MediaFile in two slots) | D20 gives each slot its own library file. |
| Watcher-dependent correctness | inotify misses remote NFS/SMB changes; scans are authoritative. |
| A separate subtitle file index or rescan round trip | One `ExtraFile` table in one process. |
| Filesystem-name checks for reflink support | Try-and-fall-back covers XFS, bcachefs and APFS. |

## Traceability

| Reference | Preserved | Changed deliberately (D19) |
|---|---|---|
| Sonarr/Radarr | Parse-based identification with grab and folder checks; sample tiers; upgrade order (quality, revision, CF); naming grammar, colon and multi-episode styles (golden tests); auto move vs copy for seeding; verified cross-mount copies; recycle bin | One pipeline instead of six forks; typed rejections with class and scope; plan-then-execute journal (stage before retire) instead of recycle-then-transfer; reflink by attempt; `(root_id, rel_path)` paths; one probe per content key |
| Whisparr v2/v3 | Graded matching, product-code fallback, reject ambiguous (D10) | Ambiguity rule generalised to every kind with per-kind margins |
| Lidarr | Distance, Munkres, AcoustID escalation, thresholds and fixtures; album-atomic imports; release switching; tag modes and retag preview | Never hardlink a file that will be tagged; watcher schedules targeted scans only |
| Bookshelf | Book distance weights, ISBN/ASIN, medium bias, fixtures | Threshold 0.30 (open item); per-slot edition; Calibre downstream only |
| Kapowarr | Comic M:N coverage, special versions, magic-byte extension fixing, library import | CBZ only; transforms opt-in and staged |
| Bazarr | Subtitle filename conventions (`.lang.hi.forced`) | One probe and one `ExtraFile` index; no rescan round trips or path mappings |
| Unpackerr / Cleanuparr | — | Out-of-band import fixes replaced by in-process typed outcomes |
| Multi-instance 4K/1080p setups | Both versions available | One download imports into several slots (D20) |
