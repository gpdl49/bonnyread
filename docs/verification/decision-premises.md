# Verification: factual premises behind D1, D4, D6, D7, D8, D9, D10, D11

Verified 2026-09-27 against `docs/decisions.md`. D3 (metadata) and D5 (licensing) are out of scope and covered elsewhere.
Unless noted, paths are relative to `submodules/`. Submodule commits: sonarr a8a82905e, radarr c90668a52, lidarr da7b4dfb1, bookshelf 3ac41f7e3, whisparr c55fdc6, whisparr-eros a99b7b5, prowlarr 12c327808, jackett 6556904.

Verdicts: **CONFIRMED**, **CORRECTED** (the premise is wrong or incomplete), **UNVERIFIABLE**.

## Summary

| Decision | Confirmed | Corrected | Unverifiable | Recommendation changes? |
|---|---|---|---|---|
| D1 Regex engine | 6 | 3 | 0 | No. The measurements make option C impossible and support B |
| D4 Multi-file | 4 | 1 | 0 | No |
| D6 Frontend | 1 | 1 (omission) | 0 | No. It leans a little further towards React |
| D7 External API | 11 | 2 | 1 | No. The context table needs edits |
| D8 rls | 4 | 0 | 0 | No. The rls license question is closed (MIT) |
| D9 Profile data | 6 | 1 | 0 | No. The PCD importer must replay SQL ops |
| D10 Whisparr | 12 | 3 | 1 | No. Restore item 1 needs rewording |
| D11 DDL | 4 | 2 | 0 | No. Kapowarr already does torrents |

No correction reverses a recommendation. These affect how a recommendation is carried out:
- **D1:**
  - regexp2 can't parse .NET named Unicode blocks (`\p{IsCyrillic}`), so a translation shim is needed.
  - Most Sonarr parser patterns can't be rewritten in RE2 cheaply, so under option B most of the parser still runs on regexp2.
- **D9:** The PCD importer has to replay SQL ops. Profilarr's portable JSON form is feature-flagged off.
- **D7:** Tautulli and Kavita are not consumers. Readarr/Bookshelf v1 has more consumers than the doc lists.

---

## D1. Regex engine

**Method.** I wrote a Go 1.27.1 harness (scratchpad `d1/h/main.go`). For every pattern it:
- compiles the pattern with Go `regexp` (RE2) and with `github.com/dlclark/regexp2` v1.12.0 (default .NET mode, IgnoreCase);
- runs a lexical scan for lookahead, lookbehind, backreferences, atomic groups, conditionals, possessive quantifiers, `\G`/`\Z`, class subtraction and inline `x`/`n` options;
- for data-file patterns that compile in both engines, checks `IsMatch` on 2,647 release titles taken from the Sonarr and Radarr `ParserTests` `[TestCase]` lines and compares the two engines.

Pattern sources:
- **TRaSH-Guides** @ `1de1361` (2026-09-27): `docs/json/{radarr,sonarr}/cf/*.json`, the `fields.value` of `ReleaseTitleSpecification` and `ReleaseGroupSpecification`.
- **Dictionarry-Hub/database** branch `v2` @ `6266304` (2026-09-27): I replayed the `insert`, `update ... set "pattern"`, rename and delete statements in all 357 `ops/*.sql` files to get the final `regular_expressions` table.
- **Prowlarr/Indexers** `definitions/v11` @ `223d3b1` (2026-09-26), and **Jackett** `src/Jackett.Common/Definitions`: the args of `regexp` and `re_replace` filters, plus `{{ re_replace ... }}` in templates.
- **Arr parsers:** string literals passed to `new Regex(`, `new(` and `new RegexReplace(` in `src/NzbDrone.Core/Parser/*.cs`.

### Results

| Source | Unique patterns | RE2 compile fails | Constructs RE2 lacks | regexp2 fails |
|---|---|---|---|---|
| TRaSH CFs | 1,342 (2,793 refs) | **71 (5.3%)** | lookahead 47, lookbehind 40, backref 1 | 0 |
| Dictionarry v2 | 615 | **534 (87%)** | lookbehind 524 (mostly the `(?<=^\|[\s.-])GROUP\b` idiom), lookahead 22 | 0 |
| Cardigann (Prowlarr v11) | 710 (1,785 refs) | **25 (3.5%)** | lookahead 16, lookbehind 7, atomic 1, conditional 1, backref 1. The rest are `\uXXXX`, `\ ` and `\p{IsBlock}` | **2** |
| Cardigann (Jackett) | 716 (1,825 refs) | 23 (3.2%) | same shape | **2** |
| Sonarr parser | 192 | **132 (69%)** | lookahead 123, lookbehind 93, backref 3 (`\k<sep>`), atomic 1 | 0 |
| Radarr parser | 77 | 28 (36%) | lookahead 25, lookbehind 20, backref 1 | 0* |
| Lidarr parser | 67 | 10 (15%) | lookahead 7, lookbehind 8 | 0* |
| Bookshelf parser | 61 | 7 (11%) | lookahead 5, lookbehind 6 | 0 |
| Whisparr v2 parser | 72 | 17 (24%) | lookahead 14, lookbehind 10, backref 1 | 0 |
| Whisparr v3 parser | 100 | 31 (31%) | lookahead 27, lookbehind 18, backref 1 | 0* |

\* The regexp2 "failures" in the parser rows were my extractor's fault: it cut off patterns that are built by string concatenation. They aren't engine gaps.

Semantics: 1,271 TRaSH and 81 Dictionarry patterns compile in both engines, and **none** gave a different match result on the 2,647-title corpus.

Caveat: that corpus is mostly ASCII. RE2's `\d`, `\w` and `\b` are ASCII-only, while .NET and regexp2 treat them as Unicode. Unicode titles can still differ.

### Premises

| Premise | Verdict | Evidence |
|---|---|---|
| Custom-format datasets (TRaSH, Dictionarry) use .NET syntax that RE2 can't handle | **CONFIRMED** | See the table. Dictionarry is 87% non-RE2, TRaSH 5.3%. Example: `radarr/cf/20-stereo.json` has `(?<=\b[12]\d{3}\b.*?)(?<!repac)…` |
| Cardigann filters `regexp`, `re_replace`, `andmatch` and `validate` are regexes | **CORRECTED** | Only `regexp` and `re_replace` (and the `{{ re_replace }}` template) are regexes: `prowlarr/src/NzbDrone.Core/Indexers/Definitions/Cardigann/CardigannBase.cs:371-383,638-650`. `validate` splits on delimiters and intersects the tokens (`:756-762`), with no regex. `andmatch` is a no-op in the row parser that defers to keyword matching (`CardigannParser.cs:709-711`). The Cardigann set is also **~96.5% RE2-compatible**, which fits its origin in the Go Cardigann. |
| Sonarr, Radarr, Lidarr, Bookshelf and Whisparr parsers use .NET syntax | **CONFIRMED** | See the table. Sonarr is heaviest: 132 of 192 patterns need lookarounds or backrefs, for example `\k<sep>` in the daily-episode patterns (`sonarr/src/NzbDrone.Core/Parser/Parser.cs`). |
| Whisparr product-code patterns | **CONFIRMED** | `whisparr/src/NzbDrone.Core/Parser/Parser.cs:21`, `ExternalIdBasePattern = [A-Z]{2,10}[-_](?=[A-Z0-9]*\d)[A-Z0-9]{2,10}`, uses a lookahead, and the ten `ExternalIdPattern`s in `:46-99` are built from it. The lookahead is easy to rewrite for RE2. |
| RE2 "doesn't accept .NET-style named groups" | **CORRECTED** | Go has accepted `(?<name>…)` since 1.22. I tested this on 1.27.1, and duplicate group names compile too. Named groups aren't a blocker. The real blockers are lookarounds, backrefs, `\uXXXX` (RE2 needs `\x{XXXX}`) and `\p{IsBlock}`. |
| Bazarr's guessit/rebulk use Python regex | **CONFIRMED** | rebulk uses the stdlib `re` by default and the `regex` module only when `REBULK_REGEX_ENABLED` is set (`bazarr/libs/rebulk/remodule.py:12-16`). guessit has 12 `(?=` and 1 `(?!`. |
| regexp2 is .NET-compatible enough for the data | **CORRECTED (small gap)** | regexp2 compiled every TRaSH and Dictionarry pattern. It **rejects .NET named Unicode blocks**: `\p{IsCyrillic}` (`0daykiev.yml`) and `\p{IsCJKUnifiedIdeographs}` (`thepiratebay.yml`), in both definition sets. A pre-compile rewrite to code-point ranges is needed. |
| regexp2 supports match timeouts | **CONFIRMED** | The `Regexp.MatchTimeout` field. In the demo, `^(a+)+$` on 40 a's plus `!` returned `match timeout after 100ms` after about 130 ms, so the timeout is coarse-grained. |
| Profilarr already uses a 100 ms timeout | **CONFIRMED** | `profilarr/src/services/parser/Endpoints/MatchEndpoints.cs:41` has `TimeSpan.FromMilliseconds(100) // Timeout to prevent ReDoS`. Whisparr v3 also sets a timeout on every regex (`whisparr-eros/src/NzbDrone.Common/RegexDefaults.cs`). |

**Effect on D1: none.** Option C is ruled out: it would break 87% of Dictionarry and 5% of TRaSH. Option B stands, with two refinements:
1. Add a pattern-normalisation shim in front of regexp2 for `\p{IsBlock}`, with a test that compiles every pattern in the dataset.
2. The "rewrite our parser in RE2 where possible" part of B covers roughly a third of Sonarr's patterns. The rest (lookbehind-anchored numbering and `\k<sep>` dates) would stay on regexp2 unless D8 replaces them with `rls`, which is pure RE2.

---

## D4. Multiple files per item

| Premise | Verdict | Evidence |
|---|---|---|
| Radarr has one file per movie | **CONFIRMED** | `radarr/src/NzbDrone.Core/Movies/Movie.cs:36` has `int MovieFileId`, and `:38` has `HasFile => MovieFileId > 0`. Whisparr v3 is the same (`whisparr-eros/.../Movies/Movie.cs:36`). |
| Sonarr has one file per episode | **CONFIRMED** | `sonarr/src/NzbDrone.Core/Tv/Episode.cs:20` has `int EpisodeFileId`. A file can cover several episodes (`MediaFiles/EpisodeFile.cs:28`), but an episode can't have several files. The same holds for Whisparr v2 (`Tv/Episode.cs:22`) and for Lidarr, one file per track (`Music/Model/Track.cs:32`). |
| Seerr duplicates everything for 4K (`is4k`) | **CONFIRMED** | `seerr/server/entity/Media.ts:140-233` has 11 `*4k` fields (`status4k`, `serviceId4k`, `externalServiceId4k`, `ratingKey4k`, `downloadStatus4k`, …). There is also `Season.ts:25`, `MediaRequest.ts:56,222,250` and server `is4k` in `server/lib/settings/index.ts:80,726-729`. |
| The Bookshelf ebook/audiobook split is its "top TODO" | **CORRECTED** | The limitation is real. `bookshelf/README.md:9-11` says "only one type of a given book is supported… you will need multiple instances". The TODO list is different: the related item "Support ebook and audio files in the same root" (`README.md:39`) is second, and it is about a shared root folder, not keeping both formats of one book. |

**Effect:** none. Cite `README.md:9-11`.

---

## D6. Frontend stack

| App | Stack (from package.json and related files) |
|---|---|
| Sonarr | React 18.3.1, zustand, react-query, Vite, TS (893 TS files vs 17 JS) |
| Radarr | React 18.3.1, Redux plus react-query, webpack (1,015 TS vs 426 JS) |
| Prowlarr | React 17, Redux, webpack (400 TS vs 474 JS) |
| Lidarr | React 18.3.1, Redux; 82% JS by lines |
| Bookshelf | React 17, Redux; 92% JS by lines |
| Whisparr v2 | React 18.3.1, Redux plus react-query |
| Whisparr v3 | React 19.3.0, zustand plus react-query, webpack, all TS |
| Seerr | Next 16.2.6, React 19.2.6, Tailwind 3 |
| Bazarr | React 19.3, Mantine 9.6, Vite |
| autobrr | React 19.2, TS, Vite, react-query, Tailwind 4 |
| Profilarr | SvelteKit 2.70, Svelte 5.57, on Deno (`profilarr/deno.jsonc:36,42`) |
| Unpackerr | Svelte 5.57 with Vite |
| Cleanuparr | Angular 22.0.4 |
| Jackett | jQuery plus DataTables (`jackett/src/Jackett.Common/Content/index.html:17-18`) |
| Kapowarr | Vanilla JS |
| Recyclarr | No UI |

- **CONFIRMED:** the stacks are as described. "Lidarr and Bookshelf are 80%+ JS" holds by lines of code, not by file count (60% and 70%).
- **CORRECTED (omission):** D6 leaves out autobrr and Whisparr v3, both React 19. With them, React is used by 10 of the 15 UIs.
- **Effect:** none. This adds weight to option A.

---

## D7. External API consumers

Each consumer's default branch was cloned and searched on 2026-09-27.

| Consumer | Claimed | Verdict | Evidence |
|---|---|---|---|
| Tautulli | Sonarr/Radarr v3 | **CORRECTED** | There is no Sonarr, Radarr or Overseerr client anywhere in Tautulli/Tautulli outside the vendored `lib/`. Tautulli is something other tools call, for example Maintainerr. |
| Maintainerr | Sonarr/Radarr v3 and Seerr v1 | CONFIRMED | `apps/server/src/modules/api/servarr-api/servarr.service.ts:28,48,64,84` uses `/api/v3/`. `seerr-api/seerr-api.service.ts:233` uses `/api/v1`, and `helpers/seerr-api.helper.ts:13` sends `X-Api-Key` only. |
| Homepage | Sonarr/Radarr v3 and Seerr v1 | CONFIRMED, and also Lidarr/Readarr v1 | `src/widgets/sonarr/widget.js:5` and `radarr/widget.js:5` use `/api/v3`. `lidarr/widget.js:4` and `readarr/widget.js:5` use `/api/v1`. `overseerr/widget.js:4` uses `/api/v1`, with `X-API-Key` set in `src/utils/proxy/handlers/credentialed.js:139`. |
| nzb360 | Sonarr/Radarr v3 | UNVERIFIABLE | It is closed source. |
| LunaSea | Sonarr/Radarr v3 | CONFIRMED, but **archived** | `lib/api/sonarr/sonarr.dart:35` and `radarr/radarr.dart:64` use `api/v3/`. Lidarr is `api/v1`. The repo is archived; its last push was 2025-04-03. |
| Notifiarr | Sonarr/Radarr v3 | CONFIRMED, through `golift.io/starr` | `go.mod:52` requires `golift.io/starr v1.4.1`. `pkg/checkapp/starr.go:16,34,43,52` covers Lidarr, Radarr, Readarr and Sonarr. |
| nzbToMedia | `Downloaded*Scan` over v3 | CONFIRMED, and also v1 | `core/auto_process/tv.py:320,326` sends `DownloadedEpisodesScan` to `/api/v3/command`. `movies.py:73,218` sends `DownloadedMoviesScan`. `music.py:108` calls Lidarr `/api/v1/command`. `books.py:40,51` sends `DownloadedBooksScan` to `/api/v1`. |
| Kavita | Lidarr/Bookshelf v1 | **CORRECTED** | Kareadita/Kavita never mentions lidarr, readarr or bookshelf. The call goes the other way: Bookshelf notifies Kavita (`bookshelf/src/NzbDrone.Core/Notifications/Kavita/KavitaServiceProxy.cs`). |
| Requestrr | Seerr v1 with `X-API-User` | CONFIRMED, and also v3 and Lidarr | `RequestrrBot/DownloadClients/Overseerr/OverseerrClient.cs:1095,1111` sends `X-API-User`. `Sonarr/SonarrClientV4.cs:629` uses `/api/v3`. |
| Doplarr | Seerr v1 | CONFIRMED, but **archived** | `backends/overseerr/impl.clj:11` uses `/api/v1`. `overseerr.clj:64` sends `X-API-User`. `sonarr/impl.clj:8` and `radarr/impl.clj:6` use `/api/v3`. |
| Home Assistant | Seerr v1 | CONFIRMED, and also Sonarr/Radarr v3 and Lidarr v1 | The `components/{sonarr,radarr,lidarr}` integrations use `aiopyarr==23.4.0`, which defaults to v3 and v1. `components/overseerr` uses `python-overseerr==0.9.0`, which calls `api/v1` with `X-Api-Key`. |
| Mylar3 (Newznab client) | Newznab/Torznab | CONFIRMED | `mylar/search.py:1038,1048` sends `t=search`. `prowlarr/.../Applications/Mylar` exists. |
| LazyLibrarian (Newznab client) | Newznab/Torznab | CONFIRMED | `lazylibrarian/providers.py:1150` and `__init__.py:820-823` handle torznab. |
| Seerr's `X-API-User` is real | Real | CONFIRMED | `seerr/server/middleware/auth.ts:13-20` |

**Effect:** none. Option B stands. Changes to the context table:
- Drop Tautulli and Kavita. Kavita is a notification target the monolith must implement, not a consumer.
- Add Readarr/Bookshelf v1 consumers: Homepage, Notifiarr, nzbToMedia and aiopyarr. The v1 `command` endpoint with `Downloaded*Scan` is worth doing early.
- Only Requestrr and Doplarr send `X-API-User`.
- LunaSea and Doplarr are archived.
- The note "most of the consumer list comes from agents' knowledge" can now point to this evidence.

---

## D8. Release parser foundation

| Premise | Verdict | Evidence |
|---|---|---|
| autobrr uses an `rls` fork | **CONFIRMED** | `autobrr/go.mod:41` requires `github.com/moistari/rls v0.6.0`, and `:7` replaces it with `github.com/autobrr/rls v0.8.1`. |
| `rls` covers all media kinds in one lexer | **CONFIRMED** | The `Type` enum in `rls.go:701-712` has App, Audiobook, Book, Comic, Education, Episode, Game, Magazine, Movie, Music and Series. It uses only stdlib `regexp` (RE2). |
| `rls` is tested | **CONFIRMED, but thin** | `tests.yaml` has 418 cases: movie 160, episode 91, music 68, series 25, game 22, app 13, book 6, education 6, audiobook 4, comic 4, magazine 2. `go test ./...` passes on v0.8.1. |
| `rls` license needs checking | **Resolved: MIT** | Upstream and the fork are both MIT. |
| Sonarr has 1,631 cases | **CONFIRMED** | There are 1,631 `[TestCase(` lines in `sonarr/src/NzbDrone.Core.Test/ParserTests`. |

**Effect:** none. The license item in D5 and D8 can be closed. Books and comics have very few cases, which supports running the spike first.

---

## D9. Community profile data

| Premise | Verdict | Evidence (TRaSH-Guides @ 1de1361; Dictionarry v2 @ 6266304; both 2026-09-27) |
|---|---|---|
| TRaSH has stable 32-hex `trash_id`s | **CONFIRMED** | All 242 Radarr and 236 Sonarr CF files match `^[0-9a-f]{32}$`. |
| TRaSH has named score sets | **CONFIRMED** | `trash_scores` has 18 named sets for Radarr and 9 for Sonarr, e.g. `default`, `sqp-1-1080p`, `german`. Quality profiles choose one with `trash_score_set`. |
| TRaSH has CF groups | **CONFIRMED** | There are 40 Radarr and 38 Sonarr `cf-groups/*.json` files, each with a `trash_id` and `{name, trash_id, required}` members. `quality-profile-groups/`, `quality-size/`, `naming/` and `conflicts.json` also exist. |
| TRaSH stores raw per-app enum ids | **CONFIRMED (for CF specifications)** | Integer `value`s appear in the Source, Resolution, Language, QualityModifier, IndexerFlag and ReleaseType specs, e.g. `radarr/cf/remux-tier-01.json` QualityModifier `5`, and `sonarr/cf/anime-dual-audio.json:31` `8`. Quality profiles use names, not ids. |
| `trash-pcd` exists | **CONFIRMED** | github.com/Dictionarry-Hub/trash-pcd, created 2026-01-27. |
| Dictionarry PCDs are SQL op files keyed by name | **CONFIRMED** | `pcd.json` depends on Dictionarry-Hub/schema 1.1.0 and there are 357 `ops/*.sql` files. The ops key rows by name, e.g. `update "regular_expressions" set "pattern" = … where "name" = 'SDR' and "pattern" = …` (`ops/200.tweak-hdr-regex.sql`). Override, align and ask are real: `profilarr/src/lib/server/pcd/core/validate.ts:10,18`. |
| The PCD importer can go "via Profilarr's portable JSON form" | **CORRECTED** | `profilarr/src/lib/shared/pcd/portable.ts` exists, but `src/lib/shared/features.ts:8` sets `importExport: false`, and every dataset ships SQL ops. An importer must replay the ops against the schema, for example in a scratch SQLite database, and read the entities back. |

**Effect:** none. The PCD importer design changes, and the extra coupling to Profilarr's schema weakens option B further.

---

## D10. Whisparr v2 vs v3 table

| Row / claim | Verdict | Evidence |
|---|---|---|
| v3 has an `ItemType` flag on the movie table | **CONFIRMED** | `whisparr-eros/src/NzbDrone.Core/Datastore/Migration/001_initial_setup.cs:345`. It is indexed, with `StudioForeignId` at `:346`. |
| Studio and Performer are first-class entities, monitorable, with `AfterDate` | **CONFIRMED** | `Movies/Studios/Studio.cs:31-35` and `Movies/Performers/Performer.cs:45-49`. Migrations: `025_afterDate.cs:11` and `243_performer_after_date.cs:25`. |
| v3 has StashDB, ThePornDB, TMDb and IMDb ids, and keys scenes by StashDB id | **CONFIRMED** | `Movies/MovieMetadata.cs:29,46-49`. `MetadataSource/SkyHook/SkyHookProxy.cs:37,108-111` uses the `stashid:` prefix. |
| v2 stores ThePornDB ids in `TvdbId` | **CONFIRMED** | `whisparr/src/NzbDrone.Core/Tv/Series.cs:21`, filled from `ForeignId` at `SkyHookProxy.cs:167`. |
| v2 uses the `api.whisparr.com/v3` proxy and v3 uses `/v4` | **CONFIRMED** | `NzbDrone.Common/Cloud/WhisparrCloudRequestBuilder.cs:19` in v2 and `:25` in v3. |
| v3 "refreshes only what changed" | UNVERIFIABLE | Not checked in code. |
| v2 rejects ambiguous matches | **CONFIRMED** | `whisparr/.../Tv/EpisodeRepository.cs:264-271` ("Returning null to prevent incorrect match"), plus `EpisodeService.cs:244-249` and `EpisodeRepository.cs:69-71`. |
| v3 fuzzy match takes a tied top score | **CONFIRMED** | `whisparr-eros/.../Movies/MovieService.cs:1294-1299` does `OrderByDescending(m => m.Score).First()` with no tie check. |
| v2 indexes product codes | **CONFIRMED** (not unique) | `Migration/221_add_episode_external_id_index.cs:12` creates a plain index. `whisparr-eros.md` calling it "unique" is wrong. |
| v3 no longer indexes product codes or uses them to match downloads | **CORRECTED** | It is true there is no index (`009_studio_alases.cs:11`, a nullable `Code`) and no code-based fallback for completed downloads, which v2 has at `CompletedDownloadService.cs:102-122,363-365`. But v3's title matching still tries the code (`MovieService.cs:708-710,804,984-986`). |
| Search settings are per indexer in v2 and global in v3 | **CONFIRMED** | v2: `Indexers/Newznab/NewznabSettings.cs:94-103`. v3: `Configuration/ConfigService.cs:135-173`. |
| Adult categories are the default | **CONFIRMED, but narrower** | Newznab defaults to the 6000 range in both versions (v2 `NewznabSettings.cs:72`, v3 `:59`). Only FileList regressed: v3 `FileListSettings.cs:35-40` defaults to movie categories, where v2 `:40-42` defaults to XXX. |
| v3 frontend "follows Sonarr (React 19, no Redux)" | **CORRECTED (minor)** | v3 is React 19.3.0 with zustand and react-query and no Redux (`whisparr-eros/package.json:55,65`). Sonarr is React **18**.3.1 on Vite, and v3 uses webpack. |
| v2 is 54 commits behind, v3 is 0 behind | **CONFIRMED** | `git -C sonarr rev-list --count 900cc21d3..HEAD` returns 54. `whisparr-eros/UPSTREAM_SYNC.md:2,15` has sync point `c90668a520`, which equals the radarr submodule HEAD. |
| There is no migration from v2 | **CONFIRMED** | The schema starts fresh at `001_initial_setup`. The only Whisparr import list reads `/api/v3/movie` (`ImportLists/Whisparr/WhisparrV3Proxy.cs:34`). |
| "14 stable releases, roughly weekly, one maintainer" | **CORRECTED** | `gh release list` shows 21 stable releases from 2026-01-15 to 2026-09-24. They are not weekly: there is a gap from 2026-03-26 to 2026-06-19. It does have one main maintainer: 380 commits since 2026-06-01, with nobody else above 6. |

**Effect:** none. Reword restore item 1 as "index product codes and add a code-based fallback for completed downloads and import". Item 4 is a small fix (FileList).

---

## D11. Direct-download acquisition

| Premise | Verdict | Evidence |
|---|---|---|
| Kapowarr scrapes GetComics, resolves links at grab time and falls back | **CONFIRMED** | `kapowarr/backend/implementations/indexer_clients/ddl/GetComics.py`. `download_preppers/ddl/GetComics.py:461,518,596,683` handles link testing and fallback. |
| The hosts are MEGA, MediaFire, Pixeldrain and WeTransfer | **CORRECTED (incomplete)** | `backend/base/definitions.py:587-597` lists Mega, MediaFire, WeTransfer, Pixeldrain, **GetComics' own servers** and **GetComics torrent (magnet) links**. The clients are in `implementations/download_clients/`: `DDL.py`, `Mega.py`, `MediaFire.py`, `PixelDrain.py`, `WeTransfer.py` and `Torrent.py`. |
| No Usenet or Newznab | **CONFIRMED** | Apart from a "future proofing" comment (`definitions.py:584-586`), there is no Usenet, Newznab or Torznab code. |
| Torrent is not supported (implied by the "third protocol" framing) | **CORRECTED** | Kapowarr sends GetComics magnet links to qBittorrent or Transmission (`implementations/external_clients/torrent/`, `features/post_processing.py:316-341`). It has no torrent indexer. |
| CloudFlare solving and host quotas | **CONFIRMED** | FlareSolverr: `implementations/flaresolverr.py:94-144`. Mega rate limits: `Mega.py:721-726,847-849`. |
| Mylar3 supports Usenet and torrents | **CONFIRMED** | The mylar3/mylar3 README (lines 8 and 30-31) mentions NZB, torrents and SAB/NZBGet. It also mentions **direct download**, so Kapowarr isn't the only DDL comic manager. |

**Effect:** none. "A now, B later" stands. If B is built, it should cover GetComics' own servers too.
