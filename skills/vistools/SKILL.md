---
description: Visually inspect, navigate, crop, sample colors, and measure focus distribution using vistools CLI. Use after modifying frontend code, analyzing screenshots, or working with large images.
when_to_use: Use when the user wants to analyze, crop, sample colors, measure blur or focus, or navigate large images. Triggered by image files or requests about screenshots, UI verification, or visual analysis.
argument-hint: "<image-path> [focus or action]"
arguments: [image, action]
allowed-tools: Bash(${CLAUDE_SKILL_DIR}/scripts/vistools *) Bash(jq *) Bash(sips *) Bash(which *) Read
paths: "**/*.{png,jpg,jpeg,webp,gif,bmp,tiff}"
---

# vistools — Visual Instruments for AI Agents

You are analyzing an image using `vistools`. The image is: `$image`
$action

The `vistools` binary is bundled with this skill at `${CLAUDE_SKILL_DIR}/scripts/vistools`. This wrapper auto-detects your platform (macOS/Linux, arm64/x64) and runs the correct binary. Always invoke it via this path.

## Step 1: Inspect (always first)

```bash
${CLAUDE_SKILL_DIR}/scripts/vistools inspect "$image" | jq .
```

Read the JSON output:
- `data.source.width` × `data.source.height` — dimensions
- `data.source.format` — "png", "jpeg", etc.
- `data.suggestion.needs_overview` — `true` if long side > 1568px
- `data.suggestion.recommended_next` — `"overview"` for large images, `"direct"` for small
- `data.suggestion.max_tile_rows` × `max_tile_cols` — recommended grid
- `data.suggestion.suggested_max_side` — recommended max side for overview

## Step 2: Decide What To Do

**Small image** (`recommended_next: "direct"`) → analyze directly, skip overview.

**Large image** (`recommended_next: "overview"`) → generate overview first:
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools overview "$image" /tmp/iv-overview.png --max-side 1200 | jq .
```
Note the `scale_factor` — divide overview coordinates by it to get source coordinates.

## Decision Policy

Follow these rules when choosing commands:

- **Always inspect first.** Never assume image dimensions.
- **Long side > 1568px → overview before any visual analysis.**
- **User asks about a specific area but gives no coordinates →** generate overview, then infer approximate region from the overview.
- **Making a visual claim about color →** use `sample` to verify with pixel data, not visual perception.
- **Need to know where the image is sharpest or blurriest →** use `focus-map` with a coarse grid before drilling down with `viewport`.
- **Reporting a UI defect →** always include source coordinates via `coordinate_mapping`.
- **Crop coordinates uncertain →** state "approximate" and prefer a larger crop over a too-tight one.
- **Image unchanged after modification →** skip further navigation, report "no visual change detected."

### When to stop drilling down
- You can clearly read the text/UI element you're investigating.
- Further crops no longer reveal new information.
- You've chained 3 crops deep — stop and report what you found.

## Step 3: Navigate — Pick Your Strategy

### A. Grid (full coverage)
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools tile "$image" --rows 2 --cols 3 --out-dir /tmp/iv-tiles
```
Output lists every tile with `source_region`. Tiles cover the full source seamlessly.
Last tile per row/col absorbs remainder pixels.

### B. Positional crop (anchor)
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools viewport anchor "$image" /tmp/iv-crop.png \
    --anchor center --width 800 --height 600
```
Anchors: `top-left` | `top` | `top-right` | `left` | `center` | `right` | `bottom-left` | `bottom` | `bottom-right`

### C. Proportional crop (percent)
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools viewport percent "$image" /tmp/iv-crop.png \
    --x 0.3 --y 0.3 --w 0.4 --h 0.4
```
Values are fractions 0.0–1.0 of the source. Strict: `x + w` and `y + h` must not exceed 1.0.

### D. Exact pixel crop (rect)
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools viewport rect "$image" /tmp/iv-crop.png \
    --x 100 --y 200 --width 800 --height 600
```
Must not exceed source bounds — check with `inspect` first.

## Step 4: Sample Colors (read-only, no output image)

### Point color
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools sample "$image" --x 120 --y 80
```
Returns `rgba`, `rgb`, `hex`, and `alpha` at that pixel.

### Region average color + alpha stats
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools sample "$image" --rect 100,80,40,40
```
Returns rounded average color, `alpha_stats` (min, max, average, transparent_ratio), and `pixel_count`.

Use `sample` to verify colors, detect transparency, or confirm what a specific pixel looks like — without relying on visual interpretation.

## Step 4b: Measure Focus Distribution

If the task involves blur, focus, text legibility, or "which area is sharpest?", use `focus-map`.

```bash
${CLAUDE_SKILL_DIR}/scripts/vistools focus-map "$image" --rows 3 --cols 4
```

Optional region-first workflow:

```bash
${CLAUDE_SKILL_DIR}/scripts/vistools focus-map "$image" --rect 100,80,800,600 --rows 2 --cols 3
```

Read the JSON output:
- `data.cells[]` — per-cell `region` + `sharpness`
- `data.best_cell` — the sharpest cell in the grid
- `data.focus_point` — center point of `best_cell`, useful for a follow-up `viewport`

Use `focus-map` to decide where to zoom next, or to verify that the subject is actually inside the sharp region.

## Step 5: Coordinate Back-Mapping

Every generated image includes `coordinate_mapping`:
```json
{
  "crop_origin_in_source": [960, 720],
  "scale_factor": null,
  "formula": "source_x = result_x + 960, source_y = result_y + 720"
}
```

**To map point (rx, ry) in output back to source:**
- Crop only (scale_factor is null): `source = (rx + origin_x, ry + origin_y)`
- With scale: `source = (rx / scale + origin_x, ry / scale + origin_y)`

**Practical use:** If you spot a UI bug at (150, 80) in the crop, and `crop_origin_in_source` is [960, 720], the bug is at **(1110, 800)** in the source image.

## Step 6: Drill Down (recursive)

Crop a crop to focus on a sub-region:
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools viewport anchor /tmp/iv-crop1.png /tmp/iv-crop2.png \
    --anchor top-left --width 400 --height 300
```
Chain coordinate mappings to trace back to the original source.

## Error Handling

All errors return `{"ok": false, "error": {"code": "...", "message": "..."}}`. Process exits non-zero.

| Code | Meaning | Fix |
|------|---------|-----|
| `Binary not found` (wrapper) | Platform binary missing from `scripts/` | CI may not have run yet, or platform unsupported. Check `scripts/` directory or see Manual Build below |
| `FILE_NOT_FOUND` | Image doesn't exist | Check path |
| `UNSUPPORTED_FORMAT` | Can't decode | Not a valid image file |
| `INVALID_COORDINATES` | Rect exceeds bounds | Check source dimensions with inspect |
| `INVALID_PARAMETERS` | Bad tile count / zero max side / malformed sample mode | Tile max 64 |
| `INVALID_DIMENSIONS` | Zero width/height | Use positive values |
| `OUTPUT_SAME_AS_INPUT` | Would overwrite source | Use a different output path |
| `OUTPUT_WRITE_ERROR` | Could not write output | Check disk space / permissions |
| `PIXEL_LIMIT_EXCEEDED` | > 100 megapixels | Use overview to scale down first |
| `PATH_ESCAPE` | Path has `..` | Use absolute or relative without `..` |

### Recovery Policy

When a command fails, try to recover before asking the user:

| Error | Recovery |
|-------|----------|
| `INVALID_COORDINATES` | Re-run `inspect` → clamp rect to source bounds → retry once |
| `PIXEL_LIMIT_EXCEEDED` | Run `overview` with smaller `--max-side` first, then operate on the overview |
| `UNSUPPORTED_FORMAT` | Suggest converting with `sips -s format png` (macOS) or ask user to provide PNG/JPEG |
| `INVALID_PARAMETERS` | Re-read the flag requirements from `--help` and correct the call |
| `FILE_NOT_FOUND` | Check path spelling, try absolute path |
| `OUTPUT_SAME_AS_INPUT` | Add a suffix: `crop.png` → `crop-vp.png` |

## Output Format

Every command returns the same envelope on stdout:
```json
{
  "ok": true,
  "operation": "viewport",
  "input": "src.png",
  "data": { ... },
  "warnings": [],
  "elapsed_ms": 12
}
```

## Report Templates

Choose the template that matches the task:

### General Visual Analysis
1. **Image profile**: dimensions, format, whether it needed scaling
2. **Navigation path**: which regions explored and why
3. **Findings**: observations in each region
4. **Source coordinates**: where issues are in the original image

### Focus / Blur Report
1. **Grid setup**: rows × cols, optional rect region
2. **Best cell**: row/col + source `region`
3. **Focus point**: source `(x, y)` from `focus_point`
4. **Evidence**: compare `best_cell.sharpness.score` against neighboring cells
5. **Next action**: whether to crop `best_cell` with `viewport`

### UI Bug Report
1. **Component**: what UI element is affected
2. **Source coordinates**: exact location via `coordinate_mapping`
3. **Expected vs Actual**: what should appear vs what was observed
4. **Verification**: `sample` result confirming color/alpha
5. **Confidence**: high (pixel-verified) / medium (visual) / low (approximate)

### Color Verification
1. **Target**: what was being checked (e.g., "primary button background")
2. **Sample coordinates**: (x, y) in source image
3. **Actual color**: RGBA + HEX from `sample`
4. **Expected color**: from design spec or user description
5. **Match**: yes / no / close (within tolerance)

## Help & Version

If you need a quick reference on any command's flags:
```bash
${CLAUDE_SKILL_DIR}/scripts/vistools --help             # list all commands
${CLAUDE_SKILL_DIR}/scripts/vistools inspect --help     # subcommand details
${CLAUDE_SKILL_DIR}/scripts/vistools --version          # e.g. "vistools 0.2.0"
```

`focus-map` is available in CLI `v0.2.3+`. If the bundled binary predates that, the source repo still has the command even if this plugin bundle has not been refreshed yet.

## Manual Build (advanced)

If the wrapper reports a missing binary for your platform:

```bash
git clone https://github.com/ZeroZ-lab/vistools && cd vistools && cargo build --release
cp target/release/vistools ${CLAUDE_SKILL_DIR}/scripts/vistools-$(uname -s | tr '[:upper:]' '[:lower:]' | sed 's/darwin/macos/')-$(uname -m | sed 's/x86_64/x64/;s/aarch64/arm64/')
```

Or [open an issue](https://github.com/ZeroZ-lab/vistools/issues) with your OS and architecture details.

## Links

- **Source repo**: [ZeroZ-lab/vistools](https://github.com/ZeroZ-lab/vistools) — Rust CLI source code, tests, and design docs
- **Skills repo**: [ZeroZ-lab/vistools-skills](https://github.com/ZeroZ-lab/vistools-skills) — Claude Code plugin, Cursor rule, Codex instructions
