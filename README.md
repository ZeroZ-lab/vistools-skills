# vistools Skills

AI agent skills for [vistools](https://github.com/zhengjianqiao/vistolls) — visual analysis tools for AI coding assistants.

## Prerequisites

Install the `vistools` CLI first:

```bash
git clone https://github.com/zhengjianqiao/vistolls
cd vistolls
cargo install --path crates/cli
```

Verify: `vistools --version`

## Installation

### Claude Code (Plugin — Recommended)

```bash
/plugin install https://github.com/zhengjianqiao/vistolls-skills
```

Then use:
```
/vistools screenshot.png
```

### Cursor

```bash
# Copy rule file
mkdir -p .cursor/rules
curl -o .cursor/rules/vistools.mdc \
  https://raw.githubusercontent.com/zhengjianqiao/vistolls-skills/main/skills/cursor/vistools.mdc
```

### Codex

```bash
# Append to AGENTS.md
curl -s https://raw.githubusercontent.com/zhengjianqiao/vistolls-skills/main/skills/codex/AGENTS.md >> AGENTS.md
```

## Usage

Once installed, the skill provides intelligent image analysis workflows:

- **Inspect**: Get image dimensions, format, and recommended next steps
- **Overview**: Generate scaled preview for large images (>1568px)
- **Tile**: Split into grid for systematic analysis
- **Viewport**: Crop specific regions with coordinate mapping
- **Resize/Rotate**: Transform with coordinate back-reference

### Example Workflow

```bash
# 1. Inspect large screenshot
/vistools screenshot.png

# Skill detects 3200x2400, recommends overview + tile
# 2. Generate overview
vistools overview screenshot.png overview.png --max-width 1200

# 3. Find bug in overview at (800, 600)
# 4. Map back to source: (800 / 0.375, 600 / 0.375) = (2133, 1600)
# 5. Crop region in source
vistools viewport rect screenshot.png bug.png \
  --x 2000 --y 1500 --width 500 --height 400
```

## What's Included

### Claude Code Skill

**`/vistools <image> [focus]`**

Automated workflow:
1. Inspect image → recommend overview/tile/viewport
2. Generate outputs with coordinate mappings
3. Report findings with source coordinates

### Cursor Rule

Auto-attached when working with `.png/.jpg/.webp/.gif/.bmp/.tiff` files.

Provides:
- Command reference
- Coordinate mapping guide
- Error handling

### Codex Instructions

Project-level guidance for:
- Workflow strategies
- Output interpretation
- Best practices

## Coordinate Mapping

Every crop/resize/rotate returns `coordinate_mapping`:

```json
{
  "crop_origin_in_source": [960, 720],
  "scale_factor": null,
  "formula": "source_x = result_x + 960, source_y = result_y + 720"
}
```

Map output coordinates back to source:
- **Crop**: `source = (result_x + origin_x, result_y + origin_y)`
- **Resize**: `source = (result_x / scale + origin_x, result_y / scale + origin_y)`
- **Rotate**: use formula string

## Troubleshooting

**"vistools: command not found"**

Install the CLI:
```bash
git clone https://github.com/zhengjianqiao/vistolls
cd vistolls
cargo install --path crates/cli
```

**"Build failed"**

Ensure Rust is installed:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

## Uninstall

```bash
# Claude Code
/plugin uninstall vistools-skills

# Cursor
rm .cursor/rules/vistools.mdc

# Codex
# Manually remove appended section from AGENTS.md
```

## Related

- **[vistools](https://github.com/zhengjianqiao/vistolls)** — Rust CLI source code
- **[AGENTS.md](skills/codex/AGENTS.md)** — Codex instructions
- **[SKILL.md](skills/claude-code/vistools/SKILL.md)** — Claude Code skill definition

## License

MIT OR Apache-2.0
