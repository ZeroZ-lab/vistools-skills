# vistools Skills

Claude Code plugin for [vistools](https://github.com/ZeroZ-lab/vistools) — visual instruments for AI coding assistants. Inspect, navigate, crop, and sample large images with structured JSON output and coordinate mapping.

## Installation

Add the marketplace and install the plugin:

```bash
/plugin marketplace add ZeroZ-lab/vistools-skills
/plugin install vistools@vistools-skills
```

Binaries for macOS (arm64/x64) and Linux (arm64/x64) are bundled automatically via CI.

## Usage

**`/vistools <image> [focus]`**

```
/vistools screenshot.png
/vistools large-image.jpg "focus on the header"
/vistools design.png "extract the login button"
```

Automated workflow:
1. Inspect image → recommend overview/tile/viewport
2. Generate outputs with coordinate mappings
3. Sample colors to verify pixels
4. Report findings with source coordinates

### Example Workflow

```bash
# 1. Inspect large screenshot
/vistools screenshot.png

# Skill detects 3200x2400, recommends overview + tile
# 2. Generate overview
vistools overview screenshot.png overview.png --max-side 1200

# 3. Find bug in overview at (800, 600)
# 4. Map back to source: (800 / 0.375, 600 / 0.375) = (2133, 1600)
# 5. Crop region in source
vistools viewport rect screenshot.png bug.png \
  --x 2000 --y 1500 --width 500 --height 400

# 6. Sample a pixel to verify its color
vistools sample screenshot.png --x 2100 --y 1550
```

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
    ├── SKILL.md             # Skill definition
    └── scripts/
        ├── vistools                 # Platform-detect wrapper
        ├── vistools-macos-arm64     # CI-built binaries
        ├── vistools-macos-x64
        ├── vistools-linux-arm64
        └── vistools-linux-x64
```

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

- **[vistools](https://github.com/ZeroZ-lab/vistools)** — Rust CLI source code
- **[SKILL.md](skills/vistools/SKILL.md)** — Skill definition

## License

MIT OR Apache-2.0
