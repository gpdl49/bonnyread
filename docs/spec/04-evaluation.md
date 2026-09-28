# 04. Evaluation

This file specifies everything between a normalised **Release** (emitted by `03-acquisition.md`) and a **grab** (handed to `05-downloads.md`). It covers the release parser package, matching releases to library Items, the per-release facts, the canonical quality model with per-kind axes, languages and indexer flags, the condition engine, per-slot scoring and profile gates, incumbents and upgrade rules, propers and repacks, season packs, delay holds, ranking, coverage-aware selection, one grab feeding several slots, the grab transaction, explainability, the IRC hot path, blocklist checks and autograb rules. The library model is owned by `02-domain-model.md`, profile storage by `07-profiles.md`, and download tracking by `05-downloads.md`. **Phases:** P1 for the parser framework, the TV and movie sub-parsers and the whole decision pipeline; P2 adds the music sub-parser, matcher and quality family; P3 adds book, adult and comic. Autograb rules ship with racing mode in a later phase (D16), but their evaluation path is specified here so the engine is stable from P1.

## 1. Concepts

| Term | Definition |
|---|---|
| **Release** | The value type owned by `03-acquisition.md` (ACQ-060): one downloadable offer from one indexer, with raw title, size (nullable), flags, announce metadata and external ids. Alias: the design sketch calls `origin` "Source". |
| **ParsedRelease** | The pure, profile-independent parse result of a title: a primary **Structure** (kind-specific: TV, movie, music, book, adult, comic), zero or more alternate structures, and **Attributes**. |
| **Attributes** | Quality axes, revision, languages, `MULTI` flag, release group, hash, container, streaming service, editions, hardcoded subs, and the obfuscated/sample/raw-disk flags, each with its detection source (name, extension, media info). |
| **Match** | The resolution of a ParsedRelease to one Title and a set of its Items, with the numbering used, the method (external id, scene mapping, XEM, clean title, alternate title, translation, product code, studio+date level, fuzzy with margin) and whether it is a full pack. Alias: the design calls Items "units" and the Title's entity the "container". |
| **Facts** | Per (release, Item set): Release + ParsedRelease + Match + aggregated languages + runtime sum + canonical quality id + the CF match set. Profile-independent and shared by every slot and rule. |
| **Target** | What a decision is for: exactly one of a **slot target** (an Item set in one slot, i.e. `ItemSlot` rows of one `SlotDefinition`) or a **rule target** (an autograb rule, later phase). Alias: design "slot" means SlotDefinition (definition), TitleSlot (configuration) or ItemSlot (monitoring), per `02-domain-model.md`. |
| **Candidate** | Facts × Target × compiled profile: the unit that stage 5 evaluates. |
| **Decision** | A candidate's verdict — `Accept`, `Reject` or `Defer{until}` — with its findings and precomputed rank key. |
| **Finding** | One typed check outcome: stage, reason code (stable enum, §3.20), outcome (`Pass`, `Fail`, `Defer`, `Unknown`), parameters and an optional condition trace. |
| **Incumbent** (alias **holder**) | Something that already satisfies an ItemSlot: a **File** (a present MediaFile linked to the Item in that slot) or an **InFlight** grab (a DownloadTarget for that ItemSlot whose Download is in a non-terminal phase). The **incumbent list** of an ItemSlot is the one list of all its holders. |
| **SlotState** | The in-memory, event-maintained map `ItemSlot → incumbent list + version`. |
| **LibraryIndex** | The in-memory matching index over the library (titles, alternate titles, external ids, product codes, per-Title Item indices). |
| **CompiledRules** | The compiled conditions, CFs and profiles of one profile-store generation. |
| **Snapshot** | The triple (`CompiledRules`, `LibraryIndex`, `SlotState`) a decision was made against, identified by generation numbers recorded on the decision. |
| **Wanted Item** | An ItemSlot that is `missing` or `cutoff_unmet` per the wanted queries of `02-domain-model.md`. |
| **Gain** of a candidate | The subset of its Items for which it is accepted as a fill (missing) or an upgrade (beats every incumbent). |
| **Hold** | A persisted deferral of a candidate (`PendingGrab`), with reason `Delay`, `ClientUnavailable`, `GrabBudget` or `Fallback`. |
| **Selection component** | Within one slot and one Title, a maximal set of accepted or deferred candidates connected by overlapping Item sets. |
| **Grab** | The commit of a selected release: one Download (owned by `05-downloads.md`) with one DownloadTarget per (Item, slot) it feeds. |
| **Autograb rule** | A library-less rule (later phase): conditions + scope + identity + limits + action, evaluated by the same engine. |

## 2. Scope summary

```
Release ─▶ [0 Intake] ─▶ [1 Parse] ─▶ [2 Match] ─▶ [3 Facts + item gates] ─▶ [4 Candidates per target]
                                                                                   │
        ┌──────────────────────────────────────────────────────────────────────────┘
        ▼
  [5 Evaluate: 5a profile → 5b incumbents → 5c timing] ─▶ Decision{Accept | Reject | Defer}
        │
        ▼
  [6 Rank + coverage-aware selection per slot] ─▶ merge per release ─▶ [7 Grab transaction] ─▶ 05-downloads
        │
        └─ Defer / Fallback ─▶ PendingGrab (persisted) ─▶ resolve job at due_at ─▶ back to 5
```

## 3. Requirements

### 3.1 Pipeline and package structure

**EVL-001** (MUST) Evaluation **MUST** run the stages in the order of §2. Stages 1–3 (parse, match, facts) **MUST** be pure and profile-independent, computed once per release (per matched Item set) and shared by every slot and every rule. _Source: design/evaluation.md §3.1._

**EVL-002** (MUST) Stage 5 **MUST NOT** perform database I/O. Profiles, library state and incumbents **MUST** come from the in-memory snapshot (§3.18). Database state **MUST** be re-validated only in the grab transaction (§3.16). _Source: design/evaluation.md §3.1._

**EVL-003** (MUST) The code **MUST** be organised as these packages, with `parse`, `quality` and `cond` having no dependency on the library or on storage, so that profile testing (`07-profiles.md` §3.17), import re-parsing (`06-import.md`), subtitles (`09-subtitles.md`, D1c) and CI can call them directly:

| Package | Responsibility |
|---|---|
| `parse` | Pure title → ParsedRelease; `parse/attr` (shared attributes), `parse/{tv,movie,music,book,adult,comic}` (structure sub-parsers), `parse/corpus` (golden fixtures), `parse/tokens` (D8 seam) |
| `match` | ParsedRelease × LibraryIndex → Match or match finding |
| `quality` | Canonical quality catalogue, families, axes → quality id; languages; indexer flags |
| `cond` | Condition compile, evaluate, trace |
| `evaluate` | Facts, candidates, profile scoring, upgrade, propers, delay policy, findings |
| `rank` | Per-kind rank keys; coverage-aware selection |
| `pending` | Persisted holds and resolve jobs |
| `grab` | The grab transaction |
| `autograb` | Later phase: rule scope, limits, dedupe identity; reuses `cond` and `evaluate` |

_Source: design/evaluation.md §3.2._

**EVL-004** (MUST) The public parser entry point **MUST** be `parse.Parse(title, hints) → ParsedRelease`, where `hints` carries the search context's kind, the release's `kind_hints` and whether the indexer or category is adult. This signature **MUST NOT** change with the D8 outcome (§3.2.4). _Source: design/evaluation.md §3.4._

### 3.2 Parser

#### 3.2.1 Structure (D1a)

**EVL-005** (MUST) Each structure sub-parser **MUST** port its reference cascade in the reference order, on regexp2 (D1), and return a kind-specific structure:

| Sub-parser | Reference | Structure fields (at least) |
|---|---|---|
| `tv` | Sonarr `Parser.cs` (142 regexes, order load-bearing) | series title, year, seasons, episodes, multi-episode, absolute numbers, air date (+ part), full season, multi-season, partial season, possible special, special, AKA titles |
| `movie` | Radarr `Parser.cs:27-70` | titles (incl. AKA), year (1800–2099), secondary year, editions, IMDb/TMDb id in name |
| `music` | Lidarr `Parser.cs:41-133` | artist, album, year, discography, discography range |
| `book` | Bookshelf title/subtitle split | author, title, subtitle, ISBN/ASIN |
| `adult` | Union of Whisparr v2 date patterns and product-code variants with Whisparr v3's scene patterns, StashID and JAV patterns | studio, date (day/month swap, 2-digit years), product code (the ten v2 variants), StashID, studio episode number, performer tokens |
| `comic` | Kapowarr `backend/base/file_extraction.py` | series, volume number, year, issue number/range, special version |

_Source: D1a; D1c; design/evaluation.md §1.1, §3.4._

**EVL-006** (MUST) Structure results **MUST** equal the reference on the ported golden corpora: the 1,631+ Sonarr `ParserTests` cases, Radarr `ParserTests`, and (in their phases) the Lidarr, Bookshelf, Whisparr v2/v3 and Kapowarr fixtures. These corpora, run as table tests in `parse/corpus`, **MUST** be the acceptance suite. A deliberate deviation **MUST** be listed in the corpus as an expected-difference entry with its reason and an upstream bug reference. _Source: D1a; D8.1; D20._

**EVL-007** (MUST) The parser **MUST** reject hashed or obfuscated titles using the Arr `RejectHashedReleasesRegex` family (Radarr `Parser.cs:73-100` and every fork) by setting `Attributes.Obfuscated`, and the adult sub-parser **MUST** reject a date more than one day in the future. _Source: design/evaluation.md §1.1._

**EVL-008** (MUST) Kind selection **MUST** be: (1) the search context's kind; else (2) the kinds implied by `kind_hints`; else (3) every sub-parser in the fixed order tv, movie, music, book, comic, with adult only when the category or indexer is adult. Every structure that parses **MUST** be kept (primary plus `alternates`); matching chooses the one that resolves to a library Item (EVL-022). Parsing **MUST NOT** reject a title because it resolves to no known kind. _Source: design/evaluation.md §3.4._

#### 3.2.2 Attributes (D1, D20 attribute authority)

**EVL-009** (MUST) `parse/attr` **MUST** be one attribute parser shared by every kind, producing: video axes (source, resolution, modifier, codec, HDR list, bit depth, VR projection), audio axes (codecs, bitrate bucket, bit depth, channels, Atmos, lossless), text format, revision, languages and `MULTI`, release group, hash, container, streaming service, editions, hardcoded subs, and the obfuscated/sample/raw-disk flags, each with its detection source. _Source: design/evaluation.md §3.3._

**EVL-010** (MUST) Attribute detection (source, resolution, modifier, revision, language, group, hash, hardcoded subs) **MUST** equal the ported Sonarr/Radarr `QualityParser`, `LanguageParser` and `ReleaseGroupParser` tests of each fork, which are its acceptance suite. Dictionarry defines custom-format *patterns* only. When a Dictionarry CF test contradicts an Arr parser test, the Arr test **MUST** win and the disagreement **MUST** be filed upstream as a bug; the Dictionarry test is recorded as an expected failure with the upstream reference. _Source: D20 (attribute authority); open-issues.md B4._

**EVL-011** (MUST) Revision semantics **MUST** follow Sonarr `Revision.cs:33-56`: `Real` is compared first, then `Version`; `IsRepack` is not part of the ordering; PROPER and REPACK both bump `Version`; an anime `v2` sets `Version`. _Source: design/evaluation.md §1.1._

**EVL-012** (MUST) Import augmenters (`06-import.md`) **MUST** be able to supply attributes from the file extension or media info and **MUST** record the detection source per axis in the same `Attributes` type. _Source: design/evaluation.md §3.3._

#### 3.2.3 Caching

**EVL-013** (MUST) Parse results **MUST** be cached in a per-node LRU keyed by `(parser_version, title)`. `parser_version` **MUST** change whenever any sub-parser, pattern or attribute table changes; no other invalidation is needed. _Source: design/evaluation.md §3.4._

#### 3.2.4 D8 seam and spike

**EVL-014** (MUST) `parse/tokens` **MUST** define a `Tokeniser` interface giving sub-parsers the normalised title, bracket groups, stripped website prefix/suffix and the attribute spans to exclude from titles. There **MUST** be two implementations during the spike: `cascade` (the ported Arr pre-processing on regexp2) and `rlshybrid` (the autobrr `rls` fork, MIT, for tokens and attributes, with ported cascades for numbering). _Source: D8; design/evaluation.md §3.4._

**EVL-015** (MUST) The golden corpora (EVL-006, EVL-010) **MUST** be ported before the spike, in this order: Sonarr, Radarr, then Lidarr, Bookshelf, Whisparr v2/v3 and Kapowarr. _Source: D8.1._

**EVL-016** (MUST) The spike **MUST** be time-boxed (duration set in `14-roadmap.md`) and **MUST** report, per implementation and per corpus: structure pass rate; **per-kind attribute agreement** with the `QualityParser`/`LanguageParser`/`ReleaseGroupParser` tests for each axis (source, resolution, modifier, codec, HDR, language, group, revision); agreement of CF results on the Dictionarry `custom_format_tests` suite (because CF results depend on attributes); p50/p99 parse latency; and the share of patterns running on RE2 versus regexp2. [OPN-019] _Source: D8.2; open-issues.md Group C; design/evaluation.md §6 Q8._

**EVL-017** (MUST) The decision after the spike **MUST** apply these criteria in order: (1) the chosen implementation must be able to reach 100% of the structure and attribute corpora (EVL-006, EVL-010), with only upstream-filed expected differences; (2) prefer `rlshybrid` only if, at the end of the time box, its per-axis attribute agreement is not lower than `cascade`'s on any axis and its combined corpus pass rate is at least equal; (3) otherwise choose `cascade`. Either way, D1a's reference and acceptance suite apply unchanged, and the unused implementation is removed. [OPN-111] _Source: D8.3; D1a; D20._

### 3.3 Matching releases to Items

#### 3.3.1 LibraryIndex

**EVL-018** (MUST) `match` **MUST** hold an in-memory LibraryIndex, rebuilt incrementally from library events and swapped atomically, containing: `clean_title → [TitleEntry{title, source ∈ primary | alternate | translation | scene_mapping{season} | xem | alias, year}]`; external-id maps (tvdb, tmdb, imdb, stash, tpdb, mbid, isbn, asin, comicvine); a **non-unique** product-code index (D10.1, A14); and per-Title Item indices: (season, episode), absolute number, scene numbering, air date, studio+date, issue number. Alternate titles, translations, scene mappings and studio aliases come from the one `AltTitle` store of `02-domain-model.md`. _Source: design/evaluation.md §3.5; design/library.md §5._

#### 3.3.2 Per-kind lookup order

**EVL-019** (MUST) Matchers **MUST** keep the reference lookup order:

| Kind | Order |
|---|---|
| TV | search-criteria titles → scene-mapping title (+ season scope) → all titles and aliases → title + year → indexer TVDB id → indexer IMDb id (Sonarr `ParsingService.cs:43-223`) |
| TV Items | by numbering mode: standard (with XEM scene → TVDB when the Title's `use_scene_numbering` is set), anime absolute, daily air date (+ part), specials by title only when the structure says "possible special" (`ParsingService.cs:276-330`, `DownloadDecisionMaker.cs:81-89`) |
| Movie | parsed IMDb/TMDb id in the name → indexer IMDb id → indexer TMDb id → title + year (year or secondary year) → title only when no year was parsed; titles include original, alternative and translated titles (Radarr `ParsingService.cs:163-203`, `MovieRepository.cs:134-210`) |
| Music | fuzzy artist and album location inside the title (Lidarr bitap), with aliases, year ±3 |
| Book | ISBN/ASIN → edition directly; otherwise author + title distance matching as in Bookshelf |
| Adult | StashID → product code (non-unique index) → studio (+ aliases) + date → graded levels in Whisparr v3 `MovieParseMatchType` order with month/year widening on `-01` dates → optional fuzzy tier **last**, only over same-date scenes |
| Comic | Kapowarr's filter: title/alias equality, volume number, year range, issue in volume, special-version conflict (a TPB never matches a normal volume) |

_Source: design/evaluation.md §1.2, §3.5; D10._

**EVL-020** (MUST) A release that maps (by scene mapping or alias) to a known external id that is **not** the library Title found **MUST** produce `MatchesOtherTitle`, never a guess (Sonarr `DownloadDecisionMaker.cs:103-114`). _Source: design/evaluation.md §1.2._

**EVL-021** (MUST) A release from a targeted search **MUST** map to the searched Items; otherwise `SearchMismatch` (Sonarr `Specifications/Search/*`, Radarr `Search/MovieSpecification.cs`). _Source: design/evaluation.md §1.2._

#### 3.3.3 The ambiguity rule (all kinds, D10)

**EVL-022** (MUST) One ambiguity rule **MUST** apply to every kind and every method, fuzzy included: walk the kind's tiers in order; within a tier apply that tier's own disambiguators (year, secondary year, season scope, studio); at the **first tier that yields any candidate**, if more than one distinct Title or distinct Item set remains, return `AmbiguousMatch{tier, candidates}`. The matcher **MUST NOT** pick the first, the highest-scoring without margin, or the most recently added result, and **MUST NOT** fall through to a later tier to break the tie. When several structures (EVL-008) resolve, the same rule applies across them. _Source: D10.2; design/evaluation.md §3.5._

**EVL-023** (MUST) A fuzzy tier **MUST** require both a threshold and a margin over the runner-up; a runner-up within the margin is ambiguous. Defaults are the reference constants: adult same-date fuzzy uses a 0–100 ratio with threshold 80 and margin 5 (Whisparr v3 `WhisparrFuzzyTitleMatchingThreshold`, `FuzzyMovieMatchMargin`); music uses Lidarr's `AlbumYearMatchingOptions.TitleFuzzThreshold` and `TitleFuzzGap`; book uses the import match threshold with the ambiguity margin. [OPN-029] _Source: D10.2; design/evaluation.md §3.5; open-issues.md Group C (import)._

**EVL-024** (MUST) A product code that resolves to more than one scene through the non-unique index **MUST** be `AmbiguousMatch{tier: ProductCode}`. _Source: D10.1; A14._

**EVL-025** (MUST) An interactive user **MAY** override an `AmbiguousMatch` or `UnknownItem` by grabbing with an explicit target; the override **MUST** be recorded (actor, chosen Items) on the Download's event log (`05-downloads.md`). Automatic paths **MUST NOT** override. _Source: design/evaluation.md §4.4._

#### 3.3.4 Other match outcomes

**EVL-026** (MUST) Match outcomes other than success **MUST** be findings, not errors: `NotWantedKind`, `UnknownTitle` (no library Title), `MatchesOtherTitle`, `UnknownItem` (Title found, Item not), `AmbiguousMatch`, `SearchMismatch`, `SplitEpisode`, `MultiSeason`, `SceneMappingMismatch`. _Source: design/evaluation.md §3.5, §3.11._

**EVL-027** (MUST) The product-code lookup **MUST** be exposed through the same `match` API with only a code as input, for the completed-download and import fallback of `05-downloads.md` and `06-import.md` (D10.1). _Source: D10.1; design/evaluation.md §3.5._

### 3.4 Facts

**EVL-028** (MUST) For each (release, matched Item set) evaluation **MUST** compute Facts once: the ParsedRelease; the Match; the aggregated languages; the runtime sum over the matched Items; the canonical quality id (§3.5); and the CF match set over the compiled CF index of the Item's kind (§3.6). _Source: design/evaluation.md §3.3._

**EVL-029** (MUST) Language aggregation **MUST** follow Sonarr `AggregateLanguages.cs:25-135`: indexer-declared languages win; languages that appear only inside the episode title are dropped; `MULTI` expands through the indexer's `multi_languages` setting; an unknown language falls back to the Item's original language. _Source: design/evaluation.md §1.2._

**EVL-030** (MUST) Every fact **MUST** carry whether it is known separately from its value. A missing Release `size` (null, ACQ-060), an announce field that was not announced, or a flag a plain feed cannot carry **MUST** be `Unknown`, which is distinct from a known empty value. _Source: design/evaluation.md §3.8._

### 3.5 Canonical quality model

**EVL-031** (MUST) Qualities **MUST** be named points in a per-family axis space, keyed by PCD canonical names where PCD has them (D9; the enum is owned by bonnyread, `07-profiles.md` §3.13):

| Family | Kinds | Axes |
|---|---|---|
| Video | tv, movie, adult | Source (CAM … Bluray) × Resolution × Modifier (NONE, REGIONAL, SCREENER, RAWHD, BRDISK, REMUX), plus a VR flag for adult |
| Audio | music, audiobook | Codec × BitrateBucket × BitDepth (Lidarr's 38 qualities) |
| Text | ebook | Format (EPUB, AZW3, MOBI, PDF, …) |
| Archive | comic | Format (CBZ, CBR, PDF) |

_Source: design/evaluation.md §3.6; A17._

**EVL-032** (MUST) `quality.Derive(axes) → quality id` **MUST** be one table per family. Sonarr `Bluray-1080p Remux` and Radarr `Remux-1080p` **MUST** collapse to one id, and Sonarr `Raw-HD` (source TelevisionRaw) and Radarr `Raw-HD` (modifier RAWHD) **MUST** collapse to one id. The Arr golden tests **MUST** assert `Derive(Parse(title)) == expected`. _Source: design/evaluation.md §3.6._

**EVL-033** (MUST) A quality profile **MUST** be bound to exactly one family. A release of another family **MUST** get `QualityFamilyMismatch`. _Source: design/evaluation.md §3.6._

**EVL-034** (MUST) Size limits **MUST** come from the profile's quality-size definitions (`07-profiles.md`), with per-family units: Video MB per minute × runtime; Audio kbps-equivalent × duration; Text and Archive absolute MB. Non-video definitions come from the extension schema and the project's starter databases (D20). _Source: design/evaluation.md §3.6, §6 Q7; D20._

**EVL-035** (MUST) Languages **MUST** be ISO 639 codes with optional region (`pt-BR`, `es-419`). `Original` and `Any` **MUST** exist only as sentinels in conditions and profiles; the parse result **MAY** be `Unknown`; `MULTI` is a parse flag, not a language. Hearing-impaired and forced are subtitle concerns (`09-subtitles.md`). _Source: design/evaluation.md §3.7._

**EVL-036** (MUST) Indexer flags **MUST** be the canonical `IndexerFlags` of ACQ-061. Freeleech, halfleech, freeleech 25/75 and `freeleech_percent = (1 − download_factor) × 100` **MUST** be views over `download_factor`/`upload_factor`, not stored flags. _Source: design/evaluation.md §3.7; ACQ-061._

### 3.6 Condition engine (`cond`)

**EVL-037** (MUST) One engine **MUST** serve custom formats, profile gates, autograb rules, the PCD testing UI and import-time CF scoring of files. There **MUST NOT** be a second evaluator (no Profilarr evaluator, no C# sidecar). _Source: design/evaluation.md §3.8; design/profiles.md §3.10.1._

**EVL-038** (MUST) A condition **MUST** be `{type, negate, required, params}`. The condition types **MUST** be the union of the Arr, PCD and autobrr types, including announce metadata:

| Class | Types | Origin |
|---|---|---|
| Text pattern (regex; wildcard list for autobrr-origin rules) | `release_title` (title or file name), `release_group`, `edition`, `description`, `uploader`, `category`, `tracker_tags`, `record_label`, `parsed_title` (structure title: show, movie, artist, album; rules only) | Arr, PCD, autobrr |
| Enum set | `source`, `resolution`, `quality_modifier`, `release_type` (single, multi, season pack), `language` (with `except` and `Original`), `indexer_flag`, `codec`, `hdr`, `audio`, `container`, `announce_type`, `origin`, `media` (CD, WEB, Vinyl, …), `audio_format`, `bit_depth`, `quality` (canonical id) | Arr, PCD, autobrr |
| Numeric range | `size` (`min < x <= max`, bytes of the whole release), `year`, `season`, `episode`, `month`, `day` (rules only for the last four), `freeleech_percent`, `seeders`, `leechers`, `log_score`, `age` | Arr, autobrr |
| Boolean | `has_log`, `has_cue`, `scene`, `hybrid`, `library_match` | autobrr |
| Source scope | `indexer` (ids or tags), `protocol` | Sonarr release-profile scoping, autobrr `indexers` |
| External (rules only) | `external_exec`, `external_webhook` (async, timeout, `on_error`) | autobrr |

Storage and publishability of every type is owned by `07-profiles.md` (PCD tables and the extension schema, PRF-067 onward). _Source: design/evaluation.md §3.8; D20._

**EVL-039** (MUST) Custom-format match semantics **MUST** be exactly those of Sonarr `CustomFormatCalculationService.cs:147-168` and `SpecificationMatchesGroup.cs:10`, for every consumer:
1. `negate` inverts a single condition's result before grouping;
2. conditions are grouped by type;
3. a group matches iff every `required` condition passes **and** at least one condition passes, i.e. `!(any(required ∧ ¬pass) ∨ all(¬pass))`;
4. a format matches iff every group matches;
5. matched formats are sorted by name; a profile's score is the sum of its scores for the matched formats.

_Source: design/evaluation.md §1.4._

**EVL-040** (MUST) Condition details that affect results **MUST** be preserved: `release_title` matches the release title **or** the file name; `size` is `min < size <= max` in GB over the whole release, not scaled per episode; regexes are case-insensitive .NET semantics (Unicode-aware `\d`, `\w`, `\b`); language `Original` resolves to the Item's original language and `except` means "contains any other language"; a known-null language never matches even when negated; a regex over a known-empty value is a non-match, so its negation is a match. _Source: design/evaluation.md §1.4._

**EVL-041** (MUST) An autobrr filter field **MUST** map to one condition with a value list (OR inside the condition) and `required = true`; an `except_*` field **MUST** map to `negate + required`. The CF grouping rule then reproduces autobrr's AND across fields. autobrr's `match_releases`/`except_releases` map to `release_title`; `shows` to `parsed_title`; `min_size`/`max_size` to one `size` range. _Source: design/evaluation.md §3.8._

**EVL-042** (MUST) A condition reading an `Unknown` fact **MUST** evaluate to `Unknown`, which is never satisfied in either polarity (a negated `Unknown` is still unsatisfied) and is reported as `FactUnknown{fact}` in the trace. _Source: design/evaluation.md §3.8._

#### 3.6.1 Lazy enrichment

**EVL-043** (MUST) After stage 5, for each target whose only non-passing checks are `Unknown` checks, evaluation **MUST** request enrichment of exactly the facts those checks read, by calling acquisition's `Enrich(ctx, release, field_set)` (ACQ-146) once per release with the union of fields over all such targets, and then re-evaluate from stage 3 with the enriched Release. It **MUST NOT** request enrichment when no target could change verdict. _Source: design/evaluation.md §3.8; ACQ-149. [OPN-018]_

**EVL-044** (MUST) The enrichment call **MUST** carry a deadline (default 10 s on the IRC path, 30 s otherwise); fields acquisition marks `unavailable`, or not filled by the deadline, **MUST** stay `Unknown`, and the decision proceeds with them. Enrichment latency **MUST** be reported separately from evaluation latency (`bonnyread_evaluation_enrich_seconds`). A `.torrent` fetched for enrichment is reused at grab through acquisition's payload cache (ACQ-134); evaluation **MUST** pass the release key unchanged to the grab so the cache hits. [OPN-018] _Source: ACQ-147, ACQ-148; design/evaluation.md §4.3._

**EVL-045** (MUST) The profile size gate (EVL-054) **MUST** keep the Arr behaviour for a size that is still unknown after enrichment was attempted or is impossible: the gate is skipped with a `SizeUnknown` finding of outcome `Unknown` (Sonarr/Radarr `AcceptableSizeSpecification`: "Release has unknown size, skipping size check"). _Source: sonarr AcceptableSizeSpecification.cs:35-37; design/evaluation.md §4.3._

#### 3.6.2 Compilation and regex safety

**EVL-046** (MUST) Patterns **MUST** be compiled once per profile-store generation through the shared normaliser and regexp2 path of `07-profiles.md` §3.19 (`\p{IsBlock}` rewrite, `IgnoreCase`, default .NET mode, per-match timeout 100 ms), de-duplicated by normalised text across all CFs and rules. _Source: D1; design/evaluation.md §3.8._

**EVL-047** (MUST) Each compiled pattern **MUST** get a **literal prefilter**: the longest case-folded literal that every match must contain (for example `GROUP` in the `(?<=^|[\s.-])GROUP\b` idiom), checked with a case-insensitive substring test before regexp2 runs. A pattern with no extractable mandatory literal runs without a prefilter. A prefilter **MUST NOT** change results; a CI test **MUST** compare prefiltered and unfiltered results over every Dictionarry and trash-pcd pattern and test title. _Source: A20; design/evaluation.md §3.8._

**EVL-048** (MUST) A pattern that fails to compile **MUST** disable its CF (never matches) and raise a health issue. A match timeout **MUST** yield `Unknown` for that condition, increment `bonnyread_regex_timeouts_total{source="custom_format"}` (`12-platform.md` PLT-134; the pattern is not a metric label, PLT-098) and mark the pattern "slow" in the UI. _Source: D1; design/evaluation.md §3.8._

**EVL-049** (MUST) Only reachable CFs **MUST** be compiled for the hot path: those with a non-zero score in at least one profile bound to an active slot or rule for that kind. The testing UI compiles on demand. _Source: design/evaluation.md §3.17._

**EVL-050** (MUST) Outside explain mode, conditions within a CF **MUST** be evaluated required-first, and the CF short-circuits on the first failed required condition. In explain mode every condition is evaluated and traced (per CF → per group → per condition: pass, fail or unknown, with the matched span). _Source: design/evaluation.md §3.8, §3.19._

### 3.7 Profiles and per-slot scoring

**EVL-051** (MUST) Evaluation **MUST** read profiles only through the `07-profiles.md` read interface (PRF-088), as a per-kind snapshot keyed by generation, and **MUST** compile it off the hot path into `CompiledRules{generation}` swapped atomically on `profiles.changed` (PRF-087). In-flight evaluations finish on the old generation; every decision **MUST** record the generation used. _Source: design/evaluation.md §3.17._

**EVL-052** (MUST) The compiled profile for a slot **MUST** contain: family; ordered quality items with group structure (items in one group are equal for upgrade purposes); allowed set; cutoff; `upgrades_allowed`; per-CF scores already resolved for the slot's score domain through the fallback chain (PRF-081 and §3.14 of `07-profiles.md`); `min_cf_score`, `upgrade_until_score`, `upgrade_score_increment`; the language gate; size definitions; the resolved delay policy (§3.12); and the propers policy (§3.10). Evaluation **MUST NOT** interpret PCD `arr_type`. [OPN-059] _Source: design/evaluation.md §3.9; design/profiles.md §3.8.1._

**EVL-053** (MUST) A slot's score **MUST** be the dot product of the shared CF match set with the slot's score vector. CF matching is computed once per Facts; only the dot product is per slot. _Source: design/evaluation.md §3.9._

**EVL-054** (MUST) Stage 5a profile checks **MUST** be, each always run and reported: `QualityFamilyMismatch`; `QualityNotAllowed`; `RawDisk` (a raw-disk release unless its quality is an allowed BR-DISK quality); `LanguageNotWanted` (PCD profile language gate: must include the language; `Original`/`Any` sentinels); `CFBelowMinimum` (`score < min_cf_score`, naming the dominant negative CF); `SizeOutOfRange` (per-quality min/max × runtime summed over covered Items; EVL-045 for unknown size); `UnknownRuntime` (runtime of every covered Item is 0); `FreeSpace`. _Source: design/evaluation.md §1.5, §3.11._

**EVL-055** (MUST) Release profiles **MUST NOT** exist. "Must not contain" is a CF with a large negative score and a minimum score ≥ 0; "must contain" is a CF with a required condition, a positive score and a raised minimum; release-profile indexer scoping is the `indexer` condition type. Radarr's HardcodedSubs setting **MUST** be a built-in default CF. _Source: design/evaluation.md §3.9._

### 3.8 Intake and item gates

**EVL-056** (MUST) Stage 0 intake checks (release only) **MUST** be: `Retention`, `MinimumAge`, `MaximumSize` (global), `ProtocolDisabled`, `IndexerBackoff`, `TorrentMinSeeders` (per indexer), `Obfuscated`, `Sample`. _Source: design/evaluation.md §3.11._

**EVL-057** (MUST) Stage 3 item gates **MUST** be: `Blocklisted` (§3.8.1); `NotMonitored` (ItemSlot `monitored ∧ TitleSlot.monitored`, `02-domain-model.md`; skipped for interactive search); `AlreadyImported` (same download identity already imported for the ItemSlot); `FullSeasonNotAired` (a pack is rejected until every episode has aired, unless the pack policy of EVL-071 allows packs before all episodes aired; single unaired monitored episodes remain grabbable, per the `grabbable(i,s)` predicate that `02-domain-model.md` DOM-088 assigns to this file: `grabbable(i,s) := monitored(i,s)` regardless of `released`, `include_specials` (DOM-054) and `search_hold` (which suppresses only automatic searches) [OPN-207]); music `EarlyRelease`, `DiscographyUnreleased`, `YearMismatch` (P2); `SeasonPackOnly` (search context requires a pack). Wanted and coverage **MUST** be read from `02-domain-model.md`'s wanted queries, including derived coverage from collected editions where enabled. [OPN-038] [OPN-047] _Source: design/evaluation.md §3.11; design/library.md §3.5, §4.4._

#### 3.8.1 Blocklist and history

**EVL-058** (MUST) A release whose identity matches a non-expired `Blocklist` row of `05-downloads.md` **MUST** get `Blocklisted{blocklist_id, reason}`. Identity is the info hash for torrents, `(indexer, guid)` for Usenet and DDL, with `(title, size)` as fallback (the identity rules of `05-downloads.md`). Matching **MUST** be global by identity, not per Title (the same bytes are the same release); the row's scope is used only for display. Expiry follows `05-downloads.md`. [OPN-110] [OPN-016] _Source: design/downloads.md §1.2 I4, §3.4; design/evaluation.md §3.15._

**EVL-059** (MUST NOT) Evaluation **MUST NOT** read grab history to decide. History is written by `05-downloads.md`; in-flight state comes from SlotState (§3.9). The Sonarr 12-hour history window and the "completed download handling off" branch **MUST NOT** exist. _Source: design/evaluation.md §3.10, §3.15._

### 3.9 Incumbents and upgrade

**EVL-060** (MUST) SlotState **MUST** hold, per ItemSlot, one incumbent list and the ItemSlot `version` (a counter on the ItemSlot row owned by `02-domain-model.md`, bumped by every change to its incumbents, monitoring, TitleSlot profile binding and by every grab transaction touching it). It **MUST** be loaded from the database at start and maintained from the durable file and download events of `06-import.md` and `05-downloads.md` (file imported or deleted, grab started, download target resolved, download reached a terminal phase). _Source: design/evaluation.md §3.10._

**EVL-061** (MUST) An incumbent **MUST** be either **File** — stored quality id, revision, release group, `date_added`, and the stored facts digest (attributes, languages, flags, size, scene name) of the MediaFile — or **InFlight** — a DownloadTarget for the ItemSlot whose Download phase is not terminal (`imported`, `failed`, `abandoned`) and whose target outcome is `pending`. `import_blocked` downloads **MUST** count as InFlight. _Source: design/evaluation.md §3.10; design/downloads.md §3.4._

**EVL-062** (MUST) An incumbent's CF score **MUST** be recomputed against the current compiled profile from its stored facts digest with the same engine, with no re-parse of history strings. _Source: design/evaluation.md §3.10._

**EVL-063** (MUST) For each Item in its gain (§3.11 for packs), a candidate **MUST** beat **every** incumbent of that ItemSlot: first `CutoffNotMet(incumbent)`, else `CutoffMet{File | InFlight}`; then `Upgrade(p, incumbent, candidate)`. In `overlapping` coverage mode (comics) the File comparison **MUST** use only the best File by profile rank, plus every InFlight incumbent. _Source: design/evaluation.md §3.10; design/library.md §3.2, §4.4._

**EVL-064** (MUST) `Upgrade(p, c, n)` **MUST** be one function returning a reason enum, applying Sonarr `UpgradableSpecification.cs:31-122` in exactly this order:
1. `n` has a better quality and the quality cutoff is not met → upgrade;
2. `n` has a worse quality → reject `BetterQuality`;
3. propers ≠ `DoNotPrefer` and `n` has a better revision → upgrade;
4. `¬p.upgrades_allowed` → reject `UpgradesNotAllowed`;
5. propers ≠ `DoNotPrefer` and `n` has a worse revision → reject `BetterRevision`;
6. `n` has a better quality (so the cutoff is met) → reject `QualityCutoff`;
7. `score(n) ≤ score(c)` → reject `CustomFormatScore`;
8. `score(c) ≥ p.upgrade_until_score` → reject `CustomFormatCutoff`;
9. `score(n) < score(c) + p.upgrade_score_increment` → reject `MinCustomFormatScore`;
10. otherwise → upgrade.

_Source: design/evaluation.md §1.6._

**EVL-065** (MUST) Supporting rules **MUST** be preserved: `QualityCutoffNotMet` uses the cutoff, or the **first allowed quality** when upgrades are disabled, and is true for a same-quality revision upgrade; `CutoffNotMet` is the quality cutoff OR the CF cutoff, where the CF cutoff uses `min_cf_score` when upgrades are disabled; `IsRevisionUpgrade` requires the **same quality** (a WEBRip PROPER never upgrades a WEBDL); quality comparison uses the profile's item order with qualities in one group equal, and revision breaks ties. _Source: design/evaluation.md §1.3, §1.6._

**EVL-066** (MUST) A release covering only part of an existing multi-Item file for the slot **MUST** get `SameItems` (Sonarr `SameEpisodesGrabSpecification`). _Source: design/evaluation.md §3.10._

**EVL-067** (MUST) When a candidate is accepted as an upgrade over an InFlight incumbent and is committed, the superseded Download's disposition **MUST** be decided by `05-downloads.md`; evaluation emits `GrabSupersedes{download_id}` in the grab transaction. [OPN-009] _Source: design/downloads.md §6 Q2; open-issues.md Group C._

### 3.10 Propers, repacks and anime versions

**EVL-068** (MUST) The propers policy **MUST** be one of `PreferAndUpgrade`, `DoNotUpgrade`, `DoNotPrefer`, read from the effective media-settings `propers_repacks` value for the slot's kind (`07-profiles.md`). _Source: design/evaluation.md §1.7._

**EVL-069** (MUST) A REPACK or an anime version bump **MUST** be accepted as an upgrade only from the **same release group** as the incumbent, and **MUST** be rejected (`RepackGroupMismatch`, `RepackGroupUnknown`) when the groups differ or either is unknown. A PROPER from any group is acceptable. _Source: Sonarr RepackSpecification.cs:27-81, AnimeVersionUpgradeSpecification.cs:56-68._

**EVL-070** (MUST) On RSS and IRC (not search), a proper for a File incumbent added more than 7 days ago **MUST** be rejected `ProperForOldFile`. Under `DoNotUpgrade` a revision-only upgrade **MUST** be rejected `PropersDisabled`. Under `DoNotPrefer` revision **MUST** drop out of both the upgrade rules and ranking. _Source: Sonarr RssSync/ProperSpecification.cs:51-55; DownloadDecisionComparer.cs:73-76._

### 3.11 Season packs and multi-Item releases

**EVL-071** (MUST) A full pack (season pack, discography, TPB) **MUST** be accepted under the pack policy `{Any (≥ 1 Item), All (100%), Threshold (≥ n%)}` computed over its Items, where a missing Item counts as upgradable and an Item with Files counts if `CutoffNotMet ∧ Upgrade = upgrade` for every incumbent (Sonarr `UpgradeDiskSpecification.cs:34-104`); otherwise `PackBelowThreshold{upgradable, total, policy}`. The pack's **gain** is its upgradable Items. The default policy **MUST** be `All` (Sonarr default, threshold 100%). The policy **MUST** be a per-kind media setting with a per-TitleSlot override. [OPN-112] _Source: design/evaluation.md §1.6, §3.10, §6 Q5; Sonarr ConfigService.cs:277-286._

**EVL-072** (MUST) A multi-Item release that is not a full pack **MUST** pass for every Item it covers; its gain is all its Items. _Source: design/evaluation.md §3.10._

### 3.12 Delay and pending holds

**EVL-073** (MUST) The delay policy **MUST** be persisted profile data: a `DelayProfile` (PCD `delay_profiles` shape with the extension-schema additions: per-protocol delays in seconds including DDL, preferred protocol, only-protocol, `bypass_if_highest_quality`, `bypass_if_above_custom_format_score` with its minimum), attached to the quality profile with a per-slot override, stored and resolved per `07-profiles.md` PRF-073. Autograb rules carry their own delay reference. [OPN-113] _Source: D20; open-issues.md B3; design/evaluation.md §3.14._

**EVL-074** (MUST) Delay semantics **MUST** follow Sonarr `DelaySpecification.cs:32-115`: a user-invoked search skips the delay; the delay is per protocol; it is bypassed (a) for a revision upgrade of an existing File on the preferred protocol when the propers policy is `PreferAndUpgrade`, (b) with `bypass_if_highest_quality` when the quality is ≥ the last allowed quality and the protocol is preferred, (c) with `bypass_if_above_custom_format_score` when score ≥ the minimum and the protocol is preferred. _Source: design/evaluation.md §1.8._

**EVL-075** (MUST) The hold window **MUST** be per (slot, Item): `window_start = min(published_at, first_seen_at)` over the acceptable candidates for that ItemSlot. A candidate becomes releasable when `now ≥ window_start + delay` for **any** of its Items; until then it is `Defer{until}` with finding `Delay{until, protocol, delay}`. _Source: design/evaluation.md §1.8, §3.14._

**EVL-076** (MUST) Movies **MUST** return `Defer{until}` with `Availability{until}` until the TitleSlot's minimum availability (plus its delay) is reached, not a permanent rejection. _Source: design/evaluation.md §3.11._

**EVL-077** (MUST) Every Defer and every unselected accepted candidate (§3.14) **MUST** be persisted as a `PendingGrab` row (§4) with reason `Delay`, `ClientUnavailable`, `GrabBudget` or `Fallback`, unique on `(slot target, release)`. For each Defer, a durable `pending.resolve` job **MUST** be enqueued in the same transaction at `due_at`, with dedupe key `pending.resolve:<slot>:<title>` and concurrency key `eval:title:<title id>` (`12-platform.md`). _Source: design/evaluation.md §3.14; A1._

**EVL-078** (MUST) The resolve job **MUST** re-evaluate every hold for the affected ItemSlots against the current snapshot from stage 3, then rank, select (§3.14) and commit the winners. Holds that are now rejected **MUST** be deleted with a `ReleaseDecision` row stating why. _Source: design/evaluation.md §3.14._

**EVL-079** (MUST) Profile, slot, monitoring and delay changes, and imports or grab terminations touching an ItemSlot with holds, **MUST** enqueue the same resolve job (collapsed by its dedupe key). `ClientUnavailable` and `GrabBudget` holds **MUST** retry with backoff (from `ErrBudget.retry_at` where given). `Fallback` holds **MUST** expire after 24 h or when every Item in their gain has its cutoff met, and **MUST** be re-evaluated immediately when a grab covering their Items fails. _Source: design/evaluation.md §3.14._

**EVL-080** (MUST NOT) Delay **MUST NOT** depend on RSS ticks or in-memory sleeps; holds survive restarts and node failover. _Source: design/evaluation.md §3.14; ACQ-101._

### 3.13 Ranking

**EVL-081** (MUST) One comparer engine **MUST** rank candidates using a per-kind ordered key list defined as data, reproducing the Arr comparers exactly:

| Kind | Keys (first differing key decides) |
|---|---|
| tv | Quality (profile index, then revision unless `DoNotPrefer`), CFScore, Protocol (preferred from the delay policy), Coverage (full pack first; then fewer Items for standard, more for anime), FirstItem (lower first episode), IndexerPriority (lower better), IndexerFlags\*, Peers (torrents only: round(log10 seeders), then round(log10 peers)), Age (Usenet only: < 1 h, ≤ 24 h, ≤ 7 d, then −round(log10 days)), Size (closest to preferred × runtime, rounded to 200 MB; larger when no preferred size) |
| movie, adult | Quality, CFScore, Protocol, IndexerPriority, IndexerFlags\*, Peers, Age, Size |
| music | Quality, CFScore, YearMatch, Protocol, IndexerPriority, Peers, Coverage (album count), Age, Size |
| book, comic | Quality, CFScore, Protocol, Coverage, IndexerPriority, Peers, Age, Size |

\* only when the slot's "prefer indexer flags" setting is on (Radarr semantics; off by default). _Source: design/evaluation.md §1.9, §3.13._

**EVL-082** (MUST) Rank keys **MUST** be precomputed into a `RankKey` tuple per candidate, so sorting is cheap and explanations can name the first differing key. _Source: design/evaluation.md §3.13._

### 3.14 Coverage-aware selection (D20)

**EVL-083** (MUST) Selection **MUST** run per slot and per Title over the batch's accepted and deferred candidates plus that slot's existing holds for the Title's Items. It **MUST** replace Sonarr's greedy "best first, skip overlaps" order. _Source: D20; open-issues.md B7._

**EVL-084** (MUST) Before optimising, a candidate **MUST** be dropped as dominated (and held as `Fallback{dominated_by}`) when another candidate has the same Item set, a gain that is a superset, and a better RankKey. Cross-indexer copies of one release (same info hash, or same title and size for Usenet; ACQ-088) are therefore resolved by the rank keys (indexer priority, peers). _Source: design/evaluation.md §3.13; ACQ-088._

**EVL-085** (MUST) Within each selection component, evaluation **MUST** choose the set S of candidates with pairwise-disjoint Item sets that is best by this lexicographic objective:
1. **coverage:** maximise the number of distinct wanted Items in the union of the members' gains, counting a missing Item and an upgradable Item equally;
2. **quality and score:** compare the members' RankKeys sorted best first, element by element; the first better element wins; if one list is a prefix of the other, the shorter list wins;
3. **determinism:** the lexicographically smaller sorted list of release ids.

Per-kind comparer invariants (EVL-081) are kept inside step 2 and in every other ordering (interactive display, holds). [OPN-114] _Source: D20 ("prefer the combination that satisfies more wanted items, then profile score and quality"); open-issues.md B7._

**EVL-086** (MUST) The optimum **MUST** be computed by exact search over the component with a node budget (default 10,000 nodes). When the budget is exhausted, a deterministic greedy fallback **MUST** be used: repeatedly add the disjoint candidate with the largest new gain, ties broken by RankKey then release id. The fallback **MUST** increment `bonnyread_evaluation_selection_fallback_total` and be recorded on the decision. [OPN-115] _Source: D20; derived._

**EVL-087** (MUST) A deferred candidate in the chosen set **MUST** hold its Items: nothing is grabbed for them in this pass, and accepted candidates overlapping it become `Fallback{held_by}` holds. This preserves Sonarr's rule that the best-ranked release for a set of Items waits and a lower one does not jump the queue (`ProcessDownloadDecisions.cs:61-65`), with "best" now defined by EVL-085. _Source: design/evaluation.md §3.13._

**EVL-088** (MUST) Every accepted candidate not in S **MUST** become a `Fallback` hold with finding `SupersededBy{release_ids, coverage_self, coverage_winner, first_differing_key}`, so a failed grab can be replaced without a new search (EVL-079). _Source: design/evaluation.md §3.13._

### 3.15 One grab feeding several slots (D20)

**EVL-089** (MUST) After per-slot selection, the selected (release, slot, gain) triples **MUST** be merged by release: one release selected in several slots of one Title **MUST** produce **one grab** whose target set is the union of `(Item, slot)` for each slot's gain. A release selected in one slot and rejected or unselected in another is normal and is explained per slot. _Source: D20 (D4 amended); open-issues.md B1._

**EVL-090** (MUST) A grab's DownloadTargets **MUST** be exactly the gain Items per slot, not every Item in the payload; import (`06-import.md`) places one library file per slot, hardlinked or reflinked where possible, else copied, and each slot upgrades independently afterwards. _Source: D20; design/downloads.md §3.9.1._

**EVL-091** (MUST) When a release is selected for a slot while a non-terminal Download of the same release identity already exists for the same Title in phase `submitting`, `downloading` or `extracting`, the grab transaction **MUST** attach new DownloadTargets to that Download instead of creating a second Download. In any later phase a new grab **MUST** be created. [OPN-116] _Source: D20; ACQ-133._

**EVL-092** (MUST) A release selected for different Titles (for example a multi-title pack) **MUST** produce one grab per Title. _Source: derived from D20 and design/downloads.md §3.4 (targets per Title)._

### 3.16 Grab transaction (HA-safe)

**EVL-093** (MUST) Every grab **MUST** be committed through `grab.Reserve` in one database transaction:
1. lock the ItemSlot rows of every target in `(slot_id, item_id)` order (`SELECT … FOR UPDATE` on PostgreSQL; `BEGIN IMMEDIATE` on SQLite);
2. compare each row's `version` with the snapshot version the decision used;
3. for every target whose version differs, reload that ItemSlot's incumbents from the database and re-run stages 5a–5c for that slot's candidate; drop targets that no longer accept (`IncumbentChanged`);
4. re-check the blocklist (EVL-058) and the existing-Download attach rule (EVL-091);
5. if targets remain: insert the Download and DownloadTargets (`05-downloads.md`) or attach targets, bump the version of every locked ItemSlot, delete the `PendingGrab` rows the grab satisfies, write the `ReleaseDecision` rows, and enqueue the download submit job (the transactional outbox, `12-platform.md`);
6. commit; then emit `GrabStarted`.

_Source: design/evaluation.md §3.16; D14; A4._

**EVL-094** (MUST) If step 3 drops targets whose slot had other selectable candidates, or all targets are dropped, the transaction **MUST** commit only the remaining work and enqueue `pending.resolve` for the affected ItemSlots; evaluation **MUST NOT** retry the same decision more than once inline. _Source: design/evaluation.md §3.16._

**EVL-095** (MUST) On the IRC path the grab transaction **MUST** carry and check the IRC network lease fence token (ACQ-142), so a node that lost the lease cannot commit. Every other path relies on the version check alone. _Source: ACQ-142; A1._

**EVL-096** (MUST) If no download client can accept the protocol at commit time, the candidate **MUST** become a `ClientUnavailable` hold instead of a grab; if acquisition reports the grab budget exhausted (`Budget.Available` = 0 or `ErrBudget`), it **MUST** become a `GrabBudget` hold due at `retry_at`. _Source: design/evaluation.md §3.11, §1.8; ACQ-127._

### 3.17 Explainability and persistence

**EVL-097** (MUST) Every check **MUST** run and record a Finding, except that stages 0 and 2 **MAY** exit early on the hot path for releases that can never be interesting (`NotWantedKind`, `UnknownTitle`, `Retention`); those carry exactly that one reason. There **MUST NOT** be Arr-style priority tiers that hide later reasons. _Source: design/evaluation.md §3.12._

**EVL-098** (MUST) Explain mode (`explain = true`: interactive search, "why wasn't this grabbed", profile testing) **MUST** add pass findings and the CF trace, and **MUST** be deterministic for the same facts and snapshot generation. _Source: design/evaluation.md §3.12._

**EVL-099** (MUST) Decisions for matched releases **MUST** be stored as `ReleaseDecision` rows, kept 7 days by default. Unmatched releases **MUST** only increment `bonnyread_evaluation_rejections_total{code}`. Reason codes **MUST** be a stable enum localised by the UI; parameters carry values, never pre-rendered text. _Source: design/evaluation.md §3.12._

**EVL-100** (MUST) Interactive search **MUST** show each release's per-slot verdict, reasons and rank relative to the winner, naming the first differing key or the coverage difference ("lost to #1: covers 10 wanted episodes vs 1"). The Item page **MUST** answer "why not grabbed" from `ReleaseDecision` and `PendingGrab` for its ItemSlots. Profile testing (`07-profiles.md` §3.17) **MUST** call the pure `evaluate.Simulate(titles | releases, profile, item?)`. API shapes are owned by `11-api-and-clients.md` (interactive search: API-106, API-107; "why not grabbed": [OPN-204]). _Source: design/evaluation.md §3.12; design/profiles.md §3.10.3._

### 3.18 Performance, the IRC hot path and HA

**EVL-101** (MUST) `CompiledRules`, `LibraryIndex`, `SlotState` (copy-on-write per Title shard) and the parse cache **MUST** be immutable snapshots behind atomic pointers. The evaluator **MUST** take no locks and do no database reads before the grab transaction (enrichment excepted). _Source: design/evaluation.md §3.19._

**EVL-102** (MUST) Snapshots **MUST** be maintained on every node that evaluates, from durable events (`12-platform.md`); `LISTEN/NOTIFY` is only a doorbell (A4). Each node **MUST** additionally compare its snapshot generations with the database at most every 5 s and reload what is behind. Staleness is safe because the grab transaction re-validates (EVL-093). [OPN-117] _Source: A4; D14; design/evaluation.md §3.19._

**EVL-103** (MUST) Placement **MUST** follow A1: RSS and search batches are evaluated by `evaluate_releases` jobs on any node (ACQ-099); IRC announces are evaluated in-process on the node holding that IRC network's lease, via `ReleaseSink.Announce`; resolve jobs run on any node. Batches **MUST** be evaluated by a bounded worker pool, with ranking and selection after all candidates of the batch are in. _Source: A1; design/acquisition.md §3.8; design/evaluation.md §3.19._

**EVL-104** (MUST) The IRC path **MUST** use the compiled-rule cache and literal prefilters (EVL-047) and **MUST** enforce a **per-release deadline** of 250 ms for the CF stage, in addition to the 100 ms per-match timeout: when it expires, remaining conditions are `Unknown` (`EvalDeadline` finding) and evaluation continues rather than stalling the consumer. Patterns that repeatedly time out **MUST** be flagged in the UI. _Source: A20; design/evaluation.md §3.19._

**EVL-105** (SHOULD) The p99 from announce to decision, excluding enrichment and the grab transaction, **SHOULD** be under 5 ms. It **MUST** be measured by the histograms `bonnyread_evaluation_parse_seconds`, `bonnyread_evaluation_match_seconds`, `bonnyread_evaluation_cf_seconds`, `bonnyread_evaluation_total_seconds`, `bonnyread_evaluation_enrich_seconds` and counters `bonnyread_evaluation_rejections_total{code}`, `bonnyread_regex_timeouts_total{source}`, `bonnyread_evaluation_deadline_exceeded_total`, `bonnyread_evaluation_grab_conflicts_total`. _Source: design/evaluation.md §3.19._

### 3.19 Autograb rules (later phase)

**EVL-106** (MUST) An autograb rule **MUST** be evaluated as "a profile without an Item": its scope (indexers, announce types, kinds, categories) and conditions run on the same Facts with the same engine and semantics (EVL-039); its optional profile gives quality ranking and scores; findings use the same Finding type (autobrr's `{field, got, want}` maps to parameters). Rules are stored in the extension schema (`07-profiles.md`). _Source: design/evaluation.md §3.18; D20._

**EVL-107** (MUST) A rule's incumbents **MUST** be its prior grabs with the same dedupe identity (title + S/E, title + year, artist + album, or the normalised title for kind-less releases). The upgrade policy **MUST** be one of `RejectDuplicates`, `ProperSameGroupOnly`, `ProfileUpgrade` (EVL-064 verbatim) or `NewerUnitsOnly` (autobrr "smart episode"). Blocklist checks for rules use the global identity match (EVL-058). _Source: design/evaluation.md §3.18._

**EVL-108** (MUST) Rules **MUST** be tried in priority order and the first accepting rule wins (autobrr semantics). A rule's limits (grabs per fixed or rolling window) **MUST** be checked in the grab transaction; racing grabs also count against per-indexer budgets. External conditions **MUST** run last, asynchronously, with a timeout and `on_error`. [OPN-008] _Source: design/evaluation.md §3.18; ACQ-151._

**EVL-109** (MUST) When a library slot accepted the same release in the same evaluation, a rule **MUST NOT** fire; it records `SkippedLibraryGrab`. [OPN-017] _Source: design/evaluation.md §3.18, §6 Q2; open-issues.md Group C._

**EVL-110** (MUST) In library mode no rules are needed: IRC releases enter the normal pipeline, and autobrr's `shows`, `years` and list fields are replaced by the library. _Source: design/evaluation.md §3.18._

### 3.20 Reason codes

**EVL-111** (MUST) The reason-code enum **MUST** contain at least the codes below, with the stated stage and effect. New codes are additive only (A7).

| Stage | Code | Effect | Parameters |
|---|---|---|---|
| 0 Intake | `Retention`, `MinimumAge`, `MaximumSize`, `ProtocolDisabled`, `IndexerBackoff`, `TorrentMinSeeders`, `Obfuscated`, `Sample` | Reject | limit, value |
| 1 Parse | `Unparseable` | Reject | kinds tried |
| 2 Match | `NotWantedKind`, `UnknownTitle`, `MatchesOtherTitle`, `UnknownItem`, `AmbiguousMatch`, `SearchMismatch`, `SplitEpisode`, `MultiSeason`, `SceneMappingMismatch` | Reject | tier, candidates, ids |
| 3 Item gates | `Blocklisted`, `NotMonitored`, `AlreadyImported`, `FullSeasonNotAired`, `EarlyRelease`, `DiscographyUnreleased`, `YearMismatch`, `SeasonPackOnly` | Reject | ids, dates |
| 5a Profile | `QualityFamilyMismatch`, `QualityNotAllowed`, `RawDisk`, `LanguageNotWanted`, `CFBelowMinimum`, `SizeOutOfRange`, `UnknownRuntime`, `FreeSpace` | Reject | got, want, dominant CF |
| 5a Profile | `SizeUnknown` | Unknown, non-blocking | — |
| 5b Incumbent | `CutoffMet{File \| InFlight}`, `BetterQuality`, `BetterRevision`, `UpgradesNotAllowed`, `QualityCutoff`, `CustomFormatScore`, `CustomFormatCutoff`, `MinCustomFormatScore`, `PackBelowThreshold`, `SameItems`, `RepackGroupMismatch`, `RepackGroupUnknown`, `ProperForOldFile`, `PropersDisabled` | Reject | incumbent, scores |
| 5c Timing | `Delay`, `Availability`, `ClientUnavailable`, `GrabBudget` | Defer | until |
| Conditions | `FactUnknown`, `PatternTimeout`, `EvalDeadline` | Unknown | fact, pattern |
| 6 Selection | `SupersededBy`, `HeldBy`, `DominatedBy` | Fallback hold | release ids, coverage, key |
| 7 Commit | `IncumbentChanged`, `ReleaseUnavailable`, `TargetAttached`, `GrabSupersedes` | Reject / info | download id |
| Rules | `SkippedLibraryGrab`, `RuleLimitReached`, `DuplicateIdentity`, `ExternalCheckFailed` | Reject | rule, limit |

_Source: design/evaluation.md §3.11; autobrr rejections.go._

## 4. Data model

Owned here. `Release` is owned by `03-acquisition.md`; `ItemSlot` (including `version`), `TitleSlot`, `SlotDefinition`, `MediaFile` (including the stored facts digest) by `02-domain-model.md`; `Download`, `DownloadTarget` and `Blocklist` by `05-downloads.md`; `DelayProfile`, `CustomFormat`, `QualityProfile` and `AutograbRule` by `07-profiles.md`.

```go
// ReleaseDecision: one row per (release, target) for matched releases; 7-day retention (EVL-099).
ReleaseDecision {
  id            int64
  release_key   string            // ACQ release key; plus indexer_id
  indexer_id    int
  title         string            // raw title, for display
  target_kind   enum(slot, rule)
  slot          -> SlotDefinition?  ; title -> Title? ; rule -> AutograbRule?
  item_ids      []int64           // the Items considered for this target
  gain_item_ids []int64
  verdict       enum(accept, reject, defer, fallback, grabbed)
  codes         []ReasonCode
  params        json              // per finding
  rank_key      bytes             // encoded RankKey
  profile_gen, library_gen int64  // snapshot generations
  download      -> Download?      // when grabbed
  created_at    time
  index (title_id, slot_id, created_at); (release_key, indexer_id); (created_at)
}

// PendingGrab: persisted holds (EVL-077).
PendingGrab {
  id            int64
  slot          -> SlotDefinition?  ; title -> Title? ; rule -> AutograbRule?
  item_ids      []int64
  release       json              // Release snapshot without `download` secrets (ACQ-062); secrets re-read via SearchResult/payload cache
  parsed_digest json              // Facts digest (no re-parse on resolve)
  reason        enum(delay, client_unavailable, grab_budget, fallback)
  reason_params json              // superseded_by, held_by, delay, protocol
  window_start  time
  due_at        time
  expires_at    time?             // fallback TTL
  created_at    time
  unique partial (slot_id, title_id, release_key, indexer_id) WHERE rule_id IS NULL      // DOM-004: no NULL-distinct reliance
  unique partial (rule_id, release_key, indexer_id) WHERE rule_id IS NOT NULL
  index (due_at); (title_id, slot_id)
}
```

Notes:
- The Release snapshot in `PendingGrab` **MUST NOT** contain `download` refs (ACQ-062). At commit the grab passes the release key; `05-downloads.md` fetches through `03-acquisition.md` `Fetch`, which reloads the release by key.
- No enum tables: reason codes, verdicts and hold reasons are Go-typed enums (D9).
- Both tables are pruned by housekeeping jobs (`12-platform.md`).

## 5. Flows

### 5.1 Batch evaluation (RSS, search)

1. Claim `evaluate_releases{releases}` (ACQ-099) or receive a search result set.
2. For each release: intake (0) → parse (1, cached) → match (2) → facts and item gates (3). Early exit only per EVL-097.
3. Build candidates per slot of each matched Title (TitleSlots of the Items' Title).
4. Evaluate 5a/5b/5c per candidate; request enrichment per EVL-043 if needed and re-evaluate.
5. Per slot and Title: drop dominated candidates, select (EVL-085), mark holds.
6. Merge selections by release (EVL-089) and run one grab transaction per (release, Title).
7. Persist holds, enqueue resolve jobs, write `ReleaseDecision` rows.
8. Failure: a crash before commit retries the job; `RssSeen` already deduplicated upstream, and the grab transaction is idempotent through the version check.

### 5.2 IRC announce

1. `ReleaseSink.Announce` on the lease-holding node, with the fence token.
2. Stages 0–3 as in 5.1, under the 250 ms CF deadline.
3. If a target is blocked only by `Unknown` facts → `Enrich` (deadline 10 s); re-evaluate.
4. Select and commit with the fence token. Lease lost → commit fails; the release is recovered by the catch-up RSS after failover (ACQ-095).

### 5.3 Resolve job

1. Fires at `due_at` or on invalidation (EVL-079).
2. Loads the holds of the ItemSlots, re-evaluates from stage 3 on the current snapshot.
3. Deletes holds that now reject; ranks and selects; commits winners; reschedules the remaining Delay holds at their new `due_at`.

### 5.4 Grab conflict

1. Node A and node B both accept release X for ItemSlot (E05, HD) from IRC and RSS at the same moment.
2. A commits first and bumps the version.
3. B locks, sees a newer version, reloads incumbents: A's Download is now InFlight at equal quality → `CutoffMet{InFlight}` or `CustomFormatScore` → target dropped → `IncumbentChanged`. No second Download.

## 6. Walkthroughs

### 6.1 Season pack vs singles under coverage-aware selection

Setup: series "Show" (TVDB 123), slot HD. Profile ranks `[WEBDL-1080p ≈ WEBRip-1080p] < Bluray-1080p`, cutoff Bluray-1080p, upgrades on, min score 0, CF cutoff 200, pack policy `Threshold 75%`, delay 0. Season 1 has 10 aired episodes; E01–E06 are WEBDL-1080p Files with score 100; E07–E10 are missing.

RSS brings A `Show.S01.1080p.BluRay.x264-GRP` (pack, score 150), B `Show.S01E07.1080p.WEB-DL.DDP5.1.H.264-XYZ` (score 100), C `Show.S01E08.1080p.BluRay.x264-ABC` (score 200).

1. **Parse/match.** A → E01–E10 (full pack); B → E07; C → E08. A and C are Bluray-1080p, B is WEBDL-1080p.
2. **Evaluate.** A: E01–E06 upgrade (better quality, cutoff not met), E07–E10 missing → 10/10 ≥ 75% → Accept, gain {E01–E10}. B: Accept, gain {E07}. C: Accept, gain {E08}.
3. **Rank** (unchanged Sonarr keys): C > A > B.
4. **Select.** One component {A, B, C} (A overlaps both). Feasible sets: {A} covers 10 wanted Items; {B, C} covers 2; {B}, {C} cover 1. Step 1 picks **{A}**.
5. **Holds.** B and C become `Fallback` with `SupersededBy{A, coverage 1 vs 10}`.
6. **Commit.** One Download for A with 10 DownloadTargets (E01–E10, HD).
7. **Later.** If A fails, the Fallback holds are re-evaluated at once (EVL-079): with A gone, {B, C} is optimal and both are grabbed. Otherwise the holds expire after 24 h or when the cutoff of every Item in their gain is met; while they wait, any re-evaluation compares them against A as an InFlight incumbent under the unchanged upgrade rules (EVL-064).
8. **Explain.** C's row: "Accepted · not selected: A covers 10 wanted episodes, this release 1."

Contrast with Sonarr (kept as the traceability record): greedy order grabs C and B and keeps A as fallback, leaving E01–E06 at WEBDL and E09–E10 missing.

### 6.2 One movie release feeding the 4K and "Best" slots

Setup: movie M (TMDb 555), 130 min, released. Slot **4K**: profile `WEBDL-2160p < Bluray-2160p < Remux-2160p`, cutoff Remux-2160p, CFs "DV HDR10" +1500 and "TrueHD Atmos" +500; incumbent WEBDL-2160p File, score 1500. Slot **Best**: profile `WEBDL-1080p < Bluray-1080p < WEBDL-2160p < Remux-2160p`, cutoff Remux-2160p, same CFs; incumbent Bluray-1080p File, score 0. Slot **HD**: profile `WEBDL-1080p < Bluray-1080p`; incumbent none. Both 4K and Best: preferred Usenet, delay 0.

Release R (Usenet, 58 GB, RSS): `Movie.2023.2160p.UHD.BluRay.REMUX.DV.HDR.HEVC.TrueHD.Atmos-FraMeSToR`, indexer TMDb 555.

1. **Parse.** Movie, 2023; Bluray × 2160p × REMUX → Remux-2160p; HDR [DV, HDR10]; TrueHD Atmos; group FraMeSToR.
2. **Match.** Indexer TMDb id → M (ExternalID).
3. **Facts.** CF set {DV HDR10, TrueHD Atmos, …} computed once.
4. **4K.** Allowed; score 2000; 446 MB/min within limits; WEBDL-2160p incumbent → better quality, cutoff not met → Accept, gain {M}.
5. **Best.** Allowed; score 2000; Bluray-1080p incumbent → better quality → Accept, gain {M}.
6. **HD.** `QualityNotAllowed{Remux-2160p}` → Reject.
7. **Select** per slot: R wins in 4K and in Best.
8. **Merge.** One grab: Download(R) with DownloadTargets {(M, 4K), (M, Best)}. Acquisition fetches once (ACQ-133); import places one file in each slot's root, hardlinked where possible (`06-import.md`).
9. **Variation (different delays).** If R were a torrent and 4K had a 60 min torrent delay while Best had none: Best grabs at once; 4K holds `Delay{until +60 min}`. At `due_at` the resolve job selects R for 4K, finds the non-terminal Download of R for M in `downloading`, and attaches target (M, 4K) (EVL-091, `TargetAttached`). If the Download had already reached `importing`, a separate grab is made.
10. **Explain.** "4K: grabbed with Best (one download). Best: upgrade from Bluray-1080p. HD: Remux-2160p not in profile."

### 6.3 IRC freeleech announce

Announce on tracker T: `Release{origin: irc, protocol: torrent, title: "Show.S02E05.1080p.WEB.h264-GRP", categories: [TV-HD], size: null, flags: {download_factor: 0.0}, announce: {type: NEW, uploader: "bot", origin: P2P}, published_at = first_seen_at = now}`.

1. **Intake** passes. **Parse** (cache miss): TV via the category hint; S02E05; WEB + h264 → WEBDL-1080p.
2. **Match.** Clean title → Show → S02E05; ItemSlot (E05, HD) monitored, missing.
3. **Facts.** CF "Freeleech" (`indexer_flag = freeleech`, a view of `download_factor 0`) matches. CF "Size < 2 GB" is `Unknown` (FactUnknown{size}).
4. **Slot HD.** Freeleech +25; the size CF is unscored, so it cannot change the verdict. The size gate is `Unknown` and every other check passes → EVL-043 requests `Enrich{size}` (the IRC deadline applies).
5. Acquisition fetches the .torrent (payload cache): 1.4 GB. Re-evaluation: within WEBDL-1080p × runtime; no incumbent; delay 0 → Accept.
6. **Commit** with the lease fence: lock (E05, HD), version matches, insert Download + target. The fetch hits the payload cache, so the .torrent is downloaded once.
7. **Latency:** about 1–3 ms of evaluation plus the enrichment fetch, reported separately.
8. **Later phase, racing rule** "T freeleech NEW < 10 GB" (`freeleech_percent in [100]` required, `size <= 10 GB` required, limit 20/day): evaluated on the same Facts → would accept, but the library slot accepted the release in this evaluation → `SkippedLibraryGrab` (EVL-109). For a non-library announce the rule grabs, with identity = normalised title.

### 6.4 Ambiguous adult scene (P3)

Setup: studio "Brazzlers" with alias "BZ"; scenes S1 "Morning Session" (Jane Doe, John Roe) and S2 "Evening Session" (Jane Doe, Mary Poe), both 2024-03-15, both monitored in slot Default.

Release `BZ.24.03.15.Jane.Doe.XXX.1080p.MP4-GRP`.

1. **Parse** (adult). Studio "BZ", date 2024-03-15 (YY.MM.DD, not in the future), tokens "Jane Doe"; no code, no StashID.
2. **Match.** StashID tier: none. Code tier: none. Studio alias → Brazzlers; same-date scenes {S1, S2}. Graded levels: Title fails for both; **PerformersTitle** matches both — the first tier with candidates yields two distinct Items → `AmbiguousMatch{tier: PerformersTitle, candidates: [S1, S2]}` (EVL-022). The fuzzy tier is not consulted, because a tier already produced candidates; even if it were, both scores are below 80, and a tie within 5 is ambiguous.
3. **Verdict** Reject. An interactive search for S1 also rejects; the user may grab with an explicit target (EVL-025), recorded on the Download. Automatic search never guesses.
4. **Variation.** `BZ.24.03.15.Jane.Doe.BZ-1234.XXX.1080p` → the code index returns only S1 → Match S1 (ProductCode); evaluation proceeds normally. The completed-download and import fallback use the same lookup (EVL-027).
5. **Explain.** "No unique match: 2 scenes on 2024-03-15 for Brazzlers match performer 'Jane Doe' (S1 Morning Session, S2 Evening Session). Add a scene code or grab with an explicit target."

## 7. Non-goals

| Not done | Reason |
|---|---|
| Sonarr's greedy "best first, skip overlaps" selection | Replaced by coverage-aware selection (D20, B7) |
| Arr specification priority tiers | Existed to avoid DB queries; stage 5 is in-memory, so all reasons are shown |
| Separate upgrade checks against disk, queue and history, re-parsing stored titles | One incumbent list per ItemSlot with stored facts |
| History 12 h window and the CDH-off branch | InFlight state is exact (tracked downloads are authoritative) |
| Release profiles (Required/Ignored terms) | Expressed by CFs and scores |
| autobrr's 80-column filters, duplicate profiles and `release/push` | Rules on the same engine; library mode needs none |
| Profilarr's evaluator, C# parser sidecar and match caches | One in-process engine and parser |
| Per-app integer enums for qualities, languages and flags | Canonical names, mapped once at compile (D9) |
| Delay re-evaluated at RSS ticks; autobrr in-memory delay | Persisted holds with timer jobs |
| Radarr availability as a permanent rejection | `Defer{until}` gives the same outcome and a truthful explanation |
| Whisparr v3 fuzzy-first, top-score-wins matching | Fuzzy last, threshold and margin, ambiguity rejects (D10) |
| Multi-instance 4K/1080p setups and grabbing one release twice for two slots | One evaluation per release, one grab feeding several slots (D4, D20) |
| Racing mode in P1 | D16 |

## 8. Traceability

| Reference | Preserved | Deliberately changed |
|---|---|---|
| Sonarr | Parser cascade and corpus; attribute parsers; revision semantics; lookup order and "matches another series"; XEM/anime/daily numbering; language aggregation; CF semantics; upgrade rule order and supporting rules; repack/anime-version same-group rule; 7-day RSS proper rule; season-pack upgrade policy (default All); delay bypass and window rules; comparer key order; blocklist identity | Coverage-aware selection instead of greedy; no tiers; no history reads; release profiles dropped; holds persisted with timers instead of RSS ticks; one grab feeds several slots; blocklist matched globally by identity |
| Radarr | Parser cascade (year 1800–2099); movie lookup order with translations; Modifier axis; hashed-release rejection; profile language gate; IndexerFlags comparer key (opt-in); unknown-size skip | Availability defers instead of rejecting; HardcodedSubs as a default CF |
| Lidarr | Album/discography patterns; fuzzy location with margin; YearMatch and AlbumCount keys; 38 audio qualities | Audio family axes in one catalogue |
| Bookshelf | Title/subtitle split | Profiles bound to one family (ebook and audiobook in separate slots) |
| Whisparr v2 | Date patterns, ten product-code variants, non-unique code index with code fallback, ambiguity rejection | — |
| Whisparr v3 | Scene/StashID/JAV patterns, graded match levels, month/year widening, fuzzy threshold 80 and margin 5 | Fuzzy runs last and never takes a tie |
| Kapowarr | Comic filename parser and match filter with human-readable rejections | Rejections as typed findings |
| autobrr | Filter fields as conditions; announce metadata; freeleech percent; lazy .torrent enrichment; rule priority; smart episode; rejection `{field, got, want}` | One semantics (CF grouping); persisted delay; dedupe identities instead of duplicate profiles; library replaces `shows` lists |
| Profilarr | CF evaluation semantics and tests; per-condition trace UX; profile simulation | In-process engine, no sidecar |
