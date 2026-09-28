# Recyclarr — Analysis Report

Source analysed: `submodules/recyclarr` @ `eadd0ce7` (2026-09-24), release line v8.x (latest tag `v8.7.2`, 2026-09-02).
All paths below are relative to `submodules/recyclarr/` unless stated. Read together with [profilarr.md](profilarr.md) and [comparison.md](comparison.md).

---

## 1. Overview

**Purpose.** Recyclarr is a headless CLI that reads **TRaSH Guides** JSON data (custom formats, CF groups, quality profiles, quality sizes, naming) from a git repo and pushes it into Sonarr/Radarr according to a user YAML file. It has **no UI, no database and no authoring layer**: users do not edit CFs, they reference upstream objects by `trash_id` and override a few fields. State is a small per-instance JSON file that maps `trash_id` to the Arr object id (`README.md`, `AGENTS.md`, `docs/architecture/*`).

**Stack / versions** (`Directory.Build.props`, `Directory.Packages.props`, `Dockerfile`):

| Layer | Choice |
|---|---|
| Language/runtime | C# 14, .NET 10 (`net10.0`), nullable, warnings-as-errors |
| CLI | Spectre.Console.Cli 0.55 (tables, live progress) |
| DI | Autofac 9 (one module per library) |
| HTTP | Refit 15 + **Refitter**-generated clients from vendored Sonarr/Radarr `openapi.json` (`src/Recyclarr.Api.{Sonarr,Radarr}/*.refitter`, ADR-006) |
| Mapping | Riok.Mapperly (DTO↔domain, `src/Recyclarr.Core/ServarrApi/*/*Mapper.cs`) |
| Config | YamlDotNet 18 + FluentValidation 12 + JSON Schema (`schemas/`) |
| JSON | System.Text.Json (+ JsonDiffPatch for previews) |
| Git | external `git` CLI via CliWrap (`src/Recyclarr.Core/VersionControl/GitRepository.cs`) |
| Logging | Serilog (console + file) |
| Misc | System.Reactive (sync events), FNV hash (state dir names), SuperLinq |
| Tests | NUnit 4, NSubstitute, AutoFixture, Testcontainers E2E (real Sonarr/Radarr) |
| Packaging | Single-file publish; Docker `dotnet/runtime:10.0-alpine` + `supercronic` + `git` + `tini` |

**Architecture** (`AGENTS.md`, `docs/architecture/sync-pipeline-architecture.md`, ADR-005/008):

```
Recyclarr.Cli  (commands, Processors, Pipelines, Migration)
  └─> Recyclarr.Core (Config/, Settings/, ResourceProviders/, TrashGuide/, SyncState/,
                     Servarr/ = ports, ServarrApi/ = gateways, Compatibility/, Notifications/)
        └─> Recyclarr.Api.Sonarr / Recyclarr.Api.Radarr (Refitter output, Generated/ not in VCS)
```

- **Resource providers** (git or local dir) → **resource queries** (dedupe by `trash_id`, last provider wins) → **Plan phase** (validate config vs guide, build planned CFs/QPs) → per-pipeline **ApiFetch → Transaction → Preview → ApiPersistence**.
- Pipelines: CustomFormat, QualityProfile (depends on CF), QualitySize, MediaNaming (service-affine), MediaManagement. Ordered by topological sort in `CompositeSyncPipeline` (`src/Recyclarr.Cli/Processors/Sync/CompositeSyncPipeline.cs`). CF pipeline is all-or-nothing; a failed CF pipeline skips QP.
- **Port + gateway** (ADR-005): pipelines talk to `ICustomFormatService`, `IQualityProfileService`… (`src/Recyclarr.Core/Servarr/`); gateways (`src/Recyclarr.Core/ServarrApi/`) **stash the fetched DTO** and merge domain changes onto it before PUT, so unknown fields round-trip.
- ADR-008 plans a second host (HTTP server with scheduler, for Kubernetes); not implemented yet.

**LOC** (non-generated `.cs`):

| Area | LOC |
|---|---|
| `src/Recyclarr.Cli` | ~9.8k (Pipelines ~4.6k) |
| `src/Recyclarr.Core` | ~10.4k |
| `src/Recyclarr.Api.*` | 0 hand-written (two ~300 KB OpenAPI specs, generated at build) |
| `tests/` | ~17.7k (Cli 9.8k, Core 6.2k, E2E 0.9k) |
| `docs/` (architecture, ADR/PDR, reference) | ~2.1k lines markdown |

About 20k LOC of product code, about a sixth of Profilarr's ~131k in `src`, because it has no UI, DB, parser or authoring.

---

## 2. Domain model

There are four data sets: (A) the user's **YAML config**, (B) **settings** YAML, (C) upstream **TRaSH Guides JSON** (plus **config-templates** repo), and (D) **state files**. There is no DB.

### 2.1 Config YAML (`recyclarr.yml`, `configs/*.yml`; `src/Recyclarr.Core/Config/Parsing/ConfigYamlDataObjects*.cs`, `schemas/config-schema.json`, `schemas/config/*.json`)

```yaml
sonarr|radarr:
  <instance_name>:                    # dict key; multiple instances per service
    base_url: http://sonarr:8989       # or implicit from secrets: <instance>_base_url
    api_key: !secret sonarr_key        # YAML tags: !secret (secrets.yml), !env_var NAME [default], !file path
    delete_old_custom_formats: bool    # default false
    include:                           # merged BEFORE this instance's own keys (root wins)
      - template: <config-templates include id>
      - config: <path, relative to includes/>
    custom_formats:
      - trash_ids: [<32-hex>, ...]
        score: int                     # entry-level default (v8.6)
        assign_scores_to:
          - name: <profile name> | trash_id: <guide QP trash_id>
            score: int                 # per-profile override
    custom_format_groups:
      skip: [<cf-group trash_id>]      # opt out of auto-synced default groups
      add:
        - trash_id: <cf-group trash_id>
          select_all: bool | select: [<cf trash_id>] | exclude: [<cf trash_id>]
          assign_scores_to: [{name|trash_id}]   # default: all matching guide-backed QPs
    quality_profiles:
      - trash_id: <guide QP trash_id>  # optional → "guide-backed"; same id may repeat with different names
        name: str                      # required if no trash_id; overrides guide name
        score_set: str                 # key into CF trash_scores; default = guide trash_score_set or "default"
        reset_unmatched_scores: {enabled: bool=false, except: [cf names], except_patterns: [regex]}
        upgrade: {allowed: bool, until_quality: str, until_score: int}
        min_format_score: int
        min_upgrade_format_score: int
        quality_sort: top|bottom       # where unlisted qualities go
        qualities:                     # top = highest priority
          - name: str
            enabled: bool=true
            qualities: [str]           # non-empty → quality group
    quality_definition:
      type: movie|series|anime|sqp-…   # guide quality-size "type"
      preferred_ratio: 0.0-1.0         # recompute preferred = min + (max-min)*ratio
      qualities: [{name, min, max: num|unlimited, preferred: num|unlimited}]
    media_naming:                      # values are KEYS into guide naming JSON, not format strings
      # sonarr:
      series: <key>   season: <key>
      episodes: {rename: bool, standard: <key>, daily: <key>, anime: <key>}
      # radarr:
      folder: <key>
      movie: {rename: bool, standard: <key>}
    media_management:
      propers_and_repacks: prefer_and_upgrade|do_not_upgrade|do_not_prefer
```

- Model types after parsing: `src/Recyclarr.Core/Config/Models/ServiceConfiguration.cs` (`QualityProfileConfig`, `ResetUnmatchedScoresConfig`, `QualityDefinitionConfig`, `QualitySizeValue` Numeric/Unlimited).
- Include merge rules (`Config/Parsing/PostProcessing/ConfigMerging/ServiceConfigMerger.cs`, `IncludePostProcessor.cs`): includes are aggregated in order, then the instance config is merged on top. CFs are normalized to one (profile, score) per entry, and B's trash_ids are subtracted from A's for the same profile. QPs merge on the composite key (trash_id, name). Scalars: last non-null wins. CF group `skip` is a union.
- Config filters (`Config/Filtering/`): duplicate instance names, non-existent `-i` names, invalid instances, and **split instances** (two instance names with the same `base_url` → error, since state is keyed by URL).
- Deprecated/removed keys produce warnings (`ConfigDeprecationPostProcessor`): `replace_existing_custom_formats`, `custom_formats[].quality_profiles` (removed in v8).

### 2.2 Settings YAML (`settings.yml`; `src/Recyclarr.Core/Settings/Models/*.cs`, `schemas/settings-schema.json`)

| Key | Meaning |
|---|---|
| `resource_providers[]` | `{name, type: trash-guides|config-templates|custom-formats, replace_default, service (custom-formats only), clone_url + reference + cache_limit (git) | path (local)}` |
| `enable_ssl_certificate_validation` | default true |
| `git_path` | path to git executable |
| `log_janitor.max_files` | default 20 |
| `notifications` | `verbosity: minimal|normal|detailed|verbose`, `apprise: {mode: stateful|stateless, base_url, key, tags, urls[]}` |

### 2.3 TRaSH Guides JSON schemas (as consumed; `src/Recyclarr.Core/ResourceProviders/Domain/*.cs`, `src/Recyclarr.Core/TrashGuide/*`, fixtures in `tests/Recyclarr.EndToEndTests/Fixtures/trash-guides-override/`)

**Identification.** Every CF, CF group and QP has a **`trash_id`**: a 32-char lowercase hex string (MD5-like), stable forever (`docs/architecture/trash-id-state-system.md` "Immutable Trash IDs"). Quality-size files are identified by their `type` string. Naming files are unkeyed and merged. Files are found through a root **`metadata.json`**, not by filename:

```json
{ "json_paths": {
    "radarr": { "custom_formats": ["docs/json/radarr/cf"], "qualities": ["docs/json/radarr/quality-size"],
                "naming": ["docs/json/radarr/naming"], "quality_profiles": ["docs/json/radarr/quality-profiles"],
                "custom_format_groups": ["docs/json/radarr/cf-groups"], "quality_profile_groups": [...] },
    "sonarr": { ...same keys... } } }
```
(`TrashGuide/RepoMetadata.cs`, `tests/Recyclarr.Core.TestLibrary/Data/TrashGuides/metadata.json`). Every `*.json` under each listed dir is loaded recursively (`ResourceProviders/Infrastructure/TrashGuidesStrategy.cs:GlobJsonFiles`). Upstream also publishes `metadata.schema.json` and `cf-groups.schema.json`. Unknown JSON fields are ignored.

**Custom format** (`CustomFormatResource.cs`): the file is the Arr API CF payload plus `trash_*` keys.
```json
{ "trash_id": "0f12c086e289cf966fa5948eac571f44",
  "trash_scores": { "default": 100, "anime-sonarr": 50, "sqp-1-1080p": 1750, ... },  // score sets
  "trash_regex": "https://regex101.com/r/…",  "trash_description": "…",               // ignored by Recyclarr
  "name": "Hybrid",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    { "name": "Hybrid", "implementation": "ReleaseTitleSpecification",
      "negate": false, "required": false,
      "fields": { "value": "<.NET regex>" } } ] }
```
- `fields` is an **object** in guide JSON but an **array** `[{name, value}]` in the API. `FieldsArrayJsonConverter` accepts both. `FieldValue` normalizes string/int/double/bool.
- Enum-valued specs (Source, Resolution, IndexerFlag, Language, QualityModifier, ReleaseType) carry **raw Arr integer ids** in `fields.value`. The guide is therefore per-app (`radarr/cf` vs `sonarr/cf`), and ids are not portable across apps.
- `trash_scores` keys are **score sets**; `default` is the fallback (`list score-sets` enumerates them).

**CF group** (`CfGroupResource.cs`, PDR-001/005/006/007):
```json
{ "trash_id": "…", "name": "[Audio] Audio Formats", "trash_description": "…",
  "default": "true",                              // STRING; "true" ⇒ auto-sync for matching QPs
  "custom_formats": [ { "name": "TrueHD ATMOS", "trash_id": "…", "required": true, "default": false } ],
  "quality_profiles": { "include": { "<QP name>": "<QP trash_id>" } } }   // was "exclude" (fail-open) before PDR-001
```
Upstream quirk: a synthetic "Base Profile" (QP group "TEST") exists only so every group has at least one include (PDR-007).

**Quality profile** (`QualityProfileResource.cs`):
```json
{ "trash_id": "…", "name": "HD Bluray + WEB", "trash_score_set": "default", "trash_url": "…",
  "trash_description": "…", "group": 1,
  "upgradeAllowed": true, "cutoff": "Bluray-1080p",          // cutoff by quality/group NAME
  "minFormatScore": 0, "cutoffFormatScore": 10000, "minUpgradeFormatScore": 1,
  "language": "Original",                                   // Radarr only; name, resolved via GET /language
  "items": [ { "name": "WEB 1080p", "allowed": true, "items": ["WEBDL-1080p","WEBRip-1080p"] },
             { "name": "Bluray-1080p", "allowed": true } ],   // treated top-to-bottom (PDR-002)
  "formatItems": { "<CF name>": "<CF trash_id>" } }          // CFs to score with trash_score_set
```
Quality names are Arr display names (e.g. Radarr `Remux-1080p` vs Sonarr `Bluray-1080p Remux`). There are no quality ids.

**Quality profile group** (`QualityProfileGroupResource.cs`): `{ "name": "Standard", "profiles": { "<QP name>": "<trash_id>" } }` (PDR-004; used only for listing).

**Quality size** (`QualitySizeResource.cs`, `TrashGuide/QualitySize/QualityItem.cs`):
```json
{ "trash_id": "…", "type": "movie",        // Radarr: movie, sqp-streaming, sqp-uhd; Sonarr: series, anime
  "qualities": [ { "quality": "Bluray-1080p", "min": 50.4, "preferred": 1999, "max": 2000 } ] }  // MB/min
```
When several providers define the same `type`, the last one wins (`QualitySizePlanComponent.cs`).

**Naming** (`RadarrMediaNamingResource.cs`, `SonarrMediaNamingResource.cs`, test data `tests/Recyclarr.Core.Tests/IntegrationTests/TrashGuide/Data/*naming*.json`):
```json
// radarr
{ "folder": { "default": "…", "plex-imdb": "…", "jellyfin-tmdb": "…" },
  "file":   { "standard": "…", "default": "…", "original": "…", "plex-imdb": "…" } }
// sonarr
{ "season": {"default": "…"}, "series": {"default": "…", "plex-tvdb": "…"},
  "episodes": { "standard": {"default:4": "…", "original": "…"}, "daily": {...}, "anime": {...} } }
```
Keys are lower-cased. Sonarr episode lookups try `<key>:4` first, then `<key>` (`MediaNaming/NamingFormatLookup.cs`, `Plan/Components/SonarrMediaNamingPlanComponent.cs:44`). The `:3` variants are Sonarr-v3 leftovers.

**CF categories**: parsed from the guide markdown `docs/{Radarr/Radarr,Sonarr/sonarr}-collection-of-custom-formats.md` (`TrashGuide/CustomFormat/CustomFormatCategoryParser.cs`). Used only for `list custom-formats` grouping.

**Config-templates repo** (`recyclarr/config-templates`; `ConfigTemplates/*`, `ResourceProviders/Infrastructure/ConfigTemplatesStrategy.cs`): root `templates.json` and `includes.json`, each `{ "radarr": [{ "id", "template": "<rel path>.yml", "hidden" }], "sonarr": [...] }`. Ids must match `^[\w-]+$`, and paths are checked for traversal (GHSA-9wh7-8wrf-9x8r, v8.7.0).

### 2.4 State / cache files (`src/Recyclarr.Core/SyncState/*`)

- Path: `<CONFIG_DIR>/state/<sonarr|radarr>/<FNV-1a-64(base_url) hex>/<state-name>.json` (`SyncStateStoragePath.cs`). The key is the **URL**, so changing `base_url` orphans the state. Unreadable JSON is treated as empty.
- Content: `{ "mappings": [ { "trash_id": "…", "name": "…", "service_id": 42 } ] }` (`TrashIdMapping.cs`, still reads legacy `custom_format_name`/`custom_format_id` from v7.5).
- Two state files: **custom formats** (key `trash_id`, 1:1) and **quality profiles** (composite key `(trash_id, name)`, which supports several QPs from one guide QP).
- Scope invariant: only resources in the user's *effective* config (explicit, or derived through QP `formatItems`/CF groups) are recorded. Deleting old CFs relies on this.
- Update (`TrashIdMappingStore.Update`): drop entries whose service id no longer exists or was deleted, full-outer-join with synced mappings on service id, keep entries not in config (so they can be deleted later), and dedupe service ids.
- Other on-disk data: `<DATA_DIR>/resources/<type>/git/<name>/` (shallow clones), `<DATA_DIR>/logs/cli/`, `<CONFIG_DIR>/{configs,includes}/`, `recyclarr.yml`, `settings.yml`, `secrets.yml` (`Platform/AppPaths.cs`).

---

## 3. Feature areas

### 3.1 Sync orchestration (`src/Recyclarr.Cli/Processors/Sync/*`, `Pipelines/Plan/*`)

- Instances are processed **sequentially** (`SyncProcessor.cs:67`), each in its own Autofac scope with base URL and API key. Filters: `sync [radarr|sonarr] -i <instance>...`.
- Per instance:
  1. **Capability check** (`Compatibility/ServiceAgnosticCapabilityEnforcer.cs`): `GET /system/status` → appName must match the config section, and Sonarr ≥ 4.0.0.0.
  2. **Plan**: CF → QP → QS → MN → MM plan components. They collect errors and warnings instead of failing fast.
  3. **Pipelines** in dependency order.
  4. **Diagnostics panel**.
  5. **Apprise notification** (once per run).
- `--preview` stops each pipeline after its Preview phase and makes no writes.
- Live progress table: rows are instances, columns are pipelines (`Processors/Sync/Progress`).

### 3.2 Custom formats (`Pipelines/CustomFormat/*`, `docs/architecture/custom-format-pipeline.md`)

- **Plan** (`CustomFormat/ConfiguredCustomFormatProvider.cs`, `Plan/Components/CustomFormatPlanComponent.cs`). Sources of planned CFs, merged by trash_id with their score assignments combined:
  - flat `custom_formats`;
  - guide-backed QP `formatItems` (implicitly assigned to that QP);
  - **default CF groups**: group `default:"true"`, at least one of the user's guide-backed QPs in `quality_profiles.include`, not in `skip`. Adds `required` and `default` CFs only;
  - **explicit CF groups** (`add`): required + (default − exclude) + (select | select_all). Validated by `Plan/Components/ExplicitCfGroupValidator.cs`.
- **ApiFetch**: `GET /customformat`, and load state.
- **Transaction**, ID-first matching (`CustomFormatTransactionPhase.cs`):
  1. state id exists in the service → compare; **update** if not equivalent (name, `includeCustomFormatWhenRenaming`, specs matched by name + implementation/negate/required/fields), else **unchanged**;
  2. state id stale → fall through to name matching;
  3. no state → case-insensitive name match: 0 → **create**; 1 → **adopt ("Replaced", warning) and update**; ≥2 → **ambiguous error**;
  4. **delete candidates** = in state, not in config, still in the service.
- **Persistence**: POST new (the returned id is written back into the shared planned object so QP can use it), PUT updated, DELETE candidates **only if `delete_old_custom_formats: true`**, then update and save state.
- Updates are full replacements of the CF (guide is the only source of truth for spec content). User edits to a managed CF's specs are overwritten on the next sync.

### 3.3 Quality profiles (`Pipelines/QualityProfile/*`, `docs/architecture/quality-profile-*.md`)

- **Guide-backed** (with `trash_id`) vs **user-defined** (name only, not tracked in state).
- Three-layer precedence per field: **service < guide < config** (`UpdatedQualityProfile.cs` `Effective*`).
- **Plan** (`Plan/Components/QualityProfilePlanComponent.cs`):
  - inherit `score_set` from `trash_score_set`;
  - convert guide `items` to config `qualities` when config has none;
  - build CF scores from the CF plan;
  - **implicit profiles**: a name in `assign_scores_to` that is not in `quality_profiles` is **update-only** (never created);
  - duplicate effective names are rejected.
- **Score resolution** per (CF, profile): explicit `score` (per-profile, then entry-level) → `trash_scores[score_set]` → `trash_scores.default` → not scored.
- **ApiFetch**: `GET /qualityprofile`, `GET /qualityprofile/schema` (template for new profiles), `GET /language`. Items are reversed on read and write (API is bottom-to-top, Recyclarr works top-to-bottom).
- **Matching** (`PipelinePhases/UpdatedProfileBuilder.cs`):
  - Pass 1: exact `(trash_id, name)` in state.
  - Pass 2: rename detection when exactly 1 unclaimed mapping and 1 unmatched profile share a trash_id.
  - Otherwise name collision: 0 → create (if allowed); 1 → adopt; ≥2 → ambiguous.
  - A cached profile whose new name is taken by another profile → "rename conflict".
- **Missing-quality fix-up**: appends schema qualities missing from an existing profile (Radarr#9738).
- **Qualities** (`QualityItemOrganizer.cs`): listed qualities and groups become the *wanted* list, in order. Unlisted qualities are kept as disallowed and placed `top` or `bottom`. New groups get the next free id. `cutoff` = `until_quality` / guide `cutoff` resolved by name, else the first allowed item. Cutoff is always set because the API validates it even when upgrades are off.
- **Scores** (`QualityProfileTransactionPhase.ProcessScoreUpdates`): full outer join of planned CF scores with the profile's existing `formatItems`:
  - planned → set score;
  - **existing but unplanned → left unchanged**, unless `reset_unmatched_scores.enabled`, in which case set to 0 except names in `except` / matches of `except_patterns`.
- **Language**: Radarr only. Guide `language` name → id from `/language`. Sonarr v4 QPs have no language.
- **Validation** (`UpdatedQualityProfileValidator.cs`): `qualities` required when creating, `until_quality` must exist, min score satisfiable.
- **Persistence**: POST (schema DTO + merged domain) or PUT onto the stashed DTO, only for changed profiles. **No QP deletion feature** (by design).

### 3.4 Quality definitions / sizes (`Pipelines/QualitySize/*`, `Plan/Components/QualitySizePlanComponent.cs`)

- Select the guide file by `type`, apply per-quality overrides (`unlimited` → null), validate min ≤ preferred ≤ max, clamp `preferred_ratio` to 0..1.
- **Limits by version** (`Limits/*`, `Compatibility/*Capabilities.cs`):
  - Radarr ≥ 5.9.0.9049: max 2000 / preferred 1999, else 400/399;
  - Sonarr ≥ 4.0.8.2158: 1000/995, else 400/395.
  - Values at the limit are sent as `null` (= unlimited).
- `preferred_ratio` recomputes preferred = `min + (min(max, prefLimit) − min) × ratio`.
- `GET /qualitydefinition` → match by quality **name** → `PUT /qualitydefinition/update` with only the changed items (merged onto the stashed DTOs). Qualities missing on the server trigger a warning.

### 3.5 Media naming (`Pipelines/MediaNaming/{Sonarr,Radarr}`, `Plan/Components/*MediaNamingPlanComponent.cs`)

- Config keys resolve to format strings from the guide naming JSON; an unknown key is an error.
- `GET /config/naming` → diff only managed fields (Radarr `renameMovies, movieFolderFormat, standardMovieFormat`; Sonarr `renameEpisodes, seriesFolderFormat, seasonFolderFormat, standard/daily/animeEpisodeFormat`) → `PUT /config/naming/{id}` of the full stashed object. Other fields (colon replacement, multi-episode style…) are untouched.

### 3.6 Media management (`Pipelines/MediaManagement/*`, v8.0)

- Only `downloadPropersAndRepacks` (`preferAndUpgrade|doNotUpgrade|doNotPrefer`). GET → merge → `PUT /config/mediamanagement/{id}`.

### 3.7 Delete old CFs / delete command

- `delete_old_custom_formats: true` removes CFs that are **in state and no longer in config**. It never touches CFs Recyclarr never managed.
- `recyclarr delete custom-formats <instance> [names…] | --all [--force] [--preview]` (`Processors/Delete/DeleteCustomFormatsProcessor.cs`): deletes by **name from the live service, ignoring state**, in parallel. `--all` wipes every CF, including hand-made ones.

### 3.8 Templates & includes

- `config create [-t template…] [-p path] [-f]`: copies a starter config (`src/Recyclarr.Cli/config-template.yml`) or a template from config-templates.
- `config list templates [-i]` / `config list local`.
- `include: [{template: id}]` pulls a YAML fragment from the config-templates repo. `include: [{config: path}]` uses local `includes/`. The config-templates repo is branch-versioned: default refs `v{major}` → `master` → `main` (`ConfigTemplatesStrategy.cs:15`).

### 3.9 Score sets

- A score set is a key in a CF's `trash_scores` map (e.g. `default`, `anime-sonarr`, `sqp-1-1080p`, `sqp-1-2160p`, `french-*`, `german-*`).
- Set per QP (`score_set`), or inherited from the guide QP's `trash_score_set`. It falls back to `default` per CF when the set lacks that CF.
- `list score-sets <service>` lists them.
- Score sets exist because one CF JSON serves many profile families. This is the TRaSH equivalent of Profilarr storing scores on the QP↔CF join row.

### 3.10 Git repo handling (`Repo/RepoUpdater.cs`, `VersionControl/GitRepository.cs`, `ResourceProviders/Storage/*`)

- Implicit official providers: `https://github.com/TRaSH-Guides/Guides.git` (refs `master`, `main`) and `https://github.com/recyclarr/config-templates.git`. Disabled per type with `replace_default: true`.
- Each run: `git init` (first time) → `git fetch --depth 1 <url> <ref…>` → `git reset --hard FETCH_HEAD`. If every ref fails to fetch, the run continues with the existing files (warning).
- `cache_limit` (default 100 MB): if `.git` grows past it, the clone is deleted and re-fetched (v8.7.1).
- Local providers: plain directories, no caching. `custom-formats` type is a flat directory of CF JSON and needs `service`.
- Precedence: official first, then YAML order. Dedupe by `trash_id` with **last wins**, so users can override single guide CFs or QPs.
- **No auth/PAT handling** (public URLs, or credentials embedded in the URL). **No push**: read-only.

### 3.11 CLI commands (`src/Recyclarr.Cli/Console/CliSetup.cs`, `Console/Commands/*`)

See §5.1.

### 3.12 Migration steps (`src/Recyclarr.Cli/Migration/Steps/*`, ADR-004)

They run automatically at startup, before any command. `migrate` is a deprecated no-op.

| Order | Step | Effect |
|---|---|---|
| 1 | `DeleteRepoDirMigrationStep` | Delete legacy `<config>/repo` (pre-resource-provider clone) |
| 2 | `MoveCacheToStateMigrationStep` | `cache/<service>/<hash>/` → `state/…`; `cache/resources` → `<data>/resources` (handles cross-filesystem moves) |
| 3 | `NormalizeStateMappingsPropertyStep` | Rename the top-level JSON property in state files to `mappings` |

Deprecated env var `RECYCLARR_APP_DATA` → hard error with instructions (`Platform/DefaultAppDataSetup.cs`).

### 3.13 Notifications (`src/Recyclarr.Core/Notifications/*`)

- Apprise API only: `POST {base_url}/notify/{key}` (stateful) or `POST /notify` with urls (stateless).
- Verbosity: `minimal` (errors only), `normal`, `detailed`, `verbose` (per-CF names). Empty results are suppressed.

---

## 4. Integrations (full lists)

| Integration | Details | Status |
|---|---|---|
| **Sonarr v4** | API `/api/v3`. **Minimum 4.0.0.0** enforced (`Compatibility/Sonarr/SonarrCapabilities.cs:5`); quality-size limit bump at 4.0.8.2158 | Active |
| **Sonarr v3** | Supported until v6.x (release profiles, `:3` naming, `hybrid` quality definition). **Dropped in v7.0.0 (2024-06-27)** | **Obsolete** |
| **Sonarr release profiles** (TRaSH "[Trash]" RP JSON, `term` / `preferred` / `ignored`) | Synced by trash_id in v2–v6; deleted `[Trash]` RPs not in config | **Obsolete** (removed v7; TRaSH replaced them with CFs) |
| **Radarr** (v4/v5+) | API `/api/v3`. No minimum enforced; quality-size limit bump at 5.9.0.9049 | Active |
| **Sonarr v5 / Radarr v6** | Nothing version-specific; vendored OpenAPI specs are `3.0.0` (`src/Recyclarr.Api.*/openapi.json`) | Unknown; a breaking `/api/v4` would need new specs |
| **TRaSH Guides** | `github.com/TRaSH-Guides/Guides` (branch `master`), `metadata.json`-driven | Active (primary data) |
| **recyclarr/config-templates** | `templates.json` / `includes.json`, branch `v{major}` | Active |
| **Custom CF dirs / guide forks** | `resource_providers` (git or local) | Active |
| **Apprise API** | Notifications | Active |
| **git CLI** | Required on host (bundled in Docker) | Active |
| **Lidarr / Readarr / Whisparr / Prowlarr** | Not supported (`TrashGuide/SupportedServices.cs` = Radarr, Sonarr) | n/a |
| **Profilarr / Dictionarry PCDs** | No interop | n/a |

---

## 5. External APIs / interfaces

### 5.1 CLI surface (`recyclarr …`)

| Command | Args / options |
|---|---|
| `sync [radarr|sonarr]` | `-c|--config <file>…`, `-p|--preview`, `-i|--instance <name>…`, `--log [debug|info|warn]` |
| `list custom-formats <svc>` | `--raw` (TSV) |
| `list custom-format-groups <svc>` | `--filter`, `--details`, `--raw` |
| `list quality-profiles <svc>` | `--filter`, `--details`, `--raw` |
| `list score-sets <svc>` / `list qualities <svc>` / `list naming <svc>` | `--raw` |
| `config create` | `-p|--path`, `-t|--template <id>…`, `-f|--force` |
| `config list local` / `config list templates` | `-i|--includes` |
| `delete custom-formats <instance> [cf_names…]` | `-a|--all`, `-f|--force`, `-p|--preview` |
| `state repair [custom-formats|quality-profiles]` | `-i`, `-p`, `-v`, `--adopt` (**deprecated no-op** since 8.4) |
| `migrate` | deprecated no-op |

- Exit codes: 0 on success, non-zero on failure (`ExitStatus`).
- Env vars:
  - `RECYCLARR_CONFIG_DIR`, `RECYCLARR_DATA_DIR`, `NO_COLOR`;
  - Docker only: `CRON_SCHEDULE`, `RECYCLARR_CREATE_CONFIG`, `RECYCLARR_LOG_LEVEL`.
- There is no HTTP API of its own.

### 5.2 Sonarr/Radarr endpoints called (all `/api/v3`, header `X-Api-Key`; `src/Recyclarr.Core/Http/RefitClientExtensions.cs`, gateways in `src/Recyclarr.Core/ServarrApi/`)

The refitter `includeTags` are CustomFormat, QualityProfile, QualityProfileSchema, QualityDefinition, MediaManagementConfig, NamingConfig, System, Language. The same endpoint set is used for both apps.

| Endpoint | Use / payload |
|---|---|
| `GET /system/status` | `appName` (service-type check), `version` (capabilities) |
| `GET /customformat` | Full list (for id/name matching and equivalence) |
| `POST /customformat` | `{name, includeCustomFormatWhenRenaming, specifications:[{name, implementation, negate, required, fields:[{name, value}]}]}`; returns `id` |
| `PUT /customformat/{id}` | Same body with `id`; full replace of specs |
| `DELETE /customformat/{id}` | `delete_old_custom_formats` and `delete custom-formats` |
| `GET /qualityprofile` | All profiles (DTOs stashed for round-trip) |
| `GET /qualityprofile/schema` | Base DTO for **new** profiles (full quality tree, all `formatItems`) |
| `GET /language` | Map guide language name → id (Radarr) |
| `POST /qualityprofile` | Schema DTO + merged fields |
| `PUT /qualityprofile/{id}` | Stashed DTO with overwritten `name, upgradeAllowed, cutoff (id), minFormatScore, cutoffFormatScore, minUpgradeFormatScore, items[] (reversed back to API order; {id?, name?, allowed, quality{id,name}, items[]}), formatItems[{format, name, score}] (every existing item kept; only managed scores changed, or zeroed by reset), language{id,name} (Radarr)` |
| `GET /qualitydefinition` | Match by `quality.name` |
| `PUT /qualitydefinition/update` | Array of stashed DTOs with `minSize, maxSize|null, preferredSize|null`, changed items only |
| `GET /config/naming`; `PUT /config/naming/{id}` | Full stashed object with managed fields replaced (§3.5) |
| `GET /config/mediamanagement`; `PUT /config/mediamanagement/{id}` | Full stashed object with `downloadPropersAndRepacks` replaced |

- Not called: `customformat/bulk`, `customformat/schema`, `qualitydefinition/limits` (hard-coded by version instead), `delayprofile`, `releaseprofile` (since v7), `tag`, commands.
- Apprise: `POST /notify[/{key}]`.
- Git: `git init|fetch --depth 1|reset --hard|status` against GitHub.

---

## 6. Background jobs

- **None in-process.** Recyclarr is a run-to-completion CLI. Scheduling is external:
  - **Docker**: `entrypoint.sh` writes `"$CRON_SCHEDULE /cron.sh"` (default `@daily`) and runs **supercronic**. `cron.sh` runs `recyclarr sync --log $RECYCLARR_LOG_LEVEL`. With arguments, the container runs a one-shot CLI command instead (`docker/scripts/*.sh`, `Dockerfile`).
  - Bare metal: user cron, systemd timer, or k8s CronJob.
- Per run:
  - migrations;
  - log janitor (keeps the newest `max_files` logs);
  - git fetch of every provider (every run, shallow);
  - sync;
  - notification.
- Planned (ADR-008): a long-lived server host with an internal scheduler and API. Not built.

---

## 7. Configuration surface

- **Files**:
  - `<CONFIG_DIR>/recyclarr.yml` and `<CONFIG_DIR>/configs/*.yml|*.yaml` (all merged; instance names must be unique);
  - `includes/`;
  - `settings.yml`;
  - `secrets.yml` (for `!secret`).
  - `CONFIG_DIR` defaults: Linux `~/.config/recyclarr`, macOS `~/Library/Application Support/recyclarr`, Docker `/config`.
- **Per instance**: see §2.1. There are no toggles for which pipelines to run: a section present means "sync it".
- **Global** (`settings.yml`): see §2.2.
- **Validation**: FluentValidation rules (`Config/Parsing/ConfigYamlDataObjectsValidation.cs`, `YamlValidatorRuleSets.cs`), JSON Schemas for editor autocompletion (`schemas/`), and YAML-centric error messages.
- **Secrets**: plaintext in YAML, `secrets.yml` or env vars. Implicit `<instance>_base_url` / `<instance>_api_key` secrets (`ImplicitUrlAndKeyPostProcessor.cs`).

---

## 8. Platform/infrastructure concerns

- **Distribution**: self-contained single-file binaries (linux/win/osx, x64/arm64), Docker (GHCR/Docker Hub, major-version tags only since 8.4, UID 1000, read-only-rootfs capable), plus community packages. MIT licence.
- **External runtime deps**: the `git` binary and outbound HTTPS to github.com on every run. Air-gapped use needs local providers.
- **Filesystem**: shallow clones, capped at `cache_limit` and rebuilt when exceeded. State is keyed by a hash of `base_url` (URL change = state loss, then auto-adopt by name).
- **Concurrency**: instances are processed sequentially, pipelines sequentially within an instance. There is no locking against concurrent runs (two crons can race on state files).
- **Security**:
  - template-manifest path traversal was fixed in v8.7.0;
  - `enable_ssl_certificate_validation: false` escape hatch;
  - API keys in plaintext.
- **Code generation at build**: Refitter from the vendored OpenAPI specs. Upstream API changes surface as compile errors when the specs are refreshed.
- **Heavy E2E**: Testcontainers spin up real Sonarr/Radarr (`tests/Recyclarr.EndToEndTests/`).

---

## 9. Interactions with other *arr apps (precise sync semantics)

**Identity model: `trash_id` → service id (persisted).** This is the core difference from Profilarr, which matches by name only.

| Resource | Match key | Rename in source | Create | Update | Delete |
|---|---|---|---|---|---|
| CF (guide) | state `trash_id`→id, then case-insensitive name | Handled: the id is kept, the Arr CF is renamed on PUT | Name not in Arr | Only if not equivalent (PUT full CF) | Only `delete_old_custom_formats` and only state-owned CFs |
| CF (hand-made in Arr, not in config) | – | – | – | **Never touched** | Only by `delete custom-formats --all/<name>` |
| QP (guide-backed) | state `(trash_id,name)`→id, rename pass, then name | 1 rename per trash_id per sync | Yes (from `/qualityprofile/schema`) | Only if changed; merges onto the live DTO | **Never** |
| QP (user-defined) | Name (case-insensitive) | Rename = new profile | Yes (needs `qualities`) | Yes | Never |
| QP (implicit, from `assign_scores_to` only) | Name | – | **No** (update-only) | Scores only | Never |
| Quality definitions | Quality name | – | No | Changed items only | No |
| Naming / media mgmt | Singleton | – | No | Managed fields only (GET-merge-PUT) | No |

**Overwrite semantics in detail:**

- **CF specs**: fully replaced with guide content. Manual Arr edits to a managed CF are reverted on the next sync. A CF equivalence check avoids no-op PUTs.
- **QP scores**: managed CF scores are overwritten. **Unmanaged CF scores are preserved** unless `reset_unmatched_scores.enabled` (default **false**), which zeroes them except `except` / `except_patterns`. Profilarr instead always sends every CF with unmanaged ones at 0.
- **QP qualities**: if the config or guide lists qualities, the listed ones are ordered and enabled as given and the rest become disallowed, placed at `quality_sort`. If no qualities are listed (implicit/user-defined without `qualities`), the existing Arr order is kept.
- **QP scalar fields**: config > guide > existing Arr value. Fields absent from both keep the Arr value.
- **Unknown/extra Arr fields**: preserved (stashed DTO round-trip, ADR-005/006).

**"Replace existing CFs" history** (CHANGELOG):
- v4.4: added `replace_existing_custom_formats` (adopt same-name CFs not in cache);
- v5.0: defaulted to `false`;
- v8.0: removed in favour of `state repair --adopt`;
- **v8.4: adoption is automatic** ("config is authoritative"): a single same-name CF/QP is adopted with a warning; two or more same-name matches is an error; `state repair` deprecated.

**Conflicts with Profilarr or manual edits on the same instance:**

- **Recyclarr + Profilarr managing the same CF name** (e.g. both define "x265 (HD)"):
  - Recyclarr adopts it into its state and PUTs guide specs.
  - Profilarr matches by name and PUTs PCD specs.
  - Last writer wins on each run (flip-flop).
  - A rename in either tool creates a duplicate. A later Recyclarr sync then hits "ambiguous" (two name matches) only if the names collide case-insensitively.
- **Same QP name**:
  - Profilarr PUTs `formatItems` with every CF, unmanaged ones at 0, which wipes Recyclarr's scores.
  - Recyclarr then restores only its own scores and leaves Profilarr's.
  - Neither side converges.
- **Profilarr cleanup job** deletes every CF/QP it doesn't manage, including Recyclarr's. Recyclarr's next run sees stale state ids, falls through to name matching (0 matches) and **recreates** them with new ids.
- **Recyclarr `delete_old_custom_formats`** only deletes CFs in its own state, so it is safe against Profilarr or hand-made CFs, unless it previously adopted a Profilarr CF by name.
- **Manual edits in the Arr UI**:
  - CF spec edits are reverted;
  - renaming a managed CF in the Arr is reverted to the guide name (the id is kept);
  - deleting a managed CF in the Arr → recreated;
  - QP score edits on managed CFs are reverted; on unmanaged CFs they are kept unless reset is on;
  - QP deletion in the Arr → recreated on the next sync (guide-backed or explicitly listed).
- **Multiple Recyclarr configs pointing at one URL**: rejected ("split instances").

**Sonarr vs Radarr differences handled:**
- naming schema (separate pipelines);
- QP `language` (Radarr only);
- quality-size limits per version;
- separate guide trees (`radarr/*` vs `sonarr/*`) with **app-specific CF spec enum ids baked into the guide JSON**, so Recyclarr needs **no enum mapping tables**, unlike Profilarr's `sync/mappings.ts`.

---

## 10. Outdated / legacy candidates

| Item | Evidence |
|---|---|
| **Sonarr v3 support, release profiles** | Removed v7.0.0 (CHANGELOG L482). TRaSH replaced RPs with CFs. Sonarr `:3` naming keys still exist in guide data and the `:4`-first lookup remains (`SonarrMediaNamingPlanComponent.cs:44`) |
| `state repair` command, `--adopt` | Deprecated 8.4.0 (`Console/Commands/StateRepairCommand.cs`, `docs/architecture/trash-id-state-system.md`) |
| `migrate` command | Deprecated no-op (ADR-004) |
| `replace_existing_custom_formats`, `custom_formats[].quality_profiles`, `repositories` setting, `--app-data`, `RECYCLARR_APP_DATA`, include files under `configs/` | Removed in v8, deprecation messages remain (`Config/Parsing/PostProcessing/Deprecations`, `Settings/Deprecations`, `Platform/DefaultAppDataSetup.cs`) |
| Legacy state field names `custom_format_name/_id` | "Remove in v9" (`SyncState/TrashIdMapping.cs`) |
| Migration steps 1–3 | Only relevant to users upgrading from < v8 |
| ADR-007 CF gateway passthrough | Deprecated after the Refit migration |
| CF group `exclude` semantics (upstream) | Replaced by `include` (PDR-001). The "Base Profile" workaround (PDR-007) is upstream data debt |
| `CustomFormatCategoryParser` (markdown scraping) | Fragile; used only for `list` output |
| Hard-coded quality-size limits by version | Could use `GET /qualitydefinition/limits` |
| Old `reset_unmatched_scores` boolean syntax | Removed in v6 |
| `trash_description`, `trash_regex`, `trash_url` | Ignored by Recyclarr; only useful for a UI |

---

## 11. Rewrite notes (Go monolith)

### 11.1 What disappears in a monolith

Everything that exists only because the source of truth lives outside the Arr:
- `SyncState/` (trash_id→service-id mapping, two-pass rename resolution, adopt/ambiguous/rename-conflict logic, split-instance filter, URL-hash state dirs);
- `ServarrApi/` gateways and stash/merge round-trip;
- Refitter clients and vendored OpenAPI specs;
- capability/version checks and quality-size limit tables;
- `ReverseItems` ordering hacks;
- schema-based new-profile creation;
- the missing-quality fix-up (Radarr#9738);
- the CF equivalence checker;
- `delete_old_custom_formats`;
- `delete custom-formats`;
- the Preview/ApiPersistence phases (become a normal "diff before apply" in the monolith's own store);
- migrations;
- external cron.

That is roughly 70% of `Recyclarr.Cli/Pipelines` and `Recyclarr.Core/{ServarrApi,Servarr,SyncState,Compatibility}`.

### 11.2 What to keep: a native TRaSH importer

The monolith should import TRaSH Guides (and optionally config-templates) into its own profile store. Needed pieces:

1. **Fetcher**: git shallow fetch (or GitHub tarball/HTTP) of `TRaSH-Guides/Guides`, read `metadata.json` `json_paths`, glob JSON. Keep Recyclarr's **multi-provider precedence** (official → user forks/local dirs, last wins by `trash_id`), which is simple and useful.
2. **Schema mapping to a unified profile model** (using the Profilarr-style schema recommended in comparison.md §4.4):

| TRaSH field | Unified model | Notes |
|---|---|---|
| CF `trash_id` | `custom_format.external_ref = ("trash", id)` | Stable upstream key. Keep it alongside the monolith's own id/name for re-import and upgrades |
| CF `name` | `custom_format.name` | Profilarr uses the name as its key. The TRaSH id survives renames, the name does not |
| CF `specifications[].implementation` | condition type enum | Same 11 implementation names as Profilarr |
| CF `fields.value` (regex) | inline pattern or shared regex entity | .NET regex → needs `regexp2` (comparison.md §5.1) |
| CF `fields.value` (int enum) | canonical enum (source/resolution/flag/language/modifier/release type) | **Needs per-app reverse maps** (Radarr vs Sonarr ids) because TRaSH stores raw Arr ids. Radarr and Sonarr CF files must be imported separately, then deduped by name/trash_id where equivalent |
| CF `trash_scores{set: n}` | `cf_score_set(cf, set_name, score)` | Unified model needs **named score sets** (Profilarr stores scores only on the QP↔CF row) |
| CF `includeCustomFormatWhenRenaming` | bool | |
| QP `trash_id`, `name`, `group`, `trash_url`, `trash_description` | profile metadata + external_ref | |
| QP `items` (names, groups, allowed) | ordered profile qualities/groups | Map Arr quality names to the canonical quality set (Radarr `Remux-1080p` ≡ Sonarr `Bluray-1080p Remux`) |
| QP `cutoff` (name) | `upgrade_until` item | |
| QP `upgradeAllowed`, `minFormatScore`, `cutoffFormatScore`, `minUpgradeFormatScore` | same | |
| QP `language` (Radarr) | profile language | |
| QP `formatItems{name: trash_id}` + `trash_score_set` | `profile_cf(profile, cf, score)` materialized from the score set, **or** a live reference "use set X" | Decide whether scores are snapshotted at import or resolved at runtime |
| CF group (`default`, `custom_formats[required, default]`, `quality_profiles.include`) | "CF bundle" / optional add-on with per-member required/default flags | No Profilarr equivalent. Useful as UI toggles ("add Audio Formats") |
| QP group | UI catalogue grouping | |
| quality-size `type` + `qualities[{quality, min, preferred, max}]` | named quality-size preset (MB/min) | Monolith owns limits, so no 400/1000/2000 caps |
| naming `folder/file/series/season/episodes.*` | naming presets keyed by (media kind, slot, preset name) | Monolith has one naming engine; drop the `:3` variants |
| Recyclarr YAML overrides (`score`, `score_set`, `reset_unmatched_scores`, `qualities`, `quality_sort`, `select/exclude`) | user override layer on imported entities | Recyclarr's 3-layer precedence (service < guide < config) becomes (default < upstream < local override) |

3. **Update semantics**: re-importing TRaSH must update entities matched by `trash_id` (not name), preserve local overrides, and flag upstream deletions. This is the problem Profilarr's op/override/conflict engine solves. Recyclarr avoids it by keeping overrides in YAML and never storing guide data.
4. **Optional compatibility**: a `recyclarr.yml` importer (one-shot migration of users' configs into monolith profiles) is cheap given the small schema (§2.1).

### 11.3 Recyclarr vs Profilarr

| Dimension | Recyclarr | Profilarr |
|---|---|---|
| Upstream data | TRaSH Guides JSON (app-specific trees, Arr-shaped payloads plus `trash_*` keys) | PCD git repos: SQL op files over the `Dictionarry-Hub/schema` relational schema. Dictionarry DB; TRaSH available through the `trash-pcd` port |
| Identity | `trash_id` (32-hex, immutable) | Entity **name** (unique, case-insensitive; cascades on rename) |
| Arr id mapping | Persisted state file per instance, `trash_id`→id (QP: (trash_id, name)→id). Handles renames | None; name match every run. Rename = new object plus orphan |
| App divergence | Separate Radarr and Sonarr files; enum ids baked in; no mapping tables | One canonical entity; `arr_type` tags; per-app enum and quality mapping tables at sync |
| Scores | CF-side `trash_scores` sets; QP picks a set; YAML overrides | QP↔CF rows with per-arr-type score |
| Optional bundles | CF groups (`default`/`required`/`select`) | None (QP includes CFs explicitly) |
| User customisation | YAML overrides on referenced objects; custom CF JSON via local providers | Full editor; local ops layered on base with override/align/ask conflict handling |
| Authoring and publishing | None (read-only consumer) | UI authoring, publishing ops to git |
| Testing | None | CF/regex tests, QP simulation with a C# parser |
| Unmanaged CF scores in a managed QP | **Preserved** (optional reset) | **Zeroed** (all CFs sent) |
| Deletes | Opt-in; only state-owned CFs; never QPs | Cleanup job deletes **everything** unmanaged (CFs and QPs) |
| Adoption of pre-existing objects | Automatic (single name match), warning; error on ambiguity | Implicit (name match = PUT) |
| No-op detection | Yes (equivalence diff; only changed items PUT) | Always PUT (drift job diffs separately) |
| Delay profiles, tags | No | Default delay profile only |
| Media management | Naming (preset keys), propers/repacks | Naming (full formats), media settings, quality definitions |
| Quality sizes | TRaSH presets + overrides + `preferred_ratio` | PCD quality-definition entities |
| Extra automation | None | Upgrades/search, rename, cleanup, drift, library viewer |
| Runtime | CLI + cron, no DB, ~20k LOC | Web app + SQLite + job queue + parser sidecar, ~131k LOC in `src` |
| Round-trip safety | Stashed DTO merge (unknown fields kept) | GET-merge-PUT for settings; full-object PUT for CF/QP |

### 11.4 Recommendation

- **Profile system model**: use **Profilarr's relational model** (comparison.md §4.4). It is app-neutral (canonical qualities, typed conditions, per-media-kind scores) and fits a monolith that owns the quality and language enums. **Add from TRaSH/Recyclarr**:
  1. an **`external_ref` (source, id)** on every CF/QP/group so upstream re-imports match by stable id instead of name (fixes Profilarr's rename-orphan weakness);
  2. **named score sets** on CFs, because TRaSH's multi-family scoring (anime, SQP, French/German) does not fit a single score per QP↔CF without duplication;
  3. **CF groups** as optional bundles with required/default members;
  4. **quality-size and naming presets** as first-class importable entities;
  5. Recyclarr's non-destructive defaults: never zero unmanaged scores, never delete what you didn't create, adopt only unambiguous matches.
- **Community data import**: support **both**, with **TRaSH as the primary source**. TRaSH is the de-facto upstream: Recyclarr and other third-party sync tools consume it (PDR-007 mentions a Clonarr-built authoring tool), and Dictionarry publishes a TRaSH port as `trash-pcd`. Its format is plain JSON with stable ids, cheap to parse, and needs no SQL engine. PCD import (SQL ops over a SQLite schema) is valuable for Dictionarry users but couples the importer to Profilarr's schema versioning and SQLite semantics. Import PCDs via their portable JSON forms (`src/lib/shared/pcd/portable.ts` in Profilarr) or through a replay-into-temp-SQLite adapter, not as the native store.
- **Hard constraint shared by both**: CF regexes are .NET syntax. TRaSH enum values are raw per-app Arr ids and must be reverse-mapped to the monolith's canonical enums at import. This is the only place where Recyclarr-style data still needs the mapping tables that Profilarr keeps at sync time.
