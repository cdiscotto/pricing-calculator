# CLAUDE.md

Guide for AI assistants working on the Pricing Calculator codebase.

## Project Overview

A single-page interactive pricing calculator for manufacturer and dealer cost pricing. Users enter any two values and the calculator derives the remaining fields automatically. Deployed as a static site on GitHub Pages.

**Live URL:** https://cdiscotto.github.io/pricing-calculator

**Current version:** v6.0 (Individual Field Recalculation)

## Architecture

This is a zero-dependency static web application. There is no build step, no package manager, no framework, and no bundler.

```
pricing-calculator/
  index.html      # Entire application (HTML + CSS + JS in one file)
  README.md       # Brief project description
  .nojekyll       # Disables Jekyll processing on GitHub Pages
  CLAUDE.md       # This file
```

### Technology Stack

- **HTML/CSS/JavaScript** — all vanilla, no frameworks
- **Hosting:** GitHub Pages (static deployment on push)
- **No build tools, transpilers, or dependencies**

## Pricing Logic

The calculator models a three-tier pricing flow:

```
BOM Cost  ---(+ Mfg Margin + NRE)---> Dealer Cost ---(+ Dealer Margin)---> MSRP
```

### Fields

| Field | Type | Auto-calculated? | Notes |
|---|---|---|---|
| BOM Cost | Dollar | Never | Input-only, manufacturer's bill of materials |
| Manufacturer Margin | Percent | Yes (from BOM & Dealer Cost) | Must be < 100% |
| NRE (Non-Recurring Engineering) | Dollar | Never | Input-only, added to Dealer Cost |
| Dealer Cost | Dollar | Yes (from BOM + Mfg Margin + NRE) | Includes NRE |
| Dealer Margin | Percent | Yes (from Dealer Cost & MSRP) | Must be < 100% |
| MSRP (Retail Price) | Dollar | Yes (from Dealer Cost + Dealer Margin) | Rounded up to nearest $9 |

### Key Formulas

- **Dealer Cost** = `BOM Cost / (1 - Mfg Margin / 100) + NRE`
- **MSRP** = `Dealer Cost / (1 - Dealer Margin / 100)`, then rounded up to nearest $9
- **Mfg Margin** (reverse) = `(1 - BOM Cost / (Dealer Cost - NRE)) * 100`
- **Dealer Margin** (reverse) = `(1 - Dealer Cost / MSRP) * 100`

### Auto-Update Behavior

- Editing **MSRP** auto-updates Dealer Margin (Dealer Cost stays fixed)
- Editing **Dealer Cost** auto-updates Mfg Margin (BOM stays fixed)
- **BOM Cost** and **NRE** are never auto-calculated
- A 300ms debounce (`currentlyEditing` tracker) prevents circular updates
- Individual "Recalc" buttons allow per-field recalculation independent of the auto-update flow

### Rounding Convention

`roundUpToNearest9()` rounds dollar values up to end in $9 (e.g., $142 becomes $149, $150 becomes $159). This is a retail pricing psychology convention.

## Code Structure (index.html)

The entire application lives in `index.html` (~510 lines):

- **Lines 1-222:** `<style>` block — all CSS including responsive breakpoints
- **Lines 223-305:** HTML markup — calculator UI, inputs, buttons
- **Lines 307-508:** `<script>` block — all JavaScript logic

### Key JavaScript Functions

| Function | Purpose |
|---|---|
| `calculate()` | Main calculation with 4 cases based on which field is being edited |
| `recalcMfgMargin()` | Derives Mfg Margin from BOM Cost and Dealer Cost |
| `recalcDealerCost()` | Derives Dealer Cost from BOM Cost, Mfg Margin, and NRE |
| `recalcDealerMargin()` | Derives Dealer Margin from MSRP and Dealer Cost |
| `recalcMsrp()` | Derives MSRP from Dealer Cost and Dealer Margin |
| `roundUpToNearest9(value)` | Rounds up to nearest dollar ending in 9 |
| `copyToClipboard()` | Copies formatted results text to clipboard |
| `clearAll()` | Resets all input fields |

### Important Variables

- `inputs` — Array of all 6 input DOM elements
- `currentlyEditing` — Tracks the field ID currently being edited (prevents circular recalculation); resets to `null` after 300ms of inactivity

## CSS Conventions

- Purple/blue gradient theme (`#667eea`, `#764ba2`)
- Glass-morphism card style with `border-radius: 20px` and heavy box-shadow
- Green gradient for copy button, orange gradient for recalc buttons
- Mobile breakpoint at **480px** (stacks flow diagram vertically)
- Input focus state uses blue border with subtle box-shadow ring
- `.has-prefix` class adds left padding for dollar symbol
- `.has-recalc` class adds right padding for recalc button

## Development Workflow

### Making Changes

1. All code is in `index.html` — edit that single file
2. Open `index.html` in a browser to test (no build step needed)
3. Changes deploy automatically when pushed to the main branch via GitHub Pages

### Deployment

GitHub Pages serves from the repository root. The `.nojekyll` file ensures files are served as-is. No CI/CD pipelines exist — pushing to `main` triggers automatic deployment.

### Git Conventions

- Branch naming: `claude/<description>-<id>` for AI-assisted work
- PR-based workflow with merge commits into `main`
- Commit messages describe the change and version bump (e.g., "Add NRE field, lock checkboxes, and copy button (v4.0)")

### Version Bumping

Version is displayed in the UI subtitle. When making significant feature changes, bump the version string at line 227:
```html
<p style="...">v6.0 - Individual Field Recalculation</p>
```

## Testing

There is no automated test suite. All testing is manual:

1. Open `index.html` in a browser
2. Enter values in various field combinations and verify calculations
3. Test edge cases: margins approaching 100%, zero values, clearing fields
4. Test the Copy Results button
5. Test responsive layout at mobile widths (< 480px)

## Common Tasks

### Adding a new field
1. Add HTML input markup in the appropriate position within `.calculator-container`
2. Add a `const` reference in the `<script>` section and include it in the `inputs` array
3. Update `calculate()` with the new field's logic
4. Add a `recalc<FieldName>()` function if the field should have a recalc button
5. Update `copyToClipboard()` to include the new field in output
6. Update `clearAll()` (already loops over `inputs` array, so new fields are covered if added to the array)

### Modifying calculation logic
- The main `calculate()` function handles 4 cases based on `currentlyEditing`
- Each `recalc*()` function handles one specific reverse calculation
- Auto-update listeners in the `input` event handler (lines 322-357) handle real-time updates as the user types

### Changing styling
- All CSS is in the `<style>` block (lines 7-221)
- Primary color: `#667eea` / secondary: `#764ba2`
- Mobile breakpoint: 480px
