# Architecture Decisions

Decisions that must be settled before architecture planning. Each entry records the context, the options and a recommendation. All decisions were settled with the user on 2026-09-27/28; each entry records the decision first, then the analysis behind it.

Background: [analysis/comparison.md](analysis/comparison.md). Section references (§) point there unless noted.

The factual premises below were verified on 2026-09-27 and corrected where needed; the evidence is in [verification/](verification/). No correction reverses a recommendation; where verification bears on a recommendation itself, the decision carries a **Verification note**.

| # | Decision | Status |
|---|---|---|
| D1 | Regex engine | **Decided** |
| D2 | Scope and phasing | **Decided** |
| D3 | Metadata sources and hosted services | **Decided** |
| D4 | Multiple files (versions) per item | **Decided** |
| D5 | License | **Decided** |
| D6 | Frontend stack | **Decided** |
| D7 | External API compatibility | **Decided** |
| D8 | Release parser foundation | **Decided** (spike, then choose) |
| D9 | Community profile data | **Decided** |
| D10 | Adult model: Whisparr v2 or v3 | **Decided** |
| D11 | Direct-download (DDL) acquisition | **Decided** |
| D12 | Client–server architecture | **Decided** |
| D13 | Database and ORM | **Decided** |
| D14 | High availability | **Decided** |
| D15 | Plug-in boundary | **Decided** |
| D16 | Secrets, racing mode, self-updater | **Decided** |
| D17 | Authentication and authorisation | **Decided** |
| D18 | Deployment range | **Decided** |
| D19 | Porting principle: redesign, don't replicate | **Decided** |
| D20 | Design-review outcomes (amends D3, D4, D9, D12, D13, D14, D16) | **Decided** |

---

## D1. Regex engine

**Status:** **Decided** (2026-09-27)

**Decision:**
- **Custom-format / attribute patterns:** Profilarr's patterns and its upstream (the Dictionarry database) are the gold standard for regexes that match attributes in release names. Whisparr-Eros supplies the adult-specific patterns. Because 87% of Dictionarry's patterns fail in Go's `regexp`, these run on **regexp2**, with a timeout on every match.
- **D1a, structural parsing** (title, year, season/episode, air date, absolute numbers, edition): the **Sonarr/Radarr cascades are the reference**, with their golden test corpora (1,631+ cases) as the acceptance suite.
- **D1b, indexer definitions:** Cardigann `regexp`/`re_replace` filters run **as-is on regexp2**, with a timeout and the small `\p{IsBlock}` rewrite step.
- **D1c, other users:** subtitle matching **reuses the main release parser**. Comics and books get sub-parsers in the same package, ported with their test fixtures. autobrr's IRC definitions already use Go `regexp` and need no change.
- Any other regex use that comes up is raised as its own question.

The analysis below is kept for reference.

**Context:** Several sources of regexes use syntax that Go's standard `regexp` package (RE2) can't handle. RE2 has no lookarounds and no backreferences, needs `\x{XXXX}` instead of `\uXXXX`, and has no .NET named Unicode blocks (`\p{IsCyrillic}`). Named groups are not a blocker: Go accepts `(?<name>…)` since 1.22.
- **.NET syntax:**
  - custom-format regexes, including the Dictionarry and TRaSH datasets;
  - Cardigann filters (`regexp`, `re_replace` and the `{{ re_replace }}` template; `validate` and `andmatch` are not regexes);
  - the Sonarr, Radarr, Lidarr, Bookshelf and Whisparr parsers;
  - Whisparr's product-code patterns.
- **Python regex:** Bazarr's guessit/rebulk patterns.

Measured on 2026-09-27 (unique patterns; see [verification/decision-premises.md](verification/decision-premises.md) §D1):

| Source | Patterns | RE2 rejects | regexp2 rejects |
|---|---|---|---|
| TRaSH custom formats | 1,342 | 71 (5.3%) | 0 |
| Dictionarry v2 | 615 | 534 (87%), mostly the `(?<=^\|[\s.-])GROUP\b` lookbehind idiom | 0 |
| Cardigann (Prowlarr v11 / Jackett) | 710 / 716 | 25 / 23 (about 3.5%) | 2 / 2 (`\p{IsCyrillic}`, `\p{IsCJKUnifiedIdeographs}`) |
| Sonarr parser | 192 | 132 (69%) | 0 |
| Radarr / Lidarr / Bookshelf / Whisparr v2 / v3 parsers | 77 / 67 / 61 / 72 / 100 | 36% / 15% / 11% / 24% / 31% | 0 |

The 1,352 dataset patterns that compile in both engines gave identical results on 2,647 test titles. That corpus is mostly ASCII, and RE2's `\d`, `\w` and `\b` are ASCII-only where .NET's are Unicode.

**Options:**
- **A.** Use `github.com/dlclark/regexp2`, a .NET-compatible engine, for everything.
- **B.** Use `regexp2` for patterns supplied by users or data files (custom formats, Cardigann, IRC announce definitions). Rewrite our own parser patterns in RE2 where possible, and fall back to `regexp2` where a rewrite isn't possible.
- **C.** Use RE2 only and require datasets to change their patterns. This breaks compatibility with existing profile datasets and indexer definitions.

**Recommendation:** **B.** `regexp2` is a backtracking engine, so every match needs a timeout (Profilarr already uses 100 ms, and Whisparr v3 sets one on every regex; regexp2's timeout is coarse, firing after about 130 ms for a 100 ms limit). The ported golden test corpora (§6) are the safety net for any rewrite to RE2. The measurements rule out C (it would break 87% of Dictionarry and 5% of TRaSH). Carrying out B needs:
- a **pattern-normalisation step** in front of regexp2 that rewrites .NET named Unicode blocks (`\p{IsBlock}`) to code-point ranges, with a test that compiles every pattern in each dataset;
- realistic expectations for the RE2 rewrite: it covers roughly a third of Sonarr's patterns. The rest (lookbehind-anchored numbering, `\k<sep>` dates) stays on regexp2 unless D8 replaces it with `rls`, which is pure RE2. Rewritten patterns must allow for RE2's ASCII-only `\d`, `\w` and `\b`.

---

## D2. Scope and phasing

**Status:** **Decided** (2026-09-27)

**Decision:** the app list below is settled, and delivery is in three phases:
- **Phase 1:** TV, movies, indexers (search, RSS and IRC), profiles, download handling (policies and extraction), requests.
- **Phase 2:** subtitles, music.
- **Phase 3:** books, adult, comics.

The data model and pipeline are designed for every media kind from day one, so later phases don't need schema rewrites.

**In scope** (submodules analysed in `analysis/`):

| Area | Replaces | Notes |
|---|---|---|
| TV, movies | Sonarr, Radarr | Core |
| Indexers | Prowlarr, Jackett, autobrr | One indexer layer with search, RSS, IRC and API capabilities |
| Profiles | Profilarr, Recyclarr | Native profile store plus importers (D9) |
| Download handling | Cleanuparr, Unpackerr | Native policies and an `Extracting` stage |
| Requests | Seerr | Users, permissions, requests, issues |
| Subtitles | Bazarr | About 50–55 providers offered in the UI (about 65 modules) to port |
| Music | Lidarr | Needs its own metadata source (D3) |
| Books | Bookshelf (Readarr fork) | **Viable:** rreading-glasses, Hardcover and OpenLibrary exist (D3) |
| Adult | Whisparr v2 and v3 (Eros) | Model choice in D10 |
| Comics | Kapowarr | The direct-download question is D11 |

**Considered and not added:**
- Readarr: archived; replaced by Bookshelf.
- Mylar3: comics, a niche Python app; Kapowarr covers comics.
- Tdarr: closed source.
- Notifiarr: its client is partly closed.
- Huntarr: its repository no longer resolves.
- Sportarr and LazyLibrarian: seen as integration targets in Cleanuparr, autobrr and Prowlarr (and Sportarr in Unpackerr's docs), but niche.

**To decide: phasing.** Every media kind shares one pipeline (§2.4), but each adds a lot of unique logic:
- music identification;
- book metadata profiles;
- the adult date matcher;
- comic coverage planning;
- subtitle providers.

**Recommendation:**
- **Phase 1:** TV, movies, indexers (search, RSS and IRC), profiles, download handling (policies and extraction), requests.
- **Phase 2:** subtitles, music.
- **Phase 3:** books, adult, comics.

The data model and pipeline are designed for all kinds from day one, including per-kind quality axes, M:N file↔item links, absolute paths and metadata profiles, so later phases don't force schema rewrites.

---

## D3. Metadata sources and hosted-service dependencies

**Status:** **Decided** (2026-09-27)

**Decision:**
- **D3a, TV:** **TVDB v4 is the canonical TV source** (episode ordering for scene/XEM/anime), using a project key; users may add a TVDB subscriber PIN. TMDb supplies images, ratings and cross-ids.
- **D3b, keys:** the binary **may ship project-registered keys**, clearly attributed, and **every key is user-overridable** in settings. Never ship keys copied from other apps. Sources that issue per-user keys (StashDB, ThePornDB, ComicVine) take user keys.
- **D3c, hosting:** **no hosted infrastructure in v1.** All sources are called directly behind a per-kind provider interface with a persistent cache that respects each source's terms. Scene mappings ship as bundled/community data. A hosted cache can be added later behind the same interface.
- **D3d, books (phase 3):** **Hardcover** (OAuth device flow) as primary, **OpenLibrary** as keyless fallback and ISBN lookup, **Audnexus/AudiobookDB** for audiobooks. Store a per-provider external-id map.
- **Other kinds** follow the recommendation below: TMDb for movies, MusicBrainz (+ Cover Art Archive, AcoustID) for music, StashDB/ThePornDB for adult, ComicVine for comics.

The analysis below is kept for reference; its Verification note is resolved by D3a and D3b.

**Context:** Almost every media kind depends on a hosted proxy today (§5.2):

| Kind | Proxy today | Operator |
|---|---|---|
| TV | SkyHook | Servarr |
| Movies | `api.radarr.video` | Servarr |
| TV scene mappings, daily-series list | `services.sonarr.tv` | Servarr. No public replacement |
| Music | `api.lidarr.audio` | Servarr. No fallback (only a URL override to another LidarrAPI-compatible server). Down from 2025-05-21 for months after a MusicBrainz schema change ([MetaBrainz forum](https://community.metabrainz.org/t/for-those-using-lidarr-and-musicbrainz-metadata/768255)); fallback servers declined ([Lidarr#5558](https://github.com/Lidarr/Lidarr/issues/5558)) |
| Books | `api.bookinfo.pro`, `hardcover.bookinfo.pro` | Advertised as rreading-glasses' hosted instances by its author (`blampe`); legal operator not named |
| Adult | `api.whisparr.com` | Whisparr, closed source. Whisparr v3 also calls StashDB and TMDb directly for import lists |
| OAuth | `auth.servarr.com` (Sonarr, Radarr, Bookshelf, Whisparr v3), `services.sonarr.tv` (MyAnimeList), `spotify.lidarr.audio` (Spotify) | Servarr |
| Indexer definitions | `indexers.prowlarr.com` | Servarr |

Seerr also uses `api.radarr.video` for IMDb ratings and hard-codes TMDb and TVDB keys (and a Rotten Tomatoes Algolia key). Radarr and Whisparr v3 hard-code the same TMDb bearer token.

**Options per kind:**

| Kind | Direct source | Notes |
|---|---|---|
| TV | TMDb (free key for non-commercial use; logo and disclaimer required; cached data must not be kept longer than 6 months; ~40 requests/second) and/or TVDB v4 (keys are per **project**: free for under $50k/year revenue with an attribution link, discounts for open source; users can add a paid subscriber **PIN**, as Seerr does), TVMaze (keyless, 20 calls per 10 s, CC BY-SA attribution) | TVDB ordering matters: scene numbering, TheXEM mappings and anime absolute numbering are keyed to TVDB, and Sonarr's whole DataAugmentation layer assumes TVDB ids. A TMDb-default TV provider needs an id/episode-order mapping layer (TMDb episode groups, XEM's TVDB origin). TheXEM directly for scene numbering (Sonarr already calls it). Scene-name mappings (`services.sonarr.tv/scenemapping`) have no public replacement and need bundled or community data: the hardest gap for "no hosted infra" |
| Movies | TMDb | Straightforward |
| Music | MusicBrainz (about 1 request/second per IP, meaningful User-Agent required; can be self-mirrored via the replication feed, CC BY-NC-SA, fine for non-commercial self-hosting) + Cover Art Archive (no key) + AcoustID (app key, non-commercial only, 3 requests/second) | Must replicate the aggregation Lidarr's server does. A first import of a large library takes hours at 1 request/second, so plan a queue with a persistent cache, incremental refresh, and an optional user-run MusicBrainz mirror |
| Books | Hardcover GraphQL (beta; free plan 5,000 requests/day, 60/minute; tokens may be reset and keys expire every 1 January; users must not paste personal tokens, so use OAuth, e.g. the device-authorization grant), OpenLibrary (keyless, 1–3 requests/second; monthly dumps for bulk, licence not stated), Audnexus (audiobooks; its README points to **AudiobookDB** as successor), or an embedded **rreading-glasses** (Go, GPL-3.0; its Goodreads mode relies on unofficial access to a retired API) | Store **an external-id map per provider**, never one "foreign id". This avoids Bookshelf's Goodreads-vs-Hardcover database split. Hardcover's quotas make it a poor sole source for bulk imports |
| Adult | ThePornDB API (per-user token; some reads work anonymously), StashDB (GraphQL; per-user key; registration needs a publicly posted, rotating invite code; new accounts are read-only, which is enough for lookups) | No project-level key exists for either. Replicating the proxy's cross-source id linking is real work, not a URL swap |
| Comics | ComicVine (per-user key; 200 requests per resource per hour plus velocity detection; non-commercial only; link-back attribution); Metron as an alternative (account required; 20/minute, 5,000/day) | Large imports take hours; must be user-keyed, queued and cached |
| OAuth | Per-provider flows, device-code flows where available (Trakt, Hardcover), or client ids supplied by the user for redirect-only providers (Simkl, AniList, MyAnimeList, Spotify, Goodreads signing) | |
| Indexer definitions | The GitHub repos directly, pinned by commit; see D5 | |

**Recommendation:**
- Use a **pluggable metadata-provider interface per kind**, going direct to each source, with a persistent local cache. The cache must respect each source's terms (TMDb data refreshed or expired within 6 months), and rate-limited sources (MusicBrainz, ComicVine, Metron, Hardcover) go through a queue with incremental refresh. Build **attribution** (TMDb logo and disclaimer; TVDB, TVMaze and ComicVine links) into the provider interface.
- Defaults:
  - TMDb for movies and TV, with TVDB optional through a project key, to which users can add their subscriber PIN;
  - MusicBrainz for music;
  - Hardcover (through OAuth, quota-aware) with an OpenLibrary fallback for books, plus Audnexus for audiobooks behind the provider interface (track its successor, AudiobookDB). rreading-glasses is optional, not a default;
  - TPDB/StashDB for adult, with per-user keys, degrading gracefully when they are absent;
  - ComicVine for comics (per-user key).
- Every provider key is supplied by the user or the project. Nothing is hard-coded. Don't repeat the upstream hard-coded keys (Seerr's TMDb/TVDB/Algolia keys, Radarr's TMDb bearer, Lidarr's AcoustID/Last.fm/Headphones keys). Telemetry is absent or opt-in.
- **Run no hosted infrastructure in v1.** This is achievable; the cost is slower first imports (music, comics, books) and losing Sonarr's scene-mapping service, which needs bundled or community-sourced data.

**Verification note:** two points may change the recommendation itself ([verification/metadata-services.md](verification/metadata-services.md) §3):
- **TMDb as the default TV provider is a correctness risk**, not only a licensing choice, because scene, XEM and anime numbering are keyed to TVDB ordering. Either add an id/episode-order mapping layer, or make TVDB the default TV provider using a project key.
- **A TVDB project key has to ship in the binary** (users can only add a PIN, not a key). That conflicts with "nothing is hard-coded" unless project-supplied, user-overridable keys are allowed.

---

## D4. Multiple files (versions) per item

**Status:** **Decided** (2026-09-27)

**Decision:** **version slots**, available from phase 1. Each item can have several named slots (for example "4K" and "1080p", or "ebook" and "audiobook"). Each slot has its own quality profile and root folder, and is grabbed, upgraded and imported independently. The decision engine, queue, history, naming and availability all work per slot.

The analysis below is kept for reference.

**Context:** Every PVR stores one version (one quality) per item, which forces users into multi-instance workarounds. Radarr and Whisparr v3 hold one file per movie, Sonarr and Whisparr v2 one file per episode (one file may hold several episodes), and Lidarr one file per track. Bookshelf allows several files per edition (multi-file audiobooks) but only one format; Kapowarr is M:N.
- 4K and 1080p Radarr/Sonarr pairs, linked by instance-to-instance import lists and Seerr's `is4k` servers, with every status field duplicated for 4K;
- ebook and audiobook Bookshelf pairs. Bookshelf's README states the limitation: "only one type of a given book is supported… you will need multiple instances" (`README.md:9-11`).

**Options:**
- **A.** One file per item, like today. This keeps the multi-instance setups.
- **B.** Multiple **version slots** per item. Each slot has its own quality profile and root folder, and is grabbed, upgraded and imported independently.

**Recommendation:** **B.** It removes the instance-to-instance lists, Seerr's 4K duplication and the ebook/audiobook split. It generalises naturally: a book has ebook and audiobook slots, and a movie has 4K and 1080p slots. The cost is that the decision engine, queue, history, naming and availability all work per slot.

---

## D5. License

**Status:** **Decided** (2026-09-27)

**Decision:**
- **D5a:** the project is licensed **AGPL-3.0**. This permits porting Profilarr's AGPL code directly, as well as the GPL-3.0 sources (AGPL-3.0 §13 and GPL-3.0 §13 allow combining them). autobrr's GPL-2.0-or-later code and `go-deluge` are used under GPL-3.0 terms. MIT and MPL-2.0 dependencies are compatible. Anyone who offers a modified version to users over a network must offer them its source.
- **D5b:** a **fresh Go Cardigann engine**, seeded from the MIT Go Cardigann (dropping the unlicensed IMDb scraper) and extended to the v11 schema by testing behaviour against the definitions and recorded HTTP fixtures, never by translating C#. **Definitions are never embedded or distributed**: they are downloaded at runtime into the data directory. **Legal review before the first public release.**
- Run a license scanner (`go-licenses`) over the final dependency graph before release.

The analysis below is kept for reference.

**Context:**

| License | Apps |
|---|---|
| GPL-3.0 | Sonarr, Radarr, Prowlarr, Lidarr, Bookshelf, Whisparr, Kapowarr, Bazarr, Cleanuparr |
| AGPL-3.0 | Profilarr |
| **GPL-2.0-only** | **Jackett.** Its license file is GitHub's GPLv2 template, with no "or later" grant anywhere and no version statement elsewhere (see [jackett.md](analysis/jackett.md) §1.5). "-only" is the conservative reading of an ambiguous situation: GPLv2 §9 lets a recipient pick any version if none is specified, but relying on that across hundreds of contributors with no CLA isn't prudent |
| GPL-2.0-or-later | autobrr (confirmed by SPDX headers), `autobrr/go-deluge` |
| MIT | Seerr, Recyclarr, Unpackerr, `xtractr`, `autobrr/go-qbittorrent`, `go-rtorrent`, `go-torrent`, `go-cache`, the `rls` fork, `regexp2`, and the original Go Cardigann (its author's own files only) |
| MPL-2.0 | autobrr's `sse` fork, `golang-lru` (an `xtractr` dependency). Compatible; needs file-level source availability |
| No license file | The Prowlarr/Indexers definitions repo. 504 of its 557 v11 files are byte-identical to Jackett's, so they are still GPL-2.0 via Jackett; only Prowlarr's own changes have no stated license |

Porting parser regexes, decision specifications or schemas is very likely a derivative work. For the **Cardigann lineage** specifically:
- Prowlarr's GPL-3.0 engine is derived from Jackett's GPL-2.0-only code (a shared FIXME comment, 142 verbatim lines, no attribution).
- The YAML definitions are effectively Jackett's files, and Prowlarr's copy has no license file.
- Jackett's engine itself began as a C# port of the MIT Go Cardigann. Literally translated code keeps its MIT permission, but it can't be separated out in practice.

So "use Prowlarr as the reference" doesn't avoid the GPL-2.0-only problem.

**Options:**
- **A.** GPL-3.0 for the monolith, with a **clean Cardigann engine**:
  - Seed it from the MIT Go Cardigann (2020), taking only its author's own files. The seed is not fully permissive: the engine imports `cardigann/releaseinfo` (GPL-3.0, ported from Sonarr's parser) and `bcampbell/fuzzytime` (GPL-3.0-or-later), which are compatible with GPL-3.0, and `cardigann/go-imdb-scraper`, which has no license. Replace `releaseinfo` with our parser and drop `go-imdb-scraper`.
  - Extend it to the v11 schema by testing its behaviour against the definitions and recorded HTTP fixtures, not by translating C#.
  - **Download the definitions at runtime** into the data folder, never bundling them and never embedding them in the binary. The argument rests on two points. First, the project never distributes the YAML, and the GPL governs distribution, not private use (GPLv2 §0). Second, the definitions are data to the engine ([FSF FAQ #IfInterpreterIsGPL](https://www.gnu.org/licenses/gpl-faq.html#IfInterpreterIsGPL)). Don't rely on "mere aggregation" ([#MereAggregation](https://www.gnu.org/licenses/gpl-faq.html#MereAggregation) says modules in the same executable are combined, so `go:embed` would weaken the case). If the project ever mirrors or bundles the YAML, ship it as separate GPL-2.0 files with Jackett's LICENSE, outside the executable.
- **B.** AGPL-3.0 (if Profilarr code is ported substantially), with the same Cardigann approach.
- **C.** A permissive license with clean-room implementations everywhere. Not realistic, since it forfeits the golden corpora, and the Go Cardigann seed isn't permissive either.

**Recommendation:** **A.**
- Re-implement Profilarr's *design*, not its code.
- Reimplement native indexers from API documentation and observed behaviour, never by translating Jackett's C#.
- **Get legal review of the Cardigann/definitions approach before the first public release.** This includes whether reproducing the v11 schema (its filter names and semantics were authored in Jackett) is a problem.
- The `autobrr/go-*` and `rls` licenses are resolved: all MIT except `go-deluge`, which is GPL-2.0-or-later and compatible with GPL-3.0. Run `go-licenses` (or similar) over the final module graph before release, since transitive dependencies weren't audited.

---

## D6. Frontend stack

**Status:** **Decided** (2026-09-27)

**Decision:** a **SvelteKit static SPA** (Svelte 5, static adapter), embedded in the Go binary via `go:embed`, with the API contract generated from OpenAPI. Profilarr's profile editor and testing screens can be ported as code (allowed under AGPL-3.0, D5). The Sonarr, Seerr and Bazarr screens are rebuilt in Svelte.

The analysis below is kept for reference; its Verification note is superseded by this decision.

**Context:** There are six UI stacks across the set:
- React at several vintages: Sonarr (React 18, TS, Vite, react-query, zustand) and Whisparr v3 (React 19, TS, zustand, react-query, webpack) are the most modern; Lidarr and Bookshelf are 80%+ JavaScript (by lines) on Redux.
- Next.js 16 / React 19 (Seerr).
- React 19 / Mantine (Bazarr).
- React 19 / TS, Vite, react-query, Tailwind (autobrr).
- SvelteKit / Svelte 5 (Profilarr), and Svelte 5 (Unpackerr).
- Angular (Cleanuparr).
- jQuery (Jackett) and vanilla JS (Kapowarr).

React is used by 10 of the 15 UIs (Recyclarr has none). The monolith needs one UI covering every domain.

**Options:**
- **A.** A React + TypeScript SPA embedded via `go:embed`.
- **B.** Server-rendered Go templates with htmx or Alpine. This makes rich editors (profiles, interactive search, calendars) harder.
- **C.** A SvelteKit static SPA embedded in the binary.

**Recommendation:** **A or C.** Either way, generate the API contract from OpenAPI.
- React matches Sonarr, Seerr, Bazarr, autobrr and Whisparr v3 (the richest UIs to port) and has the larger ecosystem.
- Svelte gives smaller bundles and matches Profilarr and Unpackerr.

Choose based on team familiarity.

**Verification note:** counting autobrr and Whisparr v3 (both React 19), which the original context left out, tilts the evidence further towards **A**. The recommendation is unchanged.

---

## D7. External API compatibility

**Status:** **Decided** (2026-09-28)

**Decision:** **no external compatibility layers.**
- Contracts are kept only for maintained outside apps that don't duplicate the monolith's functionality. None of the known consumers qualifies: Mylar3/LazyLibrarian and nzbToMedia/SAB scripts duplicate it; LunaSea and Doplarr are dead; Requestrr, Maintainerr, Homepage, Home Assistant, nzb360 and Notifiarr are **dropped** (no compatibility, no built-in replacements for now).
- **No** Sonarr/Radarr v3, Lidarr/Bookshelf v1, Seerr v1 or Newznab/Torznab server façades.
- **Webhooks and custom scripts use the monolith's own documented format only.** There are no existing custom integrations to preserve.
- **Observability is a first-class requirement:** the app must expose **metrics** in a form Prometheus can scrape, and dispatch **alerts** (health issues and failures) to external alerting systems. The notification/event system is designed so metrics and alert sinks plug in alongside user-facing notifications.
- If a simple outside app becomes important later, prefer building its function in over adding a compatibility layer.

The analysis below is kept for reference.

**Context:** Once every consumer in the set is absorbed, their Arr API calls become in-process (§4). The remaining contracts are with apps and code **outside** the monolith (§5.3):

| Contract | External consumers | Size of the surface |
|---|---|---|
| Sonarr/Radarr `/api/v3` | Maintainerr, Homepage, LunaSea (archived)/nzb360 (closed source, unverified), Notifiarr (via `golift.io/starr`), nzbToMedia/SAB scripts (`Downloaded*Scan`), Requestrr, Doplarr (archived), Home Assistant (`aiopyarr`), autobrr/Seerr instances users keep running | Large. Seerr's subset alone is about 20 operations with quirks (§1.1) |
| Lidarr/Bookshelf `/api/v1` | Homepage, Notifiarr, nzbToMedia (`DownloadedBooksScan`, Lidarr `command`), Home Assistant (`aiopyarr`), LunaSea and Requestrr (Lidarr), scripts | Medium. The `command` endpoint with `Downloaded*Scan` is worth doing early |
| Newznab/Torznab server | Mylar, LazyLibrarian, and other tools that expect Prowlarr | Small (the XML writer and caps) |
| Seerr `/api/v1` (with `X-Api-Key`; only Requestrr and Doplarr also send `X-API-User`) | Maintainerr, Requestrr/Doplarr, Homepage, mobile apps, Home Assistant | Medium: `request`, `media`, `search`, `movie`, `tv`, `issue`, `status` |
| Bazarr API | Dashboards | Small |
| Webhook payloads, `*_EventType` custom-script variables, Seerr webhook template variables | User scripts and receivers | Small but **high value** |

The consumer list was checked on 2026-09-27 by cloning each consumer and searching its code ([verification/decision-premises.md](verification/decision-premises.md) §D7); only nzb360 (closed source) is unverified. Tautulli and Kavita are **not** consumers: other tools call Tautulli, and Bookshelf notifies Kavita (a notification target the monolith must implement). Façades must accept all the auth styles consumers use today: `?apikey=`, `?access_token=` and the `X-Api-Key` header.

**Options:**
- **A.** No compatibility: a clean new API only. Every external tool breaks until it adds support.
- **B.** A **native API plus thin compatibility façades**, versioned and optional:
  - a Sonarr/Radarr v3 subset (the endpoints the known consumers use);
  - a Newznab/Torznab server;
  - a Seerr `/api/v1` subset;
  - the webhook and custom-script payloads, kept exactly.
- **C.** Full emulation of every app's API. This is very large, and it freezes the old models: one file per item, per-app ids.

**Recommendation:** **B.** Keep the webhook, custom-script and Newznab contracts exactly, because they're small and high-value. Build the v3, v1 and Seerr façades incrementally, driven by the known consumers.

A façade maps multi-version items (D4) onto "virtual instances". For example, the 4K slot appears as a second Radarr, for tools that assume one file per item.

---

## D8. Release parser foundation

**Status:** **Decided** (2026-09-28): spike first.

**Decision:**
1. Port the golden test corpora first (Sonarr, Radarr, then Lidarr, Bookshelf, Whisparr and Kapowarr fixtures). They are needed whichever approach wins.
2. Run a **time-boxed spike** measuring `rls` (autobrr fork, MIT) as a tokeniser and attribute layer against those corpora.
3. Choose between **porting the Arr cascades** (on regexp2) and a **hybrid** (`rls` for tokens and attributes, ported cascades for numbering) from the measured pass rates. Either way, D1a's reference and acceptance suite apply.

The analysis below is kept for reference.

**Context:** There are nine parsers today (§2.3):
- the .NET regex cascades in Sonarr, Radarr, Lidarr, Bookshelf and Whisparr;
- guessit (Bazarr);
- `rls` (autobrr, Go, all media kinds in one lexer, pure RE2; MIT; 418 test cases, of which only 6 are books, 4 comics and 4 audiobooks);
- Profilarr's partial C# port;
- Kapowarr's filename parser.

Golden corpora exist for Sonarr (1,631 cases), Radarr, Lidarr and Bookshelf identification, and Kapowarr.

**Options:**
- **A.** Port the Sonarr and Radarr regex cascades to Go (needs regexp2), then add the other kinds.
- **B.** Adopt `rls` (autobrr's fork) as the base lexer. Extend it until it passes the ported corpora, and map its output onto the canonical quality model.
- **C.** A hybrid: `rls` for tokenisation and attributes (quality, codec, group, edition), with the ported cascades for numbering (scene/anime/daily, dates, product codes).

**Recommendation:** **Evaluate B and C in a spike before committing.** Port the corpora first, since they are needed in every option. Then measure the `rls` pass rate. The `rls` fork's license is resolved: MIT, like its upstream.

---

## D9. Community profile data

**Status:** **Decided** (2026-09-28)

**Decision:**
- **D9a, PCD-native.** Profilarr's PCD model is the monolith's native profile store: git-linked databases, base/user op layers, override/align/ask conflict strategies, and authoring and publishing back to git. **Dictionarry is the default linked database.** Everything that exists only to sync into external Arrs is dropped (sync, drift, cleanup, per-app enum mapping). TRaSH is supported through Dictionarry's `trash-pcd` port; there is no direct TRaSH importer.
- **D9b, keep the format, new engine.** Stay compatible with the PCD on-disk format (`pcd.json`, `ops/*.sql`, the semver'd schema dependency) so existing databases work, but materialise into the monolith's own database and apply user overrides incrementally instead of Profilarr's full in-memory recompiles.
- Because the store is native, the monolith owns the quality, language and flag enums; `quality_api_mappings` and similar per-app tables are not needed.

The analysis below is kept for reference.

**Context:** Two ecosystems publish quality profiles and custom formats:
- **TRaSH Guides:**
  - plain JSON with stable 32-hex `trash_id`s;
  - named score sets and custom-format groups;
  - consumed by Recyclarr and others;
  - Dictionarry republishes it as `trash-pcd`.
- **Dictionarry PCDs:** SQL op files over Profilarr's schema, keyed by name, with local-override conflict resolution.

Both store .NET regexes. TRaSH also stores **raw per-app Arr enum ids**, which must be reverse-mapped when imported.

**Options:**
- **A.** Profilarr's relational model as the native store, with a **TRaSH importer as primary** and a **PCD importer as secondary**. The PCD importer must replay the dataset's SQL ops against Profilarr's schema (for example in a scratch SQLite database) and read the entities back: Profilarr's portable JSON form exists but is feature-flagged off (`importExport: false`), and every dataset ships SQL ops.
- **B.** PCD format as the native store (full Profilarr compatibility), which couples us to Profilarr's schema versioning and SQLite semantics. (Even option A's importer depends on that schema, which weakens B's advantage further.)
- **C.** TRaSH only.

**Recommendation:** **A.** Add these to the native model:
- an `external_ref (source, id)` on every entity, so re-imports match by stable id, not name;
- named score sets;
- custom-format groups as optional bundles;
- quality-size and naming presets;
- an override layer that survives upstream updates. Profilarr's override/align/ask strategies are the model, implemented as structured patches rather than SQL.

Use Recyclarr's non-destructive defaults: never zero unmanaged scores, and never delete what we didn't create.

---

## D10. Adult model: Whisparr v2 or v3

**Status:** **Decided** (2026-09-28)

**Decision:** model adult content (phase 3) on **Whisparr v3**: scene and movie items with first-class, followable **Studio** and **Performer** entities (with an `AfterDate` cut-off) and multi-provider ids (StashDB, ThePornDB, TMDb, IMDb). Restore from v2:
1. product codes as indexed alternate ids, with a code-based fallback for matching completed downloads;
2. reject ambiguous matches everywhere, including fuzzy matching;
3. per-indexer search settings;
4. adult categories by default for every indexer type.

"Following an entity" is a generic concept, reusable for actors, directors and authors. StashDB and ThePornDB use per-user keys (D3).

The analysis below is kept for reference.

**Context:**

| | v2 (`whisparr`) | v3 "Eros" (`whisparr-eros`) |
|---|---|---|
| Upstream | Sonarr fork; 54 commits behind | Radarr fork; **0 commits behind** (cherry-picks, with a tracked decision per upstream commit); frontend is React 19 with zustand and react-query, no Redux, on webpack (Sonarr is React 18 on Vite) |
| Model | Site = series, year = season, scene = episode; performers embedded in scenes; ThePornDB ids stored in `TvdbId` | Scene and movie items (an `ItemType` flag on Radarr's movie table); **first-class Studio and Performer** entities, each monitorable with an `AfterDate` cut-off |
| IDs | ThePornDB only | StashDB, ThePornDB, TMDb and IMDb ids plus a primary id; scenes keyed by StashDB id |
| Metadata | `api.whisparr.com/v3` proxy, full refetch | `api.whisparr.com/v4` proxy (appears to combine StashDB, ThePornDB and TMDb; closed source). Import lists also call StashDB (user key) and TMDb directly. Said to refresh only what changed (not verified in code) |
| Matching | Date parser, same-date matcher, **rejects ambiguous matches**, indexed (non-unique) product codes with a code-based fallback for completed downloads | More date formats, matches StashDB ids in release names. **Regressions:** a fuzzy match that takes a tied top score; product codes no longer indexed and no code-based completed-download fallback (title matching still tries the code) |
| Search settings | Per indexer | Global only (regression) |
| Default categories | Adult (Newznab 6000 range; FileList XXX) | Adult for Newznab; FileList regressed to movie categories |
| Maturity | Maintained, not deprecated | 21 stable releases (2026-01-15 to 2026-09-24, with a gap from late March to mid-June), one main maintainer; **no migration from v2** |

**Options:**
- **A.** Model on v2: a TV-style container with date-keyed items.
- **B.** Model on v3, with v2's safeguards restored.
- **C.** Support both models.

**Recommendation:** **B.** v3's model fits a generic item with an optional container and many linked people, and it is where upstream is heading. Restore four things from v2:
1. index product codes as alternate ids, and add a code-based fallback for grab, completed-download tracking and import;
2. reject ambiguous matches everywhere, including the fuzzy step;
3. search settings per indexer;
4. adult categories as the default for every indexer type (a small fix: only FileList regressed).

Call StashDB, ThePornDB and TMDb directly instead of the proxy (D3). StashDB and ThePornDB need per-user keys (no project-level key exists), so degrade gracefully without them; replicating the proxy's cross-source id linking is real work. Performer and studio monitoring generalises to "item-level monitoring from a followed entity", which could also serve actors, directors or authors.

---

## D11. Direct-download (DDL) acquisition

**Status:** **Decided** (2026-09-28)

**Decision:** build an **optional, isolated DDL module in phase 3**, as part of the comics phase. It is driven by declarative site and host definitions (GetComics, GetComics' own servers, and a small set of file hosts), shipped and updated separately from the core, with fixture-based tests. DDL is a third protocol alongside Usenet and torrent: a release is a set of link groups, resolved at grab time with fallbacks, and downloaded by built-in per-host downloaders with quota tracking. Comics also get Usenet and torrent through the unified indexer layer.

The analysis below is kept for reference.

**Context:** Kapowarr gets comics by scraping GetComics and downloading from file hosts (MEGA, MediaFire, Pixeldrain, WeTransfer) and from GetComics' own servers. It also sends GetComics torrent (magnet) links to qBittorrent or Transmission, but has no torrent indexer, Usenet or Newznab support. DDL would be a **third protocol** alongside Usenet and torrent. It needs:
- page-scraping indexers where one release has several groups of mirror links;
- resolving the actual download links at grab time, with fallbacks;
- downloaders for each host, with their quotas;
- CloudFlare solving.

It is the most breakage-prone and piracy-adjacent code in the whole set. Mylar3, the other comic manager, supports Usenet and torrents, which the unified indexer layer would give comics for free, and also direct download, so Kapowarr isn't the only DDL comic manager.

**Options:**
- **A.** Support comics through Usenet and torrent only, via the unified indexer layer. No DDL.
- **B.** DDL as an **optional, isolated module**, driven by declarative site and host definitions (like Cardigann), shipped separately from the core. Built-in downloaders for a small set of hosts, including GetComics' own servers.
- **C.** Port Kapowarr's DDL as-is, in core.

**Recommendation:** **A for the comics phase, with B as a later, optional add-on** if there's demand.
- DDL is how most comic users get content today, so dropping it is a real product decision, not just a technical one.
- **Keep the interfaces DDL-ready** (a release made of groups of links, and a resolve step at grab time) so B doesn't need a pipeline redesign.

---

## D12. Client–server architecture

**Status:** **Decided** (2026-09-28)

**Decision:**
- A **headless Go core server** holds all business logic and exposes the REST API (`/api/v1`, OpenAPI), live events (SSE) and `/metrics`.
- **Clients share one SvelteKit UI** (D6): the web client served by the core, a **Tauri desktop app**, and later **Tauri mobile apps**. Other clients are generated API SDKs, a CLI, and notification/metrics sinks.
- **Crux is not used.** It shares logic across *client* apps (a Rust core compiled into each app, Elm-style), which doesn't fit logic that must run continuously on a server.
- **Entry-level users:** the desktop app installs the core as an **OS service** and pairs with it through a local handshake (localhost discovery plus a one-time token in the data directory, exchanged for a scoped API key). **Advanced users** run the core as a container and point any client at it.

## D13. Database and ORM

**Status:** **Decided** (2026-09-28)

**Decision:** **ent** (entgo.io) with **Atlas versioned migrations**, generated per database from one schema. Supported databases: **PostgreSQL** (recommended for production, required for HA), **MySQL/MariaDB**, and **SQLite** (development, testing and small single-node installs). SQL Server is not supported.

## D14. High availability

**Status:** **Decided** (2026-09-28)

**Decision:** the architecture is HA-capable; a single node remains the default deployment.
- **Active-active API nodes** behind a load balancer; nodes are stateless, with state in the database.
- **Singleton work** (IRC connections, download-client polling, RSS and other schedules, profile-database git pulls) runs under **leader election** using database leases or advisory locks, with failover.
- A **durable database-backed job queue**; workers on any node claim jobs with `SELECT … FOR UPDATE SKIP LOCKED` (single-node on SQLite).
- **Cross-node events** via Postgres `LISTEN/NOTIFY`.
- **Postgres is required for HA.** Media, download and data paths must be shared storage, mounted at the same paths on every node.

## D15. Plug-in boundary

**Status:** **Decided** (2026-09-28)

**Decision:** one **WebAssembly plug-in boundary** (run in-process with `wazero`, sandboxed, with capabilities granted through host functions such as HTTP through the core's rate-limited client). It is used for extension points such as subtitle providers, and is available to indexers, DDL hosts and notification targets. Unverified native code is never loaded into the process (unlike Lidarr's plug-in loader). Consequence: plug-ins are written in languages that compile to WebAssembly (Go, Rust, and similar), so Bazarr's Python providers are ported rather than wrapped.

## D16. Secrets, racing mode and self-updater

**Status:** **Decided** (2026-09-28)

**Decision:**
- **Secrets:** deployment secrets (service keys, passwords, tokens) are stored in the database; encryption at rest is the database's responsibility. SQLite relies on disk encryption, and backups must be protected the same way. The binary embeds only project-registered public API keys (D3b).
- **Racing mode** (autobrr-style library-less grabs for ratio building) is **not in phase 1**; it is an optional feature in a later phase, built on the phase-1 IRC ingest and rule engine.
- **Self-updater:** deferred until the core architecture is established.

## D17. Authentication and authorisation

**Status:** **Decided** (2026-09-28)

**Decision:** native support for both **OIDC** and **built-in authentication** backed by the monolith's database, with **well-established default roles and permissions** (role-based access control). Media-server identity (Plex/Jellyfin/Emby login for request users) and scoped API keys build on the same user and permission model.

## D18. Deployment range

**Status:** **Decided** (2026-09-28)

**Decision:** deployment must cover the whole range:
- **Kubernetes, HA:** multiple core replicas (D14), with the web UI and native apps connecting remotely.
- **Single user terminal:** the core installed as a local service with a native app UI (D12).

**Tauri** is the preferred way to bundle the UI into native apps.

## D19. Porting principle: redesign, don't replicate

**Status:** **Decided** (2026-09-28)

**Decision:** the reference apps are specifications of *outcomes*, not of *implementations*. When porting:
- Keep user-visible behaviour and correctness invariants (for example parser results, scoring semantics, numbering rules).
- Do not copy workflows that exist because the reference apps are separate processes, grew over a decade, or work around each other (sync layers, polling mirrors, text-matching on status messages, multi-instance workarounds).
- Prefer the simplest logically correct design over faithful recreation, and re-derive workflows from first principles where the reference is convoluted.
- Getting the design logically correct from the start takes priority over speed of porting.

## D20. Design-review outcomes

**Status:** **Decided** (2026-09-28). Source: [design/open-issues.md](design/open-issues.md), which also lists the 20 accepted cross-design resolutions (group A).

**Decision:**
- **D4 amended:** one download can feed **every slot that wants it**; each slot gets its own library file (hardlink/reflink where possible, else copy) and upgrades independently. Slot definitions are global per media kind, configured per title, monitored per item; preferred edition and monitored release belong to the slot.
- **D9 amended:** for media kinds the PCD schema can't express (music, books, comics, adult), propose additive changes to `Dictionarry-Hub/schema` first, and fall back to **our own extension schema**. Announce-metadata conditions, autograb rules and slot/tag-scoped delay settings are **in the extension schema** and publishable. The project maintains starter databases for the new kinds.
- **Attribute authority:** the ported Sonarr/Radarr parser tests define attribute detection (source, resolution, modifier, language); Dictionarry defines custom-format patterns. Disagreements are bugs reported upstream.
- **D13 amended:** **MySQL/MariaDB dropped.** Supported databases are PostgreSQL (production, required for HA) and SQLite (desktop, small installs, development, tests).
- **Import-time processing:** an optional **external processing hold state**: import pauses, emits an event, and resumes when an external tool marks the file ready or failed through the API.
- **Release selection is coverage-aware:** among releases that pass the profile, prefer the combination that satisfies more wanted items, then profile score and quality.
- **"Ask" conflicts:** the user's value stays in effect until the user decides.
- **D3a risk:** apply for TVDB's free/OSS licensed tier before phase 1; if refused, fall back to TMDb TV with an episode-order mapping layer.
- **D16 amended:** optional passphrase encryption for backup archives, plus a clearly labelled sanitised export. Bootstrap secrets (database DSN, TLS key, OIDC bootstrap) live in a bootstrap config file or environment variables.
- **TMDb 180-day rule:** if TMDb data can't be refreshed in time, purge TMDb descriptions and cached images but keep TMDb ids.
- **Legal review scope (D5b)** also covers: fetched definitions stored in user databases and backups; the unlicensed Prowlarr/Indexers repository; community anime/scene datasets; third-party WASM plug-in licensing; AcoustID non-commercial keys under AGPL; Hardcover's user-data clause.
- **No mobile push in v1.** Forward-auth (trusted-header) login is **opt-in and off by default**.
- **D12 amended:** desktop pairing uses a **local socket / named pipe with OS peer credentials** (no token file), and issues a revocable **device credential**.
- **D14 amended:** schedules and polling run on every node via schedule rows claimed with `SKIP LOCKED`; leases only for long-lived connections (IRC, file watcher). Fetched definitions, PCD data and plug-ins are stored in the database. A DB-backed rate limiter is shared across nodes. LISTEN/NOTIFY is a best-effort doorbell only.

