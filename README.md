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
- [`uv`](https://github.com/astral-sh/uv) package manager
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

Add to `%APPDATA%\Claude\claude_desktop_config.json`:

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

> **Note:** `NINA_HOST` can be a remote IP address if NINA runs on a separate machine on your network (e.g., a dedicated imaging PC).

> **Important:** Pin `fastmcp==1.0` in the `--with` argument. Later fastmcp releases have breaking API changes that prevent the server from starting.

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

## Known limitations — Seestar S30 (ASCOM driver)

These quirks apply specifically to the ZWO Seestar S30 via its ASCOM driver in NINA. Other mounts/cameras may not have these behaviors.

| Issue | Workaround |
|---|---|
| `nina_slew_mount` returns an error even when the slew succeeds | Always follow with `nina_get_mount_info` to verify actual pointing |
| Tracking resets to stopped after every slew | Explicitly call `nina_set_tracking_mode` with mode `0` (sidereal) or `1` (lunar) after each slew |
| Camera disconnects intermittently | Run `nina_connect_camera` before every capture sequence |
| `download: true` consistently times out | Use `download: false` (background capture); retrieve result via `nina_get_image_thumbnail` |
| FITS files in the snapshot directory are always 0 bytes | The Seestar ASCOM driver does not write raw FITS through NINA's pipeline; JPEG thumbnails work |
| `solve: true` causes API timeouts | Plate solving exceeds the MCP timeout; omit the `solve` parameter entirely |
| RA/Dec coordinates may not reflect actual sky pointing | The Seestar S30 is alt-az; use the native Seestar app to verify target acquisition |
| Planets appear as single pixels | The Seestar S30 (30mm / 150mm FL) is a wide-field instrument; not suitable for planetary detail |

**Working image retrieval sequence:**

```
nina_capture_image(download=False)
→ nina_get_image_thumbnail(index=0)    # saves JPG to IMAGE_SAVE_DIR/thumbnails/
→ nina_get_capture_statistics()        # confirms signal: star count, HFR, median ADU
```

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
| `0` | Filter1 (clear) |
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

