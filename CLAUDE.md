# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install
uv sync

# Run CLI
uv run rip <command>

# Tests
uv run pytest                          # all tests
uv run pytest tests/test_foo.py       # single file
uv run pytest tests/test_foo.py::fn   # single test

# Lint / format
uv run ruff check streamrip/
uv run ruff format streamrip/
```

## Architecture

Streamrip is an async Python CLI for downloading music from Qobuz, Tidal, Deezer, and Soundcloud. Entry point: `rip` → `streamrip/rip/cli.py` → `streamrip/rip/main.py`.

### Layer stack

```
CLI (click)         streamrip/rip/cli.py
Main orchestrator   streamrip/rip/main.py       — login, dispatch, concurrent rips
Client layer        streamrip/client/           — per-service API clients
Media layer         streamrip/media/            — Album, Artist, Playlist, Track, Label
Downloadable layer  streamrip/client/downloadable.py — raw byte fetch + decryption
Metadata layer      streamrip/metadata/         — structs + tagger
Config              streamrip/config.py         — TOML dataclasses, versioned
Database            streamrip/db.py             — SQLite, dedup by track ID
Converter           streamrip/converter.py      — ffmpeg wrapper
```

### Key abstractions

**`Client` (`client/client.py`)** — abstract base for each service.
- `login()`, `get_metadata(item_id, media_type)`, `get_downloadable(item_id, quality)`, `search(media_type, query)`
- Concrete: `QobuzClient`, `TidalClient`, `DeezerClient`, `SoundcloudClient`

**`Media` (`media/media.py`)** — abstract base for downloadable collections/tracks.
- Lifecycle: `rip()` → `preprocess()` → `download()` → `postprocess()`
- Concrete: `Album`, `Artist`, `Playlist`, `Label`, `Track`

**`Pending*`** — unresolved URL/ID that `resolve() → Media | None` after fetching metadata. Pattern used throughout to defer API calls.

**`Downloadable` (`client/downloadable.py`)** — handles per-service byte-level download.
- `_download(path, callback)` async, with progress callback
- Source-specific subclasses handle Blowfish/AES decryption (Deezer), HLS/M3U8 (Tidal, Soundcloud)

**`TrackMetadata` / `AlbumMetadata`** — immutable dataclasses with `from_<source>()` factory methods per service. Written to files via `streamrip/metadata/tagger.py`.

### Data flow

```
URL/ID input
  → parse_url() identifies source + media_type
  → Client.login() + get_metadata()
  → Pending*.resolve() → Album / Playlist / Artist / Label
  → Media.preprocess()  — mkdir, cover art download
  → Media.download()    — asyncio gather over Downloadable._download()
  → Media.postprocess() — tag, convert (ffmpeg), update SQLite DB
```

### Config

TOML file at `~/.config/streamrip/config.toml` (OS-specific path). Loaded into dataclasses in `config.py`. Quality scale 0–4 (128kbps MP3 → 24bit/192kHz, service-dependent max). Config is versioned; migrations run on load.

### Adding a new service

1. Add `Client` subclass in `client/`
2. Add `Downloadable` subclass in `client/downloadable.py`
3. Add `from_<source>()` factory on metadata structs in `metadata/`
4. Register in `rip/main.py` dispatch and `rip/parse_url.py`
5. Add config dataclass section in `config.py`
