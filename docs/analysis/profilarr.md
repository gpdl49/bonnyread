# Profilarr — Analysis Report

Source analysed: `submodules/profilarr` @ `7fec444c` (2026-09-26), release line v2.x (latest tag `v2.2.0`).
All paths below are relative to `submodules/profilarr/` unless stated.

---

## 1. Overview

**Purpose.** Profilarr is a configuration manager for Radarr and Sonarr. Its README calls it a tool to "build, test, and deploy configurations across your media stack". It has three jobs:

1. **Build.** Author and curate quality profiles (QPs), custom formats (CFs), regular expressions, delay profiles and media-management settings (naming, media settings, quality definitions). These live in git-hosted datasets called **PCDs** (Profilarr Compliant Databases), and users can layer their own local changes on top.
2. **Test.** Test CFs against release titles using a C# parser that copies Arr parsing. Simulate how a QP scores and ranks releases for a real TMDB movie or series ("entity testing"). Validate regexes against the .NET engine and regex101 test cases.
3. **Deploy.** Push the compiled config into any number of Radarr/Sonarr instances. It also has extra Arr automation that is not about config: drift detection, stale-config cleanup, removed-media cleanup, scheduled "upgrade" searches, bulk rename, and Arr log/library viewers.

Sources: `README.md`, `docs/ARCHITECTURE.md`.

**Tech stack / versions** (`deno.jsonc`, `Dockerfile`, `.tool-versions`, `docs/ARCHITECTURE.md`):

| Layer | Choice |
|---|---|
| Language | TypeScript (server and client); C# for the parser |
| Runtime | Deno 2.9.7, shipped as a `deno compile` single binary (`dist/build/profilarr`) |
| Web | SvelteKit 2 (`@sveltejs/kit ^2.70`), Svelte 5, Vite 8, Tailwind 4, custom adapter (`src/adapter/`) |
| App DB | SQLite (WAL) via `@db/sqlite` FFI with raw SQL (`src/lib/server/db/`) |
| PCD cache | In-memory SQLite per linked database, queried through Kysely 0.27.6 |
| Scheduling | `croner` (cron) plus a custom SQLite job queue |
| Auth | bcrypt (`@felix/bcrypt`), `jose` (OIDC/JWT) |
| Other libs | `marked` (markdown), `@std/yaml`, `simple-icons`, `@lucide/svelte` |
| Parser | .NET 8 minimal API (`src/services/parser/Parser.csproj`), optional sidecar |
| External tools at runtime | `git` CLI (PCD clone/pull/push) and `tar` (backups), both installed in the image (`Dockerfile`) |

**Processes / services:**

- `profilarr` container: a single Deno binary that serves the SvelteKit UI, the API, the job dispatcher and the PCD compiler. The data dir is `/config`, holding `data/profilarr.db`, `data/databases/<uuid>` git clones, `logs/`, and `backups/` (`src/lib/server/utils/config/config.ts`).
- `profilarr-parser` container: stateless .NET 8 HTTP service on port 5000 (`Dockerfile.parser`). It is optional; only testing features need it. Outside Docker, `src/lib/server/utils/parser/spawn.ts` auto-spawns a parser binary that sits next to the main executable.
- Outbound dependencies: GitHub (git plus `api.github.com`), `raw.githubusercontent.com/Dictionarry-Hub/bulletin` (announcements and version manifest), TMDB API v3, regex101 API, and the Radarr/Sonarr v3 APIs.

**Rough size** (`find … | wc -l`):

| Area | LOC |
|---|---|
| `src/lib/server/pcd` (PCD engine: ops, compiler, conflicts, entities) | ~19.6k |
| `src/lib/server/db` (migrations, queries, schema snapshot) | ~10.9k |
| `src/lib/server/utils` (arr clients, parser client, git, auth, http, backup, logger) | ~7.8k |
| `src/lib/server/sync` | ~4.9k |
| `src/lib/server/drift` | ~3.0k |
| `src/lib/server/jobs` | ~2.6k |
| `src/lib/server/notifications` | ~2.4k |
| `src/lib/server/upgrades` / `rename` / `announcements` | ~2.3k / ~0.9k / ~1.7k |
| `src/lib/shared` (types, naming tokens, upgrade filters) | ~5.5k |
| `src/lib/client` (UI kit, stores, onboarding "cutscene") | ~14.3k |
| `src/routes` (pages, form actions, endpoints; ~38k of it Svelte) | ~51.2k |
| `src/services/parser` (C#) | ~2.7k |
| `tests` (unit ~10k, integration ~32k, e2e, scans) | ~43.7k |
| Totals in `src`: TS 81k, Svelte 47.7k, C# 2.7k | |

---

## 2. Domain model

There are **two separate data stores**:

- **(A)** The **app DB** (`profilarr.db`, migrations in `src/lib/server/db/migrations/001…072`, reference snapshot `src/lib/server/db/schema.sql`). It holds instances, sync selections, jobs, auth, caches, and **the ops log for PCDs**.
- **(B)** The **PCD schema**: configuration content. It is defined in a separate repo, `Dictionarry-Hub/schema`, and a reference copy lives at `docs/backend/0.schema.sql`. Generated TS types are in `src/lib/shared/pcd/types.ts`. It is materialised only as an **in-memory SQLite cache** per linked database, rebuilt by replaying ops.

### 2.1 PCD (configuration) schema — `docs/backend/0.schema.sql`

Key design choice: **all cross-entity foreign keys use the entity *name*** (a stable key), not the autoincrement id, with `ON DELETE CASCADE ON UPDATE CASCADE`. Names are unique case-insensitively (`docs/backend/pcd-entities.md`). Because of this, ops written by different people and replayed in any environment resolve to the same rows.

**Core**
- `tags(name UNIQUE)`
- `languages(name)`: seeded by schema ops
- `qualities(name)`: canonical Profilarr quality names, seeded
- `quality_api_mappings(quality_name, arr_type ∈ radarr|sonarr, api_name)`: maps canonical quality names to each Arr's API name. A missing row means that quality doesn't exist in that Arr.

**Regular expressions**
- `regular_expressions(name, pattern, regex101_id, description)` with `regular_expression_tags`. These are first-class shared patterns referenced by CF conditions.

**Custom formats**
- `custom_formats(name, description, include_in_rename)` with `custom_format_tags`
- `custom_format_conditions(custom_format_name, name, type, arr_type ∈ all|radarr|sonarr, negate, required)`. Condition names are unique per CF.
- Polymorphic condition data: exactly one type table per condition, keyed `(custom_format_name, condition_name)`:
  - `condition_patterns → regular_expression_name` (types `release_title`, `release_group`, `edition`)
  - `condition_languages(language_name, except_language)`
  - `condition_sources(source)`, `condition_resolutions(resolution)`, `condition_quality_modifiers(quality_modifier)`, `condition_release_types(release_type)`, `condition_indexer_flags(flag)`
  - `condition_sizes(min_bytes, max_bytes)`, `condition_years(min_year, max_year)`
- Allowed values per type, with per-arr applicability, are in `src/lib/shared/pcd/conditions.ts`:
  - `quality_modifier` and `edition` are Radarr-only.
  - `release_type` is Sonarr-only.
  - Sources `television_raw`/`bluray_raw` are Sonarr-only.
  - Indexer flags `ptp_golden`/`ptp_approved` are Radarr-only.
- `custom_format_tests(custom_format_name, title, type ∈ movie|series, should_match, description)`: embedded unit tests.

**Quality profiles**
- `quality_profiles(name, description, upgrades_allowed, minimum_custom_format_score, upgrade_until_score, upgrade_score_increment>0)` with `quality_profile_tags`
- `quality_profile_languages(quality_profile_name, language_name, type ∈ must|only|not|simple)`. Writers only ever write `simple` and sync reads only the name (see §10).
- `quality_groups(quality_profile_name, name)`: groups belong to one profile and are not shared.
- `quality_group_members(profile, group, quality_name, position)`. `position` was added by schema op 3 (`ALTER TABLE … DEFAULT 0`).
- `quality_profile_qualities(quality_profile_name, quality_name XOR quality_group_name, position, enabled, upgrade_until)`. A partial unique index `idx_one_upgrade_until_per_profile` allows only one cutoff item.
- `quality_profile_custom_formats(quality_profile_name, custom_format_name, arr_type ∈ all|radarr|sonarr, score)`. **Scores are per arr type.** An `all` row gets split into per-type rows when the user edits one type (`docs/backend/pcd-entities.md#scoring`).

**Delay profiles**
- `delay_profiles(name, preferred_protocol ∈ prefer_usenet|prefer_torrent|only_usenet|only_torrent, usenet_delay, torrent_delay, bypass_if_highest_quality, bypass_if_above_custom_format_score, minimum_custom_format_score)`. CHECK constraints tie the delays and bypass score to the protocol. There are no tags; the schema comment says the Arr only lets Profilarr overwrite the default profile (id=1).

**Media management** (separate Radarr and Sonarr tables, each row a *named config*)
- `radarr_naming(name, rename, movie_format, movie_folder_format, replace_illegal_characters, colon_replacement_format ∈ delete|dash|spaceDash|spaceDashSpace|smart)`
- `sonarr_naming(name, rename, standard/daily/anime_episode_format, series_folder_format, season_folder_format, replace_illegal_characters, colon_replacement_format INT, custom_colon_replacement_format, multi_episode_style INT)`. The int enums are converted in `src/lib/shared/pcd/mediaManagement.ts`.
- `radarr_media_settings` / `sonarr_media_settings(name, propers_repacks ∈ doNotPrefer|preferAndUpgrade|doNotUpgradeAutomatically, enable_media_info)`
- `radarr_quality_definitions` / `sonarr_quality_definitions(name, quality_name, min_size, max_size, preferred_size)`. Primary key is `(name, quality_name)`; 0 means unlimited.

**QP testing**
- `test_entities(type ∈ movie|series, tmdb_id, title, year, poster_path)` UNIQUE(type, tmdb_id)
- `test_releases(entity_type, entity_tmdb_id, title, size_bytes, languages JSON, indexers JSON, flags JSON)`

**Portable (JSON) forms** of every entity exist in `src/lib/shared/pcd/portable.ts`: `ENTITY_TYPES` = delay_profile, regular_expression, custom_format, quality_profile, radarr/sonarr_naming, radarr/sonarr_media_settings, radarr/sonarr_quality_definitions. Clone uses them today; import/export is behind a flag.

### 2.2 Storage format of PCDs (git repos)

A PCD repo (`docs/backend/pcd.md`, `src/lib/server/pcd/manifest/manifest.ts`, `…/ops/importBaseOps.ts`, `…/git/dependencies.ts`):

- `pcd.json` manifest.
  - Required: `name`, `version`, `description`, `profilarr.minimum_version`.
  - Optional: `dependencies` (must include a `/schema` repo, as a semver range), `arr_types` (validated against radarr, sonarr, readarr, lidarr, prowlarr, whisparr), `authors`, `license`, `repository`, `tags`, `links`.
- `ops/*.sql`: **append-only SQL files** holding the canonical base layer. Files are numbered by prefix. Optional header metadata lines use the form `-- @operation: …`, `-- @entity: …`, `-- @name: …`.
- `tweaks/*.sql`: optional repo-local SQL tweaks layer. The UI for it is behind a feature flag.
- `deps/schema/`: the schema dependency repo, cloned at a git **tag** matching the version. Only `.git`, `ops/` and `pcd.json` are kept.
- `announcements/<ulid>.md`: per-PCD announcements with YAML front-matter.

So the dataset itself is **SQL DML**, not YAML/JSON. v1 Profilarr used YAML (§10).

### 2.3 App DB — `src/lib/server/db/schema.sql` (72 migrations)

| Group | Tables (key columns) |
|---|---|
| Arr instances | `arr_instances(name, type, url, external_url, api_key, tags JSON, enabled [always 1], library_refresh_interval, library_last_refreshed_at)`. Duplicates are detected by normalised `type+url` (`utils/arr/url.ts`). |
| Linked PCDs | `database_instances(uuid, name, repository_url, personal_access_token, is_private, local_ops_enabled, git_user_name/email, conflict_strategy ∈ override|align|ask, local_path, sync_strategy [minutes, 0=manual], auto_pull, enabled, last_synced_at)` |
| Op log | `pcd_ops(database_id, origin ∈ base|user, state ∈ published|draft|superseded|dropped|orphaned, source ∈ repo|local|import, filename, op_number, sequence, sql, metadata JSON, desired_state JSON, content_hash, last_seen_in_repo_at, superseded_by_op_id, pushed_at, pushed_commit)` |
| Op history | `pcd_op_history(op_id, database_id, batch_id, status ∈ applied|skipped|conflicted|conflicted_pending|error|dropped|superseded, rowcount, conflict_reason, error, details)` |
| Sync selections | `arr_sync_quality_profiles(instance_id, database_id, profile_name)`; `arr_sync_database_priority(instance_id, database_id, priority)`; `arr_sync_quality_profiles_config` / `arr_sync_delay_profiles_config(database_id, profile_name)` / `arr_sync_media_management(naming_/quality_definitions_/media_settings_ database_id + config_name)`, each with `trigger, cron, next_run_at, sync_status, last_error, last_synced_at` |
| Arr automation | `upgrade_configs(filters JSON, filter_mode, current_filter_index, cron, dry_run)`, `upgrade_runs`, `arr_rename_settings`, `rename_runs`, `arr_cleanup_settings`, `arr_drift_settings`, `arr_drift_status(diff_json, diff_hash, notification hashes)` |
| Jobs | `job_queue(job_type, status, run_at, payload, source, dedupe_key UNIQUE partial, attempts)`, `job_run_history` (plus dead legacy `jobs`, `job_runs`, §10) |
| Notifications | `notification_services(id uuid, service_type, config JSON, enabled_types JSON)`, `notification_history` |
| Auth | `users`, `sessions` (UA/IP metadata, sliding expiry), `auth_settings`, `api_keys(key_hash bcrypt, key_hint, permissions JSON, expires_at)`, `login_attempts` |
| Settings | `general_settings(date_format, apply_default_delay_profiles, fail_on_referenced_delete, sync_prompt_enabled)`, `log_settings`, `backup_settings`, `tmdb_settings(api_key)`, `ai_settings` (feature off), `setup_state(default_database_linked)` |
| Caches | `parsed_release_cache(cache_key, parser_version)`, `pattern_match_cache(title, patterns_hash)`, `regex101_cache`, `github_cache` |
| Announcements | `announcements` (bulletin), `versions_snapshot`, `database_announcements` |

The migration runner is in `src/lib/server/db/migrations.ts`. Migrations support `up`/`down`/`afterUp`, and the `migrations` table tracks what has been applied. Number 068 is missing from the sequence.

---

## 3. Feature areas

### 3.1 PCD engine — git-backed, op-sourced config DB (`src/lib/server/pcd/`, `docs/backend/pcd.md`)

- **Link** (`core/manager.ts:link`, job `pcd.link`) runs these steps:
  1. Clone the GitHub repo, with an optional PAT and branch. Private repos are detected via `api.github.com`.
  2. Validate `pcd.json`.
  3. Clone the dependencies at the version tag.
  4. Import `ops/*.sql` into `pcd_ops` as `origin=base, state=published, source=repo`.
  5. Compile the cache.
- On first boot the app **auto-links** `https://github.com/Dictionarry-Hub/database` on branch `v2`, auto-checking every 60 minutes with auto-pull on (`src/hooks.server.ts`).
- **Compile** (`database/compiler.ts`, `database/cache.ts`, `ops/loadOps.ts`) replays four layers in order: schema files, then base ops (published, then drafts offset by `DRAFT_SEQUENCE_BASE=3e9`), then tweaks files, then user ops. The result is an in-memory SQLite database.
  - Helper SQL functions `qp()`, `cf()`, `dp()` and `tag()` resolve names to ids at replay time.
  - Rowcounts are tracked per op. For user ops, the result is written to `pcd_op_history`.
  - The caches are kept in a global registry (`database/registry.ts`) and **rebuilt after every write**. Each costs roughly 30–50 MB of memory per database (`docs/ARCHITECTURE.md`).
- **Writer** (`ops/writer.ts`) is the single entry point for every mutation. Entity code builds Kysely queries against the cache and compiles them to literal SQL strings (`utils/sql.ts`). Some entity code builds SQL with `esc()` string escaping instead (e.g. `qualityProfiles/general/update.ts`). The writer then:
  - validates the SQL inside a SAVEPOINT dry-run against the current cache;
  - stores the SQL together with `metadata` (operation, entity, name, previousName, stableKey, groupId, changedFields, dependsOn, generated, qualityName) and `desired_state` (from/to pairs);
  - stores a SHA-256 content hash;
  - **cancels out** a create followed by a delete when nothing depends on the create.
- **Layers / roles**:
  - The **user layer** holds end-user local tweaks. It is never exported and survives upstream pulls.
  - The **base layer** holds a PCD developer's drafts. It needs a PAT and write access. Drafts are listed on the Changes page (`ops/draftChanges.ts`, ~1k LOC), then exported as numbered SQL files, validated against a clean base view, committed and pushed to GitHub (`ops/exporter.ts`: `commit`, `git push origin HEAD:refs/heads/<branch>`).
  - `local_ops_enabled` forces the user layer even when a PAT is present.
- **Value guards and op splitting.**
  - UPDATE/DELETE statements carry `WHERE field = old_value` guards. A rowcount of 0 means a conflict.
  - Multi-field edits are split into one op per field. Media settings, naming and delay profiles are split with constrained pairs; quality definitions are split per tier row.
  - Ordered lists (QP qualities, scoring) are batched and then checked by `conflicts/fullListCheck.ts`.
- **Conflict resolution** (`conflicts/`) has three strategies, set per database in `conflict_strategy`:
  - **override** (default): the user wins. The original op is dropped, the cache recompiled, and an entity-specific handler in `entities/*/override/` regenerates the op from `desired_state`. Up to 10 rounds run, and old ops are linked via `superseded_by_op_id`.
  - **align**: upstream wins and the user op is dropped.
  - **ask**: the op is marked `conflicted_pending` and resolved in the UI at `/databases/[id]/conflicts` (actions `align`/`override`).
  - Auto-align rules in `conflicts/autoAlign/rules/`: `defaultFieldGuardRule`, `missingTargetDeleteRule`, `qualityProfileQualitiesRowRule`, `qualityProfileScoringRowRule`.
  - Conflict reasons: `guard_mismatch`, `duplicate_key`, `missing_target`.
- **Pull/update** (`manager.sync`, job `pcd.sync`):
  1. `git fetch`/compare, then pull.
  2. Sync dependencies when the schema version changed.
  3. Re-import base ops. Missing files are marked `orphaned`.
  4. Recompile.
  5. Fire `on_pull` Arr syncs.
  6. Reconcile PCD announcements.
  7. Notify.
- **Git plumbing** (`src/lib/server/utils/git/`): a git CLI wrapper with `GIT_TERMINAL_PROMPT=0`. It provides branches, status, commits, diff, incoming changes and switch branch. The UI pages are `/databases/[id]/commits`, `/changes` and `/config` (edits `pcd.json`).
- **Entity-level operations**:
  - CRUD with cascades (`entities/**`), for example: a CF rename rewrites QP scoring refs; a regex rename rewrites `condition_patterns`; deletes emit explicit dependent removals.
  - `general_settings.fail_on_referenced_delete` blocks deletion of a referenced CF or regex.
  - Clone via serialize → rename → deserialize (`entities/clone.ts`).
  - Import/export of portable JSON (`/databases/[id]/import|export`) exists but is gated by `FEATURES.importExport=false`.
- **Schema evolution** (`docs/backend/schema-bump.md`) is additive-only: new numbered schema ops, semver on the schema repo, and graceful fallbacks in code, e.g. `hasQualityGroupMembersPositionInCache()`.
- **Type generation**: `scripts/generate-pcd-types.ts` introspects the schema SQL (including CHECK constraints) to produce `src/lib/shared/pcd/types.ts`.

### 3.2 Editing UX (routes)

Every entity type has a database picker, a list and a detail page. The `first` endpoints redirect to the first database.

- **Quality profiles**: `/quality-profiles/[databaseId]/[id]/{general,qualities,scoring}`.
  - *general*: name, markdown description, tags, language.
  - *qualities*: drag-order list of qualities and groups. `GroupModal.svelte` creates groups; there are enable toggles and a single "upgrade until" marker.
  - *scoring*: `ScoringTable{Desktop,Mobile}.svelte` shows every CF in the DB with **separate Radarr and Sonarr score columns** and tag filtering, plus profile-level min score, upgrade-until score and increment.
- **Custom formats**: `/custom-formats/[databaseId]/[id]/{general,conditions,testing}`. `ConditionCard.svelte` edits each condition: type, value, negate, required and arr_type. Conditions are sorted required → negated → optional, then by type (`sortConditions`). *testing* shows per-test pass/fail with a per-condition breakdown.
- **Regular expressions**: `/regular-expressions/[databaseId]/[id]` with a regex101 link. `regex101/[...id]` proxies and caches regex101 tests; `validate` compiles the pattern on the .NET parser.
- **Delay profiles**: `/delay-profiles/[databaseId]/[name]`.
- **Media management**: `/media-management/[databaseId]/{naming,media-settings,quality-definitions}/{radarr|sonarr}/[name]`. Naming has token reference, preview and validation (`src/lib/shared/pcd/namingTokens.ts`, ~1.2k LOC of Radarr/Sonarr token tables, sample resolver and validator).
- **Sync prompt**: after saving an entity, the UI can push it to the affected Arrs immediately. This uses `sync/affectedArrs.ts` (graph regex → CFs → QPs → instances) and `sync/entitySync.ts`, and is toggled by `general_settings.sync_prompt_enabled`.
- **Dirty-state / navigation guard**: `docs/frontend/dirty.md`. **Onboarding "cutscene" system**: `src/lib/client/cutscene/`, `FEATURES.cutscene`.

### 3.3 Testing

- **CF testing** (`routes/custom-formats/[databaseId]/[id]/testing/+page.server.ts`) works per test case:
  1. The parser `/parse` extracts metadata.
  2. `/match` runs the CF's patterns against title, edition and group.
  3. `evaluateCustomFormat()` in `pcd/entities/customFormats/evaluator.ts` produces the result. It **re-implements Arr's CF match semantics**: AND across condition types; within a type, all `required` conditions must pass, otherwise at least one must pass (OR); `negate` is applied per condition.
- **QP entity testing** (`/quality-profiles/entity-testing/[databaseId]`, evaluate endpoint `…/evaluate/+server.ts`):
  - Test entities come from **TMDB search** (`routes/tmdb/search`), manual release entry, or **import of real releases from an Arr's interactive search**. The import uses Radarr `GET /release?movieId=` and Sonarr `GET /release?seriesId=&seasonNumber=`, grouped by `utils/arr/releaseImport.ts`.
  - Evaluation runs batch parse → batch match → CF evaluation for every release and every CF. The client computes scores with `allCfScores()` and ranks releases.
- **Regex testing**: `.NET` validation plus regex101 test cases (cached in `regex101_cache`).
- **Parser caches**: `parsed_release_cache` is invalidated on parser version change; `pattern_match_cache` is keyed by a SHA-256 of the pattern set.

### 3.4 Parser service (C#, `src/services/parser/`, `docs/backend/parser.md`)

- It is a .NET 8 minimal API with endpoints:
  - `POST /parse {title,type}`
  - `POST /match {text,patterns[]}`
  - `POST /match/batch`: patterns are compiled once and evaluated with `Parallel.ForEach`
  - `POST /validate/regex`
  - `GET /health`
  - Every regex runs with a 100 ms timeout.
- **What it parses** (`Parsers/*.cs`):
  - `QualityParser`: source, resolution, modifier, revision/repack.
  - `TitleParser`: movie titles, year, edition, IMDb/TMDb ids, hardcoded subs, hash.
  - `EpisodeParser`: 30+ patterns for season/episode, absolute, daily, mini-series, season packs.
  - `LanguageParser`: 59 languages.
  - `ReleaseGroupParser`.
- The regexes are visibly derived from Radarr/Sonarr's `QualityParser`/`Parser`, for example the named groups `bluray|webdl|webrip|…` and `// Resolution-only releases fallback to HDTV/SDTV in Sonarr/Radarr` (`Parsers/QualityParser.cs:240`). **This is a partial port of the Arr parsers**, kept in a separate process so matching uses the .NET regex engine.
- Why it exists: JavaScript regex ≠ .NET regex, and the CF patterns are .NET syntax.
- TS client (`src/lib/server/utils/arr/parser/client.ts`): pooled, 30 s timeout, 2 retries, health check cached for 30 s, graceful degradation when the parser is offline.

### 3.5 Sync to Arrs (`src/lib/server/sync/`, `docs/backend/sync.md`)

Details are in §9.

- There are three **sections**, registered through `registry.ts` `SectionHandler`: `qualityProfiles` (pulls its CFs along), `delayProfiles`, and `mediaManagement` (naming, media settings, quality definitions).
- **Triggers** per section:
  - `manual`: UI actions or `POST /api/v1/databases/{id}/sync`.
  - `schedule`: cron.
  - `on_pull`: fired after a PCD git pull.
  - `none`.
  - `on_change` is declared in the schema but unused.
- **Processor** (`processor.ts`): claims a section atomically (`sync_status` pending → in_progress), processes 3 instances in parallel (`CONCURRENCY_LIMIT`), runs sections sequentially, and recovers interrupted syncs on startup (`recoverInterruptedSyncs`).
- **Entity sync**: inline single-entity push (`entitySync.ts`).
- **Config cleanup** (`cleanup.ts`, job `arr.cleanup`): deletes every CF and QP on the Arr that isn't in the current selections.
- **Entity cleanup** (`entityCleanup.ts`): reads the Arr `/health` for `RemovedMovieCheck`/`RemovedSeriesCheck`, regex-extracts TMDB/TVDB ids, and offers to delete that media.
- **Resync** (`routes/arr/[id]/resync`).

### 3.6 Drift detection (`src/lib/server/drift/`, job `arr.drift`, `docs/backend/drift.md`)

- It rebuilds the expected Arr payloads with the sync transformers, fetches the live Arr state, and diffs by name.
- Coverage: CFs, QPs, the default delay profile, naming, media settings and quality definitions.
- Only the latest result is stored, in `arr_drift_status`. Notifications fire on hash change. The UI is `/arr/[id]/drift`, and progress chips appear on the sync page.
- It is read-only and never repairs. Gated by `FEATURES.drift=true` and per-instance settings.

### 3.7 Upgrades — scheduled "search for better releases" (`src/lib/server/upgrades/`, `src/lib/shared/upgrades/`, `docs/backend/upgrades.md`)

- **Filters** are nested AND/OR groups (`filters.ts`, 1.3k LOC). Fields include:
  - monitored, cutoff_met, title, quality_profile, tags, genres, keywords, year, rating
  - imdb/tmdb/tomato/trakt ratings, popularity, size_on_disk
  - date_added, digital/physical_release, first/last_aired, status, minimum_availability
  - studio, collection, network, certification, series_type, season/episode counts, release_group
  - custom_format (Radarr only)
  - original_language, runtime
- **Selectors**: random, oldest, newest, lowest_score, size_desc/asc, most/least_popular, alphabetical.
- Round-robin or random filter rotation.
- **Cooldown via Arr tags** `profilarr-<filter>`: Radarr applies them with `PUT /movie/{id}`, Sonarr with `PUT /series/editor applyTags`.
- Searches: Radarr `MoviesSearch`, Sonarr `SeriesSearch`. It then watches the Arr `/queue`.
- Dry run fetches `/release` and compares CF scores. Filter preview is available.

### 3.8 Rename (`src/lib/server/rename/`, job `arr.rename`)

- Uses the Arr rename preview (`GET /rename?movieId|seriesId`), then the commands `RenameMovie` (batch 50) / `RenameSeries` (batch 10).
- Optional folder rename via `PUT /movie/editor` / `PUT /series/editor {rootFolderPath, moveFiles:true}`, waiting for the spawned "Bulk Move" command.
- Takes before/after snapshots to diff what changed. Supports an ignore tag and dry run.

### 3.9 Arr instance utilities

- `/arr/[id]/library`: a cached library browser with interactive-search releases; job `arr.library.refresh` on an interval.
- `/arr/[id]/logs`: an Arr log viewer (`/log`, `/log/file`, `/log/file/{name}`).
- `/arr/new`: validates via `system/status`. It optionally applies the default delay profile to id=1 (`utils/arr/defaults.ts`, `general_settings.apply_default_delay_profiles`).

### 3.10 Jobs / scheduling

A SQLite queue with a single serial dispatcher, timeout-based rather than polling. Details in §6.

### 3.11 Backups

- Job `backup.create` builds a tar.gz of the data dir. The DB copy inside is **sanitised**: API keys, PATs, webhook URLs, users and sessions are stripped (`utils/backup/sanitize.ts`).
- Retention cleanup and upload/restore are supported. Restore is staged and applied at the next startup before the DB opens (`utils/backup/applyPending.ts`).
- Path-traversal checks on archives are in `utils/backup/archive.ts`.

### 3.12 Auth / security (`src/lib/server/utils/auth/`, `docs/backend/security.md`)

- `AUTH=on|off` (`oidc` is a deprecated alias).
- A local password account (bcrypt) with server-side sessions: sliding expiry, default 168 h.
- Optional OIDC SSO through `OIDC_DISCOVERY_URL`/`CLIENT_ID`/`SECRET`.
- Named, scoped **API keys** with per-area read/write, bcrypt-hashed, with expiry, driven by `x-permission` in the OpenAPI spec. There is also an env `PROFILARR_API_KEY` (≥32 chars, full access).
- API keys work only on `/api/v1/*`.
- Login rate limiting (`suspicious`/`typo`/`unknown` categories), CSRF via SvelteKit origin checks, `ORIGIN` for reverse proxies, and markdown XSS sanitising.
- Security scanning in CI: Semgrep rules, Trivy and ZAP (`tests/scan/`).

### 3.13 Logging

- Singleton logger (`src/lib/server/utils/logger/`) writing coloured console lines and daily JSON-lines files `logs/YYYY-MM-DD.log`.
- Settings in `log_settings`. Retention cleanup runs as job `logs.cleanup`. There is a UI viewer at `/settings/logs`.

### 3.14 Notifications (`src/lib/server/notifications/`)

- Notifiers **implemented**: Discord (embeds), Telegram, ntfy, generic Webhook. The README also lists Slack, Pushover, Gotify and Apprise, but there are no dedicated notifiers for these (presumably the generic webhook covers them).
- Event types (`src/lib/shared/notifications/types.ts`):
  - `backup.success|failed`
  - `pcd.linked|unlinked|updates_available|sync_success|sync_failed|link_failed`
  - `arr.sync.success|partial|failed`
  - `arr.cleanup.*`, `arr.drift.detected|failed`
  - `upgrade.*`, `rename.*`
  - `announcement.new`, `test`
- History is kept in `notification_history`.

### 3.15 Announcements / update check (`src/lib/server/announcements/`)

- Fetches `versions.json` and `announcements.json` from `Dictionarry-Hub/bulletin` every 30 min, used for the version check and an inbox.
- Per-PCD announcements come from `announcements/*.md` in each linked repo.

### 3.16 Feature-flagged or unfinished (`src/lib/shared/features.ts`)

- `importExport:false`
- `ai:false`: AI commit message generation; `ai_settings`, `utils/ai/client.ts` (OpenAI default)
- `tweaks:false`: tweaks UI
- `docs:false`
- `drift:true`, `cutscene:true`

---

## 4. Integrations (full lists)

| Integration | Details | Status |
|---|---|---|
| **Radarr** | API `/api/v3` (`BaseArrClient.apiVersion='v3'`, `utils/arr/base.ts`). README requires **Radarr v5+**. No version check is enforced at add time (`routes/arr/validate/+server.ts` only calls `system/status`). | Active |
| **Sonarr** | API `/api/v3`. README requires **Sonarr v4+**: sync assumes v4 semantics, with no QP language, CF-based language, and `/qualitydefinition/update`. Sonarr v3 is not supported. | Active |
| Lidarr | `clients/lidarr.ts` is an empty stub (`apiVersion='v1'`), listed in `ArrType` and `factory.ts`. Cannot be added (`VALID_TYPES=['radarr','sonarr']`). | Dead stub |
| Chaptarr | `clients/chaptarr.ts` is an empty stub. | Dead stub |
| Readarr / Prowlarr / Whisparr | Only accepted as manifest `arr_types` values (`manifest.ts`), Discord icons, and dev component demo. Schema comment in `arr_instances.type` mentions readarr/prowlarr. **No Prowlarr integration at all.** Readarr is a retired upstream project. | Vestigial / obsolete |
| GitHub | The only supported git host: `validateRepository` requires `https://github.com/owner/repo`, and PAT injection is hard-coded to `https://<PAT>@github.com` (`utils/git/write.ts`). Also uses `api.github.com/repos/*` (private detection, repo info, avatars; `utils/github/cache.ts` caches in `github_cache`). | Active; GitLab/Gitea etc. not supported |
| Known PCDs | `Dictionarry-Hub/database` (default, branch `v2`), `Dictionarry-Hub/trash-pcd` (TRaSH Guides port), `Dumpstarr/Database`, template `Dictionarry-Hub/database-template`, schema `Dictionarry-Hub/schema` | |
| Bulletin | `raw.githubusercontent.com/Dictionarry-Hub/bulletin/main`, overridable with `PROFILARR_BULLETIN_URL` | Active |
| TMDB | `api.themoviedb.org/3` bearer token: validate, searchMovies, searchTVShows (`utils/tmdb/client.ts`). Used by entity testing. | Active |
| regex101 | `https://regex101.com/api/regex/{id}[/{version}]`, proxied and cached | Active |
| OIDC IdP | Any provider with a discovery URL | Active |
| Notification targets | Discord webhook, Telegram bot API, ntfy, generic webhook | Active |
| AI provider | OpenAI/Anthropic settings table | Feature off |
| Outbound proxy | Standard `HTTP(S)_PROXY`/`ALL_PROXY`/`NO_PROXY` env vars (`utils/http/proxy.ts`) | Active |

---

## 5. External APIs / interfaces

### 5.1 Profilarr's own HTTP API

**Public, versioned, contract-first API**: `/api/v1/*`, spec at `docs/api/v1/openapi.yaml`, generated into `src/lib/api/v1.d.ts`. Auth is a session cookie or `X-Api-Key`.

| Method & path | Summary | Perm |
|---|---|---|
| GET `/api/v1/health` | Health (public) | – |
| GET `/api/v1/status` | System status | read |
| GET `/api/v1/openapi.json` | Spec | read |
| GET `/api/v1/arr` | List Arr instances | read |
| GET / POST `/api/v1/databases` | List / Link database (async `pcd.link` job) | read / write |
| GET / PATCH / DELETE `/api/v1/databases/{id}` | Get / Update / Unlink | read / write |
| POST `/api/v1/databases/{id}/sync` | Trigger PCD pull (+ on_pull arr syncs) | write |
| GET `/api/v1/jobs/{id}` | Job status | read |
| GET / POST `/api/v1/backups`; GET / DELETE `/api/v1/backups/{filename}`; POST `/api/v1/backups/upload`; GET / PATCH `/api/v1/backups/settings` | Backups | read / write |
| GET `/api/v1/announcements`, `/api/v1/announcements/{id}` | Inbox | read |

There is **no public API for entity CRUD (QPs, CFs and so on) or for Arr sync**. Those live only in SvelteKit form actions and page-local endpoints, which are session-only.

**Internal web routes** (session only) under `src/routes`:

- `/jobs/events`: SSE job status.
- Arr: `/arr/validate`, `/arr/[id]/library/{movies,series,…}/releases`, `/arr/[id]/logs`, `/arr/[id]/resync`, `/arr/[id]/settings/cleanup/preview`, `/arr/[id]/upgrades/preview`.
- Databases: `/databases/[id]/{changes,commits}/data`, `/databases/[id]/{import,export}`, `/databases/avatar/[owner]`.
- Testing: `/quality-profiles/entity-testing/[databaseId]/evaluate` (not `/api/v1/entity-testing/evaluate` as `docs/backend/parser.md` says), `/regular-expressions/{validate,regex101/[...id]}`, `/tmdb/{search,validate}`.
- Auth: `/auth/{login,setup,logout,oidc/login,oidc/callback}`.
- `/dev/*`.
- Form actions, for example `arr/[id]/sync` (`saveQualityProfiles`, `saveDelayProfiles`, `saveMediaManagement`, `syncQualityProfiles`, `syncDelayProfiles`, `syncMediaManagement`), `databases/[id]/changes` (`drop`, `commit`, `preview`, `pull`), `databases/[id]/conflicts` (`align`, `override`).

### 5.2 Radarr/Sonarr endpoints called (all `/api/v3`, header `X-Api-Key`)

Shared, from `BaseArrClient` in `utils/arr/base.ts`:

| Endpoint | Use |
|---|---|
| GET `/system/status` | Connection test |
| GET `/health` | Removed-media cleanup (`RemovedMovieCheck`/`RemovedSeriesCheck`) |
| GET / POST / PUT `/delayprofile[/{id}]`, DELETE | Delay-profile sync: **PUT `/delayprofile/1`** with `{id:1, enableUsenet, enableTorrent, preferredProtocol, usenetDelay, torrentDelay, bypassIfHighestQuality, bypassIfAboveCustomFormatScore, minimumCustomFormatScore, order:2147483647, tags:[]}` |
| GET / POST `/tag` | Cooldown tags |
| GET `/config/mediamanagement`; PUT `/config/mediamanagement/{id}` | GET, then merge `downloadPropersAndRepacks`, `enableMediaInfo`, then PUT the full object |
| GET `/config/naming`; PUT `/config/naming/{id}` | Radarr: `renameMovies, replaceIllegalCharacters, colonReplacementFormat(string), standardMovieFormat, movieFolderFormat`. Sonarr: `renameEpisodes, replaceIllegalCharacters, colonReplacementFormat(int), customColonReplacementFormat, multiEpisodeStyle(int), standard/daily/animeEpisodeFormat, seriesFolderFormat, seasonFolderFormat`. GET-merge-PUT. |
| GET `/qualitydefinition`; **PUT `/qualitydefinition/update`** (bulk array) | Sets `minSize/maxSize/preferredSize` for mapped tiers; PCD 0 becomes null |
| GET / POST / PUT / DELETE `/customformat[/{id}]` | CF sync (payload below) and cleanup |
| GET / POST / PUT / DELETE `/qualityprofile[/{id}]` | QP sync (payload below) and cleanup |
| GET `/command`, GET `/command/{id}` | Poll commands (backoff 0.5–5 s, 60 min cap) |
| GET `/log`, `/log/file`, `/log/file/{name}` | Log viewer |

Radarr-specific (`clients/radarr.ts`):

- GET `/movie`, GET `/moviefile?movieFileIds=…` (batched), PUT `/movie/{id}` (tagging), DELETE `/movie/{id}`.
- GET `/release?movieId=` (interactive search, 120 s timeout).
- GET `/queue?page=1&pageSize=200&includeMovie=true`.
- GET `/rename?movieId=`.
- POST `/command` with names `MoviesSearch`, `RenameMovie`, `RefreshMovie`.
- PUT `/movie/editor {movieIds, rootFolderPath, moveFiles:true}`.

Sonarr-specific (`clients/sonarr.ts`):

- GET `/series`, `/series/{id}`, DELETE `/series/{id}`.
- GET `/episode?seriesId=`, `/episodefile?seriesId=`.
- GET `/release?seriesId=&seasonNumber=`.
- GET `/queue?…includeSeries=true`.
- GET `/rename?seriesId=`.
- POST `/command` with names `SeriesSearch`, `RenameSeries`, `RefreshSeries`.
- PUT `/series/editor` for `applyTags` add/remove and `{rootFolderPath, moveFiles}`.

**CF payload** (`sync/customFormats/transformer.ts`):

```
{ id?, name, includeCustomFormatWhenRenaming? (only set when true),
  specifications: [{ name: <condition name>, implementation, negate, required, fields: [...] }] }
```

- `implementation` is one of: `ReleaseTitleSpecification`, `ReleaseGroupSpecification`, `EditionSpecification`, `SourceSpecification`, `ResolutionSpecification`, `IndexerFlagSpecification`, `QualityModifierSpecification`, `SizeSpecification`, `LanguageSpecification`, `ReleaseTypeSpecification`, `YearSpecification`.
- `fields` by type:
  - Patterns: `[{name:'value', value:<regex>}]`.
  - Enums: `value` set to an int from `sync/mappings.ts`.
  - Size: `min`/`max` in GB, rounded to 2 decimals.
  - Year: `min`/`max`.
  - Language: `value` is the Arr language id, plus `exceptLanguage:true` when set.

**QP payload** (`sync/qualityProfiles/transformer.ts`):

```
{ id?, name, items:[{quality{id,name,source,resolution}, items:[], allowed} | {id:1000+n, name, items:[…members allowed:true], allowed}] (reversed; all unused qualities appended allowed:false),
  language? (Radarr only), upgradeAllowed, cutoff:<quality id | group id>,
  minFormatScore, cutoffFormatScore, minUpgradeFormatScore:max(1,increment),
  formatItems:[{format:<arr cf id>, name, score}] (every CF on the instance; unscored = 0) }
```

---

## 6. Background jobs / commands

Job queue: `src/lib/server/jobs/` (`dispatcher.ts`, `queueService.ts`, `schedule.ts`, `handlers/*`), documented in `docs/backend/jobs.md`.

- One job runs at a time, serially. There is no polling: the dispatcher sets a timeout for the next `run_at`.
- A `dedupe_key` unique partial index keeps one scheduled job per key.
- Jobs stuck in `running` are recovered at startup. There is no auto-retry.
- SSE events are pushed to the UI.

| Job type | Payload | Schedule mechanism | Purpose |
|---|---|---|---|
| `arr.sync` | `{instanceId}` | legacy combined | Runs all sections (kept for old queued payloads) |
| `arr.sync.qualityProfiles` / `.delayProfiles` / `.mediaManagement` | `{instanceId}` | cron per section, or `on_pull` event | Push config to the Arr, then chain `arr.drift` |
| `arr.upgrade` | `{instanceId}` | cron (`upgrade_configs.cron`, default `0 */6 * * *`) | Upgrade searches |
| `arr.rename` | `{instanceId}` | cron (default `0 0 * * *`, 10 min minimum) | Bulk rename |
| `arr.cleanup` | `{instanceId}` | cron (default `0 0 * * 0`) | Delete stale CFs/QPs and removed media |
| `arr.drift` | `{instanceId}` | cron (default `0 0 * * *`) | Drift detection |
| `arr.library.refresh` | `{instanceId}` | interval minutes | Refresh library cache |
| `pcd.link` | link options | manual/system | Async clone + import + compile |
| `pcd.sync` | `{databaseId}` | interval (`sync_strategy` minutes) | Check or pull PCD (`auto_pull`) |
| `backup.create` / `backup.cleanup` | `{}` | named (`daily`/`hourly`/`weekly`/`monthly`) or cron | Backups |
| `logs.cleanup` | `{}` | daily | Log retention |
| `announcements.fetch` | `{}` | every 30 min | Bulletin |

Cron expressions are validated against a minimum interval (`jobs/scheduleUtils.ts`).

---

## 7. Configuration surface

- **Environment**: `PUID`, `PGID`, `UMASK`, `TZ`, `PORT` (6868), `HOST`, `APP_BASE_PATH` (/config), `AUTH`, `OIDC_DISCOVERY_URL`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `ORIGIN`, `PARSER_HOST`, `PARSER_PORT`, `PROFILARR_API_KEY`, `PROFILARR_BULLETIN_URL`, `HTTP(S)_PROXY`, `ALL_PROXY`, `NO_PROXY`. Parser-only: `PARSER_LOG_*`.
- **Settings UI** (`src/routes/settings/*`):
  - *general*: date format, apply default delay profiles, fail on referenced delete, sync prompt.
  - *security*: account, SSO, sessions, API keys with scopes.
  - *notifications*: services with per-event subscriptions.
  - *backups*: schedule, retention, include DB.
  - *logs*: level, file/console, retention, viewer.
  - *jobs*: scheduled list, run now, history.
  - *about*.
  - A TMDB key is also configured, and AI settings exist behind the flag.
- **Per-Arr-instance**: connection (name, type, url, external_url, api key, tags), library refresh interval, sync selections and trigger/cron per section, database priority order, cleanup schedule, drift schedule, upgrade config (filters, selector, cron, dry run), rename settings (cron, folders, ignore tag, summary vs rich notifications).
- **Per-database (PCD)**: name, repo URL, branch, PAT, git identity, `sync_strategy` interval, `auto_pull`, `local_ops_enabled`, `conflict_strategy`, and the manifest editor (`/databases/[id]/config`).

---

## 8. Platform/infrastructure concerns

- **Deployment**:
  - Two Docker images: `ghcr.io/dictionarry-hub/profilarr` (Alpine) and `profilarr-parser` (aspnet 8 Alpine), both for linux/amd64 and linux/arm64 (`.github/workflows/release.yml`).
  - Channels `latest`/`develop`/exact/major tag.
  - `tini` plus `scripts/entrypoint.sh` handle PUID/PGID/UMASK and su-exec.
  - Health check `GET /api/v1/health`.
  - Needs Linux kernel ≥3.17, so Synology DSM on kernel 3.10 is unsupported.
- **Binary**: `deno compile --node-modules-dir=none --exclude-unused-npm`. A dynamic `import()` of an npm package would be missing from the binary (`docs/ARCHITECTURE.md`). The build info is stamped into `src/lib/shared/build.ts` by the Dockerfile.
- **DB**: SQLite with `foreign_keys=ON`, WAL, `synchronous=NORMAL`, `busy_timeout=5000`, and `BEGIN IMMEDIATE` transactions (`db/db.ts`).
- **Memory**: each linked PCD keeps an in-memory compiled SQLite cache (~30–50 MB) for the life of the process, and the Arr library cache is ~10–15 MB per large library (`docs/ARCHITECTURE.md`).
- **Runtime dependencies**: the `git` and `tar` CLIs must be present.
- **Secrets**: Arr API keys, PATs, the TMDB key and webhook URLs are stored in **plaintext** in SQLite. They are stripped from backups and from frontend responses (`tests/integration/auth/specs/secretExposure.test.ts`).
- **Tests and CI** (`tests/`):
  - Unit tests cover evaluator, naming tokens, drift, upgrades, auth, backups, sync CF.
  - Integration tests spin up an isolated server per spec on unique ports (~32k LOC), including extensive PCD write and conflict suites (`tests/integration/pcd/{write,conflicts}`).
  - Playwright e2e for OIDC.
  - Semgrep/Trivy/ZAP scans.
  - Custom lints (`scripts/lint/*`: ui, crypto, datetime, clipboard, page-meta, api-permissions).
- **Licence**: AGPL-3.0.

---

## 9. Interactions with other *arr apps (sync semantics)

- **Identity and ID mapping.** Matching is **by name only**; exact, case-sensitive `Map` lookups against `GET` lists (`customFormats/syncer.ts`, `qualityProfiles/syncer.ts`). No Arr ids are persisted. Each run re-reads the Arr, and a name hit becomes a PUT (full replace) while a miss becomes a POST. Retries are idempotent. **Renaming an entity in a PCD creates a new object in the Arr; the old one stays until cleanup.**
- **Custom formats.**
  - Only CFs **referenced by selected QPs** for that arr_type are synced (`pcd/references.ts:getCustomFormatsForProfile`; `arr_type IN (<type>,'all')`).
  - Existing CFs are **overwritten completely**: specifications replaced, and `includeCustomFormatWhenRenaming` only set when true.
  - Conditions whose `arr_type` doesn't match are dropped, as are `quality_modifier` on Sonarr and `release_type` on Radarr.
  - A pattern condition's regex **is inlined** as the spec value. The regex entity doesn't exist in the Arr.
  - Enums are translated through per-app integer tables (`sync/mappings.ts`). Examples: Radarr `SOURCES` (cam 1 … bluray 9) vs Sonarr (television 1 … bluray_raw 7); indexer flags differ per app (e.g. `internal` = 32 on Radarr vs 8 on Sonarr); language ids differ per app (e.g. Arabic 31 on Radarr vs 26 on Sonarr).
- **Quality profiles.**
  - Quality names go through the PCD `quality_api_mappings` table, then fall back to a hard-coded `mapQualityName` (Sonarr `Bluray-1080p Remux` vs Radarr `Remux-1080p`, plus BR-DISK aliases).
  - Quality ids, sources and resolutions come from **hard-coded `QUALITIES` tables** mirroring each Arr's quality enum (Radarr ids 0–31, Sonarr 0–22).
  - Group ids are `1000+n`. Items are reversed. Every quality not mentioned is appended as disallowed. `cutoff` is the id of the item marked `upgrade_until`.
  - `formatItems` must list **every** CF on the instance, so unmanaged CFs get score 0. This overwrites user scores on non-Profilarr CFs inside managed profiles.
  - **Language** is set only for Radarr (`any` by default). Sonarr v4 has no QP language.
  - Per-arr score resolution prefers a specific `arr_type` row over `all`. The code relies on row order without an ORDER BY (`qualityProfiles/transformer.ts`, `cfScoresMap`), which is a latent bug.
- **Multiple databases.** QP selections can come from several PCDs. They are synced in ascending priority (lowest first) so that the highest-priority database's same-named CFs and QPs overwrite earlier ones (`arr_sync_database_priority`). Entity sync ignores priority.
- **Delay profile.** Only **the default profile id=1** is overwritten (PUT). Order is forced to max-int and tags to empty. Tagged delay profiles are unsupported by design.
- **Media management.** GET-merge-PUT: only managed fields are replaced, so unmanaged Arr settings are preserved. Quality definitions are matched by API name. Unmapped or missing tiers are reported, not created.
- **Deletes.** Normal sync never deletes. **Cleanup** deletes *all* CFs and QPs on the Arr that aren't in current selections, including ones the user created by hand in the Arr. CFs are deleted first, then QPs. A QP still assigned to media returns HTTP 500 and is skipped.
- **Ordering and failure.** Within an instance, CFs are pushed before QPs because QPs need CF ids. Errors are isolated per item. There is no rollback, and a section's status can be `success`/`partial`/`failed`.
- **Drift.** Compares expected (transformer output) with the actual Arr state. It ignores ids and unmanaged extras, but reports unmanaged CF scores ≠ 0 inside managed QPs.
- **Sonarr vs Radarr differences** handled in code:
  - enum tables, quality names and ids, language ids, sources, indexer flags;
  - QP language (Radarr only);
  - CF spec availability (Radarr: quality modifier, edition; Sonarr: release type);
  - naming field sets and Sonarr int enums;
  - rename batch sizes (50 vs 10);
  - search commands per movie batch vs per series;
  - cooldown tagging (per-movie PUT vs series editor);
  - Sonarr CF score averaged over episode files for upgrades.

---

## 10. Outdated / legacy candidates

| Item | Evidence |
|---|---|
| **Profilarr v1 (Python/Flask + YAML files, git-backed)** | The original tree had `backend/app/*.py`, a Flask Dockerfile and `backend/app/git/...` (git history, first commit 2023-09-29; tags `v0.1…v1.1.1`). v2 is a full rewrite (commit "chore(setup): sveltekit + deno", 2025-10-18). The default database branch is now `v2` (`src/hooks.server.ts`). No v1 code or migration path remains in the tree. |
| Lidarr / Chaptarr clients | Empty classes in `utils/arr/clients/{lidarr,chaptarr}.ts`; `ArrType` still includes them; `utils/arr/README.md` describes an unimplemented "future usage". |
| Readarr / Prowlarr / Whisparr mentions | Allowed only in manifest `arr_types` (`pcd/manifest/manifest.ts:104`), Discord icons, dev demo, and an `arr_instances.type` comment. Readarr is retired upstream. There is no Prowlarr support at all. |
| `jobs` / `job_runs` tables | Migration 004 created them. The replacement is `job_queue`/`job_run_history` (migration 049), yet the old tables are still in `schema.sql` and nothing references them in code. |
| `arr.sync` combined job type | `handlers/arrSync.ts` `parseLegacySections`; `jobs/schedule.ts` "Remove legacy combined scheduled jobs". |
| `should_sync` columns | Marked "deprecated" in `schema.sql` (migration 016), superseded by `sync_status` (034). |
| `trigger='on_change'` / `'manual'` | Declared in the migration 015 comment. Only `on_pull` is implemented (`sync/types.ts` `SyncTriggerEvent='on_pull'`). |
| `arr_instances.enabled` | "Always 1; retained for compatibility" (migration 067 re-enables all rows). |
| `database_instances.enabled` | Still checked for compile, but there is no UI to disable. |
| `backup_settings.compression_enabled` | Comment says "UNUSED: stored but not read". |
| `upgrade_runs.cooldown_hours` | Deprecated, written as 0 (`db/queries/upgradeRuns.ts:121`, migration 031/032). |
| `quality_profile_languages.type` (`must`/`only`/`not`) | Writers always write `simple` (`qualityProfiles/general/update.ts`), and sync ignores the type. Looks like a v1 concept (v1 generated language CFs). |
| `AUTH=oidc` alias | Deprecated, scheduled for removal in v3.0.0 (`hooks.server.ts`, `utils/config/auth.ts`). |
| `local bypass` auth | Removed (migration 071). |
| Deprecated Arr types | `utils/arr/types.ts:166,169,192` `@deprecated` aliases. `parser/client.ts:172` "legacy - use parse()". |
| Legacy override payload shapes | `quality-definitions/override.ts:74`, `qualityProfiles/override/qualities.ts:12` accept old `desired_state` shapes. |
| Stale docs and snapshots | `schema.sql` still shows `arr_sync_delay_profiles_config.profile_id`, but migration 045 changed it to `profile_name`. `docs/backend/parser.md` gives a wrong entity-testing API path. The README lists notifiers (Slack/Pushover/Gotify/Apprise) that have no dedicated implementation. `docs/backend/jobs.md` says "upgrade … for Radarr instances" while Sonarr is supported. |
| Feature-flagged dead-ish code | AI commit messages (`utils/ai/client.ts`, `ai_settings`), import/export, tweaks UI (`src/lib/shared/features.ts`). |
| Sonarr v3 / Radarr < v5 | Unsupported (README). All clients hard-code `/api/v3`, which is the current API for Radarr v5 and Sonarr v4, so any future Sonarr v5 API change would need work. |

---

## 11. Rewrite notes (Go monolith)

**Worth keeping as the design:**

1. **The PCD relational schema** (`docs/backend/0.schema.sql`) is a clean, app-neutral model of profiles:
   - canonical `qualities` with per-app `quality_api_mappings`;
   - CFs with typed, polymorphic conditions (`arr_type`, `negate`, `required`);
   - regexes as first-class shared, tagged entities referenced by name;
   - **per-arr-type CF scores** in a single profile (`quality_profile_custom_formats.arr_type`);
   - profile-scoped quality groups with ordered members and a single `upgrade_until`;
   - embedded CF test cases and QP test entities/releases.

   This is a better unified model than Sonarr's and Radarr's separate tables. In a monolith, one profile definition can serve movies and series, and the `quality_api_mappings` layer disappears because the monolith owns the quality enum.
2. **Name-based stable keys** plus **append-only ops with value guards, op splitting, desired-state, and override/align/ask conflict strategies.** This is a working model for "upstream curated dataset + local overrides that survive updates" (`src/lib/server/pcd/ops`, `conflicts/`). It is complex (about 20k LOC plus heavy tests), but if git-distributed profile packs remain a product goal, this design is the one to keep. A Go rewrite could simplify it by storing overrides as structured field-level patches rather than guarded SQL strings.
3. **Git-based profile distribution**:
   - `pcd.json` manifest with a semver'd schema dependency;
   - numbered `ops/*.sql`;
   - publishing drafts to git from the UI;
   - on-pull auto-update;
   - per-DB priority when several datasets define the same names.

   The existing ecosystem (Dictionarry, TRaSH `trash-pcd`, Dumpstarr) already publishes in this format, so compatibility is valuable. Using SQL as the interchange format ties datasets to SQLite semantics; the portable JSON types (`src/lib/shared/pcd/portable.ts`) are a candidate for a neutral format.
4. **Testing UX**:
   - CF test cases with a per-condition pass/fail breakdown;
   - regex101 integration and .NET-compatible regex validation;
   - **QP simulation against real or imported releases** (entity testing).

   In a monolith, these can call the app's own parser and CF engine directly.
5. **Drift display model** (`src/lib/shared/drift.ts`) and the **upgrade filter/selector DSL** (`src/lib/shared/upgrades/filters.ts`, `selectors.ts`) are reusable as native "smart search" features.

**Accidental complexity caused only by syncing into external apps (drop in a monolith):**

- The whole `src/lib/server/sync/` tree (~4.9k LOC): transformers, the `sync/mappings.ts` enum and id tables (quality ids, sources, indexer flags, languages per app), name-matching create/update, `formatItems` padding with score 0, group id `1000+n`, item reversal, default delay profile id=1 hacks, GET-merge-PUT of naming and media settings, section status/claim/recovery, per-instance sync schedules, `on_pull` triggers, database priority ordering, entity sync, and affected-Arr impact analysis.
- **Drift detection** (`src/lib/server/drift/`, ~3k LOC) and **config cleanup**: these only exist because the source of truth lives outside the Arr.
- Arr HTTP clients (`utils/arr/`), command polling, and library caches with refresh jobs.
- **Upgrades, rename, removed-media cleanup, library and log viewers.** These re-implement or orchestrate Arr-native features ("Search cutoff unmet", rename preview/organise, `RemovedMovieCheck` health) from outside through the API. They are also likely to duplicate what other sibling tools do. In a monolith they become first-party features of the media library.
- The per-instance `arr_*` settings tables, `arr_instances`, and the API-key storage for Arrs.
- `quality_api_mappings` and `mapQualityName` (Remux naming aliases), plus Sonarr int enum conversions (`src/lib/shared/pcd/mediaManagement.ts`).

**Parsing logic that duplicates Sonarr/Radarr:**

- The **C# parser service** (`src/services/parser/Parsers/*.cs`, ~1.5k LOC) is a partial port of the Radarr/Sonarr parsers (QualityParser, title and episode parsing, LanguageParser, ReleaseGroupParser). It exists only so that CF testing agrees with the Arrs. A Go monolith will have one canonical parser, so this whole service and its caches (`parsed_release_cache`, `pattern_match_cache`) become redundant.
- **Caveat**: CF regexes are written in **.NET regex syntax** (the reason the parser is C#). Go's RE2 lacks lookarounds and backreferences. A Go rewrite needs a .NET-compatible regex engine (e.g. `dlclark/regexp2`) or a migration of the dataset's patterns. This applies to every CF/regex feature in all four apps.
- The **CF evaluator** (`src/lib/server/pcd/entities/customFormats/evaluator.ts`) re-implements Arr CF matching (AND across types; required-AND/else-OR within a type; negate). It duplicates Sonarr's and Radarr's `CustomFormatCalculationService`. Merge them into one engine.
- The **naming token tables, resolver and validator** (`src/lib/shared/pcd/namingTokens.ts`, ~1.2k LOC) duplicate Sonarr's and Radarr's `FileNameBuilder` token logic. Keep only one implementation.
- Upgrade scoring (the Sonarr per-episode-file average score) re-derives what the Arr's decision engine knows.

**Other rewrite considerations:**

- Git host support is GitHub-only (URL regex and PAT injection). Generalise it if git distribution is kept.
- Secrets are stored in plaintext.
- Recompiling the whole in-memory PCD after every write, and holding one cache per database, will not scale well. A monolith with a real persistent profile store should materialise state and apply overrides incrementally.
- Profilarr's own public API is thin: databases, backups, jobs, status. Entity CRUD only exists as SvelteKit form actions, so there is little external API surface to preserve.
