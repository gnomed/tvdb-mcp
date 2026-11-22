# tvdb-mcp

[![License: GPL v3](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](LICENSE.txt)
[![Python 3.13+](https://img.shields.io/badge/python-3.13%2B-blue)](https://www.python.org/downloads/)

A [FastMCP](https://github.com/jlowin/fastmcp) server that exposes [TVDB](https://thetvdb.com/) v4 API endpoints as tools for AI assistants. This project is forked from [thetvdb-mcp-server](https://github.com/sesopenko/thetvdb-mcp-server) and heavily modified to be its own distinct MCP server.

TVDB responses include a large amount of metadata which can cause context bloat in AI applications. This server includes features to address this problem:

- The tools support optional `include_*` parameters (where applicable) that default to false, so only the data you explicitly request is returned.
- An optional `result_language` setting translates names and overviews to a single language and strips the translation maps from responses.

## Available Tools

| Tool | Description |
|---|---|
| `tvdb_search` | Search TVDB for series, movies, people, and companies. Returns up to 10 results by default. Response includes `results` array and `pagination` object with `offset`, `limit`, `total_items`, `has_next`. |
| `tvdb_get_movie` | Fetch a movie record. Optional include_* params (default false): characters, artworks, trailers, lists, tags, contentRatings, remoteIds, companies, awards, releases. |
| `tvdb_get_movie_by_slug` | Fetch a movie record by its slug (e.g. "the-lord-of-the-rings-the-fellowship-of-the-ring"). |
| `tvdb_get_series` | Fetch a series record. Optional include_* params (default false): characters, artworks, trailers, lists, tags, contentRatings, remoteIds, seasons, companies. |
| `tvdb_get_series_episodes` | Fetch episodes for a series. Supports pagination via `page` param, `season_type` ordering, and `lang` filtering. Returns `pagination` object with `page`, `page_size`, `total_items`, `has_next`, `has_prev`. |
| `tvdb_get_series_by_slug` | Fetch a series record by its slug (e.g. "doctor-who"). |
| `tvdb_get_season` | Fetch a season record by its numeric TVDB ID. Optional include_* params (default false): artworks, companies, episodes, trailers, tags. |
| `tvdb_get_artwork` | Fetch an artwork record by its numeric TVDB ID. |
| `tvdb_get_episode` | Fetch an episode record by its numeric TVDB ID. Optional include_* params (default false): awards, characters, companies, seasons, contentRatings, remoteIds, tags, trailers. |
| `tvdb_get_person` | Fetch a person record by their numeric TVDB ID. Optional include_* params (default false): awards, characters, biographies, remoteIds, tags. |

---

## Prerequisites

- **Docker** — for the Docker Compose deployment path
- **uv** — for the source deployment path (see [Installing uv](https://docs.astral.sh/uv/getting-started/installation/))

---

## Quick Start

### Option A — Docker Compose

1. Create a `docker-compose.yml`:

   ```yaml
   services:
     tvdb-mcp:
       image: ghcr.io/gnomed/tvdb-mcp:latest
       user: "nobody:nogroup"
       ports:
         - "8080:8080"
       volumes:
         - ./config.toml:/config/config.toml:ro
       restart: unless-stopped
   ```

   The `user: "nobody:nogroup"` line is recommended for security. The container can be run as any user id — adjust if your setup requires a specific one.

2. Copy the example config and edit it:

   ```bash
   cp config.toml.example config.toml
   ```

3. Start the server:

   ```bash
   docker compose up -d
   ```

### Option B — Run from Source

1. Install [uv](https://docs.astral.sh/uv/getting-started/installation/) if you haven't already.

2. Install dependencies:

   ```bash
   uv sync
   ```

3. Copy the example config and edit it:

   ```bash
   cp config.toml.example config.toml
   ```

4. Start the server:

   ```bash
   uv run python -m tvdb_mcp_server
   ```

---

## Security

This server has **no authentication** on its MCP endpoint. It is designed for LAN use only.

**Do not expose this server directly to the internet.**

If you need to access it remotely, place it behind a reverse proxy that handles TLS termination and access control. Configuring a reverse proxy is outside the scope of this project.

---

## Configuration

Create a `config.toml` in the working directory (or pass `--config <path>`):

```toml
[server]
host = "0.0.0.0"
port = 8080

[logging]
level = "info"

[tvdb]
api_key = "YOUR_API_KEY"
```

### [server]

| Key | Default | Description |
|---|---|---|
| `host` | `"0.0.0.0"` | Address for the MCP server to listen on. `0.0.0.0` binds all interfaces. |
| `port` | `8080` | Port for the MCP server to listen on. |

### [logging]

| Key | Default | Description |
|---|---|---|
| `level` | `"info"` | Log verbosity. One of: `trace`, `debug`, `info`, `warning`, `error`, `critical`. |

### [tvdb]

| Key | Default | Description |
|---|---|---|
| `api_key` | *(required)* | Your TVDB API key. Get one from the [TVDB dashboard](https://thetvdb.com/dashboard/account/apikey). |
| `pin` | `null` | Optional subscriber PIN for extended access. |
| `result_language` | `null` | Optional 3-character ISO 639-2 language code (e.g. `"eng"`). When set, names and overviews are translated to this language (if available) and the overviews/translations maps are stripped from responses. |

---

## Connecting an AI Application

This server uses the **Streamable HTTP** MCP transport. Clients communicate via HTTP POST with streaming responses — opening the endpoint in a browser will return a `Not Acceptable` error, which is expected.

Point your MCP-compatible AI application at the server's MCP endpoint:

```
http://<host>:<port>/mcp
```

For example, if the server is running on `192.168.1.10` with the default port:

```
http://192.168.1.10:8080/mcp
```

Consult your AI application's documentation for how to register an MCP server. Ensure it supports the Streamable HTTP transport (most modern MCP clients do).

---

## Known Limitations (TVDB API)

The following limitations are imposed by the TVDB API and/or the official Python library, not by this server:

- **Search index**: The TVDB search index includes series, movies, people, and companies. Searching for episodes, seasons, or artwork (`type="episode"`, etc.) returns empty results.
- **Search filtering**: The `year`, `primary_language`, and `country` parameters may broaden rather than filter results in some cases. The `remote_id` parameter (e.g. IMDB ID) is the most precise search method.
- **Episode filtering**: `tvdb_get_series_episodes` returns ALL episodes for a series with no server-side season or episode-number filtering. To get a specific season's episodes, use `tvdb_get_season` with `include_episodes=true`, or filter the results client-side by `seasonNumber`.
- **Season types**: `season_type="default"` works reliably. Other values (`aired`, `streaming`, `digital`, `alternate`) may return empty results for some series, depending on the TVDB database.
- **Absolute season type**: `season_type="absolute"` returns a different response structure (flat object, no `series` wrapper) and sets `seasonNumber` to 1 for all episodes.
- **Search result IDs**: Search returns prefixed IDs like `series-366924` or `person-7990590`. All `tvdb_get_*` tools expect pure numeric IDs — strip the prefix before use.
- **Episode images**: Some episodes return full URLs, others return relative paths (`/banners/...`). Handle both formats.

---

## Architecture

The server follows a clean separation of concerns:

| File | Purpose |
|---|---|
| `src/tvdb_mcp_server/tools.py` | Tool implementations — one function per tool, no framework coupling |
| `src/tvdb_mcp_server/server.py` | FastMCP wiring — tool registration and server entrypoint |
| `src/tvdb_mcp_server/tvdb_client.py` | TVDB API client — wraps the official library with rate limiting and an in-memory TTL cache |
| `src/tvdb_mcp_server/cache.py` | In-memory TTL cache for TVDB API responses |
| `src/tvdb_mcp_server/rate_limiter.py` | Async rate limiter — enforces 1 TVDB API call per second |
| `src/tvdb_mcp_server/config.py` | TOML config loading — typed dataclasses for `[server]`, `[logging]`, and `[tvdb]` sections |
| `src/tvdb_mcp_server/logging.py` | Structured JSON logger |

### Tool registration

Tools are not registered via `@mcp.tool()` decorators in `tools.py`. Instead, `server.py` registers each tool explicitly via `mcp.tool(name="...")(function)`, and `main()` loads configuration and initialises the shared TVDB client before starting the server.

### Adding a tool

1. Add a function to `src/tvdb_mcp_server/tools.py` with a Google-style docstring and full type annotations.
2. Register it in `src/tvdb_mcp_server/server.py` via `mcp.tool(name="...")(function)`.
3. Add a unit test in `tests/unit/`.
4. Add a row to the **Available Tools** table in this README.

---

## Contributing / Maintaining

See [MAINTAINERS.md](MAINTAINERS.md) for setup, development commands, and how to run tests.

---

## License

Forked from [thetvdb-mcp-server](https://github.com/sesopenko/thetvdb-mcp-server), copyright © Sean Esopenko.

This project is licensed under the [GNU General Public License v3.0](LICENSE.txt).

---

## Acknowledgement: Riding on the Backs of Giants

This project was developed with the assistance of multiple AI assistants, including Claude by Anthropic and various open-weight LLMs. A special thanks to [DavidAU](https://huggingface.co/DavidAU) and his work on open LLMs.

AI assistants are trained on enormous amounts of data — much of it written by the open-source community: the libraries, tools, documentation, and decades of shared knowledge that developers have contributed freely. Without that foundation, tools like this would not be possible.

In recognition of that debt, this project is released under the [GNU General Public License v3.0](LICENSE.txt). The GPL ensures that this code — and any derivative work — remains open source. It is a small act of reciprocity: giving back to the commons that made it possible.

To every developer who ever pushed a commit to a public repo, wrote a Stack Overflow answer, or published a package under an open license — thank you.
