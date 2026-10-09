# NINA Advanced API — MCP Server

An MCP (Model Context Protocol) server that exposes [N.I.N.A.](https://nighttime-imaging.eu/) (Nighttime Imaging 'N' Astronomy) equipment control to AI agents via the [NINA Advanced API plugin](https://github.com/christian-photo/ninaAPI).

Tested with the **ZWO Seestar S30** on Claude Desktop. Should work with any NINA-supported equipment.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![NINA](https://img.shields.io/badge/NINA-3.0+-green.svg)](https://nighttime-imaging.eu/)

---

## What it does

Gives an AI agent (Claude, or any MCP-compatible client) natural-language control of your astrophotography rig:

- **Camera** — connect, capture, cooling, binning, dew heater, statistics, image history, thumbnails
- **Mount** — slew, track, park, home, flip, tracking mode (sidereal/lunar/solar)
- **Filter wheel** — connect, change filter, info
- **Focuser** — move, temperature compensation
- **Guider** — connect, start/stop guiding, calibrate, graph
- **Dome** — open/close shutter, slew, park, follow mode
- **Flat panel** — connect, brightness, light, cover
- **Rotator** — move, sync, reverse
- **Application** — version, tab switching, plugins, screenshots, time
- **Help system** — searchable tool documentation built into the server (`nina_help`)

---

## Requirements

- Windows (NINA runs on Windows only)
- [N.I.N.A. 3.0+](https://nighttime-imaging.eu/)
- [NINA Advanced API plugin v2](https://github.com/christian-photo/ninaAPI) — install via the NINA plugin manager
- Python 3.10+ (tested on 3.13)
- Optional: [`uv`](https://github.com/astral-sh/uv) package manager (not usable with Smart App Control on, see below)
- Optional: [ASTAP](https://www.hnsky.org/astap.htm) with the D05 star database, for plate solving
- An MCP-compatible AI agent — [Claude Desktop](https://claude.ai/download) recommended

---

## Installation

### 1. Install the NINA Advanced API plugin

In NINA: **Options → Plugins → search "Advanced API" → Install → restart NINA**

Enable it under **Options → Advanced API** and note the port (default: `1888`).

### 2. Clone this repo

```bash
git clone https://github.com/paraporoco/nina-advanced-api-mcp.git
cd nina-advanced-api-mcp
```

### 3. Create your `.env`

```bash
copy .env.example .env
# Edit .env with your values
```

### 4. Configure Claude Desktop

**Option A - dedicated virtual environment (recommended, required with Windows Smart App Control).**

Install the dependencies once:

```bat
cd C:\path\to\nina-advanced-api-mcp
python -m venv .venv
.venv\Scripts\python.exe -m pip install "fastmcp==1.0" fastapi uvicorn pydantic aiohttp requests python-dotenv
```

Then add to `%APPDATA%\Claude\claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "nina": {
      "command": "C:\\path\\to\\nina-advanced-api-mcp\\.venv\\Scripts\\python.exe",
      "args": ["C:\\path\\to\\nina-advanced-api-mcp\\nina_advanced_mcp.py"],
      "env": {
        "NINA_HOST": "localhost",
        "NINA_PORT": "1888",
        "LOG_LEVEL": "INFO",
        "IMAGE_SAVE_DIR": "C:/Users/YOUR_USERNAME/Pictures/NINA",
        "AIOHTTP_NO_EXTENSIONS": "1"
      }
    }
  }
}
```

- `AIOHTTP_NO_EXTENSIONS=1` makes aiohttp use its pure-Python HTTP parser. Smart App Control blocks the compiled parser (`_http_parser.*.pyd`); aiohttp falls back anyway, but the setting stops the block events.
- Start-up time with this method: approx. 1.5 s.

**Option B - `uv run` (only if Smart App Control / WDAC is off).**

```json
{
  "mcpServers": {
    "nina": {
      "command": "uv",
      "args": [
        "run",
        "--with",
        "fastmcp==1.0,fastapi,uvicorn,pydantic,aiohttp,requests,python-dotenv",
        "fastmcp",
        "run",
        "C:/path/to/nina-advanced-api-mcp/nina_advanced_mcp.py"
      ],
      "env": {
        "NINA_HOST": "localhost",
        "NINA_PORT": "1888",
        "LOG_LEVEL": "INFO",
        "IMAGE_SAVE_DIR": "C:/Users/YOUR_USERNAME/Pictures/NINA"
      }
    }
  }
}
```

`uv run --with` builds a temporary environment at each start and generates a new, unsigned `fastmcp.exe` launcher. With Smart App Control on, Windows blocks it (`os error 4551`) and the server fails to start, often only in some Claude Desktop sessions.

> **Note:** `NINA_HOST` can be a remote IP address if NINA runs on a separate machine on your network (e.g., a dedicated imaging PC).

> **Important:** Pin `fastmcp==1.0`. Later fastmcp releases have breaking API changes that prevent the server from starting.

Restart Claude Desktop. All NINA tools will appear automatically in your conversation.

---

## Usage examples

```
Connect to NINA and show me equipment status
```
```
Take a 30-second exposure with the LP filter at gain 80, don't download
```
```
Slew to M42, enable sidereal tracking, then capture 10 frames at 60s
```
```
Show me capture statistics for the last image
```
```
Start cooling the camera to -10°C
```

---

## Known limitations - Seestar S30 (native Alpaca)

Tested 2026-10-09 with Seestar S30 Alpaca server 1.2.0-3, N.I.N.A. 3.2.0.9001, Advanced API plugin 2.2.15.2, on Windows 11 with Smart App Control on.

Current Seestar firmware has a built-in ASCOM Alpaca server (port `32323`, discovery on UDP `32227`). It exposes Telescope, Camera, Focuser, FilterWheel and Switch. N.I.N.A. finds these as "ASCOM Alpaca" devices; the ASCOM Platform is **not** necessary. The Seestar must be on the same Wi-Fi as the NINA PC (station mode).

| Issue | Workaround |
|---|---|
| Alpaca server starts approx. 1-2 min after the Seestar app reports the device ready; first NINA scan finds nothing | Wait, then rescan (see next row) |
| `nina_list_camera_devices` does not rescan; devices added after NINA start are missing | Call `GET /v2/api/equipment/{camera,filterwheel,focuser}/rescan` on the Advanced API, then connect by device ID |
| On first mount connect, NINA opens a modal **"Synchronize site settings"** dialog; `nina_connect_mount` then returns an error with an empty message | Answer the dialog on the NINA PC. Choose **mount -> N.I.N.A.** The other direction writes the NINA profile site (default 0/0) into the Seestar |
| Seestar site was overwritten with 0/0 | `PUT /api/v1/telescope/0/sitelatitude` and `.../sitelongitude` on port 32323, set the same values in the NINA profile, reconnect the mount |
| Slew (`nina_slew_mount`, `wait_for_completion=false`) | Works. Approx. 25 s for a large slew. Verify with `nina_get_mount_info` |
| Tracking after slew | The Seestar enables tracking after a go-to. Set `nina_set_tracking_mode` mode `0` explicitly anyway |
| Manual axis motion | No MCP tool yet. Use Alpaca `PUT /api/v1/telescope/0/moveaxis` (`Axis` 0 = azimuth, 1 = altitude, `Rate` 0-6 deg/s; `Rate=0` stops) |
| `SiteElevation`, `SideOfPier` not implemented by the driver | Harmless; NINA logs INFO messages |
| Park position | `nina_park_mount` folds the arm; reported altitude approx. -89.8 deg, `AtPark: true` |
| FITS files | Full 16-bit images (1080 x 1920, RGGB, 2.9 um pixels, approx. 4.15 MB). The 0-byte FITS problem of the old ASCOM path does not occur with Alpaca |
| `solve: true` in `nina_capture_image` causes API timeouts | Omit `solve`; plate-solve separately (ASTAP) |
| Planets appear as single pixels | The Seestar S30 (30 mm / 150 mm FL) is a wide-field instrument; not suitable for planetary detail |

**Image retrieval sequence:**

```
nina_capture_image(download=False)
-> wait for the exposure + approx. 15 s download/save
-> nina_get_capture_statistics()        # star count, HFR, median ADU
-> FITS file in the NINA image folder (profile setting ImageFileSettings-FilePath)
```

Reference values at gain 80 (2026-10-09, overcast): bias/dark median approx. 960 ADU; hot pixels give 50-70 false "stars" in short frames.

### Plate solving (ASTAP)

- Image scale: 2.9 um / 150 mm = 3.99 arcsec/pixel; field of view approx. 1.2 x 2.1 deg. The D05 star database is sufficient.
- Set in NINA: plate solver ASTAP, `PlateSolveSettings-ASTAPLocation` = `C:\Program Files\astap\astap.exe`, telescope focal length 150 mm, pixel size 2.9 um.
- The ASTAP installers are not code-signed, but Smart App Control did not block them in the test.

## Troubleshooting

**Tool calls fail with `Input validation error: '...' is not of type 'object'`**
Some MCP clients (including some Claude Desktop builds) flatten JSON-Schema `$ref`/`$defs` in tool schemas, so arguments arrive as strings and fail validation. The server now inlines all `$ref`s at startup, so schemas are self-contained. If you still see this, pull the latest version and restart Claude Desktop.

**Server never appears in Claude Desktop**
The config example uses `"command": "uv"`, which requires `uv` on the PATH of the Claude Desktop process. If `uv` was installed per-user or via pip, use the full path instead, e.g. `"command": "C:/Python314/Scripts/uv.exe"`.

**Server fails to start with `Failed to spawn: fastmcp ... An Application Control policy has blocked this file. (os error 4551)`**
Smart App Control or a WDAC policy blocks the unsigned `fastmcp.exe` that `uv run` generates. Check `Microsoft-Windows-CodeIntegrity/Operational` for events 3033/3077. Use Option A (virtual environment + `python.exe`). In the Claude Desktop log the same block can show as `spawn UNKNOWN`.

**NINA tools return 401 "Not connected to NINA server"**
Call `nina_connect` once per server process (also after each Claude Desktop restart). If `http://localhost:1888/v2/api/version` does not answer, check that only **one** NINA instance runs: a second instance cannot bind port 1888 (`Only one usage of each socket address`) and runs without the API.

**Locked-down (corporate) machines**
If an Application Control policy blocks uv-managed Python downloads, uv falls back to the system Python. Python 3.13 and 3.14 both work.

---

## Reference

### Tracking modes

| Mode | Tracking |
|---|---|
| `0` | Sidereal |
| `1` | Lunar |
| `2` | Solar |
| `3` | King rate |
| `4` | Stopped |

### Filter IDs (Seestar S30)

| ID | Filter |
|---|---|
| `0` | Dark (closed, for dark frames) |
| `1` | IR |
| `2` | LP (light pollution) |

---

## Project structure

```
nina-advanced-api-mcp/
├── nina_advanced_mcp.py   # MCP server — all tools
├── nina_help.json         # Searchable help content for nina_help tool
├── requirements.txt       # Python dependencies
├── .env.example           # Configuration template
├── LICENSE
└── README.md
```

---

## Acknowledgements

- [N.I.N.A.](https://nighttime-imaging.eu/) — Nighttime Imaging 'N' Astronomy
- [NINA Advanced API](https://github.com/christian-photo/ninaAPI) by christian-photo
- [Touch'N'Stars](https://github.com/Touch-N-Stars/Touch-N-Stars) — related mobile control project for NINA
- [PaDev1/Nina_advanced_api_mcp](https://github.com/PaDev1/Nina_advanced_api_mcp) — original repository this project is based on

## License

MIT — see [LICENSE](LICENSE).

