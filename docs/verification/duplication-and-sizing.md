# Verification: sizes, duplication and per-kind claims

Scope: `docs/analysis/comparison.md` §0 (sizes, stacks, versions), §2 (duplication), §2.4 (per-kind differences), §7 (sizing), plus the related `decisions.md` D4 premise.
Measured on 2026-09-27 against the submodule HEADs checked out in `submodules/` (e.g. Radarr `c90668a52`, Sonarr `a8a82905e`, Whisparr `c55fdc6`, Whisparr-Eros `a99b7b5`). The Prowlarr/Indexers repo was read through `gh api` at `223d3b12` (2026-09-26).

Verdicts:
- **CONFIRMED:** within about 10%, or exact.
- **CORRECTED:** wrong, or more than 25% off.
- **NUANCE:** the number is right but the wording misleads.
- **UNVERIFIABLE:** cannot be measured from here.

## Results

### §0 Sizes and stacks

Method: non-test, git-tracked lines, blanks included (the script is below).

| Claim | Measured | Verdict |
|---|---|---|
| Sonarr ~155k C# (Core 112k, API v3+v5 21k), .NET 10, React 18/TS, Vite, react-query, zustand | 151k non-test (234k with tests); Core 112k (107k non-test); API 21.3k. SDK 10.0.401, net10.0. React 18.3, Vite 8, react-query 5, zustand 5, 85k TS vs 1k JS | CONFIRMED |
| Radarr ~133k (Core 101k), .NET 8, React 18, Redux→TS mid-migration | 130k; Core 101k. SDK 8.0.421. React 18.3 + redux 4 + react-query, webpack; 44k JS and 57k TS | CONFIRMED |
| Lidarr ~120k (Core 95k), .NET 8, ~82% JS, Redux | 124k; Core 95k. net8.0. 85k JS vs 18.6k TS = 82% JS | CONFIRMED |
| Bookshelf .NET 6, React 17/Redux, Core ~91k | net6.0 (no global.json); React 17.0.2, redux; Core 91k (118k total) | CONFIRMED |
| Bookshelf: 32 commits since Readarr was archived | 35 commits after Readarr `develop` tip `0b79d30` (2025-06-27) | CONFIRMED (≈) |
| Whisparr v2 .NET 10, React 18 + Redux, Core ~91k | net10.0; React 18.3, redux 4 plus react-query; 72.7k TS vs 17.6k JS; Core 91k (120k total) | CONFIRMED |
| Whisparr v2 "cherry-picks upstream daily" | The submodule is a **shallow clone** (1 commit). GitHub shows batches of Sonarr ports (for example 12 on 2026-09-19) | UNVERIFIABLE locally; "daily" is loose |
| Whisparr v3 (Eros): ".NET", React 19 + react-query/zustand, no Redux | net10.0 (SDK 10.0.101); React 19.3, react-query 5, zustand 5, react-router 7, **webpack** (not Vite); 108k TS vs 0.4k JS | CONFIRMED (version is .NET 10) |
| Kapowarr ~28k Python, Flask, vanilla JS | 28.3k py; flask 3 + waitress; 5.2k JS | CONFIRMED |
| Prowlarr ~108k (Core 82k), .NET 8, React 17 "(half JS)" | 108k; Core 82k; net8.0; React 17. **40.2k JS vs 14.1k TS = 74% JS** | NUANCE: mostly JS, not half |
| Jackett ~45k C# + 563 YAML; .NET 9 **and** net471; jQuery | 42.7k non-test (46.8k all); 563 YAML; `net9.0;net471` TFMs; jQuery, Bootstrap and DataTables | CONFIRMED |
| autobrr Go 1.27, ~57k Go + 31k TS | `go 1.27.0`; 57.6k Go non-test (92k with tests); 31k TS | CONFIRMED |
| Profilarr ~81k TS + 48k Svelte, optional .NET parser | 83.7k TS non-test; 47.7k Svelte; 2.7k C# parser | CONFIRMED |
| Recyclarr ~20k C#, .NET 10 | 20k non-test; net10.0 | CONFIRMED |
| Seerr ~43k server + 52k UI, Express 5 + Next 16 / React 19, TypeORM | Server 39k non-test (48k all); UI 52k; express 5.2, next 16.2, react 19.2, typeorm 0.3 | CONFIRMED |
| Bazarr ~25k own + 22k providers (+1.3M vendored), React 19/Mantine | `bazarr/` 24.7k; `subliminal_patch/providers` 22.3k (of `custom_libs` 43k); `libs/` 1.31M; React 19.3, Mantine 9 | CONFIRMED |
| Cleanuparr ~48k C#, .NET 10 + Angular | 47.5k non-test **excluding 137k lines of generated EF migrations** (181k including them); net10.0; Angular 22 | CONFIRMED (only when EF migrations are excluded) |
| Unpackerr Go, ~20k, Svelte 5 | 19.8k non-test (37.7k with tests); go 1.27.0; svelte 5.57 | CONFIRMED |
| "About **900k** lines of first-party backend" | **~1.27M** non-test (the seven NzbDrone forks alone are 890k). The table's own rows add up to ~1.2M | **CORRECTED** |
| "About **450k** lines of frontend" | **~860k** (JS/TS/Svelte only, no CSS, tests excluded). The seven fork frontends alone are 640k | **CORRECTED** |

### §2.1 Infrastructure duplication

| Claim | Measured | Verdict |
|---|---|---|
| NzbDrone skeleton in seven copies | 7 repos with `NzbDrone.Common/Core/Host/Update/SignalR/Windows/Mono` | CONFIRMED |
| Subsystem table: `NzbDrone.Common`, ThingiProvider…, React scaffolding, migration histories each in "6" copies | All seven forks, including Prowlarr, carry each of them | **CORRECTED: 7** |
| Migration counts "226 / 140 / 44 / 81 / 41 / ~240" | Migration files in `Core/Datastore/Migration/` (the count includes `000_…check`), with the highest id in brackets: Sonarr **226** (233); Radarr **140** (242); Prowlarr **43** (44); Lidarr **81** (81); Bookshelf **42** (41); Whisparr v2 **32** (221); Whisparr-Eros **36** (245) | The first five are CONFIRMED (file counts). **"~240" is CORRECTED:** no fork has ~240 migrations. Whisparr v2 has 32 files (1–18, then 202–221) and Eros has 36 (1–26, then 236–245). ~240 is only a max id |
| Whisparr v2 migrations renumbered and incompatible with Sonarr's | Example: `216_telegram_link_preview` in Whisparr is `218_telegram_link_preview` in Sonarr; `221_add_exclusion_tags…` is Sonarr's `218` slot. Upstream 203/204/206/217/219/220 are missing | CONFIRMED |
| Seerr SQLite/Postgres migration trees 57/21 | 55 / 20 | CONFIRMED (≈) |
| autobrr 99/88 | 99 SQLite / 89 Postgres | CONFIRMED (≈) |
| Notifications: Seerr 10, autobrr ~10, Profilarr 4 | Seerr 10 agents (including WebPush); autobrr 9 senders; Profilarr 4 (discord, ntfy, telegram, webhook) | CONFIRMED |
| ≥150–200k duplicated infra (§2.1); ~200k+ (§7) | Infra directories per fork (`NzbDrone.Common/Host/Mono/Windows/Update/SignalR`, `*.Http`, Core `Datastore/Messaging/Jobs/Housekeeping/HealthCheck/Backup/Update/Authentication/ThingiProvider/Configuration/Instrumentation`): 35–47k each, ~285k across 7 forks, so ~240k is redundant | CONFIRMED (a lower bound) |

### §2.2 Providers

| Claim | Measured | Verdict |
|---|---|---|
| Download clients per fork 18–22 | Concrete `DownloadClientBase` classes (the count of `override string Name`): Sonarr 20, Radarr 19, Lidarr 18, Bookshelf 17, Whisparr 20, Eros 18, Prowlarr 18. As directories it is 15–18 | **CORRECTED: 17–20** |
| Cleanuparr clients: qBit, Transmission, Deluge, µTorrent, rTorrent | The same five | CONFIRMED |
| Kapowarr clients: qBittorrent, Transmission | The same two (plus its DDL hosts) | CONFIRMED |
| Fork notifications 19–27 | Sonarr 25, Radarr 26, Lidarr 25, Bookshelf 25, Whisparr 25, Eros 27, Prowlarr 20 | CONFIRMED (the range is 20–27) |
| Prowlarr 69 native indexers | 68 classes with `override string Name` in `Indexers/Definitions` (65 without the generic Cardigann/Newznab/Torznab); 11 are `[Obsolete]` | CONFIRMED (≈) |
| Jackett 65 natives | 66 top-level `.cs` files in `Indexers/Definitions` minus `CardigannIndexer.cs` = **65** | CONFIRMED |
| Prowlarr ~558 Cardigann definitions (v11) | **557** `definitions/v11/*.yml` in Prowlarr/Indexers@master | CONFIRMED |
| Jackett 563 YAML | 563 | CONFIRMED |
| 504 byte-identical | **504** exactly. The measure is git blob SHA equality: 556 names are shared and 504 of those have identical blobs; any-name SHA overlap is also 504. Prowlarr-only: 1; Jackett-only: 7 | CONFIRMED |
| autobrr 118 IRC announce definitions | 118 top-level `internal/indexer/definitions/*.yaml`. **115 have an `irc:` block**; the other 3 are generic_newznab/rss/torznab. Another 16 sit in `deprecated/`. BTN, PTP, RED, OPS, GGn and HDB are all present | NUANCE: 115 IRC |
| Plex watchlist implemented 4 times (Sonarr, Radarr, Seerr, Radarr's RSS variant) | Sonarr has **both** `ImportLists/Plex` (API) and `ImportLists/Rss/Plex`, and so does Radarr. Whisparr v2 carries both copies too. Seerr has `plextv.ts` + `watchlistsync.ts`. That is **7 code copies of 3 designs** (Plex API, Plex RSS, Seerr sync) | **CORRECTED** (Sonarr's RSS variant and the Whisparr copies are missing from the claim) |
| Bazarr ~60 provider modules / ~45 live | **65** provider modules in `subliminal_patch/providers` (69 files minus `__init__`, `_agent_list`, `mixins`, `utils`), plus 8 in the vendored `subliminal/providers`. The UI list (`Settings/Providers/list.tsx`) offers **55** providers (56 entries including the `anidb` integration). Excluding the 3 that the UI marks broken or that the analysis calls "Dead?" (bsplayer, greeksubtitles, subs4free), and the local `embeddedsubtitles` and `whisperai` leaves ~50 | **CORRECTED:** ~65 modules; ~50–55 offered |

### §2.3 Domain logic

| Claim | Measured | Verdict |
|---|---|---|
| Sonarr Parser.cs: 142 regexes | 142 `new Regex(` in `Parser/Parser.cs` (108 of them in the `ReportTitleRegex` cascade) | CONFIRMED |
| 1,631 `[TestCase]` parser tests | 1,631 `[TestCase(` lines in `Core.Test/ParserTests` | CONFIRMED |
| Radarr year bounds 1800–2099 | 8 title regexes use `(1(8\|9)\|20)\d{2}`. **Two active ones still use `(19\|20)`** (German/TrueFrench, line 44; the year-first fallback, line 70), and one more is commented out. `ParsingService` treats year < 1800 as unknown | NUANCE |
| CF spec types: Sonarr 8, Radarr 10, Lidarr 4, Profilarr 11 | Sonarr 8, Radarr 10, Lidarr 4, Profilarr 11 (`evaluator.ts` switch). Also Bookshelf 4, Whisparr 7, Eros 9 | CONFIRMED |
| Languages Sonarr 54, Radarr 60 "+ Any/Original", Profilarr 59 | Sonarr 54 (including Unknown and Original); Radarr 60 (**already including** Unknown, Any and Original); Profilarr enum 59 (uses Radarr ids); Lidarr 34 | CONFIRMED. The "+ Any/Original" wording double-counts |
| Integer ids differ between Sonarr and Radarr | 50 shared names: 35 have the same id and **15 differ**. Examples: Arabic S26/R31, Hindi S27/R26, Bulgarian S28/R29, Malayalam S29/R48, Ukrainian S30/R32, Thai S32/R28, Romanian S35/R27, Persian S37/R33. Sonarr has `PortugueseBrazil`; Radarr calls it `PortugueseBR` | CONFIRMED |
| Lidarr beets-style identification (distance, Munkres, AcoustID) | `TrackImport/Identification/{Distance,DistanceCalculator,Munkres,CandidateService}.cs`, `Parser/FingerprintingService.cs` | CONFIRMED |
| Bookshelf: "the same engine inherited from Lidarr, with ISBN/ASIN" | `BookImport/Identification/Distance*.cs` exists with isbn/asin weights. **No `Munkres.cs`, no fingerprinting service**; `AcoustIdResults` survives only as an unused property on `LocalBook` | **CORRECTED:** Bookshelf keeps the distance matcher but not Munkres or AcoustID |
| Bazarr and the Arrs keep two indexes of the same subtitle files | Sonarr and Radarr `TableMapping.cs:150` `SubtitleFiles` (+`ExtraFiles`, `MetadataFiles`); Bazarr `table_episodes_subtitles` / `table_movies_subtitles` (plus a `subtitles`/`missing_subtitles` column on `table_episodes`/`table_movies`) | CONFIRMED |
| Seerr availability duplicated for 4K; "three scanners + daily sync" | `Media.ts` has `status4k`, `serviceId4k`, `externalServiceId4k`, `externalServiceSlug4k`, `ratingKey4k`, `jellyfinMediaId4k` (persisted) plus 5 computed `*4k` URL fields; `Season.status4k`. `server/lib/scanners/` has **4 scanners** (plex, jellyfin, sonarr, radarr) plus `availabilitySync.ts` | CONFIRMED for 4K; **scanner count is 4, not 3** |

### §2.4 Per-kind differences

| Claim | Measured | Verdict |
|---|---|---|
| Radarr: Movie ← MovieFile, one file per movie | `Movie.MovieFileId` (a single int); `HasFile => MovieFileId > 0` | CONFIRMED |
| Sonarr: Episode ← EpisodeFile (multi-episode files) | `Episode.EpisodeFileId` (single); one `EpisodeFile` can back many episodes | CONFIRMED: one file per episode, one file may hold N episodes |
| decisions.md D4: "Every PVR stores one file per item" | True for Sonarr, Radarr, Whisparr v2/v3 and Lidarr (per track). **Bookshelf:** `Edition.BookFiles` is a `List<BookFile>` (multi-file audiobooks). **Kapowarr** is M:N (per §2.4 itself) | NUANCE: the right framing is one *version/quality* per item, not one file |
| Subtitle scoring: episode max 360, **movie max 120** | `subliminal_patch/score.py` `MAX_SCORES` = the sum of non-hash weights: **episode 360, movie 180** (60+40+30+30+15+5). 120 is only the value that migration `c00d40af333c` backfills into old `table_history_movie.score_out_of` rows (pre-March-2026 scoring) | **CORRECTED: movie max is 180** |
| ~45 live subtitle providers | See §2.2: ~50–55 offered in the UI | CORRECTED (≈) |

### Whisparr v2 / v3 overlap

| Claim | Measured | Verdict |
|---|---|---|
| Whisparr v2: only 41 unique Core files | **46** paths in `src/NzbDrone.Core` that are absent from Sonarr (45 `.cs`). Sonarr also has 381 Core files Whisparr lacks | CONFIRMED (≈, +12%) |
| 875 byte-identical after renaming | 863 byte-identical at the same path; **885** after `Sonarr→Whisparr` / `sonarr→whisparr` replacement and CRLF normalisation (of 1,368 Core files, 65%) | CONFIRMED (≈) |
| Eros: 157 unique files | 158 fork-only Core paths, **157 `.cs`** | CONFIRMED |
| Eros: ~60% of Core files identical to Radarr | 844 byte-identical (55.7%); 882 after `Radarr→Whisparr` renaming (**58.2%** of 1,515) | CONFIRMED |
| Eros: 244 Radarr files absent | 245 | CONFIRMED |
| Eros: 0 commits behind Radarr | The `UPSTREAM_SYNC.md` high-water mark is `c90668a520…`. `gh api repos/Radarr/Radarr/compare/c90668a…...develop` gives ahead 0 / behind 0, so the ledger is at Radarr's `develop` tip (2026-09-20). 2 commits are deferred. Cherry-pick history, not merges; the submodule is shallow | CONFIRMED (by the ledger plus GitHub compare) |

### §7 Sizing

| Claim | Measured | Verdict |
|---|---|---|
| Duplicated infrastructure ~200k+ | ~240k redundant (see §2.1) | CONFIRMED |
| Six UI stacks ~450k | ~860k JS/TS/Svelte | **CORRECTED** |
| Forked PVR pipelines "~5 × 40k" | `Core/{Download,MediaFiles,CustomFormats,DecisionEngine,Queue,History,Organizer,Indexers,Notifications,ImportLists}` non-test: Sonarr 59k, Radarr 55k, Lidarr 52k, Bookshelf 51k, Whisparr 52k, Eros 55k. That is **6 × ~54k ≈ 325k** | **CORRECTED** |
| Integration/sync layers ~60–80k | Recyclarr 20k + Profilarr sync/arr/drift 8k + Prowlarr `Applications/` 5k + Cleanuparr Arr code 4k + autobrr arr clients 3k + Seerr servarr API and scanners 2–4k + Bazarr sonarr/radarr 2.6k + Unpackerr 1k ≈ 45–50k by direct directory count. Mirrored models spread elsewhere could plausibly reach 60k | UNVERIFIABLE (order of magnitude plausible) |
| Irreducible domain ~150–200k; dead code ~40k+ | Not measured: these are judgement buckets | UNVERIFIABLE |

## Notes

1. **LOC method.** `git ls-files` restricted by extension and path regex; physical lines via `cat | wc -l` (blanks and comments included); "non-test" drops paths matching `test|spec|__tests__|fixtures?|.Test.|Tests?/` (case-insensitive). No cloc/tokei/scc was installed. Per-app figures in comparison.md are consistent with this method within about ±10%, except Cleanuparr: it matches only if the 137k lines of EF Core `Migrations/*.Designer.cs`/snapshots are excluded.
2. **Headline totals are the main error.** Backend is ~1.27M lines, not ~900k. Frontend is ~860k, not ~450k. The per-app table rows are right; the summary sentence and the §7 "Six UI stacks ~450k" row are not. Consequence for §7: the monolith's reduction ratio is larger than stated, not smaller.
3. **"6 copies" vs "7 copies".** §2.1's heading says seven, but its table says 6. Prowlarr has every listed subsystem, so it should say 7. The migration row lists only 6 numbers, and the sixth (~240) matches no fork. Real file counts: Sonarr 226, Radarr 140, Prowlarr 43, Lidarr 81, Bookshelf 42, Whisparr v2 32, Eros 36.
4. **Bazarr movie score max is 180, not 120** (current `MAX_SCORES`). Anything that ports thresholds (the 70% movie minimum) must use 180. 120 appears only in a history backfill.
5. **Bookshelf identification** inherited Lidarr's distance framework but not Munkres or AcoustID. §2.3's "same engine … (distance, Munkres, AcoustID)" should be read as "same distance framework".
6. **Plex watchlist.** Sonarr, Radarr and Whisparr v2 each ship an API and an RSS implementation, and Seerr has its own: 7 copies of 3 designs, not 4.
7. **Shallow submodules.** `whisparr` and `whisparr-eros` are shallow clones, so their history claims (daily cherry-picks, commit counts) cannot be checked locally; GitHub was used for Eros.
8. **Language ids.** Sonarr and Radarr agree for ids 0–25 and many others, but diverge from 26 upward (Hindi/Arabic/Romanian/Thai/Bulgarian block, Persian, Slovak, etc.). Profilarr's enum follows Radarr's ids.

### Scripts

`loc.sh` (per-repo line counter):
```bash
#!/bin/bash
# usage: loc.sh <repo> <ext-regex> [path-include-regex]; prints "<all> <non-test>"
repo=$1; ext=$2; inc=${3:-.}
cd "$repo"
files=$(git ls-files | grep -E "\.($ext)\$" | grep -E "$inc")
all=$(echo "$files" | grep -v '^$' | tr '\n' '\0' | xargs -0 cat 2>/dev/null | wc -l)
nt=$(echo "$files" | grep -viE '(test|spec|__tests__|/fixtures?/|\.Test\.|Tests?/)' | grep -v '^$' | tr '\n' '\0' | xargs -0 cat 2>/dev/null | wc -l)
echo "$all $nt"
```

`totals.sh` (backend and frontend per app): calls `loc.sh` with `cs` for .NET apps, `'js|jsx|ts|tsx' '^frontend/'` for fork UIs, `py`/`go`/`ts '^server/'` etc. for the others. Cleanuparr subtracts `Migrations/`, and Bazarr counts `bazarr/` + `custom_libs/`, excluding `libs/`.

`cmp.py` (fork vs upstream file overlap):
```python
import sys,subprocess
S='submodules/'; fork,up,sub=sys.argv[1:4]
ren=[(a.encode(),b.encode()) for a,b in (x.split('=') for x in sys.argv[4:])]
files=lambda r:{f for f in subprocess.check_output(['git','-C',S+r,'ls-files',sub]).decode().split('\n') if f}
F,U=files(fork),files(up); rd=lambda r,f:open(S+r+'/'+f,'rb').read()
def norm(b):
    b=b.replace(b'\r\n',b'\n')
    for a,c in ren: b=b.replace(a,c)
    return b
common=F&U
print(len(F), len(F-U), sum(rd(fork,f)==rd(up,f) for f in common), sum(norm(rd(fork,f))==norm(rd(up,f)) for f in common))
# run: cmp.py whisparr sonarr src/NzbDrone.Core Sonarr=Whisparr sonarr=whisparr
#      cmp.py whisparr-eros radarr src/NzbDrone.Core Radarr=Whisparr radarr=whisparr
```

Cardigann overlap:
```bash
gh api 'repos/Prowlarr/Indexers/git/trees/master?recursive=1' \
  --jq '.tree[]|select(.path|test("^definitions/v11/[^/]+\\.yml$"))|"\(.sha) \(.path)"' > p11.txt
git -C submodules/jackett ls-files -s 'src/Jackett.Common/Definitions/*.yml' | awk '{print $2" "$4}' > jk.txt
# compare basenames and blob SHAs (blob SHA equality == byte identity)
```

Other counts:
- Migrations: `git ls-files src/NzbDrone.Core/Datastore/Migration/ | grep -E '/[0-9]+_'`.
- Download clients and notifications: `git grep -h 'override string Name' -- src/NzbDrone.Core/{Download/Clients,Notifications}`.
- Languages: regex `public static Language (\w+) => new Language\((-?\d+)` on `Languages/Language.cs`.
- Parser: `grep -c 'new Regex(' Parser.cs`, and `git grep -h '[TestCase(' -- src/NzbDrone.Core.Test/ParserTests | wc -l`.
