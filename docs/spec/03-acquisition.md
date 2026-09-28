# 03. Acquisition

This file specifies everything that turns external sources into normalized **Release** values and fetches the payload for a release the system decides to grab. Sources are trackers, Usenet indexers, IRC announce channels, tracker APIs and, in phase 3, direct-download (DDL) sites. It covers the Indexer entity and its facets, the driver contract and drivers (Cardigann, native, Newznab/Torznab, generic RSS, WASM, DDL), the Cardigann engine, the definition lifecycle, IRC ingest, search planning and execution, the RSS loop, categories, rate limiting and budgets, health, the tracker HTTP layer (sessions, login, captcha, proxies, FlareSolverr), the grab fetch path, DDL resolution, and the hooks for racing mode. Deciding *what* to search is owned by the library and the backlog scheduler (`05-downloads.md`); deciding *whether* to grab is owned by `04-evaluation.md`; sending payloads to clients is owned by `05-downloads.md`; jobs, leases, schedules, the shared rate limiter and the plug-in host are owned by `12-platform.md`.

**Phases.** P1: indexer layer for TV and movies (Cardigann, Newznab/Torznab, RSS, IRC, P1 natives). P2: music natives and API enrichment for IRC. P3: book, comic and adult natives and categories, DDL. Racing mode: a later phase (D16), with hooks specified here.

---

## 1. Concepts

| Term | Definition |
|---|---|
| **Indexer** | One configured tracker or indexer instance (one row). Several instances may use the same definition. |
| **Facet** | A capability of an indexer that shares its credentials, session, proxy, limits and health: `search`, `rss`, `irc`, `api` (enrichment), `grab`, and `ddl` (P3). Each facet has its own health status. |
| **Driver** | The implementation behind an indexer: `cardigann`, `native`, `newznab`, `torznab`, `rss`, `wasm`, `ddl`. |
| **Definition** | A declarative indexer description: a Cardigann YAML (search/RSS), an autobrr IRC YAML (announce parsing), or a bonnyread DDL site YAML. Definitions are data, fetched at runtime. |
| **Definition feed** | A source of definitions: a Git repository, a path and a ref, stored as a row. |
| **Definition set** | One validated snapshot of a feed at one commit. Exactly one set per feed is *active*. |
| **Release** | The normalized value type describing one downloadable offer from one indexer (§5). The output of acquisition and the input of `04-evaluation.md`. |
| **Release key** | The stable per-indexer identity of a release: the tracker torrent id when extractable, else `sha256(guid)`. Used to dedupe RSS against IRC. |
| **Criteria** | A typed, per-media-kind description of what to search for, built by the caller (§8.1). |
| **Tier** | A group of queries at one precision level (id search, title search). Lower tiers run only if higher tiers returned nothing. |
| **Purpose** | Why a query or grab happens: `interactive`, `automatic`, `rss`, `backlog`, `racing`. It drives budget classes and priorities. Replaces reference apps' User-Agent/reverse-DNS source tracking. |
| **Origin** | Where a release came from: `search`, `rss`, `irc`, `ddl`, `manual`. Alias: `04-evaluation.md` design sketch calls this `Source`. |
| **Kind scope** | Per-indexer, per-media-kind configuration: categories and kind-specific search settings. An indexer serves a kind only if it has a kind scope with categories. |
| **Payload** | The grab result: torrent bytes, magnet, NZB bytes, or a DDL plan. |
| **Tracker alias map** | A bonnyread-authored data file that links a Cardigann id, an autobrr IRC identifier and a native driver id for the same tracker. |
| **Lease holder** | The node holding the platform lease for an IRC network (`12-platform.md`). Design documents that say "leader" for IRC mean this. There is no leader for scheduled work (A1). |
| **Egress** | The network path for tracker traffic: an explicit Proxy, or the default egress. |

Shared entities referenced here and owned by `02-domain-model.md`: MediaKind, Tag, Title, Item, SlotDefinition, ItemSlot. `Download` is owned by `05-downloads.md`. Platform entities owned by `12-platform.md`: job, schedule, lease, plug-in, health issue, rate-limiter tokens and quota buckets, egress proxies, solvers and solver clearances.

---

## 2. Indexer, facets and drivers

### 2.1 Indexer and facets

**ACQ-001** (MUST) Each configured tracker or indexer **MUST** be exactly one Indexer row. Search, RSS, IRC, API enrichment, grab and DDL **MUST** be facets of that row, sharing its settings (including credentials), session, egress, solver, limits and seed hints. There **MUST NOT** be a separate IRC or RSS "indexer" identity for the same tracker. _Source: design/acquisition.md §3.1; D2._

**ACQ-002** (MUST) Each facet **MUST** have its own health status (§11), so a tracker whose announce bot is down can still search. _Source: design/acquisition.md §3.9c._

**ACQ-003** (MUST) Each indexer **MUST** carry independent enable flags for `rss`, `auto_search`, `interactive_search` and `irc`, and a `priority` (1–50, default 25; lower is preferred) that is passed to evaluation on every Release. _Source: design/acquisition.md §3.3._

**ACQ-004** (MUST) Indexer settings **MUST** be validated against the settings schema of the indexer's driver or definition. Settings the schema marks secret (passwords, API keys, passkeys, `rsskey`, `torrent_pass`, cookies) **MUST** be stored in the database (D16), **MUST NOT** be returned by the API in clear, and **MUST** be scrubbed from logs, errors, job logs and history. _Source: design/acquisition.md §3.3, §3.10; D16._

**ACQ-005** (MUST) The IRC facet **MUST** use the same secret settings as search and RSS (one credential store per indexer). _Source: design/acquisition.md §3.8._

**ACQ-006** (MAY) Several indexers **MAY** use the same definition; each has its own settings, session, state and limits. _Source: design/acquisition.md §3.3._

**ACQ-007** (MUST) An indexer **MAY** carry seed hints (`torrent_policy`: minimum seeders, seed ratio, seed time, season-pack seed time, prefer magnet, reject blocked extensions) and a download-client hint. Acquisition **MUST** export them (`SeedPolicy(indexer)`, `ClientHint(indexer)`); enforcement of seed goals and client routing **MUST** be owned by `05-downloads.md`. [OPN-006] _Source: design/acquisition.md §6 Q6–Q7; open-issues C (acquisition)._

**ACQ-008** (MUST) Tags on an indexer **MUST** be usable by the search planner's indexer filter and by `05-downloads.md` client selection. _Source: design/acquisition.md §3.3._

### 2.2 Driver contract

**ACQ-009** (MUST) Every driver **MUST** implement:

| Operation | Contract |
|---|---|
| `Caps(ctx) → Caps` | Supported modes and parameters, categories, page-size limits, paging support, flags, redirect support. |
| `Query(ctx, Query, PageReq) → Page` | Both search and recent listing (`Mode = Recent`). Returns normalized Releases, never XML. |
| `Fetch(ctx, *Release) → Payload` | Resolve and download the grab payload (§13). |
| `Test(ctx) → error` | Login if needed, one recent query, and the definition's download test if it declares one. |

_Source: design/acquisition.md §3.4._

**ACQ-010** (MUST) A driver **MAY** additionally implement `Enricher.Enrich(ctx, *Release, FieldSet)` (API facet or .torrent metainfo) and, for DDL, `LinkResolver.Resolve(ctx, *Release) → DDLPlan`. _Source: design/acquisition.md §3.4._

**ACQ-011** (MUST) Drivers **MUST** be stateless over a per-indexer tracker HTTP client (§12). Sessions, cookies, limiter state and health **MUST** live in the tracker HTTP layer and the database, so any node can run any indexer. _Source: design/acquisition.md §3.4; D14._

**ACQ-012** (MUST) Drivers **MUST NOT** parse release titles. Acquisition passes the raw title plus source metadata; parsing is owned by `04-evaluation.md` (D1, D8). _Source: design/acquisition.md §3.3._

**ACQ-013** (MUST) The driver set is:

| Driver | Phase | Requirements |
|---|---|---|
| `cardigann` | P1 | §3 |
| `native` | P1–P3 | §2.3 |
| `newznab`, `torznab` | P1 | §2.4 |
| `rss` | P1 | §2.5 |
| `wasm` | P1 (host), as plug-ins appear | §2.6 |
| `ddl` | P3 | §16 |

_Source: design/acquisition.md §3.4._

### 2.3 Native drivers

**ACQ-014** (MUST) Native drivers **MUST** be written from API documentation, recorded HTTP fixtures and observed behaviour, never by translating C# from Prowlarr, Jackett or the Arr apps (D5). autobrr's `pkg/{btn,ggn,ops,red,ptp}` Go packages **MAY** be reused directly (GPL-2.0-or-later, D5a). _Source: design/acquisition.md §3.13; D5._

**ACQ-015** (MUST) A native driver **MUST** declare `Supersedes []string` (Cardigann ids it replaces). This declaration **MUST** replace any hard-coded list of superseded definition ids. _Source: design/acquisition.md §2.2, §3.12._

**ACQ-016** (MUST) Native drivers **MUST** be delivered in this order:

| Phase | Natives |
|---|---|
| P1 | BroadcasTheNet, PassThePopcorn, HDBits, BeyondHD, AnimeBytes, FileList, the Avistaz family; IPTorrents and TorrentDay only if the Cardigann definitions are inadequate |
| P2 | Redacted, Orpheus, the Gazelle family, with `Enricher` for IRC |
| P3 | MyAnonamouse |

_Source: design/acquisition.md §3.13; D2._

### 2.4 Newznab and Torznab

**ACQ-017** (MUST) The Newznab/Torznab driver **MUST** discover caps with `t=caps`, cache them for 24 h in `IndexerState.caps`, and refresh them on settings edit and on `Test`. Users **MUST** be able to override categories and supported parameters. _Source: design/acquisition.md §3.4; O3._

**ACQ-018** (MUST) Paging **MUST** use `offset`/`limit` within the caps' limits. Newznab error codes **MUST** be classified by a tested classifier in the driver into `auth`, `rate_limited` or `transient` (§11.2). _Source: design/acquisition.md §3.4; A10._

**ACQ-019** (MUST) Torznab attributes (`seeders`, `peers`, `infohash`, `magneturl`, `downloadvolumefactor`, `uploadvolumefactor`, `minimumratio`, `minimumseedtime`, id attributes) and Newznab attributes (`poster`, `group`, `usenetdate`, id attributes) **MUST** be mapped into Release fields (§5). The XML types **MAY** come from `autobrr/pkg/{newznab,torznab}`. _Source: design/acquisition.md §3.2._

**ACQ-020** (SHOULD) Newznab/Torznab presets (known public endpoints and default settings) **SHOULD** be a data file, not code. _Source: design/acquisition.md §3.4._

### 2.5 Generic RSS

**ACQ-021** (MUST) The `rss` driver **MUST** support generic torrent and Usenet RSS feeds, including cookie-authenticated feeds (IPT/TL style), with `Caps.Modes = {Recent}` only. Size and enclosure extraction heuristics **MUST** follow `autobrr/internal/feed/rss.go`. _Source: design/acquisition.md §3.4; O4._

### 2.6 WASM indexer drivers

**ACQ-022** (MUST) A WASM indexer plug-in **MUST** export `caps`, `query(q, page)` and `fetch(release)`, and **MAY** export `enrich` and `resolve`. It is loaded by the plug-in host in `12-platform.md` (D15). _Source: design/acquisition.md §3.16._

**ACQ-023** (MUST) For an indexer plug-in, the host's `http_do` (`12-platform.md` PLT-147) **MUST** go through that **indexer's** tracker HTTP client (§12), so rate slot, budgets, session, egress, solver, classification and redaction apply. Plug-ins **MUST** additionally be restricted to the hosts declared in their manifest. _Source: design/acquisition.md §3.16; runtime.md §3.12._

**ACQ-024** (MUST) The host **MUST** give indexer plug-ins `session_get`/`session_set`, `settings_get` (secrets passed by handle and never logged), `log`, `regex_match` (regexp2 with timeout) and `html_select`, as listed in `12-platform.md` PLT-147. _Source: design/acquisition.md §3.16._

**ACQ-025** (MUST NOT) IRC connections **MUST NOT** be pluggable; long-lived sockets stay in the core. Custom announce **parsers** (a pure function from announce lines to variables) **MAY** be plug-ins. _Source: design/acquisition.md §3.16._

---

## 3. Cardigann engine

### 3.1 Provenance and licence

**ACQ-026** (MUST) The Cardigann engine **MUST** be a fresh Go implementation, seeded only from the MIT-licensed files authored by the original Go Cardigann's author. It **MUST NOT** include `cardigann/go-imdb-scraper` (no licence) and **MUST NOT** depend on `cardigann/releaseinfo` (title parsing belongs to `04-evaluation.md`). `bcampbell/fuzzytime` (GPL-3.0-or-later) **MAY** be used for date parsing. _Source: D5b; decisions.md D5 option A._

**ACQ-027** (MUST) The engine **MUST** be extended to the Prowlarr v11 schema by testing its behaviour against the definitions and recorded HTTP fixtures, never by translating Prowlarr or Jackett C#. _Source: D5b._

### 3.2 Schema coverage

**ACQ-028** (MUST) The engine **MUST** implement the union of Prowlarr v11 and Jackett behaviour, including `replaces`, `allowtvsearchimdb`, `legacylinks`, `requestDelay`, `categorymappings`, `caps.modes`, `settings`, `login` (all methods, §12.2), `search` (paths, inputs, rows, fields, filters, preprocessing), `download` (`selectors`, `before`, `pathselector`, `infohash`, headers) and `login.test`/`login.error`. Unknown keys **MUST** produce warnings, not failures (lenient). _Source: design/acquisition.md §3.4, §3.12._

**ACQ-029** (MUST) Where Prowlarr and Jackett differ in behaviour, the engine **MUST** follow Jackett's semantics for `.Today.Year`, checkbox settings and template evaluation (§3.3), and Prowlarr's for `replaces` and schema versioning. _Source: design/acquisition.md §3.4; jackett.md §3.1.4, §11.2._

### 3.3 Template interpreter

**ACQ-030** (MUST) Templates **MUST** be executed by a **purpose-built interpreter** for the Jackett-style template language, not Go's `text/template`. It **MUST** reproduce Jackett semantics for: missing keys, `.True`/`.False` as strings, the return values of `and`/`or`, `if`/`else`/`range`, `eq`/`ne`, `join`, and `re_replace`. _Source: A19; design/acquisition.md §6 I6._

**ACQ-031** (MUST) Templates **MUST** be parsed and compiled when a definition is validated (§4.3), and a template that fails to compile **MUST** make that definition file invalid. _Source: design/acquisition.md §3.12._

**ACQ-032** (MUST) A template runtime error on a live response **MUST** be classified `definition` (§11.2). _Source: design/acquisition.md §3.9c._

### 3.4 Regular expressions

**ACQ-033** (MUST) Cardigann `regexp` and `re_replace` filters and the `re_replace` template function **MUST** run **as written** on `regexp2`, after the pattern-normalisation step that rewrites .NET named Unicode blocks (`\p{IsBlock}`) to code-point ranges. `validate` and `andmatch` are not regexes and **MUST NOT** be compiled as such. _Source: D1b._

**ACQ-034** (MUST) Every regexp2 match in the engine **MUST** run with a timeout. The default **MUST** be 100 ms per match. A timeout **MUST** fail that field extraction and increment `bonnyread_regex_timeouts_total{source="cardigann"}` (`12-platform.md` PLT-134); a row whose required field times out is dropped from the page. [OPN-101] _Source: D1; evaluation.md §3.8 (100 ms)._

### 3.5 Golden suite

**ACQ-035** (MUST) CI **MUST** run a golden suite that validates every definition file of the pinned commit of each Cardigann feed and **renders every template in every definition** against recorded HTTP fixtures, asserting the extracted Releases. This is the top fidelity risk of the project (A19); a regression **MUST** fail the build. _Source: A19; design/acquisition.md §6 I6._

**ACQ-036** (MUST) The golden suite **MUST** fetch definitions by commit at test time. Definition YAML **MUST NOT** be committed to the bonnyread repository or included in any build artifact (§4.1). Recorded fixtures **MUST** be scrubbed of credentials and personal data. _Source: D5b._

**ACQ-037** (SHOULD) A scheduled CI job **SHOULD** run the same suite against the head of each feed's followed ref and report newly failing definitions, so engine gaps are found before users receive a new set. _Source: design/acquisition.md §3.12 (derived)._

---

## 4. Definition lifecycle

### 4.1 Distribution constraint

**ACQ-038** (MUST NOT) Cardigann definitions (Prowlarr, Jackett) **MUST NOT** be embedded in the binary, bundled in installers or container images, mirrored, or otherwise distributed by the project. They **MUST** be downloaded at runtime by each installation. _Source: D5b._

**ACQ-039** (MUST) Fetched definition bodies **MUST** be stored in the database (`Definition.body`), not in a node-local data directory. Nodes **MAY** keep a disposable local cache keyed by sha256. _Source: A2; D20 (D14 amended)._

**ACQ-040** (MUST) The legal review required before the first public release (D5b, B12) **MUST** cover, for acquisition:

| Item | Reason |
|---|---|
| Fetched definitions stored in the user's database and therefore in the user's backups | A2, B12 |
| Runtime fetches from the Prowlarr/Indexers repository, which has no licence file | design/acquisition.md §6 I2; B12 |
| Jackett definitions as GPL-2.0-only, and the optional Jackett overlay feed | D5 context |
| Reproducing the v11 schema (filter names and semantics authored in Jackett) in a fresh engine | D5 recommendation |
| Licensing of third-party WASM indexer and DDL host plug-ins (combined work) | B12 |
| The embedded autobrr IRC baseline (GPL-2.0-or-later, used under GPL-3.0 terms) | D5a, §4.2 |

_Source: D5b; B12; D20._

### 4.2 Feeds

**ACQ-041** (MUST) Each definition feed **MUST** be a `DefinitionFeed` row (repository URL, path, kind, `schema_version`, ref, follow flag, precedence), so a new feed never needs code. _Source: design/acquisition.md §3.12._

**ACQ-042** (MUST) The default feeds **MUST** be:

| Feed | Kind | Default | Ref and path |
|---|---|---|---|
| `prowlarr-indexers` | cardigann | **on, primary** | follow `master`, `definitions/v11`, fetched from GitHub by commit (not `indexers.prowlarr.com`) |
| `jackett` | cardigann | off (opt-in overlay, later phase) | follow `master`, definitions path; no schema version, so every file is gated by validation |
| `autobrr` | irc | on | follow `develop`, `internal/indexer/definitions` and `deprecated/` |
| `bonnyread-ddl` | ddl | off until P3 | bonnyread's own repository (§16) |

[OPN-001] _Source: design/acquisition.md §3.12, §6 Q1, I2; open-issues C (acquisition)._

**ACQ-043** (MUST) The binary **MAY** embed a baseline of autobrr IRC definitions (GPL-2.0-or-later, D5a). On first boot the baseline **MUST** be imported as a definition set of the `autobrr` feed and then updated through the normal lifecycle; a fetched set always supersedes the baseline. [OPN-002] _Source: design/acquisition.md §6 Q2; open-issues C (acquisition)._

### 4.3 Update cycle

**ACQ-044** (MUST) Each feed **MUST** have a schedule row (`12-platform.md`) that enqueues a `definitions.update:<feed>` job daily (dedupe `WhilePendingOrRunning`), plus an on-demand "check now" action. The job **MAY** run on any node. _Source: design/acquisition.md §3.12; A1._

**ACQ-045** (MUST) The update job **MUST**:
1. resolve the feed's ref to a commit, and stop if the commit equals the active set's;
2. download the tarball for **that exact commit** and extract only the feed path;
3. validate each file (ACQ-046);
4. store the result as a `staged` DefinitionSet with bodies keyed by sha256;
5. compute a diff against the active set (added, changed, removed, renamed via `replaces`);
6. activate the set atomically (§4.4).

_Source: design/acquisition.md §3.12._

**ACQ-046** (MUST) Validation of each file **MUST** comprise: strict YAML parse (unknown keys are warnings); the **schema-version gate**; compiling every regex with regexp2 after the `\p{IsBlock}` rewrite; compiling every template; and, for IRC definitions, running the definition's own `tests:` sample lines through its patterns. Per-file errors **MUST** be stored in the set's report. _Source: design/acquisition.md §3.12; O31, O32; D1._

**ACQ-047** (MUST NOT) The engine **MUST NOT** load a definition whose schema version is not in its `Supported` set. For P1 `Supported = {11}`. For feeds without a declared schema version (Jackett), each file **MUST** pass validation against the engine's schema. _Source: O31._

**ACQ-048** (MUST) A file that fails validation in a new set **MUST NOT** become active. Indexers using that id **MUST** stay on the previously active valid body, and a health warning **MUST** name the id and the error. [OPN-102] _Source: design/acquisition.md §3.12 (derived)._

**ACQ-049** (MUST) When an upstream schema bump is announced (for example v12), the feed **MUST** keep reading the supported version's path while upstream maintains it. When that path stops receiving commits, a health notice **MUST** state that definitions are frozen at the supported version and an update of bonnyread is needed. _Source: design/acquisition.md §3.12._

### 4.4 Activation, rollback and pins

**ACQ-050** (MUST) Activation **MUST** be one transaction that switches `DefinitionFeed.active_set` and marks the previous set `retired`. Indexers following the feed **MUST** pick up the new body on their next use; no request may see a mix of two sets for one indexer. _Source: design/acquisition.md §3.12._

**ACQ-051** (MUST) After activation, in the same transaction:
- `replaces` **MUST** rewrite `Indexer.definition_id` from the old id to the new one;
- `legacylinks` **MUST** rewrite `Indexer.base_url` when it matches a legacy link;
- a removed definition **MUST** leave its indexers on the last-known body, with a "definition removed" health warning;
- an IRC tombstone (`reason`, `alias_of`) **MUST** raise a health item offering a one-click rebind to `alias_of`.

_Source: design/acquisition.md §3.12; O25, O30, O32._

**ACQ-052** (MUST) The last 3 sets per feed **MUST** be retained. A "revert to previous set" action per feed **MUST** re-activate a retained set atomically. _Source: design/acquisition.md §3.12._

**ACQ-053** (MUST) An indexer **MAY** pin its definition to a body sha256 (`definition_pin`). A pinned body **MUST** be retained while any indexer references it, regardless of set retention. `IndexerState.definition_used` **MUST** record the body in use and drive a "definition changed" notice. _Source: design/acquisition.md §3.3, §3.12._

**ACQ-054** (MUST) Definition resolution for an id **MUST** follow: custom definition > native driver `Supersedes` > primary feed > secondary feed. _Source: design/acquisition.md §3.12; O30._

**ACQ-055** (MUST) Users **MUST** be able to author custom definitions (Cardigann, IRC, DDL YAML) in a UI editor with a test runner. Custom definitions **MUST** pass the same validation, are stored in `CustomDefinition`, and take highest precedence. _Source: design/acquisition.md §3.12; O30._

**ACQ-056** (MUST) Activation **MUST** emit `acquisition.definitions_updated{feed, commit, added, changed, removed}` and update `bonnyread_acquisition_definition_set_info{feed,commit}`. _Source: design/acquisition.md §3.17._

### 4.5 Tracker alias map

**ACQ-057** (MUST) A `TrackerAlias` map **MUST** link a Cardigann id, an autobrr IRC identifier and a native id for the same tracker, together with `torrent_id_extractors` (regexes over GUID or info URL that yield the tracker torrent id). Name equality **MUST NOT** be assumed. _Source: design/acquisition.md §3.12._

**ACQ-058** (MUST) The alias map **MUST** be bonnyread-authored data, curated in the repository with CI checks against the upstream id lists of each feed, and **MAY** ship in the binary (it is not upstream definition content). [OPN-003] _Source: design/acquisition.md §6 Q3; open-issues C (acquisition)._

**ACQ-059** (SHOULD) When a Cardigann or native indexer is added and the alias map links it to an IRC definition, the UI **SHOULD** offer to enable the IRC facet. _Source: design/acquisition.md §3.12._

---

## 5. Release (output type)

**ACQ-060** (MUST) Every driver, the RSS loop, IRC ingest and DDL **MUST** emit Releases with the fields below. Acquisition owns this type; `04-evaluation.md` consumes it.

| Field | Type | Notes |
|---|---|---|
| `id` | ULID | Unique per observation |
| `indexer_id` | id | |
| `indexer_priority` | int | Copied from the indexer at emission |
| `origin` | enum `search`, `rss`, `irc`, `ddl`, `manual` | |
| `purpose` | enum `interactive`, `automatic`, `rss`, `backlog`, `racing` | |
| `search_id` | ULID, nullable | The SearchRequest that produced it |
| `guid` | string | As published |
| `key` | string | Release key (§1) |
| `title` | string | Raw, unparsed |
| `protocol` | enum `torrent`, `usenet`, `ddl` | |
| `size` | int64, nullable | Null = unknown (never 0 for unknown) |
| `published_at`, `first_seen_at` | timestamp | |
| `categories` | list of CategoryRef | Tracker-native keys plus mapped standard ids (§10) |
| `kind_hints` | list of MediaKind | From categories × kind scopes |
| `info_url` | string | |
| `download` | DownloadRef | `url`, `magnet`, `nzb_url` or `link_groups`; may contain secrets |
| `info_hash` | string, nullable | Lower-case hex v1 infohash |
| `torrent` | TorrentAttrs, nullable | seeders, leechers, grabs, files, minimum ratio, minimum seed time |
| `usenet` | UsenetAttrs, nullable | poster, group, usenet date |
| `ids` | ExternalIDs | imdb, tmdb, tvdb, tvmaze, trakt, douban, mbid, stash, isbn, product code |
| `languages` | list of language codes | Indexer-declared only |
| `flags` | IndexerFlags | Below |
| `announce` | AnnounceMeta, nullable | uploader, origin, announce type, tags, record label, log score, cue, bonus, raw variables |
| `extra` | map string→string | author, book title, artist, album, genre, year, poster |

_Source: design/acquisition.md §3.3; evaluation.md §3.3._

**ACQ-061** (MUST) `flags` **MUST** be the canonical IndexerFlags shared with `04-evaluation.md`: `download_factor` and `upload_factor` (floats, 1.0 = normal), `internal`, `scene`, `nuked`, `has_subtitles`, `golden`, `approved`, `exclusive`, and open `tracker_tags`. Drivers **MUST** normalise Torznab volume factors, native API flags and IRC `freeleech`/`freeleechPercent` into it (freeleech percent = `(1 − download_factor) × 100`). _Source: evaluation.md §3.7; O29._

**ACQ-062** (MUST NOT) `download` **MUST NOT** be serialized to any API response, event, webhook or log. _Source: design/acquisition.md §3.3._

**ACQ-063** (MUST) A Release **MUST** be persisted only in `SearchResult` (§8.4), `ReleaseDiscovered` (§9.3) and evaluation's own tables. _Source: design/acquisition.md §3.3._

---

## 6. Rate limiting and budgets

### 6.1 Transport spacing

**ACQ-064** (MUST) Every tracker request **MUST** first take a rate slot from the shared DB-backed rate limiter (`12-platform.md`, A3) under the key `acq:indexer:<id>`. The minimum interval **MUST** be the user override if set, else the definition's `requestDelay`, else 2 s for natives, else the driver default. The slot **MUST** hold across nodes. On SQLite single-node the in-process limiter backend is used with the same semantics. _Source: A3; O19; design/acquisition.md §3.9a._

### 6.2 Budgets

**ACQ-065** (MUST) Per-indexer query and grab budgets (count per hour or per day, user-set) **MUST** be enforced through the shared rate limiter's quota buckets (A3). RSS polls, searches of every purpose, enrichment requests that hit the tracker, and grabs **MUST** all be counted. _Source: A3; O18; design/acquisition.md §3.9b._

**ACQ-066** (MUST) Query budgets **MUST** be class-aware:

| Class | Allowance |
|---|---|
| `interactive` | May use the whole remaining budget (reserved lane, A3) |
| `rss` | Reserved `ceil(window / rss_interval)` polls that no other class may consume |
| `automatic` | Remaining budget after the RSS reserve |
| `backlog` | At most `limits.backlog_share` (default 50%) of what remains after the RSS reserve |
| `racing` | Counts against the grab budget like any grab (§15) |

[OPN-004] _Source: design/acquisition.md §3.9b, §6 Q4; A3._

**ACQ-067** (MUST) A denial **MUST** return `ErrBudget{retry_at}`, computed from the bucket that will next free capacity. Interactive callers **MUST** be shown `retry_at`. _Source: O18; design/acquisition.md §3.9b._

**ACQ-068** (MUST) Acquisition **MUST** expose `Budget.Available(indexer, class) → (n, reset_at)` for the backlog scheduler in `05-downloads.md`. _Source: design/acquisition.md §3.17; downloads.md §3.12._

**ACQ-069** (MAY) Under concurrency the budget **MAY** over-run by at most one request per node. _Source: design/acquisition.md §3.15._

**ACQ-070** (MUST) Usage statistics (queries per class, grabs, per hour) **MUST** be computed by SQL aggregates over limiter buckets, never by loading history into memory, and **MUST** be exported as `bonnyread_acquisition_budget_remaining{indexer,class}`. _Source: design/acquisition.md §2.2, §3.17._

---

## 7. Capabilities and kind scopes

**ACQ-071** (MUST) `Caps` **MUST** record, per mode (`search`, `tv`, `movie`, `music`, `book`, `recent`), the supported parameters from: `q, season, ep, imdbid, tvdbid, tmdbid, tvmazeid, traktid, doubanid, artist, album, label, track, year, genre, author, title, publisher`. It **MUST** also record `supports_raw_search`, page-size limits, the category tree and supported flags. `rid` (TVRage) **MUST NOT** be supported. _Source: design/acquisition.md §3.5._

**ACQ-072** (MUST) Caps for Cardigann and native drivers **MUST** be derived from the definition or code; Newznab/Torznab caps from `t=caps` (§2.4). _Source: design/acquisition.md §3.3._

**ACQ-073** (MUST) Each indexer **MUST** hold a kind scope per media kind it serves: categories plus kind-specific search settings. Defaults **MUST** be the definition's category mappings intersected with the kind's default range; users **MAY** override them. Kind-specific settings **MUST** include at least:

| Kind | Settings |
|---|---|
| TV, anime | `season_search_max_single_episode_age`, `anime_standard_format_search` |
| Adult | date formats; ordered query templates (studio+date, studio+title, code, title-only, …) |

_Source: design/acquisition.md §3.3, §3.5; O8, O17; D10._

**ACQ-074** (MUST) Adult search settings **MUST** be per indexer, not global. _Source: D10._

---

## 8. Search

### 8.1 Request and criteria

**ACQ-075** (MUST) Acquisition **MUST** expose `Search(ctx, SearchRequest) → SearchReport`, where `SearchRequest = {criteria, purpose, indexer_filter (ids, tags, protocol), per_indexer_deadline}`. Default deadlines: 30 s interactive, 60 s otherwise. _Source: design/acquisition.md §3.6._

**ACQ-076** (MUST) Criteria **MUST** be a sum type with one variant per shape:

| Variant | Fields |
|---|---|
| `TvEpisode` | ids, titles[] (series, scene and alternate titles, per-season scene names), season, episodes[], absolute[], air_date, series_type, is_special, special_titles[] |
| `TvSeason` | ids, titles[], season, episodes with air dates, series_type |
| `Movie` | ids, titles[], year |
| `MusicAlbum` | artist, album, year, disambiguation |
| `MusicArtist` | artist |
| `Book` | author, title, isbn (optional) |
| `AdultScene` | studios[] with aliases, date, title, code, performers[] |
| `AdultMovie` | ids, title, year |
| `ComicIssue` | volume, issue, year |
| `FreeText` | q, categories |

The caller builds criteria, including scene/XEM numbering and alternate titles; acquisition decides how each indexer is asked. _Source: design/acquisition.md §3.6._

### 8.2 Planner

**ACQ-077** (MUST) The planner **MUST** be a pure function of `(criteria, indexer caps, kind scope)` that returns an ordered list of tiers, each a set of queries, and **MUST** be unit-testable against recorded caps. _Source: design/acquisition.md §3.6._

**ACQ-078** (MUST) An indexer **MUST** be selected only if the facet is enabled for the purpose, a kind scope with categories exists for the criteria's kind, the facet is not in backoff or a terminal state, and caps support at least one usable mode. An unselected indexer **MUST** be skipped, not errored, with a reason in the report. _Source: O10; design/acquisition.md §3.6._

**ACQ-079** (MUST) TV planning **MUST** produce: tier 1 an id search in the order tvdbid → imdbid → tvmazeid → tmdbid, as far as caps allow; tier 2 a title search over the `titles[]` variants. Daily series **MUST** use `q="<title> yyyy.MM.dd"`. Anime **MUST** search absolute numbers with anime categories, and **MUST** add S/E queries when `anime_standard_format_search` is set. Specials **MUST** be a title search on special titles. _Source: O7, O8; design/acquisition.md §3.6._

**ACQ-080** (MUST) A `TvSeason` criteria **MUST** be split into episode queries when any episode aired more recently than the indexer's `season_search_max_single_episode_age`; otherwise it **MUST** be a season query. _Source: O8._

**ACQ-081** (MUST) Movie planning **MUST** be imdbid, then tmdbid, then title+year. Music **MUST** use `t=music&artist&album`, else `q="artist album [year]"`. Books **MUST** use `t=book&author&title`, else `q=`. Adult **MUST** use the indexer's ordered template list. Comics **MUST** use `q="volume issue"` variants. _Source: O9; D10; design/acquisition.md §3.6._

**ACQ-082** (MUST) Identical queries across tiers **MUST** be sent once. _Source: design/acquisition.md §3.6._

### 8.3 Executor

**ACQ-083** (MUST) Indexers **MUST** run in parallel. Within an indexer, tiers **MUST** run in order and stop at the first tier that returned any release; pages **MUST** run in order. _Source: O7._

**ACQ-084** (MUST) Every page **MUST** take a budget ticket (§6.2) and then a rate slot (§6.1). Paging **MUST** continue until a page is not full, `max_results` (default 1,000) is reached, or the deadline expires. _Source: O11; design/acquisition.md §3.6._

**ACQ-085** (MUST) Identical in-flight queries **MUST** be coalesced per `(indexer, canonical query)` on a node, and a per-node result memo of 10 minutes **MUST** serve repeated identical queries. The memo **MUST NOT** serve `rss` purpose queries. [OPN-004] _Source: design/acquisition.md §3.6._

**ACQ-086** (MUST) Results **MUST** be post-filtered against the expanded category set of the query; uncategorised results **MUST** be kept. _Source: O11._

**ACQ-087** (MUST) Releases lacking a title, size or category **MUST** be dropped from `automatic` and `backlog` searches and kept for `interactive` ones. _Source: O12._

**ACQ-088** (MUST) Within one indexer, releases with the same GUID **MUST** be deduplicated. Across indexers, releases with the same `info_hash` (torrent), or the same normalised title and size (Usenet), **MUST** be **grouped, not dropped**; the group is passed to evaluation, which chooses by indexer priority and seeders. _Source: design/acquisition.md §3.6._

**ACQ-089** (MUST) The executor **MUST** return `SearchReport{releases, per_indexer: {outcome (ok, skipped, error, budget_denied), count, skip_reason, error_class, retry_at}}`. _Source: design/acquisition.md §3.6._

**ACQ-090** (MUST) On completion of a search with a `search_id`, acquisition **MUST** emit the in-process `SearchCompleted{search_id}` signal consumed by the backlog scheduler (`05-downloads.md`). _Source: downloads.md §3.12._

### 8.4 Interactive results

**ACQ-091** (MUST) Interactive search results **MUST** be written to `SearchResult` with a 60-minute TTL, so a grab issued to any node can load the release. [OPN-004] _Source: design/acquisition.md §3.6._

**ACQ-092** (MUST) Interactive results **MUST** be streamed to the UI per indexer as each indexer completes, through the event stream (`11-api-and-clients.md`), rather than after the slowest indexer. _Source: design/acquisition.md §3.6._

---

## 9. RSS

### 9.1 Scheduling

**ACQ-093** (MUST) Each RSS-enabled indexer **MUST** own one schedule row (`12-platform.md`), created, updated and deleted in the same transaction as the indexer, that enqueues `rss:indexer:<id>` with dedupe `WhilePendingOrRunning`. There **MUST** be no leader for RSS scheduling or execution; any node may run the job. _Source: A1; runtime.md §3.5.1._

**ACQ-094** (MUST) The interval **MUST** be `max(indexer override or global default 15 min, definition feed.minInterval, 10 min)`, with the scheduler's deterministic per-schedule jitter. The bound **MUST** be validated at write time. _Source: O14; runtime.md §3.5.1._

**ACQ-095** (MUST) While an indexer's IRC facet is `ok`, its RSS interval **MUST** be multiplied (default ×4). When the IRC facet leaves `ok`, or its network lease changes holder, an immediate RSS catch-up job **MUST** be enqueued (priority `high`) for every indexer bound to that network. [OPN-004] _Source: design/acquisition.md §3.7, §3.15._

### 9.2 Execution and cursor

**ACQ-096** (MUST) An RSS job **MUST** issue `Query{mode: recent, categories: union of all kind scopes}` — one poll per indexer regardless of how many media kinds or consumers use it. _Source: design/acquisition.md §3.7._

**ACQ-097** (MUST) Pages **MUST** be fetched until a page contains the cursor's key or GUID, the oldest release on the page is older than the cursor's publish date, or 1,000 releases or 24 h of history have been read. _Source: O13._

**ACQ-098** (MUST) If the poll stopped at the cap without reaching the cursor, a **gap** **MUST** be recorded in `IndexerState.rss_last_gap`, a health warning raised, and `acquisition.rss_gap{indexer, from, to}` emitted. The backlog scheduler **MAY** react with targeted searches. _Source: O13; design/acquisition.md §3.7._

### 9.3 Exactly-once hand-off

**ACQ-099** (MUST) Inside one fenced job transaction (`12-platform.md`), the RSS job **MUST** insert every release key into `RssSeen(indexer, key)` with `ON CONFLICT DO NOTHING`, keep only the newly inserted releases, enqueue one `evaluate_releases` job with them, and advance `rss_cursor`. A crashed or stale worker therefore neither loses nor duplicates releases: each `(indexer, key)` reaches evaluation exactly once. _Source: O15; design/acquisition.md §3.7._

**ACQ-100** (MUST) `RssSeen` **MUST** be shared by RSS and IRC. `RssSeen` rows **MUST** be purged after 14 days. _Source: O15; design/acquisition.md §3.3._

**ACQ-101** (MUST NOT) The RSS job **MUST NOT** re-evaluate delayed or pending releases; delay is evaluation's persisted timer. _Source: design/acquisition.md §2.2._

**ACQ-102** (MUST) For each new release a durable `ReleaseDiscovered` record **MUST** be written for audit and the UI, with a retention of 14 days. _Source: design/acquisition.md §3.8._

---

## 10. Categories

**ACQ-103** (MUST) Internally, categories **MUST** be keyed by **tracker-native keys** scoped to the definition (`CategoryRef{tracker_key}`), optionally with the standard Newznab id the definition maps them to (`CategoryRef{std}`). The `100000 + id` and `100000 + uint16(SHA1)` custom-id scheme **MUST NOT** be used internally. _Source: A16._

**ACQ-104** (MUST) The standard Newznab category tree (76 ids) **MUST** be used for kind defaults and parent-to-child expansion. Tracker categories **MUST** map onto it through the definition's `categorymappings`. _Source: O16._

**ACQ-105** (MUST) Default categories per kind **MUST** be: TV 5000s, anime 5070, movies 2000s, audio 3000s, books 7000s, comics 7030, adult 6000s for **every** driver type. Users **MAY** override them per indexer. _Source: O17; D10._

**ACQ-106** (MUST) A query **MUST** send the kind's categories expanded parent → children and mapped to tracker keys. `kind_hints` on each Release **MUST** be derived from its categories and the indexer's kind scopes. _Source: design/acquisition.md §3.5._

**ACQ-107** (MAY) The `100000+` derivation **MAY** exist only inside a Prowlarr configuration importer, if one is built. [OPN-005] _Source: A16; design/acquisition.md §6 Q5._

---

## 11. Health and error classes

### 11.1 Facet status

**ACQ-108** (MUST) Each facet status **MUST** be one of `ok`, `backoff`, `auth_failed`, `needs_user`, `definition_error`, `disabled`, with `escalation`, `disabled_until`, `last_error` (class and redacted snippet), `last_success`. A success **MUST** reset the facet to `ok`. _Source: design/acquisition.md §3.3, §3.9c._

**ACQ-109** (MUST) Facet status changes **MUST** emit `acquisition.indexer_health{indexer, facet, from, to, class}` and feed the health and alert sinks (`12-platform.md`, D7). A facet failing for more than 6 h **MUST** raise a stronger "long-term failing" health item. _Source: O21; design/acquisition.md §3.9c._

**ACQ-110** (MUST) Health checks **MUST** cover: indexer facet failing and long-term failing; no RSS-capable indexer; no search-capable indexer for a monitored kind; definition removed, invalid, or tombstoned; definitions frozen at the supported schema; proxy failing; solver failing; IRC network disconnected; RSS gap. _Source: O21._

### 11.2 Error classification

**ACQ-111** (MUST) The tracker HTTP layer **MUST** classify every failure into exactly one class, with these effects:

| Class | Examples | Effect |
|---|---|---|
| `transient` | 5xx, timeout, DNS or connect failure | Backoff escalates `0, 1m, 5m, 15m, 30m, 1h, 3h, 6h, 12h, 24h` |
| `rate_limited` | 429, Newznab or tracker "limit reached" | Disabled until `Retry-After`, else 1 h; no escalation step |
| `auth` | Login failed, `login.error` matched, 401/403 after re-login | One automatic re-login attempt, then `auth_failed`; no escalating retries; alert; user action required |
| `needs_user` | Image captcha, expired user-pasted cookie | `needs_user` with the challenge stored; resumes when the user answers |
| `challenge` | Cloudflare or DDoS-Guard detected | Solver path (§12.3); `transient` if the solver fails or none is configured |
| `definition` | Selector or parse failure on a 200 response, template error | `definition_error`, not escalated; cleared by a definition update or pin change; stored with the response snippet |

_Source: O20; design/acquisition.md §3.9c._

**ACQ-112** (MUST) Text supplied by trackers (error pages, API messages, Newznab error descriptions, tracker status messages) **MUST** be matched only inside per-driver or per-definition classifiers that have tests and emit the enum above (A10). No other code may match tracker text. _Source: A10._

**ACQ-113** (MUST) Errors returned to callers **MUST** be typed: `ErrBudget{retry_at}`, `ErrBackoff{until}`, `ErrAuthFailed`, `ErrNeedsUser{challenge_id}`, `ErrDefinition`, `ErrReleaseUnavailable`, `ErrPayloadInvalid`. _Source: design/acquisition.md §3.9, §4.4._

---

## 12. Tracker HTTP layer

### 12.1 Client chain

**ACQ-114** (MUST) All tracker I/O — Cardigann, natives, Newznab/Torznab, RSS, grab fetch, enrichment and WASM indexer plug-ins — **MUST** go through one per-indexer tracker HTTP client. Nothing else may talk to a tracker. _Source: design/acquisition.md §3.1, §3.10._

**ACQ-115** (MUST) The client **MUST** apply, in order: egress; rate slot; session; solver; retry; classifier; redaction.
- **Egress:** the indexer's Proxy (HTTP CONNECT or SOCKS5), else the default egress.
- **Retry:** 2 attempts, exponential backoff with jitter, on 5xx, 408 and connect errors; honour `Retry-After`.
- **Redaction:** secret setting values scrubbed from logs, errors and history.

_Source: design/acquisition.md §3.10; O24._

**ACQ-116** (MUST) Proxies **MUST** be explicit references on the indexer (and on IRC networks), falling back to a configured default egress. Tag-intersection proxy selection **MUST NOT** be used. SOCKS4 **MAY** be added later. _Source: design/acquisition.md §2.2, §3.3._

**ACQ-117** (MUST) Outbound requests **MUST** carry OpenTelemetry spans when tracing is enabled and **MUST** update `bonnyread_acquisition_requests_total{indexer,facet,class,outcome}` and `bonnyread_acquisition_request_seconds`. _Source: design/acquisition.md §3.17; runtime.md §3.11._

### 12.2 Sessions, login and captcha

**ACQ-118** (MUST) Session state (cookies, User-Agent, expiry, how obtained: `login`, `user`, `solver`; pending captcha challenge) **MUST** be stored in a dedicated `IndexerSession` row, shared across nodes and persisted across restarts. Cookies **MUST** expire at their stated expiry, else after 30 days. _Source: O22; design/acquisition.md §3.10._

**ACQ-119** (MUST) Login **MUST** run lazily: `EnsureSession` runs `login.test` when defined and the session is older than 30 minutes, or when a response is recognised as a login page. [OPN-004] _Source: design/acquisition.md §3.10._

**ACQ-120** (MUST) Login **MUST** be single-flight across nodes per indexer. A node **MUST** claim a short-lived login lock on the `IndexerSession` row by conditional update (holder, expiry 60 s), re-read the session after claiming it, and run the login only if the session is still stale. Other nodes **MUST** wait for the lock holder's result up to the request deadline. This is the platform's row-claim single-flight primitive (`12-platform.md` PLT-062); platform leases and database advisory locks **MUST NOT** be used for this. [OPN-103] _Source: design/acquisition.md §3.10; A1; runtime.md §3.5._

**ACQ-121** (MUST) All Cardigann login methods **MUST** work: `post`, `form` (scraped inputs, `simpleCaptcha` solved automatically), `cookie` (user-pasted), `get`, `oneurl`. _Source: O22._

**ACQ-122** (MUST) An image captcha **MUST** store the challenge in `IndexerSession.captcha_challenge`, set the facet to `needs_user`, and be presented to the user; the answer **MUST** resume the login. For cookie-method indexers, a login-page response **MUST** mean "cookie expired" and set `needs_user`. _Source: O22; design/acquisition.md §3.10._

### 12.3 Cloudflare and FlareSolverr

**ACQ-123** (MUST) Challenges **MUST** be detected by server header, status (403/503) and page title (Cloudflare and DDoS-Guard), and a solver (FlareSolverr or Byparr) **MUST** be called only when a challenge is detected and the indexer references a Solver. _Source: O23._

**ACQ-124** (MUST) Solving **MUST** be single-flight per `(host, egress)` across nodes (same mechanism as ACQ-120: a claim on the `solver_clearance(host, egress_id)` row, `12-platform.md` PLT-062, PLT-123), **MUST** respect `Solver.max_sessions`, and the solver request **MUST** go through the indexer's egress. [OPN-103] _Source: design/acquisition.md §3.10; O24._

**ACQ-125** (MUST) The clearance cookies and User-Agent **MUST** be stored in `solver_clearance(host, egress_id)` (`12-platform.md`) in the database, reused by every node and every indexer on that host and egress until expiry, and the original request replayed with them. _Source: O23; design/acquisition.md §3.10._

---

## 13. Grab fetch path

**ACQ-126** (MUST) Acquisition **MUST** expose `Fetch(ctx, release_ref) → Payload`, callable from any node by `05-downloads.md`. It **MUST** be in-process; there are no proxied or encrypted download links and no API key in download URLs. _Source: design/acquisition.md §3.11; D7._

**ACQ-127** (MUST) Fetch **MUST**:
1. load the release from the caller-supplied copy, `SearchResult`, or the payload cache (ACQ-134);
2. take a `grab` budget ticket (`ErrBudget` on denial, so the caller can defer);
3. run `driver.Fetch` — for Cardigann the `download` block (`selectors`, `before` + `pathselector`, `infohash`, headers) with the session;
4. download with redirects handled manually;
5. validate the payload;
6. return `Payload{kind: torrent | magnet | nzb | ddl_plan, bytes, magnet, info_hash, files[], seed_policy, indexer_id}`;
7. record the grab against the budget and emit `acquisition.grabbed{indexer, release_key, purpose}`.

_Source: design/acquisition.md §3.11._

**ACQ-128** (MUST) Magnet vs .torrent **MUST** follow `torrent_policy.prefer_magnet`. A `Location: magnet:` redirect, or a body starting with `magnet:`, **MUST** be returned as a magnet. _Source: O26._

**ACQ-129** (MUST) Private and authenticated URLs **MUST** always be fetched by acquisition and handed to the download client as bytes; they **MUST NOT** be passed to the download client as URLs. Public magnets **MAY** pass through unchanged. _Source: O26; design/acquisition.md §3.11._

**ACQ-130** (MUST) Torrent validation **MUST** bdecode the payload, keep the **raw bytes of the `info` dictionary**, compute `info_hash = sha1(raw info bytes)`, and read the file list. If the outer dictionary is non-canonical (unsorted keys), **only the outer dictionary** **MUST** be re-encoded, with the original info bytes spliced in unchanged, so the infohash cannot change. _Source: O27; design/acquisition.md §3.11._

**ACQ-131** (MUST) NZB validation **MUST** require well-formed XML with at least one file. _Source: design/acquisition.md §3.11._

**ACQ-132** (MUST) The returned file list **MUST** let `05-downloads.md` apply "reject blocklisted hash" and "blocked extension" before adding to a client. _Source: O28._

**ACQ-133** (MUST) One grab **MUST** fetch a release once, even when the resulting Download feeds several slots (B1). _Source: B1; D20._

**ACQ-134** (MUST) A fetched payload **MUST** be kept in a payload cache for 5 minutes keyed by `(indexer, release key)`, so that an enrichment fetch followed by a grab, or a grab retried on another node, downloads it once. The cache **MUST** be stored in the database so every node can read it. [OPN-104] _Source: design/acquisition.md §3.8; evaluation.md §4.3._

**ACQ-135** (MUST) A login requiring a captcha during Fetch **MUST** return `ErrNeedsUser`; the download lifecycle keeps the grab pending and retries after the user answers. _Source: design/acquisition.md §4.4._

---

## 14. IRC ingest

### 14.1 Connections

**ACQ-136** (MUST) IRC ingest **MUST** lift `autobrr/internal/irc` (connection and channel state machines, NickServ, SASL, invite commands, bouncer and proxy dialers, flapping handling, message history) and `autobrr/internal/announce` (single- and multi-line processors, `MapVars`, templated URLs, `mappings`, and the GGn/OPS/RED special parsers). `go-deadlock` **MUST** be dropped. The autobrr release service dependency **MUST** be replaced by `ReleaseSink`. _Source: O5; design/acquisition.md §3.8._

**ACQ-137** (MUST) The topology **MUST** be `IrcNetwork` (one connection) → `IrcChannel` → `Indexer`. One network **MAY** carry several indexers' channels. Channel and announcer data come from the indexer's IRC definition; a channel **MAY** override announcers. _Source: design/acquisition.md §3.8._

**ACQ-138** (MUST) Each enabled IRC network **MUST** be held under a platform lease `irc:<network_id>` (`12-platform.md`, A1). Only the lease holder may connect. Leases are balanced across nodes by the platform. _Source: A1; runtime.md §3.5._

**ACQ-139** (MUST) Network and channel state machine transitions **MUST** map onto the indexers' `irc` facet status and emit `acquisition.irc_state{network, channel, state}`; disconnects **MUST** be notifiable (`irc.disconnected`). `bonnyread_acquisition_irc_network_up{network}` **MUST** be exported. _Source: O21; design/acquisition.md §3.8._

### 14.2 Announce hot path

**ACQ-140** (MUST) On the lease holder, each announce line **MUST** be processed as: definition regex (Go `regexp`, D1c) → variables → mappings → templated download URL (secrets from indexer settings) → Release (`origin=irc`, `size` possibly null, `announce` metadata, flags) → release key extraction (the `torrentId` variable, matched to RSS GUIDs through `TrackerAlias.torrent_id_extractors`) → `RssSeen` insert → `ReleaseSink.Announce` **in process on the same node**. _Source: design/acquisition.md §3.8; D1c._

**ACQ-141** (MUST) An announce whose key is already in `RssSeen` **MUST NOT** be handed to evaluation again. _Source: O15; design/acquisition.md §3.8._

**ACQ-142** (MUST) The `RssSeen` insert and every write derived from an announce (including evaluation's grab commit) **MUST** be fenced with the network lease token, so a former holder cannot commit after losing the lease. The lease fence **MUST** travel in the context passed to `ReleaseSink.Announce`. _Source: runtime.md §3.5._

**ACQ-143** (MUST) `ReleaseDiscovered` records for announces **MUST** be written asynchronously and **MUST NOT** delay the hand-off. _Source: design/acquisition.md §3.8._

**ACQ-144** (MUST) IRC is best-effort and RSS is the source of truth: an announce in flight when the lease holder dies **MAY** be lost, and **MUST** be covered by the catch-up RSS job (ACQ-095). _Source: design/acquisition.md §3.8._

**ACQ-145** (MUST) `POST /api/v1/indexers/{id}/announce` **MUST** inject raw announce lines through the same path, for definition debugging and tests (`11-api-and-clients.md`). _Source: design/acquisition.md §3.8._

### 14.3 Enrichment contract with evaluation

**ACQ-146** (MUST) Acquisition **MUST** implement `Enrich(ctx, *Release, FieldSet) → error` for evaluation. `FieldSet` **MUST** support at least `size`, `files`, `uploader`, `record_label`, `log_score`, `cue`, `tags`, `freeleech`. Enrich **MUST** fill only requested, currently unknown fields, and **MUST** mark each requested field `known` or `unavailable`. [OPN-018] _Source: O6; evaluation.md §3.8, §6 Q6; open-issues C (evaluation)._

**ACQ-147** (MUST) Enrich **MUST** use the indexer's API facet (`Enricher`) if it has one, else fetch the .torrent through the grab fetch path (§13) and read size and files from metainfo. A .torrent fetched for enrichment **MUST** be placed in the payload cache and **MUST** count against the grab budget only if the release is then grabbed; it counts as one query otherwise. [OPN-018] _Source: O6; design/acquisition.md §3.8._

**ACQ-148** (MUST) Enrich **MUST** honour the caller's deadline; evaluation's per-release deadline (A20) applies to the evaluation stages, and enrichment latency is reported separately. On timeout the fields remain unknown. [OPN-018] _Source: A20; evaluation.md §3.19._

**ACQ-149** (MUST) Acquisition **MUST NOT** call Enrich on its own initiative; only evaluation requests it, and only when an unknown fact could change the outcome. _Source: O6; evaluation.md §3.8._

---

## 15. Racing mode hooks (later phase, D16)

**ACQ-150** (MUST) Racing mode **MUST NOT** ship in phase 1. The phase-1 design **MUST** keep these hooks: `Purpose = racing` on queries, grabs and budgets; `ReleaseSink.Announce` as the single announce stream consumed by both library evaluation and autograb rules (`04-evaluation.md`); and Downloads with no slot (`05-downloads.md`). _Source: D16; design/acquisition.md §6 Q9._

**ACQ-151** (MUST) Racing grabs **MUST** count against the same per-indexer grab budgets as library grabs. [OPN-008] _Source: design/acquisition.md §6 Q9; open-issues C (acquisition)._

**ACQ-152** (SHOULD) Racing-specific client-side guards (max active racing downloads, ignore when slow) **SHOULD** be specified with racing mode; their placement between acquisition and `05-downloads.md` is open. [OPN-105] _Source: downloads.md §6 Q8._

---

## 16. DDL (phase 3, D11)

**ACQ-153** (MUST) DDL **MUST** be an optional, isolated module delivered in phase 3 as a third protocol (`protocol = ddl`). A DDL Release's `download` **MUST** be `link_groups[]{coverage (issue range, volume, TPB), size, links[]{host, url}}`. _Source: D11; design/acquisition.md §3.14._

**ACQ-154** (MUST) DDL site definitions **MUST** reuse the Cardigann engine's primitives (requests, selectors, filters, templates, login, solver) plus one added block, `article.groups`, that yields link groups from an article page. There **MUST NOT** be a second scraping DSL. [OPN-007] _Source: design/acquisition.md §3.14, §6 Q8; open-issues C (acquisition)._

**ACQ-155** (MUST) DDL site definitions **MUST** be delivered through the `bonnyread-ddl` feed and the normal definition lifecycle (§4), shipped and updated separately from the core. _Source: D11._

**ACQ-156** (MUST) Links **MUST** be resolved at grab time, not at search time. `Resolve` **MUST**, in parallel: follow redirects; detect file vs folder; drop and blocklist dead links; order groups by user host preference and the "avoid large downloads" rule; and choose the first non-overlapping path of working groups, returning a `DDLPlan`. _Source: D11; design/acquisition.md §3.14._

**ACQ-157** (MUST) Host plug-ins (native or WASM) **MUST** only **resolve** a link to `{url, headers, cookies, quota state}`; the core streams the bytes (`05-downloads.md` DDL client). _Source: A11._

**ACQ-158** (MUST) Host quotas **MUST** use the shared rate limiter's quota buckets keyed `ddl:host:<host>`. _Source: A3; design/acquisition.md §3.14._

**ACQ-159** (MUST) Torrent links found on DDL sites **MUST** become ordinary magnet or .torrent payloads through the fetch path. Synthesised magnets with dead trackers **MUST NOT** be produced. _Source: design/acquisition.md §3.14._

---

## 17. Placement and events

**ACQ-160** (MUST) Work placement **MUST** be:

| Work | Placement |
|---|---|
| IRC connections and announce hot path | Holder of lease `irc:<network>` |
| RSS polls, definition updates, caps refresh, purges of `RssSeen`, `SearchResult`, `ReleaseDiscovered` and payload cache | Schedule rows → jobs on any node |
| Searches, grab fetch, enrichment | Any node, in request or job |
| Login and solver | Any node, single-flight (§12) |
| Transport spacing and budgets | Shared DB rate limiter (in-process on SQLite) |

_Source: A1; A3; design/acquisition.md §3.15._

**ACQ-161** (MUST) Acquisition **MUST** emit `acquisition.rss_gap`, `acquisition.indexer_health`, `acquisition.irc_state`, `acquisition.grabbed` and `acquisition.definitions_updated`, registered in the event catalogue (`12-platform.md`), and export `bonnyread_acquisition_requests_total`, `bonnyread_acquisition_request_seconds`, `bonnyread_acquisition_budget_remaining`, `bonnyread_acquisition_rss_lag_seconds`, `bonnyread_acquisition_irc_network_up` and `bonnyread_acquisition_definition_set_info`, and count regex timeouts in `bonnyread_regex_timeouts_total{source="cardigann"}` (`12-platform.md` PLT-098, PLT-134). Metric labels **MUST** be bounded (indexer, feed, network, facet, class). _Source: D7; design/acquisition.md §3.17._

---

## 18. Data model

Acquisition-owned entities. Shared entities (MediaKind, Tag) are owned by `02-domain-model.md`; `Download` and `DownloadClient` by `05-downloads.md`; platform tables (schedule, lease, job, rate-limiter tokens and buckets, plug-in, `egress_proxy`, `solver`, `solver_clearance`) by `12-platform.md`.

```go
Indexer {
  name string
  driver enum(cardigann, native, newznab, torznab, rss, wasm, ddl)
  definition_id string?          // cardigann / native / ddl site / wasm module id
  irc_definition_id string?      // autobrr identifier; set ⇒ IRC facet available
  definition_pin string?         // body sha256; empty = follow the active set
  protocol enum(torrent, usenet, ddl)
  privacy enum(public, semi_private, private)
  base_url string
  settings JSON                  // validated by schema; secret keys marked by schema
  enable_rss, enable_auto_search, enable_interactive_search, enable_irc bool
  priority int = 25
  kind_scopes JSON [ {kind, categories []CategoryRef, search JSON} ]
  limits JSON {queries {n, window}, grabs {n, window}, min_interval_override, backlog_share = 0.5}
  torrent_policy JSON {min_seeders, seed_ratio, seed_time, pack_seed_time, prefer_magnet, reject_blocked_ext}
  client_hint → DownloadClient?  // hint only (05-downloads)
  rss_interval_minutes int?
  edges: proxy → egress_proxy?, solver → solver?, tags → Tag[], irc_channels ← IrcChannel[], state → IndexerState (1:1)
  index: (driver, definition_id)
}
IndexerState {                   // runtime only, never user-edited
  indexer_id PK
  facet_status JSON {search|rss|irc|api|grab|ddl: {state, escalation, disabled_until, last_error, last_success}}
  caps JSON, caps_fetched_at
  rss_cursor JSON {guid, key, published_at}, rss_last_gap JSON?
  definition_used string         // sha256
}
IndexerSession { indexer_id PK, cookies JSON, user_agent, expires_at, obtained_by enum(login,user,solver),
                 captcha_challenge bytes?, login_lock_holder string?, login_lock_until timestamp? }
RssSeen { indexer_id, key, seen_at  unique(indexer_id, key)  index(seen_at) }        // purge 14 d
ReleaseDiscovered { id ULID, indexer_id, key, origin, release JSON (download redacted), discovered_at
                    index(indexer_id, discovered_at) }                               // purge 14 d
SearchResult { id ULID PK, search_id, indexer_id, release JSON, expires_at  index(search_id), index(expires_at) }
PayloadCache { indexer_id, key, kind, bytes, info_hash, files JSON, expires_at  unique(indexer_id, key) }
// Proxy = 12-platform.md egress_proxy; Solver = 12-platform.md solver; SolverClearance = 12-platform.md solver_clearance
IrcNetwork { name, server, port, tls, tls_pin string? (SPKI SHA-256; verification is never disabled, ARC-068), pass (secret), nick, auth {mechanism sasl|nickserv, account, password (secret)},
             invite_command, bouncer_addr, bot_mode, enabled, edge proxy → egress_proxy? }
IrcChannel { network → IrcNetwork, name, key (secret)?, indexer → Indexer, announcers_override []string?, enabled
             unique(network, name, indexer) }
DefinitionFeed { name unique, kind enum(cardigann, irc, ddl), repo_url, path, schema_version int?, ref,
                 follow bool, precedence int, enabled, last_checked, active_set → DefinitionSet? }
DefinitionSet  { feed → DefinitionFeed, commit, fetched_at, status enum(staged, active, retired), report JSON
                 unique(feed, commit) }
Definition     { set → DefinitionSet, def_id, sha256, body text, valid bool,
                 meta JSON {name, type, language, links, legacylinks, replaces[], caps, settings_schema,
                            request_delay, min_interval, tombstone {reason, alias_of}?}
                 unique(set, def_id)  index(sha256) }
CustomDefinition { kind, def_id, body text, sha256, valid bool, report JSON, updated_at  unique(kind, def_id) }
TrackerAlias   { cardigann_id?, irc_id?, native_id?, torrent_id_extractors []string }   // shipped data, loaded read-only
```

**ACQ-162** (MUST) Deleting an indexer **MUST** cascade to its state, session, schedule row, `RssSeen`, `SearchResult`, payload cache rows and IRC channels in one transaction. _Source: runtime.md §3.5.1._

---

## 19. Flows

### 19.1 Interactive search (TV episode)

1. The API builds `TvEpisode` criteria from the library (ids, titles with scene names, scene S/E).
2. `Search(purpose=interactive)`: the planner selects indexers (interactive enabled, TV scope, not backed off, usable caps) and plans tier 1 (tvdbid+season+ep) and tier 2 (titles × `S02E05`).
3. Per indexer in parallel: interactive budget ticket → rate slot → tier-1 query through the tracker HTTP client → post-filter categories → if empty, tier 2 → page until not full.
4. Each indexer's results are written to `SearchResult` (TTL 60 min) and streamed to the UI; evaluation annotates them.
5. The user grabs release R on node B: B loads R from `SearchResult` → `05-downloads.md` → `Fetch`.

Failure paths: an indexer out of budget appears as `budget_denied` with `retry_at`; a backed-off indexer appears as skipped with its reason; an indexer exceeding its deadline returns partial results with outcome `error(timeout)`.

### 19.2 RSS cycle

1. The platform scheduler (any node) finds `rss:indexer:42` due and enqueues it (dedupe `WhilePendingOrRunning`).
2. A worker claims it and runs `Query{recent, union of categories}` until the cursor is reached.
3. Fenced transaction: insert keys into `RssSeen`, keep the new ones, enqueue `evaluate_releases`, advance `rss_cursor`, set facet `ok`; commit.
4. If the cap was hit before the cursor: record the gap, raise health, emit `acquisition.rss_gap`.

Failure paths: a worker crash before commit leaves nothing committed; the reaper releases the job and another node repeats it with the same outcome. A stale worker's commit fails the fence. A tracker error is classified (§11.2) and the job completes with the facet status updated; the schedule continues.

### 19.3 IRC announce

1. The holder of lease `irc:3` receives a PRIVMSG from the announcer bot on a channel bound to indexer 42.
2. Parse → Release → key from `torrentId` → fenced `RssSeen` insert; if not new, stop.
3. `ReleaseSink.Announce` in process; evaluation may call `Enrich(size)` → API facet or .torrent fetch → payload cache.
4. Evaluation accepts → download lifecycle → `Fetch` hits the payload cache → grab budget → Payload.
5. Asynchronously: `ReleaseDiscovered`, metrics.

Failure paths: the holder dies — the lease expires (15–20 s), another node connects, and a catch-up RSS job runs for bound indexers; the old holder's fenced writes fail. The same release later on RSS is suppressed by `RssSeen`.

### 19.4 Grab of a private torrent needing login and a Cloudflare solve

1. `Fetch(R, indexer 7)` → grab ticket → Cardigann `download` block → GET details page.
2. 403 challenge detected → single-flight solve for `(host, egress)` through the same egress → `SolverClearance` stored → replay.
3. 200 login page → classified login required → claim login lock on `IndexerSession` → re-read session (still stale) → form login (+`simpleCaptcha`) → `login.test` OK → store cookies → release lock.
4. Retry details → selector → torrent URL → GET without auto-redirect.
5. Bdecode → `sha1(raw info)` → canonicalise outer dict only → record grab → Payload.

Failure paths: image captcha → `needs_user`, `ErrNeedsUser`, grab kept pending; login error matched → one re-login, then `auth_failed` and alert; `Location: magnet:` → magnet payload; invalid bencode → `ErrPayloadInvalid`, `grab` facet error.

### 19.5 Definition update

1. The daily schedule enqueues `definitions.update:prowlarr-indexers`.
2. Resolve `master` → commit C; if equal to active, stop.
3. Download tarball at C, extract `definitions/v11`, validate each file, stage set.
4. One transaction: activate, retire previous, apply `replaces` and `legacylinks`, raise removed/invalid/tombstone health items; emit `acquisition.definitions_updated`.

Failure paths: download or extract failure → job retries with backoff, active set unchanged. Schema gate fails for the whole feed → set not activated, health notice. A regression reported by users → "revert to previous set" or per-indexer pin.

---

## 20. Non-goals

| Excluded | Reason |
|---|---|
| Newznab/Torznab server façade | D7; results are typed and in-process |
| App sync (pushing indexers into other apps), `AppIndexerMap`, per-app sync categories | Separate-process workaround (D19) |
| AES-encrypted proxied download links, API key in download URLs | The download client never fetches tracker URLs |
| Per-consumer RSS polling and Jackett's 35-minute result cache | One poll per indexer |
| Two stacked backoff layers | One per-facet status |
| `100000+` custom category ids internally | A16 |
| Aggregate/filter meta-indexers, OMDb id→title fallback, TorrentPotato, Jackett blackhole | External-consumer features |
| TVRage `rid`, Torcache, EZRSS, Fanzub, the Cloudflare captcha interceptor | Dead services |
| Source tracking by User-Agent and reverse DNS | Replaced by `Purpose` |
| `indexers.prowlarr.com` as definition source | Third-party hosted dependency (D3c); GitHub by commit instead |
| Embedding or distributing Cardigann definitions | D5b |
| Translating C# indexer or engine code | D5, D5b |
| Autobrr `release/push`, per-app flag bitmaps, list sync into filters | In-process `ReleaseSink`; "wanted" is a library query |
| Autobrr's separate RSS feed poller | Merged into the RSS loop |
| Pluggable IRC connections | Long-lived sockets stay in the core |
| Racing mode in phase 1 | D16 |
| Config importers for Prowlarr, Jackett and autobrr in phase 1 | Later [OPN-005] |

---

## 21. Traceability

| Reference | Preserved | Changed (D19) |
|---|---|---|
| Prowlarr | Cardigann v11 coverage, natives' outcomes, caps, query/grab limits, `requestDelay`, backoff schedule, health checks, `replaces`, schema-version gate, FlareSolverr detection | Fresh engine; definitions from GitHub by commit, stored in DB, atomic sets with rollback and pins; `Supersedes` instead of blocklist; class-aware budgets on the shared limiter; tracker-native category keys; auth failures do not escalate |
| Jackett | Template semantics, `legacylinks`, lenient keys, re-encoding of unsorted torrents | Purpose-built interpreter with golden suite; only the outer dict is re-encoded so the infohash is preserved; no bundled definitions; no result cache |
| Sonarr/Radarr | Tiered id → title search, season-vs-episode rule, anime and daily handling, paging to 1,000, `IsValidRelease`, RSS watermark and gap detection, NZB validation, magnet redirects | Gap is an event the backlog can act on; delay is not an RSS side effect; acquisition always fetches authenticated payloads |
| Lidarr, Bookshelf, Whisparr | Music/book/adult query shapes; adult categories for every driver; per-indexer adult search settings | Kind scopes on one indexer instead of per-app indexers |
| autobrr | IRC connection features, announce parsing, IRC definitions with tests and tombstones, on-demand enrichment, feed size heuristics | One indexer identity with an IRC facet; lease per network with fencing; `RssSeen` shared with RSS; in-process sink |
| Kapowarr | DDL link groups, grab-time link resolution with fallback, host quotas | Cardigann-based site definitions; host plug-ins resolve only, core streams (A11); quotas on the shared limiter |
