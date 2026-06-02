---
description: Visually inspect, navigate, crop, sample colors, compare images, measure focus distribution, and estimate white balance using vistools CLI. Use after modifying frontend code, analyzing screenshots, or working with large images.
globs: "**/*.{png,jpg,jpeg,webp,gif,bmp,tiff}"
alwaysApply: false
---

# vistools — Visual Inspection for Cursor

You are analyzing an image using `vistools` CLI. This tool gives you coordinate-grounded visual inspection: every observation maps back to source image coordinates.

## Quick Start

```bash
# 1. Always inspect first
vistools inspect "$image"

# 2. Large image? Generate overview
vistools overview "$image" overview.png --max-side 1200

# 3. Crop a region
vistools viewport anchor "$image" crop.png --anchor center --width 800 --height 600

# 4. Sample a pixel color
vistools sample "$image" --x 120 --y 80

# 5. Compare expected vs actual images
vistools diff "$expected" "$actual"

# 6. Measure which area is sharpest
vistools focus-map "$image" --rows 3 --cols 4

# 7. Estimate white balance bias
vistools white-balance "$image"
```

## Decision Policy

- **Always inspect first** — never assume image dimensions
- **Long side > 1568px** → generate overview before analysis
- **Making a color claim** → use `sample` to verify with pixel data
- **Comparing expected vs actual screenshots** → use `diff`
- **Need to know where text or subject is sharpest** → use `focus-map` before further cropping
- **Need to know warm/cool or green/magenta cast** → use `white-balance`
- **Reporting a UI defect** → always include source coordinates
- **Coordinates uncertain** → prefer larger crop over too-tight one

## Commands

| Command | Use When |
|---------|----------|
| `inspect <img>` | First call on any image — get dimensions, format, strategy |
| `overview <img> <out> --max-side N` | Long side > 1568px — scale down to readable size |
| `tile <img> --rows N --cols N --out-dir <dir>` | Need full coverage of large image |
| `viewport anchor <img> <out> --anchor <pos> --width N --height N` | Crop by position (top-left, center, etc.) |
| `viewport percent <img> <out> --x N --y N --w N --h N` | Crop by proportion (0.0–1.0) |
| `viewport rect <img> <out> --x N --y N --width N --height N` | Crop by exact pixels |
| `sample <img> --x N --y N` | Read single pixel color |
| `sample <img> --rect x,y,w,h` | Average color + alpha stats for region |
| `diff <expected> <actual>` | Pixel difference stats for expected vs actual images |
| `focus-map <img> --rows N --cols N` | Grid sharpness map with best cell + focus point |
| `white-balance <img>` | Gray-world gains and warm/cool or green/magenta bias |

## Coordinate Mapping

Every generated image includes `coordinate_mapping`:

```json
{
  "crop_origin_in_source": [960, 720],
  "scale_factor": null,
  "formula": "source_x = result_x + 960, source_y = result_y + 720"
}
```

To map point (rx, ry) in output back to source:
- Crop only: `source = (rx + origin_x, ry + origin_y)`
- With scale: `source = (rx / scale + origin_x, ry / scale + origin_y)`

## Error Recovery

| Error | Recovery |
|-------|----------|
| `INVALID_COORDINATES` | Re-run `inspect` → clamp rect → retry |
| `PIXEL_LIMIT_EXCEEDED` | Run `overview` with smaller `--max-side` first |
| `UNSUPPORTED_FORMAT` | Convert with `sips -s format png` (macOS) |
| `OUTPUT_SAME_AS_INPUT` | Use different output path |

## Report Format

When reporting findings:
1. **Image profile**: dimensions, format, scaling needed
2. **Navigation path**: regions explored and why
3. **Findings**: observations per region
4. **Source coordinates**: where issues are in original image

If focus matters, also report:
- `best_cell` row/col and source region
- `focus_point` in source coordinates
- whether a follow-up `viewport` crop is needed

If white balance matters, also report:
- `rgb_mean`
- `gray_world_gains`
- `temperature_bias` and `tint_bias`
- that this is directional RGB analysis, not Kelvin estimation

If visual regression matters, also report:
- `changed_pixels`
- `changed_ratio`
- `mean_delta` and `max_delta`
- `bounding_rect` if present

## Links

- Source: [ZeroZ-lab/vistools](https://github.com/ZeroZ-lab/vistools)
- Plugin: [ZeroZ-lab/vistools-skills](https://github.com/ZeroZ-lab/vistools-skills)

`focus-map` requires CLI `v0.2.4+`; `white-balance` requires CLI `v0.2.5+`; `diff` requires CLI `v0.2.6+`.
