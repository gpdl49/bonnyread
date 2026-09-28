# Design: Download lifecycle

Scope: everything from a **grab decision** to **handing files to import**, then what happens to the client payload afterwards (seeding, removal, housekeeping), plus the "when to search" side of backlog and upgrade scheduling. Replaces the download handling in the PVRs (Sonarr, Radarr, Lidarr, Bookshelf, Whisparr v2/v3), all of Cleanuparr and Unpackerr, Kapowarr's download queue, and the "when to search" parts of Cleanuparr's Seeker and Profilarr's upgrades.

Out of scope (other design docs): what to grab and how to search (acquisition/indexers, release evaluation, profiles), import mechanics and file identification (import & file management), the library and slot model, the job queue, leases and events (runtime/HA), API and auth.

Binding decisions used here: D2 (download handling is phase 1), D4 (version slots), D7 (no compatibility façades; metrics and alerts are first-class), D11 (DDL is a third protocol, phase 3, interfaces DDL-ready now), D13 (ent), D14 (HA, leader singletons, DB job queue), D15 (WASM plug-ins), D16 (racing later), D19 (redesign from outcomes).

Paths are relative to `submodules/`. Abbreviations: `Sonarr/` = `sonarr/src/NzbDrone.Core/`, `CUI/` = `cleanuparr/code/backend/Cleanuparr.Infrastructure/`, `Unp/` = `unpackerr/pkg/unpackerr/`.

---

## 1. Outcomes and invariants to preserve

These are what the user sees, or what must stay true. Everything else in the references is treated as implementation.

### 1.1 Outcomes

| # | Outcome | Evidence in the references |
|---|---|---|
| O1 | A grabbed release shows up in a queue with progress, ETA, status and the items it is for, until it is imported or fails. | `Sonarr/Download/TrackedDownloads/TrackedDownload.cs:7-47`, `Sonarr/Queue/QueueService.cs`; Seerr polls the Arr queue to show progress on requests (`seerr/server/lib/downloadtracker.ts:90,168`) |
| O2 | A finished download is imported automatically, with no user action, when it can be mapped to the grabbed items. | `Sonarr/Download/CompletedDownloadService.cs:66-126` (Check), `:128-210` (Import) |
| O3 | A download that cannot be imported is visibly **blocked**, says why, and raises a "manual interaction required" notification once. | `CompletedDownloadService.cs:285-300`, `ManualInteractionRequiredEvent.cs` |
| O4 | A download the client reports as failed (Usenet repair or unpack failure, encrypted RAR) is failed, blocklisted, and optionally searched again. | `Sonarr/Download/FailedDownloadService.cs:86-136`, `Clients/Sabnzbd/Sabnzbd.cs:99,138-147`, `RedownloadFailedDownloadService.cs:31-80` |
| O5 | A download that never makes progress (dead swarm, stuck magnet, crawling speed) is eventually given up on, replaced, and not grabbed again. | Cleanuparr stall, metadata and slow rules (`CUI/Features/Jobs/QueueCleaner.cs`, `CUI/Services/QueueRuleEvaluator.cs`); the Arrs only display `stalledDL` as a warning (`Sonarr/Download/Clients/QBittorrent/QBittorrent.cs:318-321`) |
| O6 | Unwanted or dangerous files are kept out: not downloaded where the client allows it, and never imported. | Cleanuparr Malware Blocker (`CUI/Features/MalwareBlocker/FilenameEvaluator.cs`), Arr `FailDownloads` at grab (`Sonarr/Download/TorrentClientBase.cs:262-320`) and import (`Sonarr/Download/RejectedImportService.cs:41-62`) |
| O7 | Archived releases (RAR sets, 7z, zip) are unpacked and imported, while the original archives keep seeding. | Unpackerr (`Unp/handlers.go`, `xtractr`); analysis `unpackerr.md` §9 |
| O8 | Failed releases are never grabbed again, and a replacement is searched for, without hammering indexers or looping on the same bad release. | Arr blocklist (`Sonarr/Blocklisting/Blocklist.cs`), Cleanuparr's one-search-per-tick queue and recurring-hash guard (`CUI/Features/ItemStriker/Striker.cs:20,86`) |
| O9 | Torrents keep seeding after import until the user's goals (ratio, time, inactivity) and the tracker's obligations are met, then are stopped or removed, with or without data. | Per-indexer seed criteria pushed at add (`Sonarr/Download/Clients/QBittorrent/QBittorrent.cs:109,185`, `Sonarr/Indexers/SeedCriteriaSettings.cs`), "Remove Completed" (`QBittorrent.cs:283-288`), Cleanuparr seeding rules (`CUI/Services/SeedingRuleEvaluator.cs`) |
| O10 | A torrent whose library file has been deleted or upgraded away can be cleaned up (it no longer backs anything). | Cleanuparr unlinked downloads (`CUI/Features/DownloadCleaner/Services/UnlinkedDownloadsService.cs`) |
| O11 | Torrents that are dead (no seeders) or deleted from the tracker (unregistered) are detected and dealt with. | `CUI/Features/DownloadCleaner/Services/DeadTorrentService.cs`, `CUI/Services/TrackerMessageClassifier.cs` |
| O12 | Leftover files in download directories that no client owns are quarantined, then purged. | `CUI/Features/DownloadCleaner/Services/OrphanedFilesCleanupService.cs` |
| O13 | Missing items and items below cutoff (quality or custom-format score) are searched for in a slow, fair trickle that does not exhaust indexer limits or flood the queue. | Cleanuparr Seeker (`CUI/Features/Jobs/Seeker.cs`), Profilarr upgrades (`profilarr/docs/backend/upgrades.md`, `profilarr/src/lib/shared/upgrades/filters.ts`, `selectors.ts`) |
| O14 | Private-tracker standing is protected: no hit-and-run deletes, failed private torrents are not deleted from the client by default. | `Sonarr/Indexers/SeedCriteriaSettings.cs:54` ("Under N leads to H&R"); Cleanuparr `removeFromClient = !isPrivate || DeletePrivate` (`cleanuparr.md` §3.1 step 6) |
| O15 | Downloads can go to several clients, chosen by protocol, priority and tags, falling back when one is unavailable. | `Sonarr/Download/DownloadClientProvider.cs:41-59,178-194`; Whisparr v3's fallback on failure (`whisparr-eros/src/NzbDrone.Core/Download/DownloadService.cs:83-107`) |
| O16 | Paths reported by a remote client are mapped to local paths. | `Sonarr/RemotePathMappings/RemotePathMappingService.cs`, `CompletedDownloadService.cs:307-325` |
| O17 | Users can manually fail, remove (with or without blocklisting, with or without searching again) and ignore downloads. | `FailedDownloadService.cs:73-84` (`skipRedownload`), `IgnoredDownloadService.cs:25-52` |
| O18 | A simulate (dry-run) mode shows what policies would do before they act. | `CUI/Interceptors/DryRunInterceptor.cs` |
| O19 | Blackhole (watch-folder) clients work for users without an API-capable client. | `Sonarr/Download/Clients/Blackhole/ScanWatchFolder.cs` |
| O20 | (Phase 3) Comics can be fetched by direct download from file hosts, with mirror fallback and host quota handling. | Kapowarr `kapowarr/backend/implementations/download_clients/*`, `kapowarr/backend/features/download_queue.py:652` (`_remove_all_of_service` on quota) |

### 1.2 Invariants

- **I1 Ownership.** Every download bonnyread starts is recorded **before** it is sent to a client, and is matched back by an identifier bonnyread knows (info hash, or a marker it put on the client item). Identity never depends on parsing the client's title. (Violated today: `Sonarr/Download/TrackedDownloads/TrackedDownloadService.cs:130-166` re-parses titles; Unpackerr keys everything by title, `Unp/starrpoll.go:146,170`.)
- **I2 Durability.** Download state survives restarts and failover exactly. (Violated today: Sonarr's tracked downloads live in an in-memory `ICached` rebuilt from history, `TrackedDownloadService.cs:46,67,262-275`; Cleanuparr's recurring hashes are a static set lost on restart, `Striker.cs:20`; Unpackerr restores 72 h of JSONL, `Unp/historyrestore.go`.)
- **I3 One owner per torrent.** Exactly one component decides when a client item is removed. (Violated today: the Arr removes at ratio, Cleanuparr removes at its own rules and skips anything "in any Arr queue", `cleanuparr.md` §3.4 step 3.)
- **I4 No re-grab of a blocklisted release**, identified by info hash for torrents and by (indexer, guid) for Usenet, with title+size as a fallback.
- **I5 No data deletion under a library file.** Payload data is never deleted while a library file still depends on it (hard link or pending import), or while another torrent in the client uses the same files (cross-seed).
- **I6 Private obligations first.** A private torrent is never removed from its client before the tracker's minimum seed ratio/time is met, unless a user explicitly forces it or the tracker has unregistered it.
- **I7 Importer never sees partial files.** Import starts only on a completed payload, and extracted output appears atomically.
- **I8 Per slot (D4).** A download targets one slot; queue, history, failure counting and redownload work per (item, slot).
- **I9 Time, not ticks.** A policy's outcome depends on how long a condition has held while bonnyread could observe it, not on how often a job ran; an outage (bonnyread, network or client) does not count towards "stalled".

---

## 2. How the references do it today, and what is accidental

### 2.1 The Arrs (Sonarr as the reference; Radarr, Lidarr, Bookshelf, Whisparr are copies)

- **Grab:** `ProcessDownloadDecisions` → `DownloadService` picks a client by protocol, priority, tags and a round-robin "last used" cache (`Sonarr/Download/DownloadClientProvider.cs:41-59,85-121`). The client add is followed by a `Grabbed` history row with the download id. For torrents, the `.torrent` file list is checked against the per-indexer `FailDownloads` extensions, and magnets cannot be checked (`TorrentClientBase.cs:262-320`).
- **Seed configuration** is pushed to the client at add time. At poll time it is looked up again by hash from download history with a one-hour cache (`DownloadSeedConfigProvider.cs:37-88`).
- **Tracking:** `RefreshMonitoredDownloadsCommand` runs every minute, plus a 5 s debounce after grabs and imports (`DownloadMonitoringService.cs:53-85,169-182`). Each client item is turned into a `TrackedDownload` held in an **in-memory cache**. Its state is recovered from the latest download-history event, and its media mapping from **parsing the client's title** (falling back to the grab history's source title) (`TrackedDownloadService.cs:97-218`). States: `Downloading, ImportBlocked, ImportPending, Importing, Imported, FailedPending, Failed, Ignored` (`TrackedDownload.cs:49-59`). There is a TODO for offline clients: their items are simply not refreshed (`DownloadMonitoringService.cs:100`).
- **Completion:** `CompletedDownloadService.Check/Import` (`:66-210`) validates the path, re-resolves the series by title, blocks "matched by ID" grabs (`:115-122`), and imports. Every non-import outcome is turned into **status message strings** (`"No files found are eligible for import in {0}"`, `:162`; `"One or more episodes expected…"`, `:189-209`). Only three typed rejection reasons auto-fail, and only when the indexer's `FailDownloads` allows it (`RejectedImportService.cs:41-62`). Everything else sits in `ImportBlocked` forever.
- **Failure:** the client's `Failed` status or `IsEncrypted` → `FailedPending` → `Failed` → blocklist and `DownloadFailedEvent` → `AutoRedownloadFailed` pushes an immediate episode or season search, **unthrottled and with no recurrence guard** (`FailedDownloadService.cs:86-136`, `RedownloadFailedDownloadService.cs:31-80`).
- **After import:** `DownloadEventHub` marks the item imported (optionally moves it to a post-import category, `QBittorrent.cs:57-70`) and removes it from the client **only if** the client has already *paused* the torrent at its seed limit (`QBittorrent.cs:283-288`, `DownloadEventHub.cs:50-85`). Removal is per client (`RemoveCompletedDownloads`, `RemoveFailedDownloads`, `DownloadClientDefinition.cs:17-18`).
- **Stall:** qBittorrent `stalledDL` and DHT-less `metaDL` become `Warning` with a message and nothing else (`QBittorrent.cs:318-340`). No timeouts exist.
- **Extraction:** none. Import rejects `_UNPACK_`/`_FAILED_` folders from Usenet clients (`Sonarr/MediaFiles/EpisodeImport/Specifications/NotUnpackingSpecification.cs`) and otherwise finds no files in a RAR set.
- **Backlog:** "Search all missing" and "Search cutoff unmet" are manual bulk commands. Upgrades otherwise arrive only through RSS.

### 2.2 Cleanuparr

An external janitor (`cleanuparr.md`):
- It polls every Arr queue (paged, 200 per page) and every torrent client every 5 min for rules, and **every 5 s** for the malware scan.
- It rebuilds the download → media mapping from queue records and guesses "pack" from group size.
- It records a **Strike row per job run**, with a threshold of at least 3 strikes and decay after 24 h of silence (`Striker.cs:30-86`, `cleanuparr.md` §2).
- Its failed-import rule **substring-matches the Arr's localised status messages** (`ArrClient.cs:147-150,474-495`).
- It removes items through `DELETE /queue/{id}?blocklist=true&skipRedownload=true`, then runs its own throttled search queue and polls `/command/{id}` for the result.
- Force-import calls the Arr's manual-import endpoints, with 6 h caches (`CUI/Features/Arr/ForceImport/ForceImportService.cs`).
- Seeding, unlinked (inode link count), dead and orphan cleanup act only on client items that are not in any Arr queue. Its state is carried in **client categories** (`cleanuparr-unlinked`, `cleanuparr-dead`), which a later seeding rule on that category deletes.

### 2.3 Unpackerr

It polls the Arr queues every 2 min and keys items **by title**. It extracts into `<path>_unpackerred` and moves the files back into the download folder. It infers "imported" when **the title leaves the Arr queue**, then deletes the extracted files after `delete_delay` (`Unp/handlers.go:66-97,264-345`). A snapshot-ready guard, queue-return resets, a 24 h stale sweep and a 72 h JSONL restore cover the races (`Unp/starrpoll.go:212`, `Unp/start.go:43`, `Unp/historyrestore.go`). The useful core is the `xtractr` library (MIT, pure Go): multi-volume RAR, 7z, zip, nested archives, passwords, zip-bomb limits.

### 2.4 Others

- **Kapowarr:** an in-process DDL queue that is persisted and re-prepped on restart. It has host quota handling (drops queued downloads for a host once its quota is hit), and torrent support through qBittorrent/Transmission with a `failing_download_timeout` (`kapowarr/backend/internals/settings.py:123`).
- **autobrr:** client-side guards at add time: max active downloads, ignore when slow, and a **reannounce loop** for torrents that are not registered yet on private trackers (`autobrr/internal/action/run_qbittorrent.go:125,206-260`, `run_transmission.go:148-160`).
- **Profilarr upgrades:** a filter DSL (nested AND/OR over library fields), selectors, round-robin across filters, and a **cooldown implemented as Arr tags** that are reset when a filter is exhausted (`profilarr/docs/backend/upgrades.md` "Cooldown").
- **Prowlarr:** per-indexer `SeedRatio`, `SeedTime`, `PackSeedTime` and grab limits (`prowlarr/src/NzbDrone.Core/Indexers/IndexerTorrentBaseSettings.cs`).

### 2.5 Accidental complexity (what D19 says to drop)

| Accident | Why it exists | Where |
|---|---|---|
| Tracked downloads as an in-memory cache rebuilt from history and title parsing | Legacy Arr design | `TrackedDownloadService.cs:46,97-218,262-275` |
| Status-message strings as the carrier of "why" | The Arr API had nothing typed; consumers then matched the text | `CompletedDownloadService.cs:162,189-209`; `ArrClient.cs:147-150,474-495` |
| Strikes: a row per cron run, a ≥3 minimum, a 24 h decay window | Cleanuparr can only sample from outside, on a schedule | `Striker.cs`, `QueueRule.Validate` |
| "Left the queue ⇒ imported", title-keyed maps, snapshot guards, tombstones, 24 h stale sweep | Unpackerr has no callback from the Arr | `Unp/handlers.go:66-97`, `starrpoll.go:212`, `start.go:43` |
| Extract into the download folder, then delete the extracted copy after a delay | The Arr could only import from the client's folder | `Unp/handlers.go:327-345`, `xtractr/queue.go:172` |
| `skipRedownload=true` plus a private search queue plus polling `/command/{id}` | The Arr's redownload was unthrottled and invisible to Cleanuparr | `cleanuparr.md` §3.1.5, §3.5 |
| Force import via `manualimport` GET + `ManualImport` command + history confirmation + 6 h caches | No typed override of "safe" rejections | `ForceImportService.cs` |
| Client categories as a state machine (`cleanuparr-unlinked` → a seeding rule deletes it) | Cleanuparr has no database of torrent state it owns | `UnlinkedDownloadsService.cs`, `DeadTorrentService.cs` |
| Skip "anything in any Arr queue", plus a 10 s wait | Two owners of the same torrent | `cleanuparr.md` §3.4 step 2-3 |
| Removal only when the client has paused at ratio | Sonarr could not know seed state otherwise | `QBittorrent.cs:283-288` |
| Inode link-count scans to find unlinked torrents | No provenance link between library files and torrents | `CUI/Features/Files/*HardLinkFileService.cs` |
| Triple polling (Arr 1 min, Cleanuparr 5 min and 5 s, Unpackerr 2 min) of the same clients | Three processes | all |
| Five near-identical per-client seeding-rule tables | Capability differences modelled as tables | `cleanuparr.md` §10 |
| Tag-based cooldown for upgrade searches | Profilarr has no database of the Arr's library | `profilarr/docs/backend/upgrades.md` |
| A CF-score syncer re-reading the library every 30 min | Cleanuparr cannot see scores when they are computed | `CustomFormatScoreSyncer.cs` |
| Seed config looked up again by hash from history with a 1 h cache | The grab context is not stored on the tracked download | `DownloadSeedConfigProvider.cs:37-88` |

---

## 3. Proposed design

### 3.1 Principles

1. **The `Download` row is the truth.** It is created at grab time, before the client is called, and holds the release snapshot, targets, slot, indexer, privacy and seed obligations. The client is an *observed* system: we store the latest observation, never re-derive identity from it.
2. **Two orthogonal axes.** *Phase* (what the download means for the library: downloading, importing, imported, failed) and *retention* (what is happening to the payload in the client: active, seeding, stopped, removed). A failed private torrent can keep seeding; an imported torrent can be dead. Mixing the two is why Cleanuparr needed categories as state.
3. **Conditions are durations, accumulated only while observable** (I9). Each condition keeps a seconds counter that grows by the observed Δt (capped) while the condition holds, and resets when it stops holding. This replaces strikes, decay windows and the connectivity gate with a single rule.
4. **Typed reasons everywhere.** Client adapters, the extraction stage and the importer return enums. Text appears only as a `detail` for display. Where a client only speaks text (SABnzbd `fail_message`, tracker messages), the adapter classifies it into an enum at the boundary. That is the one place where text matching is allowed, and it is unit-tested per client.
5. **Policies are data plus a pure function.** `evaluate(download, observation, policies, now) → []Action` has no I/O and is deterministic, so it can be tested with tables and run in simulate mode by construction.
6. **Push is a hint, polling is the truth.** A notification from a client triggers an immediate poll and is never trusted as state.

### 3.2 Components

```
                       acquisition (other doc)
                             │ Grab(release, slot, targets, client prefs)
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ downloads                                                                │
│  grab.Submitter ── job ──► client adapters (qbit, transmission, deluge,  │
│        │                   rtorrent, sabnzbd, nzbget, blackhole, aria2,  │
│        │                   ddl [phase 3])                                │
│        ▼                                                                 │
│  monitor.Poller (1 per client, leased) ──► observation ──► policy.Eval   │
│        │                        │                            │           │
│        │                        └──► phase transitions ◄─────┘           │
│        ▼                                                                 │
│  extract.Stage (jobs, any node, xtractr)                                 │
│        ▼                                                                 │
│  handoff ──► import (other doc) ──► ImportOutcome ──► reasonpolicy       │
│        ▼                                                                 │
│  failure.Handler (blocklist, client disposition, redownload intent)      │
│  seeding.Manager (leader tick; library events → links)                   │
│  housekeeping (leader tick: orphans, staging, vanished, retention prune)  │
│  backlog.Scheduler (leader tick → SearchIntent → acquisition)            │
│  contentpolicy (grab / post-metadata / import checks)                    │
└──────────────────────────────────────────────────────────────────────────┘
```

Package layout: `internal/downloads/{client, client/qbittorrent, …, grab, monitor, policy, extract, failure, seeding, housekeeping, backlog, contentpolicy}`.

### 3.3 Download-client abstraction

```go
type Protocol string // "torrent" | "usenet" | "ddl"

type Caps struct {
    Protocol          Protocol
    Labels            LabelKind   // None | Category (single) | Tags (multi) | Directory (Transmission-style)
    FileSelection     bool        // can skip files (qBit filePrio, Transmission files-unwanted, Deluge, rTorrent)
    FileListAtAdd     bool        // file list known before metadata (true for .torrent, false for magnet)
    TrackerStatus     bool        // per-tracker messages (qBit, Transmission; others: unsupported)
    SwarmCounts       bool        // seeders/leechers
    LastActivity      bool        // qBit last_activity (inactivity rules)
    AltSpeedProbe     bool        // qBit, Transmission
    PerItemSeedLimits bool        // client enforces ratio/time itself
    ExcludedGlobs     bool        // qBit excluded_file_names
    ReportsPrivate    bool
    ClientMarker      MarkerKind  // how we tag an item with our id: InfoHash | NzbName | PPParam | Tag
    Unpacks           bool        // client repairs and unpacks itself (SABnzbd, NZBGet)
    Progress          bool        // false for blackhole
    Push              PushKind    // None | CompletionHook (SAB/NZBGet scripts, qBit "run on finish", Transmission script-torrent-done)
}

type Client interface {
    Caps() Caps
    Add(ctx context.Context, req AddRequest) (ClientRef, error)        // idempotent on req.Marker
    List(ctx context.Context, scope ListScope) (Snapshot, error)       // ours (category/marker) or all (for cross-seed/orphans)
    Remove(ctx context.Context, ref ClientRef, deleteData bool) error  // idempotent: "not found" is success
    Health(ctx context.Context) error
}
// Optional, discovered by type assertion and gated by Caps:
type FileSelector   interface { Files(ctx, ref) ([]ClientFile, error); Skip(ctx, ref, idx []int) error }
type Labeler        interface { SetLabels(ctx, ref, add, remove []string) error }
type Stopper        interface { Stop(ctx, ref) error; Start(ctx, ref) error }
type Reannouncer    interface { Reannounce(ctx, ref) error }
type GlobPusher     interface { PushExcludedGlobs(ctx, globs []string) error }
type SeedLimitSetter interface { SetSeedLimits(ctx, ref, SeedLimits) error }
```

`Snapshot` carries `OK bool` and `Complete bool`. `Complete` is false when the listing looked collapsed, for example zero items where there were 500 (Cleanuparr's `ThrowIfTorrentListCollapsed`). No absence-based decision is taken on an incomplete snapshot.

`Observation` (normalised per item):
- `State ∈ {Queued, Metadata, Downloading, Paused, Checking, Completed, Seeding, Stopped, Failed, Warning}` plus `StateDetail` (typed: `MissingFiles`, `ClientError`, `DHTDisabled`, …);
- `Downloaded`, `Total`, `Wanted` (after skipped files) bytes; `DownRate`, `UpRate`, `ETA`;
- `Seeders`, `Leechers`, `Ratio`, `SeedingTime`, `LastActivity`, `Private`;
- `Tracker ∈ {OK, Unregistered, Down, Unknown}` (classified by the shared tracker-message classifier that the indexer layer also uses, seeded from `CUI/Services/TrackerMessageClassifier.cs`);
- `OutputPath` (local, after mapping), `Files []ClientFile` when cheap;
- `FailReason ∈ {None, UsenetRepairFailed, UsenetUnpackFailed, UsenetMissingArticles, Encrypted, DuplicateRejected, DiskFull, Other}`;
- `AltSpeedActive`.

**Supported clients in phase 1:** qBittorrent (API v2 only; drop `QBittorrentProxyV1`), Transmission (≥ 4.0 for labels), Deluge 2, rTorrent, SABnzbd, NZBGet (current only; drop the v12/v13 paths), torrent and Usenet blackhole. Later: Aria2, Flood, Synology Download Station, RQBit, Tribler, Freebox, if demanded. Dropped: uTorrent, Vuze, Hadouken, NZBVortex, Pneumatic (`sonarr.md` §10). qBittorrent, Deluge and rTorrent adapters build on `autobrr/go-qbittorrent`, `go-deluge` and `go-rtorrent` (`comparison.md` §6).

**Blackhole** writes the `.torrent`/`.nzb` into a watch folder. `ClientRef` is the file name. Completion comes from the completed folder, with the size-stability wait of `ScanWatchFolder.cs`. Caps: no progress, no labels, no removal of seeding items. Only `max_age` and import-related policies apply.

**DDL (phase 3, D11)** is an in-process client behind the same interface (§3.9.4). It is DDL-ready now because `AddRequest` takes a `Payload` that is one of `TorrentFile | Magnet | NZB | LinkGroups`.

**Labels.** Each client has one base category (default `bonnyread`) and an optional per-media-kind suffix (`bonnyread-tv`), used for client-side save paths and to scope `List`. Identity never depends on labels (I1). Lifecycle labels (`bonnyread:seeding`, `bonnyread:failed`, `bonnyread:unlinked`) are **optional, cosmetic mirrors** of DB state for users who look at their client. They are applied as tags where the client supports multiple labels, and never by moving categories (which can move files under qBittorrent's automatic torrent management).

**Client selection** (O15): protocol → tag match (clients with tags need a matching tag; untagged clients are the fallback, as in `DownloadClientProvider.cs:178-194`) → per-indexer override → priority groups → round-robin within a group. The round-robin cursor is stored in the DB, not a process cache. The submit job tries the ordered list and falls back on `Unavailable`/`Rejected` (Whisparr v3's behaviour). If every client fails, the release goes back to acquisition as `Pending(DownloadClientUnavailable)`.

**Remote path mappings** are per client: `(client_id, remote_prefix) → local_prefix`, longest prefix wins. They are applied once when an observation is ingested, and inversely when an explicit save path is sent. In HA (D14) every node sees the same local paths, so mappings do not vary per node. A health check verifies that each client's mapped download root exists and is readable, replacing Sonarr's "not a valid local path" warning (`CompletedDownloadService.cs:317-322`) with a proactive check.

**Polling vs push.** There is one poll loop per client (§3.13). The interval adapts:
- **fast** (default 5 s) while the client has non-terminal downloads in `Submitting/Downloading/Extracting`, or a UI is subscribed to the queue;
- **slow** (default 60 s) otherwise;
- **seeding-only** clients: every 5 min.

A grab, a user action, or a push hint triggers an immediate poll (with a 2 s debounce). qBittorrent uses `sync/maindata` incremental updates inside the adapter.

Push hints: `POST /api/v1/download-clients/{id}/notify` (a scoped API key; the body is ignored beyond an optional client ref). This is the native endpoint that post-processing scripts call. D7 rules out `DownloadedEpisodesScan`.

### 3.4 Data model (ent-style sketch)

```go
// DownloadClient ─────────────────────────────────────────────────────────
field.String("name").Unique()
field.Enum("kind")        // qbittorrent|transmission|deluge|rtorrent|sabnzbd|nzbget|blackhole_torrent|blackhole_usenet|ddl|…
field.Enum("protocol")    // torrent|usenet|ddl
field.Bool("enabled"); field.Int("priority").Default(1)
field.JSON("settings", ClientSettings{})        // host, creds (secret fields, D16), base category, per-kind categories
field.JSON("labels", LabelSettings{})           // mirror lifecycle labels? which ones
field.Enum("extraction").Values("auto","always","never").Default("auto") // auto = torrent/ddl on, Unpacks clients off
field.Int("poll_fast_seconds").Optional(); field.Int("poll_slow_seconds").Optional()
field.JSON("round_robin", RRState{})
edge.To("path_mappings", RemotePathMapping.Type); edge.To("tags", Tag.Type)

// RemotePathMapping: client_id, remote_prefix, local_prefix

// Download (the tracked download) ────────────────────────────────────────
field.UUID("id")
field.Int("client_id"); field.String("client_ref").Optional()   // hash (upper hex) | nzo_id | nzbget id | file | ddl job
field.String("marker")                                           // what we attached at Add (hash, nzbname, PP param)
field.Enum("protocol"); field.Enum("kind").Values("library","racing") // racing = D16, later: no targets
field.Int("slot_id").Optional()                                  // D4; null only for racing
// release snapshot (immutable after grab)
field.Int("indexer_id"); field.String("guid"); field.String("title"); field.Int64("size")
field.String("info_hash").Optional(); field.Bool("private")
field.JSON("indexer_flags", Flags{}); field.Enum("source") // rss|search|interactive|irc|backlog|redownload|user_push
field.JSON("seed_obligation", SeedGoal{})    // min ratio / min seconds from indexer (pack vs single), captured at grab
field.String("grabbed_by").Optional(); field.Time("grabbed_at")
// phase axis
field.Enum("phase")   // submitting|downloading|extracting|importing|import_blocked|imported|failed|abandoned
field.Time("phase_at"); field.Int("version")   // optimistic concurrency for CAS transitions
field.Enum("reason").Optional()               // typed: see §3.9.2
field.String("reason_detail").Optional()
// retention axis
field.Enum("retention") // none|active|seeding|stopped|removed|vanished
field.Time("retention_at"); field.Bool("data_deleted").Default(false)
field.Int("library_refs").Default(0); field.Int("hardlink_refs").Default(0)
field.Time("unlinked_at").Optional().Nillable()
// latest observation
field.JSON("obs", Observation{}); field.Time("observed_at").Optional()
field.String("output_path").Optional()        // local path
// policy clocks (seconds accumulated while observable; §3.6)
field.Int("no_progress_s"); field.Int64("progress_mark_bytes")
field.Int("metadata_s"); field.Int("slow_s"); field.Int("slow_eta_s")
field.Int("unregistered_s"); field.Int("no_seeders_s")
field.Int("absent_polls")                     // consecutive complete snapshots without the item
field.Int("inactive_s")                       // seeding inactivity (if LastActivity unsupported)
// extraction + import
field.Enum("extract_state").Optional()        // pending|running|done|failed
field.String("staging_path").Optional(); field.JSON("staged_files", []string{})
field.Int("extract_attempts"); field.Int("import_attempts")
field.Time("next_attempt_at").Optional().Nillable()
field.Time("blocked_since").Optional().Nillable(); field.Bool("attention_notified")
edge.To("targets", DownloadTarget.Type); edge.To("files", DownloadFile.Type)
edge.To("links", DownloadLink.Type); edge.To("events", DownloadEvent.Type)
index: (client_id, client_ref) unique; (info_hash); (phase); (retention); (slot_id, phase)

// DownloadTarget: download_id, item_id, slot_id, outcome(pending|imported|rejected|skipped), reason, media_file_id?
// DownloadFile: download_id, idx, path, size, verdict(allow|skip|deny), class(media|extra|archive|sample|executable|dangerous|other)
// DownloadLink: download_id, media_file_id, source_rel_path, kind(hardlink|copy|move|reflink), created_at
// DownloadEvent: download_id, at, type, reason, detail, actor(system|user id), data JSON   — the download's timeline

// Blocklist ───────────────────────────────────────────────────────────────
field.String("info_hash").Optional(); field.Int("indexer_id").Optional(); field.String("guid").Optional()
field.String("title"); field.Int64("size").Optional(); field.Enum("protocol")
field.Enum("reason"); field.String("detail").Optional(); field.UUID("download_id").Optional()
field.JSON("scope", BlockScope{})   // items+slot it was grabbed for (for UI); matching is global by identity
field.Time("created_at"); field.Time("expires_at").Optional().Nillable()   // null = permanent
index: (info_hash) ; (indexer_id, guid) ; (title, size)

// DownloadPolicy (time-based queue policy) ────────────────────────────────
field.String("name"); field.Bool("enabled"); field.Int("priority")
field.Enum("condition") // no_progress|metadata|slow_speed|slow_eta|unregistered|no_seeders|max_age|import_blocked|extract_failed
field.JSON("scope", PolicyScope{})  // protocols, privacy(public|private|any), indexer_ids, client_ids, tags,
                                    // media_kinds, completion_min/max %, max_size, min_size
field.Int("after_seconds")          // how long the condition must hold
field.JSON("params", CondParams{})  // min_progress_bytes, min_speed, max_eta, ignore_when_alt_speed
field.Enum("action")                // fail|notify|stop|relabel
field.JSON("action_params", ActionParams{}) // blocklist, search, client_disposition(remove|remove_data|keep|stop), label

// ImportReasonPolicy ─────────────────────────────────────────────────────
field.Enum("reason_class")  // permanent|content_blocked|overridable|transient|needs_user|not_wanted|partial
field.Enum("reason").Optional()      // override for a single reason
field.Enum("action")        // fail|accept|retry|wait_user|complete_without_import
field.Int("max_attempts").Optional(); field.Int("fail_after_seconds").Optional()

// FileContentPolicy ──────────────────────────────────────────────────────
field.Enum("media_kind"); field.Enum("mode").Values("denylist","allowlist")
field.JSON("rules", []GlobOrRegex{})        // "*sample*", "*.exe", "regex:…"  (Cleanuparr list format importable)
field.String("source_url").Optional(); field.Int("refresh_seconds")
field.Enum("on_blocked").Values("skip_files","fail_if_all","fail_if_any")
field.Bool("push_to_client").Default(false) // qBit excluded_file_names

// SeedingRule ────────────────────────────────────────────────────────────
field.String("name"); field.Int("priority"); field.Bool("enabled")
field.JSON("scope", SeedScope{})    // indexer_ids, tracker_domains, privacy, media_kinds, client_ids, tags, phases(imported|failed|racing)
field.Float("max_ratio").Optional(); field.Int("min_seed_seconds").Optional(); field.Int("max_seed_seconds").Optional()
field.Int("max_inactive_seconds").Optional(); field.Int("min_seeders").Optional()
field.Enum("when_unlinked").Values("ignore","remove_after_obligation","remove_now")
field.Enum("unlinked_mode").Values("provenance","hardlink")
field.Int("unregistered_after_seconds").Default(3600)
field.Enum("action").Values("stop","remove","remove_data")

// Backlog ────────────────────────────────────────────────────────────────
// BacklogRule: name, enabled, filter (DSL JSON), selector, batch, interval_s, min_cycle_s,
//              post_release_grace_s, active_download_cap, lanes weighting
// SearchIntent: id, item_id, slot_id, lane(user|redownload|added|backlog), reason, not_before,
//               dedupe_key unique, created_by, state(queued|dispatched|done|dropped), outcome
// SearchLedger: (item_id, slot_id) unique, last_searched_at, cycle_id, last_outcome, failures_window JSON
```

Notes:
- `seed_obligation` is **captured at grab** from the indexer's seed settings (single vs pack, `SeedCriteriaSettings.cs`, Prowlarr's `IndexerTorrentBaseSettings.cs`). This removes `DownloadSeedConfigProvider`'s lookup by hash.
- Custom-format scores per media file are stored by import/release evaluation when computed. The backlog reads them, and no syncer exists.

### 3.5 State machine

```
                            acquisition: Grab(release, slot, targets)
                                             │  tx: insert Download(phase=submitting) + job submit
                                             ▼
                                   ┌──────────────────┐   all clients fail  ─► back to acquisition as
                                   │    SUBMITTING    │                         Pending(ClientUnavailable); row → abandoned
                                   └───┬─────────┬────┘   content policy / release unavailable at fetch
                                       │         └──────────────► FAILED(grab_*)  [blocklist, next candidate]
                            client ack │ client_ref set, retention=active
                                       ▼
   user removed in client /   ┌──────────────────┐  policy: no_progress | metadata | slow | unregistered |
   absent N complete polls ◄──│   DOWNLOADING    │──  max_age | all_files_skipped | client FailReason |
   → ABANDONED(vanished)      └───┬──────────────┘    content policy (post-metadata)     ─► FAILED(reason)
                                  │ observed Completed (settled: not Checking; blackhole size-stable)
                                  ▼
                        needs extraction? (§3.8)
                         │yes                 │no
                         ▼                    │
                 ┌──────────────┐  permanent  │
                 │  EXTRACTING  │──────────►  FAILED(extract_*) or IMPORT_BLOCKED(password_required, disk_full)
                 └──────┬───────┘             │
                        │ staged_files        │
                        ▼                     ▼
                 ┌──────────────────────────────────┐
                 │            IMPORTING             │  job → import.Import(ImportRequest)
                 └───┬───────────┬──────────────┬───┘
       all targets   │           │ some/all     │ transient (locked, space, dest offline)
       resolved      │           │ rejected     └─► IMPORTING again at next_attempt_at (backoff)
                     ▼           ▼
                IMPORTED   reason policy (§3.9.2):
                              accept  ─► IMPORTING (with override)
                              fail    ─► FAILED(import_*)
                              wait    ─► IMPORT_BLOCKED ──(user fixes / manual import / T elapsed → fail)
                              not_wanted ─► IMPORTED(no files; reason=not_an_upgrade)

   Terminal phases: IMPORTED, FAILED, ABANDONED. User actions from any non-terminal phase:
   fail(blocklist?, search?) · remove(delete data?) · ignore (→ ABANDONED(user_ignored), payload untouched).

   Retention axis (independent):
     none ─► active ─(complete, torrent)─► seeding ─► stopped ─► removed
                  └─(usenet/ddl after import or fail)─────────────► removed | none
     any ─(absent from complete snapshots)─► vanished
```

Transition rules:
- **Single writer per phase band.** The client's poll owner writes `submitting→downloading` (on ack reconciliation), `downloading→{extracting|importing|failed|abandoned}` and all observation and clock fields. Extraction and import jobs write their own transitions. Every transition is a CAS on `(id, version, phase)`, and the follow-up job is enqueued **in the same transaction** (the DB job queue from D14 acts as a transactional outbox).
- **Submitting is idempotent.**
  - Torrents: the info hash is computed before `Add` (from the `.torrent` or magnet) and stored as `marker`. A crash after `Add` is reconciled by the next poll, which finds the hash.
  - SABnzbd: `nzbname` carries the download UUID.
  - NZBGet: a PP parameter `*bonnyread=<uuid>`.
  - Blackhole: the file name carries the UUID.
  - A submit that finds its marker already in the client just records the ref.
- **Completion settles.**
  - qBittorrent `checkingUP` counts as not complete (as in `QBittorrent.cs:300-304`).
  - Usenet completion means post-processing is finished (SAB history `Completed`).
  - Blackhole completion needs a stable size for `settle` seconds.
- **Foreign items** (in our category without our marker, e.g. a user added a torrent by hand) are listed as `unmanaged` in the queue view and ignored by policies. The user can "adopt" one, which creates a `Download` with targets chosen through the importer's manual matching. This keeps Sonarr's "in category but not grabbed" import path (`CompletedDownloadService.cs:84-88`) behind an explicit choice.
- **Vanished.** An active item missing from `absent_polls ≥ 3` *complete* snapshots, spanning at least 3 min, becomes `abandoned(vanished)` if it was downloading, or `retention=vanished` if it was imported or seeding. No blocklist and no automatic search follow, because the user removed it. This resolves Sonarr's offline-client TODO (`DownloadMonitoringService.cs:100`): an offline client yields no complete snapshot, so nothing vanishes.

### 3.6 Policy model and actions

**Clocks.** On each poll with a complete snapshot, for each download:

```
Δ = min(now − observed_at, 2 × poll_interval)          // outage gaps never count (I9)
if global.degraded or client.degraded: Δ = 0            // health subsystem flag
active := obs.State ∈ {Downloading, Metadata, Warning(stalled)}   // user-paused or client-queued: clocks frozen
if active:
    if obs.Downloaded − progress_mark_bytes ≥ min_progress:  no_progress_s = 0; progress_mark_bytes = obs.Downloaded
    else:                                                    no_progress_s += Δ
    metadata_s   = obs.State==Metadata ? metadata_s + Δ : 0
    slow_s       = (avg speed over window < min_speed and !(ignore_alt && obs.AltSpeedActive)) ? slow_s + Δ : 0
    slow_eta_s   = (obs.ETA > max_eta ...) ? slow_eta_s + Δ : 0
unregistered_s = obs.Tracker==Unregistered ? unregistered_s + Δ : 0
no_seeders_s   = (SwarmCounts and obs.Seeders==0) ? no_seeders_s + Δ : 0
```

`min_progress` is the smallest `min_progress_bytes` among policies that apply to the download. "Stalled" is defined **by our observed bytes**, not by client state strings, so it behaves the same on every client. The client state is used only to know whether the clock should run. This removes the per-client stall heuristics in Cleanuparr's `*ItemWrapper.cs`.

**Evaluation.**
1. Collect the enabled policies whose scope matches: protocol, privacy, indexer, client, tags, media kind, completion band `(min, max]`, size.
2. Keep, **per condition, the one with the highest priority.** This replaces Cleanuparr's "exactly one rule must match, otherwise skip with a warning".
3. A policy fires when its clock reaches `after_seconds`.
4. Immediate conditions (`all_files_skipped`, a client `FailReason`, `Encrypted`) fire at once, without a policy.

**Actions:**

| Action | Effect |
|---|---|
| `fail` | `failure.Handler.Fail(download, reason, {blocklist, search, client_disposition})` (§3.9.3) |
| `notify` | Emits `DownloadAttention{condition}` once per condition episode (it resets when the clock resets) |
| `stop` | Stops the item in the client; the download stays in its phase, flagged `stopped_by_policy` |
| `relabel` | Adds a label (a cosmetic mirror only) |

`client_disposition` for `fail`:
- **Public** default: `remove_data`.
- **Private** default: `keep`. The item stays in the client and moves to the seeding manager with `phase=failed`, where obligations and rules decide what to do with it (O14, I6). The user can choose `stop` or `remove_data`.

**Default policy set** (seeded; values to tune in beta):

| Condition | Scope | Threshold | Action |
|---|---|---|---|
| metadata | torrent, any | 1 h | fail (blocklist, search) |
| no_progress | torrent, public | 6 h, min progress 50 MB | fail (blocklist, search) |
| no_progress | torrent, private | 24 h | fail (blocklist, search; keep in client) |
| unregistered (while downloading) | torrent | 1 h | fail (blocklist, search) |
| slow_speed / slow_eta | — | disabled by default | — |
| max_age | blackhole | 7 d | notify |
| import_blocked (needs_user) | any | — | notify at once; no auto-fail by default |

**Simulate mode** (O18): per policy, and globally. Actions are computed and recorded as `DownloadEvent{type=would_*}`, and no side effects happen. Because evaluation is pure, this needs no interceptor.

**Health gate.** The runtime health subsystem exposes `network_degraded` (connectivity probes) and per-client `degraded` (failed or incomplete snapshots). Both set Δ = 0, which replaces Cleanuparr's connectivity gate (`CUI/Services/ConnectivityChecker.cs`) and its reason for existing (mass strikes during an outage).

### 3.7 File-content policy (three checkpoints)

One `FileContentPolicy` per media kind (optionally narrowed per slot later), in the glob + `regex:` format so that Cleanuparr's published lists can be imported. File **classes** come from the import layer's extension registry: media, extra (subtitles, nfo, cover), archive, sample, executable, dangerous. Rules act on names and classes.

1. **At grab** (Submitting), when the file list is known: a `.torrent` file, Newznab `files` attributes, or the NZB's file subjects (`NzbValidationService`).
   - Every file of the media class blocked → `FAILED(grab_content_blocked)`, blocklist, and a typed error returned to acquisition, which tries the next candidate from the same result set (the Arrs' behaviour on `ReleaseDownloadException`).
   - Otherwise, blocked file indexes are recorded in `DownloadFile.verdict=skip`.
2. **After metadata** (magnets; also immediately after `Add` for `.torrent` files, since qBittorrent cannot set priorities at add time):
   - With `FileSelection`: skip the blocked files.
   - Without it: apply `on_blocked` (`fail_if_any` or accept).
   - A download that completes with zero wanted bytes (qBittorrent `excluded_file_names` skipped everything) → `FAILED(all_files_skipped)` (Cleanuparr `AllFilesSkippedByQBit`).
3. **At import:** the importer rejects files by class (dangerous, executable, user-denied extension) with typed reasons. These map to the `content_blocked` class in §3.9.2.

**Archives are not blocked content.** Cleanuparr's default list blocks `*.7z`, `*.001` and similar (`cleanuparr.md` §3.2), which fights extraction. Here, the archive class is allowed when extraction is enabled for the download. For comics, `cbz`/`cbr`/`cb7` are *media* containers, never extracted or blocked (Kapowarr treats them as media, `kapowarr.md` §3.5).

`push_to_client` optionally pushes the deny globs into qBittorrent `excluded_file_names`, which saves bandwidth. It is idempotent by content hash, stored on the client row (replacing `BlacklistSyncHistory`).

### 3.8 Extraction stage

Replaces Unpackerr and is built on `golift.io/xtractr` (MIT; pin a tagged release).

**Trigger.** On the `downloading → completed` transition, the poll owner decides whether extraction is needed:
- the client's extraction mode is `always`, or `auto` and the client does not unpack itself (SABnzbd and NZBGet unpack; if the user disabled unpacking there, archives are found anyway);
- **and** `xtractr.FindCompressedFiles(output_path)` finds archives that are not media containers for this media kind.

If it is needed, the owner does a CAS to `extracting` and enqueues an `extract` job; otherwise a CAS to `importing` and an `import` job.

**Where output goes.** Output goes to a **bonnyread staging area**, not back into the client's folder: `<staging_root>/<download_uuid>/`, with `staging_root` on the same filesystem as the library root folders (default `<first root>/.bonnyread-staging`; configurable per root). Consequences:
- The torrent folder is never modified. Seeding and client rechecks are unaffected, and orphan scans need no special case.
- The importer receives staged files marked `payload=owned`, so it **moves** them (an atomic rename on the same filesystem). Nothing is left to delete after import: Unpackerr's `delete_delay`, `IMPORTED` inference and remnant handling disappear.
- Idempotency: every attempt starts by removing `<staging>/<uuid>`, so a crashed half-extraction cannot leak (replacing `Unp/remnants.go`).
- Atomicity (I7): extract into `<uuid>.partial/`, then rename to `<uuid>/` on success.

**Job** (runs on any node's extraction worker pool, `parallel` default 1 per node; heavy I/O is kept off the scheduler):
1. **Free-space pre-check**, which Unpackerr lacks. The uncompressed size comes from RAR, 7z and zip headers; otherwise use archive size × 1.1. It needs `need + reserve ≤ free(staging_root)`. If not → `IMPORT_BLOCKED(disk_full)`, retried when space frees (health-triggered) or by the user.
2. `xtractr.Extract` with:
   - passwords: per-indexer password, then the global list, then `{{password}}` in the release name (`Unp/handlers.go:245`);
   - limits: `MaxBytes` per media kind (Unpackerr defaults: TV 20 GB, movie 75 GB, music 4 GB, book 1 GB), `MaxFiles` 1000, `MaxRatio` 5, `MaxNested` 8, `ExtrasMaxDepth` 3;
   - progress callbacks → `obs.extract_progress` (for SSE).
3. Result → `staged_files` → CAS `extracting → importing` + `import` job.
4. Errors are typed:

| Reason | Class | Default |
|---|---|---|
| `extract_io` (transient read/write error) | transient | retry ×3 with backoff (5 min, 20 min, 1 h), then fail |
| `extract_corrupt` (CRC, truncated volume, missing volume) | permanent | fail + blocklist + search |
| `extract_password_required` | needs_user | `IMPORT_BLOCKED`; user supplies a password → retry |
| `extract_limit_exceeded` (`xtractr.IsLimitError`) | permanent | fail + blocklist (zip bomb or wrong content) |
| `extract_unsupported` | needs_user | `IMPORT_BLOCKED` |
| `disk_full` | transient | `IMPORT_BLOCKED` until space |

- **Music (phase 2):** CUE+FLAC split is an xtractr feature. It becomes an extraction sub-step feeding the music importer directly, replacing Unpackerr's Lidarr `ManualImport` round trip (`Unp/lidarr.go:163`).
- **Dropped:** Unpackerr's standalone watch-folder mode is unrelated to the library (open question Q6).

### 3.9 Import handoff, typed reasons, failure, blocklist and redownload

#### 3.9.1 Handoff contract

```go
type ImportRequest struct {
    DownloadID uuid.UUID
    Slot       SlotID
    Targets    []ItemID                // the grabbed items (authoritative: no title re-resolution)
    Release    ReleaseSnapshot         // title, indexer, flags, quality/CF parsed at grab
    Sources    []SourceFile            // paths + class; from output_path or staged_files
    Payload    PayloadOwnership        // client_seeding (hardlink/copy only) | client_done (move allowed) | owned (move)
    Overrides  []ImportRejectionReason // reasons the reason-policy has accepted (e.g. sample_indeterminate)
}
type ImportOutcome struct {
    PerTarget []TargetResult           // imported(media_file_id, link kind) | rejected(reason, detail) | skipped
    PerFile   []FileResult             // rejected files with typed reasons
}
```

Payload rules:
- A torrent still in the client is `client_seeding`.
- A Usenet or blackhole item that is complete and whose client item will be removed after import is `client_done`.
- DDL output and extraction output are `owned`.

The importer resolves items from `Targets`, not from the title. The Sonarr behaviour of blocking grabs "matched by ID" (`CompletedDownloadService.cs:115-122`) exists because the title-based re-resolution could disagree with the grab. With authoritative targets, that disagreement is an **overridable** reason, not a blocker.

#### 3.9.2 Reason classes (owned jointly with the import doc; the class set is what this doc needs)

| Class | Example reasons (from `Sonarr/MediaFiles/EpisodeImport/ImportRejectionReason.cs`) | Default action |
|---|---|---|
| `content_blocked` | `DangerousFile`, `ExecutableFile`, `UserRejectedExtension`, `NoMediaFiles`, `Sample` (only samples present) | fail + blocklist + search |
| `permanent` | `UnableToParse` *of the content*, `NoAudio`, `EpisodeNotFoundInRelease` (wrong content), `extract_corrupt` | fail + blocklist + search |
| `overridable` | `SampleIndeterminate`, `MatchedById`/`TitleMismatch`, `UnverifiedSceneMapping` | accept (auto-import with override) — replaces Cleanuparr force-import |
| `transient` | `FileLocked`, `MinimumFreeSpace`, root folder offline, `Unpacking` (external unpacker still running) | retry with backoff (1, 5, 15, 60 min …, cap 6 h); after 24 h → `IMPORT_BLOCKED` + notify |
| `needs_user` | `UnknownItem`, ambiguous match, `MultiSeason`, `SplitEpisode`, `extract_password_required` | `IMPORT_BLOCKED` + notify once; optional `fail_after_seconds` |
| `not_wanted` | `NotQualityUpgrade`, `NotCustomFormatUpgrade`, `EpisodeAlreadyImported`, `ExistingFileHasMoreEpisodes` | complete **without import** (`IMPORTED`, reason `not_an_upgrade`); **no blocklist** (the release is fine, just not better); payload goes to seeding as unlinked |
| `partial` | a pack where some targets imported and others were rejected for target-specific reasons | imported targets stay; the download is `IMPORT_BLOCKED(partial)` + notify (Sonarr's default); policy option `accept_partial_after` → `IMPORTED` and the missing targets return to wanted |

Every user-visible "why" is the reason enum, rendered through localisation. This replaces the status-message strings, so nothing downstream matches text.

#### 3.9.3 Failure handler

`Fail(d, reason, opts)` in **one transaction**:
1. CAS `phase → failed`, set `reason`, and write a `DownloadEvent`.
2. If `opts.blocklist`: insert a `Blocklist` row keyed by `info_hash`, or by `(indexer_id, guid)`, plus `(title, size)` (I4). Temporary reasons such as a host quota never blocklist.
3. Record the client disposition:
   - `remove` / `remove_data` → a `client.remove` job;
   - `keep` / `stop` → retention stays and the seeding manager takes over;
   - private items default to `keep` (§3.6).
4. If `opts.search` and redownload is allowed for this download's source (`interactive` is off by default, as `AutoRedownloadFailedFromInteractiveSearch` is): upsert a `SearchIntent{lane=redownload, targets, dedupe_key=(item, slot, lane)}`.
5. Emit `DownloadFailed` (notifications, metrics).

**Redownload rate limit and recurrence guard** (replaces `skipRedownload` + Cleanuparr's SearchQueue + recurring hashes):
- The redownload lane is served by the backlog scheduler (§3.12) at up to **R per minute** globally (default 2), subject to the indexer budget.
- **Recurrence guard per (item, slot):** count `failed` downloads for that target in a sliding window (default ≥ 3 in 24 h). Above the limit, the intent is `dropped(recurrence)`, a `NeedsAttention{recurring_failures}` alert is raised, and the item is excluded from automatic search until the user acknowledges or 7 days pass. The count is a query over `Download` rows, with no separate table.
- The **same release** cannot come back, because the blocklist is checked at decision time (I4). Cleanuparr's "recurring hash" case (the Arr re-grabbing what Cleanuparr just removed) cannot happen with one owner.

**User actions** (O17):
- `fail(blocklist=true|false, search=true|false)`;
- `remove(delete_data)` (no blocklist, no search);
- `ignore` (`abandoned(user_ignored)`; the payload is left alone and untracked by policies; the seeding manager still owns it if it is a torrent, unless the user also detaches it).

#### 3.9.4 DDL downloader (phase 3; interface now)

- A `ddl` client runs in-process. `Add` takes `LinkGroups` from the release. The resolve step belongs to the acquisition doc and D11; site and host logic may be WASM plug-ins under D15.
- Each DDL download is a **durable job with a lease and heartbeat**. It streams to `<staging_root>/<uuid>.partial` with HTTP `Range` resume, so if a node dies another node resumes after the lease expires.
- It reports observations directly (no poll), through the same `Observation` struct.
- Failure types:
  - `link_dead` → try the next link in the group, then the next group;
  - all groups dead → `FAILED(ddl_no_working_links)`, blocklist the article (guid);
  - `host_quota` → **not a failure**: every queued item for that host gets `next_attempt_at = quota reset` (Kapowarr's `_remove_all_of_service`, turned into a pause rather than a removal);
  - `cf_challenge` → `IMPORT_BLOCKED`-style attention if no solver is configured.
- Completion → `payload=owned` → extraction (for zip/rar of loose images) or a direct import.

### 3.10 Seeding manager

The seeding manager **owns every torrent after it leaves `downloading`** (I3): imported, failed-but-kept, not_wanted, and later racing (D16). There is no other remover. The Arr's paused-at-ratio condition and Cleanuparr's "not in any queue" exclusion both disappear.

**Library links** (O10), driven by events instead of inode scans:
- The importer emits `LibraryFileLinked{media_file_id, download_id, source_rel_path, kind}` → insert a `DownloadLink`, increment `library_refs` (and `hardlink_refs` when `kind=hardlink`).
- The library emits `LibraryFileRemoved{media_file_id, cause: upgrade|delete|move_out|rescan_missing}` → delete the links, decrement the counts. When a count reaches 0, set `unlinked_at = now` and enqueue an immediate seeding evaluation for that download.
- `unlinked_mode=provenance` (default): "no library file came from this payload any more".
- `unlinked_mode=hardlink`: "no library file shares storage with this payload". That is Cleanuparr's semantics, which treats copy-imports as unlinked at once; it is useful for freeing disk.
- Cross-seed: before **deleting data**, the manager lists **all** torrents in that client (`ListScope=All`, cached for the tick) and refuses data deletion if any other torrent's content path overlaps the payload's paths. It removes the torrent only (I5). This replaces Cleanuparr's `IgnoredRootDirs` inode map.
- A file in the recycle bin still counts as a hard link on disk, but not in provenance mode. With `hardlink` mode, the recycle bin's purge emits `LibraryFileRemoved(cause=delete)` in turn.

**Evaluation** (leader tick every 5 min, plus event-triggered for a single download):

```
rule      := first enabled SeedingRule by priority whose scope matches (indexer, tracker domain, privacy,
             media kind, client, tags, phase)
oblig     := d.seed_obligation (from indexer at grab; zero for public unless set)
met_oblig := ratio ≥ oblig.min_ratio OR seeding_time ≥ oblig.min_seconds     // tracker H&R semantics (either)
            // unregistered for ≥ rule.unregistered_after ⇒ obligation void
goal      := (rule.max_ratio && ratio ≥ max_ratio && seeding_time ≥ rule.min_seed_seconds)
             OR (rule.max_seed_seconds && seeding_time ≥ max_seed_seconds)
             OR (rule.max_inactive_seconds && inactive ≥ max_inactive_seconds)
             OR (unlinked(rule.unlinked_mode) && rule.when_unlinked == remove_now)
             OR (unlinked && rule.when_unlinked == remove_after_obligation)
guard     := rule.min_seeders && swarm_seeders < min_seeders            // don't abandon a thin swarm
act       := met_oblig && goal && !guard && !(d.phase ∈ {downloading, extracting, importing})
if act: apply rule.action — stop | remove | remove_data(only if hardlink_refs==0 && phase not pre-import
                                                        && no cross-seed overlap)
```

The storage check for data deletion (I5) uses **storage dependency**, not provenance:
- `hardlink`/`reflink` links block deletion;
- `copy`/`move` links do not, because the library file has its own storage;
- a payload not yet imported (`client_seeding` in `importing`/`import_blocked`) also blocks it.

Provenance (`library_refs`) only drives the "unlinked" *goal*.

- Obligations are OR'd ("ratio or time"). This matches common hit-and-run rules and Sonarr's validation messages. A per-indexer "AND" mode is an open question (Q4).
- `remove_data` is automatically demoted to `remove` if I5 would be violated, and the event records why.
- **Dead and unregistered.**
  - While seeding: `unregistered_s ≥ rule.unregistered_after` (default 1 h, which covers new-upload registration lag) → obligations are void. The action is `remove` if linked, and `remove_data` if unlinked and the rule allows it.
  - `no_seeders_s` only matters while downloading (the no-progress policy covers it). While seeding, zero *other* seeders is normal.
  - At add time on private trackers, "unregistered" in the first minutes triggers the reannounce loop (`Reannouncer`, every 7 s up to 50 attempts, from autobrr `run_transmission.go:22,148-160` and go-qbittorrent's `ReannounceTorrentWithRetry`). The unregistered clock starts only after that loop ends.
- **Pushing seed limits to the client** (`SeedLimitSetter`) is optional, off by default. When on, it pushes `max(obligation, rule)` so the client stops itself even while bonnyread is down. If the client removes an item behind our back, the item shows up as `retention=vanished`, which is harmless.
- **Usenet and blackhole:**
  - Usenet: after import (or failure), remove the item from the client's history per client setting (`RemoveCompletedDownloads`/`RemoveFailedDownloads`, `DownloadEventHub.cs:29-85`), set `retention=removed`. For `client_done`, delete the data only after the import moved or copied the files.
  - Blackhole: the retention axis stays `none`.

### 3.11 Housekeeping

A leader tick, hourly by default, runs:
1. **Orphaned files** (O12). The candidates are **top-level** entries under each client's mapped download roots (plus `staging_root`). An entry is claimed if any of these hold:
   - it is the content path of any item in any client (`ListScope=All`);
   - it is the staging path of a non-terminal download;
   - it is the output path of a download in `importing`/`import_blocked`.

   Unclaimed entries older than `min_age` (24 h) move to a quarantine directory (renamed on collision, copy+delete across devices); entries in quarantine older than `purge_after` are deleted. Safety:
   - skip the whole run if any client's snapshot is not complete (Cleanuparr's collapsed-list check);
   - skip symlinks;
   - apply exclude globs;
   - validation forbids scan roots overlapping library roots.
2. **Staging sweep:** staging directories of downloads in terminal phases, or with no row, are deleted after 1 h.
3. **Retention prune:** `Download` rows in terminal phase with `retention ∈ {removed, none, vanished}` older than the history retention (default 365 d) are deleted. `DownloadEvent` rows are compacted to a summary kept by the history subsystem. Blocklist rows are kept until they expire or the user clears them.
4. **Stuck-state guard:** a `DownloadEvent` alert for any download whose phase has not changed in longer than a ceiling (e.g. `extracting` > 12 h, `submitting` > 30 min). It alerts only; this is the backstop Unpackerr's 24 h sweep provided, turned into a visible alert instead of silent mutation.

### 3.12 Backlog and upgrade scheduler ("when to search")

This unifies Cleanuparr's Seeker, Profilarr's upgrades, the Arrs' bulk missing/cutoff searches, search-on-add and failure redownloads into one durable **SearchIntent** queue with lanes. How a search runs (criteria, season versus episode, indexer selection) belongs to the acquisition doc. This doc decides *what* and *when*.

| Lane | Source | Throttle |
|---|---|---|
| `user` | UI or API "search now" | none beyond the indexer budget |
| `redownload` | failure handler | R/min + recurrence guard (§3.9.3) |
| `added` | item/slot newly monitored (search-on-add, requests) | light (default 10/min) |
| `backlog` | BacklogRules | trickle (below) |

**Backlog rules**, replacing Seeker plus Profilarr upgrades:
- **Candidate set:** `(item, slot)` pairs that are monitored, not in the active queue (no non-terminal download for that item and slot), and past `post_release_grace` (6 h) since release or air date. Each pair must be one of:
  - `missing`;
  - `cutoff_unmet` (quality);
  - `cf_below_cutoff`: the stored score is below `cutoff × profile cutoff score`. For TV the score is the per-episode file score, not a series average (Profilarr's Sonarr average in `upgrades.md` exists only because it read from outside).
- **Filter:** Profilarr's DSL (nested AND/OR over library fields: monitored, genres, tags, year, ratings, dates, status, release group, CF names, size…), with fields generalised per media kind. The field list is owned by the library doc; the evaluator is shared with the UI's preview.
- **Selector:** `balanced_weighted` (default), `oldest_search_first`, `newest_first`, `lowest_score`, `random`, … (union of `CUI/Features/Seeker/Selectors/*` and `profilarr/src/lib/shared/upgrades/selectors.ts`).
- **Cycles instead of tags:** `SearchLedger(item, slot)` stores `cycle_id` and `last_searched_at`. A pair is eligible once per cycle, and a new cycle starts only when the pool is exhausted **and** `min_cycle` (7 d) has passed. This replaces Profilarr's `profilarr-<filter>` tags and Cleanuparr's `SeekerHistory`.
- **Rate:** each rule has `interval` and `batch` (default 10 min, 1 item), jittered. Across rules the lanes are round-robined. Before dispatch the scheduler asks acquisition's `Budget.Available(indexer set, kind=search)`. With no budget it waits (no error, no burn).
- **Active cap:** skip the backlog lane while `count(downloading, excluding downloads whose no_progress_s > 0)` ≥ `active_download_cap` (default 3). This is Cleanuparr's `ActiveDownloadLimit` with `IgnoreStruckDownloads`, computed from our own clocks.
- **Observation:** the dispatched intent carries a `search_id`. Acquisition emits `SearchCompleted{search_id, grabbed: []DownloadID}` in-process, so `SearchIntent.outcome` is set directly. Cleanuparr's `SeekerCommandMonitor` polling disappears.
- **Metrics:** searches per lane, grabs per search, upgrades per cycle (D7).

### 3.13 HA placement (D14)

| Work | Placement | Mechanism |
|---|---|---|
| Client poll loop + observation ingest + policy clocks + `downloading→*` transitions | **Singleton per client** | Lease row `lease:download-client:<id>` (DB lease/advisory lock). A per-client lease is a finer-grained form of D14's leader election: clients can spread across nodes, and a stuck client does not stall the others. Failover = lease expiry (≤ 3 × poll interval). Clocks tolerate it through the Δ cap. |
| Submit (grab) jobs | Any node | DB job queue; idempotent by marker |
| Extraction jobs | Any node with the extraction pool | Job claim; staging on shared storage (D14 requires the same paths on all nodes) |
| Import jobs | Any node | Job claim (import doc) |
| Client actions (remove, stop, label, skip files, reannounce) | Any node | Idempotent jobs; "not found" = success |
| DDL transfers (phase 3) | Any node, one per transfer | Job lease + heartbeat; resume by `Range` |
| Seeding manager tick | **Leader singleton** | Global leader lease; event-triggered single-download evaluations are jobs (any node) that CAS on `version` |
| Housekeeping (orphans, staging, prune) | **Leader singleton** | Global leader lease |
| Backlog scheduler tick | **Leader singleton** | Global leader lease; dispatch = insert intents; searches run as jobs |
| Blocklist refresh / content-policy list refresh / qBit glob push | Leader singleton | Global leader lease |
| Push-hint endpoint | Any API node | Publishes `poll-now:<client>` via LISTEN/NOTIFY to the lease holder |
| Queue API / SSE | Any API node | Reads the DB; change events via LISTEN/NOTIFY |

On SQLite (single node) every lease is trivially held. The design has no in-memory state that matters: caches are only performance aids (for example the last qBittorrent `rid`).

### 3.14 Events and metrics (D7)

Events on the bus:
- lifecycle: `DownloadGrabbed`, `DownloadProgress` (throttled, SSE only), `DownloadStalled`/`DownloadAttention{condition}`, `DownloadFailed{reason}`;
- extraction: `ExtractionStarted/Completed/Failed`;
- import: `ImportBlocked{reason}` (the manual-interaction-required notification, sent once), `DownloadImported`;
- seeding: `SeedingStopped`, `DownloadRemoved{data_deleted}`, `TorrentUnlinked`, `TorrentUnregistered`;
- housekeeping: `OrphanQuarantined`;
- search: `SearchIntentDropped{recurrence}`.

Prometheus metrics:
- downloads by phase and retention;
- policy firings by condition and action;
- time in phase (histograms);
- extraction bytes and duration;
- poll latency and errors per client;
- seeding ratio distribution;
- orphan bytes quarantined;
- searches per lane.

---

## 4. Walkthroughs

### 4.1 A public torrent that stalls at 40%

1. **Grab.** Acquisition calls `Grab(release R, slot "1080p", targets [S02E05])`. In one transaction: `Download{phase=submitting, info_hash=H, private=false, seed_obligation={}}` and a `submit` job. The job fetches the `.torrent`; the content policy passes (one mkv, one nfo). It selects qBittorrent (priority 1, tags match), calls `Add` with category `bonnyread-tv`, then skips the `.nfo` if the policy says so. It does a CAS to `downloading`, `retention=active`, and sends a `poll-now` hint.
2. The **poll owner** for qBittorrent (lease holder, fast interval of 5 s) observes 1% … 40% (4 GB of 10 GB). `no_progress_s` stays 0, because each poll sees growth ≥ 50 MB relative to `progress_mark_bytes`.
3. The swarm dies. Observations: `Downloading`/`stalledDL`, 4.0 GB, rate 0. Each poll adds Δ (5 s) to `no_progress_s`.
4. The user's NAS reboots for 20 minutes. There are no successful polls, and the first poll afterwards adds Δ = min(20 min, 10 s) = 10 s. The client being unreachable marks it `degraded`, so Δ = 0 anyway. The outage does not count (I9).
5. The user pauses the torrent in qBittorrent overnight. The state is `Paused`, so clocks freeze. After resuming, the torrent is still stuck.
6. `no_progress_s` reaches 6 h (the matching policy is "no_progress, torrent, public, 6 h"). `policy.Eval` returns `fail{blocklist, search, disposition=remove_data}`.
7. **Failure handler** (one transaction): CAS `downloading → failed(reason=no_progress)`; `Blocklist{info_hash=H, indexer, guid, title, size, reason=no_progress}`; a `client.remove(H, deleteData=true)` job; `SearchIntent{lane=redownload, item=S02E05, slot=1080p}` (unique by dedupe key); `DownloadFailed` event → notification.
8. The **backlog scheduler** (leader) sees the redownload intent. The recurrence guard finds 1 failure for (S02E05, 1080p) in 24 h, which is under 3. The global redownload rate allows it and the indexer budget is available, so it dispatches. Acquisition searches, and the blocklist rejects R/H. The next-best release R2 is grabbed, and the intent's outcome is `grabbed(D2)`.
9. The queue UI showed the first download with "No progress for 5 h 40 min (policy: fail at 6 h)" throughout. This is available because clocks are data.

*If the torrent had been private:* the policy default is 24 h and the disposition is `keep`. The item stays in the client with `phase=failed`, and the seeding manager owns it. If it later completes, it seeds under the private rule. If it stays incomplete, the user sees it under "failed, kept in client" and can remove it; obligations usually don't apply to incomplete downloads.

### 4.2 A RAR'd torrent through extraction, import and seeding

1. **Grab** as above. The `.torrent` file list has `movie.rar, movie.r00…r45, movie.nfo, Sample/movie-sample.mkv`. Content policy for movies: the archive class is allowed (extraction is `auto` for this torrent client), and `*sample*` is denied, so the sample is skipped after add (`FileSelection`).
2. **Completion:** the observation goes `Seeding`/`uploading` and the torrent is complete. The poll owner calls `FindCompressedFiles(output_path)`, finds `movie.rar` (first volume), does a CAS to `extracting` and enqueues `extract`. Retention becomes `seeding`, so the seeding manager now owns the torrent, but `act` is false while the phase is in progress.
3. **Extract job** (claimed by node B):
   - it deletes `/media/.bonnyread-staging/<uuid>*`;
   - the free-space check (header sum 23.1 GB plus reserve) passes;
   - it extracts into `<uuid>.partial/`, with progress on the queue UI;
   - it renames to `<uuid>/`, sets `staged_files=[movie.mkv]`, does a CAS to `importing` and enqueues `import`.
   The torrent folder is untouched and keeps seeding the RARs.
4. **Import job:** `ImportRequest{targets=[Movie M], slot=4K, Sources=[staged movie.mkv], Payload=owned}`. The importer moves the file (a rename on the same filesystem) into the 4K slot's root folder and returns `imported(media_file_id=F, kind=move)`. It emits `LibraryFileLinked{F, download, kind=move}`, which gives `library_refs=1, hardlink_refs=0`. CAS to `imported`; the staging directory (now empty) is removed.
5. **Seeding:** the tracker is public and the rule is "public movies: max ratio 2.0 or max 14 d; unlinked_mode=provenance; action remove_data". Provenance says the payload is still linked (F came from it), but the storage check looks only at `hardlink_refs`, which is 0: F was moved out of staging and shares no storage with the RARs. After 14 d (ratio 1.3), the torrent is removed together with the RARs, and the event records "data deleted; library file F unaffected (extracted copy)".
6. Unpackerr's "wait for the title to leave the queue, then delete the extracted copy after `delete_delay`" has no counterpart here: nothing was written into the client folder, and the extracted copy became the library file.

### 4.3 A Usenet download that fails import for a typed reason

1. **Grab** to SABnzbd with `nzbname=<uuid>` (idempotency marker) and category `bonnyread-tv`. The client unpacks itself (`Caps.Unpacks`), so extraction is `auto → skip`.
2. SABnzbd downloads, repairs and unpacks, and the history shows `Completed`, so the observation is `Completed`, `output_path` mapped through the client's remote path mapping. `FindCompressedFiles` finds nothing, so CAS to `importing` + `import` job, `Payload=client_done`.
3. The importer inspects `show.s03e07.mkv` and `show.s03e07.exe`, and returns a file rejection `ExecutableFile` for the `.exe`. The mkv turns out to be 2 MB, so the target is `rejected(Sample)` (only a sample of media is present).
4. The **reason policy** classifies `ExecutableFile` and "only sample" as `content_blocked`, whose default is `fail + blocklist + search`. The failure handler:
   - CAS `importing → failed(import_content_blocked: ExecutableFile)`;
   - `Blocklist{indexer, guid, title, size}`;
   - a `client.remove(nzo, deleteData=true)` job (SABnzbd history and files);
   - `SearchIntent{redownload}`;
   - a notification.
   Nothing matches message text.
5. *Variant: `SampleIndeterminate`.* The runtime could not be read, so the importer cannot tell whether the file is a sample. That reason is `overridable`, and the default policy is `accept`. The download goes back to `importing` with `Overrides=[SampleIndeterminate]`; the importer imports and records the override in history. This replaces Cleanuparr's ForceImport round trip (`manualimport` GET → `ManualImport` command → history poll).
6. *Variant: `UnknownItem` or an ambiguous multi-season pack* (`needs_user`) → `IMPORT_BLOCKED`. The `ImportBlocked` notification fires once (`attention_notified`). The queue shows the typed reason with a "Manual import" action, which opens the importer's manual matching on `output_path`. There is no auto-fail unless the user has configured `fail_after_seconds`.
7. *Variant: `MinimumFreeSpace`* (`transient`) → stay in `importing` with `next_attempt_at` backoff; after 24 h → `IMPORT_BLOCKED(transient_exhausted)` + notify.

### 4.4 A private-tracker torrent whose library file is later upgraded (unlinked)

1. **Grab** from private indexer P, whose seed settings are ratio 1.0 or 72 h, recorded as `seed_obligation={min_ratio:1.0, min_seconds:259200}` at grab time (this is a single episode, not a pack, so the pack seed time is not used). `private=true` comes from the indexer definition, OR'd later with the client's `Private` flag. At add the tracker reports "unregistered torrent", so the reannounce loop runs; after 2 attempts it reports OK.
2. It completes, and the importer **hardlinks** it (`Payload=client_seeding`): `LibraryFileLinked{F1, kind=hardlink}` gives `library_refs=1, hardlink_refs=1`. CAS to `imported`; retention is `seeding`.
3. Seeding rule "Tracker P: max_seed 30 d, when_unlinked=remove_after_obligation, unlinked_mode=provenance, action remove_data, min_seeders 2".
4. Day 2: an RSS release scores higher, so acquisition grabs D2 for the same (item, slot). D2 downloads and imports. The importer **replaces** F1 with F2 (F1 goes to the recycle bin) and emits `LibraryFileRemoved{F1, cause=upgrade}`. The handler deletes the link: `library_refs=0, hardlink_refs=0`, `unlinked_at=now`, and enqueues an evaluation job for D1. The optional label `bonnyread:unlinked` is added.
5. **Evaluation at day 2 + 1 min:** ratio 0.4, seeding time 40 h, so `met_oblig` is false (I6) and nothing is done. The queue and seeding view show "Unlinked — held for tracker obligation (32 h left or ratio 1.0)".
6. At 72 h, `met_oblig` becomes true, the unlinked goal is true, and the guard passes (swarm seeders 14 ≥ 2), so the action is `remove_data`. Pre-checks:
   - `hardlink_refs==0` (the only link, to F1, was deleted on upgrade);
   - the cross-seed check (`List(All)`) finds another torrent in the client whose content path is the same `…/Show.S01E01…/` (a cross-seed of D1 added by the user's cross-seed tool). The data is shared, so the action is **demoted to `remove`**.
   The torrent is removed from the client, the data is kept, and the event says "data kept: shared with torrent <hash2> (not managed)". The recycle-bin copy of F1 is purged later by the recycle-bin job; it had its own inode reference, which is no concern here.
7. *Variant:* after the upgrade the tracker deletes the old release as a dupe, so the tracker reports unregistered. After 1 h `unregistered_s` voids the obligation, and removal happens at once (with data, if no cross-seed shares it).
8. *Variant with `unlinked_mode=hardlink`:* identical here, since the import was a hardlink. For a *copy* import, the payload would count as "unlinked" from the start, and could go right after its obligations, which is Cleanuparr's behaviour for freeing space.

---

## 5. Simplifications compared with the references

| Reference mechanism | Where | Replacement | Why it is simpler or more correct |
|---|---|---|---|
| In-memory tracked-download cache rebuilt from history + title parsing | `TrackedDownloadService.cs:46,97-218` | Durable `Download` row created before `Add`, keyed by marker/hash | No re-identification; survives restarts and HA failover (I1, I2) |
| Status-message strings as the "why" | `CompletedDownloadService.cs:162,189-209` | Typed reason enums + localised rendering | Nothing downstream matches text |
| Strikes (a row per run, ≥3, 24 h decay), recurring static set | `Striker.cs` | Per-condition duration clocks on the row | One concept; independent of job frequency; outage-safe |
| Connectivity check to avoid mass strikes | `ConnectivityChecker.cs` | Δ = 0 while degraded | Falls out of the clock rule |
| Per-client "stalled" heuristics | Cleanuparr `*ItemWrapper.cs` | Stall = no byte progress while active | Client-agnostic |
| Failed-import rule on message substrings + force-import round trip + 6 h caches | `ArrClient.cs:147-150,474-495`, `ForceImportService.cs` | Reason classes → policy (`accept`, `retry`, `fail`, `wait`) | In-process; typed; no polling of commands |
| `skipRedownload` + private SearchQueue + `/command` polling; unthrottled `AutoRedownloadFailed` | `RedownloadFailedDownloadService.cs`, `Seeker.cs`, `SeekerCommandMonitor` | One SearchIntent queue with lanes, rate limits, a recurrence guard over `Download` rows | One scheduler; observable outcomes |
| Unpackerr's queue polling, title map, "left queue = imported", delete delay, remnants, 72 h JSONL restore, 24 h sweep | `Unp/handlers.go`, `starrpoll.go`, `remnants.go`, `historyrestore.go` | An `extracting` phase; staging dir on the library filesystem; the importer moves the output | No cleanup step; no inference; idempotent by wiping staging |
| Removal only when the client paused at ratio; Cleanuparr skips anything "in any queue" | `QBittorrent.cs:283-288`, `cleanuparr.md` §3.4 | The seeding manager is the single owner of client payloads | One remover (I3); works on every client |
| Categories as state (`cleanuparr-unlinked`/`-dead` → a seeding rule on that category) | `UnlinkedDownloadsService.cs`, `DeadTorrentService.cs` | Retention axis + `unlinked_at` in the DB; labels only as optional mirrors | No category moves (which can move files) |
| Inode link-count scans and cross-seed inode maps | `*HardLinkFileService.cs` | Library events maintain `DownloadLink`; one path-overlap check before data deletion | Event-driven; covers copy imports correctly |
| Seed config re-derived by hash from history with a 1 h cache | `DownloadSeedConfigProvider.cs:37-88` | `seed_obligation` captured at grab | No lookup |
| Five per-client seeding-rule tables | `cleanuparr.md` §10 | One `SeedingRule` + client `Caps` | Capabilities, not tables |
| Triple polling of each client (1 min, 5 min, 5 s, 2 min) | Sonarr, Cleanuparr, Unpackerr | One adaptive loop per client (5 s active, 60 s idle) + push hints | One set of client calls feeds every policy |
| Malware Blocker every 5 s + blacklist sync job | `MalwareBlocker.cs`, `BlacklistSynchronizer.cs` | Content policy at grab, post-metadata (on the normal poll) and import; optional push to qBittorrent | Event-driven checkpoints |
| Tag-based upgrade cooldowns; Seeker history; CF-score syncer | `profilarr/docs/backend/upgrades.md`, `CustomFormatScoreSyncer.cs` | `SearchLedger` cycles; scores stored at import | No library mirror |
| Sonarr "matched by ID ⇒ blocked" | `CompletedDownloadService.cs:115-122` | Authoritative grab targets; the mismatch is an overridable reason | Removes a common manual step, keeps the safety as policy |
| Container path remapping per tool | Cleanuparr `DownloadDirectorySource/Target`, Unpackerr `paths[]` | One per-client remote path mapping | One configuration |
| Dry-run interceptor wrapping every mutation | `DryRunInterceptor.cs` | Pure evaluation; simulate records `would_*` events | By construction |
| Kapowarr drops queued downloads when a host quota hits | `download_queue.py:652` | `next_attempt_at` = quota reset for that host's jobs | No lost work |

---

## 6. Issues with decisions, and open questions

### 6.1 Issues with decisions

1. **D14 — polling granularity (refinement, not a conflict).** D14 lists "download-client polling" as leader-elected singleton work. This design uses a **lease per download client** rather than one global leader for all polling, so that clients spread across nodes and one hung client cannot delay the others. It is still leader election, at finer grain. Please confirm this reading of D14.
2. **D14 — storage topology needs one more constraint.** D14 requires media, download and data paths on shared storage at the same paths. Extraction output needs a **staging root on the same filesystem as each library root** to allow an atomic move into the library, and hard-link imports already need download and library paths on one filesystem. D14 should state "same paths **and** filesystem co-location for staging/library (and download/library when hardlinking)" as a deployment requirement. Without it, imports silently degrade to copy, which doubles disk use and I/O.
3. **D19 — text matching cannot be eliminated at client boundaries.** SABnzbd/NZBGet failure messages and tracker announce messages (unregistered, down) are only available as text. This design confines that text matching to adapter-level classifiers with per-client tests, and emits enums. I read D19's "no text-matching on status messages" as applying to *internal* state, which this design honours. Flagging so the rule is not applied literally to adapters.
4. **D11 + D15 — plug-in boundary for DDL hosts.** If DDL host plug-ins run in wazero (D15), they should only **resolve** links (URL + headers + cookies + quota info). The core streams the bytes, because streaming gigabytes through WASM host functions would be slow and would bypass the core's lease and resume logic. D15 should say "DDL host plug-ins resolve; the core downloads".
5. **D4 — one download, one slot.** D4 says slots are grabbed independently. This design enforces **one slot per download** (I8), so a release that would satisfy two slots is downloaded twice if both slots want it. That is consistent with D4 but may surprise users (for example a single 2160p remux wanted by both a "4K" and a "best" slot). No change proposed; listed as Q1.

### 6.2 Open questions

- **Q1 Cross-slot reuse.** Should one completed download be allowed to feed a second slot by copy or hardlink, instead of a second grab? That would need `DownloadTarget` to carry a slot per target. Recommendation: not in phase 1.
- **Q2 Superseded active downloads.** When acquisition grabs a better release while a lower one is still downloading for the same (item, slot), should the lower one be cancelled automatically (`abandoned(superseded)`, no blocklist)? Recommendation: yes for public trackers; keep private ones for ratio (disposition `keep`). The acquisition doc needs to agree.
- **Q3 Partial pack default.** Keep Sonarr's default (`IMPORT_BLOCKED(partial)` until the user acts), or default to `accept_partial_after = 24 h` with the missing targets returning to wanted? Recommendation: keep the blocking default in phase 1 and expose the policy.
- **Q4 Obligation semantics per tracker.** Hit-and-run rules differ (ratio OR time, ratio AND time, time-only, "seed X h within Y days"). OR is proposed as the default; a per-indexer `obligation_mode` may be needed. Should indexer definitions (Cardigann `minimumratio`/`minimumseedtime`) populate it automatically?
- **Q5 Default thresholds** (1 h metadata, 6 h/24 h no progress, 1 h unregistered, 3 failures in 24 h, 2 redownloads per minute) need tuning against beta telemetry.
- **Q6 Unpackerr's watch-folder mode** (extracting arbitrary folders not tied to a download) has no library outcome. Proposed: drop. If wanted, it can reuse the extraction worker pool as a small optional feature.
- **Q7 Pushing seed limits to clients.** Off by default (bonnyread decides), or on as a safety net while bonnyread is down? Off is simpler; on protects ratio during long outages.
- **Q8 Racing mode (D16, later).** The model already allows `kind=racing` with no slot or targets, and the seeding manager covers it. The autograb rule and client-side guards (max active downloads, ignore when slow, from `run_qbittorrent.go:206-260`) belong to the acquisition doc when racing arrives.
- **Q9 Foreign items.** Should "adopt an unmanaged client item" be in phase 1 (it replaces Sonarr's import of category items not grabbed by the app, and the old `DownloadedEpisodesScan` workflow ruled out by D7), or can it wait for manual import from a path?
- **Q10 Blocklist expiry.** Should some failure reasons (e.g. `no_progress` on a public torrent, which may revive) blocklist for a limited time (e.g. 30 d) instead of permanently? The schema supports `expires_at`; the default is permanent, as in the Arrs.
