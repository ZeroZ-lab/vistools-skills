**[中文文档](README_CN.md)** | English

# vistools Skills

Claude Code plugin for [vistools](https://github.com/ZeroZ-lab/vistools) — visual instruments for AI coding assistants. Inspect, navigate, crop, and sample large images with structured JSON output and coordinate mapping.

## Why

Claude Code struggles with large images — reading a 3200×2400 screenshot directly loses fine detail and can exceed context limits. `vistools` solves this by giving Claude a set of image tools: inspect dimensions, generate scaled overviews, crop precise regions, and sample individual pixels. Every operation returns structured JSON with **coordinate mappings** so Claude can always trace what it sees back to the source image.

## Installation

Add the marketplace and install the plugin:

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
/vistools design.png "extract the login button"
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
    └── scripts/
        ├── vistools                 # Bash wrapper — detects OS/arch and
                                     # runs the matching binary below
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
