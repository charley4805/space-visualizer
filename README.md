# Space Visualizer

A Claude skill that transforms room photos into dimensioned blueprints and stylized remodel visions.

## What It Does

Upload a photo of any room. Claude will identify every recognizable item, look up real-world dimensions, calculate scale from the photo, and produce either:

- **Blueprint Mode** — A to-scale technical diagram matching the photo's angle, with dimension lines on every item (width, height, depth), a scale bar, and both imperial + metric measurements.
- **Remodel Vision Mode** — A stylized architectural illustration showing your space redesigned. Describe what you want added, removed, or changed, and Claude places it with smart spacing rules and highlights new items clearly.

### Blueprint Example

> *"Blueprint this bathroom"*
> → Claude identifies toilet, vanity, tile, mirror, towel bar → scales the scene → produces a dimensioned SVG matching the photo's 3/4 perspective view.

### Remodel Vision Example

> *"Show me this kitchen with a waterfall island and white shaker cabinets replacing the oak ones"*
> → Claude keeps the existing layout, removes old cabinets, adds the island at center with 42" clearance, renders the whole scene as a clean architectural illustration.

---

## Supported Room Types

Bathroom · Kitchen · Bedroom · Living Room · Any interior space

---

## Installation

### Recommended (clone directly into Claude skills directory)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/YOUR_USERNAME/space-visualizer.git ~/.claude/skills/space-visualizer
```

### Manual

```bash
mkdir -p ~/.claude/skills/space-visualizer
# Copy SKILL.md and the references/ folder into that directory
```

---

## Usage

In Claude Code, upload a room photo and say:

```
Blueprint this room
```

Or for a remodel:

```
Show me this bathroom with a freestanding soaking tub replacing the alcove tub,
and add a double vanity on the left wall
```

Or ask Claude directly:

```
/space-visualizer

[attach photo]
[describe your remodel, or just say "blueprint" for dimensions only]
```

---

## How It Works

```
[Photo] → Vision Analysis → Dimension Lookup → Scale Calculation → Spatial Model
                                                                         │
                                          ┌──────────────────────────────┤
                                          ▼                              ▼
                                    Blueprint Mode               Remodel Vision Mode
                                  (technical diagram)         (architectural illustration)
```

1. **Vision Analysis** — Every recognizable item is catalogued with pixel bounds and confidence rating
2. **Dimension Lookup** — Built-in tables cover 95% of standard items; web search handles the rest
3. **Scale Calculation** — Uses priority anchors (door > toilet > outlet > tile) to derive real-world scale. Cross-validates with a second anchor when possible.
4. **Spatial Model** — Maps every item's position in real inches from the nearest corner
5. **Rendering** — SVG output inline in chat + downloadable file

---

## Measurement System

Shows both imperial and metric by default: `30" (76cm)`

Override per session:
- *"metric only"* → `76cm`
- *"imperial only"* → `30"`

---

## Output

- SVG rendered inline in Claude chat
- Downloadable SVG file
- Summary listing all identified items, dimensions, scale, and any estimates flagged with `~`

---

## Accuracy Notes

- Estimated measurements are marked with `~` — verify before any construction work
- Scale is derived from visible anchor items in the photo; unusual angles may reduce accuracy
- This is a visual planning tool — consult a licensed contractor before any renovation

---

## Built-In Dimension Tables

The `references/dimensions.md` file contains standard dimensions for:
- Bathroom fixtures (toilets, vanities, tubs, showers)
- Kitchen cabinets and appliances
- Bedroom furniture
- Living room furniture
- Structural elements (walls, doors, windows, ceilings)
- Tile and surface materials

---

## Version History

- **1.0.0** — Initial release: Blueprint + Remodel Vision modes, 5 room types, imperial/metric

---

## License

MIT

---

## References

- Standard residential dimensions sourced from NKBA (National Kitchen & Bath Association) guidelines
- ADA Standards for Accessible Design
- IRC (International Residential Code) rough opening standards
