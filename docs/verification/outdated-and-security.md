# Verification: comparison.md §3 (outdated functionality and insecure defaults)

- Verified 2026-09-27 against `submodules/` at their checked-out commits (for example, Lidarr `develop` @ da7b4dfb1, 2026-09-13).
- Paths are relative to `submodules/`.
- For external facts, dates are the source's publication date where known; otherwise the access date (2026-09-27).

**Verdicts:**
- **CONFIRMED**: the claim holds as written.
- **CORRECTED**: the claim is partly wrong. This covers the wrong app list, the wrong count, or the external fact being wrong or overstated.
- **UNVERIFIABLE**: the claim could not be checked.

**Definition:** "six forks" means sonarr, radarr, lidarr, bookshelf, whisparr and whisparr-eros. That set is correct.

**Recurring correction:** Prowlarr is built from the same NzbDrone tree, so it carries almost every "forks" item. Most "forks" rows should read "six forks + Prowlarr".

## Summary

| Section | Items | CONFIRMED | CORRECTED | UNVERIFIABLE |
|---|---|---|---|---|
| 3.1 Dead services | 25 | 9 | 16 | 0 |
| 3.2 Legacy compat | 12 | 8 | 4 | 0 |
| 3.3 Platform history | 12 | 5 | 7 | 0 |
| 3.4 Insecure defaults | 9 | 7 | 2 | 0 |
| **Total** | **58** | **29** | **29** | **0** |

Most corrections widen or narrow the app list; the claim itself is sound. The corrections that change a conclusion are:

1. **uTorrent is not abandonware.** BitTorrent Inc. still ships Windows 3.6.0 builds. "Legacy, ad-supported, low priority" would be accurate; "dead" is not.
2. **Headphones is not dormant.** v0.6.4 was released on 2025-07-01 and the repo is not archived. It has low activity but is alive.
3. **Several "dead" Bazarr providers are alive, and most are not hidden from the UI.**
   - nekur, subsynchro and yifysubtitles sites are up; wizdom is unclear.
   - hosszupuska, nekur, subscenter, subsynchro, wizdom and yifysubtitles are all still listed in `bazarr/frontend/src/pages/Settings/Providers/list.tsx`.
4. **Things §3.1 implies are dead but are still alive:** Emby (4.10 stable, Sep 2026), Kometa (v2.5.1, 2026-09-24), BinSearch (up), Synology Download Station (4.1.0 with DSM 7.3), Prowl, Pushbullet (maintenance mode) and NZBVortex.
   - Only the *legacy code paths* for these are obsolete: Emby (Legacy) metadata, DS API v1 and so on. The integrations themselves should not be dropped wholesale.
5. **The current Twitter/X API has no free posting tier.** The v1.1 endpoints are gone, as claimed. However, since 2026-02-06 there is no free tier for new developers: POST /2/tweets is pay-per-use. So "drop" is reasonable, but not because posting is impossible.
6. **API keys in query strings:** the claim is wrong for Cleanuparr and Unpackerr.
   - Cleanuparr sends `x-api-key` in a header to the Arrs (`cleanuparr/code/backend/Cleanuparr.Infrastructure/Features/Arr/ArrClient.cs:425-428`). It uses the query string only for LazyLibrarian (`LazyLibrarianService.cs:278`).
   - Unpackerr uses `golift.io/starr` v1.4.1, which sets the `X-Api-Key` header (starr `http.go`: `req.Header.Set("X-Api-Key", c.APIKey)`).
   - Seerr (`seerr/server/api/servarr/base.ts:106`) and Bazarr (for example `bazarr/bazarr/sonarr/notify.py:13`) do put the key in the query string. This is confirmed.
7. **"Separate SQLite and Postgres migration trees in six apps" is wrong: it is three apps.**
   - The three are autobrr, Seerr and Cleanuparr.
   - The forks and Prowlarr use a single FluentMigrator tree.
8. **Lidarr's plugin loading needs a caveat.** Plugins are downloaded from GitHub at runtime but loaded only on restart.
   - The download has no signature or hash check.
   - `AssemblyLoadContext` is not a sandbox.
   - So the security conclusion stands.
9. **Additional finding: Profilarr claims to encrypt a key it stores in plaintext.** `ai_settings.api_key` is commented "Encrypted API key" (`profilarr/src/lib/server/db/schema.sql:395`), but there is no encryption code anywhere in `profilarr/src`.
10. **Missing from §3.4:**
    - Bazarr's auth is also off by default (`auth.type` default `None`, `bazarr/bazarr/app/config.py:186`).
    - Jackett's config DTO returns the first 10 characters of the password hash (`jackett/src/Jackett.Common/Models/DTO/ServerConfig.cs:83`).

---

## 3.1 Dead services, apps and protocols

| Item | Verdict | Code evidence | External evidence |
|---|---|---|---|
| TVRage ids / `rid` search (Sonarr, Prowlarr, Jackett) | CORRECTED | The three claimed apps are confirmed:<br>• `sonarr/src/NzbDrone.Core/Indexers/Newznab/NewznabRequestGenerator.cs:538,574`<br>• `prowlarr/.../Newznab/NewznabRequestGenerator.cs:148`<br>• `jackett/src/Jackett.Common/Indexers/Definitions/CardigannIndexer.cs:1435`<br>**Missed:** Bookshelf (`bookshelf/src/NzbDrone.Core/Indexers/IndexerCapabilities.cs:82,306`). | Went offline early Sep 2015, returned briefly, and went down for good around Aug 2016; there was never an official shutdown date. "2015" is roughly right. https://tvmaze.com/blogs/6/an-update-on-the-tvrage-situation (2015-09-21) |
| Torcache interceptor | CORRECTED | Also in **whisparr-eros** (`whisparr-eros/src/NzbDrone.Core/Http/TorcacheHttpInterceptor.cs`), so it is in all six forks + Prowlarr. | Dead since Aug 2016. CONFIRMED: https://torrentfreak.com/kat-shutdown-aftermath-itorrents-aims-to-replace-torcache-160802/ (2016-08-02) |
| EZRSS parser (Sonarr, Prowlarr, Lidarr) | CORRECTED | `Indexers/EzrssTorrentRssParser.cs` is in **all six forks + Prowlarr**. The claim missed Radarr, Bookshelf, Whisparr and Whisparr-eros. | Dead: ezrss.it now 301-redirects to an EZTV clone. Probe 2026-09-27; https://torrentfreak.com/uk-internet-providers-start-blocking-eztv-and-ezrss-130726/ |
| Fanzub (Sonarr, Whisparr) | CONFIRMED | Native indexer `*/src/NzbDrone.Core/Indexers/Fanzub/Fanzub.cs` in Sonarr and Whisparr. Elsewhere only remnants: Radarr mig 114 and Lidarr mig 015 remove it. | Closed 2015-03-01: https://archive.org/details/fanzub |
| CouchPotato / TorrentPotato (Radarr, Prowlarr, Jackett) | CONFIRMED | • Radarr: `Indexers/TorrentPotato/` and `ImportLists/CouchPotato/`<br>• Prowlarr: `Indexers/Definitions/TorrentPotato/`<br>• Jackett: `Controllers/ResultsController.cs` (potato API) and `Indexers/Definitions/Abstract/CouchPotatoTracker.cs` | Repo archived; last commit 2020-02-04; Python 2. https://github.com/CouchPotato/CouchPotatoServer (2026-09-27) |
| Headphones (Prowlarr, Lidarr) | CORRECTED | Code is confirmed: `prowlarr/.../Definitions/Headphones/`, `lidarr/.../Indexers/Headphones/` and `ImportLists/HeadphonesImport/`.<br>Lidarr also hard-codes a Headphones API key (`lidarr/src/NzbDrone.Core/Indexers/Headphones/HeadphonesSettings.cs:34`). | **Not dormant:** v0.6.4 released 2025-07-01, repo not archived. https://github.com/rembo10/headphones/releases (verified via `gh api` 2026-09-27). The Headphones VIP *indexer* service is a separate question and was not verified. |
| Goodreads API: 4 import lists, 2 notifications, hard-coded keys (Bookshelf) | CONFIRMED | • `bookshelf/src/NzbDrone.Core/ImportLists/Goodreads/` has Bookshelf, Lists, OwnedBooks and Series.<br>• `Notifications/Goodreads/` has Bookshelf and OwnedBooks.<br>• Keys are stored reversed: `MetadataSource/Goodreads/GoodreadsProxy.cs:33,77`.<br>• OAuth signing goes through `https://auth.servarr.com/v1/goodreads/sign` (`ImportLists/Goodreads/GoodreadsSettingsBase.cs:27`), a Servarr service that presumably died with Readarr. | No new keys from 2020-12-08; API retirement announced. https://www.goodreads.com/topic/show/21788520-api-deprecation (Dec 2020) |
| `api.bookinfo.club`, updates on `readarr.servarr.com` (Bookshelf) | CORRECTED | bookinfo.club has already been replaced: the default is now `https://api.bookinfo.pro` (`bookshelf/src/NzbDrone.Core/Configuration/ConfigService.cs:278`), community-run. `readarr.servarr.com/v1/` is still referenced (`NzbDrone.Common/Cloud/ReadarrCloudRequestBuilder.cs:15`). | api.bookinfo.club does not resolve (the domain lapsed around Aug 2024). https://wiki.servarr.com/readarr/metadata-issues |
| Readarr as sync target (Prowlarr, autobrr, Cleanuparr, Unpackerr) | CONFIRMED | • `prowlarr/.../Applications/Readarr/`<br>• `autobrr/internal/downloader/*.go`<br>• `cleanuparr/.../Cleanuparr.Domain/Enums/InstanceType.cs`<br>• `unpackerr/pkg/unpackerr/starrpoll.go`<br>Profilarr mentions `readarr` only in the instance-type comment (`schema.sql:29`). | Repo archived 2025-06-27 with a retirement announcement. https://github.com/Readarr/Readarr (verified via `gh api`: `archived: true`) |
| 11 `[Obsolete]` Prowlarr indexers, UNIT3D, `IndexerDefinitionVersion` | CONFIRMED | Exactly 11: Animedia, BinSearch, CGPeers, FunFile, HDSpace, PornoLab, Rarbg, SceneTime, Shizaproject, Xthor, ZonaQ (for example `Rarbg/Rarbg.cs:23`). UNIT3D and `IndexerDefinitionVersion` are present (`Migration/015_indexer_versions.cs`). | RARBG shut down 2023-05-31: https://torrentfreak.com/iconic-torrent-site-rarbg-shuts-down-all-content-releases-stop-230531/. Note that **BinSearch the site is still up**; only the native Prowlarr indexer is obsolete. https://www.usenet.com/binsearch/ (2026) |
| AHD flags, PTP old-settings check (Radarr) | CORRECTED | Radarr is confirmed: `Parser/Model/IndexerFlags.cs:40,59` and `HealthCheck/Checks/PTPOldSettingsCheck.cs`.<br>**Missed:** whisparr-eros also has the AHD flags. | AwesomeHD closed around 2021-02-22: https://github.com/Jackett/Jackett/issues/11046 |
| Hadouken, Vuze, Pneumatic (six forks) | CORRECTED | In all six forks **+ Prowlarr** (`Download/Clients/{Hadouken,Vuze,Pneumatic}`). | • Hadouken: last release v5.2, Aug 2015.<br>• Vuze: last release 5.7.6.0, 2017-11-02; successor BiglyBT (https://en.wikipedia.org/wiki/Vuze).<br>• Pneumatic: Python-2 Kodi add-on, cannot run on Kodi 19+ (https://iwf1.com/kodi/plugin.program.pneumatic).<br>All confirmed dead. |
| uTorrent (forks + Cleanuparr) | CORRECTED | Also in **Prowlarr**. Cleanuparr confirmed (`Cleanuparr.Domain/Enums/DownloadClientTypeName.cs`). | **Not abandonware.** Windows Classic 3.6.0 builds were still being released as of Dec 2025, and uTorrent Web is distributed. https://www.techspot.com/downloads/1597-utorrent.html (accessed 2026-09-27). It is legacy and ad-supported but alive, and users still run it. |
| Legacy client APIs: NZBGet v12/13, qBit API v1, Synology DS v1, Deluge v1 | CORRECTED | In all six forks **+ Prowlarr**:<br>• NZBGet: a `< 12.0` check (for example `sonarr/.../Nzbget/Nzbget.cs:279`) and `HasVersion(13)` in `NzbgetProxy.cs`.<br>• `QBittorrentProxyV1.cs`<br>• `DownloadStationTaskProxyV1.cs`<br>• `DelugeProxy.cs:56-61`<br>autobrr has `DownloaderTypeDelugeV1` (`autobrr/internal/downloader/connection.go:38,107`). | • NZBGet v12 dates from Jan 2014 and v13 from Jul 2014. nzbget/nzbget was archived 2024-01-05; nzbget-ng handed over to **nzbgetcom**, which is active (v26.3, 2026-08-27). https://github.com/nzbgetcom/nzbget/releases<br>• qBittorrent 4.1.0 (2018-05-04) introduced API v2; 4.2.0 (2019-12-03) dropped legacy WebAPI. https://raw.githubusercontent.com/qbittorrent/qBittorrent/master/Changelog<br>• Deluge 2.0 released Jun 2019; 2.2.0 released 2025-04-28. https://pypi.org/project/deluge/<br>• Synology Download Station itself is **alive** (4.1.0 with DSM 7.3, Oct 2025); only DS API v1 is legacy. https://community.synology.com/enu/forum/1/post/195400 |
| Twitter notification + OAuth1 (forks) | CORRECTED | In the six forks **+ Prowlarr** (`Notifications/Twitter`, `NzbDrone.Common/OAuth`). | • v1.1 `direct_messages/new` was retired 2018-09-17.<br>• v1.1 `statuses/update` is unavailable to new or free apps.<br>• Since 2026-02-06 there is no free tier for new developers; POST /2/tweets is pay-per-use ($0.015/post). https://docs.x.com/x-api/getting-started/pricing (accessed 2026-09-27)<br>"Gone" is right for the v1.1 code; posting to X is still possible via v2. |
| LunaSea agent (Seerr removed, autobrr) | CONFIRMED | • Seerr only has `seerr/server/lib/settings/migrations/0006_remove_lunasea.ts`.<br>• autobrr has `web/src/forms/settings/NotificationForms.tsx` and related files. | Repo archived 2025-04-03 (final v11.0.0). https://github.com/JagandeepBrar/lunasea |
| Roksbox, WDTV; Emby (Legacy), Kometa (Sonarr, Radarr, Lidarr) | CORRECTED | • Roksbox and WDTV are in Sonarr, Radarr, Lidarr **+ Whisparr + Whisparr-eros** (`Extras/Metadata/Consumers/`).<br>• Emby (Legacy) (`MediaBrowserMetadata.cs:29`) and Kometa are **only in Radarr and Whisparr-eros**, not Sonarr or Lidarr. | • WD TV was discontinued Aug 2016 (https://en.wikipedia.org/wiki/WD_TV).<br>• Roksbox is unsupported, but a successor Roku channel exists.<br>• **Emby and Kometa are alive.** Emby: 4.10.0.40 stable, Sep 2026 (https://github.com/MediaBrowser/Emby.Releases/releases). Kometa: v2.5.1, 2026-09-24 (https://github.com/Kometa-Team/Kometa).<br>Only the self-declared-deprecated *metadata writers* are obsolete. |
| Plex legacy models and agents (Sonarr, Radarr, Seerr) | CORRECTED | The `*Legacy` models (`Notifications/Plex/Server/PlexSection.cs`, `PlexSectionItem.cs`, `PlexPreferences.cs`) are in **all six forks**. Seerr is confirmed (`seerr/server/lib/scanners/plex/index.ts:25-37`). | Legacy agents were removed from new-library creation from PMS 1.41.7 (2025-04-23). https://forums.plex.tv/t/legacy-agents-removed-already-in-pms-1-41-7-9717-2025-04-23/914518 |
| Jackett "/all" check (Sonarr, Radarr, Lidarr) | CORRECTED | `HealthCheck/Checks/IndexerJackettAllCheck.cs` is in **all six forks**. | n/a |
| CloudFlare captcha interceptor + UI (Sonarr, Radarr) | CORRECTED | `NzbDrone.Core/Http/CloudFlare/` and the captcha UI are in **all six forks + Prowlarr**. | n/a (FlareSolverr is the current approach) |
| Socks4 (forks) | CORRECTED | In the six forks + Prowlarr, **and Jackett** (`jackett/src/Jackett.Common/Utils/Clients/WebClient.cs:63`). | It is a protocol, so there is no EOL date. It is legacy but still widely supported. "Dead" overstates it. |
| Dead Bazarr subtitle providers, "hidden from the UI" | CORRECTED | **Not hidden.** hosszupuska, nekur, subscenter, subsynchro, wizdom and yifysubtitles are still in `bazarr/frontend/src/pages/Settings/Providers/list.tsx` (for example line 287, hosszupuska).<br>shooter, argenteam, legendastv and thesubdb survive only in the backend or `custom_libs` (for example thesubdb in `custom_libs/subliminal/extensions.py`).<br>`PROVIDERS_FORCED_OFF` (`bazarr/bazarr/app/get_providers.py:143`) concerns forced-subtitle support, not dead sites, and its enforcement is commented out (around line 217). | • **Dead:** legendastv (removed 2022-12, #2012), argenteam (removed 2024-01, #2352), thesubdb (domain does not resolve), subscenter (parked domain), hosszupuska (does not resolve), shooter (API gone).<br>• **Alive:** nekur, subsynchro and yifysubtitles.<br>• **Unclear:** wizdom.<br>Source: https://github.com/morpheus65535/bazarr commit history, and site probes on 2026-09-27. |
| Kapowarr hard-coded trackers, torrentgalaxy avoid-list | CONFIRMED | `kapowarr/backend/implementations/download_preppers/ddl/GetComics.py:405` (trackers) and `:98` (avoid-list). | Tracker liveness was not individually checked. |
| autodl-irssi importer (autobrr) | CONFIRMED | `autobrr/web/src/screens/filters/_configParser.ts`, `Importer.tsx` | Last commit and release 2.6.2 on 2020-03-11; not formally archived. https://github.com/autodl-community/autodl-irssi |
| Magnetico native (Jackett) | CONFIRMED | `jackett/src/Jackett.Common/Indexers/Definitions/Magnetico.cs` | Archived 2022-01-20. https://github.com/boramalper/magnetico |

## 3.2 Legacy compatibility paths

| Claim | Verdict | Evidence |
|---|---|---|
| Bazarr: about 20 `is_legacy` branches + classic SignalR | CORRECTED (count) | There are 15 `is_legacy` occurrences in `bazarr/bazarr` (about 13 conditionals), not about 20. The classic SignalR client is confirmed: `bazarr/bazarr/app/signalr_client.py:9` (`from signalr import Connection`). |
| Seerr: language profiles, `profileId` | CONFIRMED | `languageProfile` appears in about 66 files. |
| Recyclarr `:3` naming keys | CORRECTED | The suffix is `:4`: `recyclarr/src/Recyclarr.Cli/Pipelines/Plan/Components/SonarrMediaNamingPlanComponent.cs:44` (`keySuffix = ":4"`). |
| Prowlarr "Sonarr v2" branch | CORRECTED | It is only a validation error string (`prowlarr/src/NzbDrone.Core/Applications/Sonarr/Sonarr.cs:73`), not a code branch. |
| Sonarr v3 API, `ObsoleteQueueService`, language-profile stubs in Sonarr + Whisparr | CONFIRMED | • `Sonarr.Api.V3` sits next to V5, with 9 `[Obsolete]` files.<br>• `NzbDrone.Core/Queue/ObsoleteQueueService.cs` exists.<br>• The stubs are in `Sonarr.Api.V3/Profiles/Languages/` and `Whisparr.Api.V3/Profiles/Languages/`. Whisparr-eros does not have them. |
| Profilarr v1-era language types, `on_change`, `should_sync`, legacy `jobs` | CONFIRMED | • `profilarr/src/lib/server/pcd/entities/qualityProfiles/list.ts:136`<br>• `schema.sql:444-446` (`should_sync` marked deprecated)<br>• `jobs`/`job_runs` at `schema.sql:80,106` coexist with `job_queue` (mig 049). |
| Recyclarr `state repair`, `migrate` | CONFIRMED | `recyclarr/src/Recyclarr.Cli/Console/CliSetup.cs:17,68` |
| Recyclarr v4–v7 config keys | CORRECTED | The deprecations point to the v5.0, v8.0 and v9.0 upgrade guides (`ConfigDeprecatedPropertyBehavior.cs:15-23`, `FeatureRemovalChecker.cs:21`). "v4–v7" is inaccurate. |
| Seerr Overseerr/Jellyseerr merge machinery | CONFIRMED | • `seerr/server/lib/overseerrMerge.ts`<br>• `JELLYFIN_TYPE` in `settings/migrations/0003_emby_media_server_type.ts:8`<br>• `jellyfin*` columns in `server/entity/Media.ts` |
| Jackett .NET Framework 4.7.1 / Mono, ASP.NET Core 2.x | CONFIRMED | There are four `net9.0;net471` multi-target projects; the ASP.NET Core packages are pinned at 2.3.x. |
| Radarr `EpisodeFormat`; Lidarr TV regexes + mig 067 | CONFIRMED | • `radarr/src/NzbDrone.Core/Organizer/AbsoluteEpisodeFormat.cs`<br>• `lidarr/.../Migration/067_download_clients_rename_tv_to_music.cs`<br>• `lidarr/.../Parser/Parser.cs:206,219` |
| Whisparr anime/scene/season code | CONFIRMED | Anime appears in 7 files, scene mapping in 5, and SeasonNumber in 67. Bookshelf's Lidarr music naming was not separately re-checked; the bookshelf.md analysis covers it. |

## 3.3 Platform history

| Claim | Verdict | Evidence |
|---|---|---|
| Windows tray, service installer, firewall COM, `RegisterUrl`, Inno Setup (six forks + Jackett) | CORRECTED | Confirmed in the six forks and Jackett (`Jackett.Tray`, `Jackett.Service`, `.iss`). **Prowlarr also has all of them.** |
| Unpackerr desktop tray + zenity | CONFIRMED | `unpackerr/go.mod`: energye/systray v1.0.3 and ncruces/zenity v0.10.15, used in `pkg/ui/`. |
| `NzbDrone.Mono` + Mono.Posix | CORRECTED | `NzbDrone.Mono` is in the six forks + Prowlarr. Mono.Posix is in Bookshelf, Lidarr, Prowlarr, Radarr, Whisparr-eros **and Jackett**. Sonarr and Whisparr use `Openur.Mono.Unix` instead (`sonarr/src/NzbDrone.Mono/Sonarr.Mono.csproj:7`). |
| Sonarr Harmony runtime patches | CORRECTED | They are in **Sonarr and Whisparr** (`*/src/*.RuntimePatches/`). |
| `LongPathSupport` | CONFIRMED | `NzbDrone.Common/Disk/LongPathSupport.cs`, in the six forks + Prowlarr. |
| Legacy AppData migration | CORRECTED | It is in Sonarr, Radarr, Lidarr, Prowlarr, Whisparr and Whisparr-eros (`AppFolderFactory.cs`, for example `sonarr/...:74-143`). **Bookshelf does not have it.** |
| Self-updaters (forks, Bazarr, Jackett, Unpackerr) | CONFIRMED | • Bazarr: `bazarr/bazarr/app/check_update.py` (ZipFile over the install)<br>• Jackett: `jackett/src/Jackett.Updater`<br>• Unpackerr: `unpackerr/pkg/update/check.go`, `pkg/unpackerr/restart.go` |
| Bazarr `future`/`six`/`past` + js2py "115k lines" | CORRECTED (attribution) | js2py alone is 114,607 lines. future, past, six and libfuturize/libpasteurize add about 43.7k more, for about 158k in total.<br>js2py is unmaintained: last release 0.74 on 2022-11-06, and CVE-2024-28397 (sandbox escape) has no upstream fix. https://pypi.org/project/Js2Py/ ; https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape |
| Kapowarr Python 3.8 shims | CONFIRMED | `kapowarr/pyproject.toml:10` (`requires-python = ">= 3.8"`); `backend/base/definitions.py:29` (`MIN_PYTHON_VERSION = (3, 8, 0)`). Python 3.8 reached EOL on 2024-10-07: https://endoflife.date/python |
| .NET 6 (Bookshelf) | CONFIRMED | net6.0 is used across 24 Bookshelf projects. For comparison, Radarr, Lidarr and Prowlarr are on net8.0; Sonarr, Whisparr and Whisparr-eros are on net10.0. .NET 6 reached EOL on 2024-11-12: https://devblogs.microsoft.com/dotnet/dotnet-6-end-of-support/ |
| RestSharp 106, FluentValidation 9, retry-go v3+v4, `pkg/errors` | CORRECTED (scope) | RestSharp 106.15.0 and FluentValidation 9.5.4 are in Sonarr, Radarr, Lidarr, Bookshelf, Whisparr and Prowlarr; **whisparr-eros has already upgraded** to RestSharp 114 and FV 12.1.1. autobrr is confirmed: `go.mod:22-23` (retry-go v3.0.0+incompatible and v4.7.0) and `go.mod:44` (pkg/errors v0.9.1). |
| Separate SQLite and Postgres migration trees "in six apps" | CORRECTED | There are **three**: `autobrr/internal/database/migrations/{sqlite,postgres}`, `seerr/server/migration/{sqlite,postgres}` and `Cleanuparr.Persistence.{Sqlite,Postgres}/Migrations`. The forks and Prowlarr use a single FluentMigrator tree. |

## 3.4 Insecure defaults and anti-patterns

| Claim | Verdict | Evidence |
|---|---|---|
| **Kapowarr** auth off by default; `POST /api/auth` returns the API key when no password is set | CONFIRMED | `auth_password` defaults to `''` (`kapowarr/backend/internals/settings.py:73`). In `api_auth()` (`kapowarr/frontend/api.py:223-251`), the password check is skipped when `settings.auth_password` is empty, and the function returns `{'api_key': settings.api_key}` to any caller. The login password itself is PBKDF2-SHA256 hashed with 100k iterations and a salt (`backend/base/helpers.py:286-298`), so it is not weak. |
| **Jackett** open admin without password | CONFIRMED | `CheckAuthorised` returns `true` when `AdminPassword` is empty (`jackett/src/Jackett.Server/Services/SecurityService.cs:17-18`). `UIController.Login` then auto-signs-in (`jackett/src/Jackett.Server/Controllers/UIController.cs:39-43`), and the admin UI exposes the API key.<br>Additional finding: the config DTO sends the first 10 characters of the password hash to the client (`jackett/src/Jackett.Common/Models/DTO/ServerConfig.cs:83`). |
| Weak hashing: MD5 (Bazarr), SHA-512 (Jackett) | CONFIRMED | • Bazarr uses unsalted MD5 at set time (`bazarr/bazarr/app/config.py:768-770`) and check time (`bazarr/bazarr/utilities/helper.py:23`).<br>• Jackett uses a single SHA-512 round with the API key appended as a "salt" (`SecurityService.cs:26-38`). Because that salt is the API key, anyone holding the key can brute-force the password offline. |
| API keys in query strings: Seerr, Bazarr, Kapowarr, Cleanuparr, Unpackerr; the Arrs accept it | CORRECTED | • **Seerr:** confirmed; `params: { apikey }` (`seerr/server/api/servarr/base.ts:104-107`).<br>• **Bazarr:** confirmed; for example `bazarr/bazarr/sonarr/notify.py:13` and `rootfolder.py:19`.<br>• **Kapowarr:** confirmed; its own API reads `api_key` from `request.values`, i.e. query or form (`kapowarr/frontend/api.py:101,199`).<br>• **Cleanuparr: wrong.** It sends a header to the Arrs (`ArrClient.cs:427`); only LazyLibrarian uses the query string (`LazyLibrarianService.cs:278`).<br>• **Unpackerr: wrong.** It uses a header via starr v1.4.1 (`http.go`, `X-Api-Key`).<br>• **The Arrs accept it:** confirmed; `options.QueryName = "apikey"` / `"access_token"` (`sonarr/src/Sonarr.Http/Authentication/AuthenticationBuilderExtensions.cs:50,55`, read at `ApiKeyAuthenticationHandler.cs:40`). |
| TLS verification disabled (Bazarr) | CONFIRMED | There are 28 `verify=False` calls in `bazarr/bazarr/**` covering every Sonarr/Radarr call, Plex OAuth and the UI image proxy. Examples: `bazarr/bazarr/app/ui.py:144`, `bazarr/bazarr/api/plex/oauth.py:226`, `bazarr/bazarr/radarr/info.py:34`. |
| Telemetry on by default with hard-coded secrets: Bazarr GA4; Sentry DSNs in forks | CORRECTED (scope) | • **Bazarr:** confirmed. `analytics.enabled` defaults to `True` (`bazarr/bazarr/app/config.py:400`), and `GtagMP(api_secret="qHRaseheRsic6-h2I_rIAA", measurement_id="G-3820T18GE3")` is hard-coded (`bazarr/bazarr/utilities/analytics.py:28`).<br>• **Sentry DSNs** are hard-coded in `NzbDrone.Common/Instrumentation/NzbDroneLogger.cs` of Sonarr (:76), Radarr (:75), Lidarr (:75), Whisparr (:72), Whisparr-eros (:75, reusing Radarr's DSN) **and Prowlarr** (:75).<br>• **Bookshelf has no Sentry DSN**: it was removed. |
| Hard-coded third-party keys: TMDb/TVDB and RT Algolia (Seerr), AcoustID/Last.fm/Spotify (Lidarr), Goodreads (Bookshelf) | CONFIRMED (+1) | • Seerr: TMDb `seerr/server/api/themoviedb/index.ts:188`; TVDB `seerr/server/api/tvdb/index.ts:114`; Algolia `seerr/server/api/rating/rottentomatoes.ts:110`.<br>• Lidarr: AcoustID `lidarr/src/NzbDrone.Core/Parser/FingerprintingService.cs:36`; Last.fm `ImportLists/LastFm/LastFmUserSettings.cs:23`; Spotify ClientId `ImportLists/Spotify/SpotifySettingsBase.cs:35`. **Missed:** a Headphones API key (`Indexers/Headphones/HeadphonesSettings.cs:34`).<br>• Bookshelf: Goodreads keys (reversed strings) at `MetadataSource/Goodreads/GoodreadsProxy.cs:33,77`. |
| Lidarr loads arbitrary .NET plugin assemblies from GitHub at runtime, no sandbox | CONFIRMED (with nuance) | • `PluginService.cs:52` queries `api.github.com/repos/{owner}/{name}/releases` for any user-supplied GitHub URL.<br>• `InstallPluginService.cs:73-78` downloads and extracts the zip with no signature, hash or allow-list check, then asks the user to "restart Lidarr".<br>• `PluginLoader.cs:45` / `PluginLoadContext.cs` load the assemblies in-process via a collectible `AssemblyLoadContext`, which gives isolation of versions only, not of security.<br>So the download happens at runtime, the load happens at startup, and there is no sandbox. |
| Secrets stored in plaintext: Profilarr, Kapowarr, most others | CONFIRMED (with nuance) | • **Profilarr:** Arr API keys (`schema.sql:34`) and the Git PAT (`:256`) are plaintext. The AI key (`:395`) is commented "Encrypted" but no encryption code exists in `profilarr/src`, so it is also plaintext. Profilarr's own API keys *are* bcrypt-hashed (`schema.sql:957-966`).<br>• **Kapowarr:** third-party credentials are plaintext (`credentials.password/api_key` and `external_download_clients.password/api_token` in `kapowarr/backend/internals/db.py:476-561`). Its login password is PBKDF2-hashed (see above). |

### Items missing from §3.4

- **Bazarr auth off by default:** `auth.type` default is `None` (`bazarr/bazarr/app/config.py:186`).
- **Jackett leaks a partial password hash** to the UI (`ServerConfig.cs:83`).
- **js2py CVE-2024-28397:** Bazarr bundles an unpatched, unmaintained JavaScript sandbox with a known escape (see §3.3).
- **Lidarr's Headphones key** is hard-coded in source.
