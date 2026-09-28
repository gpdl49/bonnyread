# Design: Profiles and the PCD engine

Scope: the profile store and engine, and its API. This covers how linked profile databases (PCDs) are fetched, pinned and updated; how their contents turn into bonnyread's own tables; how user overrides are stored, and how they survive upstream changes; conflicts; authoring and publishing; the canonical enums; which media kinds each entity applies to; how profiles attach to version slots; and the testing features. How a profile is **applied** to a release (custom-format matching, scoring, ranking, cutoff) belongs to the release-evaluation design. This document only defines the read interface that design consumes (§3.10).

Binding inputs: [decisions.md](../decisions.md), especially D1, D4, D9, D13, D14 and D19. Background: [analysis/profilarr.md](../analysis/profilarr.md), [analysis/recyclarr.md](../analysis/recyclarr.md) and [analysis/comparison.md](../analysis/comparison.md).

Path conventions: `profilarr/…` is `submodules/profilarr/…`, `recyclarr/…` is `submodules/recyclarr/…`, `pcd/…` is `submodules/profilarr/src/lib/server/pcd/…`. The Dictionarry data was measured on **Dictionarry-Hub/database** branch `v2` @ `6266304` (2026-09-27), **Dictionarry-Hub/trash-pcd** @ `4639972` (2026-09-24) and **Dictionarry-Hub/schema** tag `1.1.0`. All three were cloned outside the repo; nothing under `submodules/` was changed.

---

## 1. Outcomes and invariants to preserve

| # | Invariant | Evidence |
|---|---|---|
| I1 | **The PCD on-disk format stays readable and writable** (D9b). A PCD has a `pcd.json` manifest, append-only `ops/*.sql` run in numeric-prefix order (unprefixed files after), optional `tweaks/*.sql`, and a schema dependency pinned by git tag. Existing databases (Dictionarry, trash-pcd, Dumpstarr) must work unchanged, and anything bonnyread publishes must still replay in Profilarr. | `profilarr/docs/backend/pcd.md` §What is a PCD and §Layers; `pcd/ops/importBaseOps.ts:5,57-66` (ordering); `pcd/git/dependencies.ts:57-66` (dependency cloned at a tag; only `.git`, `ops/`, `pcd.json` kept). Dictionarry's `pcd.json` declares `"https://github.com/Dictionarry-Hub/schema": "1.1.0"`. |
| I2 | **Entities are keyed by name**, with foreign keys by name and `ON UPDATE CASCADE`. So an op written anywhere resolves to the same rows everywhere. | `profilarr/docs/backend/0.schema.sql` (every FK is `… REFERENCES …(name) ON DELETE CASCADE ON UPDATE CASCADE`); `profilarr/docs/backend/pcd-entities.md` §Common Patterns (names unique case-insensitively, enforced in app code, not DDL). |
| I3 | **Layering:** schema, then base (published, then drafts), then tweaks, then user. The user layer is private, is never exported, and survives pulls. The base layer is what authors publish. | `pcd/ops/loadOps.ts:36-66`; `pcd.md` §Base vs User. |
| I4 | **Conflict semantics.** Each database has a strategy: `override` (user wins), `align` (upstream wins) or `ask` (manual). A conflict exists when upstream changed the value the user's change was based on (a guard mismatch), when a created entity collides with a new upstream one (duplicate key), or when the target was deleted (missing target). If upstream now equals the user's value, the change is dropped silently ("goal achieved"). | `pcd.md` §Conflicts; `pcd/conflicts/autoAlign/rules/defaultFieldGuard.ts`, `missingTargetDelete.ts`; `pcd/database/cache.ts:120-300`. |
| I5 | **Field-level granularity.** A change to field A never conflicts with an upstream change to field B of the same entity. Ordered lists (profile qualities) are compared as a whole. | `pcd.md` §Op Splitting and §Full-List Conflicts; `pcd/conflicts/fullListCheck.ts`. |
| I6 | **Dictionarry is the default database**, linked on first boot with automatic updates (Profilarr: branch `v2`, 60-minute check, auto-pull). | D9a; `profilarr/src/hooks.server.ts` (profilarr.md §3.1). |
| I7 | **Custom-format scores are per media kind within one profile.** A score applies to all kinds, or to one specific kind that takes precedence. | `quality_profile_custom_formats(…, arr_type, score)` PK includes `arr_type` (`0.schema.sql`); per-type preference in `profilarr/src/lib/server/sync/qualityProfiles/transformer.ts:360-367`. |
| I8 | **Regexes are shared, named entities** referenced by conditions. They are **.NET syntax**, matched **case-insensitively**, with a **100 ms timeout**. | `condition_patterns.regular_expression_name` FK; Radarr/Sonarr `CustomFormats/Specifications/RegexSpecificationBase.cs:34` (`RegexOptions.Compiled \| RegexOptions.IgnoreCase`); `profilarr/src/services/parser/Endpoints/MatchEndpoints.cs:38-41`; D1 (87% of Dictionarry fails RE2). |
| I9 | **Authoring and publishing from the UI:** drafts are exported as the next numbered op file, validated against a clean base, committed and pushed. | `pcd/ops/exporter.ts:147-198` (validation), `:472-473` (`<max+1>.<slug>.sql`), `:507` (push). |
| I10 | **Testing outcomes:** CF unit tests with a per-condition breakdown, and profile simulation that ranks a set of releases for a title. Both live in the data format (`custom_format_tests`, `test_entities`, `test_releases`). | profilarr.md §3.3; `0.schema.sql`; Dictionarry ships 12 test entities with 1,734 test releases. |
| I11 | **Non-destructive defaults** (Recyclarr): never delete what the user didn't create, and never silently zero scores the user set. | recyclarr.md §3.3, §3.7, §11.4; D9 recommendation. |
| I12 | **Schema evolution is additive** and semver'd; readers tolerate columns they don't know. | `profilarr/docs/backend/schema-bump.md` §Golden Rules. |

What the data actually looks like (measured): Dictionarry v2 has 357 op files (5.4 MB). They are not simple DML:
- 0.rosettarr.sql alone is 1.4 MB of `INSERT … SELECT … FROM custom_formats WHERE …`;
- there are 451 `ON CONFLICT ("name") DO NOTHING` clauses and 17 `WITH can_insert AS (…)` CTEs;
- updates carry value guards, e.g. `ops/118.update-dd.sql`: `update "regular_expressions" set "pattern" = … where "name" = 'Dolby Digital' and "pattern" = …`.

Replaying schema 1.1.0 plus all 357 files into SQLite (foreign keys on, one transaction) takes **0.8 s** with the `sqlite3` CLI. It yields 263 CFs, 619 regexes, 11 profiles, 2,326 score rows and 1,899 conditions. trash-pcd (148 files) replays in 0.5 s. With foreign keys **off**, the replay fails (renames stop cascading). This matches Profilarr, which turns FKs on (`pcd/database/cache.ts:83`).

---

## 2. How Profilarr and Recyclarr do it today, and what is accidental

### 2.1 Profilarr (the reference for the format and the semantics)

- **Storage.** Base ops are copied from the repo into the app database (`pcd_ops`, origin `base`), with numbering offsets: unprefixed files get `2e9` (`pcd/ops/importBaseOps.ts:5`) and drafts `3e9` (`pcd/ops/loadOps.ts:11`). Files that vanish from the repo are marked `orphaned` (`importBaseOps.ts:116`).
- **User ops.** These are stored as **literal SQL strings** with value guards, plus `metadata` and `desired_state` JSON and a content hash (`pcd/ops/writer.ts`; `pcd.md` §Writer).
- **Compile.** An **in-memory SQLite database per linked PCD**, rebuilt by replaying *all* layers **after every write** (`pcd/ops/writer.ts:254,291,316` call `compile`). It is held for the life of the process, at roughly 30–50 MB per database (profilarr.md §8).
  - Helper SQL functions `qp()`, `cf()`, `dp()` and `tag()` are registered (`cache.ts:362`).
  - One failing base op **disables the whole database** (`cache.ts:351`).
- **Conflicts.** A conflict is inferred from `rowcount == 0` or constraint errors while replaying user ops. Four auto-align rules exist (`pcd/conflicts/autoAlign/rules/`).
  - Override **regenerates SQL** through an entity-specific handler for each type (`pcd/entities/*/override/`), repeating for "up to 10 rounds" (`pcd.md` §Conflict Strategies).
  - Ordered-list ops need a separate check (`fullListCheck.ts`) because a partly applied multi-statement op still reports `rowcount > 0`.
- **Renames.**
  - Local renames emit generated cascade ops for every referencing row (`pcd-entities.md` §Custom Formats, §Regular Expressions).
  - To survive *upstream* renames, override handlers rebuild a rename map by **regex-matching SQL text** of the form `update "custom_formats" set "name" = … where "name" = …` (`pcd/conflicts/overrideUtils.ts:85-160`, used by `entities/qualityProfiles/override/scoring.ts:151`). This misses any other spelling; for example, the initial Dictionarry op uses unquoted identifiers.
- **History.** `pcd_op_history` writes one row per user op per compile (`cache.ts:221-228`).
- **Git.** It shells out to the `git` CLI. Only GitHub is supported (`profilarr/src/lib/server/utils/git/write.ts:151,209-210`). Dependencies are checked out at an exact tag (`pcd/git/dependencies.ts:60`).
- **Enums.** `quality_api_mappings` plus `sync/mappings.ts` translate canonical names into each Arr's integer ids at sync time. Condition values carry per-Arr applicability (`profilarr/src/lib/shared/pcd/conditions.ts`).
- **Scores.** Editing one Arr's score on an `all` row **splits** it into per-type rows (`pcd-entities.md` §Scoring). Sync resolves the effective score by row order without an `ORDER BY`, a latent bug (`sync/qualityProfiles/transformer.ts:360-367`, profilarr.md §9).
- **Testing.** A .NET parser sidecar (`profilarr/src/services/parser`) with its own caches (`parsed_release_cache`, `pattern_match_cache`), and a TypeScript re-implementation of CF matching (`pcd/entities/customFormats/evaluator.ts`).

### 2.2 Recyclarr (what to learn, not port)

- It stores no data. It shallow-fetches the guide on every run (`recyclarr/src/Recyclarr.Core/VersionControl/GitRepository.cs`; recyclarr.md §3.10).
- Overrides live in YAML with a three-layer precedence of *service < guide < config* (recyclarr.md §3.3). They can't conflict, because upstream data is never stored and so never diffed.
- Useful ideas:
  - scores the user hasn't set are **left alone** (`reset_unmatched_scores` is opt-in);
  - delete only what you created (`delete_old_custom_formats`);
  - stable ids across renames (`trash_id`);
  - "last provider wins" precedence (recyclarr.md §3.10).
- About 70% of its code (state files, adoption, gateways, equivalence diff) exists only because it writes into another app (recyclarr.md §11.1).

### 2.3 Accidental complexity (D19) to drop

| Accidental | Why it existed | Replaced by |
|---|---|---|
| Rebuilding the whole in-memory cache after every write, one cache held per database | Using SQLite as the rules engine for user ops | Effective state persisted in ent tables. A write touches one aggregate; SQLite replay runs only on pull or publish (§3.3). |
| Guarded SQL strings as the override representation, with op splitting, desired-state, content hash and cancel-out | The layers were SQL, so overrides had to be SQL | Structured field patches `(path, base, value)` (§3.4). A guard *is* `base`, so this is equivalent (§3.4.3). |
| Override handlers per entity type that regenerate SQL, with "10 rounds" | SQL can't be rebased, so it was regenerated | Override = set `base` to the new upstream value. No regeneration and no rounds. |
| The rename map rebuilt by regex over SQL text; generated cascade ops | References were names inside SQL | References are entity ids; upstream renames are captured by SQLite triggers during replay (§3.3.2). |
| `pcd_ops` holding copies of base ops, orphan states, sequence offsets | Base ops were replayed from the database | Base ops are read straight from the git tree at the pinned commit. Only a manifest of file names and hashes is stored. |
| A `pcd_op_history` row per op per compile | Status was recomputed on every compile | One event row per *state change* of a patch. |
| `quality_api_mappings`, `sync/mappings.ts`, `mapQualityName`, per-Arr int enums, Sonarr int naming enums in the model | Syncing into two external apps | Canonical Go enums (D9), plus a codec used only at the PCD boundary (§3.7). |
| Sync, drift, cleanup, `on_pull` triggers, database priority for sync ordering, affected-Arr analysis, the Arr import of test releases | External Arrs held the live config | Gone. A profile-change event goes to the library and decision engine (§3.10). |
| The C# parser sidecar and its caches; `evaluator.ts` | JavaScript can't run .NET regex | The in-process parser (D8), regexp2 (D1) and the single CF engine owned by release evaluation. |
| `git` CLI, GitHub-only URLs, a PAT injected into the URL | Convenience | `go-git` over HTTPS with token auth on any host (§3.5). |
| One bad base op disables the database | Compile was all-or-nothing | A failed pull keeps the previous pinned state and raises a health error (§3.5). |

---

## 3. Proposed design

### 3.1 Concepts

- **Database:** a linked PCD (`kind = git`), the built-in **Local** database (`kind = local`, no remote, can be published later), or a bundled bootstrap snapshot (§3.5.5). Every entity belongs to exactly one database, and references never cross databases (§3.6).
- **Upstream state:** the result of replaying *schema, base and tweaks* at the database's **pinned commit**. It is stored as one canonical JSON document per aggregate (§3.2).
- **Patch:** one user or author intent on one path of one aggregate: `(path, base, value)`. There are two patch layers:
  - `local`: the Profilarr "user layer", private;
  - `draft`: the Profilarr "base drafts", publishable.
- **Effective state:** upstream ⊕ draft ⊕ local, materialised in normal ent tables. **This is the only thing the rest of bonnyread reads.**
- **Aggregate** (the unit of diff, patch and cache invalidation):
  - `Regex`;
  - `CustomFormat` (conditions, tests and tags inside);
  - `QualityProfile` (qualities, groups, languages, scores and tags inside);
  - `DelayProfile`;
  - `NamingPreset`;
  - `MediaSettingsPreset`;
  - `QualitySizePreset`;
  - `TestEntity` (with its releases).

  Tags are plain string sets on each aggregate. The PCD `tags` table is only a name registry.

### 3.2 Data model (ent-style sketch, D13)

```go
// ---- identity and provenance ------------------------------------------------
// PcdDatabase: one linked or local database.
field.String("name"); field.Enum("kind").Values("git", "local", "bundled")
field.String("repo_url").Optional(); field.String("branch").Default("main")
field.String("secret_ref").Optional()           // token, stored per D16
field.String("git_author_name").Optional(); field.String("git_author_email").Optional()
field.Enum("conflict_strategy").Values("override", "align", "ask").Default("override")
field.Enum("update_policy").Values("auto", "notify", "frozen").Default("auto")
field.Int("check_interval_min").Default(60)
field.Int("priority")                          // name resolution and display only (§3.6)
field.Bool("writable")                         // may hold drafts and publish
field.String("pinned_commit").Optional(); field.String("remote_head").Optional()
field.JSON("manifest", pcd.Manifest{}); field.String("schema_version")
field.JSON("applied_ops", []pcd.OpFile{})      // {path, sha256} at pinned_commit (rename journal, §3.3.2)
field.Enum("health").Values("ok", "update_blocked", "error"); field.String("last_error").Optional()

// PcdEntity: the stable identity of every aggregate. Its ID is what everything references.
field.Int("database_id"); field.Enum("kind").Values("regex", "custom_format", "quality_profile",
    "delay_profile", "naming", "media_settings", "quality_size", "test_entity")
field.String("name")                           // exact PCD name; unique (database_id, kind, name)
field.String("name_fold")                      // case-folded; local writes must keep it unique
field.Enum("origin").Values("upstream", "local", "detached")
field.Bool("upstream_present")                 // false once upstream deleted it
field.Int64("revision")                        // bumped on every effective change (cache key)
field.JSON("upstream_doc", json.RawMessage{}).Optional() // canonical doc at pinned_commit
field.String("upstream_hash").Optional()
field.JSON("problems", []pcd.Problem{})        // unknown enum, regex compile error, …

// PcdPatch: structured overrides (layer local) and drafts (layer draft).
field.Int("entity_id"); field.Enum("layer").Values("local", "draft")
field.Enum("op").Values("create", "set", "add", "remove", "delete")
field.String("path")                           // typed path, §3.4.1
field.JSON("base", json.RawMessage{}).Optional()   // upstream value when authored; nil = absent
field.JSON("value", json.RawMessage{}).Optional()
field.Enum("state").Values("active", "pending", "published")
field.String("conflict_reason").Optional()     // guard_mismatch | duplicate_key | missing_target
field.JSON("pending_upstream", json.RawMessage{}).Optional() // upstream value awaiting a decision
field.UUID("group_id"); field.Int("author_id"); field.Time("updated_at")
// unique (entity_id, layer, path)

// PcdEvent: audit trail (replaces pcd_op_history).
// {database_id, pull_id?, entity_id?, patch_id?, type: applied|satisfied|rebased|aligned|
//  pending|resolved|detached|renamed|published|pull_failed, detail JSON, at}

// PcdPull: {database_id, from_commit, to_commit, trigger, status, stats JSON,
//           renames JSON, started_at, finished_at}

// ---- effective state (read by the rest of the app) --------------------------
// Each row's ID equals PcdEntity.ID.
// Regex:          pattern, description, regex101_id, tags []string, compile_error?
// CustomFormat:   description, include_in_rename, tags
//   CfCondition:  cf_id, name, type (enum), domain ("*" | kind), negate, required,
//                 regex_id? (edge -> Regex), payload JSON (typed per condition type)
//   CfTest:       cf_id, title, domain, should_match, description
// QualityProfile: description, upgrades_allowed, min_cf_score, upgrade_until_score,
//                 upgrade_score_increment, language (canonical, optional), tags,
//                 items JSON (ordered: quality | group{name, members[]}, enabled, cutoff),
//                 kinds []Kind (derived: kinds whose quality axis the items cover)
//   ProfileScore: profile_id, cf_id (edge), domain ("*" | kind), score
//                 unique (profile_id, cf_id, domain)
// DelayProfile:   protocol preference, delays, bypass flags, min score
// NamingPreset / MediaSettingsPreset / QualitySizePreset: kind + typed fields
// TestEntity:     kind, external ids (tmdb/…), title, year, poster;
//   TestRelease:  title, size, languages[], indexers[], flags[]
```

Design notes:
- **The effective tables are relational where other code queries them.** Scores are queried for "which profiles use this CF?"; conditions have a regex edge for "who uses this regex?". Profile `items` are JSON because they are always read and written whole.
- **Profilarr's nine condition-type tables** collapse into one `CfCondition` row with a typed payload. The polymorphism existed to satisfy SQL constraints in the op format; the codec still reads and writes the nine tables (§3.7).
- **`upstream_doc` lives beside the identity row**, not in a parallel set of relational tables. It is read only for merging, diffing and showing "upstream vs yours". Dictionarry's docs total a few MB.
- **Version slots (D4)** reference `QualityProfile.id`, an edge owned by the library schema. A slot's score domain comes from its media kind (§3.8).

### 3.3 Materialisation

#### 3.3.1 Pull pipeline (leader-only job, §3.9)

```
prepare (no DB locks)                         apply (one transaction, row lock on PcdDatabase)
─────────────────────                         ────────────────────────────────────────────────
1 fetch target commit (go-git, shallow)       7 renames: update PcdEntity.name (IDs kept)
2 read pcd.json, resolve schema dependency    8 upsert upstream_doc for changed/added keys
  (tag checkout, cached per URL@tag)          9 removals: detach if in use, else tombstone (§3.4.4)
3 compatibility gate (schema major, §3.7.3)  10 re-merge every touched entity with its patches;
4 replay schema → ops → tweaks in scratch       classify conflicts; apply the strategy (§3.4.2)
  SQLite (in-memory, FKs on, qp/cf/dp/tag    11 write effective rows; bump revisions
  helpers, rename triggers)                  12 pinned_commit := target; applied_ops := manifest
5 decode tables → canonical docs (codec)     13 events; commit; NOTIFY pcd.changed{db, ids, rev}
6 diff against stored upstream_hash:
  added / changed / removed / renamed
```

- **Replay is full but cheap** (0.8 s for Dictionarry). It happens only on pull, publish validation and "preview update", never on user writes.
- **The work that is incremental is steps 7–11**, which scale with the number of aggregates that changed. A typical Dictionarry commit touches 1–5 aggregates. Diffing is by the hash of each canonical doc.
- **The result is a pure function of `(target commit, patches)`, not of the order of past pulls.** Any target works the same way: forward, a rollback ("pin to previous"), or a branch switch. Rewritten or deleted op files, which Profilarr marks as orphaned, need no special handling.
- **Docs store references as entity ids.** When upstream renames the regex "Dolby Digital", only the Regex aggregate changes; the CFs that use it keep the same doc hash. Enum values are canonical. Values the codec can't map are kept as `{"unknown":"<raw>"}` and recorded in `problems`.
- **Replay engine:** an embedded pure-Go SQLite (`modernc.org/sqlite`), independent of the deployment database. Upstream SQL *is* SQLite dialect (CTEs, `INSERT … SELECT`, `ON CONFLICT`), so a real SQLite is the only faithful interpreter. It is a **format decoder**, not a rules engine.
- **A base op that errors** aborts the pull. The previous pin stays in effect, `health = update_blocked`, and the failing file and error are reported. This differs from Profilarr, which disables the database (`cache.ts:351`).

#### 3.3.2 Upstream rename detection

- Before replay, the engine installs `AFTER UPDATE OF name` triggers on every name-keyed table (`regular_expressions`, `custom_formats`, `quality_profiles`, `delay_profiles`, `*_naming`, `*_media_settings`, and `*_quality_definitions` for the `name` column). They write `(table, old, new, current_file)` into a journal table. `current_file` is set in a one-row temp table before each file runs.
- Renames recorded in files **not in `applied_ops`** are the renames of this pull. Chains are composed (A→B→C gives A→C).
- If the journal finds nothing (for example, upstream deleted and re-inserted under a new name), there is a fallback: a removed key and an added key of the same kind whose docs are equal apart from `name` count as a rename.
- Otherwise it is a genuine delete plus add (§3.4.4).
- Unlike Profilarr's regex over SQL text (`overrideUtils.ts:143-160`), this sees every spelling of a rename, because SQLite executes it.

#### 3.3.3 User write path (any node, no git, no replay)

1. The request carries a full candidate doc (editor save) or field ops (API).
2. Validate the aggregate:
   - the schema's CHECK rules, re-expressed in Go: one cutoff, `upgrade_score_increment > 0`, the delay-profile protocol and delay pairing;
   - references exist in the same database;
   - regexes compile (§3.11);
   - case-insensitive name uniqueness.
3. Compute `diff(effective_without_this_layer, candidate)` at path granularity. For each path:
   - if a patch exists in this layer, update its `value` and keep its `base`. If the value now equals `base`, delete the patch (this generalises Profilarr's cancel-out);
   - otherwise create a patch with `base = upstream value at path` (`nil` if absent).
4. Re-merge this entity only. Write effective rows, bump `revision`, emit an event, then `NOTIFY` after commit.

Layer choice: `local` by default. `draft` if the database is `writable` and the caller has the author permission and chose "edit as draft". The merge order is upstream → draft → local, so a local override also covers an author's own draft.

### 3.4 Override representation and conflicts

#### 3.4.1 Paths

Paths are typed per aggregate. Collection members are keyed by stable keys, never by position:

| Aggregate | Paths |
|---|---|
| any | `name`, `description`, `tags` (`add`/`remove` of one tag: set operations commute and never conflict) |
| Regex | `pattern`, `regex101_id` |
| CustomFormat | `include_in_rename`; `conditions[<name>]` (the whole condition object: type, domain, negate, required, payload); `tests[<title>\|<domain>]` |
| QualityProfile | `upgrades_allowed`, `min_cf_score`, `upgrade_until_score`, `upgrade_score_increment`, `language`; `items` (the whole ordered list, as in Profilarr's `ordered_items`); `scores[<cf_id>][<domain>]` |
| DelayProfile | `protocol` (paired with the delays it constrains), `usenet_delay`, `torrent_delay`, `bypass` (paired with `min_score`); the pairs mirror Profilarr's constrained splits (`pcd-entities.md` §Delay Profiles) |
| Naming / MediaSettings | one path per field |
| QualitySize | `rows[<quality>]` (min, max, preferred together, as in Profilarr's per-tier ops) |
| entity | `create` (the whole doc as `value`), `delete` (tombstone, name-only guard, as in Profilarr) |

**Scores are wildcard plus specific**, preserving I7 without Profilarr's row splitting:
- `scores[cf][*]` is the default for every domain;
- `scores[cf][movie]` overrides it for movies;
- editing only the movie score writes one patch on `[movie]`.

The resolution order is fixed (specific, then `*`), which removes the order-dependent bug.

#### 3.4.2 Merge and classification

For each patch `p` on entity `E`, let `u` be the new upstream value at `p.path` (`absent` if the entity or member doesn't exist):

| Condition | Class | Effective value | Patch after the pull |
|---|---|---|---|
| `u == p.base` | clean | `p.value` | unchanged |
| `u == p.value` | satisfied | `u` | deleted (event `satisfied`); the "goal achieved" rule |
| `op=create`, and upstream now has the same name | `duplicate_key` | strategy | strategy |
| target entity or member absent, `op ≠ create` | `missing_target` | strategy | strategy (a `delete` patch is simply satisfied) |
| otherwise | `guard_mismatch` | strategy | strategy |

Strategies (per database, as in Profilarr; the author's `draft` layer always behaves as **ask**):

| Strategy | `guard_mismatch` | `duplicate_key` | `missing_target` |
|---|---|---|---|
| **override** | keep `p.value`; `p.base := u` (rebased: the user has now overridden the new upstream value); event `rebased` | the local create becomes field patches against the upstream entity (adopt; the user's values win) | the entity is **detached**: its last effective doc becomes a local `create`, so the user's intent keeps a target (§3.4.4) |
| **align** | delete `p`; effective `u`; event `aligned` | delete the create; local references now point to the upstream entity (same name, same database) | delete `p` |
| **ask** | `state = pending`, `pending_upstream = u`; the effective value **stays `p.value`** until resolved; resolving with *keep* runs override, *take* runs align | pending; the local doc stays effective | pending; the entity stays as it was (detached) until resolved |

Ordered `items`: `u` is compared as a whole list. Equal to `base` means clean; equal to `value` means satisfied; anything else is a conflict. This matches Profilarr's full-list check without its partial-apply problem.

#### 3.4.3 Why this is equivalent to guarded SQL

Profilarr's writer only ever produces:
- a per-field `UPDATE … SET f = v WHERE key = k AND f = old`;
- `DELETE … WHERE key = k`;
- `INSERT` of a new entity or member;
- tag delete-and-reinsert;
- batched list rewrites checked against `ordered_items`.

These are `pcd.md` §Value Guards and §Op Splitting, and `pcd-entities.md` §Summary.

Each maps one-to-one onto a patch:
- `old` becomes `base`, `v` becomes `value`;
- `rowcount = 0` becomes `u ≠ base`;
- a UNIQUE violation becomes `duplicate_key`;
- an FK violation becomes `missing_target`;
- `defaultFieldGuardRule` becomes `u == value`.

The patch form keeps everything the SQL guard expressed and adds what the SQL lacked: stable ids, commuting tag sets, and rebasing without regenerating SQL.

#### 3.4.4 Deletions, in-use entities and detaching

**Upstream removes an entity:**
- If no local or draft patch touches it, and no version slot or local entity references it, it is **tombstoned**: `upstream_present = false`, and the effective rows are deleted.
- If a slot or local entity references it, or a patch targets it, it is **detached**:
  - `origin = detached`;
  - the last effective doc is written as a local `create` patch;
  - a health notice is raised: "Dictionarry removed profile *X*; kept as a local copy used by 1,204 items".

  This applies I11: an upstream change never pulls a profile out from under a library.

**Other cases:**
- **Upstream later re-adds the same name:** this is a `duplicate_key` and follows §3.4.2. Align re-attaches to upstream; override keeps the local copy.
- **Local deletes of a referenced entity** are blocked, with the referrers listed: slots using a profile, or local profiles scoring a CF. This mirrors Profilarr's `fail_on_referenced_delete` (profilarr.md §3.1), but on by default for slots. Deleting a CF that only upstream profiles score is allowed; those score rows become inert.

### 3.5 Git lifecycle

#### 3.5.1 Fetch and pin

- The implementation is `go-git` over HTTPS, with no git CLI. A token comes from settings (D16), with basic auth `x-access-token:<token>`.
- Any host works. Anonymous access is tried first; an auth error prompts for a token. This drops Profilarr's GitHub-only rule and its `api.github.com` private-repository probe.
- Materialisation needs **only the tree at the target commit**, because history is irrelevant (§3.3.1). A depth-1 fetch is enough. For the "incoming changes" UI, the job also fetches commit metadata since `pinned_commit`; this is best-effort.
- **Schema dependency:** `pcd.json.dependencies` maps a repository URL to a version.
  - Profilarr treats it as an exact tag (`dependencies.ts:60`). Bonnyread accepts an exact version or a semver range, resolved to the highest matching tag.
  - Dependencies are cached per `url@tag` in the data directory and shared by every database that uses them. Every known PCD uses `Dictionarry-Hub/schema@1.1.0`.
- **Update policy:**

  | Policy | Behaviour |
  |---|---|
  | `auto` (default for Dictionarry) | Fetch every `check_interval`; apply if the head moved |
  | `notify` | Fetch; record `remote_head`; build an update preview (§3.10.3); apply on request |
  | `frozen` | Pinned to a commit or tag; manual only |

  A manual "pin to commit X" (including rollback) is the same job with an explicit target.
- **Local storage:** a bare object store under `<data>/pcd/<uuid>.git` is a **disposable cache**. The truth is `pinned_commit` plus the stored docs, so any node can rebuild the cache after failover. With depth 1, Dictionarry is about 0.4 MB compressed (the `ops/` directory gzips to 358 KB).

#### 3.5.2 Link and unlink

- **Link:** validate the URL and branch → fetch → manifest check → a normal pull from nothing (every aggregate is "added"). The per-database strategy and policy are chosen in the dialog; the defaults are `override` and `auto`.
- **Unlink:**
  - if no entity of the database is referenced by a slot, delete everything;
  - otherwise offer **"move in-use profiles to Local"**: each referenced profile is cloned with its CF and regex closure into the Local database (renamed on collision), and its slots are repointed. Unlink is blocked until this is done. Never orphan a slot.

#### 3.5.3 Authoring and publishing

1. **Drafts are patches in the `draft` layer.** The Changes view lists them grouped by `group_id`. Selecting a draft pulls in its dependency closure, computed from id references: a score on a CF created in a draft needs the CF's `create`. This is Profilarr's closure (`exporter.ts:370-404`), derived from the data instead of `dependsOn` metadata.
2. The **publish job** (leader) fetches the remote head. If it is ahead of `pinned_commit`, the job pulls first. Draft conflicts are always *ask*, so publishing waits until they are resolved.
3. **Emit SQL.** Diff `upstream_doc` against `upstream_doc ⊕ selected drafts`, per aggregate, and emit Profilarr-style statements with value guards. Names are resolved from ids at emit time; renames are one `UPDATE … SET name`, and PCD's `ON UPDATE CASCADE` handles the referrers. The file is `ops/<max+1>.<slug>.sql`, with the same `-- @operation: export` header and `BEGIN op`/`END op` blocks as `exporter.ts:109-118,211-216`.
4. **Round-trip validation** (stronger than Profilarr's constraint-only dry run): replay schema, then the remote ops, then the new file, in scratch SQLite. Decode and assert that the docs of the touched aggregates **equal the intended docs** and that every other doc is unchanged. A mismatch aborts the publish and shows the diff.
5. Commit with the database's git identity, then push `HEAD:refs/heads/<branch>`. A non-fast-forward rejection goes back to step 2, once.
6. Pull the new head at once. The drafts are now `u == value`, so they are **satisfied** and removed. Their events keep the `published` commit.

**The Local database** is a normal database with an empty upstream. All its patches are `draft`. "Publish to repository…" needs an empty remote: it writes `pcd.json` (with a schema dependency on the version bonnyread targets), `ops/0.initial.sql` and `README.md`, pushes, and switches `kind` to `git`.

#### 3.5.4 Multiple writers

Only the leader runs git jobs (§3.9). Concurrent draft edits by several users on the same path are ordinary writes: last write wins, with the `updated_at` and `author_id` visible. Publishing is serialised per database by the job queue's dedupe key `pcd.publish:<db>`.

#### 3.5.5 First boot

- Link Dictionarry (branch `v2`, `auto`, `override`), as Profilarr's first-boot hook does (I6).
- **Offline or failed:** fall back to a **bundled snapshot** (`kind = bundled`), a pinned Dictionarry commit embedded with `go:embed`. It is MIT-licensed; ship its licence notice. It converts to `git` in place (the same ids) on the first successful fetch, because the diff is keyed by names.
- Link the project's **starter database** for kinds Dictionarry doesn't cover (§3.8.3), once phase 2 or 3 kinds are enabled.

### 3.6 Multiple databases and priority

**Every entity is namespaced by its database.** A profile's CF references, a CF's regex references and a slot's profile reference all resolve inside one database. The measurements make this non-negotiable:

| Dictionarry v2 ∩ trash-pcd | Count |
|---|---|
| CF names shared exactly | 43 (e.g. `AMZN`, `DSNP`, `AV1`, `3D`) |
| CF names equal only case-insensitively | 3 (`CRIT`/`CRiT`, `HULU`/`Hulu`, `iP`/`IP`) |
| Regex names shared | 263 |
| … of which the patterns differ | **247** (e.g. regex `3L`: `(?<=^\|[\s.-])3L\b` vs `^(3L)$`) |
| Profile names shared | 0 |

A global name space would silently re-point one database's CFs at another's regexes.

What **priority** still does, since Profilarr's use of it (collisions when two databases sync into one Arr, `arr_sync_database_priority`) no longer exists:
1. It resolves **unqualified names** in APIs, the CLI and imports (for example `profile: "HD Bluray + WEB"`). The database with the highest priority that has the name wins; ties are an error.
2. It orders pickers and search results.
3. It picks the default profile for new slots.

**No cross-database references.** They would break the rule that every database stays exportable and self-contained (I1). Instead:
- **Clone into…** copies an entity *and its closure* (a CF with its regexes; a profile with its CFs and regexes) into a target database as `create` patches. A name collision prompts for *rename* (suffix `[trash]`), *replace* (field patches on the target's same-named entity) or *skip*.
- The regex compile cache is content-addressed (§3.11), so identical patterns across databases compile once anyway.

### 3.7 Canonical enums, and the codec at the PCD boundary

#### 3.7.1 Ownership

Per D9, bonnyread owns these as Go enums in `internal/media/quality`, `internal/media/language` and so on (the quality axes follow comparison.md §2.3):
- `Quality` (per-kind axes: video source × resolution × modifier; later audio codec/bitrate; ebook and audiobook formats; comic formats);
- `Language` (ISO-639 based, plus the specials `Any`, `Original`, `Unknown`);
- `Source`, `Resolution`, `QualityModifier`, `ReleaseType`, `IndexerFlag`.

Each value has a `Kinds` set. This replaces the only real information `quality_api_mappings` carried: which Arr has which quality. From the schema 1.1.0 seed, `CAM`, `TELESYNC`, `DVD-R`, `BR-DISK` and similar map to movie only, and `HDTV-480p` maps to neither.

There are **no enum tables in the database** and no `quality_api_mappings`.

#### 3.7.2 Codec (`internal/profiles/pcdcodec`)

The codec is a set of bijective tables between PCD strings and canonical values, restricted to the vocabulary PCD covers:
- 31 qualities (`Remux-1080p` ↔ `quality.Video{Bluray, R1080, Remux}`, and so on);
- 63 languages;
- 12 sources, 7 resolutions, 6 modifiers, 3 release types and 10 indexer flags (`profilarr/src/lib/shared/pcd/conditions.ts`);
- Radarr's string `colon_replacement_format` and Sonarr's integer `colon_replacement_format` and `multi_episode_style` (the conversions in `profilarr/src/lib/shared/pcd/mediaManagement.ts`);
- the `radarr_*`/`sonarr_*` media-management tables ↔ presets with `kind = movie|series`.

It is used in exactly two places: decode after replay, and encode at publish. It must round-trip: `encode(decode(x)) == x` for every row in Dictionarry and trash-pcd, checked by a CI test. `quality_api_mappings` rows are read only to cross-check `Kinds`, and are never written.

#### 3.7.3 Schema versions

- Bonnyread declares the schema range it supports (`^1.1`).
- **A newer minor** (additive): unknown columns and tables are ignored with a notice. New seed rows (for example a new quality) decode to `unknown`. Entities using them get a `problem`; conditions using them never match, and profile items using them are inert.
- **A new major** blocks the update (`update_blocked`) until bonnyread supports it. The previous pin stays in effect.
- `profilarr.minimum_version` is informational. Bonnyread states "PCD compatibility level 2", and a manifest requiring Profilarr major 3 or later is treated like an unsupported schema major. See Open question Q5.

### 3.8 Per-kind applicability and version slots

#### 3.8.1 Domains

A **score domain** equals a media kind: `movie`, `series`, `music`, `book`, `audiobook`, `adult`, `comic`. The PCD `arr_type` column maps to it:

| PCD `arr_type` | Domain | Notes |
|---|---|---|
| `all` | `*` | Wildcard. Applies to every domain the profile applies to |
| `radarr` | `movie` | |
| `sonarr` | `series` | Anime is a series |
| `whisparr` | `adult` | Accepted by Profilarr's manifest validator (`pcd/manifest/manifest.ts:100-105`) |
| `lidarr` | `music` | Same |
| `readarr` | `book` | Same. Audiobook is Open question Q4 |
| *(none today)* | `comic` | Needs a new value, e.g. `kapowarr`: see Issue 2 |

The `arr_type` columns are plain `VARCHAR` with **no CHECK constraint** (`0.schema.sql`: `custom_format_conditions.arr_type`, `quality_profile_custom_formats.arr_type`). So new values are valid PCD, and Profilarr ignores them: its sync filters `arr_type IN (<type>,'all')`.

**Fallback chains**, fixed in code, choose the score and condition rows when a domain has no specific row:
- `adult → [adult, movie, *]` (Whisparr v3 is a Radarr fork, D10, so movie scoring is the useful default);
- `audiobook → [audiobook, book, *]`;
- every other domain → `[domain, *]`.

The evaluation engine gets the **already resolved** score table for a domain (§3.10.1), so it never interprets `arr_type`.

Per-condition applicability (`quality_modifier` and `edition` are movie-only, `release_type` is series-only, per `conditions.ts`) becomes part of the condition-type registry. A condition can't be set to a domain where its type is meaningless, and a condition row for a domain that doesn't apply is ignored.

#### 3.8.2 Attaching profiles to version slots (D4)

- A slot definition (owned by the library design) has `quality_profile_id → PcdEntity(kind = quality_profile)`. The slot's media kind gives the domain.
- **Compatibility check** when binding: `profile.kinds ∋ slot.kind`. `kinds` is derived from the profile's `items`: the kinds for which at least one enabled item is a quality of that kind's axis. A Dictionarry 1080p profile binds to movie, series and adult slots, not to music.
- **Renames never break a binding**, because it points at the id (§3.3.2). **Upstream deletions never break it**, because the profile is detached (§3.4.4).
- Any effective change to a profile, a CF it scores, or a regex those CFs use bumps the revisions. After commit, `profiles.changed{profile_ids}` is emitted, computed from reverse edges. The library and wanted subsystems re-check cutoff and upgrade eligibility for the affected slots, lazily and batched. This replaces Profilarr's `on_pull` sync and `affectedArrs.ts`.
- There is **one profile per slot**. Profilarr's per-Arr split existed because one profile served two apps; with per-domain scores inside one profile, a "4K" slot and a "1080p" slot simply use two profiles.

#### 3.8.3 Kinds Dictionarry doesn't cover

Dictionarry and trash-pcd are video-only (`arr_types: ["radarr","sonarr"]`; qualities seeded by schema 1.1.0 are video only).

For music, books, audiobooks and comics, the PCD schema lacks:
- the qualities (FLAC/MP3 bitrates, EPUB/AZW3/M4B, CBZ/CBR);
- the condition types that make sense there (audio format/bitrate, ebook format, comic scan type);
- test-entity types (`test_entities.type CHECK (type IN ('movie','series'))`).

Plan, pending Issue 2:
- the bonnyread canonical model and ent tables are **kind-general from day one** (D2);
- the codec gains mappings as the PCD schema gains vocabulary;
- a project-maintained **starter database** (a normal PCD git repo) carries the profiles and CFs for those kinds;
- adult CFs are seeded from the Whisparr-Eros patterns per D1.

Until the schema supports them, those entities can exist **only in the Local database** and are flagged "not publishable: needs schema ≥ x".

### 3.9 HA placement (D14)

| Work | Where | Mechanism |
|---|---|---|
| Fetch, pull and apply; link; publish (push); update preview | **Leader only** | Durable job queue, dedupe key `pcd.sync:<db>` / `pcd.publish:<db>`, under the singleton lease; schedule "check every N minutes" per database |
| User edits (local or draft patches), conflict resolution, clone, reset | **Any node** | One DB transaction with `SELECT … FOR UPDATE` on the `PcdDatabase` row. The pull's apply phase takes the same lock, so an edit and a pull serialise per database; prepare runs unlocked |
| Reads, simulation, tests | Any node | Effective tables plus per-node compiled caches |
| Cache invalidation | All nodes | `NOTIFY pcd.changed {db, entity_ids, revision}` (Postgres); in-process bus on SQLite |

- **No shared filesystem is needed for git.** The object store is a disposable leader-local cache (§3.5.1). A new leader re-fetches depth 1 of `pinned_commit` in about a second.
- A leader that dies mid-apply rolls back its transaction; the job is re-claimed and runs again. It is idempotent because it diffs against the stored docs.
- Scratch SQLite is in-memory for each job. Its peak memory for Dictionarry is estimated at tens of MB (not measured), released afterwards.

### 3.10 Engine API

#### 3.10.1 Internal Go interface (consumed by release evaluation and the library)

```go
package profiles

type Store interface {
    // Resolved, immutable, cacheable views (keyed by entity id + revision).
    Profile(ctx context.Context, id EntityID) (*Profile, error)
    ProfileView(ctx context.Context, id EntityID, domain media.Kind) (*ProfileView, error)
    // ProfileView = ordered quality items (canonical), cutoff, upgrade/min scores,
    // language, and []ScoredFormat{CF *CompiledFormat; Score int} already resolved
    // through the domain fallback chain (§3.8.1). CompiledFormat carries conditions
    // filtered to the domain, with regex conditions holding *regexp2.Regexp from §3.11.
    CustomFormat(ctx context.Context, id EntityID, domain media.Kind) (*CompiledFormat, error)
    CompatibleProfiles(ctx context.Context, kind media.Kind) ([]ProfileRef, error)
    Referrers(ctx context.Context, id EntityID) (Referrers, error) // slots, profiles, CFs
    Subscribe(fn func(Changed)) (cancel func())                   // profiles.changed
}
```

The CF match semantics (AND across condition types; required conditions ANDed and the rest ORed within a type; per-condition negate) are implemented **once** by release evaluation. The testing features below call that same function.

#### 3.10.2 REST (`/api/v1`, OpenAPI, D12). Permissions (D17): `profiles:read`, `profiles:write` (local layer), `profiles:author` (drafts, publish), `profiles:admin` (link, unlink, credentials, strategy)

- `GET|POST /profile-databases`; `GET|PATCH|DELETE /profile-databases/{db}`
- `POST /profile-databases/{db}/check` · `GET /profile-databases/{db}/update-preview?commit=` · `POST /profile-databases/{db}/update {commit?}` · `GET /profile-databases/{db}/pulls` · `GET /profile-databases/{db}/events`
- `GET /profile-databases/{db}/conflicts` · `POST /profile-databases/{db}/conflicts/{patch}:resolve {keep|take}` (also in bulk)
- `GET /profile-databases/{db}/drafts` · `POST /profile-databases/{db}/publish {patch_ids, message}` · `DELETE /profile-databases/{db}/drafts/{patch}`
- For each aggregate (`regexes`, `custom-formats`, `quality-profiles`, `delay-profiles`, `naming-presets`, `media-settings`, `quality-sizes`, `test-entities`): `GET list|item`, `POST`, `PUT|PATCH ?layer=local|draft`, `DELETE`, `POST {id}:reset {paths?}`, `POST {id}:clone {target_db, name?}`.
  - `GET` returns the effective doc plus **provenance per path**: upstream value, local or draft value, and state (`clean|overridden|pending|detached`). The UI shows "upstream: -1000 · yours: 0".

#### 3.10.3 Testing features (in-process parser, D8)

- **CF tests.** `POST /profiles/test/custom-format {cf_id | cf_doc, domain, titles?}`.
  - With `cf_doc`, the editor tests **unsaved** edits.
  - Titles default to the CF's `tests[]`.
  - Each title is parsed by the parser for the domain's kind. The response gives a per-condition result (matched, negate, required, the regex span for pattern conditions) and pass/fail against `should_match`. This replaces the three-call flow `/parse` → `/match` → `evaluateCustomFormat` and the parser sidecar.
- **Regex test.** `POST /profiles/test/regex {pattern, samples[]}` returns matches and groups through the same normaliser and compile path as production (§3.11), plus `compile_error` and timeouts.
  - `regex101_id` stays a link. Importing a regex101 test list is optional and cached.
- **Profile simulation.** `POST /profiles/test/profile {profile_id | profile_doc, domain, releases}`, where `releases` is one of:
  - `test_entity:<id>` (the PCD test data);
  - `item:<library item id>[/slot]`: the item's **live interactive-search results**, recent RSS or IRC candidates, and grab history, all local now instead of Profilarr's import from an Arr (`utils/arr/releaseImport.ts`);
  - a list of titles, sizes, flags and indexers.

  The response is ranked releases with a score breakdown per CF, quality rank, cutoff position and rejection reasons, from the release-evaluation engine's pure `Evaluate` function. A test entity with a TMDb id links to the library item when one exists.
- **Compare.** `POST /profiles/test/compare {a, b, releases}` ranks the same releases with two profiles side by side and shows rank moves. The main uses:
  - *upstream vs mine* (`a = upstream_doc`, `b = effective`);
  - *before vs after an update*.
- **Update preview.** `GET /profile-databases/{db}/update-preview?commit=` runs prepare (§3.3.1) and the merge in a dry-run transaction that is rolled back. It returns:
  - the changed aggregates (with field diffs and renames);
  - the conflicts each strategy would produce;
  - for each profile bound to slots, **impact** on the library's current files: how many slots change cutoff-met status or become upgrade candidates. This is computed by re-evaluating the files' stored parse results, without re-parsing. This is what `notify` mode shows before "Apply".

#### 3.10.4 Metrics and alerts (D7)

- `pcd_pull_total{db,status}`, `pcd_pull_duration_seconds`, `pcd_conflicts_pending{db}`, `pcd_detached_entities{db}`, `pcd_update_available{db}`;
- `regex_compile_errors`, `regex_match_timeouts_total{regex}`, `regex_cache_entries`.

Alerts: a blocked update, a publish failure, and a new pending conflict.

### 3.11 Regex path (D1)

1. **Normalise** (a pure function; a golden test runs every pattern in every linked database):
   - Rewrite .NET named blocks `\p{IsX}` and `\P{IsX}` to code-point ranges, from a generated table of the .NET block names. Inside a character class, splice the ranges; outside one, wrap them in `[…]` or `[^…]`.
   - Today 0 of 619 Dictionarry and 0 of 989 trash-pcd patterns use them; Cardigann does (D1b). The same normaliser package serves the indexer layer.
   - Nothing else is rewritten, because regexp2 compiled every TRaSH and Dictionarry pattern (verification/decision-premises.md §D1).
2. **Compile:** `regexp2.MustCompile(norm, regexp2.IgnoreCase)`, default (.NET) mode. Not `RE2`, not `ECMAScript`. .NET semantics are Unicode-aware for `\d`, `\w` and `\b`, matching the Arrs. `MatchTimeout = 100ms` (it fires at about 130 ms, per D1).
3. **Cache:** node-local and **content-addressed** by `sha256(norm + options)`. It is shared across databases and CFs, so the 16 identical patterns shared by Dictionarry and trash-pcd compile once. It is bounded by an LRU of 8k entries; the working set is about 1.6k. Entries are immutable; an edit creates a new key, so invalidation is free. `CompiledFormat` values are cached by `(cf id, revision, domain)` and invalidated by `pcd.changed`.
4. **Validation:**
   - On pull, every regex is compiled. A failure is written to `Regex.compile_error` and the entity's `problems`. The condition then never matches, and a health notice is raised. The pull itself does not fail.
   - On user write, a failure rejects the write.
   - Local patterns are limited to 4 KB.
5. **At match time:** a timeout counts as *no match*, increments `regex_match_timeouts_total{regex}`, and raises a rate-limited health notice naming the regex. That makes ReDoS visible instead of silently slow.

---

## 4. Walkthroughs

Setup for all three:
- Dictionarry is linked;
- profile **"1080p Quality"** (id P) is bound to the "Default" slot of 900 movies and 300 series;
- CF **"x265 (HD)"** (id C) is in P with the upstream score row `(P, C, *) = -1000` at commit `c1`.

### 4.1 The user edits a CF score, then upstream changes the same score

**t0.** The user sets x265's score to 0 for all kinds in the Scoring tab:
- a patch `{entity: P, layer: local, path: scores[C][*], base: -1000, value: 0}` is written;
- the effective `ProfileScore(P,C,*) = 0`, the revision is bumped, and `profiles.changed{P}` is emitted;
- the library re-checks 1,200 slots' cutoff state lazily.

**t1.** Upstream commit `c2` changes the row to -2000 (`update quality_profile_custom_formats set score=-2000 where … and score=-1000`). Pull: replay → decode → P's doc hash changed → merge P: `u = -2000`, `base = -1000`, `value = 0`, so `guard_mismatch`.

| Strategy | Result after the pull | What the user sees |
|---|---|---|
| **override** | Effective stays **0**. Patch `base := -2000`. Event `rebased` (from -1000 to -2000; yours 0) | Pull summary: "1 of your overrides kept over an upstream change". Provenance: "upstream: -2000 · yours: 0". Nothing reaches the library: P's effective value didn't change, so no `profiles.changed` |
| **align** | Patch deleted. Effective **-2000**. Event `aligned` | "1 local change replaced by upstream". `profiles.changed{P}` |
| **ask** | Patch `pending`, `pending_upstream = -2000`. Effective stays **0** until resolved | The conflicts page shows the three values (was -1000, upstream -2000, yours 0), with **Keep mine**, which makes it override (`base := -2000`), and **Take upstream**, which makes it align. `pcd_conflicts_pending` is incremented and an alert is raised |

Variants:
- **Upstream moves to 0:** `u == value` means satisfied. The patch is deleted under every strategy, with no conflict.
- **Upstream changes another field of P** (e.g. `upgrade_until_score`): the score path is clean and there is no conflict (I5).
- **The user had edited only movies** (`scores[C][movie] = 0`, base absent): upstream's change to `[*]` doesn't touch that path. Movies stay 0 and series follow upstream to -2000, with no conflict.
  - Profilarr would record a conflict here. Its edit split the `all` row into two per-type rows guarded on -1000 (`pcd-entities.md` §Scoring). Base ops run first and change the row to -2000, so the user's `DELETE … WHERE arr_type='all' AND score=-1000` guard fails.

### 4.2 Upstream renames a CF referenced by a user override

State: the user has two local patches, `scores[C][*] = 0` on P (as above) and `description` on C. Commit `c3` adds `ops/412.rename-x265.sql`: `update "custom_formats" set "name"='x265 (1080p)' where "name"='x265 (HD)'`.

1. **Replay.** The rename trigger journals `(custom_formats, 'x265 (HD)' → 'x265 (1080p)', 412.rename-x265.sql)`. FK `ON UPDATE CASCADE` rewrites the scratch rows in `quality_profile_custom_formats`, `custom_format_conditions` and `condition_*`.
2. **Diff.** `412…` is not in `applied_ops`, so this is a rename of this pull. `PcdEntity(C).name := 'x265 (1080p)'`; the id is unchanged. C's doc changed only in its name. P's doc references C **by id**, so P's hash is **unchanged** and P isn't touched at all.
3. **Merge C.** The `description` patch path is clean, because upstream didn't change the description. The score patch on P was not re-examined, since P didn't change.
4. **Result:** no conflict under any strategy. Both overrides survive, the slots are unaffected, and the only event is `renamed`.
   - Draft SQL emitted later references the new name.
   - If the user had renamed C locally too, that is a `name` path patch: `base = 'x265 (HD)'`, `u = 'x265 (1080p)'`, so `guard_mismatch`, and the strategy decides which name wins.

**Contrast with Profilarr.** The user op `UPDATE quality_profile_custom_formats SET score=0 WHERE custom_format_name='x265 (HD)' AND … AND score=-1000` affects 0 rows, so it is a conflict. The override handler then has to reconstruct the rename with a regex over the SQL text of base ops (`overrideUtils.ts:85-160`, `qualityProfiles/override/scoring.ts:151`). That works only if the op is spelled exactly `update "custom_formats" set "name" = '…' where "name" = '…'`. The regex is case-insensitive but requires the quoted identifiers and that clause order; Dictionarry's initial op uses unquoted identifiers.

**Harder variant:** upstream deletes "x265 (HD)" and creates "x265 (1080p)" with *changed* conditions in the same pull. The journal is empty, and content equality fails.
- Under **override**, C is **detached**: the local copy keeps the old conditions, the score survives on it, and a notice offers "Retarget my overrides to *x265 (1080p)*?". That action moves the score and description patches to the new id with `base` taken from the new upstream values.
- Under **align**, the patches on C are dropped; P now scores the new CF at upstream's value.
- Under **ask**, both options are pending.

### 4.3 Linking a second database with overlapping names

The user links trash-pcd (§3.5.2).

1. **Pull from nothing.** It creates 201 CFs, 989 regexes, 22 profiles and so on in the **trash-pcd namespace**.
   - Dictionarry's `AMZN` CF and trash-pcd's `AMZN` CF are two entities with different ids.
   - Regex `3L` in Dictionarry (`(?<=^|[\s.-])3L\b`) and in trash-pcd (`^(3L)$`) stay distinct. Each CF resolves its regex inside its own database.
   - `CRiT` versus `CRIT` differ in case across databases, which is allowed. Case-insensitive uniqueness applies within a database only.
2. **Regex cache.** The 16 byte-identical shared patterns compile once.
3. **Priority** (trash-pcd = 2, Dictionarry = 1). An import or CLI call naming profile `"WEB-1080p"` resolves unqualified; only trash-pcd has it. Naming CF `"AMZN"` unqualified resolves to trash-pcd because of its higher priority, and the API returns a `resolved_from` field. Pickers show `WEB-1080p · TRaSH`.
4. **The user wants the Dictionarry profile with TRaSH's `AMZN` CF.**
   - *Clone into Dictionarry* of trash-pcd `AMZN` finds that Dictionarry already has `AMZN`, and prompts:
     - **rename**: creates `AMZN [trash]` plus its regex closure, with names suffixed on collision;
     - **replace**: writes field patches on Dictionarry's `AMZN`, conditions and regex references included, so the user's Dictionarry `AMZN` now behaves like TRaSH's. Future Dictionarry changes to `AMZN` conflict per the strategy.
   - All results are local patches in Dictionarry's namespace. No cross-database reference exists, and Dictionarry's effective set stays exportable.
5. **Binding.** A series slot bound to trash-pcd's `WEB-1080p` gets the series domain. trash-pcd writes per-Arr rows, and resolution is `[series, *]`.
6. **Unlinking trash-pcd later** is blocked while that slot references `WEB-1080p`. The UI offers "move to Local" (§3.5.2).

---

## 5. Simplifications compared with Profilarr

| Area | Profilarr | Bonnyread |
|---|---|---|
| Read model | In-memory SQLite per database, rebuilt after each write, 30–50 MB each for the life of the process | Persisted ent tables; scratch SQLite only during pull or publish, then freed |
| Write cost | A full replay of every layer (`writer.ts:254,291,316`) | Re-merge of one aggregate |
| Override form | Guarded SQL, `metadata`, `desired_state`, content hash, op splitting, cancel-out | `(path, base, value)` patches; splitting and cancel-out fall out of the path model |
| Override resolution | Entity-specific handlers regenerate SQL, "up to 10 rounds" | `base := upstream` |
| Auto-align | 4 rules (`conflicts/autoAlign/rules/*`) | 1 generic rule: `u == value` |
| Full-list conflicts | Separate post-check for partly applied multi-statement ops | A list is one path value; partial application can't happen |
| Renames | Generated cascade ops; the upstream rename map comes from a regex over SQL text | Id references; a trigger journal during replay |
| Base-op storage | Copied into `pcd_ops`, with sequence offsets `2e9`/`3e9` and orphan states | Not stored; read from the tree at `pinned_commit`; a file manifest for renames |
| History | A row per op per compile | A row per state change |
| Failed base op | Disables the database | Keeps the previous pin; `update_blocked` |
| Git | `git` CLI, GitHub-only, PAT in the URL | `go-git`, any HTTPS host, token auth, depth-1 fetch |
| Dependency versions | Exact tag | Exact tag or semver range, cached per `url@tag` |
| Enums | `qualities`/`languages` tables, `quality_api_mappings`, `sync/mappings.ts`, `mapQualityName` | Go enums with `Kinds`; a codec only at the boundary |
| Per-type scores | `all` rows split on edit; resolution depends on row order | A wildcard plus specific entries, with a fixed fallback chain per domain |
| Media-management tables | Separate radarr and sonarr tables, Sonarr int enums | One preset per kind; ints only in the codec |
| Multiple databases | Priority orders syncs into a shared Arr name space | Namespaced by database; priority only resolves unqualified names and ordering |
| Testing | .NET parser sidecar, two caches, a TypeScript copy of CF matching, TMDb search, import from an Arr | The in-process parser and the single CF engine; library items and live search as test sources; compare and update-impact preview |
| Publish validation | Constraint dry-run against a clean base | Round trip: replay the emitted SQL and assert the resulting docs equal the intended ones |
| Deployment | One process; the cache is per process | HA: git work on the leader; edits on any node; `NOTIFY` invalidation |
| Dropped entirely | Sync, drift, cleanup, `on_pull`, the Arr clients, `affectedArrs`, the library cache, the Arr test-release import | – |

---

## 6. Issues with decisions, and open questions

### 6.1 Issues with decisions (flagged, not deviated from)

1. **D9b (format compatibility) versus D2 (every kind from day one).** PCD schema 1.1.0 can express only video.
   - Qualities are a seeded table of 31 video qualities.
   - The condition types have no audio, ebook or comic axes.
   - `test_entities.type` has `CHECK IN ('movie','series')`.
   - The `arr_type` columns are free text, so per-kind *scores* fit. Per-kind *qualities, conditions and test data* do not.

   Either:
   - **(a)** propose additive changes to `Dictionarry-Hub/schema`: new seed qualities, new condition tables, and a relaxed test-type CHECK (the CHECK change is a table rebuild in SQLite, so it is *breaking* under `schema-bump.md`); or
   - **(b)** bonnyread publishes an extension schema dependency. Databases using it become **not Profilarr-compatible**, because Profilarr applies only `deps/schema/ops` (`pcd/ops/loadOps.ts:50-52`).

   D9b's promise therefore holds only for video kinds unless (a) is accepted. **Decision needed** before phase 2 (music).
2. **D9a "Dictionarry is the default database" leaves phase 2 and 3 kinds without a default.** No community PCD covers music, books, comics or adult. D1 names Whisparr-Eros as the source of adult patterns but assigns no owner for the data. A project-maintained starter PCD (§3.8.3) is proposed; its ownership, repository and licence need a decision. Comics also need a new `arr_type` value, which Profilarr's manifest validator rejects (`manifest.ts:100-105`). A database listing it in `arr_types` fails in Profilarr.
3. **D9a keeps "ask" but doesn't define what is in effect while a conflict is pending.** This design keeps **the user's value** in effect until resolution (§3.4.2), so nothing changes in the library without consent. Profilarr's effective behaviour is the opposite: the pending op isn't applied, and its `on_pull` sync pushes the upstream value to the Arr. Please confirm.
4. **D13 plus scratch replay:** every deployment, including Postgres and MySQL, embeds a pure-Go SQLite (`modernc.org/sqlite`) as the PCD decoder. This doesn't conflict with D13, but it is a required runtime component that D13 doesn't list.
5. **D14 wording:** it lists "profile-database git pulls" as leader singletons. This design also puts **publish (push)**, **link** and **update preview** on the leader, and keeps **edits** on every node. It needs **no shared storage** for git; the object store is a disposable cache. That is compatible, but narrower than D14's "data paths must be shared".
6. **D1 scope:** D1b attaches the `\p{IsBlock}` rewrite to Cardigann only. This design runs the same normaliser over CF patterns too; no current dataset needs it. It is harmless, but it makes the normaliser a shared package owned outside the indexer layer.

### 6.2 Open questions

- **Q1. Upstream deletion of a profile in use:** is automatic **detach to a local copy** (§3.4.4) the desired policy, or should the update block until the user re-points the slots? D4 is silent.
- **Q2. Publishing workflow:** Profilarr pushes directly to the branch (`exporter.ts:507`). Should bonnyread also offer "push to a new branch and open a PR" for shared databases? That would need a host API (GitHub, GitLab, Gitea) beyond plain git.
- **Q3. Pushing from a depth-1 clone with go-git** needs a spike. Fallbacks: fetch deeper on publish only, or bundle the git CLI for publishing.
- **Q4. The book domains:** a single `readarr` value versus separate ebook and audiobook values. D4's ebook and audiobook slots imply separate scores, but PCD has one `readarr` value.
- **Q5. `profilarr.minimum_version`:** is mapping it to a "PCD compatibility level" acceptable? The alternative is to ignore it.
- **Q6. Migrating Profilarr users:** Profilarr's user ops carry structured `desired_state` JSON. A one-shot importer into local patches is cheap. D7 rules out *compatibility layers*, not *importers*; should it be in phase 1?
- **Q7. Bundled Dictionarry snapshot** (§3.5.5): embedding MIT data for offline first boot is licence-compatible (keep the notice), but it adds a release chore (refresh the snapshot each release). Is it wanted?
- **Q8. Recyclarr users:** D9 dropped direct TRaSH import. A `recyclarr.yml` translator to *trash-pcd plus local patches* (recyclarr.md §11.2 item 4) could ease migration. Is it in scope?
- **Q9. Condition renames inside a CF** are handled as a delete plus an add of that member (§3.3.2 journals only aggregate names). Is that granularity enough, or should condition-level renames be journaled too?
