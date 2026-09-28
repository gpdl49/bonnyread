# Release Evaluation — Design

Status: draft for review (2026-09-28). Scope: everything between a normalised `Release` (handed over by the acquisition layer) and a grab decision (handed to the download lifecycle). Out of scope and owned by other designs: the library model, indexers and IRC ingest, the PCD engine itself, download tracking, import, metadata, runtime/HA, API.

Binding inputs: [../decisions.md](../decisions.md) (D1, D1a–c, D4, D8, D9, D10, D14, D16, D19 especially), [../analysis/comparison.md](../analysis/comparison.md) §2.3, and the per-app analyses. Paths below are relative to `submodules/<app>/`; `Sonarr Core/` = `sonarr/src/NzbDrone.Core/`, likewise for Radarr, Lidarr, Whisparr (v2) and Eros (v3).

---

## 1. Outcomes and invariants to keep

These are the user-visible results the design must reproduce. Each one is backed by a golden test corpus or by reference code that becomes a test fixture.

### 1.1 Parser results (D1a)

- **Structure** (title, year, season/episode, multi-episode, absolute numbers, air date, packs, editions, AKA titles) must equal the Sonarr/Radarr cascades on their corpora: 1,631 Sonarr `[TestCase]`s in `sonarr/src/NzbDrone.Core.Test/ParserTests`, and Radarr's `ParserTests`. The Sonarr cascade order is load-bearing (`Sonarr Core/Parser/Parser.cs`, 142 regexes); the Radarr cascade is `Radarr Core/Parser/Parser.cs:27-70`, with the year bounded to 1800–2099.
- **Attributes** (source, resolution, modifier, revision, language, group, hash, hardcoded subs) must equal `QualityParser.cs`, `LanguageParser.cs` and `ReleaseGroupParser.cs` in each fork.
- **Revision semantics** (`Sonarr Core/Qualities/Revision.cs:33-56`): `Real` is compared first, then `Version`. `IsRepack` is **not** part of the ordering. PROPER and REPACK both bump `Version`; anime `v2` sets `Version`.
- **Rejected names.** Hashed or obfuscated titles are rejected (Radarr `RejectHashedReleasesRegex`, `Parser.cs:73-100`, and in every fork).
- **Per-kind structure.** Adult: dates (day/month swap, a date more than one day in the future is rejected, 2-digit years), product codes (the ten v2 variants), StashIDs, studio episode numbers (`whisparr.md` §3.2, `whisparr-eros.md` §3.2). Music: album/discography forms (`Lidarr Core/Parser/Parser.cs:41-133`). Comics: Kapowarr `backend/base/file_extraction.py` with its tests.

### 1.2 Matching a release to library items

| Invariant | Evidence |
|---|---|
| TV series lookup order: search-criteria titles, then scene-mapping title (+season), then all titles/aliases, then title+year, then indexer TVDB/IMDb id | `Sonarr Core/Parser/ParsingService.cs:43-223` |
| A title that maps to a *different* known TVDB id is rejected as "matches another series", never guessed | `Sonarr Core/DecisionEngine/DownloadDecisionMaker.cs:103-114` |
| Episode resolution by numbering mode: standard (with XEM scene→TVDB when `UseSceneNumbering`), anime absolute, daily air date (+ part), specials by title | `ParsingService.cs:276-330` |
| Movie lookup: IMDb id, then TMDb id, then title+year (year or secondary year), then title only; titles include original, alternative and translated titles | `Radarr Core/Parser/ParsingService.cs:163-203`, `Radarr Core/Movies/MovieRepository.cs:134-210` |
| Music: fuzzy artist and album location inside the title (bitap), aliases, year ±3 | `lidarr.md` §3.2 |
| Adult: StashID, then product code (indexed, non-unique), then studio(+aliases)+date, then the graded same-date matcher; **reject ambiguous matches everywhere, including fuzzy** (D10) | `whisparr/src/NzbDrone.Core/Tv/EpisodeService.cs` (MatchEpisodes), `whisparr-eros/src/NzbDrone.Core/Movies/MovieService.cs:771-1012,1228-1384`, `Movies/MovieParseMatchType.cs` |
| Search-time guard: a result from a targeted search must map to the searched units | Sonarr `Specifications/Search/*`, Radarr `Search/MovieSpecification.cs`, Eros `ParsingService.cs:299-314` |
| Languages depend on the item: indexer-declared languages win; languages that appear only inside the episode title are dropped; MULTI is expanded by the indexer's `MultiLanguages`; "unknown" falls back to the item's original language | `Sonarr Core/Download/Aggregation/Aggregators/AggregateLanguages.cs:25-135` |

### 1.3 Quality ordering

- Rank comes from the **profile's item order**. Qualities in the same group are **equal** for upgrade purposes (`respectGroupOrder=false`); group order matters only for ranking and size lookup (`Sonarr Core/Qualities/QualityModelComparer.cs`, `DownloadDecisionComparer.cs:187`).
- Revision breaks ties for `QualityModel` comparison (`QualityModelComparer.cs`, `Compare(QualityModel…)`).

### 1.4 Custom-format match semantics

From `Sonarr Core/CustomFormats/CustomFormatCalculationService.cs:147-168` and `SpecificationMatchesGroup.cs:10`, re-implemented identically by Profilarr (`profilarr/src/lib/server/pcd/entities/customFormats/evaluator.ts`):

1. Conditions are **grouped by type**.
2. A group matches iff **every `required` condition passes AND at least one condition passes**. Formally: `!(any(required && !pass) || all(!pass))`.
3. `negate` inverts a single condition *before* grouping (`CustomFormatSpecificationBase.cs:23-33`).
4. The format matches iff **every group matches**.
5. Matched formats are sorted by name. The score is the sum of the profile's scores for the matched formats.

Condition details that affect results:
- `ReleaseTitle` matches the release title **or** the file name (`ReleaseTitleSpecification.cs:11`).
- `Size` is `min < size <= max` in GB over the **whole release** (`SizeSpecification.cs:35`), so it is not scaled per episode.
- Regexes are case-insensitive .NET (`Radarr Core/CustomFormats/Specifications/RegexSpecificationBase.cs`).
- Language `Original` resolves to the item's original language, and `ExceptLanguage` means "contains any other language" (`Radarr …/LanguageSpecification.cs`).
- A null field never matches, even when negated, for Language (`IsSatisfiedByWithNegate`). For regex fields a null value is a non-match, so its negation is a match.

### 1.5 Profile gates and scoring

- Quality must be allowed (`QualityAllowedByProfileSpecification`).
- `score >= MinFormatScore` (`CustomFormatAllowedByProfileSpecification.cs:21-26`).
- Size must lie within per-quality min/max, as MB per minute × runtime, where the runtime is the **sum over covered episodes** (`AcceptableSizeSpecification.cs`, `DownloadDecisionComparer.cs:210-218`).
- Radarr has a profile-level language gate with `Original`/`Any` sentinels (`Radarr …/Specifications/LanguageSpecification.cs`).
- PCD profiles carry `minimum_custom_format_score`, `upgrade_until_score`, `upgrade_score_increment > 0`, `upgrades_allowed` and one `upgrade_until` item (`profilarr/docs/backend/0.schema.sql:71-81`).

### 1.6 Upgrade rules (exact order)

`Sonarr Core/DecisionEngine/Specifications/UpgradableSpecification.cs:31-122`. Given profile `p`, current `c` and new `n`:

1. If `n` has a better quality and the **quality cutoff is not met** → upgrade.
2. If `n` has a worse quality → reject `BetterQuality`.
3. If propers are not `DoNotPrefer` and `n` has a better revision → upgrade.
4. If `!p.UpgradeAllowed` → reject `UpgradesNotAllowed`.
5. If propers are not `DoNotPrefer` and `n` has a worse revision → reject `BetterRevision`.
6. If `n` has a better quality (so the cutoff is met) → reject `QualityCutoff`.
7. If `score(n) <= score(c)` → reject `CustomFormatScore`.
8. If `score(c) >= p.CutoffFormatScore` → reject `CustomFormatCutoff`.
9. If `score(n) < score(c) + p.MinUpgradeFormatScore` → reject `MinCustomFormatScore`.
10. Otherwise → upgrade.

Supporting rules:
- `QualityCutoffNotMet` uses `Cutoff`, or the **first allowed quality** when upgrades are disabled. It is also true for a same-quality revision upgrade (`:124-140`).
- `CutoffNotMet` is the quality cutoff OR the CF cutoff, where the CF cutoff uses `MinFormatScore` when upgrades are disabled (`:142-168`).
- `IsRevisionUpgrade` requires **the same quality**, so a WEBRip PROPER never upgrades a WEBDL (`:170-182`).
- **Season packs** (`UpgradeDiskSpecification.cs:34-104`): missing episodes count as upgradable. Existing files count if `CutoffNotMet` and `IsUpgradable`. The pack is accepted under `SeasonPackUpgrade` = Any (≥1), All (100%), or Threshold (≥ n%). `FullSeasonSpecification` rejects a pack until every episode has aired, unless the release profile's `AllowSeasonPackWithoutAllEpisodesAired` is set.
- **Incumbents other than the file on disk**: the queue (`QueueSpecification.cs`) and a recent grab in history (`RssSync/HistorySpecification.cs`, 12 h, or any age when completed download handling is off) are checked with the same `CutoffNotMet` + `IsUpgradable`. A FailedPending queue item is ignored.

### 1.7 Propers and repacks

- `DownloadPropersAndRepacks ∈ {PreferAndUpgrade, DoNotUpgrade, DoNotPrefer}`.
- A **REPACK** (`RepackSpecification.cs:27-81`) and an **anime version bump** (`AnimeVersionUpgradeSpecification.cs:56-68`) are only accepted as upgrades from **the same release group**, and they are rejected when either group is unknown. A **PROPER** from any group is fine.
- In RSS only, a proper for a file added more than 7 days ago is rejected (`RssSync/ProperSpecification.cs:51-55`).
- With `DoNotPrefer`, revision drops out of ranking (`DownloadDecisionComparer.cs:73-76`) and out of the upgrade rules.

### 1.8 Delay semantics

From `Sonarr Core/DecisionEngine/Specifications/RssSync/DelaySpecification.cs:32-115`:
- A user-invoked search skips the delay.
- The delay is per protocol, from the delay profile.
- The delay is skipped for:
  - a revision upgrade of an existing file on the **preferred** protocol when `PreferAndUpgrade` is set;
  - `BypassIfHighestQuality` (quality ≥ the last allowed quality) on the preferred protocol;
  - `BypassIfAboveCustomFormatScore` (score ≥ minimum) on the preferred protocol.
- The release is accepted once the **oldest pending release for any of the same episodes** is older than the delay. It is held while its own **age (from publish date)** is below the delay.
- Held releases are **persisted** (`PendingReleases` table, `Download/Pending/PendingReleaseService.cs`) with reasons Delay, DownloadClientUnavailable or Fallback (`Download/ProcessDownloadDecisions.cs:178-194`).

### 1.9 Ranking (comparer order)

**Sonarr** (`Sonarr Core/DecisionEngine/DownloadDecisionComparer.cs:28-44`), evaluated in order until one differs:
1. **Quality**: profile index, then revision unless `DoNotPrefer`.
2. **Custom-format score**.
3. **Protocol**: preferred protocol from the delay profile.
4. **Episode count**: a full season pack is preferred. After that, *fewer* episodes rank higher for standard series and *more* for anime.
5. **Episode number**: the lower first episode.
6. **Indexer priority**: a lower value is better.
7. **Peers, if both are torrents**: round(log10(seeders)), then round(log10(peers)).
8. **Age, if both are Usenet**: buckets <1 h, ≤24 h, ≤7 d, then −round(log10(days)).
9. **Size**: closest to preferred × runtime, rounded to 200 MB. With no preferred size, larger is better.

**Radarr** (`Radarr Core/DecisionEngine/DownloadDecisionComparer.cs:29-44`): Quality → CF score → Protocol → IndexerPriority → **IndexerFlags** (only if `PreferIndexerFlags`) → Peers → Age → Size.

**Lidarr** (`Lidarr Core/DecisionEngine/DownloadDecisionComparer.cs`): Quality → CF → **YearMatch** → Protocol → IndexerPriority → Peers → **AlbumCount** → Age → Size.

**Selection** (`ProcessDownloadDecisions.cs:51-117`): decisions are ranked per series. The best is taken first, and anything that overlaps an already-grabbed episode set is skipped (Sonarr stores it as a Fallback). If a grab fails because the client is unavailable, the rest of that protocol's candidates go to pending.

### 1.10 Blocklist

A release is blocklisted **per item** (per series in Sonarr) by torrent info hash, or by title, indexer and publish date (`BlocklistSpecification.cs`). Blocklisting follows failed downloads.

### 1.11 Explainability

The UI shows rejection reasons per release:
- Arr `DownloadRejection` reason and message;
- autobrr `RejectReasons` (field, got, expected; `autobrr/internal/domain/rejections.go`);
- Profilarr's per-condition pass/fail for CF tests;
- Kapowarr's human-readable match rejections (`kapowarr.md` §3.2).

---

## 2. How the references do it today, and the accidental complexity

| Concern | Today | Accidental complexity |
|---|---|---|
| Parsing | Nine parsers (§2.3 of comparison.md). Profilarr runs a C# sidecar (`profilarr/src/services/parser/`) only so its CF tests agree with the Arrs. autobrr uses `rls` | Each app re-parses the same title. Profilarr keeps two caches (`parsed_release_cache`, `pattern_match_cache`) to hide sidecar latency |
| Spec pipeline | `DownloadDecisionMaker.GetDecisionForReport` groups specs by `SpecificationPriority` and **stops at the first tier with a rejection** (`DownloadDecisionMaker.cs:186-203`) | Tiers exist only because some specs query the DB (history, queue, pending). Users therefore see *some* reasons, not all |
| Incumbent comparison | The same `IsUpgradable` is run three times against three incumbents: disk (`UpgradeDiskSpecification`), queue (`QueueSpecification`), history (`HistorySpecification`). Each rebuilds CFs by re-parsing stored source titles (`CustomFormatCalculationService.cs:61-115`) | The history 12 h window and the "CDH disabled" branch guess whether a grab is still in flight, because tracked-download state isn't authoritative |
| Custom formats | Sonarr 8, Radarr 10, Lidarr 4, Whisparr 7, Eros 9 spec types (verification/duplication-and-sizing.md). Profilarr re-implements evaluation (`evaluator.ts`, 641 lines) plus `filterConditionsForArrType` | Per-app enum ids everywhere: indexer flags (Sonarr `Freeleech=1`, Radarr `G_Freeleech=1`, `PTP_Golden=8` in `Radarr Core/Parser/Model/IndexerFlags.cs`), languages (15 of 50 shared names differ in id), qualities (`quality_api_mappings`) |
| Release profiles | Sonarr keeps Required/Ignored terms (`Profiles/Releases/ReleaseProfile.cs`) alongside CFs | Two mechanisms for "must/must not contain" |
| autobrr filters | About 80 flat columns (`internal/domain/filter.go:449` `CheckFilter`), comma-string lists, wildcard/regex, first-match-wins by priority, duplicate profiles (`database/release.go`), smart episode | They duplicate CF logic (title, group, resolution, source, codec, HDR, language, size) and replace upgrade logic with coarse "same fields as a prior push" checks, with no quality ranking. `shows` lists mirror the library. The real decision is delegated to the Arr through `release/push`, and rejections come back as strings |
| Delay | Arr delay profiles (minutes, persisted, re-evaluated **only at the next RSS sync**: `Indexers/RssSyncService.cs:42`), autobrr filter delay (seconds, goroutine sleep, lost on restart), Profilarr pushes a delay profile into the Arr (default id=1 only) | Three places. Granularity is tied to the RSS interval (≥10 min) |
| Scoring readers | Arr, Profilarr (upgrade scoring), Recyclarr, Cleanuparr (CF-score mirror) | Four readers of one number |
| Adult matching | v3 runs a fuzzy pass **before** the graded matcher and takes the top score with no margin (`MovieService.cs:1228+`) | A correctness regression (D10) |
| Language | Aggregation in the decision path (item-dependent); Radarr profile language; language CFs | Two mechanisms (gate and CF) |
| Ranking | Three near-identical comparers | Copy-paste per fork |

---

## 3. Proposed design

### 3.1 Pipeline

```
Release ─▶ [0 Intake gates] ─▶ [1 Parse] ─▶ [2 Match] ─▶ [3 Facts] ─▶ [4 Candidates per Target]
                                                                        │
            ┌───────────────────────────────────────────────────────────┘
            ▼
      [5 Evaluate: profile → incumbents → timing] ─▶ Decision{Accept|Reject|Defer}
            │
            ▼
      [6 Rank + select per (target, unit set)] ─▶ [7 Commit: grab reservation tx] ─▶ download lifecycle
            │
            └─ Defer ─▶ pending_grab (persisted) ─▶ timer job at due_at ─▶ back to 5
```

Principles:
- **Stages 1–3 are pure and profile-independent.** They run once per release (per item). They are cached and shared by every slot and every rule.
- **Stage 5 has no DB I/O.** Incumbents come from an in-memory `SlotState` snapshot (§3.10). So every check runs every time, and every reason is reported (no "priority tiers").
- **Stage 7 re-validates** against the database inside the grab transaction (optimistic concurrency). That makes the in-memory snapshot safe across HA nodes (D14) and across concurrent paths (IRC and RSS at the same moment).

### 3.2 Package shape

```
parse/            pure: title → ParsedRelease. No library, no DB.
  attr/           shared attributes: quality axes, revision, language, group, edition, hash, hardsub, container
  tv/ movie/ music/ book/ adult/ comic/    structure sub-parsers (D1c: book/comic ported with fixtures)
  corpus/         golden fixtures (Sonarr, Radarr, Lidarr, Bookshelf, Whisparr v2/v3, Kapowarr), run as table tests
  tokens/         D8 seam: Tokeniser interface; implementations: cascade (regexp2) and rls-hybrid
match/            ParsedRelease × LibraryIndex → Match | MatchRejection (per-kind matchers)
quality/          canonical catalogue, families, axes → QualityID, languages, indexer flags
cond/             condition engine: compile (regexp2 per D1 + normalisation), evaluate, trace
evaluate/         facts, candidates, profile scoring, upgrade, propers, delay policy, findings
rank/             comparer keys per kind, selection
pending/          persisted holds, timers, fallbacks
autograb/         (later phase, D16) rule scope + limits + dedupe identity; reuses cond/ and evaluate/
```

`parse`, `quality` and `cond` have no dependency on the library or on storage. That lets the PCD test UI, the import pipeline (which re-parses file names), subtitles (D1c) and CI all call them directly.

### 3.3 Data types

```go
// Handed over by acquisition (sketch; that design owns the final shape).
type Release struct {
    ID          ReleaseID            // stable: indexer + guid (or infohash)
    Source      SourceKind           // RSS | Search | Interactive | IRC | Manual
    Search      *SearchContext       // target units + UserInvoked flag
    IndexerID   IndexerID
    Protocol    Protocol             // Usenet | Torrent | DDL (D11)
    Title       string
    Size        int64                // 0 = unknown (IRC announces may omit it)
    PublishedAt, FirstSeenAt time.Time
    Categories  []int
    Seeders, Peers *int
    IndexerPriority int
    IDs         ExternalIDs          // tvdb, tmdb, imdb, tvmaze, stash, mbid, isbn…
    Flags       IndexerFlags         // canonical, §3.7
    Languages   []Lang               // indexer-declared
    Announce    *AnnounceMeta        // IRC/tracker metadata, §3.8
    InfoHash    string
}

type ParsedRelease struct {
    ParserVersion uint32
    Title         string
    Kind          Kind           // best guess; Unknown allowed
    Structure     Structure      // *TVStructure | *MovieStructure | *MusicStructure | *BookStructure | *AdultStructure | *ComicStructure
    Alternates    []Structure    // other kinds that also parsed (resolved by Match)
    Attr          Attributes
}

type Attributes struct {
    Family     quality.Family             // Video | Audio | Text | Archive(comic)
    Video      *quality.VideoAxes         // Source, Resolution, Modifier, Codec, HDR[], BitDepth, VR projection
    Audio      *quality.AudioAxes         // Codec(s), Bitrate bucket, BitDepth, Channels, Atmos, Lossless
    Text       *quality.TextAxes          // EPUB/AZW3/MOBI/PDF; audiobook M4B/MP3 is Audio
    Revision   quality.Revision           // Version, Real, IsRepack, IsProper
    Languages  []Lang; Multi bool
    Group, Hash, Container, Streaming string
    Editions   []string
    HardcodedSubs string
    Obfuscated, Sample, RawDisk bool
    Detected   map[Axis]DetectionSource   // Name | Extension | MediaInfo (import augmenters reuse this)
}

type Match struct {
    Kind      Kind
    Container ItemRef        // series / artist / author / studio / volume
    Units     []ItemRef      // episodes / movie / album / book edition / scene / issues
    Numbering Numbering      // TVDB | Scene | Absolute | AirDate | Code | StashID
    Method    MatchMethod    // ExternalID | SceneMapping | XEM | CleanTitle | AltTitle | Translation | ProductCode | StudioDate(level) | Fuzzy(margin)
    FullPack  bool           // season pack / discography / TPB
    Evidence  []Finding
}

type Facts struct {           // per (release, item); profile-independent
    R         *Release
    P         *ParsedRelease
    M         *Match
    Languages []Lang          // aggregated per §1.2 row "Languages"
    Runtime   time.Duration   // sum over units (size rules)
    Quality   quality.ID      // canonical id derived from axes
    CF        cond.MatchSet   // bitset over the compiled CF index + optional trace
}

type Target struct {          // exactly one of these is set
    Slot *SlotRef             // D4 version slot of Match.Container/Units
    Rule *autograb.RuleRef    // D16 library-less rule (later phase)
}

type Candidate struct {
    Facts      *Facts
    Target     Target
    Profile    *CompiledProfile   // from the PCD snapshot
    Score      int
    Incumbents map[ItemRef][]Incumbent // per unit: File and/or InFlight
}

type Decision struct {
    Candidate *Candidate
    Verdict   Verdict           // Accept | Reject | Defer
    DeferUntil time.Time        // Defer only
    Findings  []Finding         // every check, pass or fail (pass omitted unless explain=true)
    Rank      RankKey           // precomputed comparer tuple (§3.13)
}

type Finding struct {
    Stage   Stage              // Intake | Parse | Match | Profile | Incumbent | Timing | Commit
    Code    ReasonCode         // stable enum (e.g. QualityNotAllowed, CFBelowMinimum, AmbiguousMatch)
    Outcome Outcome            // Pass | Fail | Defer | Unknown
    Params  map[string]any     // {field, got, want}, {score, min}, {candidates:[…]}
    Trace   *cond.Trace        // CF/condition breakdown, explain mode only
}
```

### 3.4 Parsing

**Structure (D1a).** Each sub-parser in `parse/<kind>` ports its reference cascade in order (on regexp2, per D1) and returns a kind-specific `Structure`. It is accepted when it passes the corpus.
- `tv`: Sonarr `Parser.cs`.
- `movie`: Radarr `Parser.cs`.
- `music`: Lidarr's album patterns.
- `book`: Bookshelf's title/subtitle split.
- `adult`: the **union** of v2's date patterns and product-code variants with v3's ~15 scene patterns, StashID and JAV patterns.
- `comic`: Kapowarr.

**Attributes (D1).** `parse/attr` holds a single quality/language/group/revision parser shared by all kinds. The Arr `QualityParser`/`LanguageParser` tests are its acceptance suite.

**Kind selection.** In order:
1. The search context's kind.
2. The indexer category → the set of kinds.
3. Otherwise, the sub-parsers in a fixed order (adult only when the category or indexer is adult).

Every structure that parses is kept (`Alternates`), and matching picks the one that resolves to a library item. Rejecting unknown releases at parse time would hide explanations.

**D8 seam.** `parse/tokens.Tokeniser` gives sub-parsers the normalised title, bracket groups, the stripped website prefix/suffix, and the attribute spans to exclude from titles. There are two implementations: `cascade` (the ported regex pre-processing) and `rlshybrid` (rls tokens and attributes). The spike runs both against `parse/corpus` and reports per-corpus pass rates. The package API (`parse.Parse(title, hints) *ParsedRelease`) does not change either way.

**Caching.** An LRU keyed by `(ParserVersion, title)`. The same title arrives from several indexers and in repeated RSS polls. The cache is deterministic, so no invalidation is needed beyond the version.

### 3.5 Matching

`match/` holds an in-memory **LibraryIndex** snapshot. It is rebuilt incrementally from library events and swapped atomically. It contains:
- `cleanTitle → []TitleEntry{item, source(primary|alt|translation|sceneMapping{season}|xem|alias), year}`;
- external id maps (tvdb, tmdb, imdb, stash, mbid, isbn/asin, comicvine);
- **product-code index** (non-unique, D10.1);
- per-container unit indices: (season, episode), absolute, scene numbering, air date, studio+date, issue number.

Per-kind matchers keep the reference order (§1.2). One **universal ambiguity rule** covers every kind and every method, fuzzy included (D10.2):

> At the first tier that yields any candidate, if more than one distinct item (or unit set) remains, return `AmbiguousMatch{candidates, tier}`. Never pick the first result. A fuzzy tier also needs a **margin** over the runner-up (Lidarr's and Eros's `TryGetMovieByFuzzyYear` margin); a tie is ambiguous.

Other outcomes are also findings, not errors:
- `UnknownItem`;
- `MatchesOtherItem` (a scene mapping or alias points at a known id that isn't in the library);
- `UnknownUnits` (container found but episode/issue not);
- `SearchMismatch`;
- `NotWantedKind`.

Specifics:
- **TV:** scene mappings and XEM are bundled/community data keyed by TVDB (D3a/D3c). Specials are resolved by title only when the structure says "possible special" (`DownloadDecisionMaker.cs:81-89`).
- **Movies:** the parsed IMDb/TMDb id in the name is tried before indexer ids. SecondaryYear gives off-by-one tolerance.
- **Adult:** StashID → code → studio(+aliases)+date → graded levels in `MovieParseMatchType` order, with v3's month/year widening on `-01` dates. The optional fuzzy tier runs **last**, only over same-date scenes, needs threshold **and** margin, and falls under the ambiguity rule. Product codes are also used by the download-lifecycle/import fallback (D10.1); that's the same `match` API with `Structure.Code` only.
- **Music and books:** distance-based fuzzy location as in Lidarr, with margin. ISBN/ASIN go straight to the edition.
- **Comics:** Kapowarr's match filter (title/alias equality, volume number, year range, issue in volume, special-version conflict), expressed as findings.

### 3.6 Canonical quality model (per-kind axes)

- **Families**: `Video` (TV, movies, adult), `Audio` (music, audiobooks), `Text` (ebooks), `Archive` (comics).
- **A quality is a named point in a family's axis space.** The catalogue is data, keyed by the **PCD canonical names** (D9: the monolith owns the enum; `quality_api_mappings` disappears).
  - Video: `Source × Resolution × Modifier`, where Source covers CAM…Bluray and Modifier is NONE/REGIONAL/SCREENER/RAWHD/BRDISK/REMUX (`Radarr Core/Qualities/Modifier.cs`), plus an optional VR flag for adult.
  - Audio: `Codec × BitrateBucket × BitDepth` (Lidarr's 38 qualities, `Lidarr Core/Qualities/Quality.cs`).
  - Text: `Format`.
  - Archive: `Format` (CBZ/CBR/PDF).
- `quality.Derive(axes) → ID` is one table per family. Sonarr's `Bluray-1080p Remux` and Radarr's `Remux-1080p` collapse to one ID. Sonarr `Raw-HD` (source TelevisionRaw) and Radarr `Raw-HD` (modifier RAWHD) collapse too. The Arr golden tests assert `Derive(Parse(title)) == expected`.
- **A profile is bound to exactly one family.** A release of another family gets `QualityFamilyMismatch`. This alone fixes Bookshelf's mixed ebook/audio ladder (`bookshelf.md` §2, item 1): an ebook slot and an audiobook slot have different profiles (D4).
- **Size limits** belong to a named quality-definition set (PCD `*_quality_definitions`). Units are per family: Video MB/min × runtime; Audio kbps-equivalent × duration; Text and Archive absolute MB.

### 3.7 Languages and indexer flags

- **`Lang`** is an ISO 639 code with an optional region (`pt-BR`, `es-419`, `nl-BE`), so there are no integer ids.
  - Sentinels that exist only in conditions and profiles: `Original`, `Any`.
  - The parse result may be `Unknown`.
  - `MULTI` is a parse flag, not a language; it expands through the indexer's `MultiLanguages` in stage 3.
  - Dictionarry/PCD language names map by a table.
  - Hearing-impaired and forced are subtitle concerns (the subtitles design).
- **`IndexerFlags`** are canonical:
  - `DownloadFactor float` and `UploadFactor float` (1.0 = normal). Freeleech, halfleech, Freeleech25/75 and autobrr's `freeleech_percent = (1-DF)*100` are views over these.
  - `Internal`, `Scene`, `Nuked`, `HasSubtitles`, `Golden`, `Approved`.
  - Open `TrackerTags []string`.
  - The dead AHD flags are dropped. autobrr's two push bitmaps and the per-app enums disappear. Acquisition normalises Torznab `downloadvolumefactor`/`uploadvolumefactor`, native APIs and IRC `freeleech`/`freeleechPercent` into this struct.

### 3.8 Condition engine (`cond/`)

**One engine** serves custom formats, profile gates, autograb rules, the PCD testing UI and import-time CF scoring of files.

**Condition = `{Type, Negate, Required, Params}`.** The types are the union of all sources:

| Class | Types | Origin |
|---|---|---|
| Text pattern (regex, or wildcard list for autobrr) | `release_title` (title or filename), `release_group`, `edition`, `description`, `uploader`, `category`, `tracker_tags`, `record_label` | Arr, PCD, autobrr |
| Enum set | `source`, `resolution`, `quality_modifier`, `release_type` (single/multi/season pack), `language` (+`except`, `Original`), `indexer_flag`, `codec`, `hdr`, `audio`, `container`, `announce_type`, `origin`, `media` (CD/WEB/Vinyl), `audio_format`, `bit_depth`, `quality` (canonical ID) | Arr, PCD, autobrr |
| Numeric range | `size` (`min < x <= max`, whole release), `year`, `freeleech_percent`, `seeders`, `leechers`, `log_score`, `age` | Arr, autobrr |
| Boolean | `has_log`, `has_cue`, `scene`, `hybrid`, `library_match` | autobrr |
| Source scope | `indexer` (ids or tags), `protocol` | Sonarr release-profile scoping, autobrr `indexers` |
| External (rules only) | `external_exec`, `external_webhook` (async, timeout, `on_error`) | autobrr |

**Semantics:**
- Exactly §1.4 for every consumer.
- An autobrr filter field becomes one condition with a value list (OR inside the condition) and `required=true`; `except_*` fields become `negate+required`. The CF grouping rule then reproduces autobrr's AND-across-fields.
- There is no second semantics.

**Facts and unknowns.**
- Each type declares the fact it reads. Facts carry `Known` separately from the value.
- Empty-but-known strings behave as in the Arr (a regex on an empty group is false, so its negation is true), which preserves Dictionarry results.
- A **genuinely unknown** fact evaluates to `Unknown`. This covers size missing from an announce, uploader not announced, or freeleech unknown from a plain RSS feed. `Unknown` is never satisfied in either polarity and is reported as such.
- **Lazy enrichment.** When a rule or profile could still pass *only* if an unknown fact resolves favourably, the engine requests enrichment through an interface the acquisition layer implements: a tracker API or .torrent metainfo, as in autobrr `filter/service.go:731`. It then re-evaluates. This keeps autobrr's "download the .torrent only when needed".

**Compilation.**
- Patterns are compiled once per PCD generation. D1 normalisation (`\p{IsBlock}` → ranges) is applied, then regexp2 with `IgnoreCase` and a per-match **timeout of 100 ms**.
- Patterns are de-duplicated by text across all CFs; Dictionarry shares regexes by name.
- Each compiled pattern gets a **literal prefilter**: a mandatory case-folded literal extracted from the pattern, such as `GROUP` in the `(?<=^|[\s.-])GROUP\b` idiom. It is checked with `strings.Contains` before running regexp2.
- A pattern that fails to compile disables its CF and raises a health issue (D7 alerts).
- A timeout yields `Unknown`, increments a metric, and marks the pattern "slow" in the UI.

**Output.** A `MatchSet` bitset over the compiled CF index, plus an optional `Trace` (per CF → per group → per condition: pass/fail/unknown, with the matched span).

### 3.9 Profiles and scoring per slot (D4)

- A **slot** (defined by the library design) references one quality profile. The profile is materialised from the PCD store (§3.17) and compiled into:

  ```go
  type CompiledProfile struct {
      Family        quality.Family
      Rank          []quality.ID // with group ids: equal rank within a group
      Allowed       bitset
      Cutoff        quality.ID
      UpgradeAllowed bool
      Scores        []int        // indexed by compiled CF index, kind-resolved
      MinScore, CutoffScore, MinIncrement int
      Language      *LangGate    // PCD quality_profile_languages (simple = must include; Original/Any)
      Sizes         *SizeTable
      Delay         *DelayPolicy // §3.14
      Propers       ProperPolicy // PreferAndUpgrade | DoNotUpgrade | DoNotPrefer
  }
  ```

- **The score is `dot(CF MatchSet, Scores)`.** CF matching (stage 3) is shared by all slots of an item. Only the dot product is per slot, which costs microseconds.
- **Release profiles are dropped.** "Must not contain" is the standard score idiom (large negative score, min score ≥ 0), and explanations name the offending format (`CFBelowMinimum{dominant: "x265 (HD)", -10000}`). "Must contain" is a required-condition CF with a positive score and a raised minimum. Release-profile indexer scoping becomes the `indexer` condition type.
- **Radarr's HardcodeSubs setting** becomes a built-in default CF.

### 3.10 Incumbents and upgrade

- **`SlotState`** is an in-memory, event-maintained map `(slot, unit) → []Incumbent`. It is updated by `FileImported/Deleted` and `GrabStarted/Finished/Failed`, and reloaded from the DB at start.
- An incumbent is either:
  - **File**: stored quality ID, revision, group, `DateAdded`, and the **stored `Facts` digest** (attributes, languages, flags); or
  - **InFlight**: a grab whose tracked download is not in a terminal state.
- **CF score re-evaluation.** An incumbent's CF score is recomputed against the *current* compiled profile from its stored attributes, because CF definitions change. This uses the same engine with no re-parse of history strings.
- **The history heuristic disappears.** Tracked downloads are authoritative, so "InFlight" is exact. The 12 h window and the CDH-off branch are replaced. A grab that loses its client item is failed by the download lifecycle and stops being an incumbent.
- **Rule:** for each unit covered by the candidate, the candidate must beat **every** incumbent of that unit:
  - `CutoffNotMet(incumbent)`, else `CutoffMet{incumbent kind}`;
  - then `Upgrade(p, incumbent, candidate)` as §1.6, verbatim, as one function returning a reason enum.
  - A FailedPending in-flight item is skipped (§1.6).
- **Multi-unit releases:**
  - A season pack (or discography, or TPB) is accepted under the `SeasonPackUpgrade` policy (Any/All/Threshold): missing units count as upgradable.
  - Otherwise every unit must pass.
  - `SameEpisodes`: a release covering only part of an existing multi-episode file is rejected.
- **Propers, repacks and anime versions:** see §1.7. A revision upgrade that is a REPACK or version bump needs `group == incumbent.group`, both known. In RSS/IRC (not search) a proper for a file older than 7 days is rejected. Under `DoNotUpgrade` a revision-only upgrade is rejected; under `DoNotPrefer` revisions are ignored.

### 3.11 Check catalogue (every reference spec, placed once)

| Stage | Checks (reason codes) | Replaces |
|---|---|---|
| 0 Intake (release only) | Retention, MinimumAge, MaximumSize (global), ProtocolDisabled, IndexerBackoff, TorrentMinSeeders (per indexer), Obfuscated, Sample, RawDisk (unless the profile allows BR-DISK) | Retention/MinimumAge/MaximumSize/BlockedIndexer/TorrentSeeding/NotSample/RawDisk specs |
| 2 Match | UnknownItem, MatchesOtherItem, UnknownUnits, AmbiguousMatch, SearchMismatch, SplitEpisode, MultiSeason, SceneMapping | Search/*, SceneMapping, SplitEpisode, MultiSeason, DownloadDecisionMaker branches |
| 3 Item gates | Blocklisted (per item container), NotMonitored (unit or slot; skipped for interactive), AlreadyImported (same download id), AirDate restriction (TV), FullSeasonNotAired, EarlyRelease / DiscographyUnreleased / YearMismatch (music), SeasonPackOnly | Blocklist, MonitoredEpisode/Movie, DeletedEpisodeFile, AlreadyImported, AirDate, FullSeason, Lidarr specs |
| 5a Profile | QualityFamilyMismatch, QualityNotAllowed, LanguageNotWanted, CFBelowMinimum, SizeOutOfRange (per quality × runtime), FreeSpace | QualityAllowedByProfile, Language, CustomFormatAllowedByProfile, AcceptableSize, FreeSpace, HardcodeSubs, ReleaseRestrictions, RequiredIndexerFlags |
| 5b Incumbent | CutoffMet{File\|InFlight}, BetterQuality, BetterRevision, UpgradesNotAllowed, QualityCutoff, CFScore, CFCutoff, CFMinIncrement, PackBelowThreshold, SameEpisodes, RepackGroupMismatch/Unknown, ProperForOldFile, PropersDisabled | UpgradeDisk, Upgradable, UpgradeAllowed, Queue, History, Proper, Repack, AnimeVersionUpgrade, SameEpisodes(Grab) |
| 5c Timing (Defer) | Delay{until}, Availability{until} (movies, `IsAvailable` + delay), ClientUnavailable (from commit) | Delay, Pending, Availability |
| 7 Commit | IncumbentChanged (lost the race), RateLimited (acquisition budget), ReleaseUnavailable | ProcessDownloadDecisions exceptions |

Three changes from the references:
- **Availability returns `Defer` until the availability date, not a permanent rejection** (Radarr `AvailabilitySpecification` is `Permanent`). Search and RSS both re-see the release, so the outcome is equivalent, and the explanation is truthful ("available on 2026-10-12").
- `MonitoredEpisode` and the like are skipped for interactive search, as today.
- Blocklist and history checks of *library-less* grabs use the rule's dedupe identity (§3.18).

### 3.12 Explainability

- **Every check runs, and every outcome is recorded as a `Finding`.** Stage 5 has no I/O (§3.1), so the Arr "first tier wins" short-circuit is unnecessary. The single exception is **Stage 0/2 early exit on the hot path** for releases that can never be interesting (no kind, unknown item, retention). Those carry exactly their one reason.
- **Explain mode** (`explain=true`: interactive search, the "why wasn't X grabbed" view, profile testing) adds pass findings and the CF `Trace`. It is deterministic given the same facts and snapshot generation.
- **Persistence.**
  - Decisions for **matched** releases are stored as rows in `release_decision (release_id, target, verdict, codes[], params jsonb, rank, snapshot_gen, created_at)`, kept 7 days by default.
  - Unmatched releases only increment counters by reason code (D7 metrics). This follows autobrr, which stores only matched releases.
  - Codes are a stable enum; the UI localises them.
- **Surfaces.** Interactive search shows each release with reasons and its rank relative to the winner ("lost to #1 on CF score 150 < 200"). The item page answers "why not grabbed" from `release_decision` for its units. The PCD/profile testing UI (Profilarr's UX, D6) calls `evaluate.Simulate(title, profile, fakeItem?)`.

### 3.13 Ranking and selection

- **One comparer engine.** Each kind supplies its **ordered key list as data**, reproducing §1.9 exactly:

  ```
  tv:    Quality, CFScore, Protocol, Coverage(pack, count±anime), FirstUnit, IndexerPriority, IndexerFlags*, Peers, Age, Size
  movie: Quality, CFScore, Protocol, IndexerPriority, IndexerFlags*, Peers, Age, Size          (adult movie/scene: same)
  music: Quality, CFScore, YearMatch, Protocol, IndexerPriority, Peers, Coverage(album count), Age, Size
  book/comic: Quality, CFScore, Protocol, Coverage, IndexerPriority, Peers, Age, Size
  * IndexerFlags only when "prefer indexer flags" is on (Radarr semantics; off by default, so TV stays identical)
  ```

- Keys are **precomputed into a `RankKey` tuple** per candidate, so sorting is cheap and the explanation can name the first differing key.
- **Selection** runs per target, greedily in rank order:
  - A candidate whose unit set intersects an already-selected one is not grabbed; it becomes a **Fallback** hold (§3.14) so a failed grab can be replaced without a new search.
  - Deferred candidates hold their units against lower-ranked accepted ones, as Sonarr treats `TemporarilyRejected` in `ProcessDownloadDecisions.cs:61-65`. The best-ranked release for the units waits; a lower one does not jump the queue.
- **Multiple slots:**
  - Each slot selects independently.
  - When the same release wins in several slots of one item, **one grab with several targets** is emitted (see §6 Q1).
  - A release rejected for one slot (4K) and accepted for another (1080p) is normal and explained per slot.

### 3.14 Delay and pending model (persisted, one mechanism)

- **`DelayPolicy`** is PCD `delay_profiles` shaped:
  - `PreferredProtocol`;
  - per-protocol `Delay` as a **duration with seconds granularity** (covers the Arr minutes and autobrr seconds);
  - `OnlyProtocol`;
  - `BypassIfHighestQuality`;
  - `BypassIfScoreAtLeast *int`.
  - It is resolved per candidate from the profile, and a slot override is allowed (§6 Q3). Autograb rules carry their own.
- **Hold window.** Per `(target, unit)`: `window_start = min(PublishedAt, FirstSeenAt)` over acceptable candidates for that unit. That is the Arr "oldest pending release" rule, with FirstSeen covering IRC and releases with no date.
  - A candidate becomes releasable when `now ≥ window_start + delay` for **any** of its units (Sonarr's "any overlapping pending release" semantics).
  - The bypass rules of §1.8 apply verbatim.
- **`pending_grab` table**: `(id, target, unit_ids[], release snapshot, parsed digest, reason ∈ {Delay, ClientUnavailable, Fallback}, window_start, due_at, created_at)`. Unique on `(target, release_id)`.
- **Timers, not RSS ticks.** On Defer, a durable job `pending.Resolve(target, units)` is scheduled at `due_at` in the D14 job queue. When it fires, it **re-evaluates every hold for those units against the current snapshot** (incumbents may have changed, e.g. a better file was imported), ranks them and commits the winner.
  - Delay precision becomes seconds, not the RSS interval.
  - It survives restarts and node failover.
- **Invalidation.** Profile, slot or monitoring changes, and imports touching the units, enqueue the same resolve job. Holds that become rejected are deleted, with a `release_decision` row noting why (Sonarr's `RemoveRejected` on RSS completion).
- **ClientUnavailable** holds retry with backoff. **Fallback** holds expire after a TTL (default 24 h) or when the unit's cutoff is met.

### 3.15 Blocklist and history

- **Blocklist** is scoped to the item container (series/movie/album/book/scene/volume), with a global scope for library-less rules. The key is the info hash, or `(indexer, normalised title, published_at)` for Usenet and DDL. It is checked in stage 3. Entries store the `Facts` digest, so the blocklist UI can show quality and CFs without re-parsing (`CustomFormatCalculationService.cs:61-85`).
- **History** is written by the lifecycle for grabs. Evaluation does not read history; it reads `SlotState` (§3.10).

### 3.16 Commit

A `Decision{Accept}` becomes a grab through `grab.Reserve(tx)`:
1. Lock the `(slot, unit)` rows (`SELECT … FOR UPDATE`; SQLite serialises writers).
2. Check `slot_unit.version` against the snapshot version the decision used.
3. Insert the grab with target(s).
4. Bump the versions.
5. Commit, then hand the grab to the download lifecycle.

On a version conflict, re-evaluate once against fresh state. This removes the double-grab race between IRC, RSS and search paths, and between HA nodes.

### 3.17 PCD at evaluation time (interface to the profiles design)

- The PCD engine (another design) materialises profiles, CFs, regexes, quality definitions and delay profiles into the monolith's tables and publishes `ProfilesChanged{generation}`. Evaluation depends on a read interface only:

  ```go
  type ProfileSource interface {
      Snapshot(ctx) (ProfileSnapshot, error)  // generation, profiles, CFs, conditions, regexes, quality defs, delay policies
  }
  ```

- On `ProfilesChanged`, `cond.Compile(snapshot)` builds `CompiledRules{gen}` off the hot path and swaps it atomically. In-flight evaluations finish on the old generation, and the decision records `snapshot_gen`.
- **Kind scoping**:
  - A PCD condition's `arr_type` (`all|radarr|sonarr`) and a score row's `arr_type` map to kinds as `radarr→movie` (+ adult movies), `sonarr→tv`, `all→every kind`.
  - Conditions for types invalid in a kind are dropped for that kind, as in Profilarr's `filterConditionsForArrType` but data-driven.
  - Other kinds (music, book, comic, adult scene) use `all` rows (§6 I2).
- **Enum mapping** happens once at compile: PCD source/resolution/modifier/flag/language *names* → canonical IDs. There are no per-app integer ids.
- **Built-in CF tests** (`custom_format_tests`) run through the same engine in CI and in the UI. The Dictionarry test suite is the acceptance test for `cond/` (§1.4).
- **Only reachable CFs are compiled for the hot path**: those scored non-zero in at least one profile used by an active slot or rule for that kind. The testing UI compiles on demand.

### 3.18 Autograb rules (library-less; later phase, D16)

- An autograb rule is **a profile without an item**:

  ```go
  type Rule struct {
      Scope      Scope          // indexers, announce types, kinds, categories
      Conditions []cond.Condition // compiled with the same engine; autobrr fields map 1:1 (§3.8)
      Profile    *ProfileRef    // optional: quality ranking/scores for dedupe upgrades
      Identity   IdentityPolicy // dedupe key from Structure: title+S/E | title+year | artist+album | …
      Upgrade    UpgradePolicy  // RejectDuplicates | ProperSameGroupOnly | ProfileUpgrade | NewerUnitsOnly (smart episode)
      Limits     []Limit        // max grabs per window (fixed/rolling), per rule; per-indexer budgets live in acquisition
      Delay      *DelayPolicy
      External   []cond.External // run last, async, timeout
      Action     ActionProfileRef // client + per-grab options (download lifecycle)
      Priority   int            // first accepting rule wins (autobrr semantics)
  }
  ```

- **Dedupe replaces autobrr's duplicate profiles.** The incumbents of a rule target are prior grabs by that rule with the same `Identity` key. The upgrade policy decides, and `ProfileUpgrade` reuses §3.10 verbatim. "Smart episode" is `NewerUnitsOnly`.
- **Library mode needs no rules at all.** IRC releases simply enter the normal pipeline. autobrr's `shows`/`years`/list fields are replaced by the library.
- **Library overlap.** By default a rule does not fire when a library slot accepted the same release in the same evaluation (§6 Q2).
- **Findings** are the same `Finding` type. autobrr's `{field, got, want}` maps to `Params`.

### 3.19 Performance and the IRC hot path

- **Immutable snapshots behind atomic pointers**: `CompiledRules`, `LibraryIndex`, `SlotState` (copy-on-write per container shard) and `ParseCache`. The evaluator takes no locks and does no DB reads before commit.
- **Cost per release.**
  - Parse: cached, or roughly tens of µs on a miss (regexp2).
  - Match: map lookups.
  - CF: only reachable patterns run; literal prefilters skip most regexp2 calls. Evaluation order within a CF is required conditions first, which short-circuits the CF on the first failed required condition when not in explain mode.
  - Per slot: a dot product and a few comparisons.
  - Target: **p99 < 5 ms** from announce to decision, excluding enrichment and the commit transaction. Measured by D7 histograms (`eval_parse_seconds`, `eval_match_seconds`, `eval_cf_seconds`, `eval_total_seconds`) and counters (`eval_rejections_total{code}`, `regex_timeouts_total{pattern}`).
- **Bounded worst case.** regexp2's timeout is coarse (~130 ms for 100 ms). A per-release deadline (default 250 ms for the whole CF stage) marks the remaining conditions `Unknown` rather than stalling the IRC consumer. Patterns that time out repeatedly are flagged.
- **Batches.** RSS and search result sets are evaluated by a bounded worker pool. Ranking and selection run after all candidates are in.
- **HA (D14).** IRC and RSS run on the leader. Search runs on any node. Snapshots are rebuilt per node from `LISTEN/NOTIFY` events. The commit transaction (§3.16) makes decisions safe across nodes.

---

## 4. Walkthroughs

### 4.1 Season pack vs single episodes (TV, RSS)

Setup:
- Series "Show" (TVDB 123), one slot "HD": profile ranks `[WEBDL-1080p ≈ WEBRip-1080p] < Bluray-1080p`; cutoff Bluray-1080p; upgrades on; min score 0; CF cutoff 200.
- Season 1 has 10 aired episodes. E01–E06 are on disk as WEBDL-1080p, score 100. E07–E10 are missing.
- `SeasonPackUpgrade=Threshold 75%`. Delay 0.

RSS brings:
- A: `Show.S01.1080p.BluRay.x264-GRP` (pack), score 150;
- B: `Show.S01E07.1080p.WEB-DL.DDP5.1.H.264-XYZ`, score 100;
- C: `Show.S01E08.1080p.BluRay.x264-ABC`, score 200.

1. **Parse.** A = TV, seasons [1], FullSeason. B = S01E07. C = S01E08. Attributes give Bluray/1080p, Web/1080p and Bluray/1080p, so A and C are Bluray-1080p and B is WEBDL-1080p.
2. **Match.** Clean title "show", then TVDB 123. A → units E01–E10 (FullPack). B → E07. C → E08.
3. **Facts.** CF sets are computed once per release. Languages fall back to the series' original language.
4. **Evaluate (slot HD).**
   - A: every unit's incumbent is either a file (WEBDL-1080p) or missing. For E01–E06, `Upgrade` returns "better quality & quality cutoff not met" → upgrade. E07–E10 are missing and count as upgradable. 10/10 ≥ 75% → **Accept**.
   - B: E07 is missing → **Accept**.
   - C: E08 is missing → **Accept**.
5. **Rank.** Quality first: A = C (Bluray-1080p) > B. CF: C 200 > A 150. So the order is **C, A, B**.
6. **Select.**
   - C is grabbed (E08).
   - A intersects E08 → **Fallback** hold (`SupersededBy{C}`).
   - B (E07) doesn't intersect → grabbed.

   This is Sonarr's greedy behaviour, kept as an invariant: two single grabs; the pack is kept as a fallback.
7. **Variation.** Without C, A ranks first on quality and is grabbed. B intersects E07 → Fallback. Next RSS: the in-flight pack is E07's incumbent at Bluray-1080p, so B gets `CutoffMet{InFlight}`.
8. **Explain.** B's row in interactive search reads: "Accepted · lost to A on Quality (Bluray-1080p > WEBDL-1080p)".

### 4.2 One movie release against 4K and 1080p slots

Setup:
- Movie M (TMDb 555), runtime 130 min, released.
- Slot **4K**: UHD profile `[WEBDL-2160p] < Bluray-2160p < Remux-2160p`; cutoff Remux-2160p; CFs "DV HDR10" +1500, "TrueHD Atmos" +500; min 0; delay policy: preferred Usenet, torrent delay 60 min, bypass-if-highest-quality on. Incumbent: a WEBDL-2160p file, score 1500.
- Slot **1080p**: HD profile `WEBDL-1080p < Bluray-1080p`; CF "x265 (HD)" −10000; incumbent: none.

Release R (torrent, 58 GB, from RSS): `Movie.2023.2160p.UHD.BluRay.REMUX.DV.HDR.HEVC.TrueHD.Atmos-FraMeSToR`, indexer TMDb 555.

1. **Parse.** Movie structure, year 2023. Axes Bluray × 2160 × REMUX → **Remux-2160p**. HEVC; HDR [DV, HDR10]; audio TrueHD Atmos; group FraMeSToR.
2. **Match.** The indexer TMDb id → M (method ExternalID).
3. **Facts.** The CF MatchSet (shared by both slots) includes "DV HDR10", "TrueHD Atmos" and "Remux Tier 01"; "x265 (HD)" does not match because its resolution condition is required and fails.
4. **Slot 4K.**
   - Quality allowed. Score 2000 ≥ 0. Size 58 GB / 130 min ≈ 446 MB/min, within the Remux-2160p limits.
   - Incumbent WEBDL-2160p: cutoff not met, better quality → **upgrade**.
   - Timing: the torrent is not the preferred protocol, so bypass-if-highest doesn't apply (it needs the preferred protocol). Window start = R's publish time → **Defer until publish+60 min**.
   - A `pending_grab` row and a timer job are created.
   - If a Usenet Remux-2160p arrives within the hour, it is accepted at once (bypass). It then ranks above R on Protocol, R's hold resolves to `SupersededBy`, and the timer job finds nothing to grab.
5. **Slot 1080p.** `QualityNotAllowed{Remux-2160p}` → **Reject**, with no other checks needed. With `explain=true` the CF findings are still listed.
6. **Explain (item page).** "4K: waiting until 14:32 (torrent delay 60 min; preferred protocol Usenet). 1080p: quality Remux-2160p not in profile HD."
7. **Variation.** Release `Movie.2023.1080p.BluRay.x264-GRP` is `QualityNotAllowed` in slot 4K and **Accept** in slot 1080p, as a separate grab targeted at slot 1080p. If both profiles allowed Bluray-1080p and both selected it, **one grab with targets {4K, 1080p}** results (§6 Q1).

### 4.3 IRC announce with freeleech (library match; later, a library-less rule)

Announce on tracker T (IRC ingest per the acquisition design) →

```
Release{Source: IRC, Protocol: Torrent, Title: "Show.S02E05.1080p.WEB.h264-GRP",
        Categories: [TV-HD], Size: 0 (not announced),
        Flags: {DownloadFactor: 0.0}, Announce: {Type: NEW, Uploader: "bot", Origin: P2P},
        PublishedAt = FirstSeenAt = now}
```

1. **Intake.** Retention, age, protocol and indexer health pass. MinimumAge is 0.
2. **Parse.** Cache miss. TV via the category hint; S02E05; WEB/1080p → WEBDL-1080p (Sonarr maps `WEB` + h264 to WEBDL).
3. **Match.** Clean title → series "Show" → S02E05 (a unit in slot HD, monitored).
4. **Facts.** CF "Freeleech" (`indexer_flag = freeleech`) matches from `DownloadFactor 0`. CF "Size < 2 GB" has `size` **Unknown**.
5. **Slot HD.** The profile scores Freeleech +25 and doesn't score a size CF, so the unknown size doesn't affect the outcome and no enrichment is requested. The **profile size gate** does need size, though: `SizeOutOfRange` needs the fact.
   - The engine asks acquisition to enrich (tracker API or .torrent metainfo).
   - The metainfo says 1.4 GB, which is within WEBDL-1080p limits × runtime.
   - No incumbent → Accept. Torrent delay 0 → commit.
   - `grab.Reserve` locks E05, sees version 0 and inserts the grab.
   - The same .torrent bytes fetched for enrichment are passed to the lifecycle, so it isn't downloaded twice.
   - Latency: about 1–3 ms of evaluation plus the enrichment fetch.
6. **Later phase: racing rule** "T freeleech NEW < 10 GB", scope T and announce type NEW:
   - conditions `freeleech_percent in [100]` (required), `size <= 10 GB` (required);
   - limit 20 per day;
   - action "racing client".
   - Evaluated from the same Facts (no re-parse). The size became known via enrichment → Accept.
   - Because the library slot already accepted this release in the same evaluation, the rule records `SkippedLibraryGrab` by default (§6 Q2).
   - With a non-library announce (e.g. a game) the rule grabs on its own. The dedupe identity is the rls-normalised title.

### 4.4 Ambiguous adult scene date match (phase 3)

Setup (D10 model):
- Studio "Brazzlers" with alias "BZ".
- Two scenes on 2024-03-15:
  - S1 "Morning Session" (Jane Doe, John Roe);
  - S2 "Evening Session" (Jane Doe, Mary Poe).
- Both are monitored in slot "Default".

Release: `BZ.24.03.15.Jane.Doe.XXX.1080p.MP4-GRP`.

1. **Parse** (adult sub-parser). Studio "BZ", date 2024-03-15 (YY.MM.DD; not in the future), tokens "Jane Doe". No code, no StashID.
2. **Match.**
   - StashID: none. Code: none. Studio via alias → Brazzlers. Same-date scenes: {S1, S2}.
   - Graded levels (`MovieParseMatchType` order): Title fails for both. **PerformersTitle** (tokens == a performer name) matches S1 and S2.
   - No stricter level separates them.
   - The optional fuzzy tier (tokens vs clean titles) scores both low, below threshold. Even if both were ≥ threshold, a tie within the margin is ambiguous.
   - → **`AmbiguousMatch{tier: PerformersTitle, candidates: [S1, S2]}`**, verdict Reject.
3. **Search context.** An interactive search for S1 still rejects (`AmbiguousMatch`; the search guard can't prove the release is S1). The user can **grab with an explicit target**, which is an interactive override recorded in history. Automatic search never guesses.
4. **Variation with a code.** `BZ.24.03.15.Jane.Doe.BZ-1234.XXX.1080p` → the code index returns only S1 (D10.1) → **Match S1 (ProductCode)**. Normal evaluation follows. The download-lifecycle and import fallback use the same code lookup.
5. **Explain.** "No unique match: 2 scenes on 2024-03-15 for Brazzlers match performer 'Jane Doe' (S1 Morning Session, S2 Evening Session). Add a scene code or use interactive grab."

---

## 5. Simplifications compared with the references

| Reference behaviour | Here | Why it is safe |
|---|---|---|
| Spec tiers; stop at the first tier with rejections (`DownloadDecisionMaker.cs:190-200`) | All checks run in memory; all reasons shown | Tiers existed to avoid DB queries; `SlotState` removes them |
| Upgrade checked separately against disk, queue and history, each re-parsing stored titles | One incumbent list (File \| InFlight) per unit; stored facts; one `Upgrade()` | Same function and order (§1.6); tracked downloads are authoritative |
| History 12 h window and CDH-off branch | Gone | InFlight state is exact |
| Release profiles next to CFs | CFs only (+ `indexer` condition) | The score idiom already expresses must/must-not; the explanation names the CF |
| autobrr filters (80 columns), duplicate profiles, smart episode, `shows` lists, `release/push` | Library mode: none. Library-less: a rule = conditions (same engine) + identity + limits + action | Same AND/OR semantics via required conditions; the upgrade logic is the real one |
| Profilarr evaluator, C# parser sidecar, match caches | Deleted; the testing UI calls `cond`/`evaluate` in-process | One engine, one parser |
| Delay in three places; resolved at the next RSS sync; autobrr delay lost on restart | One `DelayPolicy`; persisted holds; timer jobs at `due_at` | Same bypass and window semantics; better precision |
| Per-app enum ids (qualities, languages, flags) and TRaSH/PCD id mapping | Canonical names/codes; mapped once at compile | D9 |
| Three comparers | One engine, per-kind key lists as data | Orders reproduced exactly |
| Availability as a permanent RSS rejection | `Defer{until}` | The same grab happens; the explanation is truthful |
| Eros fuzzy-first, top score wins | Fuzzy last, threshold + margin, ambiguity rejects | D10.2 |
| Multi-instance 4K/1080p (import lists, Seerr `is4k`) | Per-slot candidates from one evaluation | D4 |
| Language aggregation + profile language + language CFs | Aggregation stays in Facts (item-dependent); gate from PCD profile languages; CFs unchanged | Nothing lost; one source for languages |
| Bookshelf one ladder for ebook + audio | Profile bound to one quality family | Structural |

---

## 6. Issues with decisions, and open questions

### Issues with decisions (flagged, not deviated from)

- **I1 — D1 vs D1a: which parser is authoritative for attributes.**
  - D1 calls Dictionarry "the gold standard for regexes that match attributes in release names". D1a makes the Arr cascades and corpora the reference for structure.
  - It's unclear whether *attribute* detection (source, resolution, modifier, codec, HDR) must follow the Arr `QualityParser` tests or Dictionarry's regexes.
  - This design assumes: **attributes = Arr `QualityParser`/`LanguageParser` behaviour (golden tests); CF conditions = Dictionarry patterns over the raw title.**
  - Dictionarry CFs that use `source`/`resolution`/`quality_modifier` *condition types* depend on the parser's attributes, so a disagreement surfaces as a CF-test failure. Please confirm, and decide which side wins when a Dictionarry CF test contradicts an Arr parser test.
- **I2 — D9b (keep the PCD format) vs D2 (all kinds from day one).**
  - PCD scopes conditions and scores by `arr_type ∈ {all, radarr, sonarr}` (`0.schema.sql:104`; `quality_profile_custom_formats.arr_type`). There is no way to express music-, book-, comic- or adult-scene-specific scores or conditions without a schema change.
  - Proposal: map `radarr→movie`, `sonarr→tv`, `all→*`, and treat other kinds as `all` until the PCD schema gains a `kind` column. That needs an upstream schema proposal to Dictionarry-Hub/schema or a local extension that breaks round-tripping.
- **I3 — D9b vs D16/§3.8: new condition types can't be published.**
  - Announce-metadata condition types (uploader, freeleech %, announce type, origin, category, tags, log score, indexer, protocol) and autograb rules have no PCD representation.
  - Either they stay local-only (not publishable to git) or the PCD schema is extended. The first is assumed.
- **I4 — D9 delay profiles vs D4 slots.**
  - PCD `delay_profiles` have no tags and the Arr only exposes the default one. Sonarr's tag-scoped delay profiles have no home under D9.
  - This design attaches `DelayPolicy` to the quality profile, with an optional slot override. A user who wants the same profile with different delays needs the override. Confirm this placement; it isn't covered by any decision.
- **I5 — D1 timeout granularity on the IRC path.** D1's 100 ms per-match timeout (≈130 ms real) is too coarse for a latency-sensitive announce path with hundreds of patterns. This design adds a per-release CF deadline and literal prefilters (§3.19). That goes beyond D1's wording, but D1 says other regex questions are raised separately; this is that question.
- **I6 — D3c scene mappings.** The "matches another series" guard (§1.2) depends on scene-mapping coverage. With bundled/community data instead of `services.sonarr.tv`, coverage will be lower at launch. More releases then end up `UnknownItem` instead of wrongly matched, which is the safe direction, but the risk needs owning in the metadata design.

### Open questions (for other designs or the user)

- **Q1 (download/import).** When one release wins in several slots of an item (4K and 1080p profiles both allow it), is it one download with several import targets (hardlink/copy into each slot's root folder)? Or is it grabbed for the higher-priority slot only? This design emits one grab with a target set.
- **Q2 (autograb, later).** When a library slot and a racing rule both accept the same announce, should the rule skip (default here), share the one download (cross-seed into the rule's client category), or grab twice?
- **Q3.** Where do delay policies live (I4): on the profile, the slot, or tag-scoped as in Sonarr?
- **Q4.** Should greedy selection (§4.1: two singles win over a pack that covers eight more missing episodes) be kept as-is (Sonarr invariant) or become coverage-aware? This design keeps Sonarr's behaviour; a coverage-aware selector would be a user-visible change.
- **Q5 (library).** Does monitoring live on the slot, the unit, or both? Stage 3 `NotMonitored` needs one answer. Is `SeasonPackUpgrade` (Any/All/Threshold) a profile setting, a slot setting, or global as in Sonarr?
- **Q6 (acquisition).** The enrichment interface (size, uploader, record label via tracker API or .torrent) and reuse of the fetched .torrent at grab time need an agreed contract.
- **Q7 (profiles).** Quality-size definitions for Audio, Text and Archive families aren't in PCD (`radarr_/sonarr_quality_definitions` only). They need local defaults or a PCD extension; this is related to I2.
- **Q8 (D8 spike).** The spike should also report per-kind *attribute* agreement between rls and `QualityParser`, not just structure pass rates, because CF results depend on attributes (I1).
