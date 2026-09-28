# 07. Profiles

This file specifies bonnyread's native profile store, built on the Profilarr Compliant Database (PCD) format (D9). It covers linked git databases and pinned commits; decoding upstream content into bonnyread's own tables; user overrides as structured patches, and what happens to them when upstream changes; conflict strategies; renames; multiple databases; authoring and publishing; the extension schema for media kinds and features the PCD schema can't express (D20); starter databases; the canonical quality, language and flag enums; per-kind scores; how version slots reference profiles; testing and simulation; the regex path; and where this work runs in HA. How a profile is **applied** to a release (custom-format matching, scoring, ranking, cutoff, delay holds) belongs to `04-evaluation.md`. This file defines only the read interface that evaluation consumes (§3.16).

Phases: **P1** for the store, video kinds (movie, tv), Dictionarry, testing and publishing. **P2** adds the music domain, the extension schema content for music, and the music starter database. **P3** adds book, audiobook, adult and comic domains and their starter databases. Autograb rules are stored and published from the phase in which D16's racing mode ships; their schema is defined here from P1 so the format is stable.

---

## 1. Concepts

| Term | Definition |
|---|---|
| **PCD** | A Profilarr Compliant Database: a git repository with a `pcd.json` manifest, append-only `ops/*.sql` files, optional `tweaks/*.sql`, and a semver'd schema dependency. |
| **Profile database** (database) | One linked PCD (`kind = git`), the built-in **Local** database (`kind = local`), or the bundled bootstrap snapshot (`kind = bundled`). Every profile entity belongs to exactly one database. |
| **Schema dependency** | A repository listed in `pcd.json.dependencies`, applied before the database's own ops. The upstream schema is `Dictionarry-Hub/schema`. The **extension schema** (§3.11) is a second, optional dependency. |
| **Pinned commit** | The commit of a git database whose content is currently in effect. |
| **Replay** | Executing schema ops, then base ops, then tweaks, in order, in a scratch embedded SQLite database, to decode a PCD. Replay is a format decoder only. |
| **Aggregate** | The unit of diff, patch and cache invalidation. The aggregates are `Regex`, `CustomFormat` (conditions and tests inside), `QualityProfile` (items, languages, scores inside), `DelayProfile`, `NamingPreset`, `MediaSettingsPreset`, `QualitySizePreset`, `TestEntity` (releases inside) and `AutograbRule`. |
| **Upstream document** (upstream doc) | The canonical JSON form of one aggregate at the pinned commit, with references held as entity ids. |
| **Patch** | One intent on one path of one aggregate: `(path, base, value)`. `base` is the upstream value the intent was written against; `value` is the intended value. |
| **Layer** | `local` (private user overrides; Profilarr's "user layer") or `draft` (publishable author changes; Profilarr's "base drafts"). |
| **Effective state** | Upstream ⊕ draft ⊕ local, materialised in ordinary tables. The rest of bonnyread reads only this. |
| **Conflict strategy** | Per database: `override` (the user's value wins), `align` (upstream wins) or `ask` (the user decides; the user's value stays in effect meanwhile, D20). |
| **Detached entity** | An entity that upstream removed but that is still in use, kept as a local copy. |
| **Score domain** | The media-kind scope of a score row or condition: `movie`, `tv`, `music`, `book`, `audiobook`, `adult`, `comic`, or the wildcard `*`. `design/profiles.md` calls the TV domain `series`; this file uses `tv` to match the media kind in `library.md`. |
| **Codec** | The translation layer between PCD strings (quality, language, source, flag names; per-Arr integer enums in media-management tables) and bonnyread's canonical enums. |
| **Generation** | A global counter bumped in the same transaction as any change to effective state. Evaluation compiles per generation. |
| **SlotDefinition, TitleSlot** | As in `02-domain-model.md` (`library.md` calls SlotDefinition `Slot`). A TitleSlot references one QualityProfile. |

---

## 2. Scope summary

What is preserved from Profilarr: the PCD on-disk format, name-keyed entities, layering, the three conflict strategies and their conflict classes, field-level granularity, UI authoring and publishing to git, CF tests and profile simulation. What is dropped: everything that exists only to push configuration into external Arr instances (sync, drift, cleanup, `on_pull`, per-Arr enum mapping, `quality_api_mappings`), the in-memory recompile after every write, SQL-string overrides, and the .NET parser sidecar (D9a, D19).

---

## 3. Requirements

### 3.1 Format compatibility

**PRF-001** (MUST) bonnyread **MUST** read any PCD that declares a supported schema dependency, including Dictionarry-Hub/database (branch `v2`), Dictionarry-Hub/trash-pcd and Dumpstarr, without modification. _Source: D9b; design/profiles.md I1._

**PRF-002** (MUST) Ops **MUST** be applied in Profilarr order: schema dependency ops, then the database's `ops/*.sql` by ascending numeric prefix, then unprefixed op files in lexical order, then `tweaks/*.sql`. _Source: design/profiles.md I1, I3._

**PRF-003** (MUST) Any op file bonnyread publishes to a database whose manifest declares only the upstream schema **MUST** replay successfully in Profilarr (it **MUST** use only tables and values defined by the declared upstream schema version). _Source: D9b; design/profiles.md I1._

**PRF-004** (MUST) bonnyread **MUST** declare the upstream schema range it supports (initially `^1.1`). A dependency may be an exact version or a semver range; a range resolves to the highest matching tag. _Source: design/profiles.md §3.5.1._

**PRF-005** (MUST) A newer **minor** schema version **MUST** be accepted: unknown tables and columns are ignored with a health notice; unknown seed values (for example a new quality) decode to `{"unknown": "<raw>"}` and are recorded as entity problems (§3.4). A new **major** version **MUST** block the update (`health = update_blocked`) and keep the previous pin in effect. _Source: design/profiles.md §3.7.3; I12._

**PRF-006** (MUST) bonnyread **MUST** state a fixed "PCD compatibility level" (initially `2`). A manifest whose `profilarr.minimum_version` requires a Profilarr major version greater than the compatibility level **MUST** be treated as an unsupported schema major (PRF-005); otherwise `minimum_version` is informational. [OPN-060] _Source: design/profiles.md Q5; open-issues.md Group C._

### 3.2 Databases, linking and the git lifecycle

**PRF-007** (MUST) Git access **MUST** use `go-git` over HTTPS. The `git` CLI **MUST NOT** be required. Any host **MUST** work. Anonymous access **MUST** be tried first; an authentication failure **MUST** surface a request for a token. Tokens are stored as secrets per D16 and sent as basic auth `x-access-token:<token>`. _Source: design/profiles.md §3.5.1._

**PRF-008** (MUST) Materialisation **MUST** need only the tree at the target commit. Fetches **MUST** be depth 1 of the target commit. Commit metadata since the pinned commit, for the "incoming changes" view, **MAY** be fetched best-effort and its failure **MUST NOT** fail the pull. _Source: design/profiles.md §3.5.1._

**PRF-009** (MUST) Each database **MUST** record `pinned_commit`, and the effective state **MUST** be a pure function of (target commit tree, schema dependency trees, patches). Moving to any target commit (forward, rollback, branch switch) **MUST** use the same pull pipeline (§5.1); rewritten or deleted upstream op files **MUST** need no special handling. _Source: design/profiles.md §3.3.1._

**PRF-010** (MUST) Each git database has an update policy:

| Policy | Behaviour |
|---|---|
| `auto` (default) | Check every `check_interval_min` (default 60); pull when the remote head moved |
| `notify` | Check; record `remote_head`; build an update preview (PRF-094); pull only on request |
| `frozen` | Never check automatically; pull only to an explicitly requested commit or tag |

A manual "pin to commit X" **MUST** be accepted under every policy. _Source: design/profiles.md §3.5.1._

**PRF-011** (MUST) Schema dependency trees **MUST** be cached per `url@tag` and shared by every database that uses them. _Source: design/profiles.md §3.5.1._

**PRF-012** (MUST) Linking **MUST** validate the URL and branch, fetch, check the manifest (PRF-004 to PRF-006), and run a normal pull from an empty state. The link dialog **MUST** let the admin choose the conflict strategy (default `override`), update policy (default `auto`) and priority. _Source: design/profiles.md §3.5.2._

**PRF-013** (MUST) Unlinking **MUST** delete the database's entities only when no TitleSlot, SlotDefinition default, autograb rule or other database's clone lineage references them. Otherwise unlinking **MUST** be blocked, listing the referrers, and **MUST** offer **"move in-use profiles to Local"**: each referenced profile is cloned with its CF and regex closure into the Local database (renamed on collision, PRF-055), and its referrers are repointed in one transaction. A slot **MUST** never be left without a profile. _Source: design/profiles.md §3.5.2._

**PRF-014** (MUST) On first boot bonnyread **MUST** link Dictionarry-Hub/database, branch `v2`, policy `auto`, strategy `override`, priority 1. _Source: D9a; design/profiles.md I6, §3.5.5._

**PRF-015** (SHOULD) The binary **SHOULD** embed a bundled snapshot of a pinned Dictionarry commit, with its MIT licence notice, used as a `kind = bundled` database when the first-boot link fails or the node is offline. On the first successful fetch the bundled database **MUST** convert in place to `kind = git`, keeping entity ids (the diff is keyed by names). The snapshot **MUST** be refreshed each release. [OPN-062] _Source: design/profiles.md §3.5.5, Q7; open-issues.md Group C._

**PRF-016** (MUST) The built-in **Local** database **MUST** always exist. It has an empty upstream; all its patches are in the `draft` layer. It **MUST** be publishable to an empty remote repository (PRF-063). _Source: design/profiles.md §3.5.3._

**PRF-017** (MUST) Once the P2 or P3 media kinds are enabled, bonnyread **MUST** offer to link the project starter database for each enabled kind that no linked database covers (§3.12). _Source: D20; design/profiles.md §3.5.5._

### 3.3 Storage of fetched content (A2)

**PRF-018** (MUST) The file tree at each database's pinned commit (`pcd.json`, `ops/`, `tweaks/`, `README*`) and each cached schema dependency tree **MUST** be stored in the deployment database as content-addressed blobs (`sha256`), with a per-commit file manifest. Nodes **MUST** treat any on-disk git object store as a disposable cache that can be rebuilt from the remote or, for replay, from the stored blobs. No shared filesystem is required for profile data. _Source: open-issues.md A2; D20 (D14 amended)._

**PRF-019** (MUST) Base ops **MUST NOT** be copied into per-op rows with sequence offsets or orphan states (Profilarr's `pcd_ops`). Only the manifest of `{path, sha256}` per applied file (`applied_ops`) is retained per database, for rename detection (§3.5). _Source: design/profiles.md §2.3._

### 3.4 Decoding and materialisation

**PRF-020** (MUST) Replay **MUST** run in an embedded pure-Go SQLite (`modernc.org/sqlite`) present in every build, independent of the deployment database, in memory, with foreign keys **on**, inside one transaction, with the helper SQL functions `qp()`, `cf()`, `dp()` and `tag()` registered with Profilarr's semantics. Replay **MUST** run only on pull, publish validation and update preview, never on a user write. _Source: open-issues.md A18; design/profiles.md §3.3.1._

**PRF-021** (MUST) After replay, the codec (§3.13) **MUST** decode the scratch tables into one canonical upstream doc per aggregate. Docs **MUST** hold references to other aggregates as entity ids, never as names. Doc serialisation **MUST** be canonical (sorted keys, stable member ordering by key) so that equal content hashes equal. _Source: design/profiles.md §3.3.1._

**PRF-022** (MUST) Each aggregate's identity **MUST** be a `PcdEntity` row keyed by `(database_id, kind, name)`, whose id is stable across renames and is what every reference in bonnyread points at. _Source: design/profiles.md §3.2._

**PRF-023** (MUST) The pull **MUST** diff decoded docs against stored `upstream_hash` values to classify each aggregate as added, changed, removed, renamed or unchanged, and **MUST** re-merge (§3.7) and rewrite effective rows **only** for aggregates that are added, changed, removed or renamed, or whose patches' classification changed. _Source: design/profiles.md §3.3.1._

**PRF-024** (MUST) A base op or tweak that fails during replay **MUST** abort the pull. The previous pin and all effective state **MUST** stay in effect, `health` **MUST** become `update_blocked`, and the failing file and SQLite error **MUST** be reported in the pull record and as a health issue. The database **MUST NOT** be disabled. _Source: design/profiles.md §3.3.1; contrast Profilarr `cache.ts:351`._

**PRF-025** (MUST) Values the codec can't map, regexes that fail to compile, and case-insensitive name collisions within one database **MUST** be recorded in the entity's `problems` without failing the pull. A condition whose value or regex is unusable **MUST** never match; a profile item with an unknown quality **MUST** be inert. _Source: design/profiles.md §3.3.1, §3.11._

**PRF-026** (MUST) The apply phase of a pull **MUST** run in one deployment-database transaction holding a row lock on the `PcdDatabase` row, and **MUST** in that transaction: apply renames, upsert upstream docs, handle removals (§3.8), re-merge touched entities, write effective rows, bump entity revisions and the global generation, set `pinned_commit` and `applied_ops`, write events, and enqueue the `profiles.changed` event (§3.16). The prepare phase (fetch, replay, decode, diff) **MUST** run without holding that lock. _Source: design/profiles.md §3.3.1, §3.9._

**PRF-027** (MUST) Re-running a pull to the same target commit **MUST** be a no-op (idempotent), so a job re-claimed after a node failure is safe. _Source: design/profiles.md §3.9._

### 3.5 Upstream rename tracking

**PRF-028** (MUST) Before replay the engine **MUST** install `AFTER UPDATE OF name` triggers on every name-keyed table (`regular_expressions`, `custom_formats`, `quality_profiles`, `delay_profiles`, the naming, media-settings and quality-definition tables, and name-keyed extension-schema tables), writing `(table, old_name, new_name, current_file)` to a journal table. `current_file` **MUST** be set before each file executes. _Source: design/profiles.md §3.3.2._

**PRF-029** (MUST) Renames journaled in files not in the database's `applied_ops` (by path and hash) are the renames of this pull. Chains **MUST** be composed (A→B→C yields A→C). A rename **MUST** update `PcdEntity.name` and keep the id. _Source: design/profiles.md §3.3.2._

**PRF-030** (MUST) If the journal records no rename, a removed key and an added key of the same kind whose docs are equal apart from `name` **MUST** be treated as a rename. Otherwise the change is a delete plus an add (§3.8). _Source: design/profiles.md §3.3.2._

**PRF-031** (MUST) Renames of members inside an aggregate (for example a condition inside a CF) **MUST** be handled as a remove plus an add of that member; condition-level rename journaling is deferred. [OPN-064] _Source: design/profiles.md Q9; open-issues.md Group C._

### 3.6 Patches (user overrides and drafts)

**PRF-032** (MUST) User overrides and drafts **MUST** be stored as structured patches `{entity, layer, op, path, base, value}`, with `op ∈ {create, set, add, remove, delete}`. SQL strings **MUST NOT** be used as the override representation. At most one patch **MUST** exist per `(entity, layer, path)`. _Source: D9 recommendation; design/profiles.md §3.4._

**PRF-033** (MUST) Paths are typed per aggregate and key collection members by stable keys, never by position:

| Aggregate | Paths |
|---|---|
| any | `name`, `description`, `tags` (`add`/`remove` of one tag) |
| Regex | `pattern`, `regex101_id` |
| CustomFormat | `include_in_rename`; `conditions[<name>]` (whole condition object); `tests[<title>\|<domain>]` |
| QualityProfile | `upgrades_allowed`, `min_cf_score`, `upgrade_until_score`, `upgrade_score_increment`, `language`, `delay_profile` (extension, §3.11); `items` (whole ordered list); `scores[<cf_id>][<domain>]` |
| DelayProfile | `protocol` (paired with the delays it constrains), `usenet_delay`, `torrent_delay`, `bypass` (paired with `min_score`), `tags` (extension) |
| NamingPreset, MediaSettingsPreset | one path per field |
| QualitySizePreset | `rows[<quality>]` (min, max, preferred together) |
| AutograbRule | one path per top-level field; `conditions[<name>]`; `limits[<key>]` |
| entity | `create` (whole doc as `value`), `delete` (tombstone) |

_Source: design/profiles.md §3.4.1._

**PRF-034** (MUST) Tag `add`/`remove` patches **MUST** commute and **MUST** never conflict. _Source: design/profiles.md §3.4.1._

**PRF-035** (MUST) A user write **MUST**: validate the candidate aggregate (the schema's CHECK rules re-expressed in Go, such as exactly one cutoff item, `upgrade_score_increment > 0`, delay protocol/delay pairing; references resolve inside the same database; regexes compile, PRF-101; case-insensitive name uniqueness within the database); compute the path-level diff against the effective state without the target layer; for each changed path update the existing patch's `value` (keeping its `base`) or create a patch with `base` = the current upstream value (`nil` if absent); delete any patch whose `value` now equals its `base`; and re-merge only that entity. A write **MUST NOT** trigger replay. _Source: design/profiles.md §3.3.3._

**PRF-036** (MUST) Writes default to the `local` layer. A write goes to the `draft` layer only when the database is `writable`, the caller holds `profiles:author`, and the caller chose "edit as draft". The merge order **MUST** be upstream → draft → local. _Source: design/profiles.md §3.3.3._

**PRF-037** (MUST) `POST /api/v1/<aggregate>/{id}/reset {paths?}` (an action sub-resource, `11-api-and-clients.md` API-012) **MUST** delete the caller-layer patches on the given paths (all paths when omitted), restoring the upstream value. _Source: design/profiles.md §3.10.2._

### 3.7 Merge, conflict classes and strategies

**PRF-038** (MUST) On each re-merge, for each patch `p` with new upstream value `u` at `p.path` (`absent` if the entity or member does not exist), the patch **MUST** be classified in this order:

| Condition | Class |
|---|---|
| `u == p.value` | `satisfied`: the patch is deleted under every strategy, event `satisfied` |
| `u == p.base` | `clean`: `p.value` is effective, the patch is unchanged |
| `op = create` and upstream now has an entity of the same kind and case-folded name | `duplicate_key` |
| target entity or member absent and `op ≠ create` | `missing_target` (a `delete` patch is `satisfied` instead) |
| otherwise | `guard_mismatch` |

_Source: design/profiles.md I4, §3.4.2._

**PRF-039** (MUST) Ordered profile `items` **MUST** be compared as one whole-list value. _Source: design/profiles.md I5, §3.4.2._

**PRF-040** (MUST) A change to one path **MUST** never conflict with an upstream change to a different path of the same entity. _Source: design/profiles.md I5._

**PRF-041** (MUST) Conflicts **MUST** be resolved per the database's strategy:

| Strategy | `guard_mismatch` | `duplicate_key` | `missing_target` |
|---|---|---|---|
| `override` | keep `p.value`; set `p.base := u`; event `rebased` | convert the local create into field patches against the upstream entity (the user's values win); repoint local references to it | detach the entity (PRF-047) so the patch keeps a target |
| `align` | delete `p`; `u` becomes effective; event `aligned` | delete the create; repoint local references to the upstream entity | delete `p` |
| `ask` | `state = pending`, `pending_upstream = u`; **`p.value` stays effective** | `state = pending`; the local doc stays effective | `state = pending`; the entity stays in effect as a detached copy |

_Source: design/profiles.md §3.4.2; D20._

**PRF-042** (MUST) While an `ask` conflict is pending, the user's value **MUST** stay in effect; no effective state that the library or evaluation reads may change for that path until the user resolves it. Resolving with **keep** **MUST** apply the `override` action and resolving with **take** **MUST** apply the `align` action, each against the upstream value current at resolution time. Resolution **MUST** be available per patch and in bulk. _Source: D20; open-issues.md B8._

**PRF-043** (MUST) Patches in the `draft` layer **MUST** always behave as `ask`, whatever the database strategy. _Source: design/profiles.md §3.4.2._

**PRF-044** (MUST) A pending conflict **MUST** raise an alert (D7), increment `bonnyread_profiles_conflicts_pending{db}`, and be listed at `GET /api/v1/profile-databases/{db}/conflicts` with the three values `base`, `pending_upstream` and `value`. If a later pull makes the upstream value equal `p.value` or `p.base`, the pending state **MUST** be re-classified (satisfied or clean) automatically. _Source: design/profiles.md §3.4.2, §3.10.4._

**PRF-045** (MUST) A score patch, conflict or strategy action **MUST NOT** emit `profiles.changed` when the effective value is unchanged (for example `override` after a `guard_mismatch`). _Source: design/profiles.md §4.1._

### 3.8 Deletions and detaching

**PRF-046** (MUST) An entity that upstream removed, that no patch touches, and that no TitleSlot, SlotDefinition, autograb rule or local entity references **MUST** be tombstoned: `upstream_present = false` and its effective rows deleted. The `PcdEntity` row **MAY** be kept so that a later re-add reuses the id. _Source: design/profiles.md §3.4.4._

**PRF-047** (MUST) An entity that upstream removed while referenced by a slot, rule or local entity, or targeted by a patch, **MUST** be **detached**: `origin = detached`, its last effective doc written as a local `create` patch, and a health notice raised naming the entity and the number of referring slots and items. The update **MUST NOT** block. [OPN-056] _Source: design/profiles.md §3.4.4, Q1; open-issues.md Group C; I11._

**PRF-048** (MUST) If upstream later re-adds the same name, it **MUST** be treated as `duplicate_key` (PRF-041): `align` re-attaches the entity to upstream, `override` keeps the local copy, `ask` pends with the local copy in effect. _Source: design/profiles.md §3.4.4._

**PRF-049** (MUST) When a removed entity was replaced upstream by a differently named, non-equal entity (the rename was not detected), bonnyread **MUST** offer **"Retarget my overrides to <new entity>"**, which moves the detached entity's patches and referrers to the new entity with `base` set to the new upstream values. _Source: design/profiles.md §4.2._

**PRF-050** (MUST) A local delete of an entity referenced by a slot, rule, or local profile score **MUST** be rejected with HTTP 409 and the list of referrers. Deleting a CF that only upstream profiles score **MUST** be allowed; those score rows become inert. _Source: design/profiles.md §3.4.4._

**PRF-051** (MUST) bonnyread **MUST NOT** delete any entity it did not create, and **MUST NOT** zero any score the user set, except by an explicit user action. _Source: design/profiles.md I11; D9 recommendation._

### 3.9 Namespacing, priority and cloning

**PRF-052** (MUST) Every entity **MUST** be namespaced by its database. A profile's CF references, a CF's regex references, and a profile's delay reference **MUST** resolve inside the same database. Cross-database references **MUST NOT** exist. _Source: design/profiles.md §3.6._

**PRF-053** (MUST) Name uniqueness (case-insensitive) applies within one database and kind only. Equal or case-variant names in different databases (for example Dictionarry `CRIT` and trash-pcd `CRiT`) **MUST** be distinct entities. _Source: design/profiles.md §3.6, §4.3._

**PRF-054** (MUST) Each database has an integer `priority`, unique across databases. Priority **MUST** be used only to: resolve **unqualified names** in the API, CLI and importers (the highest-priority database that has the name wins; the response **MUST** include `resolved_from`); order pickers and search results; and choose the default profile for new SlotDefinitions. Qualified names use the form `<database>/<name>`. _Source: design/profiles.md §3.6._

**PRF-055** (MUST) **Clone into…** **MUST** copy an entity and its closure (a CF with its regexes; a profile with its CFs, regexes and delay profile) into a target database as `create` patches in the chosen layer. On a name collision it **MUST** prompt for **rename** (suffix `[<source database short name>]`, suffixed again on further collision), **replace** (field patches on the target's same-named entity) or **skip**, per entity in the closure. _Source: design/profiles.md §3.6, §4.3._

### 3.10 Drafts and publishing

**PRF-056** (MUST) Drafts are patches in the `draft` layer. The Changes view **MUST** list them grouped by `group_id`, and selecting a draft **MUST** include its dependency closure computed from id references (for example a score on a CF created in a draft requires the CF's `create`). _Source: design/profiles.md §3.5.3._

**PRF-057** (MUST) Publishing **MUST**: fetch the remote head, and pull first if it is ahead of the pinned commit; wait while any selected draft is pending; emit one op file; validate it by round-trip replay (PRF-059); commit with the database's git author identity; push `HEAD:refs/heads/<branch>`; and pull the new head at once. _Source: design/profiles.md §3.5.3._

**PRF-058** (MUST) The emitted op file **MUST** be named `ops/<max+1>.<slug>.sql`, carry Profilarr's `-- @operation: export` header and `BEGIN op`/`END op` blocks, and contain value-guarded statements generated from `diff(upstream_doc, upstream_doc ⊕ selected drafts)` per aggregate. Names **MUST** be resolved from ids at emit time. A rename **MUST** be one `UPDATE … SET name`, relying on `ON UPDATE CASCADE`. The codec's encode direction (§3.13) **MUST** be used for all enum values. _Source: design/profiles.md §3.5.3._

**PRF-059** (MUST) Before committing, publishing **MUST** replay schema, remote ops and the new file in a scratch SQLite, decode, and assert that the docs of the touched aggregates equal the intended docs and that every other doc is unchanged. A mismatch **MUST** abort the publish and return the diff. _Source: design/profiles.md §3.5.3._

**PRF-060** (MUST) A non-fast-forward push rejection **MUST** restart the publish from the fetch step once; a second rejection **MUST** fail the publish with an alert. _Source: design/profiles.md §3.5.3._

**PRF-061** (MUST) After the post-publish pull, published drafts are `satisfied` (PRF-038) and deleted; their events **MUST** record the published commit. _Source: design/profiles.md §3.5.3._

**PRF-062** (MUST) Publishing **MUST** push directly to the database's branch. A "push to a new branch and open a pull request" flow is deferred; it needs a host API beyond git. [OPN-057] _Source: design/profiles.md Q2; open-issues.md Group C._

**PRF-063** (MUST) Publishing the Local database ("Publish to repository…") **MUST** require an empty remote, write `pcd.json` (with the upstream schema dependency bonnyread targets, plus the extension schema dependency if any published entity needs it, §3.11), `ops/0.initial.sql` and `README.md`, push, and switch `kind` to `git` keeping all entity ids. _Source: design/profiles.md §3.5.3._

**PRF-064** (MUST) Pushing **MUST** work from the depth-1 state bonnyread keeps. A spike **MUST** confirm that go-git can push a commit whose parent is a shallow commit; if it cannot, publishing **MUST** fetch deeper for publish only. The git CLI **MUST NOT** become a runtime requirement. [OPN-058] _Source: design/profiles.md Q3; open-issues.md Group C._

**PRF-065** (MUST) Concurrent draft edits on the same path by several users **MUST** be last-write-wins, with `author_id` and `updated_at` visible. Publishing **MUST** be serialised per database (§3.20). _Source: design/profiles.md §3.5.4._

### 3.11 Extension schema (D20)

**PRF-066** (MUST) For vocabulary and features the upstream PCD schema cannot express, bonnyread **MUST** first propose additive changes to `Dictionarry-Hub/schema`. Until a proposal is released upstream (or if it is declined), the feature **MUST** be expressed in bonnyread's own **extension schema**. _Source: D20 (D9 amended); open-issues.md B2._

**PRF-067** (MUST) The extension schema **MUST** be a PCD-style schema repository (semver-tagged, `ops/*.sql`), listed as a second entry in a database's `pcd.json.dependencies` and applied after the upstream schema. It **MUST** be strictly additive over the upstream schema: it **MUST NOT** alter or drop upstream tables, columns or constraints. All its tables **MUST** use the `bx_` prefix. Its repository location, name and licence are to be confirmed. [OPN-133] _Source: D20; design/profiles.md §6.1 issue 1._

**PRF-068** (MUST) A database whose manifest declares the extension dependency is **not Profilarr-compatible** (Profilarr applies only the upstream schema's ops). bonnyread **MUST** show this on the database page. Adding the extension dependency to an existing git database **MUST** be an explicit `profiles:admin` action with a warning, and bonnyread **MUST NOT** emit extension-schema ops into a database whose manifest does not declare it. _Source: design/profiles.md §6.1 issue 1._

**PRF-069** (MUST) The extension schema **MUST** cover:

| Area | Content |
|---|---|
| Non-video qualities | Seed qualities for the Audio family (codec × bitrate bucket × bit depth; Lidarr's quality set), Text family (ebook formats such as EPUB, AZW3, MOBI, PDF), audiobook formats (M4B, MP3, FLAC), Archive family (CBZ, CBR, CB7, PDF) |
| Score domains | `arr_type` values for the domains PCD lacks: `audiobook` and `comic` (see PRF-080) |
| Non-video condition types | `audio_format`, `bit_depth`, `media` (CD, WEB, Vinyl …), `record_label`, `has_log`, `log_score`, `has_cue`, ebook `format`, comic scan type, and the adult-specific conditions needed by the Whisparr-Eros patterns |
| Test entities | Test-entity and test-release rows for music, book, audiobook, adult and comic (upstream `test_entities.type` is `CHECK IN ('movie','series')`) |
| Quality sizes | Quality-definition sets for the Audio (kbps-equivalent × duration), Text and Archive (absolute MB) families |
| Announce-metadata condition types | `uploader`, `freeleech_percent`, `announce_type`, `origin`, `category`, `tracker_tags`, `seeders`, `leechers`, `indexer` (portable form, PRF-070), `protocol`, `age`, `scene` |
| Autograb rules | The `AutograbRule` aggregate: scope, conditions, optional profile reference, identity policy, upgrade policy, limits, delay reference, priority |
| Delay settings | A quality profile → delay profile reference; tag scopes on delay profiles; delays with seconds granularity |

The exact column shapes are defined in the extension schema's own repository; bonnyread's canonical model (§3.13) is the contract they decode into. _Source: D20; open-issues.md B2, B3; design/evaluation.md §3.8, §3.14, §3.18, §6 I2–I4, Q7._

**PRF-070** (MUST) The extension schema **MUST NOT** carry deployment-specific bindings. In a publishable autograb rule the download-client/action binding and any external-exec or webhook condition **MUST** be local-only fields (stored as `local`-layer patches and never emitted). Indexer scoping in published data **MUST** use indexer tags or indexer definition ids, not local numeric indexer ids. [OPN-134] _Source: D20; design/evaluation.md §3.18._

**PRF-071** (MUST) Entities that need the extension schema **MUST** be creatable in any database whose manifest declares it and in the Local database. In a git database without the extension dependency, such an entity **MUST** be flagged "not publishable: needs extension schema" and kept in the `local` layer. _Source: design/profiles.md §3.8.3._

**PRF-072** (SHOULD) When an upstream schema release adopts a feature that the extension schema provides, both representations **MUST** decode to the same canonical docs, and bonnyread **SHOULD** offer authors a generated conversion op file that moves a database from the extension representation to the upstream one. _Source: D20 (upstream-first)._

**PRF-073** (MUST) Delay settings **MUST** attach to the quality profile (`QualityProfile.delay_profile`, a reference to a `DelayProfile` in the same database). A slot **MAY** override it: `SlotDefinition.delay_profile_override` and `TitleSlot.delay_profile_override` (edges owned by `02-domain-model.md`). The delay in effect for a slot is resolved in this order: TitleSlot override → SlotDefinition override → the highest-priority tag-scoped DelayProfile of the profile's database whose tags intersect the title's tags → the profile's `delay_profile` → no delay. Autograb rules carry their own delay reference. [OPN-113] _Source: D20; open-issues.md B3; design/evaluation.md §3.14, §6 I4._

### 3.12 Starter databases

**PRF-074** (MUST) The project **MUST** maintain one starter database (an ordinary PCD git repository using the extension schema) for each media kind that no community database covers: music (P2), and book/audiobook, adult and comic (P3). Each **MUST** ship at least one profile per SlotDefinition the kind defines by default, the CFs and regexes they score, quality-size definitions, and CF tests. _Source: D20; open-issues.md B2._

**PRF-075** (MUST) The adult starter database's CF patterns **MUST** be seeded from the Whisparr-Eros patterns (D1), with their licence notice. _Source: D1; design/profiles.md §3.8.3._

**PRF-076** (MUST) Starter databases **MUST** be linked like any other database (PRF-017), with policy `auto` and strategy `override` by default. _Source: design/profiles.md §3.5.5._

### 3.13 Canonical enums and the PCD translation layer

**PRF-077** (MUST) bonnyread **MUST** own the quality, language, source, resolution, quality-modifier, release-type and indexer-flag enums as Go types (the quality model is per family, `04-evaluation.md`). Each value **MUST** carry a `Kinds` set. There **MUST** be no enum tables in the deployment database and no `quality_api_mappings` equivalent. _Source: D9; design/profiles.md §3.7.1._

**PRF-078** (MUST) The codec (`internal/profiles/pcdcodec`) **MUST** be a set of bijective tables between PCD strings and canonical values, covering at least: the 31 upstream video qualities, 63 languages, 12 sources, 7 resolutions, 6 modifiers, 3 release types, 10 indexer flags, the `arr_type` ↔ score-domain mapping (PRF-080), the Radarr string and Sonarr integer media-management enums, the `radarr_*`/`sonarr_*` media-management tables ↔ presets with `kind = movie|tv`, and every extension-schema vocabulary. The codec **MUST** be used only when decoding after replay and encoding at publish. _Source: design/profiles.md §3.7.2._

**PRF-079** (MUST) The codec **MUST** round-trip: `encode(decode(x)) == x` for every row of Dictionarry, trash-pcd and each starter database, enforced by a CI test. Upstream `quality_api_mappings` rows **MUST** be read only to cross-check `Kinds` and **MUST** never be written. _Source: design/profiles.md §3.7.2._

### 3.14 Score domains and fallbacks

**PRF-080** (MUST) Score rows and conditions **MUST** be scoped by score domain, mapped from PCD `arr_type`:

| PCD `arr_type` | Domain |
|---|---|
| `all` | `*` (every domain the profile applies to) |
| `radarr` | `movie` |
| `sonarr` | `tv` (anime included) |
| `lidarr` | `music` |
| `readarr` | `book` (ebook) |
| `whisparr` | `adult` |
| extension value for audiobook | `audiobook` |
| extension value for comics | `comic` |

_Source: design/profiles.md §3.8.1._

**PRF-081** (MUST) Ebook and audiobook **MUST** be separate score domains (`book` and `audiobook`). A TitleSlot's domain is its media kind, except that a book SlotDefinition whose attributes mark it as audio maps to `audiobook`. [OPN-059] _Source: D4; design/profiles.md Q4; open-issues.md Group C._

**PRF-082** (MUST) When a domain has no specific score or condition row, resolution **MUST** follow a fixed chain: `adult → [adult, movie, *]`, `audiobook → [audiobook, book, *]`, every other domain → `[domain, *]`. The first match wins; resolution **MUST NOT** depend on row order. Editing only one domain's score **MUST** write one `scores[cf][<domain>]` patch and **MUST NOT** split the `*` row. _Source: design/profiles.md I7, §3.4.1, §3.8.1._

**PRF-083** (MUST) Condition types **MUST** declare the domains in which they are meaningful (for example `quality_modifier` and `edition` movie-only, `release_type` tv-only). The editor **MUST** reject a condition set to a domain where its type is meaningless, and condition rows for inapplicable domains **MUST** be ignored at resolution. _Source: design/profiles.md §3.8.1._

### 3.15 Profiles and version slots

**PRF-084** (MUST) A TitleSlot's `quality_profile` and a SlotDefinition's `default_quality_profile` **MUST** reference the `PcdEntity` id of a `QualityProfile` (edges owned by `02-domain-model.md`). There is exactly one profile per TitleSlot. _Source: D4; design/profiles.md §3.8.2; open-issues.md A13._

**PRF-085** (MUST) A profile's enabled items **MUST** all belong to one quality family; an upstream profile that mixes families **MUST** get a `problem` and **MUST NOT** be bindable. A profile's `kinds` **MUST** be derived from its items (the kinds whose family contains at least one enabled item). Binding **MUST** be rejected with a validation error unless `profile.kinds ∋ slot.kind`. _Source: design/profiles.md §3.8.2; design/evaluation.md §3.6._

**PRF-086** (MUST) Upstream renames **MUST** never break a binding (bindings are by id, PRF-022), and upstream deletions **MUST** never break one (the profile is detached, PRF-047). _Source: design/profiles.md §3.8.2._

### 3.16 Change propagation and the read interface

**PRF-087** (MUST) Any effective change to a profile, a CF it scores, a regex those CFs use, or a delay profile it references **MUST** bump the affected entity revisions and the global generation, and **MUST** publish `profiles.changed{generation, profile_ids}`, with `profile_ids` computed from reverse edges, through the transactional event bus (durable delivery, `12-platform.md`). The library and wanted subsystems re-check cutoff and upgrade eligibility of the affected slots lazily and in batches. `NOTIFY` **MAY** be used only as a best-effort cache-invalidation doorbell. _Source: design/profiles.md §3.8.2, §3.9; open-issues.md A4._

**PRF-088** (MUST) The store **MUST** expose a Go read interface providing: resolved, immutable views of a profile for a domain (ordered canonical quality items with group structure, cutoff, upgrade and minimum scores, language gate, resolved delay policy, and scored CFs already resolved through the domain fallback chain); compiled CFs for a domain (conditions filtered to the domain, regex conditions holding compiled patterns from §3.19); compatible profiles for a kind; referrers of an entity; a whole-store snapshot for a kind keyed by generation, for `04-evaluation.md`; and a change subscription. Views **MUST** be cacheable by `(entity id, revision, domain)`. _Source: design/profiles.md §3.10.1; design/evaluation.md §3.17._

**PRF-089** (MUST) CF match semantics (AND across condition types; within a type, required conditions ANDed and the rest ORed; per-condition negate) **MUST** be implemented once, in the evaluation condition engine (`04-evaluation.md`). The profile store and its testing features **MUST** call that engine and **MUST NOT** re-implement matching. _Source: design/profiles.md §3.10.1._

### 3.17 Testing and simulation

**PRF-090** (MUST) CF tests (`POST /api/v1/profiles/test/custom-format {cf_id | cf_doc, domain, titles?}`) **MUST** parse each title with the in-process parser for the domain's kind (D8), evaluate it with the shared condition engine, and return per-condition results (matched, negate, required, matched span for patterns) and pass/fail against `should_match`. With `cf_doc` the test **MUST** run on unsaved edits. Titles default to the CF's own tests. _Source: design/profiles.md §3.10.3._

**PRF-091** (MUST) Regex tests (`POST /api/v1/profiles/test/regex {pattern, samples[]}`) **MUST** use the production normalise-and-compile path (§3.19) and return matches, groups, `compile_error` and timeouts. `regex101_id` **MUST** remain a link only. _Source: design/profiles.md §3.10.3._

**PRF-092** (MUST) Profile simulation (`POST /api/v1/profiles/test/profile {profile_id | profile_doc, domain, releases}`) **MUST** accept as `releases`: a test entity (`test_entity:<id>`); a library item or item slot (`item:<id>[/slot]`: its most recent interactive-search results, recent RSS/IRC candidates and grab history, all local); or an explicit list of titles, sizes, flags and indexers. It **MUST** return ranked releases with per-CF score breakdown, quality rank, cutoff position and rejection reasons, computed by the evaluation engine's pure evaluate function. A test entity with a TMDb id **MUST** link to the matching library item when one exists. _Source: design/profiles.md §3.10.3._

**PRF-093** (MUST) Compare (`POST /api/v1/profiles/test/compare {a, b, releases}`) **MUST** rank the same releases under two profile docs side by side with rank moves; `a` and `b` **MAY** be the upstream doc, the effective doc, or a draft. _Source: design/profiles.md §3.10.3._

**PRF-094** (MUST) Update preview (`GET /api/v1/profile-databases/{db}/update-preview?commit=`) **MUST** run prepare and merge in a transaction that is rolled back, and return changed aggregates with field diffs and renames, the conflicts the database's strategy would produce, and, per bound profile, the number of slots whose cutoff-met or upgrade-candidate status would change, computed from stored parse results without re-parsing. _Source: design/profiles.md §3.10.3._

**PRF-095** (MUST) Built-in PCD CF tests **MUST** run through the same engine in CI and in the UI. When a Dictionarry CF test disagrees with the ported Sonarr/Radarr parser tests about attribute detection (source, resolution, modifier, language), the parser tests are authoritative; the CF test failure **MUST** be recorded as an entity `problem` and not block the pull. _Source: D20 (attribute authority); open-issues.md B4._

### 3.18 Migration importers

**PRF-096** (SHOULD) A one-shot importer **SHOULD** convert Profilarr user ops (their `desired_state` JSON) into `local` patches, in P1 if it is cheap. [OPN-061] _Source: design/profiles.md Q6; open-issues.md Group C._

**PRF-097** (MAY) A `recyclarr.yml` translator into trash-pcd plus `local` patches **MAY** be provided after P1. There **MUST NOT** be a direct TRaSH JSON importer. [OPN-063] _Source: D9a; design/profiles.md Q8; open-issues.md Group C._

### 3.19 Regex path (D1)

**PRF-098** (MUST) Every CF pattern **MUST** pass through a pure normaliser that rewrites .NET named blocks `\p{IsX}` and `\P{IsX}` to code-point ranges from a generated table (spliced inside a character class, wrapped in `[…]`/`[^…]` outside one), and rewrites nothing else. The normaliser is a shared package also used by the indexer layer (D1b). A golden test **MUST** compile every pattern of every linked and starter database. _Source: D1; design/profiles.md §3.11._

**PRF-099** (MUST) Patterns **MUST** compile with `regexp2` in default (.NET) mode with `IgnoreCase`, and match with `MatchTimeout = 100ms`. _Source: D1; design/profiles.md I8, §3.11._

**PRF-100** (MUST) The compile cache **MUST** be node-local, content-addressed by `sha256(normalised pattern + options)`, shared across databases and CFs, bounded by an LRU of 8,192 entries, with immutable entries. Compiled-CF caches **MUST** be keyed by `(cf id, revision, domain)`. _Source: design/profiles.md §3.11._

**PRF-101** (MUST) On pull, every regex **MUST** be compiled; a failure **MUST** be written to `Regex.compile_error` and the entity's problems, the dependent condition never matches, a health notice is raised, and the pull continues. On a user write, a compile failure **MUST** reject the write with a validation error. Local patterns **MUST** be at most 4 KiB. _Source: design/profiles.md §3.11._

**PRF-102** (MUST) At match time a timeout **MUST** count as no match (evaluation reports it as `Unknown`), increment `bonnyread_regex_timeouts_total{source="custom_format"}` (`12-platform.md` PLT-134), and raise a rate-limited health notice naming the regex. _Source: design/profiles.md §3.11; design/evaluation.md §3.8._

### 3.20 HA placement

**PRF-103** (MUST) Check, pull, link, publish and update preview **MUST** run as durable, keyed jobs that any node may claim (no leader or lease), with dedupe keys `pcd.sync:<db>`, `pcd.publish:<db>` and `pcd.preview:<db>:<commit>` and the concurrency key `pcd:<db>`, so at most one git job per database runs cluster-wide. Periodic checks **MUST** be schedule rows (`12-platform.md`). _Source: open-issues.md A1; D20 (D14 amended)._

**PRF-104** (MUST) User edits, conflict resolution, clone and reset **MUST** run on any node in one transaction taking `SELECT … FOR UPDATE` on the `PcdDatabase` row, so they serialise with a pull's apply phase per database. _Source: design/profiles.md §3.9._

**PRF-105** (MUST) A job whose node dies mid-apply **MUST** leave no partial effective state (the transaction rolls back); the re-claimed job re-runs idempotently (PRF-027). A new node **MUST** be able to run any git job using only the database contents and the remote. _Source: design/profiles.md §3.9; open-issues.md A2._

**PRF-106** (MUST) Scratch SQLite instances **MUST** be per job and released when the job ends. _Source: design/profiles.md §3.9._

### 3.21 API, permissions and observability

**PRF-107** (MUST) The profile API (under `/api/v1`, `11-api-and-clients.md`) **MUST** provide: database CRUD, check, update (optional commit), pulls and events listings; conflicts listing and resolution (`POST …/conflicts/{patchId}/resolve {keep|take}` and a bulk form); drafts listing, deletion and publish (`{patch_ids, message}`); and, for each aggregate, list/get/create/update (`?layer=local|draft`)/delete, `POST …/{id}/reset {paths?}` and `POST …/{id}/clone {target_db, name?}` (action sub-resources, API-012). `GET` of an aggregate **MUST** return the effective doc plus per-path provenance (upstream value, layer value, state `clean|overridden|pending|detached`). _Source: design/profiles.md §3.10.2._

**PRF-108** (MUST) Permissions (`10-identity-and-access.md`): `profiles:read`; `profiles:write` (local layer, conflict resolution); `profiles:author` (drafts, publish); `profiles:admin` (link, unlink, credentials, strategy, policy, priority, extension dependency). These names do not match the `resource:action` vocabulary of `10-identity-and-access.md` IAM-061/IAM-062 (resource `profile`); the mapping is open [OPN-205]. _Source: design/profiles.md §3.10.2; D17._

**PRF-109** (MUST) The store **MUST** export `bonnyread_profiles_pull_total{db,status}`, `bonnyread_profiles_pull_duration_seconds`, `bonnyread_profiles_conflicts_pending{db}`, `bonnyread_profiles_detached_entities{db}`, `bonnyread_profiles_update_available{db}`, `bonnyread_profiles_regex_compile_errors` and `bonnyread_regex_cache_entries`, count match timeouts in `bonnyread_regex_timeouts_total{source="custom_format"}`, and **MUST** alert on a blocked update, a failed publish and a new pending conflict, as the notifiable events `profile.update_blocked`, `profile.publish_failed` and `profile.conflict_pending` (`12-platform.md` PLT-074). _Source: D7; design/profiles.md §3.10.4._

---

## 4. Data model

Owned here. `TitleSlot`, `SlotDefinition` and their edges to `QualityProfile` and `DelayProfile` are owned by `02-domain-model.md`. Secrets follow `12-platform.md`.

```go
// ---- databases and provenance -------------------------------------------------
PcdDatabase {
  name string unique; short_name string unique      // short_name: clone suffix, picker label
  kind enum(git, local, bundled)
  repo_url string?; branch string default "main"; secret_ref string?
  git_author_name string?; git_author_email string?
  conflict_strategy enum(override, align, ask) default override
  update_policy enum(auto, notify, frozen) default auto
  check_interval_min int default 60
  priority int unique
  writable bool
  pinned_commit string?; remote_head string?
  manifest json; schema_version string; extension_version string?   // null = Profilarr-compatible
  applied_ops json            // [{path, sha256}] at pinned_commit (PRF-019, PRF-029)
  health enum(ok, update_blocked, error); last_error string?
}
PcdBlob        { sha256 string PK; content bytes }                       // PRF-018
PcdTreeFile    { owner enum(database, dependency); owner_id; commit string; path string; blob -> PcdBlob
                 unique (owner, owner_id, commit, path) }
PcdDependency  { url string; tag string; commit string; unique (url, tag) }

PcdEntity {                    // stable identity; its id is what everything references
  database -> PcdDatabase
  kind enum(regex, custom_format, quality_profile, delay_profile, naming,
            media_settings, quality_size, test_entity, autograb_rule)
  name string; name_fold string
  origin enum(upstream, local, detached)
  upstream_present bool
  revision int64
  upstream_doc json?; upstream_hash string?
  problems json               // [{code, detail}]
  unique (database_id, kind, name)
  index (database_id, kind, name_fold)
}

PcdPatch {
  entity -> PcdEntity; layer enum(local, draft)
  op enum(create, set, add, remove, delete); path string
  base json?; value json?
  state enum(active, pending, published)
  conflict_reason enum(guard_mismatch, duplicate_key, missing_target)?
  pending_upstream json?
  local_only bool             // PRF-070: never emitted
  group_id uuid; author -> User; updated_at time
  unique (entity_id, layer, path)
  index (state)
}

PcdEvent { database -> PcdDatabase; pull -> PcdPull?; entity -> PcdEntity?; patch_id?
           type enum(applied, satisfied, rebased, aligned, pending, resolved, detached,
                     renamed, published, pull_failed, publish_failed)
           detail json; at time }
PcdPull  { database -> PcdDatabase; from_commit?; to_commit; trigger enum(schedule, manual, publish, link)
           status enum(running, applied, no_change, blocked, failed); stats json; renames json
           started_at; finished_at? }
ProfileGeneration { id = 1; generation int64 }   // PRF-087

// ---- effective state (row id == PcdEntity id) ---------------------------------
Regex          { pattern; description; regex101_id?; tags []string; compile_error? }
CustomFormat   { description; include_in_rename bool; tags }
  CfCondition  { cf -> CustomFormat; name; type enum; domain string; negate; required
                 regex -> Regex?; payload json; unique (cf_id, name) }
  CfTest       { cf -> CustomFormat; title; domain; should_match bool; description }
QualityProfile { description; upgrades_allowed; min_cf_score; upgrade_until_score
                 upgrade_score_increment; language?; tags
                 items json      // ordered: quality | group{name, members[]}, enabled, cutoff
                 family enum; kinds []Kind
                 delay_profile -> DelayProfile? }
  ProfileScore { profile -> QualityProfile; cf -> CustomFormat; domain string; score int
                 unique (profile_id, cf_id, domain); index (cf_id) }
DelayProfile   { preferred_protocol; only_protocol?; usenet_delay_s; torrent_delay_s; ddl_delay_s
                 bypass_if_highest_quality; bypass_min_score?; tags []string }
NamingPreset / MediaSettingsPreset / QualitySizePreset { kind; typed fields }
TestEntity     { kind; external_ids json; title; year?; poster? }
  TestRelease  { test_entity -> TestEntity; title; size; languages; indexers; flags }
AutograbRule   { scope json; conditions json; profile -> QualityProfile?; identity enum
                 upgrade enum; limits json; delay_profile -> DelayProfile?; priority int
                 action_ref?  /* local only */ }
```

Notes:
- Effective tables are relational where other code queries them (score → "which profiles use this CF", condition → regex edge). Profile `items` are JSON because they are always read and written whole.
- Profilarr's per-type condition tables become one `CfCondition` row with a typed payload; the codec still reads and writes the per-type tables.
- `ddl_delay_s` covers D11's third protocol; PCD has no DDL delay, so it decodes to 0 and needs the extension schema to be published.

---

## 5. Flows

### 5.1 Pull

```
prepare (no lock)                                 apply (one tx, FOR UPDATE on PcdDatabase)
1 fetch target commit, depth 1                    7  renames: PcdEntity.name := new (ids kept)
2 read pcd.json; resolve schema + extension deps  8  upsert upstream_doc/hash for added/changed
  (cached per url@tag; blobs stored, PRF-018)     9  removals: tombstone or detach (§3.8)
3 compatibility gate (PRF-004..006)               10 re-merge touched entities with patches;
4 replay in scratch SQLite, FKs on, helpers,         classify; apply strategy (§3.7)
  rename triggers                                 11 write effective rows; bump revisions + generation
5 decode → canonical docs (codec)                 12 pinned_commit, applied_ops, tree manifest
6 diff by hash: added/changed/removed/renamed     13 events; enqueue profiles.changed; commit; NOTIFY
```

Failure paths:
- Fetch or auth error: the pull record is `failed`, `health = error`, state unchanged, retry on next schedule.
- Unsupported schema major or compatibility level: `blocked`, `health = update_blocked`, state unchanged.
- Op or tweak error during replay: `blocked`, failing file and error reported (PRF-024).
- Node dies during apply: the transaction rolls back; the job is re-claimed and re-run (PRF-105).

### 5.2 User write

1. Request carries a full doc (editor) or field ops (API).
2. Validate (PRF-035); on failure return 422 with per-path errors.
3. Lock the `PcdDatabase` row; diff against effective-without-layer; upsert or delete patches.
4. Re-merge the entity; write effective rows; bump revision and generation if the effective state changed.
5. Enqueue `profiles.changed` if it changed; commit.

### 5.3 Publish

1. Enqueue `pcd.publish:<db>` with the selected patch ids and message; the closure is added (PRF-056).
2. Fetch the remote head; if ahead, pull (§5.1). If any selected draft is now pending, stop with `409 drafts_pending`.
3. Emit the op file (PRF-058).
4. Round-trip validation (PRF-059); on mismatch stop with the diff, event `publish_failed`.
5. Commit and push. On non-fast-forward, go to step 2 once (PRF-060).
6. Pull the new head; published drafts become satisfied; events record the commit.

### 5.4 Walkthrough setup

Dictionarry is linked. Profile **"1080p Quality"** (id P) is referenced by the TitleSlots of 900 movies and 300 series. CF **"x265 (HD)"** (id C) is scored in P with the upstream row `(P, C, *) = -1000` at commit `c1`.

### 5.5 Walkthrough: the user edits a score, then upstream changes it

**t0.** The user sets C's score to 0 for all domains. Patch `{P, local, scores[C][*], base -1000, value 0}`. Effective `ProfileScore(P, C, *) = 0`; revision and generation bumped; `profiles.changed{P}`; the library re-checks 1,200 slots lazily.

**t1.** Commit `c2` changes the row to -2000. Pull: P's doc hash changed → merge P: `u = -2000`, `base = -1000`, `value = 0` → `guard_mismatch`.

| Strategy | After the pull | Visible to the user |
|---|---|---|
| `override` | Effective stays **0**; `base := -2000`; event `rebased` | Pull summary "1 override kept over an upstream change"; provenance "upstream: -2000 · yours: 0". No `profiles.changed` (PRF-045) |
| `align` | Patch deleted; effective **-2000**; event `aligned` | "1 local change replaced by upstream"; `profiles.changed{P}` |
| `ask` | Patch `pending`, `pending_upstream = -2000`; effective stays **0** (PRF-042) | Conflicts page shows was -1000, upstream -2000, yours 0, with **Keep mine** (override) and **Take upstream** (align); alert raised; no `profiles.changed` until resolved |

Variants:
- Upstream moves to 0: `u == value`, satisfied, patch deleted under every strategy, no conflict.
- Upstream changes another field of P: the score path is clean (PRF-040).
- The user had edited only movies (`scores[C][movie] = 0`, base absent): upstream's change to `[*]` doesn't touch that path; movies stay 0, series follow upstream to -2000, no conflict.
- Under `ask`, a later commit `c3` reverts the row to -1000: the pending patch is re-classified clean and becomes active (PRF-044).

### 5.6 Walkthrough: upstream renames a CF referenced by an override

State: local patches `scores[C][*] = 0` on P and `description` on C. Commit `c3` adds `ops/412.rename-x265.sql`: `update "custom_formats" set "name"='x265 (1080p)' where "name"='x265 (HD)'`.

1. Replay: the trigger journals `(custom_formats, 'x265 (HD)' → 'x265 (1080p)', 412.rename-x265.sql)`; `ON UPDATE CASCADE` rewrites referencing scratch rows.
2. Diff: `412…` is not in `applied_ops`, so it is this pull's rename. `PcdEntity(C).name` changes; its id doesn't. P's doc references C by id, so P's hash is unchanged and P is not re-merged.
3. Merge C: the `description` patch is clean.
4. Result: no conflict under any strategy; both overrides survive; slots are unaffected; the only event is `renamed`. Drafts emitted later use the new name.
5. If the user had also renamed C locally (`name` patch, base `'x265 (HD)'`), `u = 'x265 (1080p)'` → `guard_mismatch`, decided by the strategy.

Harder variant: upstream deletes "x265 (HD)" and creates "x265 (1080p)" with changed conditions. The journal is empty and content differs, so it is a delete plus an add.
- `override`: C is detached (still referenced by P's patch); the score survives on the local copy; a notice offers "Retarget my overrides to *x265 (1080p)*" (PRF-049).
- `align`: patches on C are dropped; P scores the new CF at upstream's value.
- `ask`: both patches pend with C detached and in effect.

### 5.7 Walkthrough: linking a second database with overlapping names

The admin links trash-pcd with priority 2 (Dictionarry is 1).

1. Pull from nothing: 201 CFs, 989 regexes, 22 profiles and so on are created in the trash-pcd namespace. Dictionarry `AMZN` and trash-pcd `AMZN` are two entities. Regex `3L` (`(?<=^|[\s.-])3L\b` vs `^(3L)$`) stays distinct; each CF resolves its regex inside its own database. `CRiT` vs `CRIT` is allowed across databases.
2. The regex cache compiles byte-identical patterns once (PRF-100).
3. Unqualified `"WEB-1080p"` resolves to trash-pcd (only it has the name). Unqualified `"AMZN"` resolves to trash-pcd by priority, with `resolved_from: "trash-pcd"`. `"Dictionarry/AMZN"` resolves explicitly.
4. The user wants Dictionarry's profile with TRaSH's `AMZN`: *Clone into Dictionarry* finds the collision and prompts **rename** (creates `AMZN [trash]` plus its regex closure) or **replace** (field patches on Dictionarry's `AMZN`, so future Dictionarry changes to it conflict per strategy) or **skip**. No cross-database reference is created.
5. A tv TitleSlot bound to trash-pcd `WEB-1080p` resolves scores via `[tv, *]`.
6. Unlinking trash-pcd while that slot references `WEB-1080p` is blocked; "move in-use profiles to Local" is offered (PRF-013).

---

## 6. Non-goals

| Non-goal | Reason |
|---|---|
| Sync, drift detection, cleanup and `on_pull` pushes into external Radarr/Sonarr | The store is native; no external Arr holds live config (D9a, D7) |
| Per-Arr integer enum mapping, `quality_api_mappings`, enum tables | bonnyread owns the enums; the codec runs only at the PCD boundary (D9) |
| Direct TRaSH JSON importer | TRaSH is supported through trash-pcd (D9a) |
| .NET parser sidecar, parser/match caches, a second CF matcher | The in-process parser and one condition engine (D8, PRF-089) |
| Rebuilding an in-memory database after every write | Effective state is persisted; writes re-merge one aggregate (D19) |
| Cross-database references | Every database must stay self-contained and exportable |
| Pull-request publishing flow in v1 | Needs host APIs; deferred (PRF-062) |
| A shared filesystem for git data | Content is stored in the database (A2) |

---

## 7. Traceability

| Reference | Preserved | Changed deliberately (D19) |
|---|---|---|
| Profilarr | PCD format and replay order; name-keyed entities; base/tweak/user layering; override/align/ask; guard-mismatch, duplicate-key and missing-target classes; "goal achieved" drop; field granularity and whole-list items; per-domain scores; drafts exported as the next op file and pushed; CF tests, entity testing, regex testing; Dictionarry linked on first boot | Structured patches instead of guarded SQL; `ask` keeps the user's value in effect (D20); renames from replay triggers instead of regex over SQL text; references by id; persisted effective state instead of per-write in-memory recompiles; a failed base op blocks the update instead of disabling the database; go-git on any host instead of the git CLI on GitHub; round-trip publish validation; databases namespaced, priority only for unqualified names; wildcard-plus-specific scores with fixed fallback; extension schema for non-video kinds, announce conditions, autograb and delay (D20) |
| Recyclarr | Non-destructive defaults: never zero unmanaged scores, never delete what bonnyread didn't create; stable ids across renames | No per-run guide fetch without stored state; overrides can conflict and are surfaced |
| Sonarr/Radarr | CF semantics, .NET regex semantics (case-insensitive, 100 ms timeout); delay-profile bypass rules | Delay attached to the quality profile with per-slot override instead of tag-scoped Arr delay profiles only; release profiles dropped (`04-evaluation.md`) |
| autobrr | Filter fields expressible as conditions; announce metadata | Filters become publishable autograb rules in the extension schema, with local-only client bindings |
