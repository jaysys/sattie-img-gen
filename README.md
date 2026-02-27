# K-Sattie Image Gen Simulator

This service simulates a satellite that:
- receives uplink commands from a ground station,
- captures an image based on satellite type,
- and provides a downloadable downlinked image.

## Console Snapshot

![K-Sattie Image Gen Simulator Console](./snapshot.png)

## Features

- `POST /satellites`: register virtual satellites (`EO_OPTICAL` or `SAR`)
- `POST /seed/mock-satellites`: create ready-to-use EO/SAR mock satellites
- `POST /seed/mock-ground-stations`: create ready-to-use mock ground stations
- `GET /ground-stations`: list available mock ground stations
- `GET /satellite-types`: view type profiles used for mock response fields
- `POST /uplink`: send command to a satellite
- async state transitions:
  - `QUEUED -> ACKED -> CAPTURING -> DOWNLINK_READY`
  - or `FAILED`
- type-based image generation:
  - `EO_OPTICAL`: pseudo RGB image with cloud noise
  - `SAR`: grayscale image with speckle noise
  - `EXTERNAL`: map tile-based image from AOI center/bbox (OSM demo mode)
- `GET /commands/{command_id}`: command status
- `POST /commands/{command_id}/rerun`: rerun failed command with same command id
- `GET /downloads/{command_id}`: download generated PNG
- `GET /preview/external-map`: preview OSM map image from AOI center before uplink

## Quick Start

```bash
./one-shot-startup.sh
```

Server runs on `http://127.0.0.1:6005`.
Validation console runs at `http://127.0.0.1:6005/`.

Stop:

```bash
./one-shot-stop.sh
```

Console tabs:
- `Dashboard`, `Satellites`, `Send Uplink`, `Commands Monitor`, `Scenarios`

## Security Defaults

API is protected by default.

- Header: `x-api-key`
- Default key: `change-me` (set `SATTI_API_KEY` in production)
- Rate limit: `600` requests/minute per IP (set `SATTI_RATE_LIMIT_PER_MIN`)
  - Local test: `SATTI_RATE_LIMIT_PER_MIN=0` to disable
- CORS allowlist: `http://localhost:6005,http://127.0.0.1:6005` (set `SATTI_ALLOWED_ORIGINS`)

Example:

```bash
SATTI_API_KEY='your-strong-key' uvicorn app.main:app --reload --port 6005
```

Recommended production run:

```bash
SATTI_API_KEY='your-strong-key' \
SATTI_ALLOWED_ORIGINS='https://your-ui-domain' \
./venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 6005
```

Public paths:
- `/health`, `/`, `/docs`, `/redoc`, `/openapi.json`

Protected paths:
- All other operational APIs require `x-api-key`.

## Client Integration Guide

### Required request header

All protected APIs must include:

```http
x-api-key: <your-api-key>
```

If omitted or invalid:
- `401 Unauthorized`

If too many requests:
- `429 Too Many Requests`

### Recommended API flow for clients

1. `POST /seed/mock-satellites` (optional for test initialization)
2. `POST /seed/mock-ground-stations` (optional for test initialization)
3. `GET /satellites` (choose `satellite_id`)
4. `GET /ground-stations` (choose `ground_station_id` when needed)
5. `POST /uplink` (store returned `command_id`)
6. `GET /commands/{command_id}` polling until `DOWNLINK_READY` or `FAILED`
7. Read `download_url` from command response when ready
8. `GET /downloads/{command_id}` using header auth or browser query auth

### Retry model (failed command)

- `Fetch` in UI: query current command state
- `Retry` in UI: rerun a failed command using same `command_id`
- API: `POST /commands/{command_id}/rerun`
- Rerun is allowed only when state is `FAILED` (otherwise `409`)

### Download link behavior

- `POST /uplink` response does not include a ready file link yet.
- `download_url` appears only when:
  - `state == DOWNLINK_READY`
  - generated image file actually exists on server
- If not ready, `GET /downloads/{command_id}` returns `409`.
- If command/file does not exist, returns `404`.

Browser-friendly download:

- Browsers opening a plain link cannot always attach `x-api-key`.
- For UI links, use query auth:
  - `/downloads/{command_id}?api_key=<SATTI_API_KEY>`

### Business-grade uplink fields

The uplink payload supports business-oriented tasking inputs:

- Uplink requester:
  - `ground_station_id` (optional, but recommended for traceability)
- AOI geometry:
  - `aoi_center_lat`, `aoi_center_lon`
  - `aoi_bbox` (`[minLon,minLat,maxLon,maxLat]`)
- Time window:
  - `window_open_utc`, `window_close_utc` (ISO8601 UTC)
- Priority:
  - `priority` (`BACKGROUND|COMMERCIAL|URGENT`)
- EO constraints:
  - `max_cloud_cover_percent`
  - `max_off_nadir_deg`
  - `min_sun_elevation_deg`
- SAR constraints:
  - `incidence_min_deg`, `incidence_max_deg`
  - `look_side` (`ANY|LEFT|RIGHT`)
  - `pass_direction` (`ANY|ASCENDING|DESCENDING`)
  - `polarization` (e.g. `VV`, `VH`)
- Delivery:
  - `delivery_method` (`DOWNLOAD|S3|WEBHOOK`)
  - `delivery_path` (required for `S3`/`WEBHOOK`)
- Generation mode (simulator-only optional fields):
  - `generation_mode` (`INTERNAL|EXTERNAL`)
  - `external_map_source` (`OSM`)
  - `external_map_zoom` (`1~19`)

### External map mode notes (business review)

- Current implementation supports `EXTERNAL + OSM` for prototype/testing.
- For production/commercial usage, use a contracted map imagery provider or self-hosted tiles.
- Public OSM tile endpoints are policy-constrained and can be blocked under heavy/commercial use.

## API Example

1) Seed mock satellites:

```bash
curl -s -X POST http://127.0.0.1:6005/seed/mock-satellites \
  -H 'x-api-key: change-me'
```

2) Seed mock ground stations:

```bash
curl -s -X POST http://127.0.0.1:6005/seed/mock-ground-stations \
  -H 'x-api-key: change-me'
```

3) Check satellite list with profiles:

```bash
curl -s http://127.0.0.1:6005/satellites \
  -H 'x-api-key: change-me'
```

4) Check ground station list:

```bash
curl -s http://127.0.0.1:6005/ground-stations \
  -H 'x-api-key: change-me'
```

5) Send uplink command:

```bash
curl -s -X POST http://127.0.0.1:6005/uplink \
  -H 'x-api-key: change-me' \
  -H 'Content-Type: application/json' \
  -d '{
    "satellite_id":"sat-xxxxxxx",
    "ground_station_id":"gnd-xxxxxxx",
    "mission_name":"harbor-monitoring",
    "aoi_name":"busan-port",
    "width":1024,
    "height":1024,
    "cloud_percent":25,
    "fail_probability":0.05
  }'
```

6) Poll status:

```bash
curl -s http://127.0.0.1:6005/commands/cmd-xxxxxxxxxxxx \
  -H 'x-api-key: change-me'
```

When `state` is `DOWNLINK_READY`, use `download_url`:

```bash
curl -L -o result.png http://127.0.0.1:6005/downloads/cmd-xxxxxxxxxxxx \
  -H 'x-api-key: change-me'
```

Browser link example:

```text
http://127.0.0.1:6005/downloads/cmd-xxxxxxxxxxxx?api_key=change-me
```

## Notes

- Data is in-memory for state tracking.
- Output images are stored under `data/images/`.
- This is an MVP simulator and can be extended with:
  - mission windows/contact windows,
  - retry/escalation policies,
  - persistent DB/event log,
  - realistic AOI-based rendering.
