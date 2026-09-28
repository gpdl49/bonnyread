# Unpackerr — Codebase Analysis

Submodule: `submodules/unpackerr` (HEAD `c88fb41`, 2026-09-25, "v1.0.0 (September 2026)" per `INTERNALS.md:28`). Paths are relative to the submodule unless prefixed `xtractr/` or `starr/`. Those two are golift libraries cloned at HEAD for this review; the submodule pins `golift.io/xtractr v0.6.2-0.20260922231041-fec876ff1836` and `golift.io/starr v1.4.1` (`go.mod`).

**License:** MIT, confirmed.
- Unpackerr: `LICENSE`, "Copyright (c) 2018-2022 David Newhall II, 2023-2026 Go Lift".
- xtractr: `LICENSE`, "Copyright (c) 2020-2026 Go Lift".
- starr: MIT (golift).
- This is the only project in the set that is not GPL or AGPL. Its code and libraries can be vendored or imported into a closed or any-licensed monolith, provided the notice is kept.

---

## 1. Overview

**Purpose:** a daemon that finds completed downloads (by polling *arr queues or by watching folders), extracts the archives in them (RAR, 7z, zip, …) so the *arr can import the payload, and then deletes the extracted copy after import. Deleting only the extracted copy keeps the original RARs seeding.

| Item | Value |
|---|---|
| Language | Go, `go 1.27.0` (`go.mod`). Builds with `CGO_ENABLED=0` (`Dockerfile`). |
| Frontend | Svelte 5 + Vite 8 + TypeScript + Bootstrap/sveltestrap, embedded into the binary (`frontend/frontend.go`, `frontend/package.json`). About 8.9k lines of `.svelte` and 2.7k lines of `.ts`. New in v1.0. |
| Backend LOC | About 20.4k non-test Go and 17.4k test Go. |
| Persistence | No database. Config is TOML (with env overlay). Extraction history is an append-only JSONL file (`pkg/unpackerr/historyfile.go`). |
| Process model | A single goroutine, `(*Unpackerr).Run()` in `pkg/unpackerr/start.go:276`, owns all state. HTTP handlers hand mutations to it through `onMainLoop` (`pkg/unpackerr/mainloop.go`; see `INTERNALS.md` "Process model"). |
| HTTP | stdlib `http.ServeMux` on `:5656`. OpenAPI spec is committed at `pkg/unpackerr/openapi.json` (1,096 lines). Live updates go over a WebSocket (`pkg/unpackerr/livews.go`, `livehub.go`, `coder/websocket`). |

**Package layout (non-test LOC)**

| Package | LOC | Role |
|---|---|---|
| `pkg/unpackerr` | 13,498 | Daemon covering Starr polling, the state machine, the folder glue, the web API/auth, history, metrics, tray, restart and logs. |
| `pkg/configdef` | 3,188 | Generates example conf, compose, env docs and Docusaurus pages, and writes TOML atomically, all from `definitions.yml` (1,267 lines). |
| `pkg/hooks` | 1,751 | Webhook and command-hook delivery, built-in templates, and the Discord/Telegram message-edit feature. |
| `pkg/folders` | 900 | Watch-folder config, fsnotify, and a per-folder polling watcher (`radovskyb/watcher`). |
| `pkg/extract` | 332 | Shared `Status` enum and the `Extract` item type. |
| `pkg/ui`, `pkg/bindata`, `pkg/update` | ~520 | Tray menu, icons, and the GitHub release check. |
| `frontend/` | Go 122 + SPA | Embedded dashboard. |

**Key dependencies** (`go.mod`):
- `golift.io/xtractr` does the extraction.
- `golift.io/starr` is the *arr API client.
- `golift.io/cnfg` / `cnfgfile` handle env and `filepath:` config.
- `golift.io/rotatorr` handles log rotation.
- `prometheus/client_golang`, `fsnotify`, `radovskyb/watcher`, `BurntSushi/toml`, `gorilla/securecookie`, `x/crypto` (bcrypt/pbkdf2), `energye/systray` + `ncruces/zenity` (desktop GUI), `spf13/pflag`.

---

## 2. Domain model

The model is small: there are no entities beyond the tracked extraction and the config.

**`extract.Extract`** (`pkg/extract/extract.go:13`) is the tracked item. It lives in memory in `History.Map map[string]*Extract`, keyed by **download title** (`pkg/unpackerr/history.go`).

| Field group | Fields |
|---|---|
| Identity and routing | `App` (Sonarr/Radarr/Lidarr/Readarr/"Folder"), `Name` (instance label), `URL` (instance), `IDs` (map copied from the queue record: `downloadId`, `seriesId`/`episodeId`, `movieId`, `artistId`/`albumId`, `authorId`/`bookId`, `title`, `reason`). |
| Paths | `Path` (local path resolved on this host), `OutputPath` (the *arr's view, which may be UNC; used for the Lidarr ManualImport). |
| Lifecycle | `Status`, `Updated`, `Due`/`DueKind` (next timer), `Retries`, `NoRetry`, `Note` (e.g. "no extractable files", "waiting for syncthing"). |
| Policy (copied from the instance) | `DeleteOrig`, `DeleteDelay`, `Syncthing`, `SplitFlac`, `MaxBytes`. |
| Result | `Resp *xtractr.Response` (archives, extras, `NewFiles`, size, error), `XProg` (progress), `PreFiles` (pre-extract snapshot used to detect remnants). |
| Hooks | `HookFail`, `HookMessages` (slug → Discord/Telegram message id, used to edit the message in place). |

**Status enum** (`pkg/extract/status.go:15`). The order matters because the code uses `>`/`<` comparisons such as `Status > IMPORTED`.

```
0 WAITING          "Waiting, pre-Queue"        (seen complete in *arr queue; start_delay not elapsed / no archives yet)
1 QUEUED           queued in xtractr
2 EXTRACTING
3 EXTRACTFAILED    retried after retry_delay up to max_retries (default 2 → 3 attempts)
4 EXTRACTED        "Extracted, Awaiting Import"
5 IMPORTED         item vanished from *arr queue → delete_delay countdown
6 DELETING
7 DELETEFAILED     (unused)
8 DELETED          removed from Map after another delete_delay
9 EXTRACTEDNOTHING (folders)
```

Numeric ids are part of the public contract. Webhook `events = [0,3,4,…]` and `UN_WEBHOOK_*_EVENTS` accept numbers (`pkg/hooks/config.go:77`). `events=[0]` alone means "all" (`HasEvent`, `pkg/hooks/config.go:168`).

**Transitions for Starr items** (`pkg/unpackerr/handlers.go`)
- *arr queue record `status=="completed"` and its protocol is in `protocols` → new `WAITING` (`checkStarrQueue`, `starrpoll.go:152`).
- `WAITING` → `QUEUED` once `start_delay` has elapsed and archives are found (`extractCompletedDownload`, `handlers.go:153`).
- `QUEUED` → `EXTRACTING` → `EXTRACTED` / `EXTRACTFAILED` (xtractr callback, `handleXtractrCallback`, `handlers.go:357`).
- If the title leaves the *arr queue while `EXTRACTED` → `IMPORTED` (`checkQueueChanges`, `handlers.go:66`).
- A `WAITING` item that leaves the queue is dropped; it was never extracted.
- An item that comes back into the queue after `IMPORTED` is reset to `EXTRACTED`. One that comes back after `DELETED` is reset to `WAITING`.
- `IMPORTED` + `delete_delay` → queue a delete of `NewFiles` (or of the original path if `delete_orig`) → `DELETED` → removed from the Map after another `delete_delay` (`checkExtractDone`, `handlers.go:264`).
- Safety net: any item stuck in `QUEUED`/`EXTRACTING`/`EXTRACTED` for 24h is marked `DELETED` (`staleItemTimeout`, `start.go:43`).
- A "queue snapshot ready" guard stops an item from being treated as imported before the first successful poll in this process (`queueSnapshotReady`, `starrpoll.go:212`).

**Folder items**: `folders.Folder` (`pkg/folders/config.go:89`) is tracked in `u.folders.Folders` and mirrored into `Map` with `App="Folder"`. It carries the same statuses plus `WaitFile` (for `.part` files etc.), `Files`, `Archives`, `PreFiles`.

**Durable history**: `HistoryRecord` rows are written to `unpackerr.history.jsonl`, one line per transition, compacted at `2×keep_history`. On startup, rows from the last 72h are restored into `Map` so an in-progress import-wait or delete-delay survives a crash (`historyrestore.go`; `INTERNALS.md` "History JSONL"). "Forget" writes tombstones so a forgotten title is not re-queued while it is still in the *arr queue.

---

## 3. Feature areas

### Discovery
1. **Polling *arr queues.** This is the primary mode.
   - Every `interval` (default 2m, minimum 15s; `start.go:40-41`), each instance is fetched concurrently on a worker pool sized one per instance (`apps.go:147-187`).
   - Only records with `status == "completed"` **and** a protocol listed in `protocols` qualify (`isComplete`, `starrpoll.go:326`).
   - The default protocol list is `torrent,TorrentDownloadProtocol` (`apps.go:28`). **Usenet is ignored by default**, because SABnzbd and NZBGet unpack natively.
   - The download path is found by checking `paths[]/title`, then `paths[]/basename(outputPath)` for cross-platform UNC paths, and finally `outputPath` itself (`getDownloadPath`, `handlers.go:452`).
2. **Watch folders** (`[folder.<slug>]`), independent of any *arr.
   - Uses fsnotify, with an optional per-folder polling watcher `interval` for NFS/SMB (`pkg/folders/watch.go`).
   - Tracks top-level items in the watch path.
   - `wait_extensions` (e.g. `.part`, `.!qB`) block extraction while a download is incomplete (`pkg/folders/wait.go`).
   - Options: `extract_path`, `move_back`, `delete_after` (default 10m), `delete_files`, `delete_original`, `extract_isos`, `exclude_paths`, `skip_empty`, and limits (`pkg/folders/config.go:24`).
   - Skips its own `_unpackerred` output and folders that contain an extraction log (`watch.go:399-481`).
3. **No inbound webhooks from the *arr apps.** Unpackerr does not register as a Sonarr/Radarr Connect target. Discovery is poll-only for Starr apps.

### Extraction
Extraction is fully delegated to xtractr (`handlers.go:192`).
- **Starr archive types considered:** `.rar .r00 .zip .7z .7z.001 .gz .tgz .tar .tar.gz .bz2 .tbz2`, plus `.cue` when Lidarr has `split_flac` set (`handlers.go:192`).
- **Folder archive types:** everything xtractr supports, minus exclusions (`folder.go:160`).
- **Formats xtractr supports** (`xtractr/files.go` `extension2function`): rar/r00 (multi-part), 7z/7z.001, zip (with non-UTF8 filename detection), tar, tar.gz/tgz, tar.bz2/tbz/tbz2, tar.xz/txz, tar.z/tz, tlz, gz/gzip, bz2, xz, lz/lzip, lzma/lzma2, zst/zstd, z (LZW), zlib/zz, lz4, snappy/sz, s2, br/brotli, ar/deb, cpio/cpio.gz/cpgz, rpm, iso (ISO9660 + UDF), and cue/cue.txt (FLAC+CUE splitting, including the APE path `ape.go`). Content-based magic detection lives in `xtractr/magic.go`.
- **RAR:** pure Go via `nwaples/rardecode/v2` (`xtractr/rar.go:63`), **no `unrar` binary**. Handles multi-volume archives, and `FindCompressedFiles` picks only the first volume (`.partN`/`.r00` heuristics, `xtractr/find.go:67,139`).
- **7z:** `bodgit/sevenzip`.
- **Nested archives:** after the first pass, xtractr extracts "extras" (archives found inside the output). For Starr items this is capped at `MaxNested=8` and `ExtrasMaxDepth=3` (`start.go:28-36`). ISO recursion needs `RecurseISO`.
- **Passwords** (RAR and 7z only):
  - The global `passwords[]` list is tried in order. It supports `filepath:/secrets` with one password per line.
  - A password can also be embedded in the download path as `…{{password}}…` (`getPasswordFromPath`, `handlers.go:245`).
  - Password tries are handled in `xtractr/rar.go:17`, `7z.go:15`.
- **Zip-bomb and disk safety limits** (per top-level archive, `xtractr/queue.go:55-78`):
  - `MaxBytes` defaults are Sonarr 20GB, Radarr 75GB, Lidarr 4GB, Readarr 1GB (`start.go:31-34`, overridable with `max_bytes`).
  - `MaxFiles` 1000 and `MaxRatio` 5.0 for Starr items. Folders default to unlimited.
  - A limit error sets `NoRetry` (`handlers.go:423`).
  - **There is no free-disk-space check**: no statfs or ENOSPC handling in either repo.
- **Output layout:** extraction goes to the sibling temp folder `Path + "_unpackerred"`, then the files are moved back into the download folder (`TempFolder:false`, `start.go:54`, `xtractr/queue.go:172`). The *arr therefore never sees half-written files. Files that would overwrite an existing file are "Refused" and kept.
- **Remnants:** if an interrupted earlier run left partial files, `remnant_action` = `rename` (default, `.remnant` suffix), `delete` or `off` handles them before a retry (`pkg/unpackerr/remnants.go`). This uses the `PreFiles` snapshot.
- **Syncthing:** with `syncthing = true`, extraction waits while any `.tmp` file is present (`handlers.go:381`).
- **Lidarr FLAC+CUE split:** after extraction, it calls Lidarr `GET /api/v1/manualimport` then `POST /api/v1/command` (ManualImport) for the split tracks (`lidarr.go:163`).
- **Concurrency:** `parallel` (default 1) concurrent extractions. xtractr also has `FileWorkers` for zip/7z, which Unpackerr does not use.
- **Other settings:** `file_mode` / `dir_mode`.

### Cleanup and deletion timing
- **Starr items:** a delete runs `delete_delay` (default 5m, minimum 1s; negative disables) after the title leaves the *arr queue. By default it deletes only the extracted `NewFiles` and purges empty parents up to the download root. With `delete_orig`, it deletes the whole download path (`handlers.go:327-345`). Deletes run on a separate goroutine (`watchDeleteChannel`, `start.go:99`).
- **Folders:** `delete_after` (default 10m) removes extracted files and/or the original archives according to `delete_files` / `delete_original` (`folder.go:480`).

### Retries
- `EXTRACTFAILED` is retried after `retry_delay` (default 5m), up to `max_retries` (default 2) (`handlers.go:280-297`).
- There is also a manual retry, `POST /api/queue/retry`.
- A failed poll just logs, records `lastPollErr` and bumps a metric. There is no backoff.

### Notifications and hooks (`pkg/hooks`)
- **Webhooks** (`[webhook.<slug>]`) POST a Go-template body. The built-in template is picked by name or by sniffing the URL (`profiles.go:32`): notifiarr (default, JSON), discord, telegram, slack, pushover (form-encoded), gotify, ntfy, apprise, mattermost, or a custom `template_path` file.
- Discord and Telegram **edit the previous message** for the same item (`update`, `CanUpdate`; `hooks/update.go`).
- Per-hook settings: `events` filter, `exclude` (app or instance name), `silent`, `nickname`, `channel`, `token`, `headers`, `content_type`, `timeout`, `ignore_ssl`.
- **Command hooks** (`[cmdhook.<slug>]`) run a command with the payload marshalled to `UN_*` env vars (`hooks/cmd.go:44`). `shell = true` runs it through `/bin/sh -c` (or `cmd` on Windows).
- `[hooks] custom_ids` injects extra payload keys, and `[hooks.titles]` overrides the per-event titles (`apps.go:103-122`).
- Delivery runs on one buffered worker (`hooks/worker.go`). Hooks are queued under the lock and flushed after unlock (`history.go`).

### Metrics
Prometheus metrics are at `/metrics`, with API-key or Bearer auth only (`pkg/unpackerr/metrics.go:108-148`):
- `unpackerr_counters{name}`, `unpackerr_gauges{name}`, `unpackerr_buffers{name}`
- `unpackerr_app_queue_fetch_errors_total`, `_fetch_total`, `_size`, `_fetch_time_seconds`
- `unpackerr_archives_read_total`, `unpackerr_bytes_written_total`, `unpackerr_extract_time_seconds` (histogram), `unpackerr_files_extracted_total`, `unpackerr_uptime_seconds_total`

JSON stats are also available at `/api/stats`.

### Logging
- stdlib `log` with INFO/ERROR/DEBUG loggers (`start.go` `Logger`).
- Rotating file via `rotatorr` (`log_file`, `log_files`, `log_file_mb`, `log_file_mode`), and a separate HTTP access log in Apache format.
- The UI log viewer tails files over the WebSocket (`logtee.go`, `logfiles.go`).
- `activity` suppresses idle-queue lines. `log_queues` (default 1m) prints the queue summary.
- Platform log backends are in `logs_linux.go` / `logs_windows.go`.

### Web UI and API (new in v1.0)
- Auth: a login session cookie, where the client sends a PBKDF2 digest and the server stores bcrypt. Also `webauth` proxy-header auth, `noauth`, named API keys with custom roles and `area:resource:verb` permissions (`auth.go`, `apikeys.go`, `permissions.go`), and an `upstreams` trust list.
- Endpoints cover queue and history (retry/forget/clear), per-section config GET/PUT/test with an idle self-restart (`restart_*.go`), a file browser (`browse.go`), and an OpenAPI doc.

---

## 4. Integrations (full lists)

**Starr apps** (`pkg/unpackerr/{sonarr,radarr,lidarr,readarr}.go`; config maps in `apps.go:90-93`):

| App | API | Queue record fields used | Extra |
|---|---|---|---|
| Sonarr | v3 | title, status, trackedDownloadStatus/State, protocol, outputPath, size, sizeleft, downloadId, seriesId, episodeId, statusMessages | — |
| Radarr | v3 | same set + movieId | — |
| Lidarr | v1 | + artistId, albumId | `split_flac` → ManualImport |
| Readarr | v1 | + authorId, bookId | — |
| **Whisparr** | — | **No dedicated support.** The docs say to add it as a Radarr instance with `name = "Whisparr"`. "Sportarr" is added as a Sonarr instance (`pkg/configdef/definitions.yml:607-611`). | |
| Prowlarr, Bazarr, etc. | — | Not integrated (they have no download queue). | |

- Multiple named instances per app: `[sonarr.<slug>]` is a `map[string]*Config`. Legacy `[[sonarr]]` arrays still load as slugs `"0"`, `"1"`, … (`instancemap.go`).
- Version sensitivity: the protocol string changed from `torrent` to `TorrentDownloadProtocol` in newer *arr versions, so both are accepted (`apps.go:24-28`).

**Notification targets** (`pkg/hooks/profiles.go`, `templates.go`): Notifiarr, Discord, Telegram, Slack, Pushover, Gotify, ntfy, Apprise, Mattermost, a custom template, and a command hook.

**Other outbound calls**
- GitHub releases (`api.github.com/repos/Unpackerr/unpackerr/releases/latest`), used only by the tray update check (`pkg/update/check.go:27`, `tray.go:156`).

---

## 5. External APIs / interfaces

**Starr endpoints polled** (via `golift.io/starr`; header `X-Api-Key`, `starr/http.go:159`; optional HTTP basic `http_user`/`http_pass`):

| App | Request |
|---|---|
| Sonarr | `GET /api/v3/queue?page=N&pageSize=500&sortKey=timeleft&includeUnknownSeriesItems=true` |
| Radarr | `GET /api/v3/queue?…&includeUnknownMovieItems=true` |
| Lidarr | `GET /api/v1/queue?…&includeUnknownArtistItems=true` |
| Readarr | `GET /api/v1/queue?…&includeUnknownAuthorItems=true` |

- Paging comes from `GetQueue(2000, 1)`: `DefaultQueuePageSize = 2000` (`apps.go:22`), which `starr.SetPerPage` turns into 500 per page, looping pages until 2000 records or `totalRecords` (`starr/sonarr/queue.go:77-120`, `starr/paginate.go:156`).
- Lidarr only: `GET /api/v1/manualimport?folder=&downloadId=&artistId=&replaceExistingFiles=true&filterExistingFiles=false`, then `POST /api/v1/command` `{name:"ManualImport", files:[…], importMode, replaceExistingFiles}` (`lidarr.go:163-226`; `starr/lidarr/manualimport.go:80`).
- Unpackerr **never** writes to Sonarr/Radarr (no command, no import trigger). It relies on the *arr's own 1-minute `RefreshMonitoredDownloads` re-check.

**Outbound webhook payload** (default Notifiarr JSON; `pkg/hooks/templates.go:19-99`):
```json
{ "path": "...", "app": "Sonarr", "ids": {"title","downloadId","seriesId","episodeId","reason",...},
  "customIDs": {...}, "unpackerr_eventtype": "extracted", "retries": 0, "eventTitle": "...",
  "time": "...", "data": {"error","archives":[],"files":[],"start","tmp_folder","bytes","elapsed"},
  "go_version","os","arch","version","revision","branch","started" }
```
Command hooks get the same fields as `UN_*` env vars.

**Own HTTP API** (`INTERNALS.md` §HTTP routes; `pkg/unpackerr/webserver.go`):
- `/api/auth/{login,logout,me}`, `/api/stats`, `/api/system[/export]`
- `/api/queue[/item|/retry|/forget]`, `/api/history[/clear|/delete]`
- `/api/browse[/template]`
- `/api/config/{help,env,{section}[,/live,/test]}` (GET/PUT/POST)
- `/api/openapi.json`, `/metrics`, `/debug/pprof` (opt-in), WebSocket `{urlbase}ws`

**Config**
- TOML file (`examples/unpackerr.conf.example`, generated from `pkg/configdef/definitions.yml`).
- Every key is overlaid by env vars with prefix `UN_`, e.g. `UN_SONARR_<slug>_URL`, `UN_WEBHOOK_0_EVENTS`, `UN_PASSWORDS`.
- `filepath:/path` indirection works for secrets.
- CLI flags: `--config`, `--prefix`, `--webhook N` (send a sample), `--reset` (new UI password), `--version`.

---

## 6. Background jobs

All jobs run as tickers inside `Run()`'s `select` (`start.go:276-342`), apart from separate goroutines for deletes, hooks, the HTTP server and poll workers.

| Job | Interval | Function |
|---|---|---|
| Starr queue poll + change check | `interval` (default 2m, min 15s) | `retrieveAppQueues` + `checkQueueChanges` |
| Start queued extractions | `start_delay` (default 1m) | `extractCompletedDownloads` |
| Cleaner: done/retry/delete/stale, folder stats, idle restart | 5s fixed (`cleanerInterval`) | `checkExtractDone`, `checkFolderStats`, `maybeRestart` |
| Progress log | `progress` | `printProgress` |
| Queue summary log | `log_queues` (default 1m1s) | `logCurrentQueue` |
| Folder poller | per-folder `interval` (0 = fsnotify only) | radovskyb watcher |
| History compaction | when appends reach `2×keep_history` (default 400) | `historyfile.go` |
| Stale safety net | 24h age check on the cleaner tick | `staleItemTimeout` |

---

## 7. Configuration surface

Source of truth: `pkg/configdef/definitions.yml`.

- **Global:** `debug`, `quiet`, `error_stderr`, `activity`, `log_queues`, `log_file`, `log_files`, `log_file_mb`, `log_file_mode`, `keep_history`, `interval`, `progress`, `start_delay`, `retry_delay`, `max_retries`, `remnant_action`, `parallel`, `file_mode`, `dir_mode`, `passwords`. Also the undocumented `timeout` and `delete_delay` defaults.
- **`[webserver]`:** `metrics`, `pprof`, `listen_addr` (`0.0.0.0:5656`; empty disables), `ui_password` (crypt / `webauth` / `noauth` / `filepath:`), `ui_role_header`, `log_file*`, `ssl_cert_file`, `ssl_key_file`, `urlbase`, `upstreams`, `ws_origins`, `api_keys`, `roles`.
- **`[sonarr|radarr|lidarr|readarr.<slug>]`:** `name`, `url`, `api_key`, `http_user`, `http_pass`, `username`, `password`, `paths`, `protocols`, `timeout`, `delete_delay`, `delete_orig`, `syncthing`, `valid_ssl`, `max_bytes`, and `split_flac` (Lidarr only).
- **`[folders]`:** `buffer` (event channel, default 20000).
- **`[folder.<slug>]`:** `path`, `interval`, `exclude_paths`, `extract_path`, `delete_after`, `disable_recursion`, `max_bytes`, `max_files`, `max_ratio`, `max_nested`, `extras_max_depth`, `allow_symlinks`, `delete_files`, `delete_original`, `disable_log`, `move_back`, `extract_isos`, `wait_extensions`, `skip_empty`.
- **`[hooks]`:** `custom_ids`, `titles`.
- **`[webhook.<slug>]`:** `url`, `name`, `silent`, `events`, `nickname`, `channel`, `token`, `exclude`, `template_path`, `template`, `update`, `ignore_ssl`, `timeout`, `content_type`, `headers`.
- **`[cmdhook.<slug>]`:** `command`, `name`, `shell`, `silent`, `events`, `exclude`, `timeout`.
- **Runtime-editable** through `PUT /api/config/{section}`. Some fields need a restart: `parallel`, logger, file/dir mode, listen, TLS, folders (`INTERNALS.md` cheat sheet).

---

## 8. Platform / infrastructure concerns

- **No native dependencies.** Pure Go with CGO disabled. RAR, 7z, ISO/UDF and FLAC are all handled by pure-Go decoders; there is no `unrar` or `7z` binary (`xtractr/README.md`: "without Cgo"). Caveat: `rardecode` is a clean-room decoder, so edge cases in RAR5 features (e.g. some recovery or encrypted-header combinations) depend on that library.
- **OS support** (`.goreleaser.yaml`):
  - Linux amd64/arm/arm64/386, packaged as deb/rpm with systemd (`init/systemd`) and Docker (alpine; `Dockerfile`, `init/docker/Dockerfile.goreleaser`).
  - macOS amd64/arm64 as a `.app` with a tray (`init/macos`, `tray.go` build tag `windows || darwin`).
  - Windows with a tray and zenity dialogs (`password_gui.go`, `init/windows`).
  - FreeBSD rc.d (`init/bsd`).
- **Platform-specific code:**
  - Restart uses `syscall.Exec` on Unix and spawn+exit on Windows (`restart_unix.go`, `restart_windows.go`).
  - Separate signal handling (`signals_*.go`) and log handling (`logs_*.go`).
  - xtractr has `fileid_{unix,windows}.go` and `files_{unix,windows}.go` for hardlink/inode identity and path truncation (`TruncatePathForFS`).
- **Frontend build:** the SPA needs Node at build time (`go generate ./frontend`), then it is embedded.
- **Filesystem assumptions:**
  - Unpackerr must see the *same* download path as the *arr, or map it through `paths[]`. There are no remote-path-mapping rules beyond that list plus the `outputPath` basename fallback.
  - The move-back step needs write access to the download folder.

---

## 9. Interactions with other *arr apps

**How Unpackerr works with the *arr import today:**
1. The torrent client finishes. The *arr's `RefreshMonitoredDownloads` (every 1 min) sees the download as complete and tries to import. It finds only `.rar` files, so the `TrackedDownload` warns "No files found are eligible for import in {path}" (Sonarr `Core/Download/CompletedDownloadService.cs:162`) and the item stays in the queue as `importPending`/`importBlocked` with a warning.
2. Unpackerr polls the queue and sees `status=completed` with a torrent protocol, so it creates a `WAITING` item. It copies `statusMessages` into `IDs.reason` and logs "Activity Queue status" (`handlers.go:162`).
3. After `start_delay` it extracts into `<path>_unpackerred`, then moves the files into `<path>`. Doing this in a temp folder sidesteps Sonarr's `NotUnpackingSpecification`, which only knows the `_UNPACK_`/`_FAILED_` working-folder prefixes (Sonarr `Core/MediaFiles/EpisodeImport/Specifications/NotUnpackingSpecification.cs:33`).
4. Status becomes `EXTRACTED` ("Awaiting Import"). On its next 1-minute tick the *arr finds the video files and imports them, by copy or hardlink, because the torrent is still seeding.
5. The item disappears from the *arr queue, so Unpackerr marks it `IMPORTED`, waits `delete_delay`, and then deletes only the extracted files. The RARs stay for seeding. If the *arr removes the completed download itself, `delete_orig` or the client handles that.
6. Coordination is entirely **implicit through queue presence and title matching** (`Map` is keyed by `title`, `haveQitem`). There is no callback and no lock. The "wait for import" semantics are "until the title leaves the queue". Races are covered by the `queueSnapshotReady` guard, the queue-return resets, and the 24h stale sweep.
7. **Usenet** is skipped by default because SABnzbd and NZBGet unpack themselves, and Sonarr's `NotUnpacking` spec handles their `_UNPACK_` folders.

**In a monolith, this becomes an in-process step:**
- The whole poll-queue / title-match / infer-imported loop disappears. The monolith already owns the `TrackedDownload` state machine (Downloading → ImportPending → Importing → Imported, see sonarr.md "Download handling").
- Extraction becomes a stage in completed-download handling, between "client reports complete" and "import decision":
  - `Downloading` → **`Extracting`** (new tracked state) → `ImportPending` → `Importing` → `Imported` → **cleanup extracted files** (immediately after a successful import, or after a short grace period).
- The trigger is the monolith's own client-monitoring tick rather than a 2m poll. It is keyed by `downloadId` (the torrent hash or nzb id), not by title.
- The import decision should run on the **extracted file list** (`Response.NewFiles`) directly, so there is no need to wait a tick and rescan.
- `DeleteDelay` / `IMPORTED` timing becomes "after `ImportApprovedEpisodes` succeeds, delete `NewFiles`". The seeding goal and `RemoveCompletedDownloads` logic stays with the download-client layer.
- `NotUnpackingSpecification` stays only for external unpackers (SAB/NZBGet `_UNPACK_`). The monolith's own extraction is atomic from the importer's point of view.
- The Lidarr split-FLAC ManualImport round trip becomes a direct call into the music importer.

---

## 10. Outdated / legacy candidates

| Item | Evidence | Note |
|---|---|---|
| Readarr support | `pkg/unpackerr/readarr.go`, `defaultReadarrMaxBytes` | The upstream Readarr project was retired (2025). Keep only if the monolith covers books. |
| Legacy protocol strings `torrent` / `usenet` | `apps.go:24-28`, `definitions.yml:651-668` | Only needed for old *arr versions. |
| `[[sonarr]]` array config dual-read | `instancemap.go`, `INTERNALS.md:28` | Migration shim. |
| `DELETEFAILED` status | `pkg/extract/status.go:23` ("unused") | Dead enum value, kept for numeric compatibility. |
| Whisparr / Sportarr "add as Radarr/Sonarr" guidance | `definitions.yml:607-611` | A workaround, not an integration. |
| Syncthing `.tmp` wait | `handlers.go:381` | Niche. Generalize as "wait while temp/partial files exist" (as folders already do with `wait_extensions`). |
| Title-keyed tracking plus queue-absence as the "imported" signal | `handlers.go:66`, `starrpoll.go:152` | Inherently racy. The code has many guards: the snapshot-ready check, the tombstone, and the 24h sweep. Unnecessary in-process. |
| Desktop tray, zenity dialogs, GitHub update check, self re-exec restart | `tray.go`, `password_gui.go`, `pkg/update`, `restart_*.go` | Desktop-app heritage. Not needed in a server monolith. |
| Separate auth, API keys, roles, UI, config-PUT stack | `auth.go`, `apikeys.go`, `configput.go` (~2.5k LOC), SPA | Duplicates what the monolith provides once. |
| Notifiarr-default webhook payload / Discord message editing | `hooks/templates.go`, `hooks/update.go` | Replace with the monolith's notification framework. The Notifiarr payload is a niche contract. |

---

## 11. Rewrite notes

**Reuse recommendation: import `golift.io/xtractr` as a library. Do not port Unpackerr itself.**
- **Why xtractr:**
  - MIT, pure Go, no CGO.
  - Covers every format that matters: multi-volume RAR (including passwords), 7z, zip, tar variants, ISO/UDF, and single-stream codecs.
  - Queue with parallelism, progress callbacks, nested archives, zip-bomb limits (`MaxBytes`/`MaxFiles`/`MaxRatio`/`MaxNested`/`ExtrasMaxDepth`), temp-folder-then-move atomicity, refused-overwrite reporting and `FinalDests`.
  - About 9.6k LOC non-test, actively maintained (commit 2026-09-26).
  - Dependencies are all pure Go and permissively licensed (rardecode BSD-2, bodgit/sevenzip BSD-3, klauspost/compress BSD/MIT/Apache, ulikunitz/xz BSD-3, mewkiz/flac Unlicense/BSD, …). Verify each license during the dependency review.
- **Pin a tagged release.** Unpackerr pins a pseudo-version (`v0.6.2-0.2026…`), so the API is still moving (Response fields such as `Refused`/`FinalDests` are recent).
- **API to use:**
  - `xtractr.NewQueue(&Config{Parallel, Suffix, FileMode, DirMode, MaxBytes…})`
  - `q.Extract(&Xtract{Filter{Path, ExcludeSuffix: AllExcept(...)}, Passwords, MaxBytes, CBChannel/CBFunction, Progress})`
  - `xtractr.FindCompressedFiles` for the "does this download need extraction?" check.
  - `xtractr.IsLimitError` for the no-retry classification.
- **Optionally reuse `golift.io/starr`** only if the monolith keeps adapters that talk to *external* *arr instances (migration or import from a legacy Sonarr). Not needed otherwise.
- **Worth lifting as logic** (small, MIT, a clean-room rewrite is also cheap):
  - Remnant detection and cleanup (`remnants.go`, ~384 LOC: `PreFiles` snapshot → rename, delete or off).
  - The per-app default byte caps.
  - The password-in-path `{{…}}` convention.
  - The folder `wait_extensions` logic (`pkg/folders/wait.go`).
  - The FLAC+CUE split flow, which is in xtractr already.
- **Do not reuse:** the Starr polling/state machine, config/env/`configdef`, web/auth, tray, hooks, the history JSONL, or restart. The monolith has its own equivalents, and the title-keyed polling model is exactly what in-process integration removes.

**Where extraction fits in the monolith's pipeline**
```
DownloadClient poll (1m) → TrackedDownload(downloadId) status=Completed
  → [ExtractionStage] FindCompressedFiles(outputPath); if none → ImportPending
       else state=Extracting; xtractr.Extract(temp=<path>_unpacked → move back)
            ok → record NewFiles on TrackedDownload → ImportPending (import immediately)
            fail → retry w/ backoff (retry_delay, max_retries); limit/password error → ImportBlocked + ManualInteractionRequired event
  → ImportDecisionMaker over (download dir incl. NewFiles) → ImportApproved (hardlink/copy/move)
  → Post-import cleanup: delete NewFiles (+ empty dirs) unless import mode was "move"; RARs remain for seeding
  → RemoveCompletedDownloads (existing client-layer policy, seeding goals)
```

**Design notes**
- Make extraction a per-download-client or per-tag option. Default it **on for torrent clients and off for Usenet** (SABnzbd and NZBGet unpack themselves), mirroring `protocols`.
- Persist the extraction state on the `TrackedDownload` / history row so a crash can resume. That replaces the 72h JSONL restore.
- Add a **free-space pre-check**, which Unpackerr lacks. Unpacked size is known for RAR/7z/zip headers; otherwise use archive size × a factor. Put it next to the existing `FreeSpaceSpecification`.
- Keep a separate "watch folder" feature only if it is wanted as a generic tool. It is unrelated to the *arr pipeline and could be a small optional module that reuses the same xtractr queue.
- Global queue concurrency (`Parallel`) should be a monolith setting. Heavy I/O should run on its own worker pool, not on the job scheduler threads.
- Expose extraction progress (`xtractr.Progress`) through the monolith's queue API and websocket, the same way download progress is shown.
- Add events for notifications: `ExtractionStarted`, `ExtractionFailed`, `ExtractionCompleted`. These replace Unpackerr's webhook statuses.
