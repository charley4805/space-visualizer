---
name: space-visualizer
version: 1.3.0
description: |
  Transform a photo of any room into a dimensioned blueprint or a stylized remodel vision.
  Use this skill whenever a user uploads a photo of a room (bathroom, kitchen, bedroom,
  living room, or any interior space) and wants EITHER:
  (A) a perspective blueprint showing structural/trade-relevant items with real-world dimensions, OR
  (B) a stylized remodel illustration that incorporates items they describe adding or changing.
  Trigger on: "blueprint this room", "show me dimensions", "what size is everything",
  "create a layout", "remodel vision", "show me with a [new item]", "redesign this space",
  "floor plan from photo", "room diagram", "add [item] to my room", "give me the blueprint".
  Always trigger when an image is uploaded alongside any spatial or remodel intent.
---

# Space Visualizer Skill — v1.3

Two modes, one shared pipeline. Read this full file before starting.

## What changed in v1.3

- **Anchor priority completely reordered.** Wall outlet plate (2.75"×4.5", UL/NEC code) is now anchor #1. Interior door on a side wall dropped to #6 — it's foreshortened, width is unconfirmed, and may be French/closet style.
- **Two-anchor convergence required.** Every blueprint must derive scale from two independent anchors and confirm they agree within 15%.
- **Product web search added to pipeline.** Identifiable furniture SKUs get a web search for published dimensions; cross-validate against pixel-derived dims.
- **Error budget table required.** Every output must include per-dimension confidence and ±error estimate.
- **Room width corrected methodology.** Width derives from back-wall pixel span ÷ back-wall scale, not from door estimates. Harper's bedroom corrected from ~11'-0" (door-anchored guess) to ~9'-8" (outlet+ceiling anchored).
- **Phase A grid overlay is mandatory** before any blueprint render.

## Reference Files
- `references/dimensions.md` — Built-in dimension tables for all common items. Read before web-searching.
- `references/svg-rendering.md` — SVG patterns, color palettes, perspective helpers. Read before drawing.

---

## Step 0: Determine Mode

**Mode 1 — Blueprint** if the user wants:
- Dimensions / measurements of the existing space
- A to-scale layout or floor plan
- "What size is everything?" / "give me the blueprint" type requests
- No remodel prompt (image only)

**Mode 2 — Remodel Vision** if the user wants:
- To add, remove, or change items in the room
- A visualization of what a renovation would look like
- Combined: "show the dimensions AND add a new vanity"

---

## TRADE-READY FILTER (applies to Mode 1 blueprints)

Blueprints are for contractors, designers, and trades — not lifestyle photography.
**Always exclude** the following from the rendered drawing (they add no spec value):

| Category | Exclude |
|---|---|
| Soft furnishings | Pillows, throws, blankets, duvet covers, cushions |
| Decor / art | Wall art, canvas prints, mirrors (unless spec'd), sculptures |
| Signage / lighting decor | Neon signs, string lights, holiday decor |
| Loose items | Books, toys, stuffed animals, clothing, plants (unless built-in) |
| Floor coverings | Area rugs (note "carpet" or "hardwood" as finish, not rug pattern) |
| Personal items | Toiletries, food, appliances not spec'd, desktop clutter |

**Always include** (structural or trade-relevant furniture/fixtures):
- All walls, ceiling plane, floor finish type
- Doors (with casing, hardware, panel style, swing direction if known)
- Windows (with trim, type, approximate glazing)
- Built-in cabinetry, shelving, vanities
- Plumbing fixtures (toilet, sink, tub, shower)
- Appliances (range, refrigerator, dishwasher — structural footprint only)
- Permanent furniture that affects space planning (bed frame, nightstand, dresser)
- Electrical: ceiling fixtures, visible outlets/switches (as keynotes)
- Ceiling features: tray/coffered/vaulted geometry, beams, texture type
- Baseboard, crown molding, chair rail (as keynotes)

If a trade has to spec, frame, rough-in, or finish around it → include it.
If a homeowner just put it there → exclude it.

---

## RENDERING METHODOLOGY — PHOTO-ACCURATE APPROACH

**Always use this 3-phase process. Never estimate positions from scratch.**

### Phase A — Grid Overlay on Photo
Before drawing any blueprint, overlay a measurement grid on the uploaded photo:
1. Embed the photo as an HTML `<image>` tag (base64-encode if needed)
2. Overlay a reference grid SVG (10% intervals of image width/height)
3. Mark the estimated vanishing point (VP) with a crosshair
4. Trace each feature in a different color:
   - Back wall: yellow `#FFD700`
   - Left/right walls: blue `#4488FF` / cyan `#00CCFF`
   - Doors: red `#FF4444`
   - Windows: teal `#00FFCC`
   - Furniture: orange `#FF8C00`
   - Ceiling features: white `#ffffff`
5. Annotate with pixel coordinates and fractional positions
6. Output as a downloadable `.html` file via `present_files`

### Phase B — Extract True Proportions
From the trace, derive real-world positions using the photo's pixel coordinates:

**For each feature, record:**
```
Feature: [name]
Pixel position: x=A→B, y=C→D in NNNxMMM image
As fraction of back wall: left=A%, right=B%
As fraction of wall height: top=C%, bottom=D%
Depth inference: feature appears at [front/middle/back] of [wall]
```

**Depth inference rules:**
- If feature's near edge aligns with the back-wall junction pixel → feature is AT THE BACK of that side wall
- If feature appears large (close to camera edge) → feature is toward the FRONT of that side wall
- Cross-check with perspective: objects further from camera appear smaller and higher

### Phase C — Render Blueprint from Traced Positions
Map traced fractions to SVG coordinates:
```
back_wall_x = 190 + fraction * 340   (for back wall items)
left_wall_x = fraction * 190         (for left wall items, 0=front, 190=back)
right_wall_x = 530 + fraction * 150  (for right wall items, 530=back, 680=front)
```

**Left wall depth (t = 0→1, front→back):**
- `ceil_y_at_t = 150 + t * 65`
- `floor_y_at_t = 490 - t * 90`
- `wall_ht_at_t = floor_y_at_t - ceil_y_at_t`

**Scale on left wall:** `1ft_horiz = 190/room_depth_ft * (pixel_x / 190)`

---

## SHARED CORE PIPELINE (run for both modes)

### Phase 1: Vision Analysis

Examine the uploaded image carefully and inventory every TRADE-RELEVANT element:

```
ITEM INVENTORY:
[1] 6-panel interior door — left wall, foreground — px: (30, 150, 80, 380) — High confidence — Anchor: YES
[2] Double-hung window — right wall, mid-ground — px: (580, 200, 80, 180) — High confidence
[3] Twin bed frame, white ornate headboard — back wall right — px: (410, 280, 130, 140) — High confidence
[4] 3-drawer nightstand, espresso — back wall center — px: (340, 310, 80, 100) — Medium confidence
[5] Pendant light — ceiling center-right — px: (450, 10, 30, 50) — High confidence
...

EXCLUDED (decor, not trade-relevant):
- Tropical art canvas, wall art
- "Harper" neon sign
- Stuffed animals, pillows on bed
- Potted plant (not built-in)
- Area rug, loose items on floor
- Items on nightstand surface
```

**Anchor object scoring and priority — use the HIGHEST scoring anchor available:**

| Rank | Object | Known size | On-axis? | Score | Why |
|------|--------|-----------|---------|-------|-----|
| 1 | **Wall outlet cover plate** | 2.75"W × 4.50"H exactly | Back or visible wall | 9.5/10 | UL/NEC code-mandated — zero variance |
| 2 | **Light switch plate** | 2.75"W × 4.50"H exactly | Any wall | 8.0/10 | Same code standard as outlet |
| 3 | **Door knob / lever** | ~2.5" dia round knob | Any wall | 7.5/10 | ANSI A156 standard, but varies by style |
| 4 | **Ceiling height** | 8'-0", 9'-0", or 10'-0" | Back wall vertical | 7.0/10 | Residential standard — cross-validate with outlet |
| 5 | **Identifiable product** | Look up exact SKU online | Back wall | 6.5/10 | Run web search to get published dims |
| 6 | **Interior door** | 80"H standard, width varies | **Side wall only** | 4.0/10 | Width unconfirmed; foreshortening on side wall; may be French/closet |
| 7 | **Floor/wall tile** | 12×12" or 24×24" if pattern clear | Back wall | 3.5/10 | Only if grout lines are clearly countable |

**NEVER use as anchor:**
- Soft furnishings, pillows, stuffed animals — no standard size
- Art, mirrors, decorative items — custom sizes
- Doors on side walls as primary anchor — foreshortening makes width unverifiable

**The door trap:** A door on a side wall looks like a great anchor (known 80"H) but width is foreshortened and unverifiable. It was historically mis-used as anchor #1. Use outlet plate or ceiling height instead.

**Two-anchor convergence (required):** Always derive scale from two independent anchors and confirm they agree within 15%. If they disagree, flag it and use the more certain anchor.

```
Example:
  Anchor 1 — Outlet plate 4.5"H: outlet_px / 4.5 → scale A
  Anchor 2 — Ceiling 8'-0": back_wall_height_px / 96 → scale B
  Agreement: |A - B| / A < 0.15 → CONVERGED → use average
  Disagreement: flag, use outlet plate (code-mandated, zero variance)
```

### Phase 2: Dimension Lookup

For each catalogued item:
1. Check `references/dimensions.md` first (covers 95% of standard items)
2. If specialty or custom → web search: `"standard dimensions [item name]"`
3. Tag each: `[BUILT-IN]`, `[WEB-SEARCH]`, or `[ESTIMATED]`

```
DIMENSION MAP:
[1] Door — W: 32" (81cm), H: 80" (203cm) [BUILT-IN — scale anchor]
[2] Window — W: ~36" (91cm), H: ~48" (122cm), sill: ~30"AFF [ESTIMATED]
[3] Twin bed frame — W: 38" (97cm), L: 75" (190cm), HDR: ~50"H [BUILT-IN + photo]
[4] Nightstand — W: ~24" (61cm), D: ~18" (46cm), H: ~26" (66cm) [ESTIMATED]
[5] Room — W: ~11'-0", D: ~10'-0", CLG: 8'-0" [ESTIMATED from door anchor]
```

### Phase 3: Scale Calculation — Two-Anchor Method

**Step 1 — Identify anchors in photo:**
Scan image for outlet plates and switch plates first. They are on the wall, rectangular, and code-mandated at exactly 2.75"W × 4.50"H.

**Step 2 — Measure anchor in pixels:**
Record the pixel bounding box of each anchor: (x_left, y_top, x_right, y_bottom).
Use the HEIGHT measurement as primary (less affected by slight angle than width).

```
Outlet plate: y_top=560, y_bottom=592 → height = 32px
Scale from outlet: 32px / 4.5" = 7.11 px/in  ← at outlet's position on wall
```

**Step 3 — Measure back wall height in pixels:**
Back wall left-side height (most reliable): y_floor_left - y_ceil_left

```
Back wall left: y_ceil=68, y_floor=620 → height = 552px
Scale from ceiling: 552px / 96" = 5.75 px/in  ← at back wall
```

**Step 4 — Reconcile. The scales will differ because objects at different depths have different pixel scales (perspective). That is expected and correct.** Use each scale only for objects at the same depth/wall as that anchor.

```
FINAL CALIBRATED SCALES:
  Back wall objects:  5.75 px/in  (from ceiling anchor — most reliable for back wall)
  Outlet position:    7.11 px/in  (use only for objects at same depth as outlet)
  
  Cross-check: outlet plate predicted size at back-wall scale:
    4.5" × 5.75 = 25.9px  vs measured 32px → ~23% larger
    → Outlet is closer to camera than back wall (expected) — scales are consistent ✓
```

**Step 5 — Derive all room dimensions using back-wall scale:**

```
DIMENSION DERIVATION:
  Room width:    back_wall_px_width / 5.75 px/in
  Ceiling:       ANCHOR (96" / 8'-0")
  Nightstand W:  nightstand_px_width / 5.75 px/in
  (all back-wall objects use same scale)
```

**Step 6 — Cross-validate with product search:**
For any identifiable furniture or product visible in the photo, run a web search for the exact SKU or model to get published dimensions. If published dims match your pixel-derived dims within 15% → confidence HIGH. If they don't match → flag discrepancy.

```
Example:
  Nightstand derived: ~23"W from pixel measurement
  Web search "espresso 3-drawer nightstand dimensions":
    → Prepac Fremont EDC-2403: 23"W × 16"D × 29"H
    → Agreement within 2% → confidence HIGH ✓
```

**Step 7 — Assign error budget to every dimension:**

```
ERROR BUDGET TABLE (required in every blueprint output):
Object              Method          Scale used    ±Error
──────────────────  ──────────────  ─────────     ──────
Ceiling height      ANCHOR          N/A           0"
Outlet plate        ANCHOR          N/A           0"
Room width          pixel measure   5.75 px/in    ±6"
Nightstand W×H      pix + product   5.75 px/in    ±2"
Bed headboard H     pixel measure   5.75 px/in    ±6"
Window H            right wall      4.81 px/in    ±8"  (foreshortened wall)
Room depth          VP geometry     N/A           ±18" (requires focal length)
```

Note: side wall objects (doors, windows on left/right walls) have higher error because the wall itself is foreshortened. Report their dims with wider error bars.

### Phase 4: Build Spatial Model

```
SPATIAL MODEL (from rear-left corner):
Room: W: ~116" (9'-8"), D: ~120" (10'-0" est.), H: 96" (8'-0" anchor)

[1] Door — left wall, ~18" from back wall corner (traced from photo)
[2] Window — right wall, sill ~30"AFF, mid-depth on wall
[3] Bed frame — back wall right, flush to right wall
[4] Nightstand — back wall, 23"W × 29"H, left of bed
[5] Outlet — back wall, visible near baseboard — PRIMARY SCALE ANCHOR
```

**Always document anchor objects in the spatial model with "PRIMARY SCALE ANCHOR" or "CROSS-VALIDATE" tags.**

---

## MODE 1: PERSPECTIVE BLUEPRINT (DEFAULT)

After the shared pipeline, produce a **1-point perspective architectural drawing** — the same visual language as a professional interior design rendering, but stripped to trade-relevant content only with full dimension callouts.

> **Why perspective, not flat elevation?**
> Perspective blueprints communicate spatial relationships, furniture clearances, and wall adjacencies far better than flat elevations for initial design review. Flat CAD elevations are offered as a follow-up for individual walls when trades need spec-level detail.

### Perspective Setup

Read `references/svg-rendering.md` → "Perspective Rendering" section.

**SVG canvas:** `viewBox="0 0 680 595"`
**Outer border:** double rect (1px outer, 0.3px inner) at 5px inset

**Vanishing point:** `VP = (360, 278)` — centered horizontally, ~46% down from canvas top

**Room box geometry (11'W × 10'D × 8'H standard bedroom):**
```
Back wall:  x=190→530  y=215→400  (340px W = 11', 185px H = 8')
Front ceil: y=150
Front floor:y=490
Left wall:  (0,150)→(190,215)  top edge
            (0,490)→(190,400)  bottom edge
Right wall: (530,215)→(680,150) top edge
            (530,400)→(680,490) bottom edge
```

Adjust proportionally for non-standard room sizes. Scale factor = `340px / room_width_feet`.

**Scale at back wall:** `1ft = 30.9px`, `1in = 2.57px`
**Scale at back wall (vertical):** `1ft = 23.1px`, `1in = 1.93px`

### Perspective Grid Overlay (REQUIRED)

Draw a faint 2'-0" interval grid on ALL surfaces. Grid helps locate wall centers, furniture clearances, and rough-in positions without counting pixels.

**Color:** `#2840a0` at `opacity="0.12–0.15"` for normal lines, `opacity="0.35–0.45"` for centerlines
**Weight:** 0.5px normal lines, 1.0px centerlines (horizontal and vertical center of each surface)

**Back wall grid:**
```
2ft = 61.8px horizontally (340px/11ft * 2 = 61.8)
2ft = 46.3px vertically (185px/8ft * 2 = 46.3)

Vertical lines at x: 252, 314, 360(CTR), 438, 500
Horizontal lines at y: 261, 308(CTR), 354

Center crosshair: circle r=5 + crosshairs at (360, 308)
Center label: "CTR" in 7pt monospace at crosshair
```

**Left wall grid (perspective-correct):**
- Horizontal lines extend from back wall grid at y=261,308,354 following VP rays:
  - At y=261: `slope=(261-278)/(190-360)=0.1` → at x=0: y=242
  - At y=308: `slope=-0.176` → at x=0: y=341
  - At y=354: `slope=-0.447` → at x=0: y=440
- Vertical depth lines at x=38, 76, 114, 152 (every 2ft of room depth)

**Right wall grid (mirror of left):**
- At y=261: at x=680: y=246 | At y=308: y=334 | At y=354: y=421
- Vertical depth lines at x=568, 606, 644

**Floor grid:**
- Radial lines from VP through back-floor points at x=252, 314, 360, 438, 500
  - Front x = `360 + (x_back - 360) * 1.738` (factor = (490-278)/(400-278))
- Cross lines at y=420, 445, 468

**Ceiling grid:** same vertical lines as back wall, clipped to ceiling polygon

### Color Palette

```
Background:       #f8f4ec  (warm paper)
Back wall:        #f0ece4  (lightest — farthest)
Left wall:        #dcdad0  (medium — shadowed)
Right wall:       #e4e0d6  (medium-light)
Ceiling:          #d8d4cc  (flat/popcorn: add dot texture; tray: show step geometry)
Floor-hardwood:   #1c1408  (dark — with converging plank lines)
Floor-carpet:     #c8c0b0  (warm gray — with cross-hatch texture)
Object lines:     #1a1610
Grid:             #2840a0 at opacity 0.12–0.15
Dim lines:        #1a5aaa
Dim text/boxes:   #0a2060 on #eef3ff
Keynote boxes:    #0a2060 on #dde8ff
Title block:      #d4e2f8 bg, #b8ccec scale box
```

### Line Weight Hierarchy

```
2.5px — Front floor edge (viewer edge — heaviest)
1.8px — Left / right outer room edges
1.5px — Left-floor / right-floor wall junction lines
1.2px — Wall outline (back, left, right planes)
0.9px — Primary furniture outlines
0.8px — Door face, window frame
0.7px — Secondary furniture faces, casing
0.5px — Panel details, rail lines, drawer lines
0.4px — Grain texture, tile joints, detail lines
0.35px — Carpet texture, subtle fills
```

### Door Drawing Standard

Always draw the full door assembly, not just a rectangle.

**Components:**
1. Casing head (4px tall parallelogram above door opening)
2. Casing left and right jambs (4px wide)
3. Door face (perspective trapezoid matching left wall angle)
4. Warm light hint: thin vertical fill on hinge side (warm cream, 15–20% opacity)
5. Center stile (vertical line at horizontal midpoint of door face)
6. Rails (converging lines across door face):
   - Top rail: 5"/80" from door top
   - Mid rail: 40"/80"
   - Second mid rail: 44"/80" (creates lock rail zone)
   - Bottom rail: 75.5"/80"
7. Six panel insets (thin 0.3px, low opacity) in 3 pairs
8. Knob: filled circle at ~36"AFF, near latch jamb
9. Hinges: small rectangles at ~12" and ~60"AFF on hinge side

**Foreshortening on left wall:**
- Door width on left wall: `32" / (room_depth_in") * left_wall_px`
- Door height at each x: `(80"/96") * wall_height_at_x`
- `wall_height_at_x = floor_y_at_x - ceil_y_at_x`
- `ceil_y_at_x = 150 + t*(215-150)` where `t = x/190`
- `floor_y_at_x = 490 + t*(400-490)` where `t = x/190`

### Window Drawing Standard

**Components:**
1. Outer casing (parallelogram, 4px wider on all sides than frame)
2. Left and right jambs (thin vertical parallelograms)
3. Head (top flat face)
4. Sill (bottom face, slightly thicker)
5. Glass pane (blue-gray fill, 50% opacity)
6. Center rail at sill/head midpoint (double-hung window separator)
7. Blind slats (8–12 horizontal lines at 7–8px intervals, 75% opacity)
8. Exterior trim shadow (thin dark parallelogram on far side of casing)

**Foreshortening on right wall:**
- Window on right wall: horizontal dimension foreshortened by depth ratio
- `t = (x_window - 530) / 150`
- `ceil_y_at_x = 215 + t*(150-215)`
- `floor_y_at_x = 400 + t*(490-400)`
- `wall_ht_at_x = floor_y - ceil_y`
- `1in_at_x = wall_ht_at_x / 96`
- Sill y: `floor_y - sill_height_inches * 1in_at_x`
- Head y: `floor_y - head_height_inches * 1in_at_x`

### Furniture Drawing Standard

Draw only the structural frame — no cushions, no fabric, no loose items on surfaces.

**Bed frame:**
- Headboard as flat back-wall rectangle + ornate arch path above
- Left and right posts (taller than headboard, with finial circles)
- Panel detail: inner rect + carved arc paths at 0.35–0.4px weight
- Left side rail: perspective parallelogram from headboard to footboard
- Footboard: foreshortened front face of frame
- Foot posts at footboard ends
- No mattress, no pillows, no bedding, no stuffed animals

**Nightstand / dresser:**
- Front face rectangle against back wall
- Top face: thin perspective parallelogram (2–3px visible)
- Right side face: thin parallelogram (visible depth)
- Drawer rail lines
- Brass/metal pulls: small empty rectangles with colored stroke
- Nothing on top surface

**Ceiling fixtures:**
- Pendant: canopy ellipse + drop rod line + shade trapezoid + bottom ellipse
- Recessed: circle at ceiling with concentric inner ring
- Fan: motor ellipse + drop rod + blade polygons (4–5 blades, foreshortened)
- Pull chain: dashed thin line below fixture

### Dimension Line Standard

**All dimension text MUST be legible.** Minimum font size 10.5px. Use labeled boxes.

**Dimension line anatomy:**
```
Extension line:  0.55px weight, #1a5aaa, starts at object edge
Tick marks:      1.5–1.6px, 45° slash at both ends: (x-4,y+4)→(x+4,y-4)
Dim line:        1.0px weight, #1a5aaa, between tick marks
Label box:       rect fill #eef3ff, stroke #1a5aaa 0.5px, rx=2, padding ~5px
Label text:      10.5–13.5px monospace, #0a2060, font-weight="bold" for primary dims
```

**Mandatory dimensions (all blueprints):**
1. Overall room width — top of canvas, y≈38, text 13.5px bold
2. Ceiling height — right side vertical, text 12.5px bold + "CLG. HT." label
3. Door: width (across face) + height (vertical on left side)
4. Window: width (note) + height (right side vertical)
5. Each piece of furniture: width + height minimum
6. Any offset dims (e.g., "7'-8" TO DOOR" from back wall)

**Placement rules:**
- Overall room width: top, clear of all other elements
- Sub-dims: staggered below overall at y≈55, use dashed dim line for reference
- Right side dims: stack at x≈544+, clear of right wall casing
- Left side dims: x≈9–25, vertical, for door height
- Furniture dims: just above the item, with short extension lines
- Never overlap keynote boxes with dim boxes — stagger vertically

### Keynote Leaders

Numbered circles ① ② ③ ... for each catalogued trade item.

**Leader anatomy:**
- Filled circle, r=3.5, fill=#0a3080, on or near the item center
- Thin line (0.5px, #0a3080) from circle to label box
- Label box: rect #dde8ff fill, #0a3080 stroke 0.5px, rx=2
- Label text: 10.5px bold item name + 9px spec detail (type, size, material)

**Keynote placement zones:**
- Door keynote: upper-left corner (x=8, y=152 area)
- Window keynote: upper-right corner (x=490, y=152 area)
- Ceiling keynote: centered just below ceiling polygon
- Furniture keynotes: compact labels at bottom of canvas (above title block), connected by leaders

### Title Block

```
Bottom band: y=516→586 (70px tall)
Fill: #d4e2f8, border stroke #0a3080 1.2px
Double inner line: 9px inset, 0.3px

Left zone:
  Revision badge: circle r=15, fill #0a3080, white letter "A"
  Title: 13pt bold monospace — "PERSPECTIVE VIEW A — [ROOM NAME]"
  Subtitle 1: 9pt — wall layout summary (door/window/furniture positions)
  Subtitle 2: 9pt — room dimensions + finishes
  Subtitle 3: 8.5pt — "TRADE BLUEPRINT — DECOR EXCLUDED · SPACE VISUALIZER v1.2 · ANTHROPIC"

Right zone (x=490, w=186):
  Fill: #b8ccec
  Lines: "1-PT PERSPECTIVE", VP coords, anchor item, grid interval, "~ = ESTIMATED"
```

### Output Summary (after rendering)

```
Blueprint complete — [ROOM NAME], View A.
Scale anchor: [item] at [W × H]
VP: (360, 278) | Grid: 2'-0" intervals | Room: ~[W]' × ~[D]' × [H]'

Trade items shown (N):
  ① [item]: [dims] [source]
  ② ...

Excluded (decor): [brief list]

~ = estimated dimension — field verify before construction.
[Offer: "Want a flat CAD elevation of the [wall name] wall for spec-level detail?"]
```

---

## MODE 2: REMODEL VISION

After the shared pipeline, produce a stylized architectural illustration.

### Parse User's Remodel Prompt

Extract:
- **Items to ADD**: name, style/finish preference, location
- **Items to REMOVE**: what's being taken out
- **Items to CHANGE**: finish, size, style of existing items
- **Ambiguous placements**: apply smart defaults (see below)

### Smart Placement Rules

| Item | Smart Default |
|------|--------------|
| Vanity / sink | Against longest wall, centered |
| Toilet | Alcove or corner, min 15" from side wall to centerline |
| Shower | Corner, away from door |
| Freestanding tub | Center of longest wall or corner |
| Kitchen island | Center, min 42" clearance all sides |
| Refrigerator | End of run, near doorway |
| Range/stove | Center of counter run, away from corners |

Always tell the user the smart default applied and offer to adjust.

### Drawing Approach

**Style goal**: clean architectural illustration — warm neutral palette, clear light source, material hints, readable proportions.

**New items get:**
- Warm amber highlight + dashed border
- "NEW:" callout label
- Separate dimension annotation

**Removed items**: omit from drawing
**Changed items**: show new version, callout "UPDATED: [what changed]"

### Layer Rendering Order
1. Room shell with perspective
2. Floor with material finish
3. Existing items being kept
4. Updated versions of changed items
5. **New items** (highlighted amber)
6. Callout annotations
7. Legend: "NEW / UPDATED / EXISTING" key
8. Measurement summary panel

### After Rendering, Provide

```
Remodel Vision complete.

EXISTING items kept: [list]
UPDATED items: [list with what changed]
NEW items added: [list with dimensions + placement notes]

Smart placement applied:
- [Item]: placed [location] — change this? Just say where.

Note: Verify all measurements and consult a licensed contractor before construction.
```

---

## Measurement System

**Default**: Both imperial and metric — `32" (81cm)`
**User override**: "metric only" → `81cm`; "imperial only" → `32"`
**Auto-detect**: user says cm/mm → metric; user says ft/in → imperial

---

## Output Delivery

**Inline**: Always render SVG inline in chat via the Visualizer tool.

**Downloadable**: Save SVG to `/mnt/user-data/outputs/[room-type]-blueprint-v[n].svg` and call `present_files`.

File naming:
- Blueprint: `bedroom-blueprint.svg`, `kitchen-blueprint.svg`
- Remodel: `bathroom-remodel-vision.svg`

---

## Error Handling

**Blurry / dark photo**: Proceed with low-confidence items flagged. "N items clearly visible; M items uncertain — flagged with ~."

**No reliable anchor**: Ask: "Can you tell me the approximate width of your [door / room]? That calibrates the scale."

**Custom items**: "This [item] appears custom — using standard dims as baseline. Measure yours."

**Multiple rooms visible**: Focus primary room; partial second room with dashed walls.

**Item doesn't fit**: Flag clearly: "⚠️ A standard [item] ([W]") exceeds estimated room width of ~[W]". Options: [alternatives]."

---

## Quality Checklist (before outputting)

- [ ] Trade filter applied — decor excluded, structural items included
- [ ] Outlet plate or switch plate identified as primary anchor (NOT door on side wall)
- [ ] Two anchors derived and confirmed within 15% agreement
- [ ] Product web search run for any identifiable furniture SKU
- [ ] Error budget table completed for every dimension
- [ ] Room width derived from back-wall pixel span ÷ back-wall scale (not guessed)
- [ ] Phase A grid overlay HTML produced and shared via present_files
- [ ] VP set correctly, room box geometry matches photo angle
- [ ] Grid overlay on all surfaces (back wall, left, right, floor, ceiling)
- [ ] Back wall center crosshair labeled "CTR"
- [ ] All dimension text ≥ 10.5px in labeled boxes with #eef3ff fill
- [ ] Tick marks (not arrows) at all dimension endpoints, 1.5px weight
- [ ] Overall room width dim at top (13.5px bold)
- [ ] Ceiling height dim on right side
- [ ] Each furniture item has W + H dims minimum
- [ ] Keynote leaders numbered ① ② ③ with spec boxes
- [ ] Title block complete with VP, anchor used, scale, error note
- [ ] "~" prefix on all estimated dims, confidence level noted
- [ ] SVG saved to outputs dir + `present_files` called
- [ ] Output summary text with error budget provided after SVG
