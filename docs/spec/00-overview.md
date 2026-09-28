# 00. Overview

This file states what bonnyread is, which applications it replaces, how delivery is phased, the principles every other spec file follows, and the canonical vocabulary used across the specification. It also indexes the architecture decisions and maps spec files to areas. It applies to all phases (P1, P2, P3).

## Concepts

The terms used in this file are defined in the glossary (§5). "The reference apps" means the upstream applications analysed in `docs/analysis/`.

## Requirements

### 1. Purpose

bonnyread is a single self-hosted Go server (the **core**) that replaces the *arr ecosystem: media library management, indexer management, release evaluation, download handling, profile management, requests and subtitles, for TV, movies, music, books, adult content and comics. It has one data model, one pipeline, one API and one UI.

**OVR-001** (MUST) bonnyread **MUST** be delivered as one Go module and one server binary that holds all business logic, exposes the REST API `/api/v1` (OpenAPI), live events (SSE) and metrics, and embeds the web UI. _Source: D12; D6; design/runtime.md §3.1._

**OVR-002** (MUST) The data model and pipeline **MUST** cover every media kind (TV, movie, music, book, adult, comic) from P1, so that later phases add behaviour without schema rewrites. _Source: D2; open-issues A17._

**OVR-003** (MUST) The supported databases **MUST** be PostgreSQL ≥ 15 (production; required for HA) and SQLite (desktop, small installs, development, tests). MySQL/MariaDB and SQL Server **MUST NOT** be supported. _Source: D13; D20; [OPN-071]._

**OVR-004** (MUST) bonnyread **MUST** be licensed AGPL-3.0. _Source: D5a._

### 2. Goals and non-goals

Goals:

| # | Goal | Decision |
|---|---|---|
| G1 | Replace the applications in §3 with one server, without multi-instance workarounds | D2, D4 |
| G2 | Several versions of one item (for example 4K and 1080p, ebook and audiobook) as **slots**, grabbed, upgraded and imported independently; one download may feed several slots | D4, D20 |
| G3 | Keep the reference apps' user-visible outcomes and correctness invariants (parser results, scoring semantics, numbering rules) | D19, D1a |
| G4 | Community profile data (PCD, Dictionarry) as the native profile store | D9, D20 |
| G5 | Run the whole deployment range: single-user desktop service with a native app up to active-active HA on Kubernetes | D12, D14, D18 |
| G6 | No hosted infrastructure: every external source is called directly | D3c |
| G7 | Observability as a first-class feature: Prometheus metrics and alert sinks | D7 |
| G8 | Secure by default (§6.3) | D16, D17 |

**OVR-005** (MUST NOT) bonnyread **MUST NOT** provide compatibility façades for external APIs: no Sonarr/Radarr v3, Lidarr/Bookshelf v1, Seerr v1, Bazarr, or Newznab/Torznab server endpoints. Webhooks and custom scripts use bonnyread's own documented formats only. _Source: D7._

**OVR-006** (MUST NOT) bonnyread **MUST NOT** run or depend on project-hosted services (metadata proxies, OAuth relays, push relays, definition mirrors) in v1. _Source: D3c; D20 (no mobile push)._

Non-goals:

| Non-goal | Reason |
|---|---|
| API compatibility with the reference apps or their consumers (Maintainerr, Homepage, Home Assistant, nzb360, Notifiarr, Requestrr) | D7: consumers duplicate bonnyread or are dropped |
| Mylar3, Tdarr, Notifiarr, Huntarr, Sportarr, LazyLibrarian, Readarr | D2: considered and not added |
| Syncing profiles into external Arr instances (Profilarr sync, drift, cleanup) | D9a: the store is native |
| Mobile push notifications in v1 | D20: needs a hosted relay |
| Self-updater | D16: deferred (see `14-roadmap.md`) |
| Loading unverified native code as plug-ins | D15 |
| MySQL/MariaDB, SQL Server | D13, D20 |

### 3. Applications replaced

| Reference app(s) | Area | Replaced by | Phase | Spec files |
|---|---|---|---|---|
| Sonarr | TV | Library model + evaluation + downloads + import, TV kind | P1 | 02, 04, 05, 06, 08 |
| Radarr | Movies | Same pipeline, movie kind | P1 | 02, 04, 05, 06, 08 |
| Prowlarr, Jackett, autobrr | Indexers (search, RSS, IRC, API) | One indexer layer: one Indexer with facets; fresh Go Cardigann engine; definitions fetched at runtime | P1 | 03 |
| Profilarr, Recyclarr | Profiles | Native PCD store with Dictionarry as the default linked database; Recyclarr via Dictionarry's `trash-pcd` | P1 | 07 |
| Cleanuparr, Unpackerr | Download handling | Native download policies, `EXTRACTING` phase, seeding manager | P1 | 05 |
| Seerr | Requests | Users, roles, quotas, request Intents with approval, issues, TMDb discovery | P1 | 02, 08, 10, 11 |
| Bazarr | Subtitles | Subtitle pipeline; providers ported as WASM plug-ins | P2 | 09, 12 |
| Lidarr | Music | Music kind; MusicBrainz, Cover Art Archive, AcoustID direct | P2 | 02, 04, 06, 08 |
| Bookshelf (Readarr fork) | Books | Book kind with Ebook/Audiobook slots; Hardcover, OpenLibrary, Audnexus | P3 | 02, 04, 06, 08 |
| Whisparr v2, Whisparr v3 (Eros) | Adult | Adult kind modelled on v3 with v2 safeguards restored; StashDB, ThePornDB | P3 | 02, 04, 06, 08 |
| Kapowarr | Comics | Comic kind; Usenet/torrent through the indexer layer; optional DDL module | P3 | 02, 03, 05, 06, 08 |
| Tray apps of the reference apps | Desktop | Tauri desktop app + core installed as an OS service | P1 | 11, 13 |

**OVR-007** (MUST) Each replaced area **MUST** preserve the reference app's user-visible outcomes listed in the owning spec file's Traceability table, and **MUST NOT** carry over workflows that exist only because the reference apps are separate processes (D19). _Source: D19._

### 4. Phases

| Phase | Scope | Decision |
|---|---|---|
| **P1** | TV, movies; indexers (search, RSS, IRC); profiles; download handling (policies, extraction); requests; identity and API; web UI and Tauri desktop app; single node and HA runtime. Includes phase-2/3 groundwork (below) | D2, D12, D14, A17 |
| **P2** | Subtitles; music | D2 |
| **P3** | Books, adult, comics; optional DDL module | D2, D11 |
| Later | Racing mode, Tauri mobile apps, self-updater and other deferred items in `14-roadmap.md` §Deferred | D16, D12 |

**OVR-008** (MUST) P1 **MUST** include the groundwork later phases depend on: subtitle sidecar indexing and subtitle stream flags (language, HI, forced) in the shared probe; TMDb discovery with TMDb→TVDB resolution for requests; per-kind quality axes and M:N file↔item links in the schema. _Source: open-issues A17; design/metadata-and-subtitles.md §6.1 item 7._

**OVR-009** (MUST) Phase-2 and phase-3 media kinds **MUST** be registered as separate modules so a build can leave them out without affecting P1 behaviour. _Source: design/runtime.md §3.1._

Deliverables and exit criteria per phase are in `14-roadmap.md`.

### 5. Glossary

Canonical terms. Other spec files use these names; where a design doc uses a different name for the same concept, the alias is listed and **MUST NOT** be used in normative text except when naming that design's own field.

**OVR-010** (MUST) Spec files **MUST** use the canonical terms of this glossary with the meanings given here. A file that needs a new shared term defines it in its Concepts section and it is added here. _Source: CONVENTIONS.md §Canonical terms._

#### 5.1 Library and catalog (owner: `02-domain-model.md`)

| Term | Definition | Aliases / notes |
|---|---|---|
| Media kind | One of `tv`, `movie`, `music`, `book`, `adult`, `comic`. Scopes slots, profiles, permissions and modules | "kind"; PCD `arr_type` (radarr→movie, sonarr→tv) |
| Catalog | Metadata-owned, refreshable data about entities and items, whether or not the user wants them. Holds no user decisions | replaces `*Metadata` tables |
| Entity | A catalog node that groups or credits items: series, season, artist, author, comic volume, studio, network, performer, person, collection, book series, label, publisher | "container" in evaluation's `Match` |
| Item | The unit of want: the smallest thing a user monitors and acquisition targets. Item kinds: `episode`, `movie`, `album`, `book`, `scene`, `adult_movie`, `issue` | "unit" (design/evaluation.md) |
| Edition | A variant of an item (book edition; music release) that defines what "complete" means. Not a unit of want | music "release" |
| Track | A component of a music edition | — |
| Title | The library root the user added and that owns folders. Points at an entity (series, artist, author, comic volume) or an item (movie, scene, adult movie) | "series"/"artist" in reference apps |
| SlotDefinition | A named version kind defined once per media kind, for example movie {HD, UHD}, book {Ebook, Audiobook}. Carries requestability and auto-approval settings | "slot", "version slot" (D4) |
| TitleSlot | Per (title, slot): root folder, path, quality profile and pause flag | slot configuration |
| ItemSlot | Per (item, slot): the monitored flag, target edition (books) or monitored release (music), and coverage. The only stored monitoring state | — |
| Coverage | Whether an ItemSlot's content is present: `none`, `partial`, `complete` | — |
| Wanted / missing / cutoff-unmet | Queries over ItemSlots, never stored state | — |
| Availability | A query over coverage (for example "complete", "caught up") used for request status | — |
| Follow | A durable policy "keep the members of entity E in the library for slot S": which new members to add and monitor, date cut-off, defaults | series `MonitorNewItems`, performer/studio monitoring, collections |
| Intent | A recorded reason for something to be in the library: manual add, request, import-list entry, watchlist, follow expansion, disk import. Requests are Intents with an approval workflow | "request" (Seerr). Not the same as SearchIntent (§5.4) |
| Exclusion | "Never add automatically" / "never allow requests", keyed by external id or catalog node | — |
| ExternalID | A (provider, namespace, id) triple in one unique map, for every node type, with state `active`, `merged` or `deleted` | "foreign id" (reference apps); `current`/`alias` (design/library.md) |
| Product code | Non-unique adult alternate id with its own non-unique index, outside the ExternalID map | D10; A14 |
| MediaFile | A media file under a root folder, stored as root folder id + path relative to the root. Belongs to exactly one slot | "episode file", "track file" |
| FileLink | The M:N relation "this MediaFile contains this item (and track or edition part)" | — |
| Root folder | A library directory that MediaFiles live under | — |
| Metadata profile | Per-follow filter deciding which catalog members (albums, books) are admitted | Lidarr/Bookshelf metadata profile |
| Subtitle | The shared record of one subtitle of a MediaFile (embedded track, sidecar or downloaded), owned by `02-domain-model.md` | "subtitle record"; a sidecar's file is an Extra file |
| Media server | A Plex, Jellyfin, Emby or similar connection, used for refresh, sign-in and watchlists | "media-server connection" |

#### 5.2 Acquisition (owner: `03-acquisition.md`)

| Term | Definition | Aliases / notes |
|---|---|---|
| Indexer | One configured tracker or indexer instance. All capabilities are facets of it | "tracker", Prowlarr "indexer" |
| Facet | One capability of an Indexer: `search`, `rss`, `irc`, `api` (enrichment), `grab`, `ddl`. Facets share credentials, session, proxy, budgets, health and seed policy | — |
| Driver | The implementation behind an Indexer: Cardigann, native, Newznab/Torznab/RSS, or WASM plug-in. Implements `Query` and `Fetch` | — |
| Definition | A runtime-fetched indexer definition (Cardigann YAML, IRC announce, or DDL site), stored in the database, pinned by commit | "indexer definition" |
| Definition feed / set | A feed is a source repository and path; a set is one fetched commit of a feed (`staged`, `active`, `retired`) | — |
| Native indexer | A hand-written driver for a tracker API (BTN, PTP, HDBits, …) | — |
| Release | A normalized result from an indexer (search, RSS, IRC or manual) with a stable id (indexer + guid or infohash) | "search result", "report" |
| Announce | A release arriving over IRC, with announce metadata (uploader, freeleech, category, …) | — |
| SearchRequest | A typed, kind-specific search from the library, backlog or UI, planned into indexer queries | — |
| Budget | Per-indexer query and grab limits per window, class-aware (interactive, automatic, RSS, backlog) | "limits" |
| Rate limiter | The DB-backed, cross-node request-spacing and quota service shared by indexers and metadata sources | A3 |
| TrackerAlias | Mapping between Cardigann, IRC and native ids for one tracker | — |
| Protocol | `torrent`, `usenet` or `ddl` | — |
| DDL | Direct download: a release made of groups of mirror links, resolved at grab time (P3) | D11 |

#### 5.3 Evaluation and profiles (owners: `04-evaluation.md`, `07-profiles.md`)

| Term | Definition | Aliases / notes |
|---|---|---|
| ParsedRelease | The profile-independent parse of a release title: structure (numbering, year, …) and attributes | — |
| Attributes | Parsed source, resolution, modifier, codec, HDR, audio, language, revision, group, edition. The ported Arr parser tests define detection | D20 attribute authority |
| Match | The mapping of a ParsedRelease onto library entities and items, with method and evidence | — |
| Quality | The canonical quality id derived from per-kind axes within a family (Video, Audio, Text, Archive) | — |
| Custom format (CF) | A named set of conditions over a release; matched with regexp2 per D1 | CF |
| Quality profile | Per slot: allowed qualities, cutoff, CF scores and minimums, languages, delay policy | "profile" |
| Target | What a release is evaluated for: a slot of an item, or (later) an autograb rule | "import target" differs (§5.5) |
| Candidate | (release facts, target, profile, incumbents), scored | — |
| Decision | `Accept`, `Reject` or `Defer` for one Candidate, with every Finding | — |
| Finding | One check result with a stable reason code and parameters; the basis of explainability | "rejection" (reference apps) |
| Incumbent | An existing file or in-flight download for the same item and slot | — |
| Coverage-aware selection | Among releases that pass the profile, prefer the combination that satisfies more wanted items, then profile score and quality | D20 |
| Pending grab | A persisted Defer (delay policy) that is re-evaluated at its due time | "pending release" |
| Delay policy | Protocol and time preferences that defer grabs; in the PCD extension schema, attached to the quality profile with slot/tag scope | "delay profile" |
| PCD | Profilarr Compliant Database: a git repository with `pcd.json`, `ops/*.sql` and a schema dependency | — |
| Profile database | A PCD linked by git, the built-in Local database, or a bundled snapshot. Every profile entity belongs to exactly one | "database" in design/profiles.md; never "the database" alone, which means the SQL database |
| Upstream state | A profile database replayed at its pinned commit | — |
| Patch | One user or author change on one path of one aggregate; layers `local` and `draft` | Profilarr "user op", "base draft" |
| Effective state | Upstream ⊕ draft ⊕ local, materialised in normal tables; the only profile state the rest of bonnyread reads | — |
| Aggregate | The unit of diff and patch: Regex, CustomFormat, QualityProfile, DelayProfile, NamingPreset, MediaSettingsPreset, QualitySizePreset, TestEntity | — |
| Conflict strategy | `override`, `align` or `ask`; under `ask` the user's value stays in effect until the user decides | D20 |
| Extension schema | bonnyread's additive PCD schema for what upstream cannot express (non-video kinds, announce conditions, autograb rules, delay scope) | D20 |
| Starter database | A project-maintained PCD for kinds Dictionarry does not cover | D20 |
| Autograb rule | A library-less rule used by racing mode (later phase) | "filter" (autobrr) |
| Racing mode | Library-less grabs from IRC for ratio building (later phase) | D16 |

#### 5.4 Downloads (owner: `05-downloads.md`)

| Term | Definition | Aliases / notes |
|---|---|---|
| Download | The tracked download: created at grab time before the client is called; holds the release snapshot, targets, indexer, privacy and seed obligations. The source of truth | "tracked download", "queue item" |
| DownloadTarget | One (item, slot) a Download is meant to fill, with its outcome. One Download may have targets in several slots | D20 |
| Phase | What a Download means for the library: `submitting`, `downloading`, `extracting`, `processing` (external processing hold), `importing`, `import_blocked`, `imported`, `failed`, `abandoned` | "state", "status" |
| Retention | What is happening to the payload in the client, independent of phase: `none`, `active`, `seeding`, `stopped`, `removed`, `vanished` | — |
| Download client | An external client (qBittorrent, Transmission, Deluge, rTorrent, SABnzbd, NZBGet, aria2, blackhole) or the built-in DDL downloader | — |
| Observation | The latest polled client state for a Download; observed, never trusted as identity | — |
| Download policy | Data plus a pure function over (download, observation, now) producing actions (fail, remove, blocklist, search) | Cleanuparr "queue rule", "strike" |
| Seed obligation | Ratio and/or time a torrent must seed, from the indexer and seeding rules | — |
| Blocklist | Releases never to grab again, matched globally by release identity (info hash; indexer + guid; title + size) [OPN-110]; permanent by default [OPN-016] | — |
| Staging root | The extraction output directory, on the same filesystem as the library root | A5 |
| SearchIntent | A queued request to search for one (item, slot), with a lane (user, redownload, added, backlog) | distinct from Intent (§5.1) |
| Backlog scheduler | Decides when to search wanted ItemSlots within indexer budgets | — |

#### 5.5 Import (owner: `06-import.md`)

| Term | Definition | Aliases / notes |
|---|---|---|
| Import | Identification, decision and transfer of files into the library, from a Download, a path, or a library scan | — |
| ImportRequest / ImportOutcome | The handoff from downloads to import and its per-target result | — |
| Reason class | The typed class of an import rejection that drives the download's reason policy: `content_blocked`, `permanent`, `overridable`, `transient`, `needs_user`, `not_wanted`, `partial` | import's per-rejection class (Transient, Permanent, NeedsUser) maps onto it |
| Transfer method | `rename`, `reflink`, `hardlink`, `copy`, `in_place` | import.md `Rename`, `Reflink`, … |
| External processing hold | An optional import state that pauses import, emits an event and resumes when an external tool marks the file ready or failed through the API | D20 |
| Probe | Media inspection (ffprobe, TagLib, fpcalc) recorded per file, including subtitle stream flags | — |
| Extra file | A non-media file kept with a MediaFile (subtitles, NFO, artwork) | — |
| Naming scheme | Token templates that render folder and file names per kind and slot | "naming preset" in PCD |

#### 5.6 Metadata and subtitles (owners: `08-metadata.md`, `09-subtitles.md`)

| Term | Definition | Aliases / notes |
|---|---|---|
| Metadata source | One external provider (TVDB, TMDb, MusicBrainz, Hardcover, StashDB, ComicVine, …) behind a per-kind provider interface | "provider" |
| Assembler | Per kind: composes source documents into canonical catalog data | — |
| Change feed | A source's "changed since" capability used by refresh; age-based refresh is the fallback | — |
| Scene mapping | Alternate and scene titles and numbering (XEM, TVDB aliases, community data) used for search and matching | — |
| Subtitle profile | The wanted subtitle languages and flags (HI, forced), assigned per Title and applied to all its items and slots [OPN-049] | Bazarr "language profile" |
| Attribution | Source-required logos, links and disclaimers, built into the provider interface | — |

#### 5.7 Runtime and platform (owner: `12-platform.md`)

| Term | Definition | Aliases / notes |
|---|---|---|
| Node | One running core process. Nodes are stateless; state is in the database | "replica", "instance" |
| Job | A durable unit of background work in the database job queue, claimed with `SKIP LOCKED` on any node | "task", "command" |
| Dedupe key / concurrency key | Keys that give "at most one queued / running job per subject" | — |
| Schedule | A row that produces jobs at due times; every node runs the scheduler | A1 |
| Lease | A named, fenced claim held by one node, only for long-lived connections (IRC, file watcher) | "leader election" (D14 wording) |
| Row claim | A short cross-node single-flight claim taken by conditional update on the shared row it guards (tracker login, challenge solving) | "named mutex"; not an advisory lock |
| Capability-routed job | A job that declares the native tools it needs; only nodes advertising them claim it | A6 |
| Event | A typed domain event; reliable delivery goes through the job queue (outbox) | — |
| Doorbell | Best-effort LISTEN/NOTIFY wake-up and cache invalidation between nodes | A4 |
| Health issue | A persisted, typed problem raised by a health check | — |
| Notification target | A sink for events: user notifications, webhooks, scripts, alert sinks | "connection" (reference apps) |
| Plug-in | A signed WebAssembly module run in wazero with capabilities granted through host functions | D15 |
| Bootstrap config | The config file plus environment/`*_FILE` values for secrets that cannot live in the database (DSN, TLS key, OIDC bootstrap) | A8 |
| Module | A registration unit (a media kind or a provider group) wired in the composition root | — |

#### 5.8 Identity and API (owners: `10-identity-and-access.md`, `11-api-and-clients.md`)

| Term | Definition | Aliases / notes |
|---|---|---|
| User | One table for humans and service accounts | "principal" |
| Identity | A linked login method of a user: password, OIDC issuer, Plex, Jellyfin, Emby | — |
| Credential | One table with kinds `session`, `device`, `api_key` | — |
| Device credential | A revocable, inactivity-expiring credential for a native app, issued by pairing or login | D20, A15 |
| API key | A credential with a required permission subset and optional expiry [OPN-025] | — |
| Permission | `resource:action` | — |
| Role | A named permission set; built in: Owner, Admin, Manager, Member, Requester, Viewer, Service (template) | — |
| Scope | A restriction on a role assignment (media kinds, libraries, own vs any) | — |
| Quota | Per-kind request limit per rolling window, counted in each request's `quota_units` [OPN-021] [OPN-118] | — |
| Pairing | Desktop app ↔ local core handshake over a local socket or named pipe with OS peer credentials | D20 |
| Audit event | A record of a security-relevant action | — |

### 6. Guiding principles

#### 6.1 Redesign, don't replicate (D19)

**OVR-011** (MUST) Every spec file **MUST** specify outcomes and invariants of the reference apps, not their implementations, and **SHOULD** prefer the simplest logically correct design. Workflows that exist because the references are separate processes, grew over a decade or work around each other (sync layers, polling mirrors, text-matching on status messages, multi-instance setups) **MUST NOT** be carried over. _Source: D19._

**OVR-012** (MUST) Text supplied only by external systems (client failure messages, tracker status text) **MUST** be classified into enums inside per-client adapter classifiers with tests; internal state **MUST NOT** depend on text matching. _Source: open-issues A10._

**OVR-013** (MUST) Where a reference app defines a correctness invariant by a test corpus (for example Sonarr and Radarr parser tests), bonnyread **MUST** port the corpus and treat it as the acceptance suite. _Source: D1a; D8._

#### 6.2 No hosted infrastructure; direct sources

**OVR-014** (MUST) All metadata sources, definition feeds and OAuth flows **MUST** be called directly from the core, behind interfaces that allow a hosted cache to be added later. Keys **MAY** be project-registered and shipped, clearly attributed, and every key **MUST** be user-overridable; keys copied from other apps **MUST NOT** be shipped. _Source: D3b; D3c._

**OVR-015** (MUST) Indexer definitions and community datasets without a verified licence **MUST** be downloaded at runtime into the user's database and **MUST NOT** be embedded in the binary or distributed. _Source: D5b; open-issues A2, B12._

#### 6.3 Secure defaults

**OVR-016** (MUST) Authentication **MUST** always be on; there is no "none" mode and no bypass for local addresses. _Source: design/identity-api-deployment.md §1 I1._

**OVR-017** (MUST) Defaults **MUST** be the secure choice. At minimum: credentials only in headers or cookies, never query strings; argon2id password hashing; TLS verification on for outbound calls; no telemetry; the first account cannot be claimed by whoever reaches the port first; the desktop install listens on loopback only until the user enables remote access; forward-auth login off; unsigned plug-ins refused [OPN-066]; adult content hidden unless the scope includes it. _Source: design/identity-api-deployment.md §1 I2–I7, §3.4, §3.10; D20; D17._

**OVR-018** (MUST) Secrets **MUST** be stored in the database, except bootstrap secrets (bootstrap config). Secrets **MUST NOT** be returned to clients after they are saved. Backup archives **MUST** offer optional passphrase encryption, and a clearly labelled sanitised export **MUST** be available. _Source: D16; D20; open-issues A8._

#### 6.4 Licence

**OVR-019** (MUST) All code **MUST** be AGPL-3.0-compatible. Ported Profilarr (AGPL-3.0) and GPL-3.0 code, autobrr's GPL-2.0-or-later code, and MIT/MPL-2.0 dependencies are allowed. Jackett's GPL-2.0-only C# **MUST NOT** be translated; the Cardigann engine is written fresh from the MIT Go Cardigann seed and tested against definitions and fixtures. _Source: D5a; D5b._

**OVR-020** (MUST) A licence scan (`go-licenses`) of the final dependency graph and the legal review in `14-roadmap.md` **MUST** pass before the first public release. _Source: D5; D20._

#### 6.5 Operations

**OVR-021** (MUST) A single node **MUST** remain the default deployment; the same binary **MUST** run active-active behind a load balancer on PostgreSQL. _Source: D14; D18._

**OVR-022** (MUST) Schema changes **MUST** follow expand/contract with N-1 compatibility, and API v1 changes **MUST** be additive only, enforced in CI. _Source: open-issues A7._

### 7. Decision index

| # | Decision | One line |
|---|---|---|
| [D1](../decisions.md#d1-regex-engine) | Regex engine | regexp2 (with timeouts and `\p{IsBlock}` normalisation) for dataset, Cardigann and parser patterns; Sonarr/Radarr cascades are the structural reference |
| [D2](../decisions.md#d2-scope-and-phasing) | Scope and phasing | Apps in §3; P1 TV/movies/indexers/profiles/downloads/requests, P2 subtitles/music, P3 books/adult/comics; model for all kinds from day one |
| [D3](../decisions.md#d3-metadata-sources-and-hosted-service-dependencies) | Metadata sources | TVDB v4 canonical for TV; direct sources per kind; project keys allowed and overridable; no hosted infrastructure in v1 |
| [D4](../decisions.md#d4-multiple-files-versions-per-item) | Versions per item | Version slots from P1, each with its own profile and root folder (amended by D20) |
| [D5](../decisions.md#d5-license) | Licence | AGPL-3.0; fresh Go Cardigann engine; definitions fetched at runtime, never distributed; legal review before release |
| [D6](../decisions.md#d6-frontend-stack) | Frontend | SvelteKit static SPA (Svelte 5) embedded with `go:embed`; API contract from OpenAPI |
| [D7](../decisions.md#d7-external-api-compatibility) | External API compatibility | No compatibility layers; own webhook format; metrics and alerts first-class |
| [D8](../decisions.md#d8-release-parser-foundation) | Release parser | Port golden corpora first; spike `rls`; choose ported cascades or hybrid by measured pass rates |
| [D9](../decisions.md#d9-community-profile-data) | Profile data | PCD-native store, Dictionarry default, keep the on-disk format, new materialising engine (amended by D20) |
| [D10](../decisions.md#d10-adult-model-whisparr-v2-or-v3) | Adult model | Whisparr v3 model with followable Studio/Performer; restore v2 safeguards |
| [D11](../decisions.md#d11-direct-download-ddl-acquisition) | DDL | Optional isolated DDL module in P3, driven by declarative site/host definitions |
| [D12](../decisions.md#d12-clientserver-architecture) | Client–server | Headless core; one SvelteKit UI for web, Tauri desktop and later mobile; desktop installs core as OS service (amended by D20) |
| [D13](../decisions.md#d13-database-and-orm) | Database and ORM | ent with Atlas versioned migrations (amended by D20: PostgreSQL and SQLite only) |
| [D14](../decisions.md#d14-high-availability) | High availability | Active-active stateless nodes, DB job queue, Postgres required for HA (amended by D20) |
| [D15](../decisions.md#d15-plug-in-boundary) | Plug-ins | One WebAssembly boundary on wazero with capability host functions |
| [D16](../decisions.md#d16-secrets-racing-mode-and-self-updater) | Secrets, racing, updater | Secrets in DB; racing mode later; self-updater deferred (amended by D20) |
| [D17](../decisions.md#d17-authentication-and-authorisation) | Auth | OIDC and built-in auth, RBAC default roles; media-server login and scoped API keys on the same model |
| [D18](../decisions.md#d18-deployment-range) | Deployment range | Kubernetes HA to single-user desktop; Tauri for native apps |
| [D19](../decisions.md#d19-porting-principle-redesign-dont-replicate) | Porting principle | Redesign, don't replicate: keep outcomes and invariants, drop accidental workflows |
| [D20](../decisions.md#d20-design-review-outcomes) | Design-review outcomes | Multi-slot downloads, extension schema, attribute authority, MySQL dropped, external processing hold, coverage-aware selection, TVDB fallback, backup encryption, TMDb purge rule, legal scope, no push, peer-credential pairing, HA refinements |

### 8. Reading guide

| File | Area | Phase | Main design sources |
|---|---|---|---|
| `00-overview.md` | Purpose, phases, principles, glossary, decision index | all | decisions.md |
| `01-architecture.md` | Components, module boundaries, data flow between areas | all | design/runtime.md §3.1; all designs §3 |
| `02-domain-model.md` | Catalog, library, slots, monitoring, intents, requests, files (owns shared entities) | all | design/library.md |
| `03-acquisition.md` | Indexers, definitions, search, RSS, IRC, budgets, grab fetch, DDL | P1 (DDL P3) | design/acquisition.md |
| `04-evaluation.md` | Parsing, matching, quality model, custom formats, scoring, selection, delay, autograb | P1 | design/evaluation.md |
| `05-downloads.md` | Download clients, phases, policies, extraction, seeding, blocklist, backlog | P1 | design/downloads.md |
| `06-import.md` | Identification, transfer, naming, extras, scanning | P1 (kinds per phase) | design/import.md |
| `07-profiles.md` | PCD engine, patches, conflicts, git lifecycle, extension schema | P1 | design/profiles.md |
| `08-metadata.md` | Metadata sources, refresh, caching, rate limits, discovery | P1 (kinds per phase) | design/metadata-and-subtitles.md §3.A |
| `09-subtitles.md` | Subtitle profiles, search, providers, sync, upgrades | P2 | design/metadata-and-subtitles.md §3.B |
| `10-identity-and-access.md` | Users, authentication, credentials, roles, quotas, pairing, audit | P1 | design/identity-api-deployment.md §3.1–3.5, §3.9 |
| `11-api-and-clients.md` | REST conventions, OpenAPI, SSE, web and Tauri clients | P1 | design/identity-api-deployment.md §3.6–3.8 |
| `12-platform.md` | Jobs, schedules, leases, events, notifications, health, plug-in host, backup, observability | P1 | design/runtime.md |
| `13-deployment.md` | Topologies, container image, Helm chart, OS services, storage requirements | P1 | design/identity-api-deployment.md §3.10–3.13; design/runtime.md §3.15 |
| `14-roadmap.md` | Phase deliverables, exit criteria, spikes, release gates, deferred items | all | decisions.md; open-issues.md |
| `15-open-items.md` | Unconfirmed defaults, cited as `[OPN-NNN]` | all | open-issues.md §C |

Suggested order: 00, 01, 02, then the pipeline in data-flow order (03 → 04 → 05 → 06), then 07, 08, 09, then 10–13, then 14 and 15.

## Flows

The end-to-end pipeline, specified in detail by the files named:

1. A Title is added (manual, request, list, follow) as an Intent; ItemSlots become monitored (02).
2. The backlog scheduler, RSS, IRC or a user issues a SearchRequest or receives an Announce (05, 03).
3. Releases are parsed, matched and evaluated per Target; coverage-aware selection picks a grab (04).
4. A Download is created with its DownloadTargets and submitted to a client; policies watch it (05).
5. On completion it is extracted if needed and handed to import (05, 06).
6. Import identifies, decides and transfers files into each target slot's root; failures return typed reasons to the download's reason policy (06, 05).
7. Events drive notifications, SSE and metrics (12, 11). Subtitles are searched for new files (09, P2).

## Non-goals

See §2. This file specifies no behaviour of individual areas; each owning file does.

## Traceability

| Reference app | Preserved | Deliberately changed (D19) |
|---|---|---|
| Sonarr, Radarr | Parser corpora, scoring and upgrade semantics, numbering rules | One file per item → slots; 4K instance pairs dropped; no v3 API |
| Prowlarr, Jackett, autobrr | Cardigann definitions, IRC announce parsing | One Indexer with facets; no Arr sync; definitions stored in DB |
| Profilarr, Recyclarr | PCD format, conflict strategies, publishing | No sync into external Arrs; incremental materialisation |
| Cleanuparr, Unpackerr | Stall/slow/unregistered handling, extraction | Duration-based conditions instead of strikes; extraction is a download phase |
| Seerr | Requests, approvals, quotas, media-server login | 4K flags → slot settings; requests are Intents |
| Bazarr | Provider coverage, scoring, sync | Providers as WASM plug-ins; no Bazarr API |
| Lidarr, Bookshelf | Identification distance, metadata profiles | No hosted metadata proxy; ebook/audiobook as slots |
| Whisparr v2/v3 | v3 model; v2 product codes, ambiguity rejection | Direct StashDB/ThePornDB calls |
| Kapowarr | Comic coverage, DDL (optional) | Usenet/torrent via the indexer layer |
