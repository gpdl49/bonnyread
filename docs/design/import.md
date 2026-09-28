# Design: Import and File Management

Status: draft for review (2026-09-28). Scope: everything from "files are on local disk" to "organised, identified, recorded library files with their extras". Binding inputs: [decisions.md](../decisions.md) (especially D2, D4, D10, D14, D19), [analysis/comparison.md](../analysis/comparison.md) and the per-app analyses. Paths in citations are relative to `submodules/<app>/` unless noted; for the NzbDrone forks, `Core/` means `src/NzbDrone.Core/` and `Common/` means `src/NzbDrone.Common/`.

**Boundaries.** Other design areas own the following; this document states only the contract with each.

| Neighbour | What it gives import | What import gives back |
|---|---|---|
| Download lifecycle | An `ImportRequest` once a tracked download is complete and extracted. Paths are already local, because remote path mappings are applied there. It also carries the grab context and whether the client still needs the source files (seeding). | `ImportOutcome` (Complete / Partial / Blocked / Failed), per-file results, how each file was transferred (so it can do seeding/"unlinked" housekeeping) |
| Library model | Items, containers, **version slots** (D4), root folders, M:N `FileLink` rows | New and changed `MediaFile`/`FileLink`/`ExtraFile` rows (schema proposed in §3.3; the library agent owns the final shape) |
| Evaluation / profiles | Quality model per kind, custom-format engine, slot profiles, upgrade comparator | File facts to evaluate (parse + probe) |
| Parser (D1/D8) | One parser package with per-kind sub-parsers | – |
| Metadata | `CatalogLookup` by id or search (library import, unknown files) | – |
| Subtitles | Acquisition, scoring, sync | Import owns the subtitle **file** records (`ExtraFile`) and the embedded-track probe; the subtitle agent writes through the same extras service |
| Runtime / events | Durable job queue, leases, leader election, outbox/events (D14) | Domain events (§3.9) |

---

## 1. Outcomes and invariants to preserve

These are the user-visible behaviours and correctness rules that any redesign must keep (D19). The evidence is in the reference code.

### 1.1 Identification correctness

- **I1. Never import into the wrong item. When in doubt, reject.** Whisparr's graded `MatchMovies` walks match levels and returns *no match* when no level isolates one candidate (whisparr-eros `Core/Movies/MovieService.cs:771-1012`, analysis [whisparr-eros.md](../analysis/whisparr-eros.md) §3.2). D10 makes "reject ambiguous matches everywhere, including fuzzy matching" binding. Lidarr's matcher rejects above a distance threshold (`Core/MediaFiles/TrackImport/Specifications/CloseAlbumMatchSpecification.cs:12-13`: album ≤ 0.20, worst track ≤ 0.40).
- **I2. Grab context is authoritative but checked.** A file that came from a download grabbed for item X must parse as X, otherwise it is rejected: Sonarr/Radarr `MatchesGrabSpecification`, `MatchesFolderSpecification` (`Core/MediaFiles/EpisodeImport/Specifications/`). Lidarr seeds candidates from the grabbed album (`TrackImport/Identification/CandidateService.cs`).
- **I3. Parse-based kinds** (TV, movies, adult) resolve through the shared parser with ids first:
  - Radarr: IMDb id → TMDb id → title+year → title (`Core/Parser/ParsingService.cs`, [radarr.md](../analysis/radarr.md) §3);
  - Sonarr: scene mappings, XEM, anime absolute and daily air date (`Core/Parser/ParsingService.cs`, 729 lines);
  - adult: StashId → product code → studio+date (whisparr-eros `MovieService.FindScene`, `:700-755`; D10 restores the indexed product code).
- **I4. Distance-based kinds** (music, books) keep the beets-style model: candidate generation, then weighted normalised distance, then an optimal track↔file assignment (Hungarian algorithm), then thresholds. The weights and thresholds are behaviour:
  - Lidarr: `Identification/Distance.cs:11-31`, the 10 s track-length grace in `DistanceCalculator.cs:43-47`, and `Munkres.cs`. AcoustID is used only when the best distance is above 0.15 or the worst track above 0.40 (`IdentificationService.cs:155-165`).
  - Bookshelf: ISBN and ASIN weights of 10, and the ebook/audio format bias (`MediaFiles/BookImport/Identification/Distance.cs:11-38`).
  - The golden fixtures are Lidarr's `NzbDrone.Core.Test/Files/Identification/*.json` and Bookshelf's identification fixtures.
- **I5. Comics are M:N and coverage-based.** One archive can cover an issue range or a whole TPB, and special versions (TPB, one-shot, hardcover, omnibus, volume-as-issue) change how a file maps to issues (kapowarr `backend/implementations/file_matching.py:30`, `backend/base/file_extraction.py:316,611`, `backend/base/definitions.py` `SpecialVersion`).
- **I6. Sample and junk rejection.** Samples are detected by runtime against the expected runtime: 15 s / 90 s / 5 min / 10 min tiers (`Core/MediaFiles/EpisodeImport/DetectSample.cs:103-146`). Dangerous and executable extensions, archives and "still unpacking" folders are rejected (`ImportRejectionReason.cs`, `NotUnpackingSpecification.cs`).

### 1.2 Decisions and replacement

- **I7. Upgrades only.** A file replaces an existing one only if it is better by quality, then revision, then custom-format score (`UpgradeSpecification`; the rejection codes `NotQualityUpgrade`, `NotRevisionUpgrade`, `NotCustomFormatUpgrade` and `NotCustomFormatUpgradeAfterRename` in `ImportRejectionReason.cs`). Music decides upgrades at album level (`AlbumUpgradeSpecification`, `MoreTracksSpecification`).
- **I8. A new download of an album is all or nothing** unless the user overrides: `NoMissingOrUnmatchedTracksSpecification.cs:17-33` rejects new downloads with missing or unmatched tracks. Manual import may import partially.
- **I9. Replaced files are recoverable.** Upgraded and deleted files, and their extras, go to the recycle bin with their relative folder kept, and are purged after N days (`Core/MediaFiles/RecycleBinProvider.cs:70-197`; `Extras/Files/ExtraFileService.cs:104-128`).
- **I10. Quality and language come from several sources, weighed by confidence:** release name from the grab, file name, folder, download-client item, then media info (Sonarr `EpisodeImport/Aggregation/Aggregators/Augmenters/{Quality,Language}/*`).

### 1.3 Transfer semantics

- **I11. The download client's copy is never broken while it still needs it.** In Auto mode the import copies when the client item says `CanMoveFiles=false` (seeding), and moves otherwise (`EpisodeImport/ImportApprovedEpisodes.cs:110-121`, `DownloadedEpisodesImportService.cs:219-221`). "Copy" uses a hardlink when possible (`CopyUsingHardlinks`, `EpisodeFileMovingService.cs:100-107`) and a reflink on btrfs/zfs (radarr `Common/Disk/DiskTransferService.cs:340-385`). Kapowarr's `seeding_handling: copy|complete` expresses the same idea (`backend/features/post_processing.py:165-186`).
- **I12. Transfers are verified.** A cross-mount move is a verified copy followed by a delete, with size verification (`Common/Disk/DiskTransferService.cs:450-500`). A case-only rename goes through a `.backup~` temporary name (`:59-77, 280-291`).

### 1.4 Naming

- **I13. The token grammar is a user contract.** Users' saved formats and the naming presets that profile datasets publish (D9) use it. The grammar is Sonarr's `TitleRegex` (`Core/Organizer/FileNameBuilder.cs:48`):
  - `{prefix Token Name:args suffix}`;
  - `{{` and `}}` escapes;
  - the separator inside a token name sets the output separator;
  - casing follows the token's casing;
  - `:00` pads and `:30` truncates;
  - prefix and suffix are dropped when the value is empty;
  - repeated separators are collapsed and trailing separators trimmed (`:71-72`);
  - reserved device names are handled (`:86`).
- **I14. Colon replacement modes** are Delete, Dash, SpaceDash, SpaceDashSpace, Smart and Custom. **Multi-episode styles** are Extend, Duplicate, Repeat, Scene, Range and PrefixedRange (`FileNameBuilder.cs:1259-1277`). Each kind has its own formats: TV has standard, daily and anime formats plus series, season and specials folders (`Organizer/NamingConfig.cs:7-19`); music has standard and multi-disc track formats (lidarr [analysis](../analysis/lidarr.md) §3.6); books have part tokens.
- **I15. Folder names don't depend on file properties.** Radarr deprecates file-property tokens in folder formats (`Organizer/FileNameValidation.cs:12`, `HealthCheck/Checks/NamingConfigCheck.cs`), because otherwise an upgrade moves the folder.

### 1.5 Files, extras and tags

- **I16. Extras follow their media file.** This covers subtitles (`name.<lang>[.hi|.cc|.sdh][.forced].<ext>`, Bazarr's filename conventions in [bazarr.md](../analysis/bazarr.md) §11), NFO and images, and other extensions. They are renamed with the file and recycled with it (`Extras/ExtraService.cs`, `Extras/Subtitles/`, `Extras/Metadata/Consumers/Xbmc`, `Plex` for `.plexmatch`).
- **I17. Tag writing is opt-in and previewable.** The modes are No, NewFiles, AllFiles and Sync, plus Scrub and EmbedCoverArt, with a retag preview and a `TrackFileRetagged` event (lidarr `Core/MediaFiles/AudioTagService.cs:235+`, [lidarr.md](../analysis/lidarr.md) §3.4). Ebook tags are written only through Calibre today ([bookshelf.md](../analysis/bookshelf.md) "tag reading and writing").
- **I18. Existing libraries can be adopted without moving anything.** This is library import of unmapped folders (`RootFolders` `UnmappedFolders`, Radarr `movie/import`, kapowarr `backend/features/library_import.py`), with existing extras adopted too (`Extras/ExistingExtraFileService.cs`, `XbmcNfoDetector.cs`).
- **I19. Unchanged files are skipped on rescan** by comparing modification times (Lidarr's `TrackFile.Modified`; the Bookshelf fork's `LastRemoteSearchTime` stamp keeps rescans from re-querying metadata for files it already failed to match).

---

## 2. How the references do it today, and the accidental complexity

### 2.1 The shape every fork shares

Every NzbDrone fork has the same pipeline, copied per kind:

`Downloaded*ImportService` → `ImportDecisionMaker` (aggregators + specifications) → `ImportApproved*` → `*FileMovingService` → `DiskTransferService` → `UpgradeMediaFileService` → `ExtraService` → events → notifications (which include the media-server refresh).

| App | Import package | Identification | Probe | Paths |
|---|---|---|---|---|
| Sonarr | `Core/MediaFiles/EpisodeImport/` | parser + `ParsingService` | ffprobe (`MediaInfo/VideoFileInfoReader.cs:61-66`) | relative to the series folder |
| Radarr | `Core/MediaFiles/MovieImport/` | parser + `ParsingService.GetMovie` | ffprobe | relative |
| Whisparr v3 | `MovieImport/Identification/IdentificationService.cs` | parser + `FindScene`/`MatchMovies` | ffprobe | relative |
| Lidarr | `Core/MediaFiles/TrackImport/` | distance + Munkres + AcoustID | TagLib | **absolute** |
| Bookshelf | `Core/MediaFiles/BookImport/` | distance (no Munkres, no AcoustID) | TagLib + EPUB/AZW/PDF readers | **absolute**, Calibre may own placement |
| Kapowarr | `features/post_processing.py`, `implementations/file_matching.py` | filename parse + M:N | none | absolute |

### 2.2 Accidental complexity to drop

1. **Six file-name builders plus two more.** There is a `FileNameBuilder.cs` in each of the six forks, all descended from Sonarr's (Bookshelf still carries Lidarr music naming, and Radarr carries a dead `EpisodeFormat`, comparison §3.2). Kapowarr has `backend/implementations/naming.py`, and Profilarr re-implements the token set in `src/lib/shared/pcd/namingTokens.ts` only to validate formats it pushes to the Arrs.
2. **Six import-decision makers** with the same structure (aggregators → specs → approved importer) and near-identical specs (`AlreadyImported`, `FreeSpace`, `NotUnpacking`, `Upgrade` exist in every fork).
3. **A file is probed several times.** Sonarr probes at aggregation (`EpisodeImport/Aggregation/AggregationService.cs:60`), in the runtime-only sample path (`DetectSample.cs:36`), after a script import (`MediaFiles/ScriptImportDecider.cs`) and again on schema-revision rescans (`MediaInfo/UpdateMediaInfoService.cs:51`). Bazarr then probes the same file again with knowit and keeps a **pickled** cache in its own DB (`bazarr/utilities/video_analyzer.py:232-344`).
4. **Two subtitle indexes and a rescan round trip.** The Arrs' `SubtitleFiles`/`ExtraFiles` and Bazarr's `table_*_subtitles` index the same files, and Bazarr posts `RescanSeries`/`RescanMovie` so the Arr notices a new `.srt` (`bazarr/sonarr/notify.py:15`, `bazarr/radarr/notify.py:15`). Bazarr also carries arr↔Bazarr path mappings (`bazarr/utilities/path_mappings.py:17-50`).
5. **Script import.** `ScriptImportDecider` hands the transfer to a user script through `Sonarr_*` environment variables (`MediaFiles/ScriptImportDecider.cs:56-138`) and re-probes the result. It is a workaround for missing transfer and transform hooks. D7 already drops the `Sonarr_*` contracts.
6. **The upgrade ordering loses the file on a crash.** `UpgradeMediaFileService` sends the old file to the recycle bin and deletes its DB row **before** transferring the new file (`MediaFiles/UpgradeMediaFileService.cs:55-84`). A crash or a failed copy in between leaves the item with no file.
7. **Tag writing can modify a seeding file.** Lidarr may import by hardlink (`TrackFileMovingService.cs:101-104`), and `AudioTagService.WriteTags` (`:235-260`) then writes tags to the library path. No guard against a shared inode was found, so the tags are written into the torrent client's copy too. By the same reasoning, `chmod` and file-date changes on a hardlink affect both names.
8. **Reflinks are keyed on filesystem names.** Radarr tries reflinks only when both mounts report `btrfs` or `zfs` (`DiskTransferService.cs:345-347`), and only Radarr does. Trying a clone and falling back is simpler and covers XFS, bcachefs and APFS.
9. **Absolute paths in some apps and relative paths in others.** Moving a Lidarr artist rewrites every `TrackFile` row ([lidarr.md](../analysis/lidarr.md) §8), while Sonarr and Radarr store paths relative to the item folder.
10. **Watching exists in only two apps.** Lidarr and Bookshelf run a `FileSystemWatcher` with a debounce (`Core/MediaFiles/RootFolderWatchingService.cs:36-58`); Sonarr and Radarr rely on scheduled rescans. Bazarr never watches and learns of files through SignalR, webhooks or an hourly full sync ([bazarr.md](../analysis/bazarr.md) §9.4).
11. **Outside tools fix import from outside.**
    - Unpackerr infers "imported" when a title leaves the queue, and triggers Lidarr `ManualImport` after a CUE split ([unpackerr.md](../analysis/unpackerr.md) §3).
    - Cleanuparr force-imports by matching status-message **text** such as "matched to series by ID" (`ForceImportService.cs`, [cleanuparr.md](../analysis/cleanuparr.md) §3.1.4).

    In-process typed rejections make both unnecessary.
12. **The media-server refresh is duplicated.** It exists as notifications in each fork (batched by `Notifications/MediaServerUpdateQueue.cs` in Radarr), in Bazarr (`bazarr/plex/operations.py`, `bazarr/jellyfin/*`) and in Seerr's scanners.
13. **Calibre owns file placement** for Calibre roots: Bookshelf uploads the book and reads the path back ([bookshelf.md](../analysis/bookshelf.md) "Calibre integration"). That bypasses naming, the recycle bin and crash-safety.
14. **Kapowarr's CBR/RAR writing** needs the proprietary `rar` binary (`impl/converters.py`, [kapowarr.md](../analysis/kapowarr.md) §3.5).
15. **Duplicated per-app metadata writers** exist (Kodi, Plex, Roksbox, WDTV, Emby Legacy, Kometa), and some are for dead devices (comparison §3.1).

---

## 3. Proposed design

### 3.1 Principles

1. **One pipeline, with the per-kind logic behind four narrow interfaces:** `Grouper`, `Identifier`, `Prober` and `TokenSet`. Everything else is kind-neutral: discovery, aggregation plumbing, the decision framework, planning, transfer, the journal, extras, events and the refresh.
2. **Plan, then execute.** Deciding is a pure function of (files, probes, library state, settings) → `ImportPlan`. Executing the plan is the only code that touches the filesystem. It is journaled, idempotent and replayable. The same executor also performs rename, retag, move-item and delete.
3. **One probe per file content**, cached by content key and shared by every consumer: quality, custom formats, sample detection, naming, subtitles and the UI.
4. **Scanning is authoritative and watching is a hint.** The watcher only schedules targeted scans.
5. **Never mutate the source.** Anything that changes content (tags, CBZ conversion, ComicInfo.xml, a CUE split) produces a new staged file and never touches the download client's copy.

### 3.2 Pipeline stages

```
ImportRequest ──► 1 Discover ──► 2 Probe ──► 3 Group ──► 4 Identify ──► 5 Aggregate ──► 6 Decide ──► 7 Plan ──► 8 Execute ──► 9 Post-import
 (download,        classify,      one per     units        per-kind       quality,        typed         naming,     journal:        extras, tags
  rescan,          skip           content     (album,      strategy       lang, CF,       rejections    transfer    stage→retire→   already staged;
  library import,  unchanged      key,        audiobook,   → items+slot   edition         (shared +     method,     commit→record   media-server
  manual, watch)                  cached      pack…)                                      per kind)     retire set               refresh, events
```

**Stage 1: Discover.** Walk the request's paths. Classify each file by extension, and by magic bytes for archives (Kapowarr fixes extensions from magic bytes), into a `FileClass`: Video, Audio, Ebook, ComicArchive, Subtitle, Image, NFO, Lyric, Playlist/Cue/Log, Archive, Executable, Other. Then:
- Reject executables and dangerous files as a unit with the `DangerousFile` rejection. The download lifecycle also receives this as a content-policy signal.
- Skip files that are already known and unchanged, matched on `(root, rel_path, size, mtime)` (I19).
- Honour ignore globs and `.bonnyreadignore`.
- For download-sourced requests, the lifecycle guarantees extraction is finished (its `Extracting` stage). The working-folder check (`_UNPACK_`, `NotUnpackingSpecification.cs:28-50`) stays only for disk-sourced imports.

**Stage 2: Probe.** For each media file, look up `ProbeResult` by `ContentKey = (device, inode, size, mtime_ns)`, with a sampled xxhash (first, middle and last MiB) as a fallback on filesystems without stable inodes. Run a prober only on a cache miss.

| Prober | Kinds | Implementation (candidates to confirm in a spike) |
|---|---|---|
| `VideoProber` | TV, movie, adult | `ffprobe` executable (a bundled binary, as Sonarr does). A deep second pass only when the first yields no streams, as in `VideoFileInfoReader.cs:61-66`. Outputs container, video streams (codec, resolution, HDR format, bit depth), audio streams (codec, channels, language), **subtitle streams (language, forced, default, codec)**, runtime and chapters |
| `AudioProber` | music, audiobooks | TagLib. Candidates: a WASM build of TagLib run in `wazero` (the D15 runtime; `go.senan.xyz/taglib` is one such binding, still to verify) for read and write; `dhowden/tag` is read-only. Outputs tags (including MusicBrainz ids, ISBN/ASIN, narrator), codec, bitrate, sample rate, bit depth, duration, chapters (m4b) and embedded art |
| `EbookProber` | books | EPUB OPF (zip + XML), MOBI/AZW3 EXTH (a port of Bookshelf's `AzwTag`, about 500 lines), PDF Info dictionary (e.g. `pdfcpu`) |
| `ComicProber` | comics | archive listing (zip, plus a pure-Go RAR/7z reader via the `xtractr` dependencies), page count, embedded `ComicInfo.xml` |
| `Fingerprinter` | music (on demand) | `fpcalc` (Chromaprint), run only when stage 4 escalates (I4) |

One probe feeds every consumer:
- quality augmenters (stage 5) and custom formats, through the evaluation agent;
- sample detection;
- `{MediaInfo …}` tokens;
- the subtitle agent's embedded-track view;
- the UI.

Bazarr's second probe and the extra Sonarr probes disappear. A `ProbeSchemaVersion` bump re-probes lazily, not in a startup storm.

**Stage 3: Group.** A `Grouper` per kind turns files into `ImportUnit`s, the atomic unit of identification and decision.
- **Video:** one unit per video file, carrying its folder context (season pack or single release).
- **Music:** album units. Files are grouped by directory, then multi-disc subfolders (`CD1`, `Disc 2`) are merged, then files are split by album tag. This follows `TrackImport/Identification/TrackGroupingService.cs:22-25,176-236`.
- **Books:** one unit per edition *per medium*. Ebook files and audio files are never in the same unit. Multi-file audiobooks become one unit with parts ordered by track number, then disc, then natural filename order.
- **Comics:** one unit per archive, with the volume folder as context.

**Stage 4: Identify.**

```go
type Identifier interface {
    Kind() MediaKind
    // Identify maps a unit to library targets. It must return Ambiguous rather than guess (I1, D10).
    Identify(ctx context.Context, u *ImportUnit, hints Hints) (Identification, error)
}

type Hints struct {
    Grab       *GrabContext   // item ids + slot + parsed release, from the download lifecycle (I2)
    ScopeItem  *ItemRef       // manual import / rescan restricted to one item or container
    AllowRemote bool          // library import may query CatalogLookup; download imports use the local library only
}

type Identification struct {
    Status     IdentStatus    // Matched | Ambiguous | NoMatch | Unparseable
    Targets    []Target       // file → item(s), with coverage for M:N
    Candidates []Candidate    // ranked, with distance/confidence and evidence, for manual import UI
    Evidence   []Evidence     // "ISBN matched", "S02E03 from filename", "AcoustID 0.93"...
}
type Target struct { FileIdx int; Items []ItemRef; Part *PartInfo; Coverage *Coverage }
```

Each kind has a strategy:

| Kind | Strategy | Notes |
|---|---|---|
| TV | `ParseIdentifier`: parser → series by id or scene title → episodes through scene/XEM/absolute/daily numbering | A port of Sonarr's `ParsingService`. The release name from the grab is parsed as well as the file name; they must agree (I2) |
| Movie | `ParseIdentifier`: ids in the name, then TMDb/IMDb, then title+year (± `SecondaryYear`), then title | Radarr's order. Editions come from the parser |
| Adult | `ParseIdentifier` + `GradedMatcher`: StashId, then indexed product code, then studio+date with the graded levels | The fuzzy pass **must** apply the ambiguity margin (D10 restores v2's rule) |
| Music | `DistanceIdentifier`: candidates (tag MBIDs, then grab, then local fuzzy artist/album, then AcoustID recordings), distance, Munkres, thresholds | Ported with the Lidarr weights and fixtures. Fingerprinting escalates only past 0.15/0.40 |
| Books | `DistanceIdentifier` with the book weight table: candidates by ISBN, then ASIN, then provider id tags, then author+title, then title, then author | Bookshelf's order. The medium bias decides between ebook and audio editions |
| Comics | `CoverageIdentifier`: comic sub-parser → volume, issue number(s), range, special version → issues | Kapowarr's `extract_filename_data` and `refine_special_version`, plus its test corpus |

A **shared ambiguity rule** (D10, generalised): if the best two candidates resolve to *different items* and their scores are within a margin, the result is `Ambiguous` (rejection `AmbiguousMatch`, class NeedsUser). The margin is per kind: a distance difference of at most 0.05 for distance kinds, and the same match level for graded matching. Choosing between releases or editions *of the same item* is not ambiguity. That choice is governed by the item's `AnyReleaseOk`/`AnyEditionOk` setting, as in Lidarr's release switching (`ImportApprovedTracks.cs:125-131`).

Download imports identify against the **local library only**. Unknown things are not auto-added from downloads. Rescans and library import may call the metadata agent's `CatalogLookup`, rate-limited and cached (D3). Lidarr's auto-add of unknown artists on a disk scan becomes a per-root-folder option, off by default.

**Stage 5: Aggregate.** Build `FileFacts` per file:
- quality, from confidence-ordered augmenters (I10). The source order is grab release name, file name, folder name, client item, probe; the probe wins on resolution and codec, the name wins on source (WEB-DL vs Blu-ray);
- revision, languages (the probe wins for audio languages), release group, edition, `SceneName` and indexer flags from the grab;
- the custom-format score, computed by the evaluation agent's engine on these facts.

**Stage 6: Decide.** This is a list of specs over `(unit, facts, identification, slot state)`. Each spec returns `Accept` or a typed `Rejection`.

```go
type Rejection struct {
    Code   RejectionCode      // stable enum, the API and event contract
    Class  RejectionClass     // Transient (retry) | Permanent | NeedsUser
    Scope  RejectionScope     // File | Unit | Request
    Params map[string]string  // e.g. {"missing":"12","disc":"2"}, {"existing":"Bluray-1080p","new":"WEBDL-1080p"}
}
```

The rejection codes are kind-neutral where possible. Sonarr's 37 codes (`ImportRejectionReason.cs`) collapse into these groups:

| Group | Codes (class) |
|---|---|
| Source | `FileLocked` (T), `SourceMissing` (P), `SourceNotAccessible` (NU: likely a missing remote path mapping, raised as a health alert), `Unpacking` (T), `UnsupportedExtension` (P), `UserRejectedExtension` (P), `DangerousFile` (P), `ExecutableFile` (P), `ArchiveFile` (P), `ProbeFailed` (NU), `NoAudioTrack` (P), `Sample` (P, benign), `SampleIndeterminate` (NU) |
| Identification | `Unparseable` (NU), `UnknownItem` (NU), `AmbiguousMatch` (NU), `GrabMismatch` (NU), `FolderMismatch` (NU), `UnverifiedSceneNumbering` (NU), `MissingAbsoluteNumber` (NU), `TitleRequired` (T until the title appears or a timeout, from `EpisodeTitleRequired`), `PartialSeason`/`FullSeasonRequired`/`MultiSeason`/`SplitEpisode` (NU), `DistanceTooHigh` (NU), `MissingTracks` (NU), `UnmatchedFiles` (NU), `FewerTracksThanExisting` (NU), `WrongMedium` (P: an audio file for an ebook-only target) |
| Slot | `NoSlotAccepts` (P: the quality is not allowed in any slot), `SlotAmbiguous` (NU) |
| Upgrade | `AlreadyImported` (P, benign), `NotQualityUpgrade`/`NotRevisionUpgrade`/`NotCustomFormatUpgrade` (P), `ExistingFileCoversMoreItems` (P) |
| Destination | `MinimumFreeSpace` (T), `RootFolderMissing` (T → NU), `DestinationOccupied` (NU) |

Kind-specific specs register with the framework: Sonarr's season-pack specs, Lidarr's `MoreTracks` and `NoMissingOrUnmatched`, Bookshelf's medium check.

**Slot selection (D4)** is itself a spec:
- If the grab named a slot, the file goes there, provided the quality is allowed in that slot's profile.
- Otherwise, the candidates are the item's slots whose profile allows the file's quality (quality axes are per kind, so an EPUB can only fit an ebook slot).
- Among those, prefer an empty slot or one where the file is an upgrade. If more than one remains, pick the slot whose cutoff the file fits most closely. A tie gives `SlotAmbiguous`.
- A file belongs to **exactly one slot**.

**Stage 7: Plan.** For every accepted unit, the planner computes:
- the destination path(s) with the naming engine (§3.4), using the slot's root folder and naming scheme;
- the transfer method (§3.5);
- the **retire set**: the existing files in that slot the unit replaces, and their extras;
- which extras to carry (§3.6);
- the content transforms (tags, ComicInfo.xml, CBZ conversion, CUE split);
- the extras to write (NFO and images) and the media servers to refresh.

The plan is persisted as `ImportOp` rows before any filesystem change. The **manual-import UI and rename/retag previews are simply plans that are not executed**. The user edits the plan (another item, slot or release, or partial acceptance), and the edited plan runs through the same executor.

**Stage 8: Execute.** Journaled; see §3.8.

**Stage 9: Post-import.** Write extras (atomically, and only if their content hash changed), emit events through the outbox, enqueue a debounced media-server refresh, and return the `ImportOutcome` to the download lifecycle. The outcome is:
- **Complete** when every expected item from the grab is imported, or rejected only for benign reasons (`Sample`, `AlreadyImported`);
- **Partial** or **Blocked** otherwise, with typed reasons. The lifecycle maps Blocked to `ImportBlocked` and raises `ManualInteractionRequired`.

`Transient` rejections are retried by the import job with backoff (for example `FileLocked`, `MinimumFreeSpace`, `TitleRequired`) before the outcome becomes Blocked.

### 3.3 Data types (proposed to the library agent)

```go
type MediaFile struct {
    ID          FileID
    RootID      RootFolderID      // path = root.Path + RelPath; one row rewrite per root move, not per file
    RelPath     string            // relative to the root folder, including the item folder
    Size        int64
    MTime       time.Time
    ContentKey  ContentKey        // dev/inode/size/mtime (+ sampled hash) → probe cache
    Class       FileClass
    SlotID      SlotID            // exactly one slot (D4)
    Quality     QualityModel      // per-kind axes
    Languages   []Lang
    ReleaseGroup, Edition, SceneName, OriginalPath string
    IndexerFlags Flags
    Transfer    TransferRecord    // method used, source path and inode if the source was retained (seeding link tracking)
    ImportedAt  time.Time
    DownloadID  *DownloadID
}

type FileLink struct {           // M:N (D2): multi-episode files, tracks, audiobook parts, comic ranges/TPBs
    FileID   FileID
    ItemID   ItemID
    Part     *int                // audiobook part index / disc for music
    Coverage *Coverage           // comics: issue range or special version
}

type ExtraFile struct {
    ID        ExtraID
    OwnerFile *FileID           // subtitles, per-file NFO/thumb, lyrics
    OwnerItem *ItemID           // container-level: tvshow.nfo, poster.jpg, artist.nfo, cover.jpg
    Type      ExtraType          // Subtitle | Metadata | Image | Lyric | Other
    Writer    string             // "kodi", "plex", "subtitle-agent", "" for adopted/unmanaged
    Lang      *Lang; HI, Forced, Default bool
    RelPath   string; RootID RootFolderID
    Hash      []byte             // only rewrite when changed
    Managed   bool               // false = adopted from disk; never overwritten without opt-in
}

type ProbeResult struct { Key ContentKey; Schema int; Kind ProbeKind; Data json.RawMessage; ProbedAt time.Time }
```

**Paths.** A path is stored as `(root_id, rel_path)`, which answers the relative-versus-absolute question (§2.2.9). Moving an item within a root rewrites that item's rows in one SQL statement. Remapping a mount changes one root row.

### 3.4 Naming engine

**One engine, a `TokenSet` per kind, and pure functions.** `Render(template, NamingContext) → (path, []Warning)`. Previews, validation, sample output and rename all call the same function.

**Grammar** (the I13 contract, written out so that it can be re-implemented and fuzzed):

```ebnf
template   = { literal | escape | token } ;
escape     = "{{" | "}}" ;                         (* literal braces *)
token      = "{" prefix name [ ":" args ] suffix "}" ;
prefix     = { " " | "." | "-" | "_" | "[" | "(" } ;
suffix     = { " " | "." | "-" | "_" | "]" | ")" } ;
name       = word { sep word } ;                   (* e.g. "Series Title", "series.title", "Release-Date" *)
sep        = " " | "." | "-" | "_" ;
word       = alnum { alnum } ;
args       = argchar { argchar } ;                 (* [ ,a-z0-9+-], case-insensitive; no trailing space/dash *)
literal    = any character except "{" and "}" ;
```

**Rendering rules**, in order:
1. **Lookup.** Normalise `name` to its canonical form (case-folded, separators removed) and look it up in the kind's `TokenSet`, falling back to the common tokens. An unknown token is a validation error at save time, and renders literally at run time with a warning. Aliases exist only for token names that published presets use (Sonarr and Radarr names). No v2/v3 Whisparr dual set is kept (D7, D10).
2. **Value.** A token returns a typed `Value`: a string, a number, a date, a list (for example performers or languages), or empty.
3. **Args** are parsed by the token's own arg spec:
   - `:0…0` pads a number (`{season:00}`, `{track:00}`, `{issue:000}`);
   - `:N` truncates text to N characters with an ellipsis (whether `:-N` truncates from the start must be checked against Sonarr's tests before it becomes part of the grammar);
   - `:EN+DE` / `:-EN` filter a language list;
   - `:Name` selects a custom format (`{Custom Format:HDR}`);
   - `:DE` selects a translation (`{Movie Title:DE}`).
4. **Separator.** When `name` contains a separator, internal spaces in the value are replaced by that separator (`{Series.Title}` → `The.Office`).
5. **Case.** An all-lower-case token name gives lower-case output and an all-upper-case name gives upper-case output. Otherwise the value is unchanged.
6. **Affixes.** Prefix and suffix are emitted only when the value is non-empty.
7. **Clean-up after the whole string is built:**
   - collapse repeated separators (`([- ._])\1+`);
   - remove empty bracket pairs;
   - trim leading and trailing separators per path component;
   - apply colon replacement (I14);
   - replace or delete illegal characters (`ReplaceIllegalCharacters`);
   - escape reserved device names;
   - strip trailing dots and spaces per component (for Windows and SMB);
   - enforce **255 UTF-8 bytes per component**, by first shortening the longest truncatable text token, then hard-truncating. This is done once, in the engine, not per app.
8. `/` in a template separates path components. Tokens are sanitised per component, so a `/` inside a value can never create a directory.

**Template slots per naming scheme:**

| Kind | Templates |
|---|---|
| TV | `ContainerFolder` (series), `SubFolder` (season; specials), `File.Standard` / `File.Daily` / `File.Anime` |
| Movie | `ContainerFolder`, `File` |
| Music | `ContainerFolder` (artist), `File.Standard`, `File.MultiDisc` (the album folder is part of the file template, as in Lidarr) |
| Book | `ContainerFolder` (author), `File` (may contain the book folder), `File.Part` (multi-file audiobooks) |
| Comic | `ContainerFolder` (volume), `File.Issue`, `File.IssueNoTitle`, `File.SpecialVersion`, `File.VolumeAsIssue` (Kapowarr's five) |
| Adult | `ContainerFolder.Scene`, `File.Scene`, `ContainerFolder.Movie`, `File.Movie`, `File.Code` (the JAV/product-code format restored from v2, `JavEpisodeFormat`) |

- **Scope.** Each `TokenDef` declares its scope, Container or File. Container templates reject File-scoped tokens (quality, media info, release group, custom formats) at save time (I15).
- **Multi-item numbering.** A generic `RangeStyle` is applied where a file links to several numbered items:
  - TV episodes use Sonarr's six styles (Extend, Duplicate, Repeat, Scene, Range, PrefixedRange), reproduced from the Sonarr golden tests;
  - comic issue ranges render `#001-006`;
  - audiobook parts use `{Part:00} of {PartCount}`.
- **Token sets.** The common set covers ids, quality (Full/Title/Proper/Real), `MediaInfo *`, release group, custom formats, and original title and filename. The per-kind sets are the union of today's tokens (Sonarr/Radarr/Lidarr/Bookshelf/Whisparr v3/Kapowarr, listed in each analysis §3). Each token is a small typed function, not a regex branch in a 1,200-line builder. The Sonarr and Radarr `FileNameBuilder` test suites are the acceptance tests.
- **Configuration.** A `NamingScheme` is a set of templates plus options (rename on/off, colon mode, illegal-character mode, multi-item style). Schemes are named, per kind, and chosen per **slot** (D4), defaulting per root folder. D9 naming presets import into schemes. Profilarr's `namingTokens.ts` is not needed: the engine validates.
- **Rename.** A rename is a plan: `Render` for every file in scope, diffed against the current paths. Only changed paths become `ImportOp{Kind: Rename}` rows, so the preview and the execution are identical. Metadata changes (an episode title changing from "TBA") trigger a rename plan for the affected files when rename is on.

### 3.5 Transfer

```go
type TransferMethod int // Rename, Reflink, Hardlink, Copy, InPlace (library import / already in place)

type TransferPolicy struct {
    Mode         ImportMode   // Auto | Move | Copy (manual override)
    SourceRetain bool         // from the download lifecycle: the client still needs the source (seeding, or "copy" preference)
    Mutates      bool         // the plan will write tags / ComicInfo / convert → the destination must not share an inode with the source
    AllowHardlink, AllowReflink bool // per root folder
}
```

The method is resolved by trying each option in order and falling back on `EXDEV`, `ENOTSUP` or `EPERM`:
- **Retain source** (seeding, or Copy mode): reflink (`FICLONE` on Linux: btrfs, XFS, bcachefs, OpenZFS 2.2+; `clonefile` on macOS APFS), then a hardlink (**skipped when `Mutates`**), then a copy.
- **Release source** (Move): a rename on the same filesystem, then a reflink plus unlinking the source, then a copy with verification and then unlinking the source.
- A copy uses `copy_file_range` (which Go's `os.File.ReadFrom` uses on Linux, giving server-side copy on NFS 4.2 and SMB3). It is verified by size, and optionally by hash (a per-root option, useful on flaky network mounts).
- `Mutates` with a hardlink-only filesystem means a real copy. This cost is shown in the plan preview.
- The method actually used, and the source inode, are recorded in `MediaFile.Transfer`. The download lifecycle's "unlinked"/seeding housekeeping (Cleanuparr's `st_nlink` check) can then read it without scanning.

**Permissions and dates.** These are per-root options, applied after the method is known:
- `chmod`/`chgrp` for new files and directories, defaulting to inheriting the umask;
- the file date: none, air or release date, or import time (Sonarr's `FileDateType`).

Neither is applied to hardlinked destinations, because it would change the client's copy too; the plan warns instead. Directories are created with the configured mode, and empty source folders are cleaned for Move.

**Recycle bin.** One path per root folder, or global. Retired files keep `<root-relative dir>/`, and a collision gets a timestamp suffix. The recycle bin is on the same filesystem when possible, so retiring is a rename. Purging after N days is a scheduled job. Deleting with no recycle bin is deferred until after the commit (§3.8).

**Remote path mappings** are applied by the download lifecycle. Import only ever sees local paths. `SourceNotAccessible` raises a health alert that points at mappings or mounts.

### 3.6 Extras, tags and transforms

**Carried extras** travel with the unit from the download:
- **Subtitles** found next to a video (same stem, or in a `Subs/` folder) are matched to their video by stem. Language, HI and forced flags are parsed from the name, with the shared language parser (D1c). They are renamed to `<video stem>.<lang>[.hi][.forced].<ext>` and recorded as `ExtraFile{OwnerFile}`.
- Other extras, such as release `.nfo`, `.lrc`, `.cue` and `.log`, are imported only if their extension is in the per-kind allow-list (default: `.lrc`, `.cue` and `.log` for music; nothing for video).
- On upgrade, the old file's extras join the retire set (I9). The `file.upgraded` event lets the subtitle agent re-evaluate, because timings may not fit the new release.

**Metadata writers** implement one interface and run on file and item events:

```go
type MetadataWriter interface {
    ID() string                                   // "kodi", "plex", "abs", "comicinfo"...
    Kinds() []MediaKind
    ForItem(ctx, item, files) []ExtraSpec          // container-level (tvshow.nfo, poster.jpg, artist.nfo)
    ForFile(ctx, file, items) []ExtraSpec          // per-file (episode .nfo, -thumb.jpg)
}
type ExtraSpec struct { RelPath string; Content func() (io.Reader, error); Hash []byte }
```

| Writer | Kinds | Output |
|---|---|---|
| Kodi / Jellyfin / Emby (Jellyfin and Emby read Kodi NFO) | TV, movie, music, adult | `tvshow.nfo`, episode and movie `.nfo` with `<uniqueid>`, `artist.nfo`/`album.nfo`, `poster`/`fanart`/`folder.jpg` |
| Plex | TV, movie | `.plexmatch` (Sonarr's `Consumers/Plex`), images |
| ComicInfo | comics | `ComicInfo.xml` **inside** the CBZ. This is a content transform, so it is staged (below), and Komga and Kavita read it |
| Audiobookshelf (open question) | audiobooks | `metadata.json` or OPF sidecar |

The writers for dead devices (Roksbox, WDTV), Emby Legacy and Kometa are not ported (comparison §3.1). Writers are idempotent: they write to a temporary file, rename it into place, and skip the write when the hash is unchanged. Files that are adopted and unmanaged are never overwritten unless the user opts in (I18).

**Tag writing** (I17) has the modes Never, NewFiles, AllFiles and Sync, plus Scrub and EmbedCover, set per kind and root.

| Format | Tag writer |
|---|---|
| Audio (music, audiobooks; ID3v2, MP4 atoms including m4b chapters, Vorbis, FLAC) | TagLib |
| EPUB | Native OPF rewrite: title, creators, identifiers (ISBN/ASIN), series (`calibre:series` and EPUB 3 `belongs-to-collection`), language |
| MOBI, AZW3, PDF | Read-only in v1 |

Tag writing and all other content transforms run **on the staged destination copy before commit** (§3.8). A retag of existing files is a plan with `Mutates=true`, and it copies first if the library file is still hardlinked to a seeding source.

**Other transforms**, which also run only on staged copies:
- CBR/CB7 → CBZ conversion, which drops RAR *writing* (Kapowarr's proprietary `rar`, §2.2.14);
- splitting a multi-issue archive;
- FLAC+CUE splitting. Unpackerr/xtractr does this today and then calls Lidarr ManualImport; here it is a music pre-group transform instead.

### 3.7 Scanning and watching

- **Scan** (the authority) runs for a scope: root, container or item folder.
  1. List the files and join them with the `MediaFile`/`ExtraFile` rows by `(root, rel_path)`.
  2. **Missing** files are marked missing. The row is kept for a grace period, because a mount outage must not wipe the library, and a health alert fires if a whole root is empty or unreachable. After the grace period the row is deleted, with `file.deleted{reason: Missing}`.
  3. **Changed** files (size or mtime) are re-probed, and quality and facts are re-derived.
  4. **New** files inside a known item folder go through the pipeline with `Source=Rescan, Mode=InPlace`. They are auto-accepted only when identification is `Matched` and the file is not an upgrade conflict. Otherwise they land in the **unmapped files** list for manual import. New files outside any item folder are handled as library-import proposals.
- **Watch.** An fsnotify watcher on each root schedules a debounced scan of the affected *container folder*, not the whole root (Lidarr debounces a full scan, `RootFolderWatchingService.cs:45-58`). inotify does not see changes made by other NFS/SMB clients, so watching is best-effort. A periodic scan per root (default 12 h, with a light mtime-only pass) remains the backstop.
- **Import-originated changes don't trigger the watcher.** The executor records the paths it touches, and the watcher suppresses events for them for a short window.

### 3.8 Execution, crash safety and HA

**Journal.** Every filesystem mutation is an `ImportOp` row with a deterministic id and a state machine:

```
Planned → Staged → Transformed → Retired → Committed → Recorded → Cleaned → Done
                         └──────────── Failed (with compensating actions) ────────────┘
```

| Step | Action | Postcondition, checked on replay |
|---|---|---|
| Stage | Transfer the source to `<dest dir>/.<name>.br-<opid>.partial`, with the method from §3.5. For a same-filesystem Move, create a **hardlink** to the staging name, so both names exist until commit; if hardlinks are unsupported, the rename itself happens at Commit | The staged file exists with the expected size |
| Transform | Tags, ComicInfo.xml, conversion, applied to the staged copy only | Transform marker hash recorded |
| Retire | Rename each old file (and its extras) to the recycle bin, or with no recycle bin to `.<name>.br-<opid>.retired` beside it | Old paths are absent and the retired paths recorded |
| Commit | `rename(staged, final)` (atomic, on the same filesystem by construction) | The final path exists, and the inode matches the staged one |
| Record | One DB transaction: upsert `MediaFile`, `FileLink` and `ExtraFile` rows, delete the retired rows, write history, write the **outbox** events, and set op = Recorded. **All the units of one album or audiobook share one transaction**, so partial albums never appear | The rows exist |
| Clean | Unlink the Move source (if not retained), delete `.retired` files, remove empty source directories | The source is absent |

- **Recovery.** A recovery job runs at start-up and when a lease is taken over. It walks non-Done ops and rolls each forward where the postconditions allow it. Otherwise it compensates: restore retired files to their original paths and delete the staged files.
- Because staging happens *before* retiring, and retiring is a rename, an item never loses its file to a crash or a failed copy. This fixes §2.2.6.
- Each step checks its postcondition first, so a replay is safe (idempotent).

**HA (D14).**
- **Jobs.** Import, scan, rename, retag and recovery run on the durable DB job queue and can be claimed by any node. The job key (for example `import:download:<id>:<attempt>`) makes duplicate deliveries collapse into one job.
- **Leases.** A job holds a **DB lease per container** (series, movie, album-artist, author or volume folder) through Plan and Execute, renewed by heartbeat. Two nodes never plan against the same slot state at once, and a crashed node's lease expires into recovery. Leases are DB rows or advisory locks, not `flock`, which is unreliable on NFS.
- **Shared storage.** It must be mounted at identical paths (D14), and `(root_id, rel_path)` removes any per-node path divergence.
- **Hardlinks and reflinks need the download and media paths on the same filesystem.** On Kubernetes that means one volume or PVC for both. A health check tests this per (download client, root folder) pair with a test link at start-up.
- **Heavy work.** Probing, copying and hashing run on bounded per-node worker pools, separate from the scheduler, with global concurrency limits per root, because network mounts saturate quickly.
- **Singletons.** The watcher is a leader-elected singleton. A new leader re-arms its watches and schedules a light scan of every root, because events were missed during failover.
- **Probe cache.** It is in the DB, so a probe done on one node serves all nodes.

### 3.9 Events emitted

All events are written to the outbox in the Record transaction; the runtime agent handles delivery, SSE, webhooks, notifications and metrics.

| Event | Payload (abridged) | Main consumers |
|---|---|---|
| `import.completed` | request id, download id, outcome, per-unit results, transfer methods | Download lifecycle (tracked-download state, seeding and cleanup), notifications |
| `import.rejected` | unit, `[]Rejection`, candidates | UI (manual import), notifications, alerts for NeedsUser |
| `import.manual_interaction_required` | download id, summary | notifications, request users (Seerr-style issue) |
| `file.imported` | file, items, slot, source (download, rescan, library, manual) | availability and requests, subtitle agent (search), media-server refresh, history |
| `file.upgraded` | new file, retired files, recycle paths | subtitle agent, notifications, refresh |
| `file.deleted` | file, reason (Upgrade, Missing, Manual, ItemDeleted, SlotRemoved) | subtitles, availability, refresh |
| `file.renamed` / `file.moved` | old → new paths | subtitle agent (none needed: extras move with the file), refresh |
| `file.retagged` | file, tag diff | notifications (Lidarr's `TrackRetag`) |
| `extra.written` / `extra.deleted` | extra, owner | refresh (also fired when the subtitle agent writes a subtitle, replacing Bazarr's Rescan and refresh calls) |
| `library.scan_completed` | scope, counts (new, changed, missing, unmapped) | UI, health |
| `library.unmapped_found` | root, folders or files | UI (library import) |

**Media-server refresh** is a subscriber to the file and extra events. It batches per server with a short debounce (Radarr's `MediaServerUpdateQueue` pattern), deduplicates container paths, and issues **path-scoped** refreshes where the server supports them:
- Plex partial scan by path;
- Jellyfin/Emby `Library/Media/Updated`;
- Kodi `VideoLibrary.Scan` with a directory;
- Subsonic/Navidrome `startScan`;
- Audiobookshelf and Kavita/Komga library scans;
- Stash `metadataScan` with paths, optionally followed by identify with StashDB ids.

The servers are the same client objects the requests agent uses (comparison §2.2).

---

## 4. Walkthroughs

### 4.1 A TV season pack with one sample file

The series is "Show" (TVDB), with slot `hd` (profile HD-1080p, root `/tv`, scheme "Standard"). Season 2 has 10 episodes; E01–E03 already have HDTV-720p files. The torrent `Show.S02.1080p.WEB-DL.DDP5.1.H.264-GRP/` contains 10 `.mkv` files, `Sample/show.s02e01.sample.mkv`, `GRP.nfo` and `Subs/Show.S02E0{1..10}.English.srt`. The client is still seeding.

1. **Intake.** The lifecycle sends an `ImportRequest{Source: Download, Paths:[…], Grab:{series, slot hd, S02E01–E10, parsed release}, SourceRetain: true}`. The job key is `import:dl:abc:1`, and the job takes a lease on container "Show".
2. **Discover.** 11 videos, 10 subtitles, 1 NFO. The NFO extension is not in the TV allow-list and is ignored silently.
3. **Probe.** ffprobe runs 11 times (cache misses). The sample's runtime is 41 s; each episode's is about 44 min.
4. **Group.** 11 video units, with folder context "season pack S02".
5. **Identify.** The parser reads `S02Exx` from each file name and the release name agrees (grab check passes). The sample also resolves to S02E01.
6. **Aggregate.** Quality WEBDL-1080p from the release name, confirmed by the 1920×1080 probe. Audio language comes from the probe. The CF score is computed.
7. **Decide.**
   - The sample fails `Sample`: 41 s is below the 600 s tier for a 44-minute episode (P, benign). It is excluded before the duplicate-target check.
   - E01–E03 pass the upgrade check (WEBDL-1080p > HDTV-720p); E04–E10 fill empty slots.
   - Slot `hd` comes from the grab.
8. **Plan.**
   - Each file's destination is `/tv/Show/Season 02/Show - S02E01 - Pilot Part 2 [WEBDL-1080p].mkv`.
   - Transfer: `SourceRetain` and no mutation → reflink if supported, else hardlink, else copy.
   - The retire set is the three HDTV-720p files and their old extras.
   - Each `.srt` becomes `…S02E01 - Pilot Part 2 [WEBDL-1080p].en.srt`, owned by its episode file.
9. **Execute.** Stage 10 videos and 10 subtitles → retire 3 old files to `/recycle/Show/Season 02/` → commit the renames → one Record transaction per unit.
10. **Post-import.**
    - Events: `file.upgraded` ×3, `file.imported` ×7, and `import.completed{Complete}`, because the only rejection is benign.
    - The lifecycle marks the download Imported and keeps seeding, since the library holds separate reflinks or hardlinks.
    - Plex receives one refresh for `/tv/Show/Season 02`.
    - The subtitle agent sees `file.upgraded` and re-checks E01–E03 against the language profile.

### 4.2 A music album with a disc missing

The album is "Artist – Album", a 2×CD release with 24 tracks; the album is `AnyReleaseOk`. The download `Artist - Album (2008) [FLAC]/CD1/01-12.flac` plus `cover.jpg`, `.cue` and `.log` has no `CD2`.

1. **Discover and probe.** TagLib reads 12 files. The tags carry MB release-group and release ids, and track numbers 1–12 with disc 1/2.
2. **Group.** One album unit. The multi-disc merge finds only `CD1`.
3. **Identify.** The candidates are the tagged MB release (2×CD, 24 tracks), the other releases of the grabbed album, and any local fuzzy matches. Distance and Munkres give:
   - the 2×CD release at 0.09: `missing_tracks` 12/24 × 0.6 weight, everything else near zero;
   - a 1×CD "Standard" release with the same 12 recordings at 0.02.

   Both are releases of the **same album**, so there is no ambiguity. Because `AnyReleaseOk` is set, the identifier chooses the 1×CD release with 0 missing tracks. No fingerprinting runs, because the best distance is at most 0.15.
   - *Variant:* if no 12-track release exists, the only candidate is the 2×CD release with 12 missing tracks. `MissingTracks{missing: 12, media: "2"}` (NeedsUser) fires for this new download (I8), the unit is **Blocked**, `import.completed{Blocked}` is sent, the lifecycle marks the download `ImportBlocked`, and `ManualInteractionRequired` fires.
   - In manual import, the user sees the plan with disc 2 greyed out and chooses "import partial". That edits the plan by setting `AllowPartial` on the unit and re-runs Decide. The 12 files import, and the 12 disc-2 tracks stay wanted.
4. **Aggregate and decide.** Quality is FLAC 16-bit from the probe. The album-level upgrade check passes against no existing files.
5. **Plan.**
   - With the 1×CD release, the Standard track format applies: `/music/Artist/Album (2008)/Artist - Album - 01 - Title.flac`. With the partial 2×CD import, the MultiDisc format adds `CD 01/`.
   - Tag writing is `NewFiles`, so the plan sets `Mutates=true`. Transfer is therefore reflink or copy, never hardlink (§2.2.7 fixed).
   - `.cue` and `.log` are carried as extras; `cover.jpg` is adopted as the album image.
   - The monitored release switches to the one chosen (Lidarr's `ImportApprovedTracks.cs:125-131` behaviour).
6. **Execute.** Stage 12 files, write the MB tags on the staged copies, commit, and **record the whole album in one transaction**.
7. **Post-import.** Events `file.imported` ×12 (and `album.release_switched` from the library agent). Navidrome receives `startScan`.

### 4.3 An m4b audiobook and an epub ebook of the same book, into two slots

The book is "Title" by Author, with two slots:
- `ebook`: profile EPUB > AZW3 > PDF, root `/books`, preferred ebook edition E1 (ISBN 978…);
- `audio`: profile M4B > MP3, root `/audiobooks`, preferred audio edition E2 (ASIN B0…).

A manual import (or a mixed download) holds `Title.epub` and `Title.m4b`.

1. **Probe.**
   - EbookProber reads the OPF: title, author, `urn:isbn:978…`, language `en`.
   - AudioProber (TagLib) reads MP4 atoms: title, author (©ART), narrator, ASIN, 42 chapters, 11 h 3 m.
2. **Group.** The two files are in different media, so they form two units.
3. **Identify.** A `DistanceIdentifier` with the book weights runs for each unit:
   - the epub's ISBN matches edition E1 (weight 10); distance 0.01;
   - the m4b's ASIN matches edition E2 (weight 10); distance 0.02, with the audio-format bias favouring "Audible Audio" editions.

   Both resolve to the same work, which is not ambiguous because the target item is the same book.
4. **Slot selection.**
   - The EPUB quality is allowed only in `ebook` (`WrongMedium` would reject it for `audio`).
   - M4B is allowed only in `audio`.
   - Each slot is empty, so each file is accepted into its own slot and linked to its own edition. **This requires the library model to allow a per-slot edition; see §6.**
5. **Plan.**
   - The ebook goes to `/books/Author/Title (2019)/Title - Author.epub` with the slot `ebook` scheme. Tag writing `NewFiles` for EPUB is a native OPF rewrite that adds series and identifiers, so `Mutates` applies.
   - The audiobook goes to `/audiobooks/Author/Series Name/03 - Title/Title.m4b` with the slot `audio` scheme. The MP4 atoms (title, album=book, artist=author, composer=narrator, series, ASIN) are written to the staged copy.
   - If the audiobook were 20 MP3 parts, it would be one unit with `FileLink.Part` 1–20 and `File.Part` naming (`{Book Title} - Part {Part:00} of {PartCount}`).
6. **Execute.** Each file is staged, transformed and committed independently; the two units lock the same container, so they are serialised.
7. **Post-import.**
   - Events: `file.imported{slot: ebook}` and `file.imported{slot: audio}`.
   - Refreshes: Kavita (or Calibre, open question §6) for `/books`, and Audiobookshelf for `/audiobooks`.
   - Requests and availability are per slot: "ebook available" and "audiobook available".

### 4.4 Library import of an existing, unmanaged movie folder

Root `/movies` holds `Blade Runner (1982) [Final Cut]/` containing `Blade.Runner.1982.Final.Cut.1080p.BluRay.x264-GRP.mkv`, `movie.nfo` (Kodi, with `<uniqueid type="tmdb">78</uniqueid>`), `poster.jpg` and `….en.srt`.

1. **Scan.** A root scan lists this folder as **unmapped**, because no item's container maps to it, and emits `library.unmapped_found`.
2. **Proposal** (library import, `AllowRemote`):
   - The parser gives "Blade Runner", 1982, edition "Final Cut".
   - The Kodi NFO detector reads the TMDb id 78.
   - `CatalogLookup(tmdb:78)` matches exactly. An id match is unambiguous, so the proposal is pre-selected. A title-only match would need confirmation if it were within the ambiguity margin.
   - The user confirms, choosing the slot `hd` (the default is inferred from the probe: 1080p fits `hd` and not `uhd`). The library agent creates the Movie item, with the container set to this existing folder.
3. **Import job** `{Source: LibraryImport, Mode: InPlace}`:
   - The probe gives 1080p, H.264, 117 min, so the file is not a sample.
   - Identification is trivially scoped to the new item (`ScopeItem`) and parses consistently.
   - Quality is Bluray-1080p and edition "Final Cut".
4. **Plan.**
   - Transfer is `InPlace`. Rename on library import is off by default, so the path is unchanged. If it is on, the plan shows the rename diff, and the folder is renamed only when the scheme's container template differs.
   - The `.srt` is adopted as `ExtraFile{Subtitle, en, Managed:true}`, since subtitles are always managed.
   - `movie.nfo` and `poster.jpg` are adopted as `Managed:false`. The Kodi writer, if enabled, won't overwrite them unless "take over existing metadata" is set.
5. **Execute.** No staging is needed; there is only a Record transaction (`MediaFile` with `Transfer=InPlace`, the links and history).
6. **Post-import.** `file.imported{source: library}` fires. There is no media-server refresh, because nothing on disk changed.

---

## 5. Simplifications compared with the references

| Area | References | Here | Why it's safe |
|---|---|---|---|
| Import pipeline | 6 forked `ImportDecisionMaker`s + Kapowarr post-processing | One pipeline; per-kind `Grouper`/`Identifier`/`Prober`/`TokenSet` plus spec registrations | The per-kind logic (I3–I5) is kept and ported with its fixtures |
| Naming | 6 `FileNameBuilder`s + Kapowarr `naming.py` + Profilarr `namingTokens.ts` | One engine, an EBNF grammar, typed tokens per kind | The grammar is Sonarr's; the Sonarr/Radarr builder tests are the acceptance suite |
| Probing | Up to 4 ffprobe calls per file in Sonarr + Bazarr knowit (pickled) + separate TagLib paths | One probe per content key, cached in the DB and shared by all consumers | The same data is consumed; lazy re-probe on schema bump |
| Subtitle files | Arr `SubtitleFiles` + Bazarr `table_*_subtitles` + `Rescan*` round trips + path mappings | One `ExtraFile` table keyed to the media file; the subtitle agent writes through it | One process and one filesystem (Bazarr §9.6) |
| Transfer | Hard-coded btrfs/zfs checks for reflinks (Radarr only), script import | Try reflink → hardlink → copy with fallback; `Mutates` forbids shared inodes | Fixes tag-on-hardlink corruption; covers XFS and APFS |
| Script import | `ScriptImportDecider` + `Sonarr_*` env | Dropped. Content transforms are built in (tags, CBZ, CUE); the event and webhook stream covers post-processing (see §6) | D7 drops the env contract |
| Upgrade ordering | Recycle the old file, then transfer the new one | Stage → retire → commit → record, journaled | Removes the no-file window on crash |
| Paths | Relative (Sonarr/Radarr) vs absolute (Lidarr/Bookshelf/Kapowarr) | `(root_id, rel_path)` everywhere | One row changes on a mount move |
| Change detection | Watcher in 2 apps, rescans elsewhere, Bazarr via SignalR/webhook/hourly sync | Scan is authoritative; a watcher gives targeted debounced scans; in-process events | Nothing depends on the watcher for correctness |
| Manual import and previews | Separate manual-import services and APIs per app, and a separate rename preview | Plans are previews; editing a plan and executing it is manual import | One code path, so the preview equals the execution |
| Out-of-band fixes | Unpackerr "left queue = imported", Cleanuparr text-matched force import | Typed `ImportOutcome` and `Rejection` classes consumed in-process | No text matching |
| Media-server refresh | Per-app notifications + Bazarr + Seerr scanners | One event subscriber, batched and path-scoped | The same servers and fewer calls |
| Calibre | Calibre owns placement for Calibre roots | Proposed: Calibre as a downstream target only (§6) | Keeps naming, the journal and the recycle bin authoritative |
| Comics archives | CBR/RAR writing with the proprietary `rar` | Read RAR/7z; write CBZ only | Pure Go |
| Metadata writers | Kodi, Plex, Roksbox, WDTV, MediaBrowser, Kometa per app | Kodi(+Jellyfin/Emby), Plex, ComicInfo, optional ABS; one interface | Dead devices dropped (comparison §3.1) |

---

## 6. Issues with decisions, and open questions

### 6.1 Issues with decisions

1. **D4 (slots) and books need a per-slot edition.** Bookshelf allows one monitored edition per book (`Edition.Monitored`, [bookshelf.md](../analysis/bookshelf.md) §2), but ebook and audiobook slots need *different* editions (an ISBN ebook edition and an ASIN audio edition). The library model must attach the "preferred edition" (and music's "monitored release") to the **slot**, not the item. This does not conflict with D4, but D4's example ("ebook and audiobook" slots) does not work without it.
2. **D4 does not say whether one file can serve two slots.** This design assumes it cannot: each file has exactly one slot, and `SlotAmbiguous` is raised when the selection rule ties. If users expect a single 1080p file to satisfy both an "any HD" slot and a "1080p" slot, D4 needs a clarification. Hardlinking one file into two roots is possible but complicates upgrades and deletion.
3. **D10's "reject ambiguous everywhere" applied to distance matching** can reject imports that Lidarr accepts today. For example, Lidarr's `SucceedWhenManyAlbumsHaveSameTitle` fixture relies on picking the lowest distance. The design limits the rule to candidates that are **different items**, with a per-kind margin. The margin values need tuning against the Lidarr and Bookshelf fixtures, and must be confirmed as consistent with D10's intent.
4. **D18 (single-user desktop to Kubernetes) and external binaries.** `ffprobe` and `fpcalc` are native executables. The Tauri and OS-service installers (D12, D18) and the container images must bundle them per platform. Pure Go is not realistic for ffprobe. A WASM build via wazero (D15's runtime) is possible for TagLib, but not practical for ffprobe at speed.
5. **D14 and hardlinks on Kubernetes.** Seeding-friendly imports need the download and media paths on the same filesystem. D14 says only "shared storage mounted at the same paths". The deployment guidance should require one volume for downloads and media when hardlinks or reflinks are wanted. Otherwise every seeding import is a full copy. This design adds a health check for it.
6. **D7 and D15 against script import.** Dropping `ScriptImportDecider` removes a hook some users rely on for remuxing or transcoding at import. The D15 WASM plug-ins cannot run ffmpeg. The options are:
   - a built-in "external processing" hold state: an import holds until a webhook callback marks the file ready;
   - a declared host-function capability for plug-ins to request a transcode;
   - accepting the loss.

   This needs a decision.
7. **D3 AcoustID key.** AcoustID application keys are non-commercial. Under D3b, shipping a project key is allowed and a user may override it. Fingerprinting stays off by default and is escalation-only, which keeps the call volume low. This is only flagged to confirm that the key terms fit AGPL distribution (D5).

### 6.2 Open questions

1. **Calibre.** Keep Bookshelf's "Calibre owns placement" roots, or make Calibre a downstream target (add a copy to Calibre after import, or notify Calibre's library to rescan)? The recommendation is downstream-only, because owning placement defeats naming, the journal and the recycle bin.
2. **The bookshelf fork loosened the match threshold from 0.20 to 0.50.** Which default should books use? The recommendation is 0.30 with the ambiguity margin, validated on the Bookshelf fixtures.
3. **Auto-adding unknown artists or authors found on disk** (Lidarr's `DiskScanService.Scan(addNewArtists: true)`). Keep it as a per-root option, off by default?
4. **A metadata writer for Audiobookshelf** (`metadata.json`/OPF) and **Stash**. Is a path-scoped Stash scan plus identify enough, or should the StashDB ids be pushed to Stash directly?
5. **EPUB tag writing.** Should the native OPF rewrite be on by default for new files, or opt-in like audio tags? Rewriting changes the file hash, which some readers' sync (KOReader, Kobo) treats as a new book.
6. **The grace period for missing files** before rows are deleted (proposed: 7 days), and the alert when a root disappears.
7. **Hash verification** of cross-filesystem copies. Should it be default-on for network roots? It doubles the read I/O.
8. **The Sonarr `TitleRequired` behaviour** (holding an import until an episode title replaces "TBA"). Keep it, with the Always / BulkSeasonReleases / Never setting (`EpisodeTitleRequiredType.cs`) and a timeout?
9. **Library-import rename default.** Keep the path (the default here) or rename to the scheme?
10. **Comic transforms** (CBR→CBZ, splitting ranges, ComicInfo.xml). Should they be on by default for new downloads? They always force a copy (`Mutates`), which uses more disk while seeding.
