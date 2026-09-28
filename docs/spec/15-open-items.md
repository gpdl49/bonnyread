# 15. Open items

This file lists every choice in the specification that is written as a requirement but is not yet confirmed. Each entry carries the question, the proposed default that the rest of the specification uses, the design section it comes from, and the spec files that depend on it. Entries come from three sources, in three number blocks: OPN-001–072 are the Group C items of `docs/design/open-issues.md` §C, numbered in area order (acquisition, downloads, evaluation, identity/API, import, library, metadata/subtitles, profiles, runtime); OPN-101–199 are items the spec writers raised while writing the owning files (§10); OPN-201–299 are findings of the cross-file consistency review that need a product decision (section "Review findings needing a decision", at the end of this file). OPN-073–076 are the review and closure rules (§11). Other files cite an entry as `[OPN-NNN]`. Applies to P1, P2 and P3; each entry states its phase.

## Index

Machine-readable index. Numbers are stable: an entry is never renumbered or reused. A closed entry keeps its row with its outcome (OPN-075).

| ID | Short title | Default |
|---|---|---|
| OPN-001 | Primary Cardigann definition feed | Prowlarr `definitions/v11` from GitHub, pinned by commit; Jackett overlay opt-in, later |
| OPN-002 | Embedded IRC definition baseline | Embed an autobrr IRC baseline (GPL-2.0-or-later); fetch updates at runtime |
| OPN-003 | TrackerAlias map curation | Curated in-repo data file with CI checks against upstream id lists |
| OPN-004 | Indexer budget default values | Backlog share 50%, RSS reserve, 10 min query memo, 60 min interactive TTL, IRC-healthy RSS ×4; validate in beta |
| OPN-005 | Prowlarr/Jackett/autobrr config importers | Not in P1; later phase |
| OPN-006 | Seed policy and client routing ownership | Download lifecycle owns enforcement and routing; indexer supplies hints |
| OPN-007 | DDL site definitions format | Extend the Cardigann engine with an `article.groups` block |
| OPN-008 | Racing grabs and indexer budgets | Racing grabs count against the same per-indexer budgets |
| OPN-009 | Superseded in-flight download handling | Cancel on public trackers; keep (seed) on private trackers |
| OPN-010 | Partial pack import default | Block import (`import_blocked(partial)`) until the user acts; policy is a setting |
| OPN-011 | Seed obligation mode default | "ratio OR time", populated from definitions where available |
| OPN-012 | Download policy threshold defaults | 1 h metadata, no progress 6 h public / 24 h private, 1 h unregistered, 3 failures/24 h, 2 redownloads/min; tune in beta |
| OPN-013 | Unpackerr watch-folder mode | Dropped |
| OPN-014 | Pushing seed limits to download clients | Off by default |
| OPN-015 | Adopting unmanaged client items | In P1 only if cheap; otherwise manual import from a path |
| OPN-016 | Blocklist entry expiry | Permanent; 30-day expiry for `no_progress` on public torrents |
| OPN-017 | Library slot vs racing rule on one announce | The racing rule skips the announce |
| OPN-018 | Release enrichment contract | Agreed contract between acquisition and evaluation, fixed in the spec |
| OPN-019 | D8 spike attribute-agreement metric | Spike reports per-kind attribute agreement as well as structure pass rates |
| OPN-020 | Passkey (WebAuthn) login | Later, not v1 |
| OPN-021 | Requester quota default | 10 movies and 10 seasons per 7 days |
| OPN-022 | Mandatory TOTP for Owner/Admin | Mandatory unless the server is loopback-only |
| OPN-023 | macOS service mode default | Per-user LaunchAgent; LaunchDaemon `.pkg` for server mode |
| OPN-024 | OIDC back-channel logout | Not in v1 |
| OPN-025 | API key default expiry | 1 year; "never" allowed for service accounts |
| OPN-026 | SSE event replay window | 10 minutes or 50,000 events, whichever retains more |
| OPN-027 | OpenAPI server generator choice | oapi-codegen strict server, confirmed by a spike first |
| OPN-028 | Calibre integration role | Downstream target only |
| OPN-029 | Book identification match threshold | 0.30 distance, with the ambiguity margin; validate on fixtures |
| OPN-030 | Auto-add unknown artists/authors on scan | Per-root option, off by default |
| OPN-031 | Audiobookshelf and Stash metadata writers | Later phase |
| OPN-032 | EPUB metadata tag writing | Opt-in |
| OPN-033 | Missing-file grace period | 7 days |
| OPN-034 | Hash verification of cross-filesystem copies | On for network roots, off otherwise |
| OPN-035 | Episode-title-required import hold | Keep Sonarr's behaviour (Always / BulkSeasonReleases / Never) with a timeout |
| OPN-036 | Library-import rename default | Keep existing paths |
| OPN-037 | Comic file transforms default | Opt-in |
| OPN-038 | Collected editions satisfying singles | Opt-in per title slot; off by default |
| OPN-039 | Requester availability notification trigger | Notify on "complete"; "caught up" optional |
| OPN-040 | Import-list retraction file deletion | Delete files only for titles added solely by that list |
| OPN-041 | Import-list manual approval | Allowed (`approval=manual`) |
| OPN-042 | Metadata profile filter scope | Filters membership and provider fetches where supported |
| OPN-043 | Follow "new members" cut-off semantics | Fixed `from_date`, default the follow's creation time |
| OPN-044 | Alternate TV episode orderings | After v1; TVDB aired order only |
| OPN-045 | Unknown-title disk auto-add per kind | Available for every kind, off by default |
| OPN-046 | Slot inference on a shared root folder | Infer the slot by profile fit |
| OPN-047 | Specials inclusion scope | `include_specials` per title and per request |
| OPN-048 | IMDb and Rotten Tomatoes ratings | Dropped in v1 |
| OPN-049 | Subtitle profile granularity | One subtitle profile per item, all slots |
| OPN-050 | Subtitles on media upgrade | Drop and re-search downloaded; re-sync uploaded and manual |
| OPN-051 | Default subtitle sync tool | alass |
| OPN-052 | subzero OCR dictionary licence | Check provenance before porting OCR fixes |
| OPN-053 | Metadata change-feed spikes | Run spikes; non-blocking, age-based refresh fallback |
| OPN-054 | Hardcover OAuth client registration | Project registers one (device grant) |
| OPN-055 | Plug-in ABI payload encoding | JSON in ABI v1 |
| OPN-056 | Upstream deletion of an in-use profile | Detach to a local copy |
| OPN-057 | PCD publishing workflow | Direct push; PR flow later |
| OPN-058 | go-git push from shallow clone | Spike before relying on it |
| OPN-059 | Ebook and audiobook score domains | Separate domains |
| OPN-060 | PCD `minimum_version` handling | Map to a PCD compatibility level |
| OPN-061 | Profilarr user-ops importer | In P1 if cheap |
| OPN-062 | Bundled Dictionarry snapshot | Bundle for offline first boot |
| OPN-063 | Recyclarr configuration translator | Later |
| OPN-064 | Condition-level rename journaling | Later; aggregate-level only |
| OPN-065 | Plug-in runtime: Extism vs custom ABI | Decide by spike |
| OPN-066 | Plug-in signing trust root | Admin-managed trust keys |
| OPN-067 | Declarative entity provisioning | Later |
| OPN-068 | Cluster log viewer scope | Per-node ring buffer plus per-job logs |
| OPN-069 | Completed job visibility | 7 days |
| OPN-070 | wazero context-close timeout cost | Measure in the plug-in spike |
| OPN-071 | Minimum PostgreSQL version and PgBouncer | PostgreSQL 15; PgBouncer session mode only |
| OPN-072 | SQLite to PostgreSQL `db copy` path | Supported and tested upgrade path |
| OPN-101 | Cardigann regex timeout default | 100 ms per regexp2 match |
| OPN-102 | Invalid file in a new definition set | File stays inactive; indexers keep the last valid body; health warning |
| OPN-103 | Cross-node login/solver single-flight mechanism | Row claim by conditional update (ACQ-120, ACQ-124, PLT-062), 60 s hold, solver keyed `(host, egress)`; cross-file conflict resolved |
| OPN-104 | Shared payload cache storage | In the database, 5 min, keyed by (indexer, release key) |
| OPN-105 | Racing client-side guards placement | Specified with racing mode; placement undecided |
| OPN-106 | Superseded targets on multi-slot downloads | Per-target supersession; abandon only when no pending targets remain |
| OPN-107 | Processing hold failure default | `import_blocked(processing_failed)`, notify, no blocklist |
| OPN-108 | Processing hold timeout default | 6 h, then `import_blocked(processing_timeout)` and notify |
| OPN-109 | Unparseable content reason class | `needs_user`; per-code override allowed |
| OPN-110 | Blocklist matching scope | Global by release identity; scope for display only |
| OPN-111 | D8 spike decision criteria | 100% reachable; `rlshybrid` only if no axis worse; else `cascade` |
| OPN-112 | Season pack upgrade policy location | `All`; per-kind setting with per-TitleSlot override |
| OPN-113 | Delay-profile resolution order | TitleSlot → SlotDefinition → tag-scoped → profile's delay profile → none |
| OPN-114 | Coverage gain weighting | Missing and upgradable Items weigh equally |
| OPN-115 | Coverage selection search budget | 10,000 nodes, then deterministic greedy |
| OPN-116 | Late target attach to an existing download | Attach while `submitting`, `downloading` or `extracting` |
| OPN-117 | Snapshot staleness bound | Generation check at most every 5 s |
| OPN-118 | Request quota units per kind | DOM-070 units (TV seasons, movie 1, music albums, book books, adult items, comic volumes); withdrawn and declined requests do not count; IAM-080 aligned |
| OPN-119 | Forward-auth header details | `Remote-User`/`-Groups`/`-Email`/`-Name`; JIT provisioning off |
| OPN-120 | Local OS identity binding for pairing | `local_os` link; `allowLocalPairing` on by default |
| OPN-121 | Audio tag library binding | TagLib; binding chosen by spike |
| OPN-122 | Identification ambiguity margins | Distance difference ≤ 0.05; same level for graded matching |
| OPN-123 | Naming negative truncation | `:-N` excluded until settled against Sonarr tests |
| OPN-124 | Slot name naming token | `{Slot Name}` in the common token set |
| OPN-125 | Recycle bin retention default | 7 days |
| OPN-126 | Merge fold collision rule | Older row survives; union of monitoring; coverage recomputed |
| OPN-127 | Default slot definitions on first boot | One per kind: `tv/HD`, `movie/HD`, `music/Lossless`, `book/Ebook`, `adult/HD`, `comic/Digital` |
| OPN-128 | Hand-monitored specials without `include_specials` | Grabbable by RSS and interactive search; not missing, not backlog-searched |
| OPN-129 | Auto-unmonitor on delete default | Off |
| OPN-130 | Movie availability delay scope | Global setting, default 0 days |
| OPN-131 | TMDb TV fallback episode-order mapping layer | Needed only if TVDB OSS tier refused; design open |
| OPN-132 | TMDb purge scope for titles and dates | Keep titles, original title, year, release dates |
| OPN-133 | Extension schema repository, naming and licence | PCD-style additive repo with `bx_` prefix; location, name, licence open |
| OPN-134 | Portable references in publishable autograb rules | Client binding and exec/webhook conditions local-only; indexers by tag or definition id |
| OPN-135 | Bootstrap config file format and location | TOML at `<DATA_DIR>/config.toml`, `--config` override |
| OPN-136 | Notification template language | Not chosen; no default |
| OPN-137 | Unauthenticated metrics listener | None; metrics require `metrics:read` |
| OPN-138 | Probe endpoint paths | `/livez`, `/readyz`, unauthenticated, main and admin listeners |
| OPN-201 | Owner of the credential store for external services | A platform-owned `service_credential` table |
| OPN-202 | Plug-in archive helper limits and captcha solver configuration | 50 MiB uncompressed, 1,000 entries; captcha solver as a platform entity |
| OPN-203 | Owner and schema of the shared item history | One `History` table owned by `02-domain-model.md` |
| OPN-204 | "Why not grabbed" API shape | `GET /api/v1/items/{id}/slots/{slotId}/decisions` |
| OPN-205 | Profile permission names vs the IAM vocabulary | Map to `profile:read`, `profile:update`, `profile:execute:publish`, `profile:manage` |
| OPN-206 | API id format vs int64 table keys | Opaque string ids in the API; tables keep their key types |
| OPN-207 | `grabbable` predicate | `monitored(i,s)`, ignoring `released`, `include_specials` and `search_hold` |
| OPN-208 | Event naming: Go type names vs notifiable ids | CamelCase Go types; dotted lowercase notifiable ids mapped per area |
| OPN-209 | Signal that reports a search's grabs | Evaluation emits `SearchEvaluated{search_id, grabbed}` |

## Concepts

- **Open item.** A choice that the specification needs and has written down, but that the user has not confirmed. It has a stable ID `OPN-NNN`.
- **Proposed default.** The value the specification uses until the item is closed. Requirements elsewhere are written with the default and tagged `[OPN-NNN]`.
- **Closure.** The user's confirmation or change of the default. Closure is recorded in this file; a changed default is propagated to every file in the entry's "Used by" list.
- **Spike.** A time-boxed experiment whose result decides the item. Spikes are scheduled in `14-roadmap.md`.

## Requirements

Each entry below is itself a requirement: the bold line states the default as the specification applies it. "Source" uses the design-doc section notation of `open-issues.md` (for example `acq Q1` = `design/acquisition.md` §6.2 Q1).

### 1. Acquisition

**OPN-001** (MUST) Primary Cardigann definition feed. The primary Cardigann feed is the Prowlarr/Indexers repository, `definitions/v11`, fetched from GitHub as a commit tarball and pinned by commit. A Jackett feed is an opt-in overlay that is not delivered in P1.
- Question: is the Jackett overlay (52 differing files, 7 Jackett-only ids) worth supporting, and when? Is `master` `definitions/v11` what `indexers.prowlarr.com` serves?
- Phase: P1 (Prowlarr feed); overlay later.
- Source: design/acquisition.md §3.12, §6.1 I2, §6.2 Q1.
- Used by: 03-acquisition.md, 14-roadmap.md.

**OPN-002** (MUST) Embedded IRC definition baseline. The binary embeds a small baseline of autobrr IRC announce definitions (GPL-2.0-or-later, compatible under D5a) for offline first run and deterministic tests, and fetches newer definitions at runtime; fetched definitions take precedence.
- Question: embed a baseline, or fetch only?
- Phase: P1.
- Source: design/acquisition.md §3.8, §6.2 Q2.
- Used by: 03-acquisition.md, 14-roadmap.md.

**OPN-003** (MUST) TrackerAlias map curation. The `TrackerAlias` map (Cardigann id ↔ IRC identifier ↔ native id) is a data file curated in the bonnyread repository, with CI checks against the upstream Prowlarr and autobrr id lists.
- Question: who maintains the alias map, and how?
- Phase: P1.
- Source: design/acquisition.md §3.3, §6.2 Q3.
- Used by: 03-acquisition.md.

**OPN-004** (SHOULD) Indexer budget default values. Budget defaults are: backlog share 50% of each indexer's query budget; a reserved RSS share; a 10-minute query result memo; a 60-minute interactive search-result TTL; RSS interval multiplier ×4 while the indexer's IRC facet is healthy. The values are validated against real private-tracker limits (for example BTN's hourly call limit) before release.
- Question: are these defaults right for real tracker limits?
- Phase: P1.
- Source: design/acquisition.md §3.6, §3.9, §6.2 Q4.
- Used by: 03-acquisition.md, 14-roadmap.md.

**OPN-005** (MUST NOT) Prowlarr/Jackett/autobrr config importers. P1 **MUST NOT** include configuration importers for Prowlarr, Jackett or autobrr (indexers, credentials, IRC networks). If built later, an importer carries the Newznab `100000+` category derivation and the select-index/select-key setting mapping.
- Question: will there be migration importers, and in which phase?
- Phase: later (not P1).
- Source: design/acquisition.md §6.1 I5, §6.2 Q5; open-issues A16.
- Used by: 03-acquisition.md, 14-roadmap.md.

**OPN-006** (MUST) Seed policy and client routing ownership. The download lifecycle enforces seed obligations (including private-tracker safeguards) and makes the download-client routing decision. The indexer supplies a `TorrentPolicy` and an optional download-client pin as hints only.
- Question: does seeding and routing belong to downloads, with indexer hints?
- Phase: P1.
- Source: design/acquisition.md §6.2 Q6, Q7.
- Used by: 03-acquisition.md, 05-downloads.md.

**OPN-007** (MUST) DDL site definitions format. DDL site definitions are written in the Cardigann definition language extended with an `article.groups` block (a release as groups of mirror links). They are project-authored data and **MAY** be embedded.
- Question: extend Cardigann, or a separate DSL? May they be embedded?
- Phase: P3.
- Source: design/acquisition.md §3.14, §6.2 Q8; D11.
- Used by: 03-acquisition.md, 14-roadmap.md.

**OPN-008** (MUST) Racing grabs and indexer budgets. When racing mode exists, racing grabs count against the same per-indexer grab and query budgets as library grabs.
- Question: do racing grabs share per-indexer limits?
- Phase: later phase (racing mode, D16).
- Source: design/acquisition.md §6.2 Q9.
- Used by: 03-acquisition.md, 04-evaluation.md, 14-roadmap.md.

### 2. Downloads

**OPN-009** (MUST) Superseded in-flight download handling. When a better release is grabbed for an (item, slot) that has a lower release still downloading, the lower download is cancelled as `abandoned(superseded)` without blocklisting if it is Usenet, DDL or a public torrent; a private-tracker torrent is kept (disposition `keep`) for ratio.
- Question: cancel automatically, and does private vs public matter?
- Phase: P1.
- Source: design/downloads.md §6.2 Q2.
- Used by: 04-evaluation.md, 05-downloads.md.

**OPN-010** (MUST) Partial pack import default. A pack whose files cover only part of its targets goes to `import_blocked(partial)` until the user acts. The policy (including an `accept_partial_after` duration) is exposed as a setting, unset by default.
- Question: block (Sonarr behaviour) or auto-accept after 24 h?
- Phase: P1.
- Source: design/downloads.md §3.9.2, §6.2 Q3.
- Used by: 05-downloads.md.

**OPN-011** (MUST) Seed obligation mode default. The default seed obligation mode is "ratio OR time". A per-indexer `obligation_mode` is supported, and is populated from definition fields (`minimumratio`, `minimumseedtime`) where available.
- Question: which obligation semantics by default; auto-populate from definitions?
- Phase: P1.
- Source: design/downloads.md §3.10, §6.2 Q4.
- Used by: 05-downloads.md.

**OPN-012** (SHOULD) Download policy threshold defaults. Default policy thresholds are: 1 h waiting for metadata; without progress for 6 h (public torrent, minimum progress 50 MB) or 24 h (private torrent); 1 h unregistered; a recurrence guard of 3 failed downloads per (item, slot) in 24 h; at most 2 automatic redownloads per minute. They are tuned against beta telemetry.
- Question: are these defaults right?
- Phase: P1.
- Source: design/downloads.md §3.6, §6.2 Q5.
- Used by: 05-downloads.md, 14-roadmap.md.

**OPN-013** (MUST NOT) Unpackerr watch-folder mode. bonnyread **MUST NOT** provide extraction of arbitrary watch folders not tied to a Download.
- Question: drop Unpackerr's watch-folder mode?
- Phase: P1.
- Source: design/downloads.md §3.8, §6.2 Q6.
- Used by: 05-downloads.md.

**OPN-014** (MUST NOT) Pushing seed limits to download clients. By default bonnyread **MUST NOT** push seed limits to the download client; bonnyread decides when seeding stops. Pushing limits as a safety net is an opt-in per-client setting.
- Question: off (simpler) or on (protects ratio while bonnyread is down)?
- Phase: P1.
- Source: design/downloads.md §3.10, §6.2 Q7.
- Used by: 05-downloads.md.

**OPN-015** (MAY) Adopting unmanaged client items. P1 **MAY** include "adopt an unmanaged client item" (creating a Download for an item in bonnyread's category without its marker) if it is cheap; otherwise P1 covers the case with manual import from a path, and adoption follows later.
- Question: is adoption needed in P1?
- Phase: P1 or later.
- Source: design/downloads.md §3.5, §6.2 Q9.
- Used by: 05-downloads.md, 14-roadmap.md.

**OPN-016** (MUST) Blocklist entry expiry. Blocklist entries are permanent by default. An entry with reason `no_progress` for a public torrent expires after 30 days.
- Question: should some reasons expire?
- Phase: P1.
- Source: design/downloads.md §3.4 (Blocklist), §6.2 Q10.
- Used by: 00-overview.md, 04-evaluation.md, 05-downloads.md.

### 3. Evaluation

**OPN-017** (MUST) Library slot vs racing rule on one announce. When a library slot and a racing (autograb) rule both accept the same announce, the library slot grabs and the racing rule skips the announce.
- Question: skip, share one download (cross-seed), or grab twice?
- Phase: later phase (racing mode).
- Source: design/evaluation.md §3.18, §6 Q2.
- Used by: 04-evaluation.md, 14-roadmap.md.

**OPN-018** (MUST) Release enrichment contract. Acquisition's enrichment interface (size, uploader, record label and similar fields fetched through a tracker API or `.torrent` metainfo, and reuse of the fetched `.torrent` at grab time) is fixed by the contract in `03-acquisition.md` and consumed as specified in `04-evaluation.md`.
- Question: what is the agreed enrichment contract?
- Phase: P1 (API enrichment for IRC grows in P2).
- Source: design/evaluation.md §6 Q6; design/acquisition.md §3.4 (`Enricher`).
- Used by: 03-acquisition.md, 04-evaluation.md.

**OPN-019** (MUST) D8 spike attribute-agreement metric. The D8 parser spike reports per-media-kind attribute agreement (source, resolution, modifier, codec, HDR, language) between `rls` and the ported `QualityParser`/`LanguageParser` tests, in addition to structure pass rates.
- Question: what must the spike measure?
- Phase: P1 (pre-implementation).
- Source: design/evaluation.md §6 I1, Q8; D8; D20 (attribute authority).
- Used by: 04-evaluation.md, 14-roadmap.md.

### 4. Identity and API

**OPN-020** (MUST NOT) Passkey (WebAuthn) login. v1 **MUST NOT** depend on passkeys; WebAuthn as a login method and second factor comes later.
- Question: passkeys in v1 or later?
- Phase: later.
- Source: design/identity-api-deployment.md §6 open questions.
- Used by: 10-identity-and-access.md, 14-roadmap.md.

**OPN-021** (MUST) Requester quota default. The default Requester quota is 10 movies and 10 seasons per rolling 7 days. Units per kind are the `quota_units` of `02-domain-model.md` DOM-070 (OPN-118).
- Question: default quota values, and unit counting per kind.
- Phase: P1 (movies, TV); P2/P3 kinds later.
- Source: design/identity-api-deployment.md §4.3, §6 open questions.
- Used by: 00-overview.md, 10-identity-and-access.md, 14-roadmap.md.

**OPN-022** (MUST) Mandatory TOTP for Owner/Admin. Users holding the Owner or Admin role must enrol TOTP whenever the server listens on any non-loopback address.
- Question: mandatory TOTP for privileged roles on exposed servers?
- Phase: P1.
- Source: design/identity-api-deployment.md §3.2, §6 open questions.
- Used by: 01-architecture.md, 10-identity-and-access.md.

**OPN-023** (MUST) macOS service mode default. On macOS the desktop installer installs the core as a per-user LaunchAgent by default; a system LaunchDaemon installed by a `.pkg` is offered for "server mode".
- Question: LaunchAgent (no admin, stops at logout) or LaunchDaemon (always on)?
- Phase: P1.
- Source: design/identity-api-deployment.md §3.13, §6 open questions.
- Used by: 01-architecture.md, 13-deployment.md, 14-roadmap.md.

**OPN-024** (MUST NOT) OIDC back-channel logout. v1 **MUST NOT** implement OIDC back-channel logout.
- Question: is back-channel logout needed in v1?
- Phase: later.
- Source: design/identity-api-deployment.md §6 open questions.
- Used by: 10-identity-and-access.md, 14-roadmap.md.

**OPN-025** (MUST) API key default expiry. New API keys expire after one year by default; "never" is allowed for service accounts.
- Question: one year, or no default expiry with a UI warning?
- Phase: P1.
- Source: design/identity-api-deployment.md §3.3, §6 open questions.
- Used by: 00-overview.md, 10-identity-and-access.md.

**OPN-026** (MUST) SSE event replay window. The SSE replay window (for `Last-Event-ID` resumption) is 10 minutes or 50,000 events, whichever retains more (as `12-platform.md` PLT-070 and `11-api-and-clients.md` API-065 specify); a resume outside it gets a "reset" signal and refetches.
- Question: the window size, sized with the event table.
- Phase: P1.
- Source: design/identity-api-deployment.md §3.7, §6 open questions; design/runtime.md §3.6.
- Used by: 11-api-and-clients.md, 12-platform.md, 14-roadmap.md.

**OPN-027** (MUST) OpenAPI server generator choice. The API is spec-first, generated with oapi-codegen (strict server), subject to a 3-endpoint spike (spec-first, validation, SSE documentation, TypeScript client) that runs before committing; Huma and ogen are the fallbacks.
- Question: oapi-codegen, Huma or ogen?
- Phase: P1 (pre-implementation).
- Source: design/identity-api-deployment.md §3.6, §6 open questions.
- Used by: 11-api-and-clients.md, 14-roadmap.md.

### 5. Import

**OPN-028** (MUST) Calibre integration role. Calibre is a downstream target only (copy into Calibre after import, or trigger its rescan). bonnyread never delegates file placement to Calibre.
- Question: keep Bookshelf's "Calibre owns placement" roots?
- Phase: P3.
- Source: design/import.md §6.2 Q1.
- Used by: 06-import.md, 14-roadmap.md.

**OPN-029** (SHOULD) Book identification match threshold. The default book match threshold is a distance of 0.30, combined with the per-kind ambiguity margin, validated on the Bookshelf fixtures.
- Question: 0.20 (Readarr), 0.50 (Bookshelf fork), or in between?
- Phase: P3.
- Source: design/import.md §6.1 item 3, §6.2 Q2.
- Used by: 04-evaluation.md, 06-import.md, 14-roadmap.md.

**OPN-030** (MUST) Auto-add unknown artists/authors on scan. Adding unknown artists or authors found during a disk scan (Lidarr's `addNewArtists`) is a per-root-folder option, off by default.
- Question: keep it, and at which default?
- Phase: P2 (music), P3 (books).
- Source: design/import.md §6.2 Q3.
- Used by: 06-import.md.

**OPN-031** (MUST NOT) Audiobookshelf and Stash metadata writers. The initial release of each kind **MUST NOT** depend on Audiobookshelf (`metadata.json`/OPF) or Stash metadata writers; they come in a later phase.
- Question: are these writers needed, and is a Stash path-scoped scan enough?
- Phase: later.
- Source: design/import.md §3.6, §6.2 Q4.
- Used by: 06-import.md, 14-roadmap.md.

**OPN-032** (MUST) EPUB metadata tag writing. Rewriting EPUB OPF metadata on import is opt-in (off by default), because it changes the file hash that reader sync (KOReader, Kobo) relies on.
- Question: on or opt-in?
- Phase: P3.
- Source: design/import.md §3.6, §6.2 Q5.
- Used by: 06-import.md.

**OPN-033** (MUST) Missing-file grace period. A MediaFile found missing on scan is marked missing and kept for 7 days before its row is deleted (`file.deleted{reason: missing_from_disk}`); a health alert fires if a whole root is empty or unreachable.
- Question: the grace period length.
- Phase: P1.
- Source: design/import.md §3.7, §6.2 Q6.
- Used by: 02-domain-model.md, 06-import.md.

**OPN-034** (MUST) Hash verification of cross-filesystem copies. Hash verification after a cross-filesystem copy is a per-root option, on by default for roots on a network filesystem and off otherwise; size verification always applies.
- Question: default-on for network roots, given the doubled read I/O?
- Phase: P1.
- Source: design/import.md §3.5, §6.2 Q7.
- Used by: 06-import.md.

**OPN-035** (MUST) Episode-title-required import hold. Sonarr's episode-title-required behaviour is kept: a transient `TitleRequired` rejection holds an import while the episode title is "TBA", with the setting Always / BulkSeasonReleases / Never, and a timeout after which the import proceeds. The timeout value is set in `06-import.md`.
- Question: keep it, and with a timeout?
- Phase: P1.
- Source: design/import.md §3.2 (rejection table), §6.2 Q8.
- Used by: 06-import.md.

**OPN-036** (MUST) Library-import rename default. Importing an existing library (unmanaged folders) keeps existing paths by default; renaming to the naming scheme is an explicit option.
- Question: keep paths or rename?
- Phase: P1.
- Source: design/import.md §4.4, §6.2 Q9.
- Used by: 06-import.md.

**OPN-037** (MUST) Comic file transforms default. Comic transforms (CBR→CBZ, range splitting, ComicInfo.xml) are opt-in, because they force a copy while seeding.
- Question: on by default for new downloads?
- Phase: P3.
- Source: design/import.md §3.6, §6.2 Q10.
- Used by: 06-import.md, 14-roadmap.md.

### 6. Library

**OPN-038** (MUST) Collected editions satisfying singles. A TPB or omnibus satisfying the single issues it collects is a per-title-slot opt-in, off by default (Kapowarr parity).
- Question: should collected editions satisfy the singles?
- Phase: P3.
- Source: design/library.md §4.4, §6.2 Q1.
- Used by: 02-domain-model.md, 04-evaluation.md, 06-import.md, 14-roadmap.md.

**OPN-039** (MUST) Requester availability notification trigger. Requesters are notified when a requested season or series is "complete" (every episode present, aired or not, Seerr parity). Notifying at "caught up" (all aired episodes present) is an option.
- Question: notify at complete, caught up, or both?
- Phase: P1.
- Source: design/library.md §3.8, §6.2 Q2.
- Used by: 02-domain-model.md.

**OPN-040** (MUST) Import-list retraction file deletion. When an item leaves an import list, the `remove_delete_files` retraction action may delete files only for titles whose only intent is that list; titles with any other intent are at most unmonitored.
- Question: keep file deletion (scoped), or cap retraction at unmonitor/remove-keep-files?
- Phase: P1.
- Source: design/library.md §3.7, §6.2 Q3.
- Used by: 02-domain-model.md.

**OPN-041** (MUST) Import-list manual approval. Import lists support `approval=manual`, under which list entries arrive as pending request intents.
- Question: allow manual approval on lists?
- Phase: P1.
- Source: design/library.md §3.7, §6.2 Q4.
- Used by: 02-domain-model.md.

**OPN-042** (MUST) Metadata profile filter scope. A metadata profile filters both catalog membership and provider fetches, where the provider supports filtered fetches (relevant to Hardcover and MusicBrainz quotas).
- Question: membership filter only, or also a fetch filter?
- Phase: P2 (music), P3 (books).
- Source: design/library.md §6.2 Q5.
- Used by: 02-domain-model.md, 08-metadata.md, 14-roadmap.md.

**OPN-043** (MUST) Follow "new members" cut-off semantics. A Follow's "new" members are those dated on or after a fixed `from_date`, defaulting to the Follow's creation time, replacing Lidarr's "newer than the newest existing album".
- Question: is the fixed from-date acceptable?
- Phase: P1 (model); P2 (music).
- Source: design/library.md §3.4, §6.2 Q6.
- Used by: 02-domain-model.md.

**OPN-044** (MUST NOT) Alternate TV episode orderings. v1 **MUST NOT** support per-title alternate TVDB orderings (DVD, absolute, alternate); TVDB aired (default) order is the only ordering.
- Question: support alternate orderings in v1?
- Phase: after v1.
- Source: design/library.md §6.1 I5, §6.2 Q7; design/metadata-and-subtitles.md §6.2 Q1.
- Used by: 02-domain-model.md, 08-metadata.md, 14-roadmap.md.

**OPN-045** (MUST) Unknown-title disk auto-add per kind. "Auto-add unknown titles found in a root folder" is available for every media kind, off by default.
- Question: every kind, or music and books only?
- Phase: each kind's phase.
- Source: design/library.md §3.10, §6.2 Q8.
- Used by: 02-domain-model.md, 06-import.md.

**OPN-046** (MUST) Slot inference on a shared root folder. When several slots share one root folder, a scanned file's slot is inferred by profile fit; roots are not required to belong to one slot. An unresolved tie raises `SlotAmbiguous`.
- Question: infer by profile fit, or require one slot per root?
- Phase: P1.
- Source: design/library.md §3.10, §6.2 Q9; design/import.md §6.1 item 2.
- Used by: 02-domain-model.md, 06-import.md.

**OPN-047** (MUST) Specials inclusion scope. `include_specials` is set per title and per request, not globally.
- Question: per title and request, or global (Seerr)?
- Phase: P1.
- Source: design/library.md §6.2 Q10.
- Used by: 02-domain-model.md, 04-evaluation.md.

### 7. Metadata and subtitles

**OPN-048** (MUST NOT) IMDb and Rotten Tomatoes ratings. v1 **MUST NOT** show IMDb or Rotten Tomatoes ratings (no legitimate direct source); IMDb non-commercial datasets as an opt-in download are not in v1.
- Question: drop, or offer IMDb datasets opt-in?
- Phase: P1.
- Source: design/metadata-and-subtitles.md §6.2 Q3.
- Used by: 08-metadata.md.

**OPN-049** (MUST) Subtitle profile granularity. Each item has one subtitle (language) profile, applied to all its slots. A per-slot override may be added later.
- Question: per item, or per slot?
- Phase: P2.
- Source: design/metadata-and-subtitles.md §3.B3, §6.2 Q4.
- Used by: 00-overview.md, 02-domain-model.md, 09-subtitles.md, 14-roadmap.md.

**OPN-050** (MUST) Subtitles on media upgrade. When a media file is replaced by an upgrade, subtitles downloaded for the replaced file are dropped and re-searched; `uploaded` and `manual` subtitles are kept and re-synced to the new file.
- Question: drop and re-search, or keep and re-sync when the release group matches?
- Phase: P2.
- Source: design/metadata-and-subtitles.md §3.B5, §3.B8, §6.2 Q5.
- Used by: 09-subtitles.md, 14-roadmap.md.

**OPN-051** (MUST) Default subtitle sync tool. alass is the default subtitle sync tool; ffsubsync is optional where installed.
- Question: alass or ffsubsync; is a comparison on a sample corpus needed?
- Phase: P2.
- Source: design/metadata-and-subtitles.md §3.B7, §6.2 Q6; open-issues A6.
- Used by: 09-subtitles.md, 14-roadmap.md.

**OPN-052** (MUST) subzero OCR dictionary licence. The subzero OCR-fix dictionaries **MUST NOT** be ported until their provenance and licence are verified; without a compatible licence the OCR-fix mod ships without them.
- Question: what licence do the dictionaries have?
- Phase: P2.
- Source: design/metadata-and-subtitles.md §3.B7, §6.2 Q7.
- Used by: 09-subtitles.md, 14-roadmap.md.

**OPN-053** (MUST) Metadata change-feed spikes. Spikes verify change-feed semantics for TVDB `/updates`, TMDb `/changes` (14-day window), MusicBrainz merged-MBID redirects, StashDB `updated_at`, ComicVine `date_last_updated` and Hardcover `updated_at`. They are non-blocking: where a feed is missing or unusable, the refresh engine uses age-based refresh policies.
- Question: which change feeds are usable?
- Phase: P1 (TVDB, TMDb); P2/P3 for the others.
- Source: design/metadata-and-subtitles.md §3.A8, §6.2 Q8.
- Used by: 08-metadata.md, 14-roadmap.md.

**OPN-054** (MUST) Hardcover OAuth client registration. The project registers a Hardcover OAuth client (device-authorization grant) and records the implications of Hardcover's clause on commercial use of user-owned data, which is in the legal review scope (D20).
- Question: register a project client; what does the user-data clause imply?
- Phase: P3.
- Source: design/metadata-and-subtitles.md §6.2 Q9; open-issues B12.
- Used by: 08-metadata.md, 14-roadmap.md.

**OPN-055** (MUST) Plug-in ABI payload encoding. Plug-in ABI v1 encodes host↔guest payloads as JSON. A binary encoding (for example CBOR) requires a new ABI version.
- Question: JSON or a binary encoding?
- Phase: P1 (ABI defined); P2 (first plug-ins: subtitle providers).
- Source: design/metadata-and-subtitles.md §3.B10, §6.2 Q10; design/runtime.md §3.12.
- Used by: 09-subtitles.md, 12-platform.md, 14-roadmap.md.

### 8. Profiles

**OPN-056** (MUST) Upstream deletion of an in-use profile. When a linked database's update deletes a profile (or other aggregate) that is in use, the entity is automatically detached into a local copy, the slots keep using it, and the user is notified; the update does not block.
- Question: detach automatically, or block until slots are re-pointed?
- Phase: P1.
- Source: design/profiles.md §3.4.4, §6.2 Q1.
- Used by: 07-profiles.md.

**OPN-057** (MUST) PCD publishing workflow. Publishing a database pushes directly to the configured branch. A "push to a new branch and open a pull request" flow (needing a GitHub/GitLab/Gitea API) is later.
- Question: direct push only, or also a PR flow?
- Phase: P1 (direct push); PR flow later.
- Source: design/profiles.md §3.5.3, §6.2 Q2.
- Used by: 07-profiles.md, 14-roadmap.md.

**OPN-058** (MUST) go-git push from shallow clone. Publishing from a depth-1 go-git clone is spiked before it is relied on. Fallback: deepen the fetch on publish only; the git CLI never becomes a runtime requirement (`07-profiles.md` PRF-007, PRF-064).
- Question: can go-git push from a shallow clone?
- Phase: P1 (pre-implementation).
- Source: design/profiles.md §3.5, §6.2 Q3.
- Used by: 07-profiles.md, 14-roadmap.md.

**OPN-059** (MUST) Ebook and audiobook score domains. Ebooks and audiobooks are separate score domains, so one book profile set can score the Ebook and Audiobook slots independently; the PCD `readarr` value maps to both.
- Question: one `readarr` domain or two?
- Phase: P3.
- Source: design/profiles.md §3.8.1, §6.2 Q4.
- Used by: 02-domain-model.md, 04-evaluation.md, 07-profiles.md, 14-roadmap.md.

**OPN-060** (MUST) PCD `minimum_version` handling. A database manifest's `profilarr.minimum_version` is informational and is mapped to a PCD compatibility level that bonnyread declares (level 2 initially); a manifest requiring a higher level (Profilarr major 3 or later) is treated like an unsupported schema major.
- Question: map it, or ignore it?
- Phase: P1.
- Source: design/profiles.md §3.7.3, §6.2 Q5.
- Used by: 07-profiles.md.

**OPN-061** (MAY) Profilarr user-ops importer. P1 **MAY** include a one-shot importer of Profilarr user ops (structured `desired_state` JSON) into local patches, if cheap. It is an importer, not a compatibility layer (D7).
- Question: in P1?
- Phase: P1 if cheap; otherwise later.
- Source: design/profiles.md §6.2 Q6.
- Used by: 07-profiles.md, 14-roadmap.md.

**OPN-062** (MUST) Bundled Dictionarry snapshot. Each release bundles a Dictionarry snapshot (MIT, notice kept) so first boot works offline; it is a database of `kind = bundled` that converts in place to the linked `git` database (same ids) on the first successful fetch. Refreshing the snapshot is a release checklist step.
- Question: is the snapshot worth the release chore?
- Phase: P1.
- Source: design/profiles.md §3.5.5, §6.2 Q7.
- Used by: 07-profiles.md, 14-roadmap.md.

**OPN-063** (MUST NOT) Recyclarr configuration translator. P1 **MUST NOT** include a `recyclarr.yml` translator (to trash-pcd plus local patches); it is a later-phase migration aid.
- Question: is a Recyclarr translator in scope?
- Phase: later.
- Source: design/profiles.md §6.2 Q8.
- Used by: 07-profiles.md, 14-roadmap.md.

**OPN-064** (MUST) Condition-level rename journaling. Upstream rename detection journals aggregate names only; a condition renamed inside a custom format is treated as a delete plus an add. Condition-level journaling is later.
- Question: is aggregate-level granularity enough?
- Phase: P1 (aggregate level); later (condition level).
- Source: design/profiles.md §3.3.2, §6.2 Q9.
- Used by: 07-profiles.md, 14-roadmap.md.

### 9. Runtime

**OPN-065** (MUST) Plug-in runtime: Extism vs custom ABI. The choice between the Extism Go SDK and a custom wazero ABI is made by a spike measuring per-call timeouts, memory caps and instance pooling. Until then the spec defines the host contract independently of either.
- Question: Extism or a custom ABI?
- Phase: P1 (pre-implementation).
- Source: design/runtime.md §3.12, §6 open questions 1.
- Used by: 09-subtitles.md, 12-platform.md, 14-roadmap.md.

**OPN-066** (MUST) Plug-in signing trust root. Plug-in packages are signed (minisign/ed25519); the trust root is admin-managed keys only, with an explicit "allow unsigned" toggle that is off by default.
- Question: project key, per-publisher keys, or admin-managed keys only?
- Phase: P1 (host); P2 (first plug-ins).
- Source: design/runtime.md §3.12, §6 open questions 1.
- Used by: 00-overview.md, 12-platform.md.

**OPN-067** (MUST NOT) Declarative entity provisioning. v1 **MUST NOT** include declarative (GitOps-style) provisioning of entities such as indexers, clients and notification targets; settings pinning covers settings only.
- Question: is Grafana-style provisioning needed?
- Phase: later.
- Source: design/runtime.md §3.10, §6 open questions 2.
- Used by: 12-platform.md, 13-deployment.md, 14-roadmap.md.

**OPN-068** (MUST) Cluster log viewer scope. The UI log viewer shows per-job run logs (persisted, cluster-wide) and the serving node's in-memory ring buffer (last 10,000 records at or above the configured level). Logs are not written to the main database.
- Question: is that enough, or should WARN+ go to a bounded DB table?
- Phase: P1.
- Source: design/runtime.md §3.11, §6 open questions 3.
- Used by: 12-platform.md.

**OPN-069** (MUST) Completed job visibility. Completed jobs stay visible in the UI's tasks and queue pages for 7 days, then are pruned; there is no separate summarised job-run history table.
- Question: retention period; separate history table?
- Phase: P1.
- Source: design/runtime.md §3.4, §6 open questions 5.
- Used by: 11-api-and-clients.md, 12-platform.md.

**OPN-070** (MUST) wazero context-close timeout cost. The plug-in spike measures the cost of wazero's `WithCloseOnContextDone` on hot plug-in paths (for example subtitle scoring). If it is significant, plug-in calls are made coarser-grained.
- Question: what does per-call timeout enforcement cost?
- Phase: P1 (pre-implementation).
- Source: design/runtime.md §3.12, §6 open questions 6.
- Used by: 12-platform.md, 14-roadmap.md.

**OPN-071** (MUST) Minimum PostgreSQL version and PgBouncer. The minimum PostgreSQL version is 15. PgBouncer is supported in session pooling mode only; transaction pooling is not supported.
- Question: minimum version; is transaction pooling supported at all?
- Phase: P1.
- Source: design/runtime.md §3.3, §6 open questions 7; open-issues A4.
- Used by: 00-overview.md, 01-architecture.md, 13-deployment.md.

**OPN-072** (MUST) SQLite to PostgreSQL `db copy` path. `bonnyread db copy` from SQLite to PostgreSQL is a supported upgrade path, covered by CI tests.
- Question: supported and tested, or best effort?
- Phase: P1.
- Source: design/runtime.md §3.1, §6 open questions 8; D18.
- Used by: 01-architecture.md, 12-platform.md, 13-deployment.md, 14-roadmap.md.

### 10. Items raised while writing the specification

These entries are not Group C items of `open-issues.md`; the spec writers raised them while writing the owning files. They are numbered from OPN-101 in area order; the numbers between the review rules and OPN-101 are left unused. "Source" names the spec requirements where the item was raised, since there is no design-doc question behind them.

#### 10.1 Acquisition

**OPN-101** (MUST) Cardigann regex timeout default. Every regexp2 match in the Cardigann engine runs with a timeout of 100 ms per match; a timeout fails that field extraction, increments `bonnyread_regex_timeouts_total{source="cardigann"}`, and drops a row whose required field timed out.
- Question: is 100 ms per match the right default for definition regexes?
- Phase: P1.
- Source: ACQ-034 (from D1 and design/evaluation.md §3.8).
- Used by: 03-acquisition.md.

**OPN-102** (MUST) Invalid file in a new definition set. A file that fails validation in a newly fetched definition set does not become active; indexers using that id stay on the previously active valid body, and a health warning names the id and the error.
- Question: keep the last valid body per file, or reject the whole set?
- Phase: P1.
- Source: ACQ-048 (derived from design/acquisition.md §3.12).
- Used by: 03-acquisition.md.

**OPN-103** (MUST) Cross-node single-flight mechanism for login and challenge solving. Tracker login is single-flight per indexer and challenge solving single-flight per `(host, egress)` across nodes, with a maximum hold of 60 s, and the holder re-reads shared state after acquiring. The mechanism is the platform's row claim (`12-platform.md` PLT-062): a conditional update of `lock_holder`/`lock_until` on the `IndexerSession` row (ACQ-120) or the `solver_clearance(host, egress_id)` row (ACQ-124, PLT-123). Database advisory locks remain reserved for the migration lock (PLT-008, design/runtime.md §3.5) and leases for long-lived connections. **Status:** the conflict between `03-acquisition.md` and `12-platform.md` is resolved in favour of the row claim (PLT-062 rewritten, 2026-09-28); the default awaits the user's confirmation.
- Question: confirm the row claim and the `(host, egress)` solver key.
- Phase: P1.
- Source: ACQ-120, ACQ-124, PLT-062 (design/acquisition.md §3.10, §3.15; design/runtime.md §3.5).
- Used by: 03-acquisition.md, 12-platform.md.

**OPN-104** (MUST) Shared payload cache storage. A fetched release payload (for example a `.torrent`) is kept for 5 minutes in a payload cache keyed by `(indexer, release key)` and stored in the database, so that an enrichment fetch followed by a grab, or a grab retried on another node, downloads it once.
- Question: database storage (visible to every node) or a per-node cache; is 5 minutes right?
- Phase: P1.
- Source: ACQ-134 (design/acquisition.md §3.8; design/evaluation.md §4.3).
- Used by: 03-acquisition.md.

**OPN-105** (SHOULD) Racing client-side guards placement. Racing-specific client-side guards (maximum active racing downloads, ignore when slow) are specified with racing mode; whether they belong to acquisition or to `05-downloads.md` is not decided. No default placement is proposed.
- Question: do racing guards live in acquisition or in the download lifecycle?
- Phase: later phase (racing mode, D16).
- Source: ACQ-152 (design/downloads.md §6 Q8).
- Used by: 03-acquisition.md.

#### 10.2 Downloads

**OPN-106** (MUST) Superseded targets on multi-slot downloads. Supersession is per target: when a new Download takes over a target `(item, slot)` still pending on an older Download in `submitting` or `downloading`, only that target is marked `superseded`; the older Download is abandoned (per OPN-009) only when it has no remaining `pending` targets. Downloads already in `extracting`, `processing` or `importing` are not superseded; the importer's upgrade check decides.
- Question: is per-target supersession, abandoning only when no pending targets remain, the right rule for multi-slot downloads?
- Phase: P1.
- Source: DLD-016 (design/downloads.md §6.2 Q2; D20).
- Used by: 05-downloads.md.

**OPN-107** (MUST) Processing hold failure default. When an external tool marks any held file `failed`, the processing rule's `on_failed` applies, defaulting to `import_blocked(processing_failed)` with a notification and no blocklist entry. A callback for a download not in `processing` returns `409 Conflict` without changing state.
- Question: is "block import and notify, no blocklist" the right default on a processing failure?
- Phase: P1.
- Source: DLD-079 (D20).
- Used by: 05-downloads.md.

**OPN-108** (MUST) Processing hold timeout default. A processing hold times out 6 h after `processing_at`; the default `on_timeout` is `import_blocked(processing_timeout)` with a notification (alternatives: `fail` without blocklist, `import_unprocessed`).
- Question: is 6 h with "block import and notify" the right default?
- Phase: P1.
- Source: DLD-080 (D20).
- Used by: 05-downloads.md.

**OPN-109** (MUST) Unparseable content reason class. The import rejection `Unparseable` maps to reason class `needs_user` by default; a user may override the class or action of any single code with an `ImportReasonPolicy` row.
- Question: is `needs_user` right for `Unparseable`, or should it be `permanent` or `content_blocked`?
- Phase: P1.
- Source: DLD-084 (design/downloads.md §3.9.2; design/import.md §3.2 stage 6).
- Used by: 05-downloads.md.

**OPN-110** (MUST) Blocklist matching scope. A blocklist row matches globally by release identity (info hash for torrents; `(indexer, guid)` for Usenet and DDL; `(title, size)` as fallback), not per Title; the row's recorded scope (container and targets) is for display only.
- Question: global by identity, or scoped to the container or Title the entry was created for?
- Phase: P1.
- Source: DLD-090, EVL-058 (design/downloads.md §1.2 I4, §3.4; design/evaluation.md §3.15).
- Used by: 00-overview.md, 04-evaluation.md, 05-downloads.md.

#### 10.3 Evaluation

**OPN-111** (MUST) D8 spike decision criteria. After the D8 parser spike, the implementation is chosen by these criteria in order: (1) it must be able to reach 100% of the structure and attribute corpora, with only upstream-filed expected differences; (2) `rlshybrid` is preferred only if its per-axis attribute agreement is not lower than `cascade`'s on any axis and its combined corpus pass rate is at least equal; (3) otherwise `cascade`. The unused implementation is removed.
- Question: are these the right decision criteria?
- Phase: P1 (pre-implementation).
- Source: EVL-017 (D8.3; D1a; D20).
- Used by: 04-evaluation.md.

**OPN-112** (MUST) Season pack upgrade policy location. The pack upgrade policy (`Any`, `All`, `Threshold n%`) defaults to `All` (Sonarr default) and is a per-kind media setting with a per-TitleSlot override.
- Question: where does the pack policy live (per kind, per profile, per slot), and is `All` the right default?
- Phase: P1.
- Source: EVL-071 (design/evaluation.md §1.6, §3.10, §6 Q5).
- Used by: 04-evaluation.md.

**OPN-113** (MUST) Delay-profile resolution order for slots and tags. The delay in effect for a slot resolves in this order: TitleSlot override → SlotDefinition override → the highest-priority tag-scoped DelayProfile of the profile's database whose tags intersect the title's tags → the quality profile's `delay_profile` → no delay. Autograb rules carry their own delay reference.
- Question: is this order right, in particular tag-scoped profiles ranking above the profile's own delay profile?
- Phase: P1.
- Source: EVL-073, PRF-073 (D20; open-issues B3; design/evaluation.md §3.14, §6 I4).
- Used by: 04-evaluation.md, 07-profiles.md.

**OPN-114** (MUST) Coverage gain weighting. The selection objective maximises the number of distinct wanted Items covered first, counting a missing Item and an upgradable Item equally, then compares RankKeys, then release ids.
- Question: should missing Items outweigh upgradable Items in the coverage term?
- Phase: P1.
- Source: EVL-085 (D20; open-issues B7).
- Used by: 04-evaluation.md.

**OPN-115** (MUST) Coverage selection search budget. The optimal selection is computed by exact search with a node budget of 10,000 nodes per component; when it is exhausted, a deterministic greedy fallback is used and counted in `bonnyread_evaluation_selection_fallback_total`.
- Question: is 10,000 nodes the right budget?
- Phase: P1.
- Source: EVL-086 (D20).
- Used by: 04-evaluation.md.

**OPN-116** (MUST) Late target attach to an existing download. A grab of a release whose identity matches a non-terminal Download for the same Title in `submitting`, `downloading` or `extracting` attaches new DownloadTargets to that Download; in any later phase a new grab is created.
- Question: is attaching up to and including `extracting` safe, or should attach stop earlier?
- Phase: P1.
- Source: EVL-091 (D20; ACQ-133).
- Used by: 04-evaluation.md.

**OPN-117** (MUST) Snapshot staleness bound. Each evaluating node compares its snapshot generations with the database at most every 5 s and reloads what is behind, in addition to event-driven updates.
- Question: is 5 s the right bound?
- Phase: P1.
- Source: EVL-102 (A4; D14; design/evaluation.md §3.19).
- Used by: 04-evaluation.md.

#### 10.4 Identity and API

**OPN-118** (MUST) Request quota units per kind. Each user request stores `quota_units` (`02-domain-model.md` DOM-070): TV counts requested seasons (all seasons for a whole-series request), movie counts 1, music counts albums, book counts books, adult counts items (scenes and adult movies), comic counts volumes. Quota usage is the sum of `quota_units` over pending and approved requests in the window (`10-identity-and-access.md` IAM-080, IAM-082); declined and withdrawn requests do not count. **Status:** IAM-080 now defers to DOM-070 (2026-09-28), so the files agree; the default awaits the user's confirmation.
- Question: the unit per kind for music, books, comics and adult; confirm that withdrawn requests do not count.
- Phase: P1 (TV, movie); P2/P3 kinds in their phase.
- Source: DOM-070, IAM-080 (design/identity-api-deployment.md §3.4; O15).
- Used by: 00-overview.md, 02-domain-model.md, 10-identity-and-access.md.

**OPN-119** (MUST) Forward-auth header details. Forward-auth header names are configurable, defaulting to `Remote-User`, `Remote-Groups` (comma-separated), `Remote-Email` and `Remote-Name`. The user header maps to a `forward_auth` identity, or by username when "match by username" is enabled; groups map through the OIDC group rules; JIT provisioning is off by default.
- Question: are these header defaults and the off-by-default JIT provisioning right?
- Phase: P1.
- Source: IAM-047.
- Used by: 10-identity-and-access.md.

**OPN-120** (MUST) Local OS identity binding for pairing. Later local pairing over the local endpoint returns a device credential when the peer OS principal is linked (`local_os`) to a human user whose `allowLocalPairing` setting is on (default on); otherwise the app signs in over loopback HTTP and may then offer to link the OS account. All local pairings are audited.
- Question: is binding an OS account to a user, with local pairing on by default, acceptable?
- Phase: P1.
- Source: API-097.
- Used by: 11-api-and-clients.md.

#### 10.5 Import

**OPN-121** (MUST) Audio tag library binding. Audio probing and tag writing use TagLib; the Go binding is chosen by a spike.
- Question: which TagLib binding (cgo, Wasm or other)?
- Phase: P2.
- Source: IMP-013 (design/import.md §3.2 stage 2; A6).
- Used by: 06-import.md.

**OPN-122** (MUST) Identification ambiguity margins. When the best two candidates resolve to different items within the kind's margin, the result is `Ambiguous` (`AmbiguousMatch`). Margins: a distance difference of at most 0.05 for distance kinds; the same match level for graded matching.
- Question: are these margins right per kind?
- Phase: P1 (TV, movie); later kinds in their phase.
- Source: IMP-026 (design/import.md §3.2 stage 4, §6.1 item 3; D10; A13).
- Used by: 06-import.md.

**OPN-123** (MUST) Naming negative truncation. The `:-N` token argument (truncate from the start) is not part of the naming grammar until its behaviour is settled against Sonarr's tests.
- Question: does `:-N` truncate from the start, and is it kept?
- Phase: P1.
- Source: IMP-061 (design/import.md §3.4 rendering rules).
- Used by: 06-import.md.

**OPN-124** (SHOULD) Slot name naming token. A `{Slot Name}` token is available in the common naming token set, so that slots sharing a root folder can be distinguished in paths.
- Question: add the token to the common set?
- Phase: P1.
- Source: IMP-066 (design/import.md §3.4 token sets; D10; D20).
- Used by: 06-import.md.

**OPN-125** (MUST) Recycle bin retention default. Recycle-bin entries are purged after 7 days by a scheduled job, which emits `recycle.purged` for each purged file.
- Question: is 7 days the right default retention?
- Phase: P1.
- Source: IMP-088 (design/import.md §3.5; design/downloads.md §3.10).
- Used by: 06-import.md.

#### 10.6 Library

**OPN-126** (MUST) Merge fold collision rule. When a merge fold collides on a unique key, the older row survives, its `ItemSlot.monitored` is `true` if either row was monitored, and its coverage is recomputed from the moved FileLinks.
- Question: is "older row survives, union of monitoring" the right reconciliation of design/library.md §3.6 and design/metadata-and-subtitles.md §A6?
- Phase: P1.
- Source: DOM-037.
- Used by: 02-domain-model.md, 08-metadata.md.

**OPN-127** (MUST) Default slot definitions on first boot. First boot creates one default slot per media kind: `tv/HD`, `movie/HD`, `music/Lossless`, `book/Ebook`, `adult/HD`, `comic/Digital`.
- Question: are these the right default slots and names?
- Phase: P1 (TV, movie); other kinds in their phase.
- Source: DOM-039 (D4; design/library.md §4 examples).
- Used by: 02-domain-model.md.

**OPN-128** (MUST) Hand-monitored specials without `include_specials`. A special monitored by hand on a title with `include_specials=false` is grabbable by RSS and interactive search, but does not appear in `missing` and is not backlog-searched.
- Question: should hand-monitored specials also be backlog-searched?
- Phase: P1.
- Source: DOM-054 (design/library.md §4.1, reconciled with O2).
- Used by: 02-domain-model.md.

**OPN-129** (MUST) Auto-unmonitor on delete default. The setting `library.unmonitor_on_delete` defaults to off; when on, deleting the last covering file of an ItemSlot unmonitors it, except for deletion reasons `upgrade`, `manual_override` and `missing_from_disk`.
- Question: off by default?
- Phase: P1.
- Source: DOM-085 (O8).
- Used by: 02-domain-model.md.

**OPN-130** (MUST) Movie availability delay scope. The movie availability delay is a global library setting `library.movie_availability_delay_days`, default 0.
- Question: global setting, or per movie/profile as in Radarr?
- Phase: P1.
- Source: DOM-089 (O3; Radarr `Movie.cs`).
- Used by: 02-domain-model.md.

#### 10.7 Metadata

**OPN-131** (MUST) TMDb TV fallback episode-order mapping layer. If the TVDB free/OSS tier is refused, TV falls back to TMDb as structure authority with an episode-order mapping layer producing TVDB-equivalent season, episode and absolute numbering (from TMDb episode groups, `external_ids.tvdb_id` and XEM's TVDB origin). The layer's design is not yet specified.
- Question: how is the mapping layer designed, and is it needed (TVDB application outcome)?
- Phase: P1 (contingency).
- Source: MET-018 (D20; open-issues B9; verification/metadata-services.md F2).
- Used by: 08-metadata.md.

**OPN-132** (MUST) TMDb purge scope for titles and dates. A TMDb data purge keeps the fields the library needs to operate (titles, original title, year, release dates) as well as ids.
- Question: do titles and dates count as purgeable TMDb data?
- Phase: P1.
- Source: MET-067 (D20; design/metadata-and-subtitles.md §6.1 item 3).
- Used by: 08-metadata.md.

#### 10.8 Profiles

**OPN-133** (MUST) Extension schema repository, naming and licence. The bonnyread extension schema is a PCD-style, semver-tagged schema repository, applied after the upstream schema as a second `pcd.json` dependency, strictly additive, with `bx_` table prefixes. Its repository location, name and licence are not yet chosen.
- Question: where does the repository live, what is it called, and under which licence?
- Phase: P1.
- Source: PRF-067 (D20; design/profiles.md §6.1 issue 1).
- Used by: 07-profiles.md.

**OPN-134** (MUST) Portable references in publishable autograb rules. In a publishable autograb rule, the download-client/action binding and external-exec or webhook conditions are local-only fields (local-layer patches, never emitted); indexer scoping uses indexer tags or indexer definition ids, not local numeric ids.
- Question: are tags and definition ids enough for portable indexer scoping?
- Phase: later phase (racing mode); schema in P1.
- Source: PRF-070 (D20; design/evaluation.md §3.18).
- Used by: 07-profiles.md.

#### 10.9 Runtime and deployment

**OPN-135** (MUST) Bootstrap config file format and location. The bootstrap configuration file is TOML at `<DATA_DIR>/config.toml`, unless `--config` names another path. It ranks below flags and `BONNYREAD_<KEY>` environment variables and above built-in defaults.
- Question: TOML, and at that location?
- Phase: P1.
- Source: PLT-014, DEP-061 (open-issues A8; D20; design/runtime.md §3.10).
- Used by: 12-platform.md, 13-deployment.md.

**OPN-136** (MUST) Notification template language. Notification targets render events with a default template per event type or a per-target override. The template language is not yet chosen; the specification proposes no default.
- Question: which template language do default and override templates use?
- Phase: P1.
- Source: PLT-078 (design/runtime.md §3.7; I8).
- Used by: 12-platform.md.

**OPN-137** (MUST) Unauthenticated metrics listener. There is no unauthenticated metrics endpoint: the admin listener is off by default, and `/metrics` and `/metrics/cluster` require a credential with `metrics:read` (probes stay unauthenticated).
- Question: should an unauthenticated metrics option exist (for example on a private admin listener)?
- Phase: P1.
- Source: DEP-011 (design/identity-api-deployment.md O10).
- Used by: 12-platform.md, 13-deployment.md.

**OPN-138** (MUST) Probe endpoint paths. Health probes are `/livez` (process responsive) and `/readyz` (database reachable, schema compatible, LISTEN subscribed, not draining), unauthenticated, served on both the main and admin listeners.
- Question: are these paths and semantics right?
- Phase: P1.
- Source: DEP-012.
- Used by: 01-architecture.md, 11-api-and-clients.md, 12-platform.md, 13-deployment.md.

### 11. Review and closure

**OPN-073** (MUST) Every requirement elsewhere in the specification that depends on an unconfirmed choice **MUST** carry an `[OPN-NNN]` tag naming an entry in this file.

**OPN-074** (MUST) A new open item **MUST** take the next unused number of its block (OPN-001–072 are the closed set of Group C items; OPN-101–199 are items raised while writing a spec file, grouped by area in §10; OPN-201–299 are cross-file review findings needing a decision, in the section "Review findings needing a decision"), **MUST** be added to the index, and **MUST NOT** reuse or renumber an existing ID. Numbers left unused between blocks stay unused.

**OPN-075** (MUST) When an item is closed, its entry and index row **MUST** record the outcome and date; if the default changed, every file in its "Used by" list **MUST** be updated in the same change, and `docs/decisions.md` or `docs/design/open-issues.md` gains a record of the decision.

**OPN-076** (SHOULD) Items decided by a spike (OPN-019, OPN-027, OPN-053, OPN-058, OPN-065, OPN-070, OPN-121) **SHOULD** be closed with the spike's measured result attached, before the dependent implementation work in `14-roadmap.md` starts.

## Flows

**Closing an open item**
1. The user reviews an entry and confirms or changes the default.
2. The entry and its index row are updated with the outcome (OPN-075).
3. If the default changed, each "Used by" file is updated and its `[OPN-NNN]` tag is kept as traceability.
4. If a spike result contradicts the default, the entry is reopened with the measured result and a new proposed default; it is not closed silently.

## Non-goals

- This file does not hold Group A resolutions or Group B decisions from `open-issues.md`; those are accepted or decided and are specified directly in the owning files.
- It does not restate the requirements that use each default; see the "Used by" files.

## Traceability

| Source | Entries |
|---|---|
| open-issues.md §C Acquisition | OPN-001–008 |
| open-issues.md §C Downloads | OPN-009–016 |
| open-issues.md §C Evaluation | OPN-017–019 |
| open-issues.md §C Identity and API | OPN-020–027 |
| open-issues.md §C Import | OPN-028–037 |
| open-issues.md §C Library | OPN-038–047 |
| open-issues.md §C Metadata and subtitles | OPN-048–055 |
| open-issues.md §C Profiles | OPN-056–064 |
| open-issues.md §C Runtime | OPN-065–072 (the first runtime bullet is split into OPN-065 and OPN-066) |
| Raised while writing the specification (no design-doc question) | OPN-101–138 |
| Cross-file consistency review, needing a decision | OPN-201–209 |

## Review findings needing a decision

These entries were found by the cross-file consistency review of 2026-09-28. Each records a gap or conflict that the review could not resolve mechanically without a product decision. Each gives both positions and a proposed default; the specification currently cites the entry where the gap shows. Mechanical fixes made by the same review are not listed here.

**OPN-201** (MUST) Owner of the credential store for external services. Credentials for external services (metadata-source keys and tokens such as the TVDB bearer token and Hardcover refresh token, subtitle-provider credentials, and similar) are held in one platform-owned `service_credential` table (`12-platform.md`): kind, owning area and subject, write-only secret fields (D16), `Test()` result and last validation time. `08-metadata.md` `MetadataSourceState.credential_id` and `09-subtitles.md` `SubtitleProvider` reference it.
- Positions: (a) one shared table, which MET-084 (limits per credential), MET-091 (per-credential `Test()` and periodic validation) and `MetadataSourceState.credential_id` already assume; (b) secret fields on each owning entity or settings namespace (`12-platform.md` PLT-018, PLT-022), with no separate table. `08-metadata.md` named `02-domain-model.md` as owner and `09-subtitles.md` named a `Credential` table of `02-domain-model.md`, but neither exists; `10-identity-and-access.md`'s `Credential` table holds user sessions and keys only.
- Proposed default: (a), owned by `12-platform.md`.
- Phase: P1.
- Source: MET-016, MET-084, MET-090, MET-091; SUB-062; `08-metadata.md` and `09-subtitles.md` data models.
- Used by: 08-metadata.md, 09-subtitles.md, 12-platform.md.

**OPN-202** (MUST) Plug-in archive helper limits and captcha solver configuration. The host's archive extraction for plug-ins (`archive_list`/`archive_read`, and the host-side extraction of SUB-038) is limited by admin settings `plugins.archive_max_bytes` (default 50 MiB uncompressed per archive) and `plugins.archive_max_entries` (default 1,000). `captcha_solve` uses a captcha-solving service configured as a platform entity like `solver` (kind, URL, write-only key, optional egress).
- Positions: fixed platform defaults with admin settings (proposed) versus limits requested per manifest within a platform ceiling (as memory is, PLT-149). For captcha: a platform entity (proposed) versus plug-in-local settings, which would put a third-party key inside each plug-in's configuration.
- Proposed default: as stated; the numbers are to be validated against Bazarr's provider archives.
- Phase: P2.
- Source: PLT-147; SUB-038, SUB-061; design/metadata-and-subtitles.md §B7, §B10 (limits named but not sized).
- Used by: 09-subtitles.md, 12-platform.md.

**OPN-203** (MUST) Owner and schema of the shared item history. Several files write or read a "history" that no file defines: `05-downloads.md` DLD-109 compacts `DownloadEvent` rows into it, `09-subtitles.md` SUB-007 records subtitle events in it, `06-import.md` IMP-044 writes it in the Record step, `02-domain-model.md` DOM-036 moves it on merges, `04-evaluation.md` EVL-059 says downloads write it, and `10-identity-and-access.md` IAM-062 scopes "item history". Proposed default: one `History` table owned by `02-domain-model.md`: `{item?, slot_definition?, title?, event_type, download_id?, media_file_id?, actor, data JSON, at}`, written in the same transaction as the event it records, retained for 365 days by default.
- Positions: (a) one shared table owned by `02-domain-model.md` (proposed; matches the Arr "history" view per item and slot); (b) no shared table: each area keeps its own records (`DownloadEvent`, `ImportRun`, subtitle events, `ReleaseDecision`) and the UI composes the item history from them.
- Phase: P1.
- Source: DLD-109; SUB-007; IMP-044; DOM-036; EVL-059; IAM-062.
- Used by: 02-domain-model.md, 05-downloads.md, 06-import.md, 09-subtitles.md.

**OPN-204** (MUST) "Why not grabbed" API shape. `04-evaluation.md` EVL-100 requires the Item page to answer "why not grabbed" from `ReleaseDecision` and `PendingGrab` and says API shapes are owned by `11-api-and-clients.md`, which defined none. Proposed default: `GET /api/v1/items/{id}/slots/{slotId}/decisions` (cursor-paged `ReleaseDecision` rows with findings and rank keys) and `GET /api/v1/items/{id}/slots/{slotId}/holds` (`PendingGrab` rows), permission `library:read`.
- Positions: per-ItemSlot sub-resources (proposed) versus a global `GET /api/v1/decisions?itemId=&slotId=` list.
- Phase: P1.
- Source: EVL-097–EVL-100; API-106.
- Used by: 04-evaluation.md, 11-api-and-clients.md.

**OPN-205** (MUST) Profile permission names vs the IAM vocabulary. `07-profiles.md` PRF-108 names `profiles:read`, `profiles:write`, `profiles:author` and `profiles:admin`, and PRF-036/PRF-068 use them; `10-identity-and-access.md` IAM-061/IAM-062 fixes the resource `profile` and the actions `read|create|update|delete|execute|approve|manage`, and forbids other files to add actions. Proposed default: `profiles:read` → `profile:read`; `profiles:write` → `profile:update` (local layer, conflict resolution); `profiles:author` → `profile:execute:publish` (a new facet for drafts and publishing); `profiles:admin` → `profile:manage` (link, unlink, credentials, strategy, policy, priority, extension dependency). IAM-071's `profile:*` row for Manager then grants database administration too, which should be confirmed.
- Positions: map onto IAM's actions with one new facet (proposed) versus adding `author`/`admin` actions to IAM-061 (contradicts its closed action list).
- Phase: P1.
- Source: PRF-036, PRF-068, PRF-107, PRF-108; IAM-061, IAM-062, IAM-071.
- Used by: 07-profiles.md, 10-identity-and-access.md, 11-api-and-clients.md.

**OPN-206** (MUST) API id format vs int64 table keys. `11-api-and-clients.md` API-010 said API ids are UUIDv7 strings; `02-domain-model.md` DOM-001 gives every domain table an int64 auto-increment id, while `05-downloads.md` (`Download.id`) and `10-identity-and-access.md` (`User`, `AuditEvent`) use UUIDs. Proposed default: API ids are opaque strings; tables keep their key types; int64 keys are exposed as decimal strings (so clients never rely on the format).
- Positions: (a) opaque strings over mixed key types (proposed; no schema change); (b) UUIDv7 keys for every table exposed by the API (changes DOM-001 and every edge); (c) a separate public UUID column on int64 tables.
- Phase: P1.
- Source: API-010; DOM-001; DLD-001 (Download UUID); `10-identity-and-access.md` data model.
- Used by: 02-domain-model.md, 11-api-and-clients.md.

**OPN-207** (MUST) `grabbable` predicate. `02-domain-model.md` DOM-088 assigns `grabbable(i,s)` (early grabs such as TV RSS for unaired monitored episodes) to `04-evaluation.md`, and EVL-057 referred back to `02-domain-model.md`; neither defined it. Proposed default: `grabbable(i,s) := monitored(i,s)` (ItemSlot and TitleSlot monitored), independent of `released`, of `include_specials` (DOM-054) and of `search_hold`, which suppresses only automatic searches (DOM-083); interactive search ignores monitoring (DOM-086).
- Positions: whether `search_hold` also blocks RSS and IRC grabs (proposed: no), and whether an unreleased movie is grabbable before its minimum availability (proposed: no; EVL-076 defers it, which already applies).
- Phase: P1.
- Source: DOM-054, DOM-083, DOM-088; EVL-057, EVL-076.
- Used by: 02-domain-model.md, 04-evaluation.md.

**OPN-208** (SHOULD) Event naming: Go type names vs notifiable ids. `02-domain-model.md`, `04-evaluation.md` and `08-metadata.md` name events in CamelCase (`ItemSlotMonitoringChanged`, `AvailabilityChanged`, `EntityMerged`, `TitleAdded`, `LateMembersAppeared`, `ItemRenumbered`, `GrabStarted`), while `05-downloads.md`, `06-import.md` and `12-platform.md` use dotted lowercase ids (`download.grabbed`, `file.imported`) that form the webhook contract (PLT-074, PLT-075). Proposed default: CamelCase names are the Go event types; every event that is notifiable or exposed to webhooks gets a dotted lowercase id `<area>.<event>`, listed by its owning file (for example `AvailabilityChanged` → `request.available` for the requester notification of DOM-091).
- Positions: two naming layers with an explicit mapping (proposed) versus renaming every domain event to the dotted form.
- Phase: P1.
- Source: DOM-081, DOM-083, DOM-091; EVL-093; MET-058; PLT-074.
- Used by: 02-domain-model.md, 04-evaluation.md, 08-metadata.md, 12-platform.md.

**OPN-209** (MUST) Signal that reports a search's grabs. `05-downloads.md` DLD-118 needs the downloads grabbed for a dispatched `search_id` to set the SearchIntent's outcome; `03-acquisition.md` ACQ-090 emits `SearchCompleted{search_id}` when the queries finish, before evaluation commits any grab. Proposed default: evaluation emits `SearchEvaluated{search_id, grabbed []DownloadID}` after the grab transactions of that search's batch (`04-evaluation.md` §5.1), and the backlog scheduler sets the outcome from it; `SearchCompleted` is kept for the budget and lane bookkeeping.
- Positions: a second signal from evaluation (proposed) versus acquisition waiting for evaluation before emitting `SearchCompleted`, which couples the two areas.
- Phase: P1.
- Source: ACQ-090; DLD-118; EVL-103.
- Used by: 03-acquisition.md, 04-evaluation.md, 05-downloads.md.
