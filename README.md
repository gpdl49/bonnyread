# bonnyread

bonnyread is a planned single application that replaces the most-used *arr apps with one Go server and one set of clients.

**Status:** specification. No application code exists yet.

## What it replaces

| Area | Reference apps |
|---|---|
| TV, movies | Sonarr, Radarr |
| Music | Lidarr |
| Books and audiobooks | Bookshelf (Readarr fork) |
| Adult | Whisparr v2 and v3 (Eros) |
| Comics | Kapowarr |
| Indexers | Prowlarr, Jackett, autobrr |
| Quality profiles and custom formats | Profilarr, Recyclarr |
| Requests | Seerr (Overseerr/Jellyseerr) |
| Subtitles | Bazarr |
| Download handling | Cleanuparr, Unpackerr |

The reference apps evolved over a decade into a set of separate services that sync with and poll each other. bonnyread keeps what they achieve but not how they achieve it: one data model, one pipeline and no inter-app sync layers. See [D19](docs/decisions.md).

## Shape

- A headless **Go core server** holds all business logic and exposes a REST API (`/api/v1`, OpenAPI), live events (SSE) and Prometheus metrics.
- **Clients** share one SvelteKit UI: the web UI served by the core, Tauri desktop apps, and later Tauri mobile apps.
- **Deployments** range from a single-user desktop install (the core runs as an OS service with SQLite) to a Kubernetes HA cluster (several replicas on PostgreSQL).
- Data access uses **ent** with Atlas migrations. Profiles use the Profilarr/Dictionarry PCD format. Plug-ins are WebAssembly.

## Repository layout

| Path | Contents |
|---|---|
| [`docs/spec/`](docs/spec/) | **The specification.** Start at [`00-overview.md`](docs/spec/00-overview.md). |
| [`docs/decisions.md`](docs/decisions.md) | Architecture decisions D1–D20 and the reasoning behind each. |
| [`docs/design/`](docs/design/) | Design rationale per area, plus the consolidated design review ([`open-issues.md`](docs/design/open-issues.md)). |
| [`docs/analysis/`](docs/analysis/) | Analysis of each reference app, and the cross-app [comparison](docs/analysis/comparison.md). |
| [`docs/verification/`](docs/verification/) | Independent verification of the analysis claims. |
| `submodules/` | The reference apps' source code, as git submodules, used for analysis only. |

To fetch the reference sources:

```sh
git submodule update --init
```

The `whisparr` and `whisparr-eros` submodules are large, so a shallow clone is recommended:

```sh
git submodule update --init --depth 1
```

## License

[AGPL-3.0](LICENSE). The reference apps keep their own licenses; see [`docs/analysis/comparison.md`](docs/analysis/comparison.md) §5.4 and [D5](docs/decisions.md).
