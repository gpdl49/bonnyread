# Licensing verification

Scope: every licensing claim in `docs/analysis/comparison.md` §0 (License column), §5.4 and §6, and in `docs/decisions.md` D5 and D8. Checked on 2026-09-27 against the submodule checkouts (jackett `6556904`, autobrr `cd78a70b`, prowlarr `12c327808`), `gh api`, a shallow clone of `Prowlarr/Indexers` (`223d3b1`, 2026-09-26), and the Jackett wiki.

**This is not legal advice.** It checks facts (what the license files, headers and metadata say) and gives a high-level reading of the FSF's published guidance. D5 already says to get legal review before the first public release, and that still applies.

Verdicts: **CONFIRMED** = the claim matches the evidence. **CORRECTED** = the claim is wrong or materially incomplete. **UNVERIFIABLE** = the evidence can't settle it.

## 1. Per-app licenses (comparison.md §0, §5.4; decisions.md D5 table)

| Claim | Verdict | Evidence | Correction |
|---|---|---|---|
| Sonarr GPL-3.0 | CONFIRMED | `submodules/sonarr/LICENSE.md:1-5` (GPLv3 text); `gh api repos/Sonarr/Sonarr` → `GPL-3.0` | – |
| Radarr GPL-3.0 | CONFIRMED | `radarr/LICENSE:1-2`; gh → `GPL-3.0` | – |
| Lidarr GPL-3.0 | CONFIRMED | `lidarr/LICENSE.md:1-2`; gh → `GPL-3.0` | – |
| Bookshelf GPL-3.0 | CONFIRMED | `bookshelf/LICENSE.md:1-2`; gh `pennydreadful/bookshelf` → `GPL-3.0`, fork of `Readarr/Readarr` | – |
| Whisparr (v2) GPL-3.0 | CONFIRMED | `whisparr/LICENSE:1-2`; gh → `GPL-3.0` | – |
| Whisparr v3 (Eros) GPL-3.0 | CONFIRMED | `whisparr-eros/LICENSE.md:1-5` (Markdown GPLv3); gh `Whisparr/Whisparr-Eros` → `GPL-3.0` | – |
| Kapowarr GPL-3.0 | CONFIRMED | `kapowarr/LICENSE:1-2`; gh `Casvt/Kapowarr` → `GPL-3.0` | – |
| Prowlarr GPL-3.0 | CONFIRMED | `prowlarr/LICENSE:1-2`; gh → `GPL-3.0` | See §3: the Cardigann engine inside it descends from GPL-2.0 code. |
| Jackett GPL-2.0-only | CONFIRMED (as the safe reading) | See §2 | – |
| autobrr GPL-2.0-or-later | CONFIRMED | 353 of 379 `.go` files and 168+ TS files carry `SPDX-License-Identifier: GPL-2.0-or-later` (e.g. `autobrr/cmd/autobrr/main.go:2`, `internal/irc/handler.go:2`, `internal/announce/announce.go:2`, `pkg/torznab/caps.go:2`). No other SPDX id appears anywhere. `README.md:435`: "GNU General Public License v2 or later (GPLv2+)". `LICENSE` is the GPLv2 text. | – (26 `.go` files have no header; the README grant covers them) |
| Profilarr AGPL-3.0 | CONFIRMED | `profilarr/LICENSE:1-2` (GNU AFFERO GPL v3); it is the only license file; gh → `AGPL-3.0` | – |
| Recyclarr MIT | CONFIRMED | `recyclarr/LICENSE:1-3` "MIT License, Copyright (c) 2021 Robert Dailey"; gh → `MIT` | – |
| Seerr MIT | CONFIRMED | `seerr/LICENSE:1-3` "MIT License, Copyright (c) 2020 sct"; gh → `MIT` | – |
| Bazarr GPL-3.0 | CONFIRMED | `bazarr/LICENSE:1-2`; gh → `GPL-3.0` | Note: the vendored `libs/` (229 entries) have mixed licenses, including GPL (`unidecode`, `auditok`, `fese`), LGPL (`guessit`, `pycountry`) and MPL (`bidict`, `certifi`). This only matters if Python code is ported. |
| Cleanuparr GPL-3.0 | CONFIRMED | `cleanuparr/LICENSE:1-2`; gh → `GPL-3.0` | – |
| Unpackerr MIT | CONFIRMED | `unpackerr/LICENSE:1-4` (MIT text, © 2018-2022 David Newhall II, 2023-2026 Go Lift); gh → `MIT` | – |

Nested license files in the Servarr forks are all for small third-party pieces: `AutoMoq` (MIT), `OAuth`, test media, and `Expansive` (Ms-PL, Sonarr and Whisparr only). None of them changes the top-level license.

## 2. Jackett: GPL-2.0-only

| Claim | Verdict | Evidence | Correction |
|---|---|---|---|
| LICENSE is GPLv2 with no "or later" grant | CONFIRMED | `jackett/LICENSE:1-2` "GNU GENERAL PUBLIC LICENSE Version 2, June 1991". The only "any later version" text is §9 (`LICENSE:243-245`) and the sample notice in the How-to-Apply appendix (`LICENSE:297-299`). Neither is a grant. | – |
| (jackett.md) the file is the "verbatim" GPLv2 text | CORRECTED (minor) | `LICENSE:293-294` has GitHub-template placeholders `{description}` / `Copyright (C) {year} {fullname}`, and `LICENSE:4-5` uses the old FSF street address | It is GitHub's GPL-2.0 template. The operative terms are identical to GPLv2, but the text isn't byte-verbatim FSF. The conclusion doesn't change. |
| LICENSE unchanged since first commit | CONFIRMED | `git log --follow -- LICENSE` → only `cdc829cec` "Initial commit" (2015-04-13). The repo is not shallow (19,042 commits). | – |
| README states no license or version | CONFIRMED | `README.md:1533-1541` "License and Credits" names no license. `git log -i -G"gpl\|general public" -- README.md` → no hits in any version. | – |
| No source headers | CONFIRMED | `grep -rIl -i "later version\|GPL\|General Public\|SPDX\|PackageLicense"` → only `LICENSE`, plus false hits inside base64 in `src/Jackett.Tray/Main.resx:5061,10147`. `git log -S"GPL"` over `*.cs/*.md/*.csproj` → none. | – |
| No PackageLicenseExpression | CONFIRMED | No license element in any `*.csproj` or `src/Directory.Build.props`. `src/Jackett.Server/Jackett.Server.csproj:76` only copies `LICENSE` to the output. `Installer.iss` has no license entry. | – |
| No CONTRIBUTING/NOTICE/COPYING grant | CONFIRMED | `CONTRIBUTING.md`, `SECURITY.md` and `.github/` have no license text. No NOTICE, COPYING or AUTHORS files exist. | – |
| Wiki | CONFIRMED (no grant) | Cloned `Jackett/Jackett.wiki.git` (18 pages): no match for `licen\|gpl\|later version`. | – |
| GitHub detection | CONFIRMED | `gh api repos/Jackett/Jackett/license` → `spdx_id: "GPL-2.0"` (the deprecated id, which doesn't say "-only" or "-or-later"). Third-party packaging: winget manifest `Jackett.Jackett.locale.en-US.yaml` → `License: GPL-2.0`. | – |
| Treat as GPL-2.0-only | CONFIRMED as the conservative reading, with a caveat | GPLv2 §9 (`LICENSE:246-248`): "If the Program does not specify a version number of this License, you may choose any version ever published". Jackett has no notice at all, so someone could argue that no version was "specified" and that GPLv3 is therefore available. The FSF FAQ [#LicenseCopyOnly](https://www.gnu.org/licenses/gpl-faq.html#LicenseCopyOnly) says a bare license file leaves even the basic grant uncertain. The more natural reading is that shipping only the v2 text specifies v2. Relying on the §9 argument across hundreds of contributors with no CLA is not prudent. | Keep "GPL-2.0-only" as the planning assumption, but note that it is an interpretation of an ambiguous situation, not an explicit "-only" statement by Jackett. |
| Jackett is the "original Cardigann C# port" | CONFIRMED, with a nuance | `CardigannIndexer.cs` was added in `21cffe2d3` (2016-10-27, "Add basic support for Cardigann definitions (#571)"). The Go project `cardigann/cardigann` was created 2016-07-15. | Jackett's engine is a C# port of MIT code. Any code that was literally translated keeps its MIT permission; only Jackett's own additions are purely GPL-2.0. In practice it can't be separated out, so this doesn't change the plan. |

## 3. Cardigann lineage (Prowlarr engine, Prowlarr/Indexers, Go Cardigann)

| Claim | Verdict | Evidence | Correction |
|---|---|---|---|
| Prowlarr/Indexers has no license | CONFIRMED | `gh api repos/Prowlarr/Indexers/license` → HTTP 404. The repo root holds only `.editorconfig .gitattributes .github .gitignore .yamllint CONTRIBUTING.md README.md VERSIONS definitions requirements.txt scripts`. `README.md` and `CONTRIBUTING.md` contain no license text. | Missing license ≠ "no license applies". The copied files are still GPL-2.0 from Jackett's authors (GPLv2 §6: each recipient gets a licence from the original licensor). Omitting the license text is arguably a GPLv2 §1 compliance defect on Prowlarr's side. Files that Prowlarr authored or modified have no stated licence. |
| Its definitions are copies of Jackett's | CONFIRMED (quantified) | `PIDX/README.md` "Definitions from Jackett": "synced upstream with Jackett via the indexer-sync script". `scripts/indexer-sync-v2.sh:4` "keep Prowlarr/Indexers up to date with Jackett/Jackett". A byte comparison of `definitions/v11/*` (557 files) against `jackett/src/Jackett.Common/Definitions/*.yml` (563) gave **504 byte-identical**, 53 differing (51 of them by ≤6 lines) and 1 not in Jackett. | – |
| Prowlarr's Cardigann engine is derived from Jackett's | CONFIRMED | (a) The same FIXME comment appears in both: `prowlarr/.../Cardigann/CardigannBase.cs:782` "FIXME: fix jackett header handling (allow it to specify the same header multiple times)" and `jackett/.../CardigannIndexer.cs:2194` (same text with the original typos "specifiy … multipe"). (b) 142 of 693 distinct non-trivial lines (>40 chars) in Prowlarr's `Cardigann/*.cs` appear verbatim in Jackett's `CardigannIndexer.cs`, including idiosyncratic identifiers such as `regexpreplace_regex`, `_LogicFunctionRegex.Match(template)` and `variables[prefix + ".AbsoluteUri"]`. (c) Engine introduced in Prowlarr commit `d041b152d` (2020-10-22, ta264) together with `DateTimeUtil.cs`, `ParseUtil.cs` and `StringUtil.cs`, which are Jackett utility class names. (d) No attribution to Jackett anywhere in Prowlarr's `README.md` or source. | – (the claim "the definitions data is effectively Jackett's" is also confirmed) |
| Original Go `cardigann/cardigann` is MIT | CONFIRMED | `gh api repos/cardigann/cardigann/license` → MIT, "Copyright (c) 2016 lxt". Last commit 2020-04-25; not archived. | – |
| Go Cardigann is a "license-clean base" (§6, D5-A) | **CORRECTED** | The MIT licence covers lxt's own files only. The engine imports **GPL** packages: `indexer/runner.go:28-29` imports `github.com/cardigann/releaseinfo` (**GPL-3.0**; its README says "Ported from Sonarr's parser code") and `github.com/cardigann/go-imdb-scraper` (**no licence**). `indexer/filters.go:15` imports `github.com/bcampbell/fuzzytime` (**GPL-3.0-or-later**, per its LICENSE header). All three are vendored under `vendor/`. | For a GPL-3.0 monolith this is compatible: MIT plus GPL-3.0(+) is fine. But the seed is not "permissive", so it doesn't rescue option C. `go-imdb-scraper` must be dropped or rewritten because it has no licence. When seeding, take only lxt's files and swap `releaseinfo` for the in-house parser. |

## 4. Reusable Go code (comparison.md §6; D5 and D8 "verify" items)

Module paths are from `submodules/autobrr/go.mod`. Licences were checked at the pinned tag where one exists.

| Claim | Verdict | Evidence | Correction |
|---|---|---|---|
| autobrr IRC/announce/torznab/newznab are GPL-2.0+ | CONFIRMED | SPDX headers, e.g. `internal/irc/handler.go:2`, `pkg/torznab/caps.go:2` | Path nit: the definitions live in `internal/indexer/definitions/` (118 YAML files plus a `deprecated/` directory), not `definitions/`. |
| `autobrr/go-*` clients: licence "Verify" | **RESOLVED — mixed** | `github.com/autobrr/go-qbittorrent v1.18.0` → **MIT** (© 2022 autobrr). `github.com/autobrr/go-rtorrent v1.12.0` → **MIT** (© 2015 Michael Robinson; fork of `mrobinsn/go-rtorrent`). `github.com/autobrr/go-deluge v1.4.0` → **GPL-2.0-or-later** (fork of `gdm85/go-libdeluge`; `LICENSE` is GPLv2, and `delugeclient.go:2-6` at `v1.4.0` says "either version 2 of the License, or (at your option) any later version"). `github.com/autobrr/go-torrent v1.1.1` → MIT. `github.com/autobrr/go-cache v1.0.0-rc1` → MIT. | go-deluge is **not** permissive. It is still compatible with a GPL-3.0 monolith because it is "or later". Update §6 and D5 to drop the "verify" item. |
| `rls` fork: licence "Verify (upstream believed MIT)" | **RESOLVED — MIT** | `go.mod`: `replace github.com/moistari/rls => github.com/autobrr/rls v0.8.1`. `autobrr/rls` at `v0.8.1` has `LICENSE` "The MIT License (MIT), Copyright (c) 2020-2022 m. jo istari". It is a fork of `moistari/rls`, which is also MIT. | Replace "Verify" with "MIT" in §6, D5 and D8. |
| Other autobrr infrastructure libraries are "various" | CONFIRMED | chi MIT, scs MIT, go-oidc Apache-2.0, zerolog MIT, modernc sqlite BSD-3-Clause (gh `cznic/sqlite`), ergochat/irc-go ISC, transmissionrpc MIT. Also note: `replace github.com/r3labs/sse/v2 => github.com/autobrr/sse/v2` is **MPL-2.0**. | All are GPL-3.0-compatible. MPL-2.0 needs file-level source availability for that package. |
| `golift.io/xtractr` is MIT | CONFIRMED | `gh api repos/golift/xtractr/license` → MIT, "Copyright (c) 2020-2026 Go Lift". Unpackerr pins `golift.io/xtractr v0.6.2-0.20260922231041-fec876ff1836` (`unpackerr/go.mod:27`). | – |
| xtractr's key dependencies | CONFIRMED permissive (not previously stated) | From `golift/xtractr/go.mod`: `nwaples/rardecode/v2` BSD-2-Clause; `bodgit/sevenzip` BSD-3-Clause (+ `bodgit/plumbing`, `bodgit/windows` BSD-3); `Unpackerr/iso9660` BSD-2; `golift.io/udf` BSD-3; `ulikunitz/xz` BSD-3; `therootcompany/xz` CC0-1.0; `klauspost/compress` BSD-3 (plus bundled MIT/Apache parts); `dsnet/compress` BSD-3; `pierrec/lz4` BSD-3; `andybalholm/brotli` MIT; `cavaliergopher/cpio` and `rpm` BSD-3; `peterebden/ar` MIT; `mewkiz/flac` Unlicense; `saintfish/chardet` MIT; `sshaman1101/dcompress` BSD-2 (stated only in its README; no LICENSE file); `stangelandcl/ppmd` MIT; `icza/bitio`, `spf13/afero`, `go4.org` Apache-2.0; `hashicorp/golang-lru/v2` **MPL-2.0** (indirect). | The dependency tree has no copyleft beyond MPL-2.0. |
| `blampe/rreading-glasses` is GPL-3.0, Go | CONFIRMED | `gh api repos/blampe/rreading-glasses` → licence `GPL-3.0`; languages `{"Go":239154,"Makefile":1426,"Dockerfile":551}`; module `github.com/blampe/rreading-glasses`; active (pushed 2026-09-25). No per-file headers, so it's unclear whether it is "-only" or "-or-later". | Either way it is compatible with a GPL-3.0 monolith. |
| `dlclark/regexp2` (D1) | CONFIRMED MIT | `gh api repos/dlclark/regexp2/license` → MIT, "Copyright (c) Doug Clark". `ATTRIB` says parts are ported from `dotnet/corefx` (MIT, © Microsoft). | Keep both notices when vendoring. |

## 5. Compatibility reasoning (D5)

| Claim | Verdict | Evidence | Correction |
|---|---|---|---|
| GPL-2.0-only code cannot be combined into a GPL-3.0 work | CONFIRMED | FSF GPL FAQ [#v2v3Compatibility](https://www.gnu.org/licenses/gpl-faq.html#v2v3Compatibility): "if you tried to combine code released under GPLv2 with code under GPLv3, you would violate section 6 of GPLv2. However, if code is released under GPL 'version 2 or later,' that is compatible with GPLv3". See also the [#AllCompatibility](https://www.gnu.org/licenses/gpl-faq.html#AllCompatibility) table. | – |
| So autobrr (GPL-2.0+) and go-deluge (GPL-2.0+) can go into GPL-3.0 | CONFIRMED | Same FAQ entry | – |
| "Prowlarr as the reference doesn't avoid the GPL-2.0-only problem" | CONFIRMED | §3 evidence of derivation. Prowlarr's GPL-3.0 label doesn't relicense Jackett-authored lines; FAQ #v2v3Compatibility implies Prowlarr itself has an unresolved v2/v3 mix. | – |
| Downloading definitions at runtime "arguably makes it mere aggregation" | CONFIRMED as arguable, and the stronger argument is a different one | (1) If the project never distributes the YAML (the user's instance fetches it), there is no distribution by the project to which the GPL could attach. The GPL governs copying and distribution, not private use ([#GPLRequireSourcePostedPublic](https://www.gnu.org/licenses/gpl-faq.html#GPLRequireSourcePostedPublic), GPLv2 §0 "Activities other than copying, distribution and modification are not covered"). (2) The better-fitting FAQ entry is [#IfInterpreterIsGPL](https://www.gnu.org/licenses/gpl-faq.html#IfInterpreterIsGPL): "The interpreted program, to the interpreter, is just data". This treats the YAML definitions as data for the engine, so their licence doesn't flow into the engine or the reverse. (3) [#MereAggregation](https://www.gnu.org/licenses/gpl-faq.html#MereAggregation) is about *separate programs* shipped together; it says "If the modules are included in the same executable file, they are definitely combined", so `go:embed`-ing the YAML weakens the argument. | Reword D5-A to rest mainly on (1) and (2) and to avoid embedding definitions in the binary. If the project ever mirrors or bundles the YAML, ship it as separate GPL-2.0 files with Jackett's LICENSE (GPLv2 §1) and keep them out of the executable. |
| The clean-engine plan (seed from MIT Go Cardigann, extend by black-box behaviour) avoids Jackett's copyright | UNVERIFIABLE (legal judgement) | Direction is sound: copying behaviour and data formats is generally not copying expression. But the v11 schema itself (`schema.json`, filter names and semantics) was authored in Jackett, and whether reproducing it infringes is a legal question. | Keep D5's "get legal review" step. Add the corrections from §3 about the seed's GPL-3.0 and unlicensed dependencies. |
| Profilarr AGPL-3.0 → option B | CONFIRMED | GPLv3 §13 lets GPLv3 code be combined with AGPLv3 code; the AGPL portions keep their network-use clause. | – |

## 6. Required edits (summary)

1. **comparison.md §6 / D5 / D8:** replace the "Verify" entries.
   - `autobrr/go-qbittorrent`, `go-rtorrent`, `go-torrent` and `go-cache` are MIT.
   - `autobrr/go-deluge` is **GPL-2.0-or-later** (compatible with GPL-3.0).
   - The `rls` fork (`github.com/autobrr/rls v0.8.1`, replacing `moistari/rls`) is **MIT**, and so is its upstream.
2. **comparison.md §6 "Cardigann engine seed" and D5-A:** it isn't entirely licence-clean.
   - The Go Cardigann engine imports `cardigann/releaseinfo` (GPL-3.0; a port of Sonarr's parser) and `bcampbell/fuzzytime` (GPL-3.0-or-later). Both are compatible with a GPL-3.0 monolith but are not permissive.
   - It also imports `cardigann/go-imdb-scraper`, which has **no licence** and must not be reused.
3. **D5 aggregation wording:** base the argument on non-distribution by the project and on FSF #IfInterpreterIsGPL (definitions are data). Don't rely on #MereAggregation, and never embed the YAML in the binary.
4. **§5.4 / D5 "No license" for Prowlarr/Indexers:** add that the files are still GPL-2.0 via Jackett. 504 of 557 v11 files are byte-identical to Jackett's. Only Prowlarr-authored changes have no stated licence.
5. **jackett.md §1.5 (minor):** the LICENSE is GitHub's GPL-2.0 template (with placeholders), not verbatim FSF text. The "-only" status is an interpretation (GPLv2 §9 ambiguity), not something Jackett states explicitly.
6. **§6 path nit:** autobrr's definitions are at `internal/indexer/definitions/`.

## Notes

- All other licence claims in the scope (the 16 app licences, autobrr GPL-2.0-or-later, xtractr MIT, rreading-glasses GPL-3.0/Go, regexp2 MIT, and v2-only/v3 incompatibility) are confirmed.
- Only autobrr among the 16 apps states "-or-later" explicitly (SPDX headers). None of the GPL-3.0 Servarr apps has per-file notices. That doesn't matter for a GPL-3.0 monolith.
- The Go dependency licences come from GitHub's licence detection plus the LICENSE text for anything reported as `NOASSERTION` or `NONE`. Transitive dependencies of go-qbittorrent and the other clients were not audited. Run `go-licenses` (or similar) over the final module graph before release.
