# Design Review: Consolidated Issues

This consolidates section 6 ("Issues with decisions" and "Open questions") of the nine design documents in this folder, as of 2026-09-28. Issues are grouped as follows:

- **A. Cross-design resolutions.** These are conflicts or gaps between designs that can be settled on engineering grounds. Each lists a proposed resolution, to be confirmed.
- **B. Decisions for the user.** These are product or policy choices the designs cannot settle.
- **C. Proposed defaults.** These are smaller questions. Each carries a proposed default, to be carried into the specification as a reviewable open item.

References such as `acq I3` mean design document `acquisition.md`, section 6, item I3. Other abbreviations: `dl` = downloads.md, `eval` = evaluation.md, `id` = identity-api-deployment.md, `imp` = import.md, `lib` = library.md, `meta` = metadata-and-subtitles.md, `prof` = profiles.md, `rt` = runtime.md.

---

## Outcome (2026-09-28)

- **Group A:** accepted as written, subject to the user's final review of the specification.
- **Group B:** decided as recorded in the table below (column "Decision") and in `../decisions.md` D20.
- **Group C:** carried into the specification as proposed defaults, each marked as an open item for the user's final review.

## A. Cross-design resolutions (accepted, pending final review)

| # | Issue | Raised in | Proposed resolution |
|---|---|---|---|
| A1 | What "leader election" means in D14. Acquisition assumed a leader-scheduled RSS tick, downloads used a lease per client, and runtime proposed no leader for schedules at all | acq I4, dl 1, rt 2 | Adopt **runtime's model**. Every node runs the scheduler over schedule rows claimed with `SKIP LOCKED`, and dedupe/concurrency keys give "at most one execution per subject". Download-client polling becomes one schedule row per client with a per-client concurrency key, which is equivalent to downloads' per-client lease. **Leases are kept only for long-lived connections** (IRC, file watcher). Git publish, link and update-preview (prof 5) run as keyed jobs. |
| A2 | In HA, where runtime-fetched content lives: indexer definitions, IRC definitions, PCD git data, plug-in modules | acq I1, id 6, prof 5, rt | **Store it in the database.** Nodes keep disposable local caches. No shared data volume is needed for these. The legal review (D5b) must cover "stored in the user's database and therefore in their backups". |
| A3 | Cross-node rate limiting for trackers and strictly limited metadata sources | acq I3, meta 5b, rt Q4 | A **DB-backed shared rate limiter**, with "next allowed at" tokens plus quota buckets, used by indexers (request spacing, query and grab budgets) and metadata sources (MusicBrainz, ComicVine, Hardcover, Metron, AcoustID). Interactive traffic has reserved lanes. |
| A4 | Reliability of LISTEN/NOTIFY | rt 3 | NOTIFY is only a best-effort **doorbell and invalidation** channel. Anything that must be delivered goes through the transactional job queue, which doubles as the outbox. The listener needs a session-level connection (no PgBouncer transaction pooling). |
| A5 | Filesystem co-location for staging, downloads and library | dl 2, imp 5 | A **deployment requirement plus a health check**. The extraction staging root must be on the same filesystem as each library root. Downloads must be on the library's filesystem when hardlink or reflink imports are wanted. Otherwise imports fall back to copying, visibly rather than silently. |
| A6 | Jobs that need native tools (ffprobe, fpcalc, alass, ffsubsync) | meta 5a, imp 4 | **Capability-routed jobs.** Nodes advertise the tools they have, and jobs declare the tools they need. Desktop installers and container images bundle ffprobe and fpcalc. alass is bundled where the licence allows, and ffsubsync is optional. |
| A7 | Rolling upgrades in active-active HA, and version skew between native apps and servers | rt 7, id 3 | **Expand/contract migrations with N-1 compatibility** for the schema, and **additive-only changes within API v1**, enforced in CI (oasdiff). Apps negotiate `apiVersion`/`minClientApiVersion` and fall back to the server's own web UI. |
| A8 | Secrets that can't live in the database they unlock (DSN, TLS key, OIDC bootstrap) | id 5 | A **bootstrap config file plus env/`*_FILE`** for these values only. Everything else stays in the database (D16). |
| A9 | `/metrics` with N replicas | rt 5 | Per-node `/metrics` for process metrics, plus `/metrics/cluster` for database-derived gauges. |
| A10 | D19 and text from external systems | dl 3 | D19 applies to **internal** state. Text that only external systems supply (SABnzbd/NZBGet failures, tracker status messages) is matched only inside per-client adapter classifiers, which have tests and emit enums. |
| A11 | DDL host plug-ins | dl 4 | Plug-ins **resolve** links (URL, headers, cookies, quota). The **core streams the bytes**. |
| A12 | Absolute file paths (D2 wording) | lib I1 | Store **root folder id plus a path relative to the root**. This covers files outside a title folder and makes root moves cheap. |
| A13 | Where slots are defined, configured and monitored (D4) | lib I2, imp 1 | Slot **definitions** are global per media kind. **Configuration** (profile, root folder) is per title. **Monitoring and coverage** are per item. The preferred **edition** (books) and monitored **release** (music) belong to the slot, not the item. |
| A14 | Uniqueness of product codes (D10) | lib I4 | A separate **non-unique** index, not part of the unique external-id map. |
| A15 | Credential issued by desktop pairing (D12) | id 1 | A **device credential** (expires when unused, listed per device, revocable) stored in the same table as API keys, not a long-lived API key. |
| A16 | Custom Newznab category ids (`100000+`) | acq I5 | Not used internally; categories are keyed by tracker-native keys. The derivation is kept only if a Prowlarr importer is built. |
| A17 | Phase-1 groundwork for later phases (D2) | meta 7 | Phase 1 must include: subtitle sidecar indexing and stream flags in the shared probe; TMDb discovery with TMDb→TVDB resolution for requests; per-kind quality axes and M:N file links in the schema. |
| A18 | The PCD decoder needs SQLite on every deployment | prof 4 | Embed pure-Go SQLite (`modernc.org/sqlite`) as a library in every build. It is also the SQLite driver for small installs, so the image stays `CGO_ENABLED=0`. |
| A19 | Cardigann template fidelity | acq I6 | A **purpose-built interpreter** for the Jackett-style template language, not Go's `text/template`, plus a golden suite that renders every template in every definition against recorded fixtures. Tracked as the top fidelity risk. |
| A20 | IRC regex latency | eval I5 | A per-release evaluation deadline plus literal prefilters before regexes, in addition to D1's per-match timeout. |

## B. Decisions for the user

| # | Question | Raised in | Options (proposed first) | Decision |
|---|---|---|---|---|
| B1 | **One release wanted by two slots.** Evaluation assumed one download feeding several slots; downloads assumed one slot per download (the release is grabbed twice). Import assumed one file fills one slot. | eval Q1, dl 5/Q1, lib I3, imp 2 | (a) One download imports into every slot that wants it, as separate files hardlinked or copied into each slot's root; (b) one slot per download (grab twice); (c) reject overlapping slot profiles at configuration time | **(a)** One download imports into every slot that wants it |
| B2 | **PCD beyond video.** The PCD schema only expresses video (31 video qualities, no audio/ebook/comic conditions, tests limited to movie/series). No community database covers music, books, comics or adult. | prof 1–2, eval I2, Q7 | (a) Propose additive changes to Dictionarry's schema upstream, falling back to (b); (b) our own extension schema (databases using it aren't Profilarr-compatible); plus who maintains starter databases for the new kinds | **(a)** Upstream first; our extension schema as fallback; project maintains starter databases |
| B3 | **Local-only profile features.** Announce-metadata conditions, autograb rules and tag- or slot-scoped delay have no PCD representation. | eval I3–I4 | (a) Local-only, stored in our DB and not publishable; delay attached to the quality profile with a per-slot override; (b) include them in the extension schema from B2 | **(b)** In the extension schema (publishable) |
| B4 | **Authority for attributes.** When the Arr parser tests and a Dictionarry CF test disagree about source, resolution or modifier detection, which wins? | eval I1 | (a) Arr parser tests define attribute detection, and Dictionarry defines CF patterns; a conflict is a bug reported upstream; (b) Dictionarry wins | **(a)** Arr parser tests win for attribute detection |
| B5 | **MySQL/MariaDB.** It can only be single-node (HA needs Postgres, D14), which SQLite already covers, yet it costs a third SQL dialect, a third migration tree and a third CI leg. | rt 1 | (a) Drop it: support Postgres and SQLite only; (b) keep it, labelled single-node best-effort; (c) fully supported | **(a)** Drop MySQL/MariaDB |
| B6 | **Import-time processing hook** (remux or transcode), replacing Sonarr's script import. WASM plug-ins can't run ffmpeg. | imp 6 | (a) An "external processing" hold state: import waits until a webhook or API callback marks the file ready; (b) a host capability for plug-ins to request a transcode; (c) accept the loss | **(a)** External processing hold state |
| B7 | **Coverage-aware selection.** Sonarr's greedy ranking can pick two single episodes over a season pack that also covers eight more missing episodes. | eval Q4 | (a) Coverage-aware selection: prefer the choice that satisfies more wanted items, subject to profile scores; (b) keep Sonarr's behaviour | **(a)** Coverage-aware selection |
| B8 | **"Ask" conflict strategy.** What is in effect while a conflict waits for the user? | prof 3 | (a) The user's value stays in effect (nothing changes without consent); (b) the upstream value applies until the user decides (Profilarr's effective behaviour) | **(a)** User's value stays in effect |
| B9 | **TVDB licensing risk.** TV works as decided only if the project obtains TVDB's free/OSS licensed tier; otherwise every user needs a paid PIN. | meta 1 | (a) Contact TVDB before phase 1; if refused, fall back to TMDb TV plus an episode-order mapping layer; (b) accept the paid-PIN requirement | **(a)** Apply for TVDB free/OSS tier; TMDb + mapping fallback if refused |
| B10 | **Backups contain plaintext secrets.** | rt 4, id 5 | (a) Optional passphrase encryption of backup archives, plus a clearly labelled sanitised export; (b) nothing beyond D16 | **(a)** Optional backup encryption + sanitised export |
| B11 | **TMDb 180-day rule** for library data copied from TMDb (descriptions, images, ids). | meta 3, meta Q2 | (a) Refresh within 180 days; if impossible, purge TMDb descriptions and cached images but keep ids; (b) purge ids too | **(a)** Purge descriptions and images, keep ids |
| B12 | **Scope of the legal review** (before the first public release). | acq I1–I2, meta 4, meta 6, imp 7, meta Q9 | Confirm it covers: fetched definitions stored in user databases and backups; the **unlicensed** Prowlarr/Indexers repo; community anime and scene datasets with unverified licences; licensing of third-party WASM plug-ins (combined work); AcoustID non-commercial keys under AGPL; Hardcover's clause on user-owned data | All items in scope |
| B13 | **Smaller product policies** | id 4, id 7, id 2 | Mobile push in v1: **none** (SSE plus existing targets such as ntfy). Forward-auth (Authelia/Authentik) login: **opt-in, off by default**. Pairing secret: **ACL-restricted file** (D12) or **local socket with OS peer credentials** (stronger) | No push in v1; forward-auth opt-in, off by default; pairing over local socket with OS peer credentials |

## C. Proposed defaults (carried into the spec as open items)

**Acquisition**
- Prowlarr v11 is the primary definition feed, fetched from GitHub by commit; Jackett is an opt-in overlay, later.
- Embed a baseline of autobrr IRC definitions (GPL-2.0-or-later) and fetch updates.
- The tracker alias map is curated in-repo with CI checks.
- Budget defaults are to be validated against real tracker limits.
- Config importers for Prowlarr, Jackett and autobrr are later, not phase 1.
- Seed policy and download-client routing are owned by the download lifecycle, with hints from the indexer.
- DDL site definitions extend the Cardigann engine with an `article.groups` block.
- Racing grabs count against per-indexer budgets.

**Downloads**
- Superseded active downloads are cancelled on public trackers and kept on private ones.
- Partial packs are blocked until the user acts, with the policy exposed as a setting.
- The seed obligation mode defaults to "ratio OR time", populated from definitions where available.
- Default timeouts are tuned in beta.
- Unpackerr's watch-folder mode is dropped.
- Seed limits are not pushed to the client by default.
- "Adopt an unmanaged client item" comes in phase 1 only if cheap; otherwise use manual import from a path.
- Blocklisting is permanent by default, with expiry for `no_progress` on public torrents.

**Evaluation**
- When a library slot and a racing rule accept the same announce, the racing rule skips it.
- The enrichment contract with acquisition is agreed in the spec.
- The D8 spike measures attribute agreement as well as structure.

**Identity and API**
- Passkeys come later.
- Requester quota defaults to 10 movies and 10 seasons per 7 days.
- TOTP is mandatory for Owner/Admin unless the server is loopback-only.
- macOS uses a per-user agent by default, and a daemon `.pkg` for server mode.
- No OIDC back-channel logout in v1.
- API keys expire after one year by default.
- The event replay window is 10 minutes or 50k events.
- An oapi-codegen spike runs before committing to it.

**Import**
- Calibre is a downstream target only.
- The book match threshold is 0.30, validated on fixtures.
- Auto-adding unknown artists or authors found on disk is a per-root option, off by default.
- Audiobookshelf and Stash writers come later.
- EPUB tag writing is opt-in.
- Missing-file grace period: 7 days.
- Hash verification of copies is on for network roots.
- Sonarr's "episode title required" behaviour is kept, with a timeout.
- Library import keeps existing paths.
- Comic transforms are opt-in.

**Library**
- A TPB/omnibus satisfying the singles it collects is opt-in.
- Requesters are notified on "complete", with "caught up" as an option.
- List retraction may delete files only for titles added solely by the list.
- Lists may use manual approval.
- Metadata profiles filter both membership and fetches where the provider supports it.
- Lidarr's "New" semantics become a fixed from-date.
- Alternate TV orderings come after v1.
- Disk auto-add is available for every kind, off by default.
- When slots share a root folder, a scanned file's slot is inferred by profile fit.
- `include_specials` is set per title and per request.

**Metadata and subtitles**
- IMDb and Rotten Tomatoes ratings are dropped in v1.
- One subtitle profile per item.
- On a media upgrade, downloaded subtitles are dropped and re-searched; uploaded and manual ones are re-synced.
- alass is the default sync tool.
- The licence of the subzero OCR dictionaries must be checked.
- Change-feed spikes (non-blocking).
- The project registers a Hardcover OAuth client.
- The plug-in ABI uses JSON in v1.

**Profiles**
- A profile deleted upstream while in use is detached to a local copy.
- Publishing offers direct push; a PR flow comes later.
- Spike go-git pushes from shallow clones.
- Separate ebook and audiobook score domains.
- `minimum_version` maps to a compatibility level.
- A Profilarr user-ops importer comes in phase 1 if cheap.
- A bundled Dictionarry snapshot enables an offline first boot.
- A Recyclarr translator comes later.
- Condition-level rename journaling comes later.

**Runtime**
- Plug-ins: spike Extism against a custom ABI; plug-ins are signed with admin-managed trust keys.
- Declarative provisioning comes later.
- Logs: a per-node ring buffer plus per-job logs.
- Completed jobs stay visible for 7 days.
- Measure the wazero timeout cost.
- Minimum Postgres version is 15; PgBouncer is supported in session mode only.
- `db copy` from SQLite to Postgres is a supported upgrade path.
