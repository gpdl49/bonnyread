# 14. Roadmap

This file specifies how bonnyread is delivered: the deliverables and exit criteria of each phase, the spikes that must run before the work that depends on them, the gates that must pass before the first public release, and the items explicitly deferred beyond the phases. It does not restate area behaviour; each deliverable points to the file that specifies it. Applies to P1, P2 and P3, and to later work.

## Concepts

- **Phase.** A delivery stage with a fixed media-kind and feature scope (D2): **P1**, **P2**, **P3**. "Later" means after P3 or unscheduled.
- **Deliverable.** A feature set, specified elsewhere, that a phase ships.
- **Groundwork.** Schema and pipeline work shipped in an earlier phase so a later phase needs no schema rewrite or rescan (D2; open-issues A17).
- **Exit criterion.** A verifiable condition that must hold before a phase is declared complete.
- **Spike.** A time-boxed experiment that decides an open item. Its result is attached to the open item in `15-open-items.md` (OPN-076).
- **Release gate.** A condition that must hold before the first public release, independent of phase.
- **Deferred item.** A feature explicitly out of P1–P3 scope, recorded so that interfaces stay ready for it.

## Requirements

### 1. General

**RDM-001** (MUST) Phases **MUST** be delivered in order P1, P2, P3. A phase's deliverables **MUST NOT** require a schema rewrite of data created by an earlier phase; additive, expand/contract migrations are allowed. _Source: D2; open-issues A7._

**RDM-002** (MUST) The data model for every media kind (Title, Item, SlotDefinition, per-kind quality axes, FileLink M:N) **MUST** exist from the first P1 release, even where the kind's behaviour ships later. _Source: D2; 00-overview.md OVR-002._

**RDM-003** (MUST) Media kinds of later phases **MUST** be implemented as modules that a build can exclude, and an excluded or not-yet-released kind **MUST NOT** appear in the UI, API listings or permissions defaults. _Source: design/runtime.md §3.1._

**RDM-004** (MUST) Work that depends on a spike (§5) **MUST NOT** be committed to until the spike's open item is closed or its documented fallback is chosen. _Source: D8; open-issues §C._

### 2. Phase 1: TV, movies, indexers, profiles, download handling, requests

**RDM-005** (MUST) P1 **MUST** deliver the following.

| Area | Deliverable | Spec |
|---|---|---|
| Parser | Ported Sonarr and Radarr golden corpora (first task of P1, before parser implementation); the parser chosen by the D8 spike; regexp2 with timeouts and `\p{IsBlock}` normalisation; Lidarr, Bookshelf, Whisparr and Kapowarr fixtures ported as data (run in their phases) | 04 |
| Library | Catalog, Title/Item/Edition, SlotDefinition/TitleSlot/ItemSlot, Follow, Intent, Exclusion, MediaFile/FileLink with root-relative paths; TV and movie kinds; wanted and availability queries; import lists | 02 |
| Metadata | TVDB v4 (project key, optional user PIN) as canonical TV source, or the TMDb TV fallback per RDM-027; TMDb for movies, images, ratings and cross-ids; TheXEM; runtime-fetched scene-mapping and anime-id datasets (never embedded, MET-028); TMDb 180-day rule enforcement; attribution | 08 |
| Acquisition | One Indexer with facets; fresh Go Cardigann engine with a purpose-built template interpreter; Prowlarr v11 definitions fetched by commit and stored in the DB [OPN-001]; Newznab/Torznab/RSS; RSS loop; IRC ingest with an embedded autobrr baseline [OPN-002]; P1 native indexers (BTN, PTP, HDBits, BeyondHD, AnimeBytes, FileList, the Avistaz family, IPT/TorrentDay where YAML is inadequate); budgets and the DB-backed rate limiter; FlareSolverr-compatible solvers and proxies | 03 |
| Evaluation | Canonical quality model (per-kind axes); custom formats; per-slot profiles and scoring; upgrade, proper/repack and delay rules; coverage-aware selection; one download feeding several slots; explainable findings | 04 |
| Downloads | qBittorrent, Transmission, Deluge, rTorrent, SABnzbd, NZBGet and blackhole adapters (aria2 and Flood after P1, DLD-028); phase/retention state machine; duration-based policies; extraction with staging root; seeding manager; blocklist; backlog scheduler; DDL-ready interfaces (link groups, resolve step) without DDL | 05 |
| Import | Identification, decisions, transfer (rename, reflink, hardlink, copy), naming engine, extras, library scan and watch, external processing hold state, missing-file handling, TV and movie kinds | 06 |
| Profiles | PCD engine: link, pull, pin, materialise, local/draft patches, override/align/ask conflicts (user value in effect under `ask`), authoring and direct-push publishing [OPN-057]; Dictionarry as default; bundled snapshot [OPN-062]; extension schema for announce conditions, autograb rules and slot/tag-scoped delay | 07 |
| Requests | Request Intents with approval, per-slot requestability and auto-approval, quotas [OPN-021], issues, TMDb discovery with TMDb→TVDB resolution | 02, 08, 10 |
| Identity | Built-in auth (argon2id, TOTP), OIDC, Plex/Jellyfin/Emby login, sessions, device credentials, API keys, RBAC default roles, scopes, audit log, forward-auth opt-in (off) | 10 |
| API and clients | `/api/v1` OpenAPI (generator chosen by spike [OPN-027]), SSE with replay [OPN-026], SvelteKit web UI, Tauri desktop app with local-socket pairing, generated TypeScript client | 11 |
| Platform | Durable job queue, schedules, leases for IRC and file watcher, events with outbox, notifications/webhooks/scripts/alert sinks, health checks (including filesystem co-location), `/metrics` and `/metrics/cluster`, WASM plug-in host [OPN-065], backup/restore with optional encryption and sanitised export, capability-routed jobs | 12 |
| Deployment | Container image (`CGO_ENABLED=0`, pure-Go SQLite, bundled ffprobe and fpcalc), Helm chart for HA, OS service installers for Windows, macOS [OPN-023] and Linux, `db copy` SQLite→PostgreSQL [OPN-072] | 13 |

_Source: D2; D4; D20; design/acquisition.md §3.13; open-issues A1–A20._

**RDM-006** (MUST) P1 **MUST** ship the groundwork for P2 and P3:
1. subtitle sidecar files indexed into the shared subtitle table with language, HI and forced flags;
2. subtitle stream dispositions recorded by the shared probe;
3. TMDb discovery and TMDb→TVDB id resolution for requests;
4. per-kind quality axes (Video, Audio, Text, Archive) and M:N file↔item links in the schema;
5. the plug-in ABI and host (subtitle providers are its first P2 users);
6. DDL-ready release and download interfaces (release as link groups, grab-time resolve).

_Source: open-issues A17; design/metadata-and-subtitles.md §6.1 item 7; D11; D15._

**RDM-007** (MUST) P1 exit criteria:

| # | Criterion |
|---|---|
| E1.1 | Every case of the ported Sonarr and Radarr parser corpora passes, or is listed as a reviewed exception with the upstream bug reference (D20 attribute authority) |
| E1.2 | The Cardigann golden suite renders every template in every active v11 definition against recorded fixtures without error; failures are listed per definition (open-issues A19) |
| E1.3 | Every Dictionarry custom-format pattern compiles after normalisation, and Dictionarry CF tests pass or are listed as reported upstream disagreements |
| E1.4 | Platform conformance tests (job claim, dedupe, leases, schedules, rate limiter) pass on SQLite and on PostgreSQL 15 with at least two nodes |
| E1.5 | A rolling upgrade across one schema migration runs with N-1 compatibility; oasdiff reports no breaking API change against the previous P1 release |
| E1.6 | `db copy` from a populated SQLite install to PostgreSQL passes its CI test |
| E1.7 | Security invariants have tests: no unauthenticated mode, query-string credentials rejected, first Owner created only through the bootstrap paths (local-endpoint pairing, setup URL, bootstrap owner; IAM-002), one authoriser for every entry point |
| E1.8 | The desktop first run (install service, pair over local socket, create Owner) works on Windows, macOS and Linux |
| E1.9 | Every P1-scoped spike (RDM-018) is closed in `15-open-items.md` |
| E1.10 | The TVDB licensing outcome is known and, if refused, the fallback of RDM-027 is shipped |

### 3. Phase 2: subtitles and music

**RDM-008** (MUST) P2 **MUST** deliver:

| Area | Deliverable | Spec |
|---|---|---|
| Subtitles | Subtitle profiles [OPN-049], wanted computation, embedded-track detection, search orchestration, matching and scoring, post-processing, sync (alass default [OPN-051]), upgrade handling [OPN-050] | 09 |
| Subtitle providers | Wave 1 at launch: opensubtitlescom, subdl, subsource, gestdown, jimaku, animetosho, napiprojekt; wave 1b JSON regional providers after launch; all as signed WASM plug-ins | 09, 12 |
| Music | Music kind: artist/album/release/track model, Lossless/Lossy slots, metadata profiles [OPN-042], follows | 02 |
| Music metadata | MusicBrainz (+ Cover Art Archive, AcoustID) direct with rate-limited queues, incremental refresh and optional user-run mirror | 08 |
| Music import | Distance-based identification ported with Lidarr fixtures, fingerprinting as escalation only, audio tag writing | 06 |
| Music evaluation | Audio quality axes and music CF conditions via the PCD extension schema or upstream schema changes; project starter database for music | 04, 07 |
| Indexers | Redacted, Orpheus and the Gazelle family as natives, with API enrichment for IRC | 03 |

_Source: D2; design/metadata-and-subtitles.md §3.B11; design/acquisition.md §3.13; D20._

**RDM-009** (MUST) The PCD schema route for non-video kinds (upstream proposal to `Dictionarry-Hub/schema`, or the extension-schema fallback) **MUST** be settled before music evaluation work starts. _Source: D20; design/profiles.md §6.1 item 1._

**RDM-010** (MUST) P2 exit criteria:

| # | Criterion |
|---|---|
| E2.1 | Ported Lidarr identification fixtures pass, or are listed as reviewed exceptions (including cases changed by ambiguity rejection, D10) |
| E2.2 | A first import of a 5,000-album library completes within MusicBrainz limits without exceeding the shared rate limiter's budget |
| E2.3 | Wave-1 subtitle providers pass their recorded fixtures; subtitle matching uses the main release parser (D1c) |
| E2.4 | The subzero OCR dictionary licence is resolved before the OCR-fix mod ships [OPN-052] |
| E2.5 | Music starter database is published and linkable |

### 4. Phase 3: books, adult, comics

**RDM-011** (MUST) P3 **MUST** deliver:

| Area | Deliverable | Spec |
|---|---|---|
| Books | Book kind with Ebook and Audiobook slots, per-slot preferred edition, separate score domains [OPN-059]; Hardcover (OAuth device flow) primary, OpenLibrary fallback, Audnexus/AudiobookDB; book identification [OPN-029]; Calibre downstream only [OPN-028] | 02, 06, 07, 08 |
| Adult | Adult kind on the Whisparr v3 model with followable Studio and Performer; v2 safeguards (non-unique product-code index with code-based fallback, ambiguity rejection, per-indexer search settings, adult default categories); StashDB and ThePornDB with per-user keys | 02, 04, 06, 08 |
| Comics | Comic kind; ComicVine (per-user key) with Metron optional; Usenet and torrent through the indexer layer; collected editions [OPN-038]; comic transforms [OPN-037] | 02, 06, 08 |
| DDL | Optional, isolated DDL module: declarative site definitions [OPN-007], host resolvers (core streams bytes), per-host quota via budgets | 03, 05 |
| Indexers | MyAnonamouse native | 03 |
| Profiles | Starter databases for books, adult and comics | 07 |

_Source: D2; D3d; D10; D11; design/acquisition.md §3.13; open-issues A11._

**RDM-012** (MUST) P3 exit criteria:

| # | Criterion |
|---|---|
| E3.1 | Ported Bookshelf, Whisparr v2/v3 and Kapowarr fixtures pass, or are listed as reviewed exceptions |
| E3.2 | An ebook and an audiobook of the same book import into their two slots with different editions |
| E3.3 | Ambiguous adult date matches are rejected, including in fuzzy matching |
| E3.4 | DDL site and host definitions pass fixture tests; the DDL module can be disabled without affecting other protocols |
| E3.5 | Hardcover OAuth client is registered and its user-data clause is covered by the legal review [OPN-054] |

### 5. Spikes

**RDM-013** (MUST) The D8 parser spike **MUST** run after the Sonarr and Radarr corpora are ported and before parser implementation. It measures `rls` (autobrr fork) as tokeniser and attribute layer against the corpora and reports: structure pass rate per corpus; per-kind attribute agreement (source, resolution, modifier, codec, HDR, language) with the ported `QualityParser`/`LanguageParser` tests [OPN-019]; and latency. The result chooses between ported cascades on regexp2 and the hybrid (rls for tokens and attributes, cascades for numbering). _Source: D8; D20; design/evaluation.md §6 Q8._

**RDM-014** (MUST) The API generator spike **MUST** implement three endpoints spec-first with oapi-codegen strict server, covering request validation, SSE documentation and a generated TypeScript client, before API work is committed; Huma and ogen are the fallbacks [OPN-027]. _Source: design/identity-api-deployment.md §6._

**RDM-015** (MUST) The plug-in spike **MUST** compare the Extism Go SDK with a custom wazero ABI on per-call timeouts, memory caps and instance pooling [OPN-065], and measure the cost of `WithCloseOnContextDone` on a hot path such as subtitle scoring [OPN-070]. It runs before the plug-in host is built in P1. _Source: design/runtime.md §6 open questions 1, 6._

**RDM-016** (SHOULD) Metadata change-feed spikes **SHOULD** verify TVDB `/updates` and TMDb `/changes` (14-day window) in P1, MusicBrainz merged-MBID redirects in P2, and StashDB `updated_at`, ComicVine `date_last_updated` and Hardcover `updated_at` in P3. They are non-blocking: the refresh engine falls back to age-based policies [OPN-053]. _Source: design/metadata-and-subtitles.md §6.2 Q8._

**RDM-017** (MUST) The go-git spike **MUST** verify pushing from a depth-1 clone before publishing is built; the fallback is deepening the fetch on publish only; the git CLI never becomes a runtime requirement (`07-profiles.md` PRF-064) [OPN-058]. _Source: design/profiles.md §6.2 Q3._

**RDM-018** (MUST) Spike schedule:

| Spike | Open item | Must close before | Phase |
|---|---|---|---|
| D8 parser / rls, incl. attribute agreement | OPN-019 | Parser implementation | P1 (start) |
| oapi-codegen | OPN-027 | API handlers | P1 (start) |
| Extism vs custom ABI | OPN-065 | Plug-in host | P1 |
| wazero timeout cost | OPN-070 | Plug-in host | P1 |
| go-git shallow push | OPN-058 | PCD publishing | P1 |
| TagLib binding | OPN-121 | Music probing and tag writing | P2 |
| Change feeds: TVDB, TMDb | OPN-053 | (non-blocking) TV/movie refresh tuning | P1 |
| Change feeds: MusicBrainz | OPN-053 | (non-blocking) music refresh tuning | P2 |
| Change feeds: StashDB, ComicVine, Hardcover | OPN-053 | (non-blocking) P3 refresh tuning | P3 |

**RDM-019** (SHOULD) Defaults marked for tuning (indexer budgets [OPN-004], download policy thresholds [OPN-012], book match threshold [OPN-029]) **SHOULD** be validated against beta telemetry or fixtures before the phase that introduces them exits.

### 6. Pre-release gates

The first public release requires every gate below, regardless of phase.

**RDM-020** (MUST) Legal review. A legal review **MUST** be completed before the first public release and **MUST** cover at least:
1. the fresh Cardigann engine and runtime-fetched definitions approach (D5b), including reproducing the v11 schema whose filter names and semantics were authored in Jackett (GPL-2.0-only);
2. fetched definitions and PCD data stored in the user's database and therefore in the user's backups (open-issues A2);
3. runtime fetching from the unlicensed Prowlarr/Indexers repository;
4. community anime and scene datasets with unverified licences (Fribb, Anime-Lists and similar), fetched at runtime and never embedded;
5. licensing of third-party WASM plug-ins loaded in-process (combined-work question) and what the plug-in index may host;
6. AcoustID non-commercial application keys under AGPL-3.0 distribution;
7. Hardcover's clause on commercial use of user-owned data.

_Source: D5b; D20; open-issues B12; design/acquisition.md §6.1 I1–I2; design/metadata-and-subtitles.md §6.1 items 4, 6; design/import.md §6.1 item 7._

**RDM-021** (MUST) The legal review outcome **MUST** be recorded in `docs/decisions.md`, and any item it rejects **MUST** be removed or changed in the specification before release.

**RDM-022** (MUST) TVDB licensing. The project **MUST** apply for TVDB's free/open-source licensed tier before P1 implementation of TV metadata begins. _Source: D20; open-issues B9._

**RDM-023** (MUST) If TVDB refuses the licensed tier, TV **MUST** fall back to TMDb TV plus an episode-order mapping layer (TMDb episode groups, XEM's TVDB origin) so that scene, XEM and anime numbering keep working; a user-supplied TVDB subscriber PIN **MAY** still enable TVDB. _Source: D20; D3a verification note; design/metadata-and-subtitles.md §6.1 item 1._

**RDM-024** (MUST) Licence scan. `go-licenses` (or an equivalent) **MUST** be run over the final module graph, including transitive dependencies, and **MUST** report only licences compatible with AGPL-3.0 distribution; bundled native tools (ffprobe, fpcalc, alass) **MUST** be included in the check. _Source: D5; open-issues A6._

**RDM-025** (MUST) Project-registered keys (TMDb, TVDB, AcoustID, OpenSubtitles and others) **MUST** be registered by the project, attributed, and user-overridable; no key copied from another app may ship. Subtitle-provider keys are held by the host and passed to plug-ins at configure time, never compiled into modules. _Source: D3b; design/metadata-and-subtitles.md §6.1 item 2._

**RDM-026** (MUST) Source attribution (TMDb logo and disclaimer; TVDB, TVMaze, ComicVine and MusicBrainz links and notices) **MUST** be visible in the UI before release. _Source: D3._

**RDM-027** (MUST) The TVDB outcome of RDM-022 **MUST** be known before the P1 exit, and **MUST** decide whether P1 ships TVDB as canonical TV source or the RDM-023 fallback.

### 7. Deferred items

**RDM-028** (MUST) The following **MUST NOT** be part of P1–P3 scope unless re-planned in this file. Interfaces named in the "Kept ready" column **MUST** be preserved so that the item can be added without redesign.

| Item | Status | Kept ready | Source |
|---|---|---|---|
| Self-updater | Deferred until the core architecture is established | `apiVersion`/`minClientApiVersion` negotiation; server web UI fallback | D16; A7 |
| Racing mode (library-less autograb) | Later phase, after P1 | IRC ingest, announce conditions and autograb rules in the extension schema, `kind=racing` downloads; budgets [OPN-008]; slot vs rule overlap [OPN-017] | D16; D20 |
| Tauri mobile apps | Later | Shared UI, native transport, device credentials, QR/deep-link pairing | D12 |
| Mobile push (APNs/FCM) | Not in v1; needs a hosted relay | Notification target interface; SSE resume; ntfy and similar targets | D20; B13 |
| Alternate TV orderings (DVD, absolute) | After v1 [OPN-044] | TVDB ordering stored per episode | design/library.md §6.2 Q7 |
| Passkeys (WebAuthn) | Later [OPN-020] | Identity table with multiple methods | design/identity-api-deployment.md §6 |
| OIDC back-channel logout | Later [OPN-024] | Sessions per identity | design/identity-api-deployment.md §6 |
| Jackett definition overlay | Later [OPN-001] | Multiple definition feeds with precedence | design/acquisition.md §6.2 Q1 |
| Prowlarr/Jackett/autobrr config importers | Later [OPN-005] | Tracker-native category keys | design/acquisition.md §6.2 Q5 |
| Recyclarr translator | Later [OPN-063] | Local patch layer | design/profiles.md §6.2 Q8 |
| PCD pull-request publishing | Later [OPN-057] | Publish pipeline | design/profiles.md §6.2 Q2 |
| Condition-level rename journal | Later [OPN-064] | Rename journal | design/profiles.md §6.2 Q9 |
| Declarative provisioning (GitOps) | Later [OPN-067] | Settings pinning | design/runtime.md §6 |
| Audiobookshelf and Stash metadata writers | Later [OPN-031] | `MetadataWriter` interface | design/import.md §6.2 Q4 |
| Hosted metadata cache | Later, optional | Per-kind provider interface | D3c |
| Per-slot subtitle profile override | Later, if asked for [OPN-049] | Subtitle profile reference | design/metadata-and-subtitles.md §6.2 Q4 |
| Binary plug-in ABI (for example CBOR) | Later, new ABI version [OPN-055] | ABI version in manifests | design/metadata-and-subtitles.md §6.2 Q10 |
| Subtitle provider waves 2 and 3, AI generation/translation | After P2 launch; wave 3 after solver capabilities are proven | Plug-in capabilities | design/metadata-and-subtitles.md §3.B11 |
| Profilarr user-ops importer, unmanaged-item adoption | P1 only if cheap, else later [OPN-061], [OPN-015] | Local patches; manual import from a path | design/profiles.md §6.2 Q6; design/downloads.md §6.2 Q9 |

## Flows

**Phase exit**
1. All deliverables of the phase are merged behind their module.
2. Each exit criterion is checked and its evidence (CI run, fixture report, spike result) is linked in the release notes.
3. Open items of the phase are either closed or explicitly carried forward in `15-open-items.md`.
4. If a criterion fails, the phase stays open; a criterion is only relaxed by a recorded decision.

**First public release**
1. P1 exit criteria hold.
2. Legal review (RDM-020) is complete and recorded (RDM-021).
3. TVDB outcome is known and reflected (RDM-022, RDM-023, RDM-027).
4. Licence scan passes (RDM-024); keys and attribution are in place (RDM-025, RDM-026).

**TVDB refusal**
1. TVDB refuses the licensed tier.
2. P1 TV metadata uses TMDb TV with the episode-order mapping layer (RDM-023).
3. Scene, XEM and anime numbering tests run against the mapping layer; a TVDB subscriber PIN remains an optional upgrade.

## Non-goals

- Dates and staffing: this file orders work and defines completion, not a calendar.
- Features not in D2's scope list (00-overview.md §2).

## Traceability

| Reference | Preserved | Changed (D19) |
|---|---|---|
| D2 phasing | P1/P2/P3 scope as decided | Groundwork made explicit (A17) |
| D8 | Corpora first, spike, then choose | Spike also measures attribute agreement (OPN-019) |
| D5b, D20 | Legal review before release | Scope extended to the D20/B12 items |
| D16 | Racing mode later, self-updater deferred | — |
