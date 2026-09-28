# 05. Downloads

This file specifies the download lifecycle, from an accepted grab to the hand-off of complete files to import, and what happens to the client payload afterwards. It covers the download-client abstraction and its adapters, the `Download` record and its two state machines (phase and retention), time-based queue policies, file-content checks, extraction, the external processing hold, the reaction to typed import rejections, failure handling, the blocklist and redownload, the seeding manager, orphan housekeeping, and the search queue that decides *when* to search. What to grab is in `04-evaluation.md`; how files are identified, placed and recorded is in `06-import.md`. Phases: **P1** (torrent and Usenet clients, blackhole, policies, extraction, processing hold, seeding, search queue), **P2** (music CUE split, additional adapters), **P3** (DDL in-process downloader, book/comic/adult specifics).

## Concepts

| Term | Definition |
|---|---|
| Download | The durable record of one payload bonnyread sent to one download client. Created before the client is called. Alias: Sonarr's "tracked download". |
| DownloadTarget | One `(Item, SlotDefinition)` pair that a Download is expected to fill. A Download has one or more targets and, per D20, the targets may name **several slots**. Alias: evaluation's grab "target". |
| Download client | A configured external client (qBittorrent, SABnzbd, …) or an in-process client (blackhole, DDL). Represented by an adapter implementing `Client` (DLD-017). |
| Marker | The identifier bonnyread attaches to, or computes for, a client item so that it can match the item back: info hash, SABnzbd `nzbname`, NZBGet PP parameter, blackhole file name, DDL job id. |
| Observation | The normalised latest state of a client item, produced by an adapter from a client listing (a **snapshot**). |
| Phase | What the download means for the library: `submitting`, `downloading`, `extracting`, `processing`, `importing`, `import_blocked`, `imported`, `failed`, `abandoned`. |
| Retention | What is happening to the payload in the client: `none`, `active`, `seeding`, `stopped`, `removed`, `vanished`. Orthogonal to phase. |
| Condition clock | A per-download counter of seconds for which a condition (no progress, metadata, slow, …) has held **while bonnyread could observe it**. |
| Policy | A `DownloadPolicy` row: a condition, a scope, a threshold and an action. |
| Staging root | A bonnyread-owned directory on the same filesystem as the library root folders that receives extraction output, DDL output and processing-hold working copies. |
| Processing hold | The `processing` phase: import waits for an external tool to mark files ready or failed through the API (D20). |
| Reason class | The download-level category of an import rejection code that selects the default action: `content_blocked`, `permanent`, `overridable`, `transient`, `needs_user`, `not_wanted`, `partial`. |
| Seeding manager | The single component that decides when a torrent payload is stopped or removed after it leaves `downloading`. |
| Seed obligation | The tracker's minimum seeding requirement captured at grab time from the indexer's seed policy. |
| Unlinked | A payload from which no current library file came (provenance mode), or with which no library file shares storage (hardlink mode). |
| SearchIntent | A durable request to search for one `(Item, SlotDefinition)` in a lane (`user`, `redownload`, `added`, `backlog`). |
| Lane | A class of SearchIntent with its own throttle. |

Entity names for the library (`Title`, `Item`, `SlotDefinition`, `TitleSlot`, `ItemSlot`, `MediaFile`, `RootFolder`) follow `02-domain-model.md`. "Slot" in this file means a `SlotDefinition` as configured on the item's `TitleSlot`.

## Requirements

### 1. Ownership and principles

**DLD-001** (MUST) The `Download` row is the source of truth for every download bonnyread starts. It **MUST** hold the immutable release snapshot, the targets, the indexer, privacy, seed obligation and the client reference. The client is an observed system: its listings update the latest observation and **MUST NOT** be used to re-derive a download's identity, targets or items. _Source: design/downloads.md §1.2 I1, §3.1._

**DLD-002** (MUST) All download state that affects behaviour (phase, retention, clocks, targets, attempt counters, blocklist, search ledger) **MUST** be persisted in the database and survive restarts and HA failover exactly. In-memory state **MAY** exist only as a performance cache (for example the qBittorrent `rid`). _Source: design/downloads.md I2, §3.13._

**DLD-003** (MUST) Exactly one component decides when a client item bonnyread manages is removed from its client: the failure handler while the download is before `imported`/`failed`, and the seeding manager (DLD-095) afterwards. Other code paths **MUST NOT** remove a managed client item. _Source: design/downloads.md I3._

**DLD-004** (MUST) Every reason carried by a download (failure, block, attention, policy firing) **MUST** be a typed enum value with an optional free-text `detail` for display only. Components outside client adapters **MUST NOT** branch on text supplied by a client, tracker or indexer. _Source: design/downloads.md §3.1.4; D19; A10._

**DLD-005** (MUST) Where a client or tracker reports state only as text (SABnzbd/NZBGet failure messages, tracker announce messages), the adapter **MUST** classify the text into an enum at the adapter boundary, and each such classifier **MUST** have per-client unit tests with recorded messages. The tracker-message classifier **MUST** be shared with the indexer layer (`03-acquisition.md`). _Source: A10; design/downloads.md §3.3._

**DLD-006** (MUST) Policy evaluation **MUST** be a pure function `evaluate(download, observation, policies, now) → []Action` with no I/O, so that it is table-testable and simulate mode (DLD-060) needs no interception layer. _Source: design/downloads.md §3.1.5._

**DLD-007** (MUST) A push notification from a client or script **MUST** only trigger an immediate poll of that client; it **MUST NOT** change download state by itself. _Source: design/downloads.md §3.1.6._

### 2. Grab hand-off and Download creation

**DLD-008** (MUST) When evaluation commits a grab (`04-evaluation.md`, grab reservation), the `Download` row with `phase=submitting`, `retention=none`, one `DownloadTarget` row per `(item, slot)` target, and a `download.submit` job **MUST** be inserted **in the same database transaction** as the grab reservation. The client **MUST NOT** be called inside that transaction. _Source: design/downloads.md §3.5; design/evaluation.md §3.16._

**DLD-009** (MUST) The grab hand-off **MUST** carry: the release snapshot (indexer id, GUID, title, size, protocol, info hash if known, indexer flags, parsed quality and custom-format digest), the payload reference (`TorrentFile | Magnet | NZB | LinkGroups`), the source (`rss | search | interactive | irc | backlog | redownload | user_push`), the target set, the grabbing user (if any), and client-routing hints (tags, per-indexer client override). _Source: design/downloads.md §3.4, §3.9.1; design/evaluation.md §3.3._

**DLD-010** (MUST) A single grab whose release was selected by several slots of the same item(s) **MUST** produce **one** Download with one target per `(item, slot)` pair. bonnyread **MUST NOT** send the same release to a client twice because several slots want it. _Source: D20 (amends D4); open-issues B1(a); design/evaluation.md §3.13._

**DLD-011** (MUST) For torrents, the seed obligation (minimum ratio, minimum seeding seconds, and pack seeding seconds when the release is a pack) **MUST** be captured into `Download.seed_obligation` at grab time from the indexer's seed policy returned with the fetched payload (`03-acquisition.md`). It **MUST NOT** be looked up again later. Seed policy and download-client routing are owned by the download lifecycle; the indexer supplies hints only. [OPN-006] _Source: design/downloads.md §3.4 notes; design/acquisition.md §3.11; open-issues C (Acquisition)._

**DLD-012** (MUST) `Download.private` **MUST** be set from the indexer definition at grab and OR'd with the client's reported private flag when the adapter reports one (`Caps.ReportsPrivate`). Once true it **MUST NOT** revert to false. _Source: design/downloads.md §4.4 step 1._

**DLD-013** (MUST) The `download.submit` job **MUST**, in order:
1. fetch the payload through acquisition's `Fetch` (which takes a grab budget ticket);
2. for torrents, compute the info hash from the `.torrent` or magnet and persist it as `marker` and `info_hash` before calling the client;
3. apply the grab-time file-content check (DLD-063);
4. select a client (DLD-023);
5. call `Client.Add` with the marker, category/labels and save path;
6. on acknowledgement, set `client_ref`, CAS `submitting → downloading`, set `retention=active`, and request an immediate poll of that client.
_Source: design/downloads.md §3.5, §4.1 step 1._

**DLD-014** (MUST) Submission **MUST** be idempotent on the marker. A submit that finds its marker already present in the client **MUST** record the client reference and continue, instead of adding a second item. The poll **MUST** reconcile a download stuck in `submitting` whose marker appears in a complete snapshot (a crash between `Add` and the CAS). _Source: design/downloads.md §3.5 transition rules._

**DLD-015** (MUST) If the payload cannot be fetched because the release is no longer available, or the grab-time content check blocks every media file, the download **MUST** go to `failed` with reason `grab_unavailable` or `grab_content_blocked` respectively, the release **MUST** be blocklisted, and a typed error **MUST** be returned to acquisition so that it can try the next candidate from the same result set. _Source: design/downloads.md §3.5, §3.7._

**DLD-016** (MUST) When a new Download is created with a target `(item, slot)` that is still pending on an older Download in `submitting` or `downloading`, the older target **MUST** be marked `superseded`. When an older Download has no remaining `pending` targets:
- on a public tracker (or Usenet), it **MUST** go to `abandoned(superseded)` with client disposition `remove_data` and no blocklist entry;
- on a private tracker, it **MUST** go to `abandoned(superseded)` with disposition `keep`, leaving the client item running under the seeding manager.
Older Downloads already in `extracting`, `processing` or `importing` **MUST NOT** be superseded; the importer's upgrade check decides. [OPN-009] [OPN-106] _Source: design/downloads.md §6.2 Q2; open-issues C (Downloads); D20._

### 3. Download-client abstraction

**DLD-017** (MUST) Every adapter **MUST** implement the core interface:

```go
type Client interface {
    Caps() Caps
    Add(ctx context.Context, req AddRequest) (ClientRef, error)       // idempotent on req.Marker
    List(ctx context.Context, scope ListScope) (Snapshot, error)      // ScopeOurs (category/marker) | ScopeAll
    Remove(ctx context.Context, ref ClientRef, deleteData bool) error // "not found" is success
    Health(ctx context.Context) error
}
```

Optional capabilities **MUST** be separate interfaces discovered by type assertion and gated by `Caps`: `FileSelector` (list files, skip file indexes), `Labeler` (add/remove labels), `Stopper` (stop/start), `Reannouncer`, `GlobPusher` (push excluded-file globs), `SeedLimitSetter`. _Source: design/downloads.md §3.3._

**DLD-018** (MUST) `Caps` **MUST** declare at least: protocol (`torrent | usenet | ddl`); label kind (`none | category | tags | directory`); `FileSelection`; `FileListAtAdd`; `TrackerStatus`; `SwarmCounts`; `LastActivity`; `AltSpeedProbe`; `PerItemSeedLimits`; `ExcludedGlobs`; `ReportsPrivate`; marker kind (`info_hash | nzb_name | pp_param | file_name | job_id`); `Unpacks` (client repairs and unpacks itself); `Progress`; push kind (`none | completion_hook`). Policies and actions that need a capability the client lacks **MUST** be skipped for that client, and the UI **MUST** show which policies are inapplicable. _Source: design/downloads.md §3.3._

**DLD-019** (MUST) `AddRequest.Payload` **MUST** be one of `TorrentFile | Magnet | NZB | LinkGroups`, so that DDL (P3) uses the same interface. _Source: design/downloads.md §3.3; D11._

**DLD-020** (MUST) `Snapshot` **MUST** carry `OK` and `Complete`. An adapter **MUST** set `Complete=false` when the listing looks collapsed (for example zero items where the previous complete snapshot had many, or a paged listing that ended early). Decisions based on the **absence** of an item (vanished, orphan) **MUST NOT** be taken on a snapshot that is not complete. _Source: design/downloads.md §3.3 (Cleanuparr `ThrowIfTorrentListCollapsed`)._

**DLD-021** (MUST) The adapter **MUST** normalise each item into an `Observation` with:
- `State ∈ {Queued, Metadata, Downloading, Paused, Checking, Completed, Seeding, Stopped, Failed, Warning}` and a typed `StateDetail` (`MissingFiles`, `ClientError`, `DHTDisabled`, …);
- `Downloaded`, `Total`, `Wanted` (after skipped files) bytes; `DownRate`, `UpRate`, `ETA`;
- `Seeders`, `Leechers`, `Ratio`, `SeedingTime`, `LastActivity`, `Private` where supported;
- `Tracker ∈ {OK, Unregistered, Down, Unknown}` from the shared classifier;
- `OutputPath` (local, after path mapping) and `Files` when cheap;
- `FailReason ∈ {None, UsenetRepairFailed, UsenetUnpackFailed, UsenetMissingArticles, Encrypted, DuplicateRejected, DiskFull, Other}`;
- `AltSpeedActive`.
_Source: design/downloads.md §3.3._

**DLD-022** (MUST) Each client **MUST** have one base category (default `bonnyread`) with an optional per-media-kind suffix (for example `bonnyread-tv`), used for client save paths and to scope `List`. Identity **MUST NOT** depend on labels. Lifecycle labels (`bonnyread:seeding`, `bonnyread:failed`, `bonnyread:unlinked`) **MAY** be mirrored as cosmetic tags where the client supports multiple labels, and **MUST NOT** be applied by moving categories. _Source: design/downloads.md §3.3 Labels._

**DLD-023** (MUST) Client selection **MUST** filter by protocol, then by tags (a client with tags needs a matching tag; untagged clients are the fallback), then apply the per-indexer client override, then order by priority group, and round-robin within a group using a cursor stored in the database. The submit job **MUST** try the ordered list and fall back to the next client on `Unavailable` or `Rejected`. If every client fails, the Download **MUST** go to `abandoned(client_unavailable)` and the release **MUST** return to evaluation as a `ClientUnavailable` hold (`04-evaluation.md`). _Source: design/downloads.md §3.3 Client selection; O15._

**DLD-024** (MUST) Remote path mappings **MUST** be per client, `(client_id, remote_prefix) → local_prefix`, longest prefix wins. They **MUST** be applied once when an observation is ingested, and inversely when a save path is sent. Import **MUST** receive only local paths. Mappings **MUST NOT** vary per node. _Source: design/downloads.md §3.3; D14._

**DLD-025** (MUST) A health check **MUST** verify, per client, that the client is reachable and that each mapped download root exists and is readable on every node (node-scoped check), and raise a typed health issue otherwise (`12-platform.md`). _Source: design/downloads.md §3.3._

**DLD-026** (MUST) The API **MUST** expose `POST /api/v1/download-clients/{id}/notify`, authorised by a scoped API key, which triggers an immediate poll of that client (DLD-007), debounced to at most one poll per 2 s. The body **MUST** be ignored beyond an optional client reference. There **MUST NOT** be a `DownloadedEpisodesScan`-style endpoint (D7). _Source: design/downloads.md §3.3 Polling vs push; D7; contract in `11-api-and-clients.md`._

### 4. Adapters

**DLD-027** (MUST) Phase 1 **MUST** ship these adapters:

| Adapter | Protocol | Library | Marker | Notes |
|---|---|---|---|---|
| qBittorrent (Web API v2 only) | torrent | `autobrr/go-qbittorrent` | info hash | `sync/maindata` incremental listing; tags; file priorities; `excluded_file_names`; reannounce |
| Transmission ≥ 4.0 | torrent | `hekmon/transmissionrpc/v3` | info hash | labels; `files-unwanted`; reannounce |
| Deluge 2 | torrent | `autobrr/go-deluge` (GPL-2.0-or-later, used under AGPL-3.0) | info hash | label plug-in; file priorities |
| rTorrent | torrent | `autobrr/go-rtorrent` | info hash | file priorities |
| SABnzbd | usenet | native (small JSON API) | `nzbname` = Download UUID | `Unpacks`; history `fail_message` classified |
| NZBGet (current versions only) | usenet | native (JSON-RPC) | PP parameter `*bonnyread=<uuid>` | `Unpacks`; status classified |
| Torrent blackhole | torrent | in-process | file name carries UUID | no progress, no labels, no removal of seeding items |
| Usenet blackhole | usenet | in-process | file name carries UUID | as above |

_Source: design/downloads.md §3.3 Supported clients; analysis/autobrr.md §client table; D5._

**DLD-028** (SHOULD) After phase 1, bonnyread **SHOULD** add Aria2 (JSON-RPC, as in autobrr `pkg/aria2`) and Flood adapters behind the same interface. Synology Download Station, RQBit, Tribler and Freebox **MAY** follow on demand. _Source: design/downloads.md §3.3; analysis/autobrr.md._

**DLD-029** (MUST NOT) bonnyread **MUST NOT** implement qBittorrent API v1, NZBGet v12/v13 compatibility paths, Deluge 1.x, uTorrent, Vuze, Hadouken, NZBVortex or Pneumatic. _Source: design/downloads.md §3.3; analysis/autobrr.md._

**DLD-030** (MUST) A blackhole client **MUST** write the `.torrent`/`.nzb` into its watch folder, named with the Download UUID, and **MUST** detect completion from its completed folder only after the entry's total size has been stable for `settle_seconds` (default 60). Only `max_age` and import-related policies apply to blackhole downloads, and their retention axis stays `none`. _Source: design/downloads.md §3.3 Blackhole; O19._

**DLD-031** (MUST) [P3] The DDL downloader **MUST** be an in-process `Client` whose `Add` takes `LinkGroups`. Host plug-ins (WASM, D15) and native host drivers **MUST** only **resolve** a link into a URL, request headers, cookies and quota information; the core **MUST** stream the bytes itself. _Source: A11; D11; design/downloads.md §3.9.4._

**DLD-032** (MUST) [P3] Each DDL transfer **MUST** run as a durable job with a heartbeat, streaming to `<staging_root>/<uuid>.partial` with HTTP `Range` resume, so that another node can resume it after the job is reclaimed. It **MUST** report observations through the same `Observation` struct without a poll. On success the payload **MUST** be treated as `owned` (DLD-083). _Source: design/downloads.md §3.9.4; A1._

**DLD-033** (MUST) [P3] DDL failures **MUST** be handled as follows:
- `link_dead`: try the next link in the group, then the next group;
- all groups dead: `failed(ddl_no_working_links)`, blocklist the release by `(indexer, guid)`;
- `host_quota`: **not a failure**. Every queued or running transfer for that host **MUST** be paused with `next_attempt_at` set to the quota reset time from the shared budget service (`03-acquisition.md`);
- `cf_challenge` with no solver configured: `import_blocked(ddl_challenge)` with a notification.
_Source: design/downloads.md §3.9.4; design/acquisition.md (host quotas use the budget service)._

### 5. Polling, matching and observation

**DLD-034** (MUST) A client item **MUST** be matched to a Download only by `(client_id, client_ref)` or by marker. Titles **MUST NOT** be parsed to identify items. _Source: design/downloads.md I1._

**DLD-035** (MUST) Each client **MUST** be polled on an adaptive interval: **fast** (default 5 s) while it has downloads in `submitting`, `downloading` or `extracting`, or while a UI is subscribed to the queue; **slow** (default 60 s) otherwise; **seeding-only** (default 300 s) when it holds only seeding items. A grab, a user action or a push hint **MUST** trigger an immediate poll, debounced by 2 s. _Source: design/downloads.md §3.3._

**DLD-036** (MUST) Completion **MUST** be settled before the download leaves `downloading`: a qBittorrent item in `checkingUP` (or any adapter's `Checking` state) is not complete; a Usenet item is complete only when the client's post-processing has finished (SABnzbd history `Completed`); a blackhole item needs a stable size (DLD-030). _Source: design/downloads.md §3.5 Completion settles._

**DLD-037** (MUST) Items in bonnyread's category without a bonnyread marker **MUST** be listed as `unmanaged` in the queue view and **MUST** be ignored by policies, the seeding manager and orphan housekeeping (they are claimed content). _Source: design/downloads.md §3.5 Foreign items._

**DLD-038** (MAY) In phase 1 the user **MAY** "adopt" an unmanaged item, which creates a Download with targets chosen through the importer's manual matching (`06-import.md` IMP-043). If adoption is not built in phase 1, the replacement is manual import from the item's path. [OPN-015] _Source: design/downloads.md §6.2 Q9; open-issues C (Downloads)._

**DLD-039** (MUST) A managed item that is missing from at least 3 consecutive **complete** snapshots spanning at least 3 minutes **MUST** become `abandoned(vanished)` if it was in `submitting` or `downloading`, or **MUST** set `retention=vanished` if it was past `downloading`. A blocklist entry or search **MUST NOT** follow. An unreachable client yields no complete snapshot, so nothing vanishes during a client outage. _Source: design/downloads.md §3.5 Vanished._

### 6. Phase and retention state machines

**DLD-040** (MUST) `Download.phase` **MUST** take exactly the values `submitting`, `downloading`, `extracting`, `processing`, `importing`, `import_blocked`, `imported`, `failed`, `abandoned`. `imported`, `failed` and `abandoned` are terminal. `processing` is the external processing hold (§10). _Source: design/downloads.md §3.5; D20._

**DLD-041** (MUST) `Download.retention` **MUST** take exactly the values `none`, `active`, `seeding`, `stopped`, `removed`, `vanished`, and **MUST** change independently of phase. A failed private torrent **MAY** keep seeding; an imported torrent **MAY** be dead. _Source: design/downloads.md §3.1.2, §3.5._

**DLD-042** (MUST) Allowed phase transitions:

| From | To | Trigger |
|---|---|---|
| submitting | downloading | client acknowledgement or poll reconciliation (DLD-014) |
| submitting | failed | `grab_unavailable`, `grab_content_blocked` |
| submitting | abandoned | `client_unavailable`, `user_ignored`, `superseded` |
| downloading | extracting | settled completion and extraction needed (DLD-068) |
| downloading | processing | settled completion, no extraction, a processing rule matches (DLD-076) |
| downloading | importing | settled completion, no extraction, no processing rule |
| downloading | failed | policy `fail`, immediate condition, client `FailReason`, post-metadata content check, user fail |
| downloading | abandoned | `vanished`, `superseded`, `user_ignored` |
| extracting | processing / importing | extraction success |
| extracting | failed / import_blocked | typed extraction error (DLD-073) |
| processing | importing | all held files marked ready (DLD-079) |
| processing | import_blocked / failed | files marked failed, or timeout (DLD-080) |
| importing | importing | transient retry at `next_attempt_at`, or accepted override |
| importing | imported / failed / import_blocked | reason policy (DLD-086) |
| import_blocked | importing | user action (retry, supply password, manual import), space freed, or fixed configuration |
| import_blocked | failed | `fail_after_seconds` elapsed, or user fail |
| any non-terminal | failed / abandoned | user fail / user ignore |

Any transition not in the table **MUST** be rejected. _Source: design/downloads.md §3.5._

**DLD-043** (MUST) Every phase transition **MUST** be a compare-and-swap on `(id, version, phase)` that increments `version`, and the follow-up job (extract, import, client action, search intent) **MUST** be enqueued in the same transaction. A CAS that affects zero rows **MUST** be treated as "someone else moved it" and its side effects discarded. _Source: design/downloads.md §3.5 Transition rules; `12-platform.md` job queue as outbox._

**DLD-044** (MUST) Writers per phase band: the client poll job writes all observation and clock fields and the transitions out of `submitting` (reconciliation) and `downloading`; extraction jobs write transitions out of `extracting`; the processing-hold handler writes transitions out of `processing`; import jobs write transitions out of `importing`; user actions **MAY** write any transition allowed in DLD-042. _Source: design/downloads.md §3.5._

**DLD-045** (MUST) Retention transitions: `none → active` on client acknowledgement; for torrents `active → seeding` when the item completes; `seeding → stopped | removed` only by the seeding manager or a user action; for Usenet and DDL `active → removed` (or `none`) after import or failure per the client's removal setting; any value `→ vanished` per DLD-039. `Download.data_deleted` **MUST** record whether a removal deleted payload data. _Source: design/downloads.md §3.5, §3.10._

**DLD-046** (MUST) Housekeeping **MUST** raise an alert (not change state) for any download whose phase has not changed for longer than a ceiling: `submitting` > 30 min, `extracting` > 12 h, `importing` > 24 h, `processing` beyond its rule's timeout plus 1 h. [OPN-012] _Source: design/downloads.md §3.11 step 4._

### 7. Multiple slots per download (D20)

**DLD-047** (MUST) Each `DownloadTarget` **MUST** carry `item_id`, `slot_id`, `outcome ∈ {pending, imported, not_wanted, rejected, skipped, superseded}`, the typed rejection reason when rejected, and the resulting `media_file_id` when imported. A Download **MUST NOT** have a single `slot_id` column; its slots are the set of its targets' slots. _Source: D20; design/downloads.md §3.4 (DownloadTarget), §6.1 item 5 (superseded)._

**DLD-048** (MUST) Queue views, history, failure counting, the recurrence guard and redownload **MUST** work per `(item, slot)` target, not per Download. A Download targeting slots A and B appears in the queue of both slots. _Source: design/downloads.md I8, amended by D20._

**DLD-049** (MUST) At import, one `ImportRequest` **MUST** carry every pending target of the Download, and the importer **MUST** place the payload into every targeted slot that accepts it, each as its own library file (`06-import.md` IMP-037). Targets are resolved independently: one slot **MAY** import while another rejects. _Source: D20; open-issues B1(a)._

**DLD-050** (MUST) After import, each slot's file **MUST** be upgraded, deleted and unlinked independently. A later upgrade in slot A **MUST NOT** affect slot B's file or the payload's link from slot B. _Source: D20 ("upgrades independently")._

### 8. Time-based policies

**DLD-051** (MUST) On each poll with a complete snapshot, for each download in `downloading` the poll job **MUST** update the condition clocks as follows:

```
Δ = min(now − observed_at, 2 × poll_interval)          // outage gaps never count
if network_degraded or client.degraded: Δ = 0
active := obs.State ∈ {Downloading, Metadata} or (obs.State == Warning and detail is stalled)
if active:
    if obs.Downloaded − progress_mark_bytes ≥ min_progress: no_progress_s = 0; progress_mark_bytes = obs.Downloaded
    else:                                                    no_progress_s += Δ
    metadata_s = obs.State == Metadata ? metadata_s + Δ : 0
    slow_s     = (mean DownRate over window < min_speed and !(ignore_alt_speed and obs.AltSpeedActive)) ? slow_s + Δ : 0
    slow_eta_s = (obs.ETA > max_eta) ? slow_eta_s + Δ : 0
unregistered_s = obs.Tracker == Unregistered ? unregistered_s + Δ : 0
no_seeders_s   = (Caps.SwarmCounts and obs.Seeders == 0) ? no_seeders_s + Δ : 0
```

Paused and client-queued items **MUST** freeze the clocks (no increment, no reset). `min_progress` **MUST** be the smallest `min_progress_bytes` among the policies in scope for the download. The same clock rule **MUST** drive the seeding-time inactivity counter `inactive_s` when the client lacks `LastActivity`. _Source: design/downloads.md §3.6 Clocks; I9._

**DLD-052** (MUST) "Stalled" **MUST** be defined as the `no_progress` condition over bonnyread's observed bytes, identically for every client. Client state strings **MUST** be used only to decide whether a clock runs. _Source: design/downloads.md §3.6._

**DLD-053** (MUST) The health subsystem **MUST** expose `network_degraded` (connectivity probes) and per-client `degraded` (failed or incomplete snapshots); either **MUST** set Δ = 0. There **MUST NOT** be a separate connectivity gate or strike counter. _Source: design/downloads.md §3.6 Health gate; `12-platform.md`._

**DLD-054** (MUST) `DownloadPolicy.condition` **MUST** be one of:

| Condition | Clock / test | Applies |
|---|---|---|
| `no_progress` (stalled) | `no_progress_s` | torrent, DDL |
| `metadata` (metadata stuck) | `metadata_s` | torrent (magnets) |
| `slow_speed` | `slow_s` with params `min_speed`, `window_seconds`, `ignore_when_alt_speed` | torrent, usenet |
| `slow_eta` | `slow_eta_s` with param `max_eta` | torrent, usenet |
| `unregistered` | `unregistered_s` while downloading | torrent with `TrackerStatus` |
| `no_seeders` | `no_seeders_s` | torrent with `SwarmCounts` |
| `max_age` | wall time since `grabbed_at` | any; the only one for blackhole |
| `import_blocked` | time in `import_blocked`, optionally restricted to a reason class | any |
| `extract_failed` | time in `import_blocked` with an extraction reason | any |

_Source: design/downloads.md §3.4 DownloadPolicy, §3.6._

**DLD-055** (MUST) These conditions **MUST** fire at once without a policy: `all_files_skipped` (the item completes with zero wanted bytes, DLD-064); a client `FailReason` other than `None`; `Encrypted`. Each **MUST** fail the download with the corresponding reason, blocklist it and request a redownload (DLD-089). _Source: design/downloads.md §3.6 Evaluation step 4, §3.7; O4._

**DLD-056** (MUST) For each download, the evaluator **MUST** collect enabled policies whose scope matches (protocols, privacy `public | private | any`, indexers, clients, tags, media kinds, completion band `(min, max]` percent, size bounds) and **MUST** keep, per condition, the matching policy with the highest priority. A policy fires when its clock reaches `after_seconds`. _Source: design/downloads.md §3.6 Evaluation._

**DLD-057** (MUST) Policy actions **MUST** be:

| Action | Effect |
|---|---|
| `fail` | the failure handler with `{blocklist, search, client_disposition}` from `action_params` (DLD-089) |
| `notify` | emit `download.attention{condition}` once per condition episode; the episode ends when the clock resets |
| `stop` | stop the item in the client (`Stopper`); the phase is unchanged and `stopped_by_policy` is set |
| `relabel` | add a cosmetic label (DLD-022) |

_Source: design/downloads.md §3.6 Actions._

**DLD-058** (MUST) The default `client_disposition` for `fail` **MUST** be `remove_data` for public torrents and Usenet, and `keep` for private torrents. A kept private item **MUST** pass to the seeding manager with `phase=failed`. _Source: design/downloads.md §3.6; O14; I6._

**DLD-059** (MUST) A fresh installation **MUST** seed these policies (values tunable, not hard-coded):

| Condition | Scope | Threshold | Action |
|---|---|---|---|
| metadata | torrent, any privacy | 1 h | fail (blocklist, search) |
| no_progress | torrent, public | 6 h, min progress 50 MB | fail (blocklist, search, remove_data) |
| no_progress | torrent, private | 24 h, min progress 50 MB | fail (blocklist, search, keep) |
| unregistered | torrent, while downloading | 1 h | fail (blocklist, search) |
| slow_speed / slow_eta | — | disabled | — |
| max_age | blackhole | 7 d | notify |
| import_blocked (needs_user) | any | 0 | notify; no auto-fail |

[OPN-012] _Source: design/downloads.md §3.6 Default policy set; open-issues C (Downloads: default timeouts tuned in beta)._

**DLD-060** (MUST) Simulate mode **MUST** be available globally and per policy. In simulate mode the evaluator's actions **MUST** be recorded as `DownloadEvent{type=would_<action>}` and side effects (client call, phase change, blocklist, search intent) **MUST NOT** happen. _Source: design/downloads.md §3.6 Simulate mode; O18._

**DLD-061** (SHOULD) The queue API **SHOULD** expose each running clock with the threshold of the policy that would fire (for example "no progress for 5 h 40 min; fails at 6 h"). _Source: design/downloads.md §4.1 step 9._

### 9. File-content policy

**DLD-062** (MUST) There **MUST** be one `FileContentPolicy` per media kind, in `denylist` or `allowlist` mode, with rules as globs or `regex:` patterns in the Cleanuparr list format (importable, optionally refreshed from a `source_url`), and `on_blocked ∈ {skip_files, fail_if_all, fail_if_any}`. File classes (media, extra, archive, sample, executable, dangerous, other) **MUST** come from the import layer's extension registry (`06-import.md` IMP-005). _Source: design/downloads.md §3.7._

**DLD-063** (MUST) Checkpoint 1, at grab (in `submitting`), when the file list is known (`.torrent`, Newznab `files` attributes, or NZB file subjects): if every file of the media class is blocked, the download **MUST** fail with `grab_content_blocked` per DLD-015; otherwise blocked file indexes **MUST** be recorded as `DownloadFile.verdict=skip`. _Source: design/downloads.md §3.7 item 1._

**DLD-064** (MUST) Checkpoint 2, after metadata (magnets; and immediately after `Add` for `.torrent` files): with `FileSelection`, the adapter **MUST** skip the blocked files; without it, `on_blocked` applies (`fail_if_any` fails; otherwise accept). A download that completes with zero wanted bytes **MUST** fail with `all_files_skipped`. _Source: design/downloads.md §3.7 item 2._

**DLD-065** (MUST) Checkpoint 3, at import, is the importer's class-based rejection (`DangerousFile`, `ExecutableFile`, `UserRejectedExtension`; `06-import.md` IMP-006); these codes map to reason class `content_blocked` (DLD-084). _Source: design/downloads.md §3.7 item 3._

**DLD-066** (MUST) The archive class **MUST NOT** be treated as blocked content when extraction is enabled for the download's client. Comic archives (`cbz`, `cbr`, `cb7`) **MUST** be treated as media containers for comics, and never extracted or blocked. _Source: design/downloads.md §3.7._

**DLD-067** (MAY) A policy with `push_to_client=true` **MAY** push its deny globs to qBittorrent `excluded_file_names`. The push **MUST** be idempotent by content hash stored on the client row. _Source: design/downloads.md §3.7._

### 10. Extraction and staging

**DLD-068** (MUST) On settled completion the poll job **MUST** decide whether extraction is needed: the client's `extraction` mode is `always`, or `auto` and the client does not `Unpacks`; **and** `xtractr.FindCompressedFiles(output_path)` finds archives that are not media containers for the media kind. If needed, it **MUST** CAS to `extracting` and enqueue an `extract` job; otherwise it continues per DLD-042. _Source: design/downloads.md §3.8 Trigger._

**DLD-069** (MUST) Extraction output **MUST** go to `<staging_root>/<download_uuid>/`, never into the client's download folder. A staging root **MUST** exist on the same filesystem as each library root folder that a download may import into (default `<root>/.bonnyread-staging`, configurable per root). The payload in the client **MUST NOT** be modified. _Source: design/downloads.md §3.8; A5._

**DLD-070** (MUST) Every extraction attempt **MUST** first delete `<staging_root>/<uuid>` and `<uuid>.partial`, extract into `<uuid>.partial/`, and rename it to `<uuid>/` only on success, so that import never sees partial output and a crashed attempt leaves nothing behind. _Source: design/downloads.md §3.8; I7._

**DLD-071** (MUST) Before extracting, the job **MUST** check `need + reserve ≤ free(staging_root)`, where `need` is the uncompressed size from RAR, 7z or zip headers, or archive size × 1.1 when headers give none. If the check fails the download **MUST** go to `import_blocked(disk_full)` and be retried when the free-space health check clears or the user retries. _Source: design/downloads.md §3.8 Job step 1._

**DLD-072** (MUST) Extraction **MUST** use `golift.io/xtractr` (MIT, a pinned tagged release) with: passwords tried in the order per-indexer password, global password list, `{{password}}` in the release name; limits `MaxBytes` per media kind (defaults TV 20 GB, movie 75 GB, music 4 GB, book 1 GB), `MaxFiles` 1000, `MaxRatio` 5, `MaxNested` 8, `ExtrasMaxDepth` 3; progress reported to the queue (SSE). Jobs **MUST** run in the `io` queue with a default parallelism of 1 per node. _Source: design/downloads.md §3.8 Job step 2; `12-platform.md` queues._

**DLD-073** (MUST) Extraction errors **MUST** be typed:

| Reason | Class | Default |
|---|---|---|
| `extract_io` | transient | retry ×3 with backoff 5 min, 20 min, 1 h, then fail |
| `extract_corrupt` (CRC, truncated or missing volume) | permanent | fail + blocklist + search |
| `extract_password_required` | needs_user | `import_blocked`; a user-supplied password retries |
| `extract_limit_exceeded` | permanent | fail + blocklist |
| `extract_unsupported` | needs_user | `import_blocked` |
| `disk_full` | transient | `import_blocked` until space is available |

_Source: design/downloads.md §3.8 step 4._

**DLD-074** (MUST) [P2] For music, a FLAC+CUE image **MUST** be split as an extraction sub-step (xtractr) whose output feeds the music importer directly. _Source: design/downloads.md §3.8._

### 11. External processing hold (D20)

**DLD-075** (MUST) Users **MUST** be able to define `ProcessingRule`s (name, enabled, priority, scope over media kinds, slots, tags, indexers, clients and protocols; `timeout_seconds`; `on_timeout`; `on_failed`). No rule exists by default. _Source: D20 (import-time processing); open-issues B6(a)._

**DLD-076** (MUST) When a download completes (after extraction, if any) and at least one enabled ProcessingRule matches, the download **MUST** CAS to `processing` instead of `importing`. The highest-priority matching rule applies. _Source: D20._

**DLD-077** (MUST) Before emitting the hold event, the payload's media-class files **MUST** be available in a bonnyread-owned working directory `<staging_root>/<uuid>/`: extraction or DDL output is already there; otherwise each file **MUST** be copied in by reflink, falling back to a full copy, and **MUST NOT** be hardlinked, because the external tool may rewrite it. The client's copy **MUST NOT** be handed to the tool. After the hold, the payload for import is `owned`. _Source: D20; design/import.md §3.1.5 ("never mutate the source")._

**DLD-078** (MUST) Entering `processing` **MUST** publish a durable `download.processing_requested` event (webhook, script and notification targets per `12-platform.md`) carrying the download id, targets, release snapshot, working directory, the media files (relative path, size, class) and the deadline. _Source: D20 ("emits an event")._

**DLD-079** (MUST) The API **MUST** accept `POST /api/v1/downloads/{id}/processing` from a principal holding `activity:update:processing` (`10-identity-and-access.md` IAM-061), with either a whole-download result `{result: ready|failed, detail?}` or per-file results `{files: [{path, result: ready|failed, detail?}]}`. A file marked `ready` **MAY** have been replaced or renamed inside the working directory; files may be added. When every held file is resolved:
- if all are `ready`, the download **MUST** CAS `processing → importing` and the importer **MUST** discover the working directory afresh (`06-import.md` IMP-004);
- if any is `failed`, the rule's `on_failed` applies (default `import_blocked(processing_failed)` with a notification and no blocklist).
A callback for a download not in `processing` **MUST** return `409 Conflict` without changing state. [OPN-107] _Source: D20 ("resumes when an external tool marks the file ready or failed through the API"); contract in `11-api-and-clients.md`._

**DLD-080** (MUST) If not every held file is resolved by `processing_at + timeout_seconds` (wall-clock database time), the rule's `on_timeout` **MUST** apply: `import_blocked(processing_timeout)` with a notification (default), `fail` (no blocklist), or `import_unprocessed` (continue to import with the files as they are). The default timeout **MUST** be 6 h. [OPN-108] _Source: D20; task scope._

**DLD-081** (MUST) The user **MUST** be able to release a hold manually (import as-is) or fail it from the queue. _Source: D20._

### 12. Import hand-off and reason policy

**DLD-082** (MUST) When a download enters `importing`, an `import` job **MUST** be enqueued with an `ImportRequest` (`06-import.md` IMP-002) that carries the Download id, all targets with outcome `pending`, the release snapshot, the source files (from `output_path` or the staging/working directory), the payload ownership, and the overrides already accepted by the reason policy. The importer **MUST** resolve items from the targets, never from the title. _Source: design/downloads.md §3.9.1._

**DLD-083** (MUST) Payload ownership **MUST** be:
- `client_seeding` for a torrent still in its client (transfer **MUST** retain the source);
- `client_done` for a completed Usenet or blackhole item whose client copy will be removed after import (move allowed);
- `owned` for extraction output, DDL output and processing-hold output (move within the staging filesystem).
_Source: design/downloads.md §3.9.1 Payload rules._

**DLD-084** (MUST) Each import rejection code (`06-import.md` IMP-030) **MUST** map to a reason class. The default map **MUST** be the importer's class (`transient → transient`, `permanent → permanent`, `needs_user → needs_user`) with these exceptions:

| Reason class | Codes |
|---|---|
| `content_blocked` | `DangerousFile`, `ExecutableFile`, `UserRejectedExtension`, `NoMediaFiles` (including samples-only payloads) |
| `permanent` | `NoAudioTrack`, `WrongMedium`, `NoSlotAccepts`, `extract_corrupt`, `extract_limit_exceeded` |
| `overridable` | `SampleIndeterminate`, `GrabMismatch`, `FolderMismatch`, `UnverifiedSceneNumbering` |
| `transient` | `FileLocked`, `MinimumFreeSpace`, `RootFolderMissing`, `Unpacking`, `TitleRequired` |
| `needs_user` | `Unparseable`, `UnknownItem`, `AmbiguousMatch`, `SlotAmbiguous`, `DestinationOccupied`, `SourceNotAccessible`, `SourceMissing`, `ProbeFailed`, `extract_password_required`, `extract_unsupported`, `processing_failed`, `processing_timeout` |
| `not_wanted` | `AlreadyImported`, `NotQualityUpgrade`, `NotRevisionUpgrade`, `NotCustomFormatUpgrade`, `ExistingFileCoversMoreItems` |

A user **MAY** override the class or action of a single code with an `ImportReasonPolicy` row. [OPN-109] _Source: design/downloads.md §3.9.2; design/import.md §3.2 stage 6._

**DLD-085** (MUST) Default actions per class:

| Class | Default action |
|---|---|
| `content_blocked` | fail + blocklist + search |
| `permanent` | fail + blocklist + search |
| `overridable` | accept: re-run the import once with the code in `Overrides`; the override **MUST** be recorded in history |
| `transient` | stay in `importing` with `next_attempt_at` backoff 1, 5, 15, 60 min, doubling to a cap of 6 h; after 24 h in total → `import_blocked(transient_exhausted)` + notify |
| `needs_user` | `import_blocked` + notify once (`attention_notified`); auto-fail only if `fail_after_seconds` is set |
| `not_wanted` | the target is `not_wanted`; no blocklist; the payload goes to the seeding manager as unlinked |
| `partial` | see DLD-087 |

_Source: design/downloads.md §3.9.2._

**DLD-086** (MUST) Given an `ImportOutcome`, the download lifecycle **MUST**:
1. if any rejection is `overridable` with action `accept` and that code is not yet in `Overrides`, re-enqueue the import with the code added and stop;
2. if a request-scope rejection is `content_blocked`, fail the download;
3. set each target's outcome (`imported`, `not_wanted`, `rejected(code)`, `skipped`);
4. if every target is `imported`, `not_wanted`, `skipped` or `superseded`, CAS to `imported` (reason `not_an_upgrade` when no target imported);
5. if at least one target is `imported` and at least one is `rejected`, apply the `partial` class;
6. otherwise apply the action of the highest-precedence class among the rejected targets, in the order `content_blocked`, `permanent`, `needs_user`, `transient`.
_Source: design/downloads.md §3.9.2; D20 (per-target outcomes)._

**DLD-087** (MUST) A partial outcome **MUST** keep the imported targets, set the download to `import_blocked(partial)` and notify, until the user acts. A policy option `accept_partial_after` (disabled by default) **MUST**, when set, move the download to `imported` after that duration and return the rejected targets to wanted. [OPN-010] _Source: design/downloads.md §3.9.2, §6.2 Q3; open-issues C (Downloads)._

**DLD-088** (MUST) Entering `import_blocked` **MUST** publish `download.manual_interaction_required` exactly once per blocked episode, with the typed reason and the available actions (retry, manual import, supply password, fail, ignore). _Source: design/downloads.md O3, §3.14._

### 13. Failure handling, blocklist and redownload

**DLD-089** (MUST) `Fail(download, reason, opts{blocklist, search, client_disposition})` **MUST**, in one transaction:
1. CAS the phase to `failed`, set `reason`, and write a `DownloadEvent`;
2. if `opts.blocklist` and the reason is not temporary (host quota, client unavailable), insert a Blocklist row (DLD-090);
3. record the client disposition: `remove` or `remove_data` enqueues a `client.remove` job; `keep` or `stop` leaves retention to the seeding manager; private torrents default to `keep`;
4. if `opts.search` and redownload is allowed for the download's source, upsert one `SearchIntent{lane=redownload}` per target whose outcome is not `imported` or `not_wanted`; downloads from `interactive` grabs **MUST NOT** redownload unless the user enables it;
5. publish `download.failed{reason}`.
_Source: design/downloads.md §3.9.3; D20 (per-target redownload)._

**DLD-090** (MUST) A Blocklist row **MUST** record identity (`info_hash` for torrents; `(indexer_id, guid)` for Usenet and DDL; `(title, size)` as a fallback for both), protocol, typed reason, optional detail, the originating download, the scope (container and targets, for display) and `expires_at` (null = permanent). Evaluation **MUST** check the blocklist before accepting a release (`04-evaluation.md` EVL-058). Matching **MUST** be global by release identity, never per Title or container; the recorded scope is for display only. [OPN-110] _Source: design/downloads.md §3.4 Blocklist, I4; design/evaluation.md §3.15._

**DLD-091** (MUST) Blocklist entries **MUST** be permanent by default. Entries with reason `no_progress` for **public** torrents **MUST** expire after 30 days. Users **MUST** be able to set or clear expiry per entry and per reason. [OPN-016] _Source: design/downloads.md §6.2 Q10; open-issues C (Downloads)._

**DLD-092** (MUST) The `redownload` lane **MUST** be dispatched at no more than R intents per minute globally (default 2), subject to indexer budgets. _Source: design/downloads.md §3.9.3._

**DLD-093** (MUST) Before dispatching a redownload intent for `(item, slot)`, the scheduler **MUST** count Downloads in phase `failed` with a target on that `(item, slot)` in a sliding window (default 24 h). If the count is ≥ 3, the intent **MUST** be dropped with outcome `dropped(recurrence)`, a `search.intent_dropped{recurrence}` alert **MUST** be raised, and that `(item, slot)` **MUST** be excluded from automatic search until the user acknowledges it or 7 days pass. The count **MUST** be a query over Download rows, not a separate counter. [OPN-012] _Source: design/downloads.md §3.9.3._

**DLD-094** (MUST) Users **MUST** be able to, on any non-terminal download: `fail(blocklist, search)`; `remove(delete_data)` (no blocklist, no search); `ignore` (`abandoned(user_ignored)`; the payload is left alone; a torrent stays with the seeding manager unless the user also detaches it); and on `import_blocked`: retry, manual import, supply an archive password. _Source: design/downloads.md §3.9.3 User actions; O17._

### 14. Seeding manager

**DLD-095** (MUST) The seeding manager **MUST** own every torrent Download once it leaves `downloading` (imported, failed-but-kept, not wanted, superseded-and-kept, and later racing downloads), and **MUST** be the only component that stops or removes them. It **MUST NOT** act while the phase is `extracting`, `processing` or `importing`. _Source: design/downloads.md §3.10; I3._

**DLD-096** (MUST) The seeding manager **MUST** maintain `DownloadLink` rows from library events, not from inode scans:
- on `file.imported` / `file.upgraded` with a `download_id` (`06-import.md` IMP-095): insert a link per new MediaFile with its transfer method, increment `library_refs`, and increment `hardlink_refs` for `hardlink` or `reflink` transfers;
- on `file.deleted` (any reason of `06-import.md` IMP-095: `upgrade`, `user_delete`, `manual_override`, `missing_from_disk`, `item_deleted`, `slot_removed`) or `file.moved` out of the library: delete the link and decrement the counts; when `library_refs` reaches 0, set `unlinked_at` and enqueue an immediate evaluation of that download.
With several slots (D20), each slot's file is its own link. _Source: design/downloads.md §3.10 Library links; D20._

**DLD-097** (MUST) `SeedingRule.unlinked_mode` **MUST** be `provenance` (default: no current library file came from this payload) or `hardlink` (no library file shares storage with the payload). In `hardlink` mode a library file moved to the recycle bin **MUST** continue to count as sharing storage until the recycle bin purges it (`recycle.purged`, `06-import.md` IMP-088). _Source: design/downloads.md §3.10._

**DLD-098** (MUST) For each owned download, the manager **MUST** evaluate, using the first enabled `SeedingRule` by priority whose scope matches (indexers, tracker domains, privacy, media kinds, clients, tags, phase `imported | failed | racing`):

```
met_oblig := obligation satisfied per obligation_mode (DLD-099), or tracker unregistered ≥ rule.unregistered_after
goal      := (rule.max_ratio set and ratio ≥ max_ratio and seeding_time ≥ rule.min_seed_seconds)
          or (rule.max_seed_seconds set and seeding_time ≥ max_seed_seconds)
          or (rule.max_inactive_seconds set and inactive ≥ max_inactive_seconds)
          or (unlinked(rule.unlinked_mode) and rule.when_unlinked ∈ {remove_now, remove_after_obligation})
guard     := rule.min_seeders set and seeders < min_seeders
act       := met_oblig and goal and not guard      // public torrents without an obligation have met_oblig = true
```

When `act` holds, the rule's action (`stop | remove | remove_data`) **MUST** be applied through a `client.*` job. A private torrent **MUST NOT** be removed before `met_oblig` unless the user forces it or the tracker has unregistered it. _Source: design/downloads.md §3.10 Evaluation; I6; O9, O14._

**DLD-099** (MUST) Seed obligations **MUST** support `obligation_mode ∈ {ratio_or_time, ratio_and_time, time_only}` per indexer, defaulting to `ratio_or_time`, populated from the indexer definition's minimum ratio and seed time (Cardigann `minimumratio` / `minimumseedtime`) where available. [OPN-011] _Source: design/downloads.md §3.10, §6.2 Q4; open-issues C (Downloads)._

**DLD-100** (MUST) `remove_data` **MUST** be demoted to `remove` (and the event **MUST** record why) if any of these hold:
- `hardlink_refs > 0`;
- the download has not finished import (`importing`, `import_blocked`, `processing` with client payload);
- any other torrent in the same client (listed with `ListScope=All`, cached per evaluation tick) has a content path overlapping the payload's paths (cross-seed).
_Source: design/downloads.md §3.10; I5._

**DLD-101** (MUST) While seeding, `unregistered_s ≥ rule.unregistered_after_seconds` (default 3600) **MUST** void the obligation. The resulting action **MUST** be `remove` when the payload is linked, and `remove_data` when unlinked and the rule allows it. `no_seeders` **MUST NOT** be evaluated while seeding. _Source: design/downloads.md §3.10 Dead and unregistered; O11; A10._

**DLD-102** (MUST) For a private torrent that reports `Unregistered` within its first minutes after add, the adapter's `Reannouncer` **MUST** be called every 7 s for up to 50 attempts; the `unregistered_s` clock **MUST** start only after that loop ends. _Source: design/downloads.md §3.10 (autobrr `run_transmission.go`, go-qbittorrent `ReannounceTorrentWithRetry`)._

**DLD-103** (MAY) An opt-in client setting **MAY** push `max(obligation, rule)` seed limits to the client (`SeedLimitSetter`) as a safety net while bonnyread is down. It **MUST** be off by default. A client-side removal **MUST** surface as `retention=vanished`. [OPN-014] _Source: design/downloads.md §3.10, §6.2 Q7; open-issues C (Downloads)._

**DLD-104** (MUST) For Usenet, after import or failure the client history item **MUST** be removed per the client's `remove_completed` / `remove_failed` settings, and `retention` set to `removed`. For `client_done` payloads, data **MUST** be deleted only after the importer has moved or copied every placement. _Source: design/downloads.md §3.10 Usenet and blackhole._

**DLD-105** (MUST) The seeding evaluation **MUST** run every 5 min for all owned downloads and immediately, as a single-download job, after a link change or a phase change. _Source: design/downloads.md §3.10._

### 15. Housekeeping

**DLD-106** (MUST) Hourly (default), housekeeping **MUST** treat top-level entries under each client's mapped download roots and each staging root as orphan candidates. An entry is claimed if it is the content path of any item in any client (`ListScope=All`), the staging or working path of a non-terminal download, or the output path of a download in `importing`, `processing` or `import_blocked`. Unclaimed entries older than `min_age` (default 24 h) **MUST** be moved to a quarantine directory (renamed on collision; copy then delete across devices), and quarantined entries older than `purge_after` (default 7 d) deleted. [OPN-012] _Source: design/downloads.md §3.11 step 1; O12._

**DLD-107** (MUST) An orphan run **MUST** be skipped entirely if any client's snapshot is not complete. It **MUST** skip symlinks and honour exclude globs. Configuration validation **MUST** reject scan roots that overlap library roots. _Source: design/downloads.md §3.11._

**DLD-108** (MUST) Staging directories of downloads in terminal phases, or with no Download row, **MUST** be deleted after 1 h. _Source: design/downloads.md §3.11 step 2._

**DLD-109** (MUST) Download rows in a terminal phase with retention `removed`, `none` or `vanished` older than the history retention (default 365 d) **MUST** be deleted, with their `DownloadEvent` rows compacted into the shared history [OPN-203]. Blocklist rows **MUST** be kept until they expire or the user clears them. _Source: design/downloads.md §3.11 step 3._

### 16. Search queue ("when to search")

**DLD-110** (MUST) All automatic and user-triggered "search for this item and slot" requests **MUST** be `SearchIntent` rows in one durable queue with lanes, unique on `dedupe_key = (item, slot, lane)`. How a search runs is owned by `03-acquisition.md`; this file decides what is searched and when. _Source: design/downloads.md §3.12._

**DLD-111** (MUST) Lanes and throttles:

| Lane | Source | Throttle |
|---|---|---|
| `user` | UI or API "search now" | indexer budget only |
| `redownload` | failure handler | DLD-092 and DLD-093 |
| `added` | item/slot newly monitored (search-on-add, requests, follows) | default 10 per minute |
| `backlog` | BacklogRules | trickle (DLD-117) |

Lanes **MUST** be served round-robin, with `user` first. Intents for several slots of the same item **MAY** be coalesced into one search; evaluation then selects per slot (`04-evaluation.md`). _Source: design/downloads.md §3.12._

**DLD-112** (MUST) Before dispatch the scheduler **MUST** ask acquisition `Budget.Available(indexer set, class=search)`. With no budget it **MUST** wait until the reported reset without consuming an attempt or raising an error. _Source: design/downloads.md §3.12 Rate; A3._

**DLD-113** (MUST) A BacklogRule's candidate set **MUST** be the `(item, slot)` pairs that are monitored, have no non-terminal Download targeting them, are past `post_release_grace` (default 6 h) since release or air date, are not excluded by DLD-093, and are `missing`, `cutoff_unmet` (quality), or `cf_below_cutoff` (stored file custom-format score below the profile's cutoff score; per file, never averaged over a series). _Source: design/downloads.md §3.12 Backlog rules._

**DLD-114** (MUST) BacklogRules **MUST** filter candidates with Profilarr's filter DSL (nested AND/OR groups over library fields such as monitored, genres, tags, year, ratings, dates, status, release group, custom-format names, size), with fields generalised per media kind over the library entities of `02-domain-model.md`. The same evaluator **MUST** back the UI preview. _Source: design/downloads.md §3.12; profilarr upgrades filters._

**DLD-115** (MUST) Each rule **MUST** have a selector from at least `balanced_weighted` (default), `oldest_search_first`, `newest_first`, `lowest_score` and `random`. _Source: design/downloads.md §3.12 Selector._

**DLD-116** (MUST) `SearchLedger(item, slot)` **MUST** record `cycle_id`, `last_searched_at` and `last_outcome`. A pair **MUST** be eligible once per cycle; a new cycle **MUST** start only when the rule's pool is exhausted **and** `min_cycle` (default 7 d) has passed since the cycle began. Tags **MUST NOT** be written to represent cooldowns. _Source: design/downloads.md §3.12 Cycles instead of tags._

**DLD-117** (MUST) Each BacklogRule **MUST** have `interval` and `batch` (defaults 10 min, 1 pair), with deterministic jitter. The backlog lane **MUST** be skipped while the number of Downloads in `downloading` whose `no_progress_s = 0` is at or above `active_download_cap` (default 3). [OPN-012] _Source: design/downloads.md §3.12 Rate, Active cap._

**DLD-118** (MUST) A dispatched intent **MUST** carry a `search_id`; the completion signal for that `search_id` **MUST** set the intent's outcome directly, without polling. Acquisition's `SearchCompleted{search_id}` (`03-acquisition.md` ACQ-090) carries no grabs, because grabs are committed later by evaluation; which signal reports `grabbed []DownloadID` is open [OPN-209]. _Source: design/downloads.md §3.12 Observation._

### 17. HA placement

**DLD-119** (MUST) Download work **MUST** be placed per A1 (schedule rows claimed with `SKIP LOCKED` on every node, dedupe and concurrency keys for "at most one execution per subject"; no leader and no lease):

| Work | Mechanism |
|---|---|
| Client poll, observation ingest, clocks, transitions out of `downloading` | One per-entity schedule row per client (`dlclient.poll:<id>`), created and deleted with the client; job concurrency key `dlclient:<id>`, dedupe `WhilePendingOrRunning`, `high` priority. The job sets the row's `next_run_at` from the adaptive interval (DLD-035); hints set it to now and ring the scheduler doorbell. |
| Submit | Job on any node, idempotent by marker; concurrency key `download:<id>` |
| Client actions (remove, stop, label, skip files, reannounce) | Idempotent jobs on any node; "not found" is success |
| Extraction | `io`-queue job on any node; concurrency key `download:<id>` |
| Processing-hold timeout | A job scheduled at the deadline; the callback and the timeout both CAS on `version` |
| Import | `06-import.md` IMP-093 |
| DDL transfer (P3) | One long-running job per transfer with heartbeat; the reaper returns a dead node's job and another node resumes with `Range` |
| Seeding tick, housekeeping, blocklist and content-list refresh, qBittorrent glob push | System schedule rows with dedupe `WhilePendingOrRunning` |
| Backlog dispatch | One schedule row per BacklogRule, plus the lane dispatcher job (dedupe `WhilePendingOrRunning`) |
| Push-hint endpoint, queue API, SSE | Any API node |

_Source: A1; D20 (D14 amended); design/downloads.md §3.13; design/runtime.md §3.4–§3.5._

**DLD-120** (MUST) A poll job reclaimed after a node failure **MUST** resume from the persisted observation and clocks; the Δ cap in DLD-051 **MUST** prevent the failover gap from counting. Every write caused by a job **MUST** be fenced by the job's `(id, attempt)` (`12-platform.md`). _Source: design/downloads.md §3.13; design/runtime.md N1._

**DLD-121** (MUST) LISTEN/NOTIFY **MUST** be used only to wake the scheduler and invalidate UI state; correctness **MUST NOT** depend on a notification being delivered. _Source: A4._

### 18. Events and metrics

**DLD-122** (MUST) The download lifecycle **MUST** publish these events through the outbox (`12-platform.md`): `download.grabbed`, `download.progress` (throttled, SSE only), `download.attention{condition}`, `download.failed{reason}`, `download.extraction_started`, `download.extraction_completed`, `download.extraction_failed`, `download.processing_requested`, `download.processing_timed_out`, `download.manual_interaction_required`, `download.imported`, `download.seeding_stopped`, `download.removed{data_deleted}`, `download.unlinked`, `download.unregistered`, `download.orphan_quarantined`, `search.intent_dropped{recurrence}`. Every payload **MUST** include the targets as `(item, slot)` pairs. _Source: design/downloads.md §3.14; D7._

**DLD-123** (MUST) Prometheus metrics **MUST** include: downloads by phase and retention; policy firings by condition and action; time in phase (histograms); extraction bytes and duration; processing-hold duration and outcome; poll latency and errors per client; seeding ratio distribution; orphan bytes quarantined; searches per lane, grabs per search and upgrades per cycle. _Source: design/downloads.md §3.14, §3.12; D7; A9._

## Data model

Shared entities (`Item`, `SlotDefinition`, `TitleSlot`, `ItemSlot`, `MediaFile`, `Tag`) are owned by `02-domain-model.md`, `Indexer` by `03-acquisition.md` and `User` by `10-identity-and-access.md`. `Download`, `DownloadTarget` and `Blocklist` are owned by this file (as `02-domain-model.md` §3.3 states); a Download has no single slot column (DLD-047).

```go
DownloadClient {
  name string unique; kind enum(qbittorrent, transmission, deluge, rtorrent, sabnzbd, nzbget,
                                blackhole_torrent, blackhole_usenet, aria2, flood, ddl)
  protocol enum(torrent, usenet, ddl); enabled bool; priority int default 1
  settings json            // host, credentials (secret fields), base category, per-kind categories, remove_completed/remove_failed
  labels json              // which lifecycle labels to mirror
  extraction enum(auto, always, never) default auto
  poll_fast_s, poll_slow_s, poll_seeding_s int?
  round_robin json         // DB cursor per priority group
  glob_push_hash string?
  edges: path_mappings -> RemotePathMapping; tags M2M Tag; schedule -> Schedule (dlclient.poll:<id>)
}
RemotePathMapping { client -> DownloadClient; remote_prefix, local_prefix string; unique(client_id, remote_prefix) }

Download {
  id uuid
  client -> DownloadClient; client_ref string?; marker string
  protocol enum; kind enum(library, racing)                  // racing: later phase, no targets
  // release snapshot, immutable after grab
  indexer -> Indexer?; guid, title string; size int64; info_hash string?; private bool
  indexer_flags json; release_digest json                    // parsed quality + CF digest from evaluation
  source enum(rss, search, interactive, irc, backlog, redownload, user_push)
  seed_obligation json      // {mode, min_ratio, min_seconds}
  grabbed_by -> User?; grabbed_at time
  // phase axis
  phase enum(submitting, downloading, extracting, processing, importing, import_blocked, imported, failed, abandoned)
  phase_at time; version int; reason enum?; reason_detail string?
  // retention axis
  retention enum(none, active, seeding, stopped, removed, vanished); retention_at time; data_deleted bool
  stopped_by_policy bool; library_refs int; hardlink_refs int; unlinked_at time?
  // latest observation
  obs json; observed_at time?; output_path string?
  // clocks (seconds accumulated while observable)
  no_progress_s int; progress_mark_bytes int64; metadata_s, slow_s, slow_eta_s, unregistered_s, no_seeders_s, inactive_s int
  absent_polls int
  // extraction, processing, import
  extract_state enum(pending, running, done, failed)?; staging_path string?
  processing_rule -> ProcessingRule?; processing_deadline time?
  extract_attempts, import_attempts int; next_attempt_at time?
  blocked_since time?; attention_notified bool; overrides []string
  edges: targets -> DownloadTarget; files -> DownloadFile; links -> DownloadLink; events -> DownloadEvent
  index: unique(client_id, client_ref); (info_hash); (phase); (retention)
}
DownloadTarget {
  download -> Download; item -> Item; slot -> SlotDefinition
  outcome enum(pending, imported, not_wanted, rejected, skipped, superseded); reason string?; media_file -> MediaFile?
  unique(download_id, item_id, slot_id); index(item_id, slot_id, outcome)
}
DownloadFile { download -> Download; idx int; path string; size int64
               verdict enum(allow, skip, deny); class enum(media, extra, archive, sample, executable, dangerous, other)
               processing_result enum(pending, ready, failed)? }
DownloadLink { download -> Download; media_file -> MediaFile; source_rel_path string
               kind enum(hardlink, reflink, copy, move); created_at time; unique(download_id, media_file_id) }
DownloadEvent { download -> Download; at time; type string; reason string?; detail string?
                actor enum(system, user) + user -> User?; data json }

Blocklist {
  info_hash string?; indexer -> Indexer?; guid string?; title string; size int64?; protocol enum
  reason enum; detail string?; download -> Download?; scope json   // container + targets, display
  created_at time; expires_at time?                                 // null = permanent
  index: (info_hash); (indexer_id, guid); (title, size)
}

DownloadPolicy { name; enabled; priority int
  condition enum(no_progress, metadata, slow_speed, slow_eta, unregistered, no_seeders, max_age, import_blocked, extract_failed)
  scope json      // protocols, privacy, indexer_ids, client_ids, tags, media_kinds, completion_min/max, min/max_size
  after_seconds int; params json   // min_progress_bytes, min_speed, window_seconds, max_eta, ignore_when_alt_speed, reason_class
  action enum(fail, notify, stop, relabel); action_params json  // blocklist, search, client_disposition, label
  simulate bool }
ImportReasonPolicy { reason_class enum(content_blocked, permanent, overridable, transient, needs_user, not_wanted, partial)
  code string?    // override for one code
  action enum(fail, accept, retry, wait_user, complete_without_import)
  max_attempts int?; fail_after_seconds int?; accept_partial_after_seconds int? }
FileContentPolicy { media_kind enum; mode enum(denylist, allowlist); rules json; source_url string?; refresh_seconds int
  on_blocked enum(skip_files, fail_if_all, fail_if_any); push_to_client bool default false }
ProcessingRule { name; enabled; priority int; scope json   // media_kinds, slot_ids, tags, indexer_ids, client_ids, protocols
  timeout_seconds int default 21600
  on_timeout enum(import_blocked, fail, import_unprocessed) default import_blocked
  on_failed enum(import_blocked, fail) default import_blocked }
SeedingRule { name; priority; enabled
  scope json      // indexer_ids, tracker_domains, privacy, media_kinds, client_ids, tags, phases
  max_ratio float?; min_seed_seconds, max_seed_seconds, max_inactive_seconds, min_seeders int?
  when_unlinked enum(ignore, remove_after_obligation, remove_now); unlinked_mode enum(provenance, hardlink)
  unregistered_after_seconds int default 3600; action enum(stop, remove, remove_data) }

BacklogRule { name; enabled; filter json (DSL); selector enum; batch int; interval_s int; min_cycle_s int
  post_release_grace_s int; active_download_cap int; schedule -> Schedule }
SearchIntent { item -> Item; slot -> SlotDefinition; lane enum(user, redownload, added, backlog); reason string
  not_before time?; dedupe_key string unique; created_by -> User?
  state enum(queued, dispatched, done, dropped); search_id ulid?;   // 03-acquisition.md Release.search_id outcome json }
SearchLedger { item -> Item; slot -> SlotDefinition; cycle_id int; last_searched_at time; last_outcome string
  unique(item_id, slot_id) }
```

## Flows

### F1. Public torrent stalls at 40%

1. Evaluation reserves a grab for `(S02E05, slot hd)`. In the same transaction: `Download{phase=submitting}`, one target, a `download.submit` job (DLD-008).
2. The submit job fetches the `.torrent`, stores hash H as marker, passes the content check, selects qBittorrent, calls `Add` (category `bonnyread-tv`), CASes to `downloading`, `retention=active`, and pokes the client's poll schedule.
3. The poll job (fast interval) sees growth ≥ 50 MB per poll, so `no_progress_s` stays 0. The swarm dies at 4 GB; each poll adds Δ.
4. The NAS reboots for 20 min: the client is `degraded`, Δ = 0 (DLD-053). The user pauses overnight: clocks freeze.
5. `no_progress_s` reaches 6 h; the evaluator returns `fail{blocklist, search, remove_data}`.
6. The failure handler, in one transaction: CAS to `failed(no_progress)`; Blocklist `{info_hash=H, expires_at=+30 d}`; `client.remove(H, deleteData)` job; `SearchIntent{redownload, S02E05, hd}`; `download.failed`.
7. The redownload dispatcher checks the recurrence guard (1 failure in 24 h), the rate and the budget, then dispatches. Evaluation rejects H by the blocklist and grabs the next-best release.

*Failure path (private):* threshold 24 h, disposition `keep`; the item stays in the client with `phase=failed` and the seeding manager owns it.

### F2. One release, two slots, RAR'd torrent (D20)

1. Evaluation selects `Movie.2023.2160p.UHD.BluRay.REMUX-GRP` for slots `uhd` and `best` of movie M, and commits one grab with targets `{(M, uhd), (M, best)}` (DLD-010).
2. The submit job adds one torrent. Completion settles; `FindCompressedFiles` finds `movie.rar`; CAS to `extracting`; retention becomes `seeding`.
3. The extract job wipes staging, checks space, extracts into `<uuid>.partial/`, renames to `<uuid>/`, and CASes to `importing` (no processing rule matches).
4. The import job receives `ImportRequest{targets=[(M,uhd),(M,best)], payload=owned}`. The importer places a reflink (or, if unsupported, a copy) into the `best` root and moves the staged file into the `uhd` root (`06-import.md` IMP-052). Two MediaFiles, two `file.imported` events.
5. The seeding manager inserts two DownloadLinks (`reflink`, `move`): `library_refs=2`, `hardlink_refs=1` if the reflink was used. Both targets `imported`; CAS to `imported`.
6. Later, slot `best` is upgraded: one link is deleted, `library_refs=1`; slot `uhd` is unaffected (DLD-050).

*Failure path:* the probe shows the file is 1080p mislabelled. `uhd` rejects with `NoSlotAccepts`, `best` imports. The outcome is partial: `import_blocked(partial)` and a notification (DLD-087).

### F3. External processing hold

1. A ProcessingRule "remux movies" matches movie downloads. An SABnzbd download completes; no extraction is needed.
2. CAS `downloading → processing`. The media file is reflinked (or copied) into `<staging>/<uuid>/`; `download.processing_requested` is published with the working directory and a deadline of +6 h.
3. The user's tool remuxes `movie.mkv` in place and calls `POST /api/v1/downloads/{id}/processing {result: ready}`.
4. CAS `processing → importing`; the importer discovers the working directory afresh and moves the file (payload `owned`).

*Failure paths:* the tool reports `failed` → `import_blocked(processing_failed)` + notify; no callback by the deadline → `import_blocked(processing_timeout)` + notify; a callback after the download left `processing` → `409`.

### F4. Private torrent unlinked by an upgrade

1. The torrent is imported by hardlink: `library_refs=1`, `hardlink_refs=1`; obligation ratio 1.0 or 72 h.
2. On day 2 an upgrade retires the file; `file.upgraded` removes the link: `library_refs=0`, `hardlink_refs=0`, `unlinked_at=now`; an evaluation runs: obligation not met, nothing happens.
3. At 72 h the obligation is met, the unlinked goal holds, the swarm guard passes; action `remove_data`. The cross-seed check finds another torrent sharing the content path, so the action is demoted to `remove` (DLD-100).

### F5. Usenet import rejected for a typed reason

1. SABnzbd unpacks; the observation is `Completed`; `payload=client_done`; import job.
2. The importer returns request-scope `NoMediaFiles` (only a 2 MB sample) and file-scope `ExecutableFile`.
3. Reason class `content_blocked` → fail + blocklist `(indexer, guid)` + `client.remove(deleteData)` + redownload intent. Nothing matches message text.

*Variants:* `SampleIndeterminate` → overridable → re-import with override; `AmbiguousMatch` → `import_blocked` + one notification; `MinimumFreeSpace` → backoff in `importing`, after 24 h `import_blocked(transient_exhausted)`.

## Non-goals

| Non-goal | Reason |
|---|---|
| Unpackerr's standalone watch-folder extraction (folders not tied to a download) | No library outcome; the extraction pool could serve it later as an optional feature. [OPN-013] |
| Strikes, strike decay and connectivity gates | Replaced by condition clocks with Δ = 0 while degraded (D19). |
| Client categories as a state machine (`cleanuparr-unlinked` etc.) | State is in the database; labels are cosmetic mirrors only. |
| Inode link-count scans to detect unlinked torrents | Replaced by library events and `DownloadLink`. |
| Title-keyed tracking, "left the queue ⇒ imported", delete delays | The Download row and the staging root make them unnecessary. |
| Force-import via text matching of status messages | Replaced by typed reason classes and the `overridable` class. |
| `DownloadedEpisodesScan` and other compatibility endpoints | D7. |
| Removing only when the client paused at ratio | The seeding manager decides for every client. |
| One slot per download (grab the same release twice) | Superseded by D20. |
| Window-based obligations ("seed X h within Y days") | Not expressible by current indexer definitions; may be added as an `obligation_mode` later. |

## Traceability

| Reference | Preserved | Changed deliberately (D19) |
|---|---|---|
| Sonarr/Radarr/Lidarr/Bookshelf/Whisparr download handling | Queue with progress and targets (O1); auto-import (O2); visible block with one notification (O3); client failure → fail + blocklist + redownload (O4); client selection by protocol, tags, priority (O15); remote path mappings (O16); manual fail/remove/ignore (O17); blackhole (O19); `checkingUP` not complete | Durable Download row keyed by marker instead of in-memory tracked downloads rebuilt by title parsing; typed reasons instead of status strings; authoritative grab targets; multi-slot downloads (D20); throttled redownload with recurrence guard |
| Whisparr v3 | Fallback to the next client on failure | — |
| Cleanuparr | Stall, metadata, slow rules; malware/content lists (importable format); failed-import handling; seeding rules; unlinked, dead and unregistered detection; orphan quarantine; simulate mode; private-tracker protections; collapsed-list guard | Condition clocks instead of strikes; one policy table with capabilities instead of per-client tables; in-process reason policy instead of text matching and force-import round trips; event-driven links instead of inode scans |
| Unpackerr | Multi-volume RAR/7z/zip extraction via xtractr with passwords and limits; seeding archives untouched | Staging root on the library filesystem, importer moves the output; no delete delay, title map or history restore; watch-folder mode dropped |
| autobrr | go-* client libraries; reannounce loop for unregistered private torrents | — |
| Kapowarr | DDL queue with host quota handling (P3) | Quota pauses queued work instead of dropping it; hosts resolve, core streams (A11) |
| Profilarr upgrades | Filter DSL, selectors, round-robin | Cycles in `SearchLedger` instead of cooldown tags |
| Prowlarr | Per-indexer seed ratio, seed time, pack seed time | Captured at grab into the Download |
