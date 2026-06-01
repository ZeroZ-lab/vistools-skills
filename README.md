**[中文文档](README_CN.md)** | English

# vistools Skills

> **A coordinate-grounded visual inspection protocol for AI coding agents.**

Claude Code plugin for [vistools](https://github.com/ZeroZ-lab/vistools) — gives AI coding assistants the ability to inspect, navigate, crop, and sample large images with structured JSON output and coordinate mappings back to the source.

This repository is the **plugin distribution package**. The core Rust CLI lives in [ZeroZ-lab/vistools](https://github.com/ZeroZ-lab/vistools).

## The Problem

When an AI agent receives a 3200×2400 screenshot, it sees the whole thing at once — compressed, zoomed out, details lost. It might say "the button looks correct" with no way to verify the actual pixel values or trace back to the source coordinate.

## Before vs After

**Without vistools:**
```
Claude reads a 3200×2400 screenshot
  → details lost in compression
  → claims "the button looks correct"
  → no way to verify
```

**With vistools:**
```
1. inspect → 3200×2400, needs overview
2. overview --max-side 1200 → scaled preview, scale_factor = 0.375
3. Spot anomaly at overview (800, 600)
4. Map to source: (800 / 0.375, 600 / 0.375) = (2133, 1600)
5. viewport rect → exact crop of the region
6. sample --x 2133 --y 1600 → color is #e74c3c, not the expected #2563eb
7. Report: "Button at source coordinate (2133, 1600) has incorrect background color"
```

## Installation

```bash
/plugin marketplace add ZeroZ-lab/vistools-skills
/plugin install vistools@vistools-skills
```

Binaries for macOS (arm64/x64) and Linux (arm64/x64) are bundled automatically via CI.

## Usage

**`/vistools <image> [focus]`**

The `focus` argument is an optional natural-language instruction that tells Claude what to look for or do. It is interpreted by the AI, not parsed as a CLI flag.

```
/vistools screenshot.png
/vistools large-image.jpg "focus on the header"
/vistools design.png "check if the login button color matches the design spec"
```

Automated workflow:
1. Inspect image → recommend overview/tile/viewport
2. Generate outputs with coordinate mappings
3. Sample colors to verify pixels
4. Report findings with source coordinates

### Example Workflow

> Commands prefixed with `/vistools` are typed by you in the chat. Commands prefixed with `vistools` (no slash) are run by Claude internally.

```bash
# 1. You invoke the skill on a large screenshot
/vistools screenshot.png

# Claude sees 3200×2400 and decides to generate a scaled overview
# 2. Overview (Claude runs this internally)
vistools overview screenshot.png overview.png --max-side 1200

# 3. You spot a bug in the overview at approx. (800, 600)
# 4. Claude maps back to source: (800 / 0.375, 600 / 0.375) = (2133, 1600)
# 5. Crop the region from the source image
vistools viewport rect screenshot.png bug.png \
  --x 2000 --y 1500 --width 500 --height 400

# 6. Sample a pixel to verify its color
vistools sample screenshot.png --x 2100 --y 1550
```

## Commands

| Command | Purpose |
|---------|---------|
| `inspect` | Read metadata + get strategy suggestion (sub-ms) |
| `overview` | Scaled-down preview (`--max-side`) |
| `tile` | Grid split (`--rows`/`--cols`) |
| `viewport` | Crop region: `anchor` / `percent` / `rect` modes |
| `sample` | Point or region color picker (read-only) |

All commands return structured JSON with `coordinate_mapping` for tracing back to source coordinates.

## Coordinate Mapping

Every generated image returns `coordinate_mapping`:

```json
{
  "crop_origin_in_source": [960, 720],
  "scale_factor": null,
  "formula": "source_x = result_x + 960, source_y = result_y + 720"
}
```

Map output coordinates back to source:
- **Crop**: `source = (result_x + origin_x, result_y + origin_y)`
- **Overview (scaled)**: `source = (result_x / scale + origin_x, result_y / scale + origin_y)`

## Plugin Structure

```
.claude-plugin/
└── plugin.json              # Plugin manifest

skills/
└── vistools/
    ├── SKILL.md             # Skill definition (instructions for Claude)
    ├── VERSION              # Plugin + CLI version mapping
    ├── CURSOR.md            # Cursor rule
    └── scripts/
        ├── vistools                 # Bash wrapper — detects OS/arch
        ├── vistools-macos-arm64     # CI-built binaries
        ├── vistools-macos-x64
        ├── vistools-linux-arm64
        └── vistools-linux-x64
```

## Version Compatibility

| Plugin version | Bundled CLI version | Built from |
|---|---|---|
| 0.5.0 | 0.2.1 | [vistools v0.2.1](https://github.com/ZeroZ-lab/vistools) |

Verify your local binary: `skills/vistools/scripts/vistools --version`

## Troubleshooting

**"vistools binary not found for platform"**

Compile from source (needs Rust 1.88+):
```bash
git clone https://github.com/ZeroZ-lab/vistools && cd vistools && cargo build --release
cp target/release/vistools ../vistools-skills/skills/vistools/scripts/vistools-macos-arm64  # adjust for your platform
```

**"Build failed"**

Ensure Rust is installed:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

## Uninstall

```bash
/plugin uninstall vistools
```

## Related

- **[ZeroZ-lab/vistools](https://github.com/ZeroZ-lab/vistools)** — Rust CLI source code, tests, and design docs
- **[SKILL.md](skills/vistools/SKILL.md)** — Full skill definition with decision policy
- **[CURSOR.md](skills/vistools/CURSOR.md)** — Cursor IDE rule

## License

MIT OR Apache-2.0
