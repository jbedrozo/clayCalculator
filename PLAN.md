# Clay Calculator — Project Plan

## Overview

A personal pottery planning tool hosted on GitHub Pages. Calculates throwing sizes based on clay shrinkage rates, estimates clay weight, and saves project data across devices via the GitHub Contents API. Single `index.html` file, no build step, no dependencies to install.

**Repo:** `github.com/jbedrozo/clayCalculator`
**Hosting:** GitHub Pages (main branch)
**Data storage:** GitHub Contents API (reads/writes JSON files in the repo)

---

## File Structure

```
clayCalculator/
├── index.html
├── PLAN.md
├── data/
│   ├── saved-calculations.json   # initialized as []
│   └── custom-clays.json         # initialized as []
└── images/
    ├── laguna-speckled-buff.png
    ├── laguna-bmix5.png
    ├── laguna-bmix5-speckles.png
    ├── cac-oregon-brown.png
    ├── cac-walnut.png
    └── sps-seamix6.png
```

---

## Tech Stack

- Single `index.html` — all CSS and JS inline, no build step
- Google Fonts: **Lora** (serif, headings) + **DM Sans** (sans, body/UI)
- No frameworks, no npm, no bundler
- GitHub Contents API for cross-device data persistence
- `localStorage` only for storing the GitHub PAT on each device

---

## Formulas

### Throwing size
```
throwing_size = desired_size / (1 - shrinkage_pct / 100)
```
Applied uniformly to height, width, and depth.

### Clay weight estimation
1. Calculate bounding box volume from desired dimensions (H × W × D)
2. Apply taper multiplier to approximate actual form volume:
   - Straight-walled: 1.0
   - Tapers at base: 0.65
   - Tapers at shoulder: 0.65
   - Tapers at both: 0.45
3. Estimate shell volume using wall thickness and foot thickness
4. Convert volume to weight using wet clay density (~1.9 g/cm³)
5. Apply 15% trimming loss factor
6. Output result in pounds

### Unit conversion
- Default: inches
- Toggle to centimeters (1 in = 2.54 cm)
- All internal calculations in inches; convert display only

---

## Data Models

### Clay body
```json
{
  "id": "laguna-speckled-buff",
  "brand": "Laguna",
  "name": "Speckled Buff",
  "sku": "WC403",
  "color": "Tan with speckles",
  "type": "Stoneware",
  "coneMin": 5,
  "coneMax": 6,
  "shrinkage": 12,
  "absorption": 3,
  "image": "images/laguna-speckled-buff.png",
  "custom": false
}
```

### Saved calculation
```json
{
  "id": "uuid-string",
  "projectName": "My mug set",
  "notes": "For the farmers market",
  "clayId": "laguna-speckled-buff",
  "formType": "cylinder",
  "taper": "straight",
  "unit": "in",
  "desiredHeight": 4,
  "desiredWidth": 3.5,
  "desiredDepth": 3.5,
  "wallThickness": 0.25,
  "footThickness": 0.375,
  "throwingHeight": 4.55,
  "throwingWidth": 3.98,
  "throwingDepth": 3.98,
  "estimatedWeightLbs": 1.4,
  "savedAt": "2025-05-29T12:00:00Z"
}
```

---

## Preloaded Clay Bodies

All are cone 5–6 stoneware unless noted.

| Brand | Name | SKU | Shrinkage | Absorption | Color |
|---|---|---|---|---|---|
| Laguna | Speckled Buff | WC403 | 12% | 3% | Tan with speckles |
| Laguna | B-Mix 5 | WC401 | 12% | 2.3% | Light gray (fires cream) |
| Laguna | B-Mix 5 with Speckles | WC408 | 12% | 2.75% | Light gray with speckles |
| Clay Art Center | Oregon Brown Smooth | CL221 | 12.9% | n/a | Red/brown (fires dark brown) |
| Clay Art Center | Walnut Smooth | — | 13% | 0.2% | Light brown (fires medium brown) |
| Seattle Pottery Supply | Sea Mix 6 | SP648 | 11.32% | 1.64% | Off-white |

---

## GitHub Contents API

### Authentication
- On first use, prompt the user to enter their GitHub PAT
- Store PAT in `localStorage` as `clay_calculator_pat`
- Never commit PAT to the repo
- PAT should be a fine-grained token with **Contents: Read and Write** scoped to `jbedrozo/clayCalculator` only

### Reading data (no auth required for public repo)
```
GET https://api.github.com/repos/jbedrozo/clayCalculator/contents/data/{filename}.json
```
Response includes `content` (base64-encoded) and `sha` (required for writes).

### Writing data
```
PUT https://api.github.com/repos/jbedrozo/clayCalculator/contents/data/{filename}.json
Authorization: Bearer {PAT}
Body: {
  "message": "update {filename}",
  "content": "{base64-encoded JSON}",
  "sha": "{current file sha}"
}
```

### Strategy
- On app load: fetch both JSON files from GitHub, cache in memory
- On save/delete: read current file SHA, then PUT updated content
- Show a loading state while syncing
- Show a clear error if the PAT is missing or the write fails
- Last-write-wins (acceptable for solo use)

---

## UI Layout

### Navigation
- Fixed bottom nav bar with three tabs: **Calculator**, **Saved**, **Library**
- Active tab uses deep olive `#4A5240`
- Safe area insets for mobile (env(safe-area-inset-bottom))

### Color Palette
```css
--bg:            #F5F0E8;   /* page background, porcelain off-white */
--surface:       #FDFAF4;   /* card surface, cream white */
--surface-alt:   #E8E4D8;   /* input backgrounds, section dividers */
--olive-deep:    #4A5240;   /* primary buttons, active nav, key values */
--olive-mid:     #6B7560;   /* secondary buttons, borders */
--sage:          #8A9070;   /* inactive nav, muted UI */
--sage-light:    #C5C9B8;   /* borders, dividers */
--terra-deep:    #8B4513;   /* accent text on tint */
--terra-mid:     #C1714A;   /* accent borders */
--terra-tint:    #F0D9C8;   /* badge/highlight backgrounds */
--text-primary:  #2C2418;   /* main text, warm dark brown */
--text-secondary:#5C5242;   /* labels, secondary text */
--text-muted:    #8A7E6E;   /* hints, placeholders */
```

### Typography
```css
--font-serif: 'Lora', Georgia, serif;       /* headings, app name */
--font-sans:  'DM Sans', system-ui, sans-serif; /* all UI text */
```

### Spacing & Radius
- Page padding: 16px horizontal
- Card radius: 12px
- Input radius: 8px
- Bottom nav height: 64px + safe area inset

---

## Tab 1: Calculator

### Inputs (stacked, single column)
1. **Unit toggle** — `in` / `cm` pill toggle, default inches
2. **Clay selector** — searchable dropdown, options grouped by type, each option shows clay thumbnail + name + brand
3. **Form type** — dropdown: Cylinder/Mug, Bowl, Plate/Shallow Dish, Vase, Lidded Jar, Tumbler
4. **Desired dimensions** — three inputs side by side: Height, Width, Depth
5. **Taper** — dropdown: Straight-walled, Tapers at base, Tapers at shoulder, Tapers at both
6. **Wall thickness** — number input, default 0.25 in
7. **Foot/base thickness** — number input, default 0.375 in

### Dynamic illustration
- SVG cross-section profile rendered live as inputs change
- Shows outer wall, inner hollow, wall thickness as visible inset, foot ring at base
- Silhouette shape driven by form type + taper selection
- Proportions driven by H/W/D values
- Style: clean architectural line drawing, thick outer contour, thinner inner wall line, light gray fill on wall cross-section

### Results: Throwing dimensions
Card showing:
- Throwing Height, Width, Depth (large olive values)
- Clay name, shrinkage %, cone range, absorption % as reference info below

### Results: Estimated clay weight
Separate card below showing:
- Estimated weight in lbs (large value)
- Small note: "approximate — includes 15% trim loss"

### Save button
- "Save this calculation" button at bottom
- Opens an inline form (not a modal — no fixed positioning) with: Project name (required), Notes (optional textarea)
- Confirm save → writes to GitHub

---

## Tab 2: Saved Calculations

### Layout
- Projects grouped by clay body (section header shows clay thumbnail + name)
- Within each group, cards sorted by most recently saved
- Each card shows:
  - Project name (bold)
  - Form type + taper
  - Desired dimensions → throwing dimensions (e.g. "4 × 3.5 × 3.5 in → 4.55 × 3.98 × 3.98 in")
  - Estimated weight
  - Notes (if any, muted)
  - Date saved
  - Delete button (trash icon, requires confirmation tap)

### Empty state
Friendly message if no calculations saved yet.

### Export
"Export JSON" button at top — downloads `saved-calculations.json` as a local backup.

---

## Tab 3: Clay Library

### Layout
- Scrollable list of clay cards
- Each card shows: thumbnail image, brand + name, type badge, cone range, shrinkage %, absorption %
- Preloaded clays: cannot be deleted
- Custom clays: marked with a small "custom" badge (terracotta tint), can be deleted

### Add custom clay form
Below the list, a form to add a new clay body:
- Brand (text)
- Name (text)
- Color (text)
- Type (dropdown: Stoneware, Porcelain, Earthenware, Raku, Other)
- Cone min / Cone max (number inputs)
- Shrinkage % (number)
- Absorption % (number)
- Image (optional — file input; if provided, converts to base64 and stores inline since binary uploads to GitHub are complex)
- Save button → appends to `custom-clays.json` on GitHub

---

## Settings / PAT Setup

### First-run flow
- If no PAT found in `localStorage`, show a setup banner at the top of the app
- Banner explains: "To save your work across devices, enter your GitHub personal access token."
- Link to GitHub PAT creation page
- Input field (password type) + Save button
- Once saved to `localStorage`, banner dismisses and data sync activates

### Offline / error handling
- If GitHub API call fails, show a small toast: "Sync failed — check your connection or PAT"
- App still works for calculation even without a PAT (just can't save)

---

## Mobile Considerations

- All tap targets minimum 44px height
- Inputs large and clearly labeled
- Bottom nav accounts for iOS safe area (`padding-bottom: env(safe-area-inset-bottom)`)
- SVG illustration scales gracefully on narrow screens
- Saved projects tab optimized for quick vertical scanning
- No horizontal scrolling anywhere

---

## Deployment Steps

1. Repo already exists at `jbedrozo/clayCalculator` with all images uploaded to `images/`
2. Add `index.html` to the repo root
3. Add `data/saved-calculations.json` (initialized as `[]`) to the repo
4. Add `data/custom-clays.json` (initialized as `[]`) to the repo
5. Enable GitHub Pages: Settings → Pages → Source: main branch, root `/`
6. Create a fine-grained PAT: GitHub → Settings → Developer Settings → Fine-grained tokens
   - Scope: only `jbedrozo/clayCalculator`
   - Permissions: Contents → Read and Write
7. Visit the live URL, enter PAT when prompted, start planning

---

## Out of Scope (future ideas)
- Glaze calculator
- Firing log
- Photo uploads for finished pieces
- Multi-user / sharing
