# Jackett — Codebase Analysis (with a diff against Prowlarr)

Submodule: `submodules/jackett` (HEAD `655690496`, 2026-09-26, `git describe` = `v0.24.2670-2`). All paths are relative to that root unless marked otherwise. `JC/` = `src/Jackett.Common/`.

To compare the two projects, I also shallow-cloned two other repos into a scratch directory, outside this repo:
- **Prowlarr/Indexers** (HEAD `223d3b1`, 2026-09-26), cited below as `PIDX/`.
- The **original Go Cardigann** (`cardigann/cardigann`, HEAD `4ad57c5`, 2020-04-26), cited below as `CG/`.

Prowlarr paths are relative to `submodules/prowlarr`. Read this alongside [prowlarr.md](prowlarr.md) and [comparison.md](comparison.md).

**TL;DR**
- Jackett is the **upstream** for Cardigann YAML. Prowlarr/Indexers pulls definitions from Jackett automatically, three times a day.
- The two YAML formats are **~99% the same**:
  - The JSON schemas differ in exactly one `required` entry.
  - 504 of the 556 shared definitions are byte-identical.
- The two C# engines have drifted in a handful of small ways (§3.1).
- Jackett's license is **GPL-2.0, with no "or later" grant anywhere**. Treat it as **GPL-2.0-only** (§1.4).
- The original engine is Go and **MIT**-licensed (`cardigann/cardigann`). It is the cleanest-licensed starting point for a Go reimplementation.

---

## 1. Overview

### 1.1 Purpose
Jackett is a single-process Torznab/TorrentPotato proxy (README "Introduction").
- **Supported trackers**: native C# indexers plus Cardigann YAML scrapers, all bundled in the binary.
- **Endpoints**: one Torznab endpoint per tracker, plus `all` (aggregate) and filter meta-endpoints.
- **Clients**: Sonarr, Radarr, etc. add each Jackett endpoint by hand as a "Torznab" indexer.
- **What it does not have**: app sync, a database, download-client integration (only a blackhole folder), history, or stats.

### 1.2 Stack / versions
- **Runtime targets**:
  - .NET 9: `net9.0` for Server, Updater and tests; `net9.0-windows` for Tray and Service.
  - .NET Framework 4.7.1: **`net471`**. This is the Mono / full-framework build, and it is still built in CI (`azure-pipelines.yml:108-114`, "Jackett.Binaries.Mono.tar.gz").
  - `JC` and `DateTimeRoutines` target `netstandard2.0;net9.0`.
  - Sources: `src/*/​*.csproj`, `azure-pipelines.yml:9-10` (`net9.0`, SDK `9.0.x`).
- **Web**:
  - ASP.NET Core via `Microsoft.AspNetCore.*` **2.3.13** packages, needed so net471 can keep compiling.
  - There are `#if NET471` dual code paths in `src/Jackett.Server/Startup.cs:73-89`.
  - Controllers use MVC with Newtonsoft JSON.
- **Libraries**:

  | Purpose | Library (version) |
  |---|---|
  | DI | Autofac 8 |
  | Logging | NLog 5.5 |
  | Retries | Polly 8.8 (`JC/Indexers/BaseIndexer.cs:468`) |
  | Cardigann YAML | YamlDotNet **18.1** |
  | HTML | AngleSharp 1.8 + AngleSharp.Xml |
  | Torrent re-encode | BencodeNET 4 |
  | FlareSolverr client | FlareSolverrSharp 4 |
  | SOCKS proxy | DotNet4.SocksProxy |
  | Updater tar/zip | SharpZipLib |
  | Mono support | Mono.Posix |
  | CLI flags | CommandLineParser |
  | Tests | NUnit, MSTest, FluentAssertions, Selenium (integration tests) |

- **Frontend**: a static jQuery/Bootstrap/Handlebars SPA (`JC/Content/`). It has about 1.75k LOC of custom JS plus vendored libs: jQuery, DataTables, Handlebars, moment, tagify and bootstrap-multiselect. There is no React and no build step.
- **Releases**: continuous. Every merge ships a release (0.24.**2670**), because **definitions are only delivered inside the binary** (§3.1.6). Over the last 30 days there were 195 commits; 164 of them touched `Definitions/`.

### 1.3 LOC (`.cs`, including blank lines)

| Project | LOC | Files |
|---|---|---|
| `src/Jackett.Common` | 37,301 | 226 |
| ↳ `JC/Indexers/Definitions/*.cs` (65 native + Cardigann) | 24,001 | |
| ↳ `JC/Indexers/Definitions/CardigannIndexer.cs` | 2,559 | 1 |
| `src/Jackett.Test` | 3,850 | 43 |
| `src/Jackett.Server` | 2,995 | 20 |
| `src/Jackett.Updater` | 1,173 | 2 |
| `src/DateTimeRoutines` (vendored fuzzy date parser) | 606 | 1 |
| `src/Jackett.Tray` (WinForms) | 527 | 4 |
| `src/Jackett.IntegrationTests` | 168 | 2 |
| `src/Jackett.Service` (Windows service) | 153 | 3 |
| `JC/Definitions/*.yml` | 563 files + `schema.json` (1,025 lines) | |
| `JC/Content` custom JS | ~1,750 | |

Jackett is about 45k C# LOC in total, against about 110k for Prowlarr. The Cardigann engine is a **single 2,559-line class**. Prowlarr splits the same engine across `Cardigann/*.cs`, 3,488 LOC over 9 files.

### 1.4 Relationship to Prowlarr and to the original Cardigann
- **Cardigann (Go, MIT)**:
  - `CG/LICENSE`: "MIT License, Copyright (c) 2016 lxt".
  - Written in Go, using real `text/template` (`CG/indexer/runner.go:18`).
  - Has 58 definitions and 11 filters (`CG/indexer/filters.go`: querystring, timeparse/dateparse, regexp, split, replace, trim, append, prepend, timeago/fuzzytime/reltime).
  - The README (`CG/README.md:267`) says it was "Inspired by Jackett, or at least born of frustration with it always crashing and requiring a mono runtime."
  - Dormant since 2020.
- **Jackett** ported the Cardigann YAML engine to C#. Its template support is a **regex-based imitation** of Go templates (§3.1.4), not a real template engine. Jackett extended the format a lot. It is now the **source of truth** for the definition format and the definition set.
- **Prowlarr** forked Jackett's C# engine into `NzbDrone.Core/Indexers/Definitions/Cardigann/`, where comments still mention Jackett, e.g. `CardigannBase.cs:782` "FIXME: fix jackett header handling".
  - Its definitions live in the separate **Prowlarr/Indexers** repo. `PIDX/README.md:13-16`: "Prowlarr Cardigann Indexers are synced upstream with Jackett via the indexer-sync script".
  - The sync runs by GitHub Action 3×/day to an `automated-indexer-sync` branch that is merged by hand (`PIDX/CONTRIBUTING.md:231-251`, `PIDX/scripts/indexer-sync-v2.sh`).
  - Jackett files are excluded through `PIDX/scripts/blocklist.txt`.

### 1.5 License (precise)

| Evidence | Finding |
|---|---|
| `LICENSE` (340 lines) | **GitHub's GPL-2.0 template** of the GNU GPL version 2 (June 1991): the operative terms are identical to GPLv2, but it is not byte-verbatim FSF text (the "How to Apply" appendix keeps the template placeholders `{description}` / `Copyright (C) {year} {fullname}` at `LICENSE:293-294`, and `LICENSE:4-5` has the old FSF street address). The only "any later version" wording is in §9 and in the appendix **sample notice** (`LICENSE:244-245, 297-299`). Neither is a grant by the Jackett authors. **Unchanged since the repo's first commit** (`git log --follow LICENSE` → `cdc829cec Initial commit`). |
| `README.md:1533-1542` "License and Credits" | Names no license and no version at all. |
| Source headers | **None**. `grep -ri "GNU General Public\|GPL\|any later version\|SPDX"` over the whole repo matches only `LICENSE` (plus two false hits inside base64 in `Jackett.Tray/Main.resx`). |
| csproj / NuGet metadata | No `PackageLicenseExpression`. `Jackett.Server.csproj:76` just copies `LICENSE` to the output folder. |
| COPYING / NOTICE / AUTHORS | **None exist.** |
| GitHub license detection (`gh api repos/Jackett/Jackett`) | `spdx_id: "GPL-2.0"`. This is the deprecated, ambiguous SPDX id; it does not say "-only" or "-or-later". |
| Prowlarr/Indexers | **No LICENSE file**; the GitHub API returns `license: null`. The definitions it ships are copies of Jackett's GPL-2.0 files. |
| Prowlarr / Sonarr / Radarr | GPL-3.0 (comparison.md §0). |

**Conclusion.** Jackett is licensed under **GPL version 2, and nowhere grants "or (at your option) any later version"**.

There is one arguable loophole. GPLv2 §9 says "If the Program does not specify a version number of this License, you may choose any version ever published". One could argue that a bare LICENSE file does not "specify" a version. However:
- The FSF's own guidance is that a copy of the license alone is not even an explicit grant.
- The only version-bearing artifact is the v2 text itself.
- There are hundreds of contributors and no CLA.

So for compatibility planning, treat Jackett as **GPL-2.0-only**. **GPL-2.0-only code cannot be combined into a GPL-3.0 work.**

Consequences:
- **Porting Jackett C# code, or copying its engine, into a GPL-3.0 monolith is not safe.**
- The same issue technically clouds **Prowlarr's** Cardigann engine (GPL-3.0 on top of GPL-2.0-derived code) and the **unlicensed Prowlarr/Indexers** YAML set. Reimplementing from Prowlarr does not launder the Jackett lineage.
- The clean paths are:
  - build the engine from the **MIT Go Cardigann**, plus a behavioural spec (the schema and the test corpus), written without translating C#; and
  - treat definitions as **runtime-loaded data fetched from upstream** that is never vendored into the source tree or binary (§11).

(Not legal advice. Get counsel before release.)

---

## 2. Domain model

Jackett has **no database**. All state is JSON files in the data folder:
- Unix: `~/.config/Jackett`. Windows: `%ProgramData%\Jackett`. Override with `-d/--DataFolder`.
- Sources: `JC/Services/ConfigurationService.cs`, README "Quick Reference".

| Entity | Storage | Notes |
|---|---|---|
| `ServerConfig` | `ServerConfig.json` (`ConfigurationService.cs:161` = `type.Name + ".json"`) | Fields in §7. Legacy migration from `config.json` / `api_key.txt` (`:268-290`). |
| Indexer instance config | `Indexers/{indexerId}.json` + `.bak` (`JC/Services/IndexerConfigurationService.cs:129-130`) | A **JSON array of ConfigurationItems** `[{id, type, name, value}]` (`JC/Models/IndexerConfig/ConfigurationData.cs:38-51`), not a typed object. Always present: `SiteLink`, `Tags`, hidden `CookieHeader` (**the login session cookie is persisted here**) and hidden `LastError`. Passwords are encrypted with ASP.NET DataProtection (`PasswordConfigurationItem`; `Jackett.Server/Services/ProtectionService.cs`). The key purpose string is hard-coded, overridable with env `JACKETT_KEY`. Keys are stored in `DataProtection/`. |
| Indexer identity | **The id is the key.** There is exactly **one instance per indexer id**: you cannot configure the same tracker twice. The id doubles as the filename and the URL segment. | Contrast Prowlarr, which gives each configured indexer a DB row with an int id and allows many instances per definition. |
| Renames | YAML `replaces: [oldid]` or C# `override string[] Replaces` | `JC/Services/IndexerManagerService.cs:61-110` renames the config file on startup, and `GetIndexer` (`:~290`) keeps the old id working in URLs with a warning. **Prowlarr has no such mechanism.** |
| Config-item types | `JC/Models/IndexerConfig/ConfigurationData.cs:103-363` | String, HiddenString, DisplayInfo, Bool, DisplayImage (captcha), SingleSelect, **MultiSelect**, Password, Tags. There are 15 standard ConfigurationData* presets (Basic login, 2FA, Cookie, CookieUA, APIKey, Passkey, PinNumber, …) and 21 `Bespoke/` configs for native indexers. |
| Caps | `JC/Models/TorznabCapabilities.cs`, `TorznabCatType.cs`, `TorznabCapabilitiesCategories.cs` | Built at runtime from C# or from YAML `caps`. Tracker-specific categories are exposed as `100000 + trackerCatId`, the same scheme as Prowlarr. `TvSearchImdbAvailable` is set by YAML `allowtvsearchimdb` (**Jackett-only**; see §3.1). |
| `ReleaseInfo` | `JC/Models/ReleaseInfo.cs` | Title, Guid, Link, MagnetUri, InfoHash, Details, PublishDate, Category[], Size, Files, Grabs, Seeders, Peers, Poster, Imdb/TMDb/TVDB/RageID/TVMaze/Trakt/Douban, Genres, Year, Author/BookTitle/Publisher/Artist/Album/Label/Track, DownloadVolumeFactor/UploadVolumeFactor, MinimumRatio/MinimumSeedTime. |
| Search cache | **In-memory only** (`JC/Services/CacheService.cs`, `TrackerCache*`) | See §3.6. |
| Log cache | In-memory ring (`LogCacheService.cs`) plus `log.txt` / `updater.txt` | |

---

## 3. Feature areas

### 3.1 Cardigann engine: a precise diff vs Prowlarr

#### 3.1.1 Location and count
- **Jackett**:
  - Definitions: `JC/Definitions/*.yml`, **563 definitions** (415 private, 87 public, 61 semi-private).
  - Schema: `JC/Definitions/schema.json` (JSON Schema 2019-09).
  - Loaded from, in order (`JC/Services/ConfigurationService.cs:214-240`):
    1. `~/.config/cardigann/definitions/` (Unix) or `%ProgramData%\Jackett\cardigann\definitions\` (Windows);
    2. `/etc/xdg/cardigan/definitions/` (sic: the misspelling is in the code, `:221`);
    3. the bundled `Definitions/` folder.
  - Because the first file loaded for an id wins and later duplicates are skipped (`IndexerManagerService.cs:~226`), **user overrides beat bundled definitions**, and a native C# id beats any YAML.
- **Prowlarr**:
  - Definitions: `PIDX/definitions/v11/*.yml`, **558 definitions**.
  - Fetched at runtime from `indexers.prowlarr.com/master/11` (prowlarr.md §3.2).
  - `PIDX/definitions/v1…v10/` now hold only `schema.json`; the older sets are frozen and removed.
  - `PIDX/VERSIONS`: `MIN=MAX=CURRENT=11`, `NEXT_VERSION=12`.

**Set diff** (by file id, case-insensitive):

| | Count | Ids |
|---|---|---|
| Shared | **556** | 504 byte-identical; 52 differ, mostly by 1–4 lines |
| Jackett only | 7 | `anirena, ildragonero, magnetz, rutracker-ru, therarbg, torrentgalaxyclone, u2p`. All 7 are on Prowlarr's sync `blocklist.txt`. |
| Prowlarr only | 1 | `brasiltracker`. Jackett replaced its YAML with native `BrasilTracker.cs`. |

Typical drift in the 52 differing files:
- Prowlarr-side `inputs: {} # TODO: remove in v10` leftovers (aither-api, reelflix-api, …).
- Sync lag: Prowlarr's `torrentleech.yml:71` lacks `imdbid` in `tv-search`.
- `infire`/`hdsky` `headers` moved between `search` and `download` blocks.
- `reelflix-api` featured/freeleech logic.

These are content lag, not format divergence.

#### 3.1.2 Schema / versioning
- **Jackett has no schema version.** The engine and definitions ship together in one binary, so a format change is atomic.
- **Prowlarr** pins `DEFINITION_VERSION = 11` because it fetches definitions independently of the binary. The sync script validates each Jackett file against each `vN/schema.json` (`PIDX/scripts/indexer-sync-v2.sh:201-241`). A file that fails is flagged: "Cardigann update likely needed. Version [v12] required."
- **Schema diff**: `diff <(json.tool JC/Definitions/schema.json) <(json.tool PIDX/definitions/v11/schema.json)` gives exactly **one** difference: `PageTestBlock.required`. Jackett requires `["path","selector"]`; Prowlarr requires only `["path"]`.
  - Everything else is identical, including `replaces` and `caps.allowtvsearchimdb`, which Prowlarr's schema accepts but Prowlarr's engine ignores.
  - Both schemas use `additionalProperties: false`.
- **Parser strictness differs**:
  - Jackett: `DeserializerBuilder()` **without** `IgnoreUnmatchedProperties` (`IndexerManagerService.cs:182-184`, where it is commented out). An unknown key means the definition fails to load.
  - Prowlarr: `.IgnoreUnmatchedProperties()` (`IndexerDefinitionUpdateService.cs:64-65`). Unknown keys are silently dropped.

#### 3.1.3 Model fields
Compare `JC/Models/IndexerDefinition.cs` with Prowlarr `Cardigann/CardigannDefinition.cs`. The blocks are field-for-field identical (the class names only change case, e.g. `loginBlock` → `LoginBlock`) **except**:

| Field | Jackett | Prowlarr |
|---|---|---|
| top-level `replaces: []` | ✔ used for id rename/migration (28 defs use it) | ✘ ignored |
| `caps.allowtvsearchimdb` | ✔ sets `TvSearchImdbAvailable` (`CardigannIndexer.cs:133`; added in `4efa1531d`; 11 defs) | ✘ ignored, so tv-search imdbid is not advertised |
| `login.captcha.image` | ✘ not a field; YAML would fail to load | throws "Deprecated, use login/captcha/selector" |
| `searchPathBlock.categories` default | `new List<string>()` | `null` |

Everything else matches, including defaults such as `Testlinktorrent=true`, `Inheritinputs=true` and `Queryseparator="&"`. That covers:
- top level: `id, name, description, type, language, encoding, requestDelay, links, legacylinks, followredirect, testlinktorrent, certificates, settings, caps, login, ratio, search, download`;
- `login`: `path, submitpath, cookies, method, form, selectors, inputs, selectorinputs, getselectorinputs, error, test, captcha, headers`;
- `search`: `path/paths, headers, keywordsfilters, allowEmptyInputs, inputs, error, preprocessingfilters, rows{after,dateheaders,count,multiple,missingAttributeEqualsNoResults}, fields`;
- `download`: `selectors, method, before{pathselector}, infohash{hash,title,usebeforeresponse}, headers`;
- `response`: `type, noResultsMessage`.

#### 3.1.4 Template language
Both engines implement the same regex-based pseudo Go template. Jackett's is at `CardigannIndexer.cs:325-535`.
- **Supported syntax** (identical regexes):
  - `{{ .Var }}`
  - `{{ if X }}…{{ else }}…{{ end }}`, where X may be `and`/`or`/`eq`/`ne` (`_LogicFunctionRegex :68`)
  - `{{ range [$i,] [$e :=] .X }}…{{.}}…{{end}}`
  - `{{ re_replace .X "pat" "rep" }}`
  - `{{ join .X "sep" }}`
- **Variables** (Jackett `GetBaseTemplateVariables :251-322`, `PerformQuery :1413-1520`):

| Variable | Jackett | Prowlarr |
|---|---|---|
| `.Config.sitelink`, `.Config.<setting>`, `.True`, `.False`, `.Categories`, `.Keywords`, `.Result.*` | ✔ | ✔ |
| `.Query.{Type,Q,Series,Ep,Season,Movie,Year,Limit,Offset,Extended,Categories,APIKey,TVDBID,TVRageID,IMDBID,IMDBIDShort,TMDBID,TVMazeID,TraktID,DoubanID,Album,Artist,Label,Track,Genre,Episode,Author,Title,Publisher,Keywords}` | ✔ | ✔ (Extended and APIKey are always null) |
| `.Query.Is{BookSearch,DoubanQuery,GenreQuery,IdSearch,ImdbQuery,MovieSearch,MusicSearch,RssSearch,Search,TVRageQuery,TVSearch,TmdbQuery,TraktQuery,TvdbQuery,TvmazeQuery}` (`:1452-1466`) | ✔ | ✘. Unused by any shipped definition (0 hits). |
| `.Today.Year` | **Year − 1 in January** (`:258`) | calendar year |
| checkbox setting value | `.True` → "True", `.False` → null (`:273`) | `".True"` literal string or null (`CardigannBase.cs:319`); truthy either way |

#### 3.1.5 Filters, selectors, settings, login and fields
- **Filters**: both engines implement the same 25 names (`CardigannIndexer.cs:1095-1250` vs Prowlarr `CardigannBase.cs`). The schema enums are identical:
  `querystring, timeparse, dateparse, regexp, re_replace, split, replace, trim, prepend, append, tolower, toupper, urldecode, urlencode, htmldecode, htmlencode, timeago, reltime, fuzzytime, validfilename, diacritics, jsonjoinarray, hexdump, strdump, validate`.
  - Row filters: `andmatch, strdump`.
  - Usage across Jackett definitions: re_replace 1,566; replace 838; dateparse 569; append 567; regexp 307; querystring 285; andmatch 60; timeparse 1.
  - Filters **no shipped definition uses**: `jsonjoinarray, hexdump, strdump, htmlencode, reltime`.
- **JSON selectors**: `:has(key)`, `:not(key)`, `:contains(key)` in both (`CardigannIndexer.cs:2503-2560`).
- **Selector modifiers** (both): `selector, optional, default, text, attribute, remove, filters, case`.
- **Settings types**:

| Type | Jackett | Prowlarr |
|---|---|---|
| `text, password, checkbox, select, info, info_cookie, info_flaresolverr, info_useragent, info_category_8000` | ✔ | ✔ |
| `multi-select` (`CardigannIndexer.cs:159`) | ✔ code only: not in `schema.json`, used by 0 definitions | ✘ (**prowlarr.md §3.2 is wrong to list it**) |
| `cardigannCaptcha` | ✘ (captcha is rendered as a DisplayImage item) | ✔ synthetic setting |

  - Select storage also differs. Jackett stores the option **key**. Prowlarr stores the **index into options sorted by key** (`CardigannBase.cs:296, 327`). This matters only when importing configs.
  - Usage counts: select 903, info 722, checkbox 600, text 551, password 214, info_cookie 138, info_flaresolverr 60, info_useragent 15, info_category_8000 11.
- **Login methods**: both engines support `post, form, cookie, get, oneurl` (`CardigannIndexer.cs:563-850`).
  - Usage across 563 definitions: cookie 138, form 123, get 117, post 93, none 91, default-form 1. `oneurl` is used by none.
  - Captcha blocks appear in 45 definitions.
- **Response types**: html 487, json 119, xml 26 search paths.
- **Row fields**:
  - Both engines handle the same set, **except `comments`**, which is Prowlarr-only (`CardigannParser.cs:481`) and used by 0 definitions.
  - Fields prefixed `_` are scratch variables in both.
  - The `|append`/`|noappend` modifier appears once.
- **Other engine behaviour only in Jackett**: login redirect-domain hints (`:853-871`); "skip duplicated GET requests" and `download.before` custom headers (recent commits).

**Bottom line.** A Go engine that implements the union of the two (schema v11 plus `replaces`, `allowtvsearchimdb`, and lenient unknown-key handling) can load **both repos unchanged**.

### 3.2 Native indexers
There are 65 concrete C# classes in `JC/Indexers/Definitions/`. The full list and diff are in §4.

Family bases (`Definitions/Abstract/`):
- `GazelleTracker` (11 subclasses)
- `AvistazTracker` (5)
- `SpeedAppTracker` (2)
- `PublicBrazilianIndexerBase` (3)
- `CouchPotatoTracker` (**0 subclasses: dead**)

`Feeds/BaseFeedIndexer` and `BaseNewznabIndexer` have **no concrete subclass (dead)**. **Jackett has no generic Torznab, Newznab or RSS indexer, and no Usenet support at all.**

The biggest natives are RuTracker (1,888 LOC), LostFilm (859), DonTorrent (789), AnimeBytes (784), MejorTorrent (673) and AniDUB (640).

### 3.3 Torznab / TorrentPotato / results API
- `Jackett.Server/Controllers/ResultsController.cs`, route `api/v2.0/indexers/{indexerId}/results`:
  - `GET …/results?Query=&Category[]=&Tracker[]=`: the manual-search JSON used by the UI (`:188`).
  - `GET …/results/torznab/api?t=caps|search|tvsearch|movie|music|book|indexers&…` (`:335`).
    - `t=indexers` lists all indexers with caps, for meta indexers only; `&configured=true|false` filters the list.
    - `&cache=false` bypasses the cache.
  - `GET …/results/potato/api`: the TorrentPotato (CouchPotato) API (`:~565`).
- Legacy rewrites (`Startup.cs:147-149`): `/torznab/{id}` → `/api/v2.0/indexers/{id}/results/torznab`, and `/potato/{id}` likewise.
- Download proxy (`DownloadController.cs`): `GET /dl/{indexerId}?jackett_apikey&path=<base64url(DataProtection(url))>&file=`.
  - Magnets become 302 redirects.
  - `.torrent` files are **re-bencoded with sorted keys** (a Sonarr compatibility fix, `:~75`).
- Image proxy: `GET /img/{indexerId}?path=` (posters).
- **Meta indexers** (`JC/Indexers/Meta/`):
  - `all` = `AggregateIndexer` (`MetaIndexers.cs:12-16`).
  - Filter indexers use expressions `type:, tag:, lang:, test:passed|failed, status:healthy|failing|unknown`, combined with `!`, `+` (AND) and `,` (OR) (`JC/Utils/FilterFunc.cs`, README "Filter Indexers").
  - Predefined filters: `public`, `private`, `semi-public`, plus every tag.
  - **IMDb fallback** (`Fallbacks.cs`): an imdbid-only query to a meta indexer is resolved to a title via OMDb (`ImdbResolver.cs`, needs `OmdbApiKey`), and the query is re-issued. `ResultFilters.cs` then filters results by title.
  - The README warns that aggregate results are capped at 1000 and that tracker categories (≥100000) are unusable.

### 3.4 FlareSolverr / proxy
- **FlareSolverr**: global `FlareSolverrUrl` and `FlareSolverrMaxTimeout` (default 55,000 ms), via the `FlareSolverrSharp.ClearanceHandler` HTTP handler (`JC/Utils/Clients/HttpWebClient2.cs:65`, `HttpWebClient.cs:75`).
  - The handler is applied to **every** indexer's client.
  - Definitions only show an `info_flaresolverr` hint (60 definitions).
  - Prowlarr instead uses a **tag-scoped** FlareSolverr proxy.
- **Proxy**: a single global proxy, `ProxyType {Disabled, Http, Socks4, Socks5}` + URL/port/user/pass (`JC/Models/Config/ProxyType.cs`, `JC/Utils/Clients/WebClient.cs:40-90`). There are no per-indexer proxies.
- **HTTP clients**: `HttpWebClient` (legacy) and `HttpWebClient2` (default), selectable with `-c/--UseClient`. Each indexer gets **its own WebClient instance** for a separate cookie jar (`IndexerManagerService.cs:142`).
- **Rate limiting**: only YAML `requestDelay` or a C# per-client delay (`WebClient.cs:128-141`). There is **no global per-host limiter and no query/grab limits** (Prowlarr has both).
- **Retries**: Polly retry, 2 attempts by default, configurable per indexer (`BaseIndexer.cs:432-480`). It honours 429 `Retry-After` (`:395`).

### 3.5 Blackhole
`GET /bh/{indexerId}?jackett_apikey&path&file` (`BlackholeController.cs`) writes the `.torrent` to the global `BlackholeDir`. It is triggered from the manual-search UI, and it is Jackett's only "download client".

### 3.6 Caching
`CacheService` keeps an in-memory dict keyed on indexer. For each query key it caches release lists, with:
- `CacheTtl` default **2100 s** (35 min). The comment at `ServerConfig.cs:21-22` explains the choice as fitting Sonarr's 15-minute RSS interval, Radarr's 60 minutes and so on.
- `CacheMaxResultsPerIndexer` default 1000.

Pruning is **lazy**: expired entries are removed whenever the cache is accessed (`PruneCacheByTtl`), and the oldest queries are evicted once an indexer exceeds its max-results limit (`PruneCacheByMaxResultsPerIndexer`, `CacheService.cs:210-228`). There is no background job. The cache can be viewed and cleared in the UI (`GET api/v2.0/indexers/Cache`). It is lost on restart.

### 3.7 Updater
- `UpdateService` (`JC/Services/UpdateService.cs`) first checks after **1 h**, then every **24 h** (`:65-75`). It polls `https://api.github.com/repos/Jackett/Jackett/releases`, optionally including prereleases.
- It picks the asset for its `Variants.JackettVariant`: FullFrameworkWindows, Mono, CoreWindows, CoreMacOs(+Arm64), CoreLinuxAmdx64/Arm32/Arm64, and the Musl variants.
- It extracts the asset and launches `JackettUpdater[.exe]` (`src/Jackett.Updater`), which kills Jackett, replaces the files and restarts.
- It can be disabled with `UpdateDisabled` or `--NoUpdates`.
- This is the **only** way definitions update.

### 3.8 Tray / service / installers
- `src/Jackett.Tray`: WinForms tray app (`net9.0-windows`).
- `src/Jackett.Service`: Windows service host.
- `Installer.iss`: Inno Setup installer.
- `-i/-u/-s/-k` CLI flags install, uninstall, start and stop the service; `-r` reserves URLs with `netsh` (`ServerService.cs:351-364`).
- Unix/macOS scripts: `install_service_systemd.sh`, `install_service_systemd_mono.sh`, `install_service_macos` (launchd), `Upstart.config`, and `jackett_launcher.sh/.bat`.
- There is a Docker image (linuxserver/jackett) but no Dockerfile in the repo.

---

## 4. Integrations (full lists)

### 4.1 Native indexers (65)
Legend: **[J-only]** = Jackett has it and Prowlarr has no native class; **[dead?]** = service believed dead or niche, verify.

| Class | Base / notes |
|---|---|
| AlphaRatio | Gazelle |
| AniDUB | RU anime |
| Anilibria | RU anime API **[J-only]** |
| AnimeBytes | API |
| AnimeZ | Avistaz. Replaces `animetorrents`. |
| ApacheTorrent | PublicBrazilian **[J-only]** |
| AudioBookBay | Public **[J-only]**. Prowlarr blocklists its YAML and has no class, so Prowlarr **lacks ABB**. |
| AvistaZ | Avistaz |
| BakaBT | |
| BeyondHDAPI | API |
| BitHDTV | Cookie |
| BrasilTracker | **[J-only]** (Prowlarr has it as YAML) |
| BroadcasTheNet | JSON-RPC |
| BrokenStones | Gazelle **[dead?]** (also flagged in prowlarr.md) |
| CinemaZ | Avistaz |
| DesiGaane | Gazelle |
| DICMusic | Gazelle |
| DivxTotal | ES public **[J-only]** |
| DonTorrent | ES public; replaces `todotorrents` **[J-only]** |
| EpubLibre | ES ebooks **[J-only]** |
| EraiRaws | Anime RSS **[J-only]** |
| ExoticaZ | Avistaz; replaces `yourexotic` |
| FileList | API |
| GazelleGamesAPI | Gazelle |
| GreatPosterWall | Gazelle; replaces `seals` |
| HDBitsApi | API |
| HDRTorrent | PublicBrazilian **[J-only]** |
| ImmortalSeed | |
| IPTorrents | Cookie |
| Knaben | Meta-search |
| Libble | |
| LostFilm | RU TV **[J-only]** |
| Magnetico | Self-hosted DHT crawler **[J-only] [dead?]** (upstream project abandoned) |
| MejorTorrent | ES public **[J-only]** |
| MTeamTp | API |
| MyAnonamouse | API |
| NebulanceAPI | API |
| NorBits | |
| Orpheus | Gazelle |
| PassThePopcorn | API |
| PixelHD | |
| PreToMe | |
| PrivateHD | Avistaz |
| Redacted | Gazelle; replaces `passtheheadphones` |
| RedeTorrent | PublicBrazilian **[J-only]** |
| RetroFlix | SpeedApp |
| RevoPeers | **[J-only]** |
| RuTracker | |
| SceneHD | API |
| SecretCinema | Gazelle |
| Shazbat | TV |
| Simurg | Gazelle **[J-only]** |
| SpeedApp | SpeedApp |
| SpeedCD | Replaces `speedcdcookie` |
| SubsPlease | Anime |
| Toloka | UA |
| TorrentBytes | |
| TorrentDay | Cookie |
| TorrentNetwork | DE **[J-only]** |
| TorrentsCSV | Public |
| TorrentSyndikat | |
| TVStore | HU **[J-only]** |
| Uniotaku | BR anime |
| Wolfmax4K | ES **[J-only]** |
| XSpeeds | |

**Prowlarr has, Jackett lacks:**
- Generic **Newznab**, **Torznab**, TorrentRss and TorrentPotato (Jackett is torrent-only and has no generic feeds).
- NzbIndex, Headphones and BinSearch (Usenet).
- Anidex, HDTorrents (Jackett has it as YAML), RevolutionTT.
- Plus Prowlarr's `[Obsolete]` stubs: Rarbg, Xthor, ZonaQ, Animedia, Shizaproject, and the "moved to YML" ones.

**Jackett keeps no `[Obsolete]` stubs.** It deletes dead indexers, keeping only `Replaces` aliases for renames.

### 4.2 Cardigann definitions
- 563 YAML files. See §3.1.1 for the set diff and §3.1.5 for feature usage.
- Languages: en 205, zh 125, ru 50, hu 28, de 26, fr 22, es 13, pt 12, pl 12, …
- 203 definitions carry `legacylinks`, which shows how fast tracker domains churn. One of them, `rudub`, has its domain **rotated daily by a GitHub Action** (`.github/workflows/definitions-update.yml`).
- Liveness can't be checked offline. Upstream's practice is to delete dead definitions quickly, so the shipped set is "live" as of HEAD.
- The 7 Jackett-only files are ones Prowlarr **chose** to block (YGG/UnionGang-style, TorrentGalaxy clones, TheRARBG, etc.; `PIDX/scripts/blocklist.txt`). They are not dead.

### 4.3 Other external services
- GitHub Releases API (updater).
- OMDb (IMDb → title fallback).
- FlareSolverr.
- An HTTP or SOCKS proxy.

There are no notification, download-client or *arr integrations.

---

## 5. External APIs / interfaces

| Surface | Route | Auth |
|---|---|---|
| Torznab | `/api/v2.0/indexers/{id\|all\|<filter>}/results/torznab[/api]` | `apikey=` or `passkey=` query param (`ResultsController.cs:36-50`; skipped when a debugger is attached in DEBUG builds). Torznab XML errors: 100 invalid key, 200/201 indexer unknown/unconfigured, 900. |
| TorrentPotato | `/api/v2.0/indexers/{id}/results/potato[/api]` | apikey |
| Manual search JSON | `/api/v2.0/indexers/{id\|all}/results` | apikey |
| Download / blackhole / image | `/dl/{id}`, `/bh/{id}`, `/img/{id}` | `jackett_apikey` param (dl/bh). The path is DataProtection-encrypted. |
| Admin: indexers | `GET /api/v2.0/indexers[?configured=]`; `GET/POST /api/v2.0/indexers/{id}/Config`; `POST /api/v2.0/indexers/{id}/Test`; `DELETE /api/v2.0/indexers/{id}`; `GET /api/v2.0/indexers/Cache` | **Cookie session only** (`IndexerApiController.cs`, global `AuthorizeFilter` in `Startup.cs:82-86`) |
| Admin: server | `/api/v2.0/server/{AdminPassword,Update,Config (GET/POST),Logs,Shutdown}` | Cookie. `Shutdown` is `[AllowAnonymous]` + `RequiresApiKey` (`ServerConfigurationController.cs:291-294`). |
| UI | `/UI/{Login,Logout,Dashboard,TestCookie}`; `/` → 301 to `/UI/Dashboard` | Cookie `Jackett` (`Startup.cs:51-58`) |
| Health | `GET /health` | anonymous |

**Auth model:**
- There is a single optional admin password. **If it is empty, the UI and admin API are open** (`Jackett.Server/Services/SecurityService.cs:14-15`).
- The password is stored as **SHA-512(password + APIKey)** (`:22-30`). That is weak, and rotating the API key invalidates the password.
- The API key is a 32-character random string.
- There are no users, roles or API-key auth on the admin API. That makes the admin API unusable for automation, unlike Prowlarr's `/api/v1` with `X-Api-Key`.
- Forwarded headers are trusted from RFC1918 and ULA ranges (`Startup.cs:60-71`). CORS allow-all is optional.

---

## 6. Background jobs

There is no scheduler. The only background work is:
1. **Update check**: a dedicated thread, first after 1 h and then every 24 h (`UpdateService.cs:65-75`).
2. **Cache pruning**: not a job. It happens lazily when the cache is accessed (§3.6).
3. Startup-only tasks (`ServerService.Initialize`, `IndexerManagerService.InitIndexers`):
   - environment/Mono/root/Docker checks and notices (`ServerService.cs:76-320`), including a "build older than 3 months" notice (`ServerService.cs:290-300`);
   - renamed-indexer config migration;
   - `RemoveLegacyConfigurations`, which **deletes** `Indexers/*.json` for ids that no longer exist (`IndexerManagerService.cs:~258`).

There is no RSS sync, no periodic indexer testing and no health-check framework. Indexer health comes from the last test and last error, read by the `status:`/`test:` filters.

---

## 7. Configuration surface

- **`ServerConfig.json`** (`JC/Models/Config/ServerConfig.cs:35-66`, defaults at `:13-26`):

  | Group | Keys (defaults in parentheses) |
  |---|---|
  | Server | `Port` (9117), `LocalBindAddress` (127.0.0.1), `AllowExternal` (true on Unix), `AllowCORS`, `BasePathOverride`, `BaseUrlOverride` |
  | Auth | `APIKey`, `AdminPassword`, `InstanceId` |
  | Blackhole | `BlackholeDir` |
  | Updates | `UpdateDisabled`, `UpdatePrerelease` |
  | Cache | `CacheEnabled` (true), `CacheTtl` (2100), `CacheMaxResultsPerIndexer` (1000) |
  | FlareSolverr | `FlareSolverrUrl`, `FlareSolverrMaxTimeout` (55000) |
  | OMDb | `OmdbApiKey`, `OmdbApiUrl` |
  | Proxy | `ProxyType`, `ProxyUrl`, `ProxyPort`, `ProxyUsername`, `ProxyPassword` |
  | Runtime | `RuntimeSettings` |

- **CLI** (`JC/Models/Config/ConsoleOptions.cs`):
  - `-i/-u/-s/-k` install, uninstall, start and stop the Windows service; `-r` reserves URLs.
  - `-l` logs all requests and responses; `-t` enables tracing.
  - `-c` picks the web client; `-x`/`-z` listen publicly or privately; `-p` sets the port.
  - `-n` ignores SSL errors; `-d` sets the data folder.
  - `--NoRestart`, `--PIDFile`, `--NoUpdates`.
- **Other files**: optional `appsettings.json` in the data folder (`Program.cs:98`).
- **Environment variables**:
  - `JACKETT_KEY` (DataProtection purpose)
  - `DisableRootWarning`
  - `XDG_CONFIG_HOME` / `HOME`, which move the data folder
- **Per-indexer settings**: SiteLink, Tags, and the definition's settings or C# ConfigurationData items. Plus, for some natives, configurable retry attempts.
- **Custom definitions**: drop YAML into `~/.config/cardigann/definitions/`.

---

## 8. Platform / infrastructure concerns

- **Dual-targeting net9.0 and net471 (Mono)**:
  - The whole web stack is pinned to ASP.NET Core **2.x** packages (2.3.13, legacy) so that net471 still compiles.
  - This drags in `#if NET471` branches (`Startup.cs`) and a large block of Mono checks: version ≥5.8, mono-devel, ca-certificates-mono, the Mono.Security cert store (`ServerService.cs:145-235`).
  - The Mono build exists for ARMv6 and FreeBSD (README "Linux Installation (ARMv6 or below)"; the FreeBSD/Mono patch at `azure-pipelines.yml:262-272`).
- **Windows-specific**: Tray (WinForms), Windows service, `netsh` URL reservation, Inno Setup installer, winget workflow, `%ProgramData%` data folder, and `Microsoft.NETFramework.ReferenceAssemblies`.
- **CI**: Azure Pipelines builds about 11 runtime variants (win x86/x64, macOS x64/arm64, linux x64/arm/arm64 plus musl, and Mono). GitHub Actions only handles CodeQL, the issue bot, winget and the rudub domain rotation.
- **Scale**: there is no DB, so the design scales to one user and one instance per tracker. State is flat JSON files with `.bak` copies.

---

## 9. Interactions with other *arr apps

- **Setup**: users add each Jackett indexer to Sonarr, Radarr, Lidarr and so on **by hand**, as a "Torznab" indexer:
  - URL `http://host:9117/api/v2.0/indexers/{id}/results/torznab/`
  - API key = Jackett's key
  - categories taken from `t=caps`
  - Radarr even ships a "Jackett" Torznab preset with the `YOURINDEXER` placeholder (`submodules/radarr/src/NzbDrone.Core/Indexers/Torznab/Torznab.cs:50`).
  - Jackett never pushes anything; Prowlarr's app sync is exactly what it lacks.
- **Grabs**: grabs go through Jackett's `/dl/` proxy. The `.torrent` is re-bencoded with sorted keys "to fix torrents … not supported by Sonarr" (`DownloadController.cs`).
- **The "/all" aggregate endpoint is rejected**, in two places in Sonarr, Radarr and Lidarr:
  - An **indexer-validation warning** on save, `JackettAll()` (`submodules/sonarr/src/NzbDrone.Core/Indexers/Torznab/Torznab.cs:126-140`; radarr `:153-166`; lidarr `:113-125`). It fires when `ApiPath` or `BaseUrl` contains `/torznab/all` or `/api/v2.0/indexers/all/results/torznab`. It is `IsWarning = true`, so saving still succeeds. The message is "Jackett's all endpoint is not supported, please add indexers individually".
  - A **health check**, `IndexerJackettAllCheck` (`submodules/sonarr/src/NzbDrone.Core/HealthCheck/Checks/IndexerJackettAllCheck.cs:26-50`, and Radarr's equivalent), wiki anchor `#jackett-all-endpoint-used`.
  - **Why they reject it**: per-indexer caps, categories, seed criteria and backoff all get merged into one fake indexer. One failing tracker disables everything. Results are capped at 1000, and tracker categories are lost (README "Aggregate Indexers").
  - **Gap**: the check matches only the literal id `all`. **Filter endpoints** (e.g. `/indexers/tag:tv/results/torznab`, `/indexers/private/…`) aggregate in exactly the same way but pass the check silently.
- **Caching fit**: Jackett's 35-minute cache TTL is tuned to *arr RSS intervals (`ServerConfig.cs:21-22`). In practice, Sonarr's RSS polls within that window are often served from cache.
- **Other clients**: Jackett still speaks TorrentPotato, for CouchPotato, which is dead.

---

## 10. Outdated / legacy candidates

| Item | Where |
|---|---|
| Mono / .NET Framework 4.7.1 build and all Mono environment checks | `*.csproj` (`net471`), `ServerService.cs:145-260`, `azure-pipelines.yml:108-114, 233-285`, `install_service_systemd_mono.sh` |
| ASP.NET Core 2.x packages (kept only for net471) | `Jackett.Server.csproj` |
| TorrentPotato API (CouchPotato is dead) + legacy `/potato/` route | `ResultsController.cs` `Potato`, `Models/DTO/TorrentPotato*`, `Startup.cs:149` |
| `CouchPotatoTracker` base, 0 subclasses | `JC/Indexers/Definitions/Abstract/CouchPotatoTracker.cs` |
| `BaseFeedIndexer` / `BaseNewznabIndexer`, 0 concrete indexers | `JC/Indexers/Feeds/` |
| Legacy `/torznab/{id}` route | `Startup.cs:148` |
| `GetSonarrConfigFile()` / `sonarr_api.json`, never called | `JC/Services/ConfigurationService.cs:249`, `IConfigurationService.cs:12` |
| `config.json` / `api_key.txt` migration | `ConfigurationService.cs:268-290` |
| `/etc/xdg/cardigan/definitions/` (misspelled path) | `ConfigurationService.cs:221` |
| Two HTTP clients (`HttpWebClient` legacy vs `HttpWebClient2`) | `JC/Utils/Clients/` |
| `multi-select` setting type: implemented, not in schema, 0 users | `CardigannIndexer.cs:159` |
| `.Query.Is*` template variables, 0 users; filters `jsonjoinarray/hexdump/strdump/htmlencode/reltime` and login `oneurl`, 0 users | `CardigannIndexer.cs:1452-1466` |
| SHA-512(pw + apikey) password hash; open admin when no password is set | `SecurityService.cs` |
| Magnetico (abandoned upstream), BrokenStones | natives |
| Windows Tray / service / Inno Setup / netsh | `src/Jackett.Tray`, `src/Jackett.Service`, `Installer.iss` |
| `.Today.Year` returning the previous year in January (quirk, probably intentional for "new year" listings) | `CardigannIndexer.cs:258` |

---

## 11. Rewrite notes

### 11.1 Which is the better reference?

| Concern | Better reference | Why |
|---|---|---|
| **Definition set (data)** | **Jackett** (upstream) or Prowlarr/Indexers v11 (downstream mirror) | Same content. Jackett is fresher, since Prowlarr lags by hours or days and hand-merges. Prowlarr's copy is **versioned** (`v11/`, `VERSIONS`), which is what a separately shipped binary needs. |
| **Format spec** | Either `schema.json`; they are identical except `PageTestBlock.required` | Use Jackett's (stricter). Add `replaces` and `allowtvsearchimdb` handling. |
| **Engine structure to mimic** | **Prowlarr** | It is split into RequestGenerator, Parser and Base (easier to map onto Go packages), has typed search criteria, and plugs into the ThingiProvider settings model, per-host rate limiting, query/grab limits and tag-scoped proxies. Jackett's engine is one 2.5k-line class with global state. |
| **Engine semantics / edge cases** | **Jackett** | It is the reference implementation the definitions are written and tested against. New features appear here first (`allowtvsearchimdb`, the `download.before` headers, duplicate-GET skip). Prowlarr silently ignores keys it doesn't know. |
| **Product model (multi-instance, DB, app sync, history, limits)** | **Prowlarr** | Jackett allows one instance per id, stores flat JSON, and has no limits, history or stats. |
| **Native indexer logic** | Union | Prowlarr lacks ABB, LostFilm, Anilibria, the Spanish and Brazilian publics, EraiRaws, etc. (§4.1). Jackett lacks generic Newznab/Torznab and Usenet. |
| **License cleanliness** | **Neither.** Base the engine on MIT **cardigann/cardigann (Go)** | Jackett is GPL-2.0 (treat as -only). Prowlarr's engine is GPL-3.0 but derived from Jackett's GPL-2.0 code. Prowlarr/Indexers has no license. |

### 11.2 Definitive recommendation
1. **Engine**:
   - Write a **fresh Go Cardigann engine**. Seed it from the MIT Go original (`cardigann/cardigann/indexer/*.go`, about 3.2k LOC: loader, runner, filters, selectors, real `text/template`).
   - Extend it to the **v11 schema** as the spec. Validate it behaviourally against the shipped definitions and recorded HTTP fixtures, **not** by translating either C# file.
   - Implement the **union** of both engines:
     - Prowlarr v11 behaviour;
     - Jackett's `replaces` (id aliasing) and `allowtvsearchimdb`;
     - lenient unknown keys, with a warning;
     - `.Today.Year` and checkbox semantics matching Jackett, since the definitions are authored against Jackett.
   - Note that Go `text/template` is *stricter* than the regex imitation: `{{ if .X }}` on a missing key, `.True` as a string, `and`/`or` returning values. Either:
     - write a compatible mini-interpreter; or
     - pre-process templates into Go-template-safe form and test all 563 definitions.
   - Filters use **.NET regex** (lookarounds, etc.; see comparison.md §3). Use a .NET-compatible regex engine such as `dlclark/regexp2` for `regexp`, `re_replace`, `andmatch` and `validate`.
2. **Definitions: consume both repos, vendor neither.**
   - The formats are compatible, so one loader can read Jackett's flat `src/Jackett.Common/Definitions/*.yml` **and** Prowlarr's `definitions/v11/*.yml`.
   - Pick **one primary feed** to avoid duplicate ids. Recommendation: **Prowlarr/Indexers v11**, because it is versioned for out-of-band delivery, pre-filtered by a blocklist, and already served as `package.zip` from `indexers.prowlarr.com`. Allow an optional Jackett feed or overlay for fresher fixes and the 7 extra ids.
   - Resolve by `id` with the precedence **user custom > primary > secondary**, the same as Jackett's directory order.
   - **Do not embed the YAML in the binary or repo.** Download it at runtime into the data dir, as Prowlarr does. This keeps the GPL-2.0-derived data out of the GPL-3.0 program: arguably "mere aggregation" rather than combination. Confirm with counsel.
   - Add your own schema-version gate, as Prowlarr's `DEFINITION_VERSION` does, so an engine that is too old refuses newer-format files.
3. **Native indexers**:
   - Port the union behind one Go interface. Prioritise the private trackers used by TV and movie users (BTN, PTP, HDBits, BeyondHD, IPT, TorrentDay, FileList, MAM, AnimeBytes, the Avistaz family, Gazelle).
   - The natives are code, so they must be **reimplemented from API documentation and behaviour**, not translated from Jackett (GPL-2.0) C#.
4. **What not to carry over from Jackett**:
   - the `all` and filter aggregate endpoints (the monolith searches in-process);
   - TorrentPotato;
   - blackhole;
   - the Mono/net471 and Windows tray/service code;
   - the global-only proxy and FlareSolverr (use tag-scoped, as in Prowlarr);
   - the flat-JSON single-instance config;
   - the SHA-512 admin password.
5. **Worth keeping from Jackett**:
   - `replaces` id aliasing;
   - persisted per-indexer `LastError` plus test status as a cheap health signal;
   - the result cache keyed by query with TTL ≈ RSS interval;
   - sorted-key re-bencoding on grab;
   - `legacylinks` auto-migration (Prowlarr has it too);
   - OMDb-style id → title fallback, which the monolith can do with its own metadata instead.
