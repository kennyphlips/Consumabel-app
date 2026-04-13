# CLAUDE.md — Consumabel-app

## Project Overview

**Magazijn Master** is a warehouse consumable-ordering application for technicians at TVH Equipment. It is a zero-dependency, single-file vanilla HTML/CSS/JS SPA (~370 lines). Orders are submitted via a `mailto:` link to `request.backoffice@tvhequipment.com`.

- **Language of the UI**: Dutch (all user-facing labels, alerts, and placeholders are in Dutch)
- **Target users**: Warehouse technicians on mobile and desktop
- **Runtime environment**: Any browser; no build step, no server — open `index.html` directly

---

## Repository Structure

```
Consumabel-app/
├── index.html     # Entire application: HTML markup, inline CSS, inline JavaScript
└── README.md      # Minimal title-only readme
```

Everything lives in `index.html`. There are **no** external files, modules, node_modules, package.json, bundlers, or test suites.

---

## Architecture

### Single-file SPA

The file is divided into three logical sections:

| Section | Lines (approx.) | Purpose |
|---|---|---|
| `<head>` / CSS | 1–53 | Theming (CSS variables), layout, responsive styles |
| `<body>` HTML | 55–81 | Static markup: container, nav-bar, grid, cart, footer button |
| `<script>` JS | 83–367 | All application logic |

### State (in-memory only)

```js
let winkelwagen = [];     // Shopping cart: [{ id: string, aantal: number }]
let navigatiePad = [];    // Navigation stack: [string]  (category path)
```

`localStorage` stores only the user name under key `"magazijnUser"`.

### Navigation model

`navigatiePad` is a stack of category-key strings. Pushing a key navigates deeper; popping navigates back. `renderMenu()` always rebuilds the grid from scratch based on the current stack.

---

## Data Catalog (`DATA` object)

Product catalog is hard-coded at the top of `<script>` (lines 86–224). The shape is a recursive tree:

```
DATA
└── <TOP_CATEGORY>          // e.g. "ELECTRONICS"
    ├── img: URL
    └── sub
        └── <SUB_CATEGORY>  // e.g. "ZEKERINGEN"
            ├── img: URL
            └── sub
                └── <LEAF_GROUP>    // e.g. "STANDARD"
                    ├── img: URL
                    └── sub: [ { name, pn, specs }, ... ]   // leaf items
```

`sub` at a non-leaf node is a **plain object** keyed by category name.  
`sub` at a leaf node is an **array** of item objects.  
The leaf detection in `renderMenu()` is `Array.isArray(content)`.

### Current top-level categories

| Key | Prefix | Description |
|---|---|---|
| `ELECTRONICS` | `EL_` | Fuses, lighting, plugs, cable connectors |
| `FASTENERS` | `FD_` | Bolts, nuts, washers |
| `CHEMIE` | `CT_` | Loctite, brake clean, oils |
| `BATTERIJEN` | `VT_` | Varta batteries |

Prefixes are defined in the `PREFIXES` object (line 84) and prepended to part numbers in cart display.

### Item object shape

```js
{ name: "Standard 5A Beige", pn: "S-05", specs: "Hella" }
// name  — display name shown in the grid and cart
// pn    — part number (will be prefixed on add to cart)
// specs — small descriptor shown below name in the item card
```

---

## Key Functions Reference

| Function | Description |
|---|---|
| `renderMenu()` | Redraws the item grid based on `navigatiePad`. Auto-detects leaf vs. category level. Uses `createElement` + `textContent` throughout. |
| `renderItems(items)` | Renders an array of leaf items as `.card.is-item` cards. Shows "Geen resultaten gevonden." when the array is empty. Triggers a CSS flash animation on the card when clicked. |
| `zoekArtikel()` | Recursive full-text search (name + pn) across entire `DATA` tree; triggers on `oninput` when query ≥ 2 chars. Sets `catPrefix` on each result so the correct part number prefix is preserved. Title shows result count. |
| `gaDieper(sleutel)` | Pushes a key onto `navigatiePad` and calls `renderMenu()`. |
| `gaTerug()` | If a search is active, clears the search and restores the current navigation level. Otherwise pops from `navigatiePad` and calls `renderMenu()`. |
| `voegToe(naam, pn, explicietePrefix)` | Adds item to `winkelwagen`. `explicietePrefix` is supplied by search results; falls back to `PREFIXES[navigatiePad[0]]` during normal navigation. Deduplicates by full name; increments `aantal` if duplicate. |
| `aanpassenAantal(index, delta)` | Adjusts cart item quantity; removes item if quantity reaches 0. |
| `verwijderItem(index)` | Removes item from `winkelwagen` by index. |
| `updateWinkelwagenLayout()` | Rebuilds the cart DOM from `winkelwagen` using `createElement`/`textContent` (no `innerHTML`). Updates the cart badge count and disables the send button when the cart is empty. Shows "Nog geen artikelen toegevoegd." placeholder when cart is empty. |
| `saveUser()` | Persists the name field value to `localStorage` under key `"magazijnUser"`. |
| `saveExtraInfo()` | Persists the extra info textarea value to `localStorage` under key `"magazijnExtraInfo"`. |
| `verstuurBestelling()` | Validates name + non-empty cart, shows `confirm()` dialog, then builds a `mailto:` URL using `encodeURIComponent` for subject and body. |

---

## CSS Conventions

- **CSS variables** in `:root` for all theme colors; a `@media (prefers-color-scheme: dark)` block overrides them. Never hard-code colors directly — always use `var(--*)`.
- **Primary accent color**: `#1a73e8` (Google-blue)
- **Success / send button**: `#28a745`
- **Danger / delete**: `#dc3545`
- **Max container width**: `700px` (centered, responsive)
- **Body bottom padding**: `140px` to prevent the fixed footer button from overlapping cart content
- **Font**: `-apple-system, system-ui, sans-serif` (system stack, no web fonts)
- **Focus styles**: Use `:focus-visible` (not `:focus`) to show outlines only for keyboard users.
- **Card hover**: `.card` has a subtle `translateY(-2px)` lift on hover via `transition`.
- **Add-to-cart flash**: `.card.is-item.flash` triggers a `@keyframes addFlash` animation (blue highlight). The class is removed and re-added after a forced reflow to allow re-triggering.

### CSS variable names

```
--bg-color          Background of <body>
--container-bg      White card container
--text-color        Primary text
--card-bg           Item/category card background
--card-border       Card border color
--input-bg          Input and textarea background
--nav-bg            Nav-bar and cart-item background
--muted-color       Secondary/placeholder text (e.g. item specs, empty messages)
--img-placeholder   Background shown while category images load
```

---

## Development Workflow

### Running the app

No build step. Open `index.html` directly in a browser, or serve it with any static file server:

```bash
# Python
python3 -m http.server 8080

# Node (npx)
npx serve .
```

### Adding a new top-level category

1. Add an entry to `PREFIXES` (line 84):
   ```js
   const PREFIXES = { ..., "NEW_CAT": "NC_" };
   ```
2. Add the matching entry to `DATA` (after line 224):
   ```js
   "NEW_CAT": {
       img: "https://loremflickr.com/320/240/keyword",
       sub: { ... }
   }
   ```

### Adding products to an existing category

Locate the appropriate leaf `sub` array in `DATA` and append an item object:
```js
{ name: "Descriptive Name", pn: "PART-NO", specs: "Brand or spec detail" }
```

### Changing the recipient email

Line 361: `const mail = "request.backoffice@tvhequipment.com";`

---

## Git & Branch Conventions

- **Main branch**: `main`
- **Feature/task branches**: `claude/<short-description>-<id>` (e.g. `claude/add-claude-documentation-jXyBw`)
- All commit messages in this repo historically follow `"Update index.html"` — prefer more descriptive messages for new work.
- No CI/CD pipeline is configured.

---

## Important Constraints

- **No framework, no dependencies**: Do not introduce npm, React, Vue, or any external library. The entire app must remain a single self-contained HTML file.
- **No build step**: The file must be openable directly as `file://` in a browser.
- **Dutch UI text**: All user-visible strings must remain in Dutch.
- **No backend**: Orders are sent via `mailto:`. Do not add a server or API calls.
- **No test suite**: Manual browser testing is the only verification method.
- **Inline scripts only**: All JavaScript stays in the `<script>` block inside `index.html`.
- **DOM construction**: Prefer `createElement` + `textContent` over `innerHTML` with data. Never inject user-entered text (name, extra info) or external values via `innerHTML`.
- **mailto encoding**: Always use `encodeURIComponent()` for the subject and body of `mailto:` URLs. Do not manually percent-encode line breaks.
- **Prefix resolution in `voegToe`**: Always pass `item.catPrefix` from search results as the third argument. For normal navigation, omit the third argument so it falls back to `PREFIXES[navigatiePad[0]]`.
- **Single init call**: Initialisation (localStorage restore + `renderMenu()` + `updateWinkelwagenLayout()`) happens once inside `window.onload`. Do not add a standalone `renderMenu()` call at module level.
