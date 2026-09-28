# 09. Subtitles

This file specifies subtitle management: subtitle records tied to MediaFiles, language profiles with hearing-impaired and forced handling, the wanted computation, search triggered by import events, host-side matching with the shared release parser, Bazarr's scoring and selection rules (preserved exactly), post-processing (decoding, normalisation, mods, synchronisation), upgrades, provider throttling, and the WASM contract for subtitle-provider plug-ins together with the order in which providers are ported. The generic plug-in runtime is in `12-platform.md`. Where subtitle files are written and renamed is owned by `06-import.md`.

Phases: **P2**. The groundwork in **P1** (sidecar indexing, subtitle stream dispositions in the shared probe, the shared subtitle table) is specified in `06-import.md` and `02-domain-model.md` (A17).

---

## Concepts

| Term | Definition |
|---|---|
| **Subtitle record** | One row in the shared `Subtitle` table (`02-domain-model.md` DOM-110), keyed to a MediaFile. It covers embedded tracks, sidecars indexed by import, and subtitles written by this subsystem. A sidecar's file is also an `ExtraFile{type=subtitle}` row of `06-import.md`, referenced by the Subtitle record; this file uses **Subtitle** for the record. |
| **Origin** | How a subtitle came to exist: `embedded`, `external` (sidecar found by import), `downloaded`, `uploaded`, `manual` (downloaded through interactive search), `extracted`, `translated`, `generated`. |
| **Subtitle profile** | An ordered list of profile items, plus a cutoff, must-contain and must-not-contain filters, and an original-format flag. Bazarr calls it a "languages profile". |
| **Profile item** | One wanted subtitle: language, `forced` flag, **HI mode** and an audio rule. |
| **HI mode** | `required` (only hearing-impaired subtitles satisfy it), `accepted` (either satisfies it; non-HI preferred), `excluded` (only non-HI satisfies it). Bazarr's names are "True", "False" (normal) and excluded; its internal names "force HI", normal and "force non-HI". |
| **Forced** | A subtitle that covers foreign-language parts only. A forced item is a separate wanted unit from the full subtitle in the same language. |
| **Want key** | `(language, forced, hi_mode)` of a profile item. Selection runs per want key. |
| **VideoFacts** | What the host tells a provider about a file: titles, ids, numbering, parsed release attributes, size, duration, fps and requested hashes. |
| **Candidate** | One subtitle offered by a provider, described as facts (release names, flags, claims). |
| **Claim** | A match a provider asserts itself and the host cannot recompute: `hash`, `imdb_id`, `tvdb_id`, `series_imdb_id`, `series_tvdb_id`, `tmdb_id`. |
| **Match set** | The attribute names that a candidate matches, computed by the host. |
| **Score / score_max** | The Bazarr score of a candidate and the maximum for its media type (360 episode, 180 movie). |
| **Provider** | A configured instance of a subtitle-provider plug-in. |
| **Syncer** | A tool that aligns subtitle timing to a reference: alass or ffsubsync. |
| **Mod** | A text transformation applied to a subtitle after download (ported from subzero). |

---

## Requirements

### Subtitle records

**SUB-001** (MUST) Subtitles **MUST** be stored in the single shared `Subtitle` table owned by `02-domain-model.md` (DOM-110), keyed to a MediaFile with cascade on delete. There **MUST NOT** be a second subtitle index, a mirror of library tables, path mappings, or rescan calls. _Source: design/metadata-and-subtitles.md §B1, §B2; design/import.md §3.6._

**SUB-002** (MUST) Each subtitle record **MUST** carry at least: `media_file`, `origin`, `language` (BCP 47-style: ISO 639 code plus optional region or script, for example `pt-BR`, `zh-Hant`, `es-419`), `hi`, `forced`, `format` (`srt`, `ass`, `ssa`, `vtt`, `sub`, `pgs`, `vobsub`, …), `path` (null for embedded), `stream_index` (embedded only), and for downloaded subtitles `provider`, `provider_sub_id`, `release_info`, `uploader`, `score`, `score_max` and `matches`. It **MUST** also record `sync` (tool, reference, offset in ms, framerate fixed, checker result) and the applied `mods`. `02-domain-model.md` DOM-110 owns the schema and defines these fields; `path` is the path of the referenced `ExtraFile` (`06-import.md`). _Source: design/metadata-and-subtitles.md §B2._

**SUB-003** (MUST) Phase 1 import **MUST** already index sidecar subtitles (language, HI, forced from the filename) and record embedded subtitle streams with their `forced` and `hearing_impaired` dispositions in the shared probe. This subsystem consumes those records and **MUST NOT** probe a file itself. _Source: A17; design/import.md §3.2, §3.6._

**SUB-004** (MUST) Embedded tracks **MUST** count as present subtitles. Title heuristics ("SDH", "CC", "Forced", "Signs") **MUST** fill missing dispositions. Settings **MUST** allow ignoring PGS, VobSub and ASS embedded tracks, and set a default language for `und` tracks. _Source: design/metadata-and-subtitles.md S10, §B4._

**SUB-005** (SHOULD) When a sidecar's filename has no language, content-based language detection **SHOULD** be used. _Source: design/metadata-and-subtitles.md §B4._

**SUB-006** (MUST) Extracting an embedded text track to an external file **MUST** be an explicit action or a profile option, run as a capability-routed tool job requiring `ffmpeg` (SUB-048). _Source: A6; design/metadata-and-subtitles.md §B4._

**SUB-007** (MUST) Subtitle history **MUST** use the shared history and event table [OPN-203] with these event types: `subtitle.downloaded`, `subtitle.upgraded`, `subtitle.manual`, `subtitle.uploaded`, `subtitle.synced`, `subtitle.modified`, `subtitle.deleted`, `subtitle.extracted`, `subtitle.blocklisted`. Upgrades **MUST** record `upgraded_from`. Bazarr's numeric action codes are not kept. _Source: D7; design/metadata-and-subtitles.md §B2, §B8._

### Language profiles

**SUB-008** (MUST) A subtitle profile **MUST** have: a name; ordered items; an optional cutoff item; `must_contain` and `must_not_contain` lists matched against release info; and `original_format`. Each item **MUST** have: `language`, `forced`, `hi_mode ∈ {required, accepted, excluded}`, and `audio_rule ∈ {always, skip_if_audio_is <lang>, only_if_audio_is <lang>}`. _Source: design/metadata-and-subtitles.md S9, §B2._

**SUB-009** (MUST) A forced item **MUST NOT** have `hi_mode = required`. The profile editor and the API **MUST** reject it. _Source: design/metadata-and-subtitles.md S5, §B3._

**SUB-010** (MUST) Global language equivalences (`language_equals`, for example `pt` ↔ `pt-BR`) **MUST** be configurable and used wherever languages are compared. Custom codes `pt-BR`, `zh-Hant` and `es-419` **MUST** be supported. _Source: design/metadata-and-subtitles.md S9._

**SUB-011** (MUST) One subtitle profile **MUST** be assigned per Title (series or movie), with defaults from root folder and tags, and it **MUST** apply to every file of every slot of that Title. [OPN-049] _Source: open-issues Group C; design/metadata-and-subtitles.md §6.2 item 4._

### Wanted computation

**SUB-012** (MUST) Wanted subtitles **MUST** be computed on demand from (file, profile, present subtitles, audio languages) and **MUST NOT** be stored as state:

```
Wanted(file, profile, present, audioLangs):
  for item in profile.items (in order):
    if item.audio_rule excludes this file given audioLangs: continue
    if satisfied(item, present):
      if item is profile.cutoff: return ∅
      continue
    emit want_key(item)

satisfied(item, p) ⇔ equiv(p.language, item.language)
                   ∧ p.forced == item.forced
                   ∧ (item.hi_mode == required ⇒ p.hi)
                   ∧ (item.hi_mode == excluded ⇒ ¬p.hi)
                   ∧ p counts (not an ignored embedded image format, SUB-004)
```

_Source: design/metadata-and-subtitles.md §B3._

**SUB-013** (MUST) Per-want-key search state (`first_missing_at`, `attempts`, `last_attempt_at`, `next_attempt_at`) **MUST** be stored per (media file, want key) and removed when the key stops being wanted. _Source: design/metadata-and-subtitles.md §B2._

### Triggers

**SUB-014** (MUST) The subtitle subsystem **MUST** react to these events and schedules:

| Trigger | Action |
|---|---|
| `file.imported` (`06-import.md`) | Compute wanted; enqueue a search job at priority `high` |
| `file.upgraded` | Apply SUB-052; search the new file |
| `file.deleted` | Nothing beyond the cascade; import removes sidecars |
| `file.renamed` / `file.moved` | Nothing; subtitles move with the file (import) |
| Profile edited or reassigned | Recompute wanted for affected files; enqueue searches at priority `background` |
| Wanted sweep, every 6 h | Enqueue files with a wanted key whose `next_attempt_at ≤ now` |
| Upgrade sweep, every 12 h | SUB-049 |
| Manual or interactive search | Synchronous fan-out returning every candidate with score and matched and unmatched attributes |
| Playback started (optional, from the media-server integration) | Search that file |

_Source: design/metadata-and-subtitles.md §B5; design/import.md §3.9; bazarr `general.wanted_search_frequency` (6), `general.upgrade_frequency` (12)._

**SUB-015** (MUST) Adaptive searching **MUST** be on by default: once a want key has been missing for 3 weeks, `next_attempt_at` **MUST** be set 1 week after each attempt. Both durations **MUST** be settings. _Source: design/metadata-and-subtitles.md S8; bazarr `adaptive_searching_delay` 3w, `adaptive_searching_delta` 1w._

**SUB-016** (MUST) Release information (scene name, release group, source, streaming service) **MUST** be taken in-process from the grab and import records of the file. _Source: design/metadata-and-subtitles.md §B5._

### Search orchestration

**SUB-017** (MUST) A search **MUST** make one provider call per (file, provider), passing every wanted language with a `forced_mode` of `only`, `also` or `none` derived from the want keys, and the HI preference per language. Selection then runs per want key. _Source: design/metadata-and-subtitles.md §B5; bazarr `download.py:48-50`._

**SUB-018** (MUST) Providers **MUST** be called concurrently with a per-call deadline. Throttled or disabled providers **MUST** be skipped. _Source: design/metadata-and-subtitles.md §B5._

**SUB-019** (MUST) `VideoFacts` **MUST** include: media kind; titles (primary, original, alternates, and scene titles from `08-metadata.md`); year; ids (IMDb, TMDb, TVDB series and episode, AniDB, MAL, AniList); season and episode in TVDB **and** scene numbering, absolute number, special flag; the parsed release (group, source, resolution, codecs, streaming service, edition); file size, duration and fps; and the hashes the provider's manifest requests. _Source: design/metadata-and-subtitles.md §B5._

**SUB-020** (MUST) The host **MUST** compute each requested file hash (OpenSubtitles hash, napiprojekt MD5 of the first 10 MiB, and any other declared in a manifest) once per file content and reuse it across providers. _Source: design/metadata-and-subtitles.md §B5._

### Matching

**SUB-021** (MUST) Matching **MUST** run in the host. Plug-ins **MUST** return candidate facts only. The host **MUST** parse candidate release names with the shared release parser (D1c, `04-evaluation.md`) and compare them with VideoFacts to produce the match set. Title comparison **MUST** use normalised titles plus aliases and scene titles. _Source: D1c; design/metadata-and-subtitles.md §B6._

**SUB-022** (MUST) A candidate's claims **MUST** be accepted only if the provider's manifest declares them in `capabilities.claims`. _Source: design/metadata-and-subtitles.md §B6._

**SUB-023** (MUST) A candidate's `hash_verifiable`, `hi_verifiable` and `forced_verifiable` properties **MUST** come from the manifest, and a candidate whose HI or forced flag is unknown (null) **MUST** be treated as not verifiable for that flag. _Source: design/metadata-and-subtitles.md §B10._

### Scoring (preserved exactly)

**SUB-024** (MUST) Scores **MUST** use exactly these weights:

| Attribute | Episode | Movie |
|---|---|---|
| hash | 359 | 179 |
| series | 160 | — |
| title | — | 60 |
| year | 90 | 40 |
| season | 30 | — |
| episode | 30 | — |
| source | 25 | 30 |
| edition | — | 30 |
| release_group | 20 | 15 |
| audio_codec | 1 | 1 |
| resolution | 1 | 1 |
| video_codec | 1 | 1 |
| hearing_impaired | 1 | 1 |
| streaming_service | 1 | 1 |
| **max** (sum without hash) | **360** | **180** |

Hash **MUST** always equal max − 1. Scores are not user-configurable. _Source: bazarr `custom_libs/subliminal_patch/score.py:31-64`; design/metadata-and-subtitles.md S1._

**SUB-025** (MUST) Hash handling **MUST** be:
- If the provider is hash-verifiable and `hash` is matched: for a movie, or for an episode that is not a special, the hash is **trusted** if the match set contains all of {series, season, episode, source} (episode) or {video_codec, source} (movie), and the match set is then reduced to {hash}. Otherwise `hash` is removed and the other matches are kept. For a TV special, the hash is neither validated nor removed.
- If the provider is not hash-verifiable and `hash` is matched, the match set is reduced to {hash}.

_Source: bazarr `score.py:101-130`; design/metadata-and-subtitles.md S2._

**SUB-026** (MUST) After hash handling, these equivalences **MUST** be added:
- episode: `title` ⇒ episode; `series_imdb_id` ⇒ series, year; `imdb_id` ⇒ series, year, season, episode; `tvdb_id` ⇒ series, year, season, episode, title; `series_tvdb_id` ⇒ series, year; for a special with title, series and year matched ⇒ season, episode;
- movie: `imdb_id` ⇒ title, year.

_Source: bazarr `score.py:138-200`; design/metadata-and-subtitles.md S3._

**SUB-027** (MUST) `hearing_impaired` **MUST** be added to the match set when the candidate's HI flag equals the want key's preference, where the preference is HI for `hi_mode = required` and non-HI for `accepted` and `excluded`. _Source: bazarr `core.py:552`, `score.py:148-155`; design/metadata-and-subtitles.md S4._

**SUB-028** (MUST) The score **MUST** be the sum of the weights in the final match set. A second value, `score_without_hash`, **MUST** be the sum over the original matches plus equivalences, excluding hash. Scores **MUST** be stored with `score_max`. _Source: bazarr `score.py`; design/metadata-and-subtitles.md S7._

**SUB-029** (MUST) A golden test suite porting Bazarr's scoring and selection tests, plus recorded provider fixtures, **MUST** be the acceptance gate for SUB-024 to SUB-036. _Source: design/metadata-and-subtitles.md §B6._

### Selection

**SUB-030** (MUST) For each want key, the host **MUST** keep only candidates whose language is equivalent to the key's language and whose forced flag equals the key's. _Source: design/metadata-and-subtitles.md §B6._

**SUB-031** (MUST) The host **MUST** drop candidates whose release info fails the profile's `must_contain` or `must_not_contain`, and candidates blocklisted for the item as (provider, provider subtitle id). _Source: design/metadata-and-subtitles.md §B6._

**SUB-032** (MUST) For `hi_mode ∈ {required, excluded}`, the host **MUST** skip a candidate whose HI flag is verifiable and does not match the key's preference (`hearing_impaired` not in the match set). For `accepted`, HI and non-HI candidates are both eligible. _Source: bazarr `core.py:604-609`; design/metadata-and-subtitles.md S4._

**SUB-033** (MUST) Candidates **MUST** be sorted by (score, score_without_hash) descending. Iteration **MUST** stop at the first candidate whose score is below `min_score = floor(score_max × minimum_score% / 100)`, with `minimum_score` defaulting to **90 %** for episodes (324) and **70 %** for movies (126). Both percentages **MUST** be settings. _Source: bazarr `core.py:580-590`, `subtitles/utils.py:73-87`, `config.py` `minimum_score`/`minimum_score_movie`; design/metadata-and-subtitles.md S6._

**SUB-034** (MUST) For episodes, a candidate **MUST** be skipped unless its original matches (before hash handling and equivalences) include season and episode and either series or imdb_id. The check **MUST** be waived when the provider is not hash-verifiable and `hash` is in the final match set. _Source: bazarr `core.py:611-622`; design/metadata-and-subtitles.md S6._

**SUB-035** (MUST) If downloading the chosen candidate fails, the host **MUST** try the next eligible candidate. _Source: design/metadata-and-subtitles.md S6._

**SUB-036** (MUST) Candidates flagged `ai_translated` or `machine_translated` **MUST** be excluded unless the provider's settings opt in. _Source: design/metadata-and-subtitles.md §B10 (manifest `flags`)._

### Post-processing

**SUB-037** (MUST) Post-processing **MUST** run in this order: decode → normalise → mods → sync (when gated in) → place → history → events. _Source: design/metadata-and-subtitles.md S12, §B7._

**SUB-038** (MUST) If a plug-in returns an archive, the host **MUST** extract it (zip; RAR with a pure-Go reader; 7z with a pure-Go reader) with the size and entry-count limits of `12-platform.md` PLT-147 (`plugins.archive_max_bytes`, `plugins.archive_max_entries`), and **MUST** pick the entry by matching entry names against the file's episode with the shared parser. No native unrar or 7z binaries are downloaded. _Source: design/metadata-and-subtitles.md §B7._

**SUB-039** (MUST) The host **MUST** detect the encoding and re-encode to UTF-8 unless the "keep encoding" setting is on, and **MUST** convert to SRT unless the profile's `original_format` is set. _Source: design/metadata-and-subtitles.md §B7._

**SUB-040** (MUST) The host **MUST** provide native Go ports of subzero's mods: common, remove_tags, reverse_rtl, fix_uppercase, emoji, OCR_fixes (with per-language dictionaries), remove_HI, color, shift_offset, change_FPS, two_point_fit. _Source: design/metadata-and-subtitles.md §B7._

**SUB-041** (MUST NOT) `remove_HI` **MUST NOT** be applied to a subtitle selected for a `hi_mode = required` want key. _Source: design/metadata-and-subtitles.md §B7._

**SUB-042** (MUST) The OCR_fixes dictionaries **MUST NOT** be ported until their licence and provenance are confirmed. [OPN-052] _Source: open-issues Group C; design/metadata-and-subtitles.md §6.2 item 7._

**SUB-043** (MUST) The subtitle file **MUST** be written through `06-import.md`'s `PlaceSubtitle(file, bytes, language, hi, forced, format)`, which names the sidecar `name.<lang>[.hi|.cc|.sdh][.forced].<ext>` (HI suffix per setting, single-language mode, subfolder mode), replaces an upgraded predecessor atomically, emits `extra.written`, and returns the subtitle record. _Source: design/metadata-and-subtitles.md S11, §B7; design/import.md §3.6, §3.9._

### Synchronisation

**SUB-044** (MUST) Synchronisation **MUST** be behind a `Syncer` interface with two implementations: **alass** (the default when present) and **ffsubsync** (optional, for parity). [OPN-051] _Source: A6; open-issues Group C; design/metadata-and-subtitles.md §B7._

**SUB-045** (MUST) Automatic sync **MUST** be off by default. When on, it **MUST** run for every downloaded subtitle unless the threshold gate is enabled; with the gate enabled it **MUST** run only when the score percentage is **≤** the threshold (default 90 % for episodes, 70 % for movies). Sync **MUST** also be available on demand. _Source: bazarr `config.py` `subsync.*`, `subtitles/sync.py:53`; design/metadata-and-subtitles.md S12._

**SUB-046** (MUST) The sync reference **MUST** be an embedded text subtitle of any language if one exists, else the audio track. Framerate correction **MUST** be off by default. The maximum search offset **MUST** default to 60 s. _Source: design/metadata-and-subtitles.md §B7; bazarr `subsync.no_fix_framerate`, `subsync.max_offset_seconds`._

**SUB-047** (MUST) A quality checker **MUST** reject a sync result whose offset exceeds 30 s, whose framerate deviation exceeds 0.1, or whose tool confidence is below the minimum score (default 0.0). A rejected sync **MUST** keep the unsynced subtitle and record the reason. Checker exclusions per provider and per language **MUST** be settings. _Source: bazarr `config.py` `subsync.quality_*`, `subsync.checker.*`; design/metadata-and-subtitles.md §B7._

**SUB-048** (MUST) Sync and extraction **MUST** run as **capability-routed jobs** that declare the tools they require (`alass`, `ffsubsync`, `ffmpeg`); only nodes advertising those tools claim them (`12-platform.md`). alass is bundled where its licence allows; ffsubsync is optional. _Source: A6; design/metadata-and-subtitles.md §B12._

### Upgrades

**SUB-049** (MUST) The upgrade sweep **MUST** select subtitle records with `origin = downloaded` (plus `manual`, `uploaded` and `translated` when "upgrade manual" is on), created within `days_to_upgrade` (default 7, range 0–30), with `score < score_max − 3`, whose want key is still wanted with the same HI mode. _Source: bazarr `upgrade.py:260-261,324,406`, `config.py` `days_to_upgrade_subs`; design/metadata-and-subtitles.md S7, §B8._

**SUB-050** (MUST) For each selected record, the sweep **MUST** enqueue a search for that one want key with `min_score = old score + 1`. A winner **MUST** replace the old file atomically through `PlaceSubtitle`, and history **MUST** record the chain. _Source: bazarr `download.py:66-68`; design/metadata-and-subtitles.md §B8._

**SUB-051** (MUST) Upgrades **MUST** compare scores only against the same `score_max`. _Source: design/metadata-and-subtitles.md S7._

**SUB-052** (MUST) On a media upgrade (`file.upgraded`), subtitles with `origin = downloaded` on the replaced file **MUST** be dropped and the new file searched. Subtitles with `origin ∈ {uploaded, manual}` **MUST** be carried to the new file and re-synced against it. [OPN-050] _Source: open-issues Group C; design/metadata-and-subtitles.md §6.2 item 5, §B5._

### Throttling and blocklist

**SUB-053** (MUST) Plug-in typed errors **MUST** map to provider back-off durations, defaulting to Bazarr's table:

| Error code | Default back-off |
|---|---|
| `too_many_requests` | 1 h |
| `download_limit` | 3 h |
| `unavailable` | 20 min |
| `api_throttled` | 10 min |
| `parse` | 6 h |
| `timeout` | 1 h |
| `auth`, `config` | 12 h |
| `ip_blocked`, `not_found`, any other | 10 min |

A plug-in's manifest **MAY** override any duration, including a daily reset at a local time (`reset: {at: "00:00", tz: "<IANA zone>"}`). _Source: bazarr `get_providers.py:58-141, 409-446`; design/metadata-and-subtitles.md S13, §B9._

**SUB-054** (MUST) The error codes `too_many_requests`, `unavailable`, `api_throttled` and `timeout` **MUST** throttle only after 5 occurrences within 120 s; other codes **MUST** throttle immediately. _Source: bazarr `get_providers.py:472-500`._

**SUB-055** (MUST) Throttle state, quotas and failure counts **MUST** live in the database so every node sees the same state: back-off as the `cooldown_until` of the provider's rate key `subtitle:<provider_id>` and quotas as its quota buckets in the shared rate limiter (`12-platform.md` PLT-126–PLT-132), and the failure counts of SUB-054 on the `SubtitleProvider` row. _Source: design/metadata-and-subtitles.md §B9._

**SUB-056** (MUST) A `blocklist_this` error **MUST** blocklist (provider, provider subtitle id) for the item automatically. A manual blocklist **MUST** delete the subtitle file, record (provider, id) for the item and re-search. _Source: design/metadata-and-subtitles.md S13, §B9._

### Plug-in contract (WASM, D15)

**SUB-057** (MUST) Subtitle providers **MUST** be WASM plug-ins of kind `subtitle_provider` running in the plug-in host of `12-platform.md` (wazero, in-process, no filesystem, no environment, no sockets). Memory caps (PLT-149; a subtitle provider's manifest may request up to 64 MiB), response caps (PLT-147), per-call deadlines (PLT-150: search 20 s, download 60 s), instance pooling, signing and auto-disable follow `12-platform.md`; this file sets no other limits. _Source: D15; design/runtime.md §3.12._

**SUB-058** (MUST) A subtitle-provider plug-in **MUST** implement these operations. Their byte-level binding (export names, buffer convention, encoding) is the shared plug-in ABI of `12-platform.md`; v1 encodes messages as JSON. [OPN-055] [OPN-065]

| Operation | Request → response |
|---|---|
| `abi` | → ABI version (1) |
| `describe` | → manifest |
| `configure` | settings and credentials → `{ok, field_errors}` |
| `search` | `SearchRequest` → `SearchResponse` |
| `download` | `DownloadRequest` → `DownloadResponse` |
| `test` (optional) | → connectivity and credential check |

_Source: design/metadata-and-subtitles.md §B10; design/runtime.md §3.12._

**SUB-059** (MUST) The manifest **MUST** declare: `id`, `abi`, `version`, `license`, `kind: subtitle_provider`, `media` (`movie`, `episode`), `languages` (list or `any`), `hosts` (egress allow-list), `capabilities` (`hashes`, `claims`, `hi_verifiable`, `forced_verifiable`, `hash_verifiable`, `archives`, `cloudflare`, `captcha`), `rate` (`rps`, `burst`), `quotas` (name, period, limit or limit-from-setting, reset), `throttle` overrides, `credentials` (name, kind, `project_default`), `settings_schema` (JSON Schema) and `flags` (`ai_translated_opt_in`, `machine_translated_opt_in`). The administrator **MUST** approve the declared hosts and capabilities on install and on any change. _Source: design/metadata-and-subtitles.md §B10._

**SUB-060** (MUST) Messages **MUST** have at least these fields:
- `SearchRequest{video: VideoFacts, languages: [{code, forced_mode: none|also|only, hi_preference: required|accepted|excluded}], limit}`;
- `Candidate{id, language, hi?, forced?, format, release_names[], season?, episode?, year?, fps?, claims[], uploader, page_url, ai_translated, machine_translated, download_token}` (`download_token` is opaque and echoed back);
- `SearchResponse{candidates[], error?}`, `DownloadRequest{candidate}`, `DownloadResponse{filename, content, is_archive, error?}`;
- `PluginError{code, retry_after_s?, reset_at?, message}` with `code ∈ {too_many_requests, download_limit, api_throttled, unavailable, parse, timeout, auth, config, not_found, ip_blocked, blocklist_this}`.

_Source: design/metadata-and-subtitles.md §B10._

**SUB-061** (MUST) The host **MUST** provide these host functions of `12-platform.md` PLT-147 to subtitle-provider plug-ins, each checked against the manifest and grants:

| Function | Behaviour |
|---|---|
| `http_do` | Through the core HTTP client: host allow-list, SSRF guard, per-host rate limit and the plug-in's rate and quotas, proxy settings, TLS verification always on, redirects only within the allow-list, the response cap of PLT-147, per-instance cookie jar persisted in the KV store; the FlareSolverr path only with `cloudflare: true` and a configured solver |
| `kv_get` / `kv_put(ttl)` / `kv_delete` | Namespaced per provider instance, stored in the database (shared across nodes) |
| `archive_list` / `archive_read` | Only with `archives: true`; lets a plug-in pick an entry itself (the host otherwise extracts, SUB-038); limits per PLT-147 |
| `captcha_solve(kind, site_key, page_url)` | Only with `captcha: true` and a configured captcha-solving service |
| `log`, `now_ms`, and the default functions of PLT-147 (`regex_match`, `regex_replace`, `html_select`, `parse_release`) | As defined there |

_Source: design/metadata-and-subtitles.md §B10; design/runtime.md §3.12._

**SUB-062** (MUST) Credentials **MUST** be delivered to a plug-in only by the host at configure time, and only those the manifest declares. Credentials **MUST NOT** be compiled into modules. Project application keys (for example an OpenSubtitles.com key, `project_default: true`) **MUST** be held by the host under `08-metadata.md` MET-086 and be user-overridable. _Source: D3b extension; design/metadata-and-subtitles.md §6.1 item 2._

**SUB-063** (MUST) First-party plug-ins **MUST** live in the repository (`plugins/subtitles/<id>`), be built to `.wasm` in CI, and be embedded in the binary. Third-party and updated plug-ins come from a plug-in index pinned by sha256. The licensing of third-party plug-ins **MUST** be in scope of the D5b legal review. _Source: design/metadata-and-subtitles.md §B10; D20; open-issues B12._

**SUB-064** (MUST) Each plug-in **MUST** ship a conformance kit (recorded HTTP fixtures and expected candidates) that runs in CI. _Source: design/metadata-and-subtitles.md §B10._

### Provider port waves

**SUB-065** (MUST) Providers **MUST** be ported in these waves:

| Wave | Providers |
|---|---|
| Built-in (host) | Embedded tracks (SUB-004); replaces Bazarr's `embeddedsubtitles` |
| 1, at the phase-2 launch | opensubtitlescom, subdl, subsource, gestdown, jimaku, animetosho, napiprojekt |
| 1b, JSON regional | subsro, regielive, subtis, subx, assrt, betaseries, bayflix, vladoonmooo, tsukihime, hdbits, subsarr, subsdump |
| 2, HTML scrapers (need the archive helper and fixtures) | yifysubtitles, subf2m, animesubinfo, napisy24, legendasnet, subtitulamostv, subclub, subssabbz, subsunacs, yavkanet, titrari, subtitrarinoi, karagarga, ktuvit, xsubs, pipocas, animekalesi, subtitriid, nekur, wizdom, subsynchro, subtitlecat (opt-in; machine translations) |
| 3, need Cloudflare or captcha | addic7ed, titlovi, zimuku, subs4series, greeksubs, prijevodionline, soustitreseu, legendasdivx, turkcealtyaziorg, titulky, avistaz/cinemaz |

_Source: design/metadata-and-subtitles.md §B11; verification/outdated-and-security.md (nekur, subsynchro and yifysubtitles are alive)._

**SUB-066** (MUST) Before a wave-2 or wave-3 provider is ported, its site **MUST** be checked to be alive. wizdom's status is unclear and **MUST** be confirmed before porting. _Source: verification/outdated-and-security.md §"Dead Bazarr subtitle providers"._

**SUB-067** (MUST NOT) These providers **MUST NOT** be ported: legendastv, argenteam, thesubdb, subscenter, hosszupuska, shooter (dead sites), podnapisi (removed upstream), bsplayer, greeksubtitles, subs4free, and the hidden supersubtitles and tvsubtitles unless requested. _Source: verification/outdated-and-security.md; design/metadata-and-subtitles.md §B11._

### HA placement

**SUB-068** (MUST) Subtitle work **MUST** be placed as follows. No subtitle component uses a lease (A1).

| Component | Placement |
|---|---|
| Wanted sweep, upgrade sweep, plug-in index refresh | Schedule rows claimed by any node with `SKIP LOCKED`, dedupe `WhilePendingOrRunning` |
| Search and download jobs, interactive search | Any node; plug-ins compiled per node |
| Provider status, quotas, KV store, blocklist | Database |
| Sync and extraction | Capability-routed tool jobs (SUB-048); media paths are shared storage |
| File placement | `06-import.md`'s port |

Search jobs **MUST** use the dedupe key `subtitle:search:<media_file>:<want_key>`. _Source: A1, A6; D20 (D14 amended); design/metadata-and-subtitles.md §B12._

### Health and metrics

**SUB-069** (MUST) Health issues **MUST** be raised for: a provider auto-disabled, or throttled because of an `auth` or `config` error; a provider credential failing validation; no enabled provider for a profile language; a configured Syncer unavailable on every node. _Source: design/metadata-and-subtitles.md §B9; design/runtime.md §3.8._

**SUB-070** (MUST) Metrics **MUST** include, per provider: calls, errors per code, throttle state, candidates returned, downloads and quota use; and overall: wanted keys, search backlog, sync outcomes. _Source: D7._

---

## Data model

The `Subtitle` record is owned by `02-domain-model.md` (DOM-110; field requirements in SUB-002). This file owns the tables below.

```go
SubtitleProfile {
  name              string unique
  cutoff_item       -> SubtitleProfileItem?
  must_contain      []string
  must_not_contain  []string
  original_format   bool
  edges: items <- SubtitleProfileItem
}
SubtitleProfileItem {
  edges: profile -> SubtitleProfile
  position          int
  language          string        // BCP 47-style
  forced            bool
  hi_mode           enum(required, accepted, excluded)   // CHECK: forced ⇒ hi_mode ≠ required
  audio_rule        enum(always, skip_if_audio_is, only_if_audio_is)
  audio_language    string?
  unique (profile_id, position)
}
LanguageEquivalence { from_lang, to_lang string; unique (from_lang, to_lang) }

// Profile assignment lives on the Title (02-domain-model.md): Title.subtitle_profile -> SubtitleProfile?

SubtitleSearchState {
  edges: media_file -> MediaFile (cascade)
  want_key          string        // "<lang>:<forced>:<hi_mode>"
  first_missing_at, last_attempt_at, next_attempt_at time?
  attempts          int
  unique (media_file_id, want_key)
  index (next_attempt_at)
}
SubtitleBlocklist {
  edges: item -> Item
  provider          string
  provider_sub_id   string
  reason            enum(manual, blocklist_this)
  unique (item_id, provider, provider_sub_id)
}
SubtitleProvider {
  edges: plugin -> Plugin (12-platform.md)
  enabled           bool
  priority          int
  settings          json          // validated against the manifest's settings_schema; declared credentials are write-only secret fields (D16) [OPN-201]
  failure_counts    json          // per error code, within the SUB-054 window
}
// Back-off and quotas: the shared rate limiter (12-platform.md), rate key "subtitle:<provider_id>".
// Plug-in KV: the shared plug-in KV store (12-platform.md), namespaced per SubtitleProvider.
```

---

## Flows

### F1. New episode wanting HI English and forced English

Profile: item 1 `eng, forced=false, hi_mode=required`; item 2 `eng, forced=true, hi_mode=accepted`; no cutoff; minimum score 90 %.

1. `file.imported` for S02E05, release `Show.S02E05.1080p.AMZN.WEB-DL.DDP5.1.H.264-GRP`. The probe shows one embedded English SubRip track with no dispositions.
2. Wanted: the embedded track is `eng, hi=false, forced=false`, so wanted = {`eng:false:required`, `eng:true:accepted`}.
3. A search job builds VideoFacts, computes the OpenSubtitles hash once, and calls each enabled, non-throttled provider once with `[{eng, forced_mode: also}]`.
4. Candidates: A (HI, hash claim, same release), B (non-HI, same release), C (forced, non-HI, same release), D (HI, other group).
5. Scoring: A's hash is trusted (series, season, episode, source match) → {hash} + hearing_impaired = 360. B = 160+30+30+90+25+20+4 = 359 with no HI match. D = 160+30+30+90+25+4+1 = 340.
6. Selection: `eng:false:required` skips B (verifiable, not HI); A (360 ≥ 324) wins. `eng:true:accepted` keeps only C; C scores 359 + 1 (non-HI preferred) = 360 and wins.
7. Post-processing: both are UTF-8 SRT; mods run without `remove_HI` for A; auto-sync is off by default.
8. `PlaceSubtitle` writes `Show - S02E05.en.sdh.srt` and `Show - S02E05.en.forced.srt`; two records (scores 360/360); history and events.
9. Failure paths: a provider returns `too_many_requests` → counted, and after 5 within 120 s the provider is throttled for its back-off; A's download fails → the next eligible candidate for that key is tried; no forced candidate exists → the key stays wanted and the sweep retries it, weekly after 3 weeks.

### F2. Upgrade

1. The upgrade sweep finds a downloaded record with score 340 of 360, created 2 days ago (340 < 357).
2. It enqueues a search for that want key with `min_score = 341`.
3. A candidate scoring 360 replaces the file atomically; history records `upgraded_from`.
4. After 7 days the record is no longer eligible.

### F3. Media upgrade

1. `file.upgraded`: the new file replaces the old one.
2. Downloaded subtitles of the old file are dropped (import retires the sidecars); uploaded and manual subtitles are carried over and a sync job is enqueued against the new file.
3. The new file's wanted set is computed and searched.
4. Failure path: no node advertises a Syncer → the carried subtitles are kept unsynced and a health issue is raised.

### F4. Sync with quality check

1. A downloaded subtitle scores 88 % with auto-sync and the threshold gate on (≤ 90 %).
2. A tool job requiring `alass` is claimed by a node that has it; the reference is an embedded text track, else the audio.
3. The result's offset is 42 s (> 30 s): the checker rejects it; the unsynced subtitle stays and the reason is recorded.

---

## Non-goals

| Excluded | Reason |
|---|---|
| Bazarr's mirror of Sonarr/Radarr, SignalR clients, path mappings, `Rescan*` calls | The subsystem reads the library directly (D19) |
| A second media probe | One shared probe (`06-import.md`) |
| Per-provider guessit matching | Host-side matching with the shared parser (D1c) |
| Python providers or wrappers | D15; providers are ported to WASM |
| Custom score overrides | Removed upstream; SUB-024 weights are fixed |
| Bazarr API, numeric history codes | D7 |
| Translation (Google, Gemini, Lingarr) and Whisper generation in the first subtitle release | Need audio extraction; later as host-side generators producing `translated`/`generated` records at a fixed default score of 50 |
| Native Go alignment | Later option; phase 2 uses alass/ffsubsync |
| Dead providers | SUB-067 |
| Subtitle profiles per slot | One profile per Title (SUB-011); a per-slot override may come later |
| Telemetry (Bazarr's GA4) | D3 |

---

## Traceability

| Reference | Preserved | Changed (D19) |
|---|---|---|
| Bazarr scoring (`subliminal_patch/score.py`) | Weights (episode 360, movie 180), hash = max − 1, hash-trust rule, equivalences, HI +1 | Matching runs in the host with the shared parser |
| Bazarr selection (`subliminal_patch/core.py`) | Sort by (score, score without hash); stop below 90 %/70 %; series/season/episode check unless hash unverifiable; HI skip for verifiable providers; fallback to next candidate | Runs per want key |
| Bazarr profiles | Ordered items, forced as a separate unit, HI required/normal/excluded, audio rules, cutoff, must-contain filters, `language_equals`, custom codes | "normal" named `accepted`; wanted is computed, not stored |
| Bazarr upgrades and adaptive search | 7-day window, `score < max − 3`, `min_score = old + 1`; 3-week/1-week adaptive search; 6 h wanted and 12 h upgrade sweeps | Upgrade chain kept in shared history |
| Bazarr throttling | Default back-off table, count-before-throttle for transient errors, per-provider overrides, daily resets, blocklist by (provider, id) | State in the DB via the shared rate limiter |
| Bazarr sync | Threshold gate, max offset, quality checker | alass default; capability-routed tool jobs |
| subzero mods | The mod set | Native Go ports; OCR dictionaries pending licence check |
| Sonarr/Radarr extras | Sidecar naming `name.<lang>[.hi|.cc|.sdh][.forced].<ext>` | One subtitle table shared with import |
