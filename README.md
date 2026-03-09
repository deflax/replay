# Replay Service

Replay is a small Quart service that turns each top-level directory in a media library into an endless HLS channel.

Each discovered channel gets its own `ffmpeg` process, writes HLS output under `/tmp/hls/<channel>/`, and serves playlists and segments over HTTP.

## What it does

- Discovers channels from subdirectories under `LIBRARY_DIR`
- Recursively scans each channel directory for `.mp4`, `.mkv`, and `.ts` files
- Shuffles files and loops them forever through `ffmpeg`
- Serves HLS playlists at `/<channel>/playlist.m3u8`
- Polls the library for file or directory changes and restarts affected channels automatically

## Repository layout

- `app/main.py` - HTTP routes and service lifecycle
- `app/channel.py` - channel discovery, file watching, `ffmpeg` orchestration, and HLS generation
- `app/config.py` - environment variables, defaults, logging, and shared in-memory state
- `scripts/run.sh` - container entrypoint
- `Dockerfile` - container image build
- `docker-compose.yml` - compose service definition used by the original stack

## Requirements

- Python 3.11+
- `ffmpeg` and `ffprobe`
- A media library where each top-level directory is a channel

## Quick start with Docker

1. Copy the example environment file:

```bash
cp variables.env.example variables.env
```

Keep `LIBRARY_DIR` aligned with the container bind mount. The examples in this repo mount the host library at `/library`, so `variables.env` should keep `LIBRARY_DIR=/library` unless you also change the container mount target.

2. Create a library with at least one channel directory:

```text
data/
  library/
    movies/
      clip-01.mp4
      clip-02.mp4
    sports/
      match-01.ts
```

3. Build and run the container:

```bash
docker build -t replay-service .
docker run --rm \
  -p 8090:8090 \
  --env-file variables.env \
  -v "$(pwd)/data/library:/library:ro" \
  replay-service
```

4. Check the service:

```bash
curl http://localhost:8090/health
```

5. Open a playlist:

```text
http://localhost:8090/movies/playlist.m3u8
```

## Docker Compose

The checked-in `docker-compose.yml` still reflects its origin in a larger stack. It mounts the library and loads `variables.env`, but it does not publish a host port.

For a standalone setup, either:

- use the `docker run` command above, or
- add a `ports:` mapping before running `docker compose up --build`

Example:

```yaml
services:
  replay:
    ports:
      - "8090:8090"
```

## Local development

Install dependencies and run the app directly:

```bash
python -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
export LIBRARY_DIR="$(pwd)/data/library"
export REPLAY_PORT=8090
uvicorn main:app --app-dir app --host 0.0.0.0 --port "$REPLAY_PORT" --workers 1
```

Notes:

- local runs still require `ffmpeg` and `ffprobe` on your PATH
- keep `--workers 1`; channel state is stored in process memory

## Configuration

These environment variables are read in `app/config.py`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `LIBRARY_DIR` | `/library` | Root directory scanned for channel folders |
| `REPLAY_PORT` | `8090` | HTTP listen port |
| `REPLAY_SCAN_INTERVAL` | `60` | Seconds between library scans |
| `HLS_SEGMENT_TIME` | `4` | HLS segment duration in seconds |
| `HLS_LIST_SIZE` | `20` | Number of entries kept in the live playlist |
| `TRANSCODE_ENABLED` | `false` | Enables re-encoding for all channels when true |
| `VIDEO_BITRATE` | `4000k` | Video bitrate used when transcoding |
| `AUDIO_BITRATE` | `128k` | Audio bitrate used when transcoding |

## Channel layout

Each top-level directory inside `LIBRARY_DIR` becomes a channel name.

Example:

```text
/library
  /movies
    movie-a.mp4
    movie-b.mkv
  /sports
    game-01.ts
```

Behavior:

- `TRANSCODE_ENABLED=false` keeps copy mode enabled
- `TRANSCODE_ENABLED=true` forces re-encoding through `libx264` and AAC

Copy mode is the default and is the cheapest option, but it only keeps files that are compatible for concat playback. The service uses `ffprobe` to compare stream signatures and skips incompatible files. If channels contain mixed codecs, frame rates, or audio layouts, set `TRANSCODE_ENABLED=true` in `variables.env`.

## HTTP endpoints

- `/` - basic service metadata and discovered channel names
- `/health` - readiness summary for each channel
- `/<channel>/playlist.m3u8` - HLS playlist
- `/<channel>/<filename>` - HLS segments and related files

## Operational notes

- the service writes generated HLS output to `/tmp/hls`
- one `ffmpeg` process runs per discovered channel
- library scanning is polling-based and runs every `REPLAY_SCAN_INTERVAL` seconds
- channel directory additions and removals are picked up on the next scan
- channel file additions and removals trigger a channel restart on the next scan
- changing `TRANSCODE_ENABLED` requires restarting the service
- `/health` always returns a JSON summary; use each channel's `ready` flag to see whether `playlist.m3u8` exists yet

## Current gaps

This repo currently has no automated test suite or CI configuration. For now, the safest baseline checks are Python syntax validation and a container build.
