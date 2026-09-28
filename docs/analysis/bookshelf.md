# Bookshelf (Readarr fork) — Codebase Analysis

Source: `submodules/bookshelf` (HEAD `3ac41f7e3`, 2026-09-10; origin `github.com/pennydreadful/bookshelf`). Paths are relative to the submodule; `Core/` = `src/NzbDrone.Core/`. Fork point = last upstream Readarr commit `0b79d3000` ("Retirement announcement", 2025-06-27). "Fork diff" below = `git diff 0b79d3000 HEAD`.

Shared NzbDrone infrastructure (ThingiProvider, Datastore/Dapper, Messaging/commands, TaskManager, HealthCheck, Housekeeping, Backup, auth, SignalR, update, Http plumbing, DecisionEngine skeleton, CustomFormats, Download tracking) is **not** re-documented here — see [sonarr.md](sonarr.md) / [radarr.md](radarr.md). This report covers the book domain and divergences.

---

## 1. Overview

**Purpose:** ebook/audiobook PVR. Tracks authors → books (works) → editions, grabs releases from indexers, imports/identifies files by reading embedded tags (EPUB/AZW3/MOBI/PDF/audio), renames, optionally pushes ebooks into a Calibre library via the Calibre content server. A single instance handles **either** ebooks **or** audiobooks for a given book in practice (README: "If you want both an audiobook and ebook of a given book you will need multiple instances").

**Stack / versions**
- Backend: C# **.NET 6** (`net6.0` in every csproj; `mise.toml` pins `dotnet = "6.0.428"`, no `global.json`). .NET 6 has been EOL since Nov 2024. The fork briefly moved the SDK to 8 (`c92fd145c`, "use dotnet 8") and then back to a 6.0 SDK in `bb9069615`; the runtime target never changed. Upstream NzbDrone lineage packages: DryIoc 5.4, Dapper 2.0.151, Servarr.FluentMigrator 3.3.2.9, System.Data.SQLite.Core.Servarr + Npgsql 7, NLog 5.1, FluentValidation 9.5.4, Polly 8.5, MailKit 4.8, Ical.Net 4.3, MonoTorrent 2.0.7, ImageSharp 3.1.11, Swashbuckle 6.6, LazyCache 2.4 (`src/Directory.Packages.props`).
- Book-specific libraries: **TagLibSharp-Lidarr 2.2.0.19** (audio tags), **PdfSharpCore 1.3.65** (PDF info dict), an in-tree EPUB reader (`Core/MediaFiles/EpubTag/`, a vendored VersOne.Epub derivative) and an in-tree MOBI/AZW3 header parser (`Core/MediaFiles/AzwTag/`).
- Frontend: React **17.0.2**, Redux 4.2, react-router-dom 5.2, webpack 5, TypeScript 5.1, @microsoft/signalr 6. There are 1,392 source files (~103k LOC), of which only 411 are .ts/.tsx; the rest is JS. This is the same pre-migration Servarr frontend Readarr had.
- DB: SQLite or Postgres. Three databases: main, logs, and **cache** (`Core/Datastore/CacheDatabase.cs`, table `HttpResponse` for cached metadata HTTP responses; the fork mostly bypasses it).
- Runtime targets were cut to `linux-x64; linux-musl-x64; linux-musl-arm64` (`src/Directory.Build.props`). The only distribution the fork maintains is Docker (`docker/Dockerfile`, LinuxServer alpine base with s6). `distribution/osx` and `distribution/windows` are stale leftovers from Readarr.

**LOC (.cs, non-test unless noted)**

| Project | LOC | Files |
|---|---:|---:|
| NzbDrone.Core | 90,863 | 1,319 |
| Readarr.Api.V1 | 9,924 | 155 |
| Readarr.Http | 3,255 | 71 |
| NzbDrone.Common | 13,241 | 158 |
| NzbDrone.Host | 1,152 | 11 |
| NzbDrone.Mono / Windows | 1,383 / 231 | 14 / 2 |
| NzbDrone.Update | 655 | 10 |
| NzbDrone.SignalR / Console / tray | 94 / 131 / 227 | — |
| NzbDrone.Core.Test | 40,086 | 284 |
| frontend/src | ~103k | 1,392 |

Larger Core areas by file count: Download 223, Notifications 176, MediaFiles 120 (plus EpubTag/AzwTag), Datastore 93, Indexers 85, Books 80, ImportLists 51, MetadataSource 42, DecisionEngine 41.

**Lineage:** NzbDrone (2010) → Sonarr → **Lidarr** (music) → **Readarr** (first release "New: Readarr 0.1", commit `08496c82a`, 2020-05) → **Bookshelf** (fork from 2025-08-08). The Lidarr ancestry is still visible: about 85 Core files use Artist/Album/Track vocabulary, for example `CloseAlbumMatchSpecification.cs`, `AlbumUpgradeSpecification.cs`, `ImportArtistDefaults.cs`, `TrackGroupingService.cs`, `TrackImportedEvent`, `Parser.ReportMusicTitleRegex`, and `DistanceCalculator` weights such as `track_title`/`recording_id`. Internal names are unchanged: the solution is `Readarr.sln`, the binary is `Readarr`, custom-script env vars use the `Readarr_` prefix, the API is `/api/v1`, and the openapi title is "Readarr". Version is `READARR_VERSION: 0.4.21` (`.github/workflows/build.yml:21`).

**Fork activity:** 10,682 commits in total. After the retirement commit there are **32 commits**: 22 by `pennydreadful`, the rest from about 12 drive-by contributors. They came in bursts (Aug 2025: 16, Jan–Feb 2026: 9, Sep 2026: 9). The fork diff touches 215 files, +96k/−15k lines, but that is mostly `packages.lock.json` files and a Weblate locale churn. Real code change is small (~3k lines), and 2.3k of it is the MyAnonaMouse indexer. README TODOs still open: "Monitor series" and "Support ebook and audio files in the same root". Last upstream Readarr release: 0.4.19 (2025-06-15, `f48eac7e3`).

---

## 2. Domain model

Tables (`Core/Datastore/TableMapping.cs`, 41 entities). The book-specific ones are:

- **Author** (`Core/Books/Model/Author.cs`, `Authors`): the per-library record. AuthorMetadataId, CleanName, Monitored, `MonitorNewItems` (`NewItemMonitorTypes`: All, None, New), LastInfoSync, Path, RootFolderPath, QualityProfileId, **MetadataProfileId**, Tags, AddOptions (embedded).
- **AuthorMetadata** (`AuthorMetadata.cs`, `AuthorMetadata`): data shared across library records. ForeignAuthorId (a Goodreads author id, or a Hardcover id on the hardcover image), TitleSlug, Name, SortName, NameLastFirst, SortNameLastFirst, Aliases, Overview, Disambiguation, Gender, Hometown, Born, Died, Status (`AuthorStatusType` Continuing/Ended), Images, Links, Genres, Ratings. The split mirrors Lidarr's ArtistMetadata. It lets Book rows reference authors that are not in the library (co-authors).
- **Book** (`Book.cs`, `Books`) = a **work**. AuthorMetadataId, ForeignBookId (work id), ForeignEditionId (the preferred edition), Title, CleanTitle, TitleSlug, ReleaseDate, Links, Genres, **RelatedBooks** (list of ints), Ratings, Monitored, **AnyEditionOk**, LastSearchTime (mig 039), LastInfoSync, Added, AddOptions.
- **Edition** (`Edition.cs`, `Editions`): BookId, ForeignEditionId, **Isbn13, Asin**, Title, Language, Overview, **Format** (free text such as "Kindle Edition" or "Audible Audio"), **IsEbook**, Disambiguation, Publisher, PageCount, ReleaseDate, Images, Links, Ratings, **Monitored** (one monitored edition per book, index in mig 022), ManualAdd.
- **Series** (`Series.cs`) and **SeriesBookLink** (`SeriesBookLink.cs`): ForeignSeriesId, Title, Description, Numbered, WorkCount, PrimaryWorkCount. The link table holds Position (string, e.g. "1.5"), SeriesPosition (int, mig 018) and IsPrimary. Series are metadata only and cannot be monitored (README TODO).
- **BookFile** (`Core/MediaFiles/BookFile.cs`, `BookFiles`): Path (absolute, not relative as in Sonarr/Radarr), Size, Modified, DateAdded, OriginalFilePath, SceneName, ReleaseGroup, Quality, IndexerFlags, MediaInfo, **EditionId** (a file belongs to an edition, not to a book), **CalibreId**, **Part/PartCount** (multi-file audiobooks, mig 010/012), **LastRemoteSearchTime** (fork mig 041).
- **MetadataProfile** (`Core/Profiles/Metadata/MetadataProfile.cs`), unique to Readarr. Fields: MinPopularity, SkipMissingDate, SkipMissingIsbn, SkipPartsAndSets, SkipSeriesSecondary, AllowedLanguages (CSV of ISO-639-3 codes plus "null"), MinPages, Ignored (terms). `MetadataProfileService.FilterBooks` (lines ~150–240) decides which of an author's works and editions get stored at all:
  - popularity = `Ratings.Popularity` (votes × average). Books with a future release date always pass the popularity filter.
  - a part-or-set regex matches "1 of 3", "1/3", "1-3".
  - secondary series items are dropped when SkipSeriesSecondary is set.
  - there is a special "None" profile.
  - the default "Standard" profile uses MinPopularity 350. On the Hardcover image (env `HARDCOVER=true`, `MetadataProfileService.cs:286`, fork `eec7491bd`) it uses 50.
- **RootFolder** (`Core/RootFolders/RootFolder.cs`) carries book-specific defaults: DefaultMetadataProfileId, DefaultQualityProfileId, DefaultMonitorOption, DefaultNewItemMonitorOption, DefaultTags, **IsCalibreLibrary + CalibreSettings** (embedded).
- **Monitor options** (`MonitorTypes`): All, Future, Missing, Existing, Latest, First, None, Unknown.

**Ebook vs audiobook.** Nothing on Author or Book marks it as ebook or audio. The distinction comes from four places:
1. **Qualities** (`Core/Qualities/Quality.cs:73-114`):
   - ebook: Unknown Text(0), PDF(1), MOBI(2), EPUB(3), AZW3(4)
   - audio: MP3(10), FLAC(11), M4B(12), UnknownAudio(13)
   - Weights place all audio above all text, so one quality profile containing both would "upgrade" an EPUB to an MP3. Users keep separate profiles, and in practice separate instances.
2. **Extension map** (`Core/MediaFiles/MediaFileExtensions.cs`): `.epub/.kepub`→EPUB, `.mobi`, `.azw3`, `.pdf`, and a long audio list (`.m4b`→M4B; `.mp3/.m4a/.aac/.ogg/.wma`→MP3; `.flac/.ape/.wav/.alac`→FLAC).
3. **Edition.Format / IsEbook** from metadata. Identification biases toward the matching format (see §3).
4. **Calibre** is used only for ebooks.

**Migrations:** 41 (`Core/Datastore/Migration/000–041`). Readarr squashed its history at 001, which is much smaller than Sonarr's 226 or Radarr's 140. The fork's only schema change is `041_bookfile_last_remote_search.cs`. Hardcover-image databases use the same schema but store **Hardcover IDs in the Foreign* columns**, so switching between the Goodreads and Hardcover images needs a fresh DB (README).

---

## 3. Feature areas

Top-level `Core/` folders. [S] = shared NzbDrone infra (same concept as Sonarr/Radarr); [B] = book-specific or substantially diverged.

- Annotations [S]: field-definition attributes for provider UI schemas.
- Authentication [S]: forms/basic/none, the `Users` table.
- AuthorStats [B]: per-author and per-book file, size and count aggregates (SQL). Equivalent to SeriesStats.
- Backup [S].
- Blocklisting [S].
- **Books [B]**: the Author/Book/Edition/Series model, AddAuthor/AddBook, Refresh{Author,Book,Edition,Series,SeriesBookLink}Service (`RefreshEntityServiceBase` is the generic diff/merge from Lidarr), `ShouldRefreshAuthor`, BookMonitoredService, MonitorNewBookService, MoveAuthorService, **Calibre/** (content-server client + language maps). `Books/output.txt` is an empty stray file.
- Configuration [S+B]: book keys listed in §7.
- CustomFilters [S].
- CustomFormats [S]: a subset of Radarr's (release title and release group specs only).
- Datastore [S]: plus CacheDatabase.
- DecisionEngine [S+B]: `DiscographySpecification` (author-wide packs) and `EarlyReleaseSpecification` (per-indexer "early download limit") are book/Lidarr-specific.
- DiskSpace [S].
- Download [S]: pending, tracked downloads, CDH, and 17 clients.
- Exceptions [S].
- Extras [S, vestigial]: the metadata-consumer framework remains, but **there are no consumers** (Roksbox/Wdtv/Xbmc were deleted at Readarr 0.1). "Others" still copies extra files.
- HealthCheck [S+B]: 26 checks, including `CalibreRootFolderCheck`, `ImportListRootFolderCheck`, `IndexerJackettAllCheck`.
- History [S].
- Housekeeping [S]: 33 housekeepers.
- Http [S]: CachedHttpResponseService, CloudFlare interceptor, `TorcacheHttpInterceptor` (dead service).
- **ImportLists [B]**: Goodreads ×4, Hardcover, LazyLibrarian, Readarr (see §4).
- IndexerSearch [B]: criteria types are `AuthorSearchCriteria` and `BookSearchCriteria` only.
- Indexers [S+B]: Newznab/Torznab `t=book` with author/title params, plus **native MyAnonaMouse** (fork).
- Instrumentation [S]: **Sentry removed** by the fork.
- Jobs [S].
- Languages [S]: small (5 files).
- Lifecycle [S].
- Localization [S]: 45 locales.
- MediaCover [S+B]: author and book covers. The fork skips covers that have invalid URLs.
- **MediaFiles [B]**: DiskScan, BookImport (identification, aggregation, specifications, manual import), **EbookTagService**, **AudioTagService**, EpubTag/AzwTag readers, retag commands, RootFolderWatchingService (FS watcher).
- Messaging [S].
- **MetadataSource [B]**: BookInfo proxy, Goodreads proxy, search proxy (deep dive below).
- Notifications [S+B]: Goodreads shelf notifications, Kavita, Subsonic.
- Organizer [S+B]: book naming tokens.
- Parser [B]: release title → author/book (`ReportBookTitleRegex`), plus leftover `ReportMusicTitleRegex`, ISO languages and countries.
- Profiles [S+B]: Quality, Delay, Releases, and **Metadata** [B].
- ProgressMessaging [S].
- Qualities [B]: the 9 book/audio qualities.
- Queue [S].
- RemotePathMappings [S].
- RootFolders [B]: Calibre settings and default profiles per root.
- Security [S].
- Tags [S].
- ThingiProvider [S].
- Update [S]: still points at `readarr.servarr.com`.
- Validation [S].
- *Analytics*: **deleted** by the fork (`5319d12b9`, "remove spyware").

### Deep dive: metadata source(s) — how Bookshelf survived

**Upstream Readarr** called `https://api.bookinfo.club/v1/{route}` (`ReadarrCloudRequestBuilder.Metadata`, removed in the fork). This was Servarr's closed-source Goodreads-derived "BookInfo" server, and it was shut down along with the project. Search went **directly** to `https://www.goodreads.com/book/auto_complete?format=json`, sending a spoofed Chrome UA.

**Bookshelf changes (fork commits `ae242f17e` "work with selfhosted metadata", `6759e22a5`, `b27f807ad`):**
1. `ConfigService.MetadataSource` (`Core/Configuration/ConfigService.cs:266-288`) resolves in this order:
   - the DB config value `MetadataSource` (editable in UI **Settings → Development**, `frontend/src/Settings/Development/DevelopmentSettings.js:66`)
   - env **`METADATA_URL`**
   - hard default **`https://api.bookinfo.pro`**
2. `MetadataRequestBuilder` (`Core/MetadataSource/MetadataRequestBuilder.cs`) always builds `{MetadataSource}/{route}`. There is no longer a `/v1` prefix and no Servarr fallback.
3. Search was moved off goodreads.com onto the same server. `GoodreadsSearchProxy` now calls `{MetadataSource}/search?q=…` (`Core/MetadataSource/GoodreadsSearchProxy/GoodreadsSearchProxy.cs`).
4. Docker images (`.github/workflows/build.yml:109-160`, `docker/Dockerfile`):
   - `ghcr.io/pennydreadful/bookshelf:softcover*`: `METADATA_URL=https://api.bookinfo.pro`, Goodreads-backed and backward-compatible with Readarr DBs.
   - `:hardcover*`: `METADATA_URL=https://hardcover.bookinfo.pro`, `HARDCOVER=true`, Hardcover-backed, and **not** DB-compatible.
5. The server behind both URLs is **rreading-glasses** (`github.com/blampe/rreading-glasses`), a separate project: Go, GPL-3.0, a drop-in replacement for the BookInfo API. It needs PostgreSQL and, for the Hardcover backend, a Hardcover API token. It says it runs on a Raspberry Pi. The public instances at `bookinfo.pro` are community-run; the repo does not say who operates them. Bookshelf contains **no server code**. It depends on that third-party service or a self-hosted copy. Hardcover (`hardcover.app`) has an official GraphQL API. Goodreads has had no public API since Dec 2020, so the Goodreads-backed flavour has to obtain data without an official API.
6. Caching was mostly disabled so that fixes on the metadata server show up immediately:
   - `GetAuthorInfo(useCache=false)`
   - the author poll cache was cut from 10 to 1 minute
   - `_cachedHttpClient.Get(..., false, ...)` everywhere
   - `GetChangedBooks` returns null
   - the README describes this as "metadata is no longer cached locally"
7. Search errors now propagate (`BookInfoProxy.cs:240-255`: `catch (GoodreadsException) … throw`). Search-result reordering by id was removed (`7f0942293`, "preserve search ordering").

**Routes consumed** (BookInfo API contract, `Core/MetadataSource/BookInfo/BookInfoProxy.cs`):

| Route | Purpose |
|---|---|
| `author/{id}` | Full author with works, editions and series. Polled up to 60× with 2s sleeps while the server returns 202/"refreshing" (`PollAuthorUncached`, ~line 600). |
| `work/{id}` | Single work plus authors (`PollBook`). |
| `book/{id}` | Edition lookup, returns its work. |
| `book/bulk` | POST a list of edition ids. |
| `author/changed?since=` | Delta of changed authors. |
| `search?q=` | Title, author, ISBN or ASIN search. Returns `[{bookId, workId, …}]`, which is then expanded through `book/bulk`. |

The Resource DTOs are in `Core/MetadataSource/BookInfo/BookInfoResource/`. Search prefixes (`BookInfoProxy.cs:164-216`): `author:<id>`, `work:<id>`, `edition:<id>`, `isbn:<x>`, `asin:<x>`.

**Direct www.goodreads.com calls that remain** (`Core/MetadataSource/Goodreads/GoodreadsProxy.cs`). These use **hardcoded Goodreads developer keys stored reversed** (`"gSuM2Onzl6sjMU25HY1Xcd".Reverse()`, `"whFzJP3Ud0gZsAdyXxSr7T".Reverse()`) and a spoofed Android UA:
- `series/{id}?format=xml&page=` for the Goodreads Series import list (paginated by the fork in `0122bb7ba`)
- `book/list/listopia.xml` for the Goodreads Lists import list
- `api/book/basic_book_data/{id}`, used by `ImportListSyncService.cs:180` to resolve edition ids from lists

These are legacy Goodreads API endpoints. Goodreads stopped issuing keys in 2020, and this path only works while those old keys are tolerated.

### Deep dive: book/edition identification (import)
Pipeline (`Core/MediaFiles/BookImport/`, inherited from Lidarr's beets-style matcher):
1. **TrackGroupingService** groups files in a folder into releases (multi-file audiobooks become one LocalEdition).
2. **Aggregation**: `AggregateFilenameInfo`, `AggregateQuality`, `AggregateReleaseGroup`, `AggregateCalibreData` (reads Calibre metadata for files inside a Calibre library).
3. **CandidateService** (`Identification/CandidateService.cs`) looks for candidates in this order:
   - local DB
   - remote search by **ISBN**
   - remote search by **ASIN**
   - Goodreads id tags
   - author+title
   - title only
   - author only
4. **DistanceCalculator** (`Identification/DistanceCalculator.cs`) scores each candidate using the weights in `Distance.cs:11-38`:
   - author 3, book 3, ISBN 10, ASIN 10, language 5, year 1, publisher 0.5, book_id 5
   - **format bias**: text files add a penalty (`ebook_format` 0.1) if the edition format is not in {"Kindle Edition","Nook","ebook"}, and `wrong_format` (5.0) if it is an audio format. Audio files prefer {"Audiobook","Audio CD","Audible Audio","MP3 CD",…}.
   - titles are cleaned by stripping "(unabridged)" and series parentheses.
5. **CloseBookMatchSpecification** accepts a match only below a threshold. The fork raised it **0.20 → 0.50** (`Specifications/CloseAlbumMatchSpecification.cs`), which accepts looser matches ("Improved matching" in the README).
6. Fork addition: **LastRemoteSearchTime**. If a remote search completed but found no match, the file is stamped (`IdentificationService.StampRemoteSearchResult`) and later rescans skip it (`MediaFileService.FilterUnchangedFiles`, `DiskScanService`). The stamp resets when the file changes. This keeps library scans from hammering the metadata server. Commits `0cbabbff1` and `3ac41f7e3`.

### Deep dive: tag reading and writing
- `EBookTagService.ReadTags` (`Core/MediaFiles/EbookTagService.cs:61-79`) dispatches on extension:
  - `.pdf` → PdfSharpCore Info dictionary
  - `.epub/.kepub` → `EpubTag/EpubReader`, reading OPF title, authors, identifiers (ISBN, and ASIN by scheme), language, publisher, description, and `calibre:series`/`calibre:series_index`
  - `.azw3/.mobi` → `AzwTag/Azw3File`, reading EXTH title, authors, ISBN, ASIN, language, publisher, imprint, source
  - anything else → filename parser
- **Ebook tag writing only happens through Calibre** (`WriteTagsInternal` → `CalibreProxy.SetFields`). Files without a CalibreId are skipped. Readarr never writes EPUB/OPF itself.
- Audio: `AudioTagService` / `AudioTag.cs` use TagLib (ID3v2, MP4 atoms for m4b, Vorbis, and so on). They read and write title, authors, book, series, ISBN/ASIN and Goodreads id frames, with config options for writing and scrubbing tags. The fork fixed a null `OriginalReleaseDate` diff (`0ee317170`). There is **no fingerprinting**: the `AllowFingerprinting` config is a dead Lidarr leftover with no consumers.

### Deep dive: Calibre integration
`Core/Books/Calibre/CalibreProxy.cs` talks to the **Calibre content server** (`calibre-server`) over HTTP with basic or digest auth, configured per root folder (`RootFolder.IsCalibreLibrary`, `CalibreSettings`: Host, Port, UrlBase, Username, Password, Library, OutputFormat, OutputProfile, UseSsl). Endpoints used:
- `cdb/add-book/{job}/{dups}/{filename}/{library}`
- `cdb/delete-books/{ids}/{library}`
- `cdb/set-fields/{id}/{library}`
- `cdb/cmd/embed_metadata`
- `conversion/book-data/{id}`, `conversion/start/{id}`, `conversion/status/{job}` (automatic format conversion)
- `ajax/book/{id}/{library}`, `ajax/books/{library}`
- `ajax/category/616c6c626f6f6b73/{library}` (the hex encodes "allbooks")

When a book is imported into a Calibre root, Readarr uploads it to Calibre (Calibre decides the path) and then reads back the path. Languages are canonicalized to ISO-639-3 with `Languages/{2to3,by3,name_map}.json`. The fork mapped `"e"` to `eng` in name_map (`1db98a878`).

### Deep dive: naming tokens
`Core/Organizer/FileNameBuilder.cs`. `NamingConfig` has RenameBooks, ReplaceIllegalCharacters, ColonReplacementFormat, **StandardBookFormat**, **AuthorFolderFormat**. There is no book-folder format; the book folder is part of StandardBookFormat as a path. Tokens:
- Author: `{Author Name|NameThe|SortName|CleanName|NameFirstCharacter|Disambiguation}`
- Book: `{Book Title|TitleThe|TitleNoSub|TitleTheNoSub|CleanTitle|CleanTitleNoSub|Subtitle|SubtitleThe|CleanSubtitle|Disambiguation|Series|SeriesTitle|SeriesPosition}`
- Dates: `{Release Year}`, `{Release YearFirst}`, `{Edition Year}`
- Parts: `{PartNumber}`, `{PartCount}` (multi-file audiobooks)
- Quality: `{Quality Full|Title|Proper|Real}`
- MediaInfo: `{MediaInfo AudioCodec|AudioBitRate|AudioChannels|AudioSampleRate|AudioBitsPerSample}`
- Other: `{Release Group}`, `{Original Title}`, `{Original Filename}`, `{Custom Formats}`

Title and subtitle are split on ":".

### Deep dive: search criteria
`Core/IndexerSearch/Definitions/`:
- `AuthorSearchCriteria` (author only)
- `BookSearchCriteria` (BookTitle, BookYear, BookIsbn, Disambiguation; `BookQuery` = title with the subtitle and author stripped via `SplitBookTitle`)

Newznab/Torznab (`Core/Indexers/Newznab/NewznabRequestGenerator.cs`):
- when the caps advertise `book-search`: `t=book&author=…&title=…`
- otherwise: `t=search&q=` fallbacks

Commands: BookSearch, AuthorSearch, MissingBookSearch, CutoffUnmetBookSearch. Book-specific decision specs: `DiscographySpecification` and `EarlyReleaseSpecification`. The fork added Prowlarr-style `IndexerCapabilities`, `NewznabStandardCategory` and `IndexerFlag` classes to support MAM (`Core/Indexers/IndexerCapabilities*.cs`, `CategoryMapping.cs`).

---

## 4. Integrations (full lists)

**Indexers (10)** in `Core/Indexers/`:
- Newznab, Torznab, TorrentRss
- IPTorrents, Torrentleech, FileList, Gazelle
- Nyaa (anime tracker, mostly irrelevant for books)
- **MyAnonaMouse** (new in the fork, `90cac8195`, ported from Prowlarr). Settings: `mam_id` session cookie, search type, search languages, search in description/series/filenames, **freeleech wedge** use. It is the main private book and audiobook tracker.

**Download clients (17)** in `Core/Download/Clients/`:
- Aria2
- Torrent/Usenet Blackhole
- Deluge
- Download Station (torrent and usenet)
- Flood
- **Hadouken** (dead)
- NZBGet
- **NzbVortex** (dead)
- **Pneumatic** (Kodi add-on, dead)
- qBittorrent: the fork added **API-key auth for qBit 5.x** (`8bcbde5a6`) and fixed the qBit 5.2 HTTP 204 login (`ad3938e28`)
- rTorrent
- SABnzbd
- Transmission
- uTorrent
- **Vuze** (dead)

**Import lists (7)** in `Core/ImportLists/`:
- **Goodreads Bookshelf** (OAuth): ⚠ dead or at risk. OAuth request signing goes through `https://auth.servarr.com/v1/goodreads/sign` (Servarr infrastructure), and Goodreads closed its API in 2020.
- **Goodreads Owned Books** (OAuth): ⚠ same.
- **Goodreads Lists** (Listopia XML with a hardcoded key): ⚠ at risk.
- **Goodreads Series** (series XML with a hardcoded key): ⚠ at risk. The fork paginated it.
- **Hardcover** (new in the fork, `967b20864`, `f45af798b`, `fda6f3cb1`):
  - GraphQL at `https://api.hardcover.app/v1/graphql` with a personal API key
  - picks lists or reading statuses (Want to Read, Currently Reading, Read, Paused, DNF, Ignored) for any username
  - it maps Hardcover ids into the `*GoodreadsId` fields, so it **only works correctly with the hardcover metadata image**
- **LazyLibrarian** (imports from an LL instance).
- **Readarr** (syncs from another Readarr/Bookshelf instance by profile or tag; this is how the ebook and audiobook instance pair is kept in sync).

**Notifications (25)** in `Core/Notifications/`:
- Apprise, CustomScript, Discord, Email, Gotify, Mailgun, Notifiarr, Ntfy
- **Goodreads Bookshelf** and **Goodreads Owned Books**: add to a shelf on import. ⚠ Dead or at risk (OAuth via auth.servarr.com).
- **Join** (dead service)
- **Kavita** (triggers a library scan)
- Plex Server (with plex.tv PIN auth)
- Prowl, PushBullet, Pushover, SendGrid, Signal, Simplepush, Slack
- **Subsonic** (audiobook library refresh)
- **Synology Indexer**
- Telegram
- **Twitter** (OAuth1 consumer keys; the API is now paid, effectively dead)
- Webhook

Not present, compared with Sonarr/Radarr: Emby/Jellyfin, Kodi, Trakt, Pushcut. The book servers people actually use, Audiobookshelf and Calibre-Web, are **not** integrated.

**Metadata consumers:** none. The `Core/Extras/Metadata` framework is still there but has zero implementations. OPF/cover sidecars are not written, and Calibre fills that role.

**Calibre:** the content server, configured per root folder (§3).

**Metadata sources:** rreading-glasses (bookinfo.pro, softcover or hardcover) and www.goodreads.com legacy XML (§3).

---

## 5. External APIs / interfaces

- **REST `/api/v1`** only (`src/Readarr.Api.V1`, `openapi.json` "Readarr" 1.0.0). Auth uses an API key (`X-Api-Key` header or `apikey` query param). There is a SignalR hub at `/signalr/messages`. **Prowlarr's "Readarr" application integration speaks this API unchanged.**
- Controllers, grouped:
  - Library: `author`, `author/lookup`, `author/editor`, `book`, `book/lookup`, `book/editor`, `edition`, `series`, `bookfile`, `bookshelf` (bulk author and book monitoring), `search` (combined author/book lookup), `rename`, `retag`, `manualimport`, `mediacover`, `parse`.
    - The fork's `BookController` update now re-points BookFiles to the newly monitored edition (`3ac41f7e3`).
  - Acquisition: `indexer`, `indexerflag`, `release`, `release/push`, `downloadclient`, `queue`, `queue/details`, `queue/status`, `blocklist`, `history`, `wanted/missing`, `wanted/cutoff`.
  - Profiles: `qualityprofile` (+schema), `metadataprofile` (+schema), `delayprofile`, `releaseprofile`, `customformat`, `qualitydefinition`.
  - Config: `config/{host,ui,naming,mediamanagement,downloadclient,indexer,metadataprovider,development}`. `metadataprovider` holds tag-writing options. `development` holds the **MetadataSource URL**.
  - Lists and connections: `importlist`, `importlistexclusion`, `notification`, `metadata` (consumer framework, empty).
  - System: `system/status`, `system/task`, `system/backup`, `health`, `log`, `log/file`, `log/file/update`, `update`, `command`, `diskspace`, `filesystem`, `rootfolder`, `remotepathmapping`, `tag`, `tag/detail`, `language`, `localization`, `customfilter`, `calendar` + iCal feed (`/feed/v1/calendar/readarr.ics`).
- **Webhook events** (`Core/Notifications/Webhook/WebhookBase.cs`): Grab, Download, Rename, **Retag**, BookDelete, BookFileDelete, AuthorAdded, AuthorDelete, Health, ApplicationUpdate, Test. Payloads use author/book/edition/bookfile shapes.
- **Custom script env vars** (`Core/Notifications/CustomScript/CustomScript.cs`), prefix `Readarr_`:
  - `Readarr_EventType`
  - `Readarr_Author_{Id,Name,Path,GRId|GoodreadsId,DeletedFiles}`
  - `Readarr_Book_{Id,Title,GRId|GoodreadsId,ReleaseDate,DeletedFiles}`
  - `Readarr_Release_{BookCount,BookReleaseDates,BookTitles,BookIds,GRIds,Title,Indexer,Size,Quality,QualityVersion,ReleaseGroup,IndexerFlags}`
  - `Readarr_Download_{Client,Client_Type,Id}`
  - `Readarr_AddedBookPaths`, `Readarr_DeletedPaths`, `Readarr_DeletedDateAdded`, `Readarr_Delete_Reason`
  - `Readarr_BookFile_{Id,Path,Quality,QualityVersion,ReleaseGroup,SceneName,Edition_Id,Edition_Name,Edition_GoodreadsId,Edition_Isbn13,Edition_Asin}`
  - `Readarr_Tags_{Diff,Scrubbed}` (retag event)
  - `Readarr_Health_Issue_{Level,Message,Type,Wiki}`
  - `Readarr_Update_{Message,NewVersion,PreviousVersion}`
  - Naming is inconsistent: both `GRId` and `GoodreadsId` spellings are used. On the hardcover image these "GR" ids are actually Hardcover ids.

---

## 6. Background jobs

`Core/Jobs/TaskManager.cs:60-130`:

| Task | Interval |
|---|---|
| RefreshMonitoredDownloads | 1 min |
| MessagingCleanup | 5 min |
| ImportListSync | 5 min (each list has its own minimum interval) |
| RssSync | config, default 15 min |
| ApplicationUpdateCheck | 6 h |
| CheckHealth | 6 h |
| **RefreshAuthor** | 24 h |
| **RescanFolders** | 24 h |
| Housekeeping | 24 h |
| Backup | config |

`ShouldRefreshAuthor` (`Core/Books/Utilities/ShouldRefreshAuthor.cs`):
- always refresh if last sync was more than 30 days ago
- never refresh if it was less than 12 hours ago
- refresh "Continuing" authors every 2 days
- refresh if the author's newest book is less than 30 days old

With the fork's caching removed, each refresh is a live call to bookinfo.pro. Other commands: BulkRefreshAuthor/Book, RefreshBook, MoveAuthor/BulkMove, RenameAuthor/RenameFiles, **RetagAuthor/RetagFiles**, DownloadedBooksScan, CleanUpRecycleBin, the search commands, ManualImport. The `RootFolderWatchingService` filesystem watcher (config `WatchLibraryForChanges`) triggers rescans.

---

## 7. Configuration surface (book-specific)

**`ConfigService` keys (DB `Config` table):**
- **MetadataSource** (URL, env fallback `METADATA_URL`)
- **WriteAudioTags** (No / NewFiles / AllFiles / Sync), **ScrubAudioTags**
- **WriteBookTags** (NewFiles / AllFiles / Sync; Calibre only), **UpdateCovers**, **EmbedMetadata** (Calibre `embed_metadata`)
- **WatchLibraryForChanges**
- **RescanAfterRefresh** (Always / AfterManual / Never)
- **AutoUnmonitorPreviouslyDownloadedBooks**
- CreateEmptyAuthorFolders
- **AllowFingerprinting** (dead)
- the rest (retention, CDH, recycle bin, permissions, UI) is shared

**Environment variables (fork):**
- `METADATA_URL`
- `HARDCOVER=true` (lowers default MinPopularity to 50)
- standard LSIO `PUID/PGID`

Per root folder: default quality, metadata and monitor profiles, tags, and Calibre settings. Metadata profiles: §2. Naming: §3. Removed by the fork: `AnalyticsEnabled`, `FilterSentryEvents` (config.xml/LogOptions).

---

## 8. Platform/infrastructure concerns (divergences only)

- **.NET 6 (EOL)** while Sonarr is on 10 and Radarr/Prowlarr on 8. It still has the old `Mono.Posix` and a custom SQLite build (`System.Data.SQLite.Core.Servarr`), and Npgsql 7.
- **Linux-only builds** (x64 and musl arm64). The Windows/macOS projects and installers remain in the tree unused.
- **No Sentry and no analytics** (`5319d12b9`). Piwik and the donations UI were removed from the frontend.
- The update check still calls `https://readarr.servarr.com/v1/update/{branch}` (`src/NzbDrone.Common/Cloud/ReadarrCloudRequestBuilder.cs`). This is a dead or retired Servarr endpoint. Docker sets `UpdateMethod=docker`, so it is mostly moot.
- Leftover links point to `wiki.servarr.com/readarr` and `auth.servarr.com`.
- The Docker image ships a LinuxServer s6 `init-deprecate` script that prints "This image is deprecated", copied from LSIO's retired Readarr image (`docker/root/etc/s6-overlay/s6-rc.d/init-deprecate`).
- **Third DB (cache.db)** for HTTP response caching, now mostly bypassed.
- BookFile paths are **absolute**, and Calibre can own the file layout. That makes path moves and remote mappings different from Sonarr/Radarr.
- The frontend is still React 17 + Redux + mostly JS (≈Prowlarr era). It has not had the Sonarr v5 react-query/TS rewrite.

---

## 9. Interactions with other *arr apps

- **Prowlarr**: Prowlarr's `Readarr` application sync pushes indexers to `/api/v1/indexer`, and it works unchanged because the API is intact. Default Newznab categories are 3030 (Audiobook), 7020 (EBook) and 8010 (`Core/Indexers/Newznab/NewznabSettings.cs:56`). Bookshelf also added native MAM, so Prowlarr is optional for the main book tracker (a README goal).
- **Readarr/Bookshelf ↔ Readarr**: the "Readarr" import list keeps a paired ebook instance and audiobook instance in sync.
- **LazyLibrarian** import list.
- **Hosted services:**
  - **rreading-glasses** at `api.bookinfo.pro` / `hardcover.bookinfo.pro` (third-party, critical)
  - www.goodreads.com legacy XML API
  - api.hardcover.app (lists)
  - auth.servarr.com (Goodreads OAuth signing, legacy)
  - readarr.servarr.com (updates, legacy)
- No Profilarr support: Profilarr targets Sonarr/Radarr only.
- Downstream apps that read the Readarr API: Kavita, and third-party tools such as Readarr-compatible request apps.

---

## 10. Outdated / legacy candidates

| Item | Evidence |
|---|---|
| Goodreads API (keys, XML endpoints) | `GoodreadsProxy.cs:32-34,76-78` hardcode reversed dev keys. Goodreads stopped issuing API keys and began retiring the API in Dec 2020. |
| Goodreads OAuth import lists and notifications (Bookshelf, OwnedBooks) | `ImportLists/Goodreads/GoodreadsSettingsBase.cs:27-30`, `Notifications/Goodreads/GoodreadsSettingsBase.cs:26-29`. They depend on `auth.servarr.com` plus Goodreads OAuth, both unmaintained. |
| Goodreads ID as the universal key | `ForeignAuthorId`/`ForeignBookId` hold Goodreads *or* Hardcover ids depending on the image. Custom-script vars still say `GRId`/`GoodreadsId`. |
| `api.bookinfo.club` | Removed by the fork (the original Servarr metadata server, shut down). |
| `readarr.servarr.com` updates, `wiki.servarr.com` links | `ReadarrCloudRequestBuilder.cs:15`, `HealthCheck.cs:42`, `ProviderResource.cs:42`. |
| Sentry / analytics | Removed (`5319d12b9`). |
| .NET 6 runtime | `mise.toml`, all csproj `net6.0`. |
| Lidarr leftovers | `AllowFingerprinting` (no consumers), `ReportMusicTitleRegex`, `recording_id`/`track_*` distance weights, Album/Artist class names, `DistanceCalculator.VariousAuthorIds` MusicBrainz GUID. |
| Extras/Metadata consumer framework | No implementations (deleted at Readarr 0.1). |
| Dead download clients | Hadouken, Vuze, NzbVortex, Pneumatic. |
| Dead notifications | Join, Twitter (paid API), Goodreads ×2. |
| `TorcacheHttpInterceptor` | Dead torrent-cache service. |
| Nyaa indexer | Anime tracker, little relevance to books. |
| Calendar iCal ProductId `-//readarr.com//` | Cosmetic. |
| `Books/output.txt` | Empty stray file. |
| Win/macOS installers | `distribution/` unused, since the fork builds Linux only. |
| Metadata caching (cache.db, `ICachedHttpResponseService`) | Effectively disabled by the fork but still wired in. |

---

## 11. Rewrite notes

**Book-specific logic to preserve**
1. **The work/edition split with one monitored edition per book.** Also needed:
   - `AnyEditionOk`
   - BookFile → Edition linkage, which the fork fixed on edition change
   - `AuthorMetadata` shared with non-library co-authors
2. **Metadata profiles** (`MetadataProfileService.FilterBooks`) decide which works and editions exist at all. This is the main defence against author catalogues full of junk, translations and box sets:
   - popularity threshold
   - future-release bypass
   - part/set regex
   - secondary-series skip
   - language allowlist (ISO-639-3 + "null")
   - min pages
   - ignored terms
   - ISBN/ASIN required
3. **Identification pipeline** (Candidate → Distance → CloseMatch threshold):
   - candidate order: ISBN → ASIN → id → author+title → title → author
   - the weight table
   - the ebook/audio format bias lists
   - "(unabridged)" stripping
   - multi-file audiobook grouping and Part/PartCount
   - the fork's "don't re-search unmatched files" stamp
   - port the `NzbDrone.Core.Test` identification fixtures (`IdentificationServiceRemoteSearchFixture`, etc.) as golden tests
4. **Tag readers.** Needed: EPUB OPF (including `calibre:series`), MOBI/AZW3 EXTH (ISBN/ASIN), the PDF Info dictionary, and audio via TagLib, including m4b chapters/atoms. Go has libraries for audio tags (dhowden/tag is read-only) and EPUB is simple zip+XML. MOBI EXTH has to be ported, about 500 lines (`Core/MediaFiles/AzwTag/`).
5. **Calibre content-server client.** Covers add/convert/set-fields/embed/delete and per-root library settings. It is a niche feature but loyal users rely on it.
6. **Search:** `t=book&author=&title=` with fallbacks; title/subtitle splitting (`SplitBookTitle`); Discography (author pack) and EarlyRelease specs.
7. **Naming tokens** (§3), including series title and position and the part tokens.
8. **Integration points worth adding** (none exist today): Audiobookshelf library scan, Hardcover lists (keep), MAM native (keep; it is the key book tracker).

**Viability of a free book-metadata source for a monolith**
- The only drop-in option is **rreading-glasses**: it implements the exact BookInfo API that Readarr expects, is GPL-3, and is written in **Go**. It can be (a) called as an external service (the bookinfo.pro community instances, bus-factor risk) or (b) **embedded or self-hosted** as a sidecar. Its needs are small: Postgres, plus a Hardcover token for the Hardcover backend.
  - For a Go monolith, (b) is attractive. Its resolvers could be vendored as a library behind our own `MetadataProvider` interface. Check GPL-3 compatibility with our license.
- **Hardcover** has an official GraphQL API (`api.hardcover.app/v1/graphql`, free personal token, rate-limited). The data is good but it is a single company's service, and its ids are not Goodreads ids.
- **OpenLibrary** (`openlibrary.org` works/editions/authors JSON, plus bulk dumps under CC0) is the most durable free option: a work/edition model that maps 1:1, ISBN lookups, and no key. The drawbacks are sparse popularity data (there are no ratings counts that would make MinPopularity meaningful), noisy series data, and weak audiobook/ASIN coverage.
- **Goodreads** has no official API. Using it means scraping, which is fragile and against its ToS. Avoid it as a primary source.
- **Audible/Audnexus** (`api.audnex.us`) is the de-facto free source for audiobook ASIN, narrator and chapter data (used by Audiobookshelf). It is worth adding for audiobooks, which Readarr models poorly: it has no narrator field.
- Recommendation: define our own provider interface along the lines of BookInfo (author, work, edition bulk, search, changed). Default to Hardcover or an embedded rreading-glasses, add OpenLibrary as a keyless fallback and for ISBN resolution, and use Audnexus for audiobook enrichment. **Never persist a provider id as "the" id.** Store an external-id map per provider instead, to avoid the Goodreads-versus-Hardcover DB incompatibility that Bookshelf has.

**Accidental complexity to drop:**
- the Lidarr music leftovers
- the cache DB
- the empty Extras/Metadata framework
- polling loops (60 × 2s) for the metadata server's asynchronous "refreshing" responses
- the `useCache` flags threaded through every call
- separate instances for ebook versus audiobook. Model a media type per edition or file and per quality profile, and allow both formats of a book in one library. This is the top README TODO.

### Divergence from Sonarr/Radarr/Readarr
**Versus Sonarr/Radarr (domain):**
- The hierarchy is 3 levels (Author → Book(work) → Edition → BookFile) rather than Series → Episode or Movie → MovieFile.
- Files attach to editions and use absolute paths.
- Metadata profiles are an extra per-author profile axis.
- There are two media families (text and audio) in one quality ladder.
- Identification uses a tag-driven, beets-style distance matcher inherited from Lidarr, not release-name parsing.
- Tag writing is available (audio directly, ebooks via Calibre).
- An external library manager (Calibre) can own file placement.
- Import lists are reading-site shelves (Goodreads, Hardcover).
- No metadata consumers, and no Kodi/Emby/Jellyfin/Trakt.
- API v1 only.
- .NET 6 and React 17/Redux.

**Versus upstream Readarr (the 32 fork commits):**
1. Pluggable metadata URL (UI, env or default) pointing at rreading-glasses `api.bookinfo.pro`/`hardcover.bookinfo.pro`. Search moved from goodreads.com to that server.
2. Two image flavours (softcover = Goodreads-backed, hardcover = Hardcover-backed, `HARDCOVER` env).
3. Metadata caching disabled.
4. Search errors propagate, plus the LastRemoteSearchTime stamp (mig 041).
5. Match threshold loosened from 0.20 to 0.50.
6. Native MyAnonaMouse indexer with Prowlarr-style capability and category classes.
7. Hardcover import list (lists and reading statuses, any username).
8. Goodreads series pagination.
9. qBittorrent API-key auth and 5.2 login fix.
10. Sentry and analytics removed.
11. Null-safety fixes (series, author id, edition switch, AudioTag diff, cover URLs, "refresh all").
12. Connection-string cleansing in logs.
13. Linux-only Docker builds (LSIO base).
