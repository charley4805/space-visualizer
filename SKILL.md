---
name: space-visualizer
version: 1.0.0
description: |
  Transform a photo of any room into a dimensioned blueprint or a stylized remodel vision.
  Use this skill whenever a user uploads a photo of a room (bathroom, kitchen, bedroom,
  living room, or any interior space) and wants EITHER:
  (A) a to-scale blueprint/layout diagram showing all items with their real-world dimensions, OR
  (B) a stylized remodel illustration that incorporates items they describe adding or changing.
  Trigger on: "blueprint this room", "show me dimensions", "what size is everything",
  "create a layout", "remodel vision", "show me with a [new item]", "redesign this space",
  "floor plan from photo", "room diagram", "add [item] to my room". Always trigger when an
  image is uploaded alongside any spatial or remodel intent.
---

# Space Visualizer Skill

Two modes, one shared pipeline. Read this full file before starting.

## Reference Files
- `references/dimensions.md` — Built-in dimension tables for all common items. Read before web-searching.
- `references/svg-rendering.md` — SVG patterns, color palettes, perspective helpers. Read before drawing.

---

## Step 0: Determine Mode

**Mode 1 — Blueprint** if the user wants:
- Dimensions / measurements of the existing space
- A to-scale layout or floor plan
- "What size is everything?" type requests
- No remodel prompt (image only)

**Mode 2 — Remodel Vision** if the user wants:
- To add, remove, or change items in the room
- A visualization of what a renovation would look like
- Combined: "show the dimensions AND add a new vanity"

---

## SHARED CORE PIPELINE (run for both modes)

### Phase 1: Vision Analysis

Examine the uploaded image carefully and inventory every recognizable element:

**Catalog each item with:**
```
- Item name (specific: "wall-hung toilet", not just "toilet")
- Location in frame: left/center/right + foreground/mid/background
- Estimated pixel bounds: (x, y, width, height) in the image
- Confidence: High / Medium / Low
- Anchor candidate: Yes/No (is this a good scale anchor?)
```

**Categories to scan:**
- Plumbing fixtures (toilet, sink, tub, shower, faucets)
- Cabinetry (vanity, wall cabinets, base cabinets, pantry)
- Appliances (range, refrigerator, dishwasher, microwave)
- Furniture (bed, sofa, dresser, chairs, tables)
- Structural (walls, doors, windows, ceiling, floor)
- Surfaces (tile, countertops, backsplash, flooring material)
- Hardware & accessories (towel bars, mirrors, light fixtures, outlets)

**Output format for Phase 1:**
```
ITEM INVENTORY:
[1] Standard toilet — center-left, mid-ground — px: (120, 340, 95, 140) — High confidence — Anchor: YES
[2] Single sink vanity, 30" range — left wall, foreground — px: (30, 260, 180, 220) — High confidence — Anchor: YES
[3] Subway tile wall — background — fills upper 60% of back wall — Medium confidence
...
```

### Phase 2: Dimension Lookup

For each catalogued item:

1. **Check `references/dimensions.md` first** — covers 95% of standard items
2. If item is specialty, custom, or unusual → web search: `"standard dimensions [item name]"`
3. For structural elements (walls, ceiling) → use standard values unless photo shows otherwise
4. Note the dimension source: `[BUILT-IN]`, `[WEB-SEARCH]`, or `[ESTIMATED]`

Build a dimension map:
```
DIMENSION MAP:
[1] Toilet — W: 14–18" (36–46cm), D: 28–30" (71–76cm), H: 28–30" (71–76cm) [BUILT-IN]
[2] Vanity — W: 30" (76cm), D: 21" (53cm), H: 32" (81cm) [BUILT-IN + estimated width from photo]
[3] Ceiling height — H: 96" (244cm) [BUILT-IN — standard, verify if older home suspected]
```

### Phase 3: Scale Calculation

Read `references/dimensions.md` → "SCALE ANCHOR PRIORITY" section.

```
SCALE CALCULATION:
Primary anchor: [item] — real size: X" — pixel measurement: Ypx
→ Scale: 1px = X/Y inches = Z cm

Secondary anchor (if available): [item] — cross-check result
→ Agreement within 15%? YES/NO

Final scale: 1px ≈ X.X inches (X.X cm)
Note any perspective distortion or items where scale estimate may be off ±N%
```

For perspective-matched views (what this skill uses by default):
- Items closer to camera appear larger — note which items are affected
- Use the most central, upright items for scale anchor
- Flag if the photo angle makes scale derivation uncertain

### Phase 4: Build Spatial Model

Map every item's position in real-world coordinates (inches from nearest corner):

```
SPATIAL MODEL (from rear-left corner of room):
Room: W: ~96" (8ft), D: ~60" (5ft), H: ~96" (8ft) [estimated]

[1] Toilet — X: 18" from right wall, Y: 4" from back wall, Z: floor
[2] Vanity — X: left wall flush, Y: 4" from back wall, Z: floor
[3] Back wall tile — covers back wall, 0–84" height
...
```

---

## MODE 1: Blueprint Diagram

After the shared pipeline, produce a dimensioned technical drawing.

### Drawing Approach

Read `references/svg-rendering.md` → "Mode 1: Blueprint Diagram" section.

**Match the photo's angle**: reproduce the same viewpoint as the photo — if photo shows a 3/4 perspective view of the room, draw it from that angle, not top-down. If the photo is nearly straight-on to one wall, draw an elevation view.

**Perspective type selection:**
- Photo shows mostly one wall → **Elevation view** (flat, front-facing)
- Photo shows two walls + floor → **2-point perspective** (most bathrooms/kitchens)
- Photo shows top-down or looking down → **Plan view** (top-down)
- Default if unclear → **2-point perspective** at horizon = 42% down canvas

### Layer Rendering Order
1. Room shell (walls, floor, ceiling)
2. Floor surface material
3. Fixed floor-standing fixtures
4. Wall-mounted fixtures and cabinets
5. Countertops and surfaces
6. Items sitting on surfaces
7. Ceiling elements (dashed)
8. **Dimension lines** (every item gets W + H minimum; D where visible)
9. Item labels
10. Scale bar + title block

### Dimension Line Rules
- Every item: show Width and Height
- Where depth is visible: show Depth too
- Dimension lines run outside the item footprint, not through it
- Group dimensions to avoid overlapping — stagger if needed
- Show both imperial and metric: `30" (76cm)`

### Output
Produce the SVG inline (rendered in chat) AND note it's also available as a downloadable file.

Tell the user:
```
Blueprint complete. Scale: 1" on screen ≈ X real-world feet.
Items identified: [count] | Dimensions shown: [count]
Measurements marked with ~ are estimated — verify before any construction work.
[List any items where web search found conflicting standard sizes]
```

---

## MODE 2: Remodel Vision

After the shared pipeline, produce a stylized architectural illustration.

### Parse User's Remodel Prompt

Extract from the user's description:
- **Items to ADD**: name, style/finish preference, location ("left wall", "where the tub was", etc.)
- **Items to REMOVE**: what's being taken out
- **Items to CHANGE**: finish, size, style of existing items
- **Ambiguous placements**: note and apply smart defaults (see below)

### Smart Placement Rules (when location is vague)

| Item | Smart Default Placement |
|------|------------------------|
| Vanity / sink | Against longest wall, centered |
| Toilet | Alcove or corner, min 15" from side wall to centerline |
| Shower | Corner placement, away from door |
| Freestanding tub | Center of longest wall or corner |
| Kitchen island | Center of room, min 42" clearance on all sides |
| Refrigerator | End of run, near doorway |
| Range/stove | Center of counter run, away from corners |

Always tell the user the smart default you applied and offer to adjust.

### Drawing Approach

Read `references/svg-rendering.md` → "Mode 2: Remodel Vision" section.

**Style goal**: clean architectural illustration — not a photo, not a cartoon. Think: high-quality interior design rendering with clear light source, material hints, and readable proportions. Warm neutral palette.

**New items get:**
- Warm amber highlight + dashed border (see svg-rendering.md)
- "NEW:" callout label
- Their dimensions listed separately in the annotation

**Removed items**: simply omit from the drawing (don't show a ghost outline unless user asks)

**Changed items**: show the new version, add callout "UPDATED: [what changed]"

### Layer Rendering Order
1. Room shell with perspective + lighting gradients
2. Floor with material (tile, wood, etc.)
3. Existing items being kept
4. Updated versions of changed items
5. **New items** (highlighted)
6. Lighting overlay (ambient gradient)
7. Callout annotations
8. Legend: "NEW" / "UPDATED" / "EXISTING" key
9. Measurement summary panel (compact list of new item dimensions)

### After Rendering, Provide

```
Remodel Vision complete.

EXISTING items kept: [list]
UPDATED items: [list with what changed]
NEW items added: [list with dimensions + placement notes]

Smart placement applied:
- [Item]: placed [location] based on [rule]. Change this? Just say where you'd like it.

Estimated new items dimensions:
- [Item]: W×D×H — [BUILT-IN / WEB-SEARCH source]

Note: This is a visual planning tool. Verify all measurements and consult a 
licensed contractor before any construction work.
```

---

## Measurement System

**Default**: Show both imperial and metric — `30" (76cm)`

**User override**: If user says "metric only" → switch to `76cm` format. If "imperial only" → `30"` format.

Detect from user's prompt: if they use cm/mm/m → default to metric. If they use ft/in → imperial.

---

## Output Delivery

**Inline**: Always render the SVG inline in the chat so the user can see it immediately.

**Downloadable**: Always save the SVG to `/mnt/user-data/outputs/[room-type]-[mode]-[timestamp].svg` and use `present_files` tool to offer the download link.

File naming:
- Blueprint: `bathroom-blueprint.svg`, `kitchen-blueprint.svg`
- Remodel: `bathroom-remodel-vision.svg`

---

## Error Handling & Edge Cases

**Blurry or dark photo**: Proceed with low-confidence items, flag them. "I could make out X items clearly; Y items are uncertain due to image quality."

**Photo angle makes scale impossible**: Tell the user which anchor items you used and their confidence level. If no reliable anchor exists, ask the user: "Can you tell me the approximate width of your [item] or room? That'll let me calibrate the scale."

**Custom/luxury items**: If an item looks non-standard (unusual size, clearly custom-built), note it: "This [item] appears custom — I've used standard dimensions as a baseline. Measure yours for accuracy."

**Multiple rooms visible**: Focus on the primary room. If a second room is partially visible, include it as a partial with dashed walls.

**Remodel conflicts**: If user asks to add an item that won't fit given the scale (e.g., a freestanding tub in a 35" wide bathroom), flag it clearly: "⚠️ A standard freestanding tub (60") would exceed your estimated room width of ~48". Options: [corner tub, alcove tub, smaller soaking tub]."

---

## Quality Checklist (before outputting)

- [ ] Every identifiable item has a real-world dimension assigned
- [ ] Scale anchor documented with source
- [ ] Perspective matches photo angle
- [ ] All dimensions labeled (W + H minimum per item)
- [ ] Both imperial + metric shown (unless user specified one)
- [ ] Scale bar present
- [ ] Estimated measurements marked with `~`
- [ ] SVG saved to outputs directory + present_files called
- [ ] Summary text provided after SVG
