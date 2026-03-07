 # `neighbors.html` Design / Code Walkthrough
 
 This document explains **all major features and code paths** in `neighbors.html` (page title: “Net Neighbors - Detective Board”). The goal is that future AI agents can read this file and confidently edit:
 
 - `neighbors.html` (the interactive neighbors board)
 - This design doc (to keep it in sync as features change)
 
 ## Purpose / user experience
 
 The page presents your “neighbors” (88x31 web buttons) on an interactive **detective corkboard**:
 
 - Visitors can **click buttons** to open neighbor sites.
 - The board is **pannable** by dragging.
 - **Red string connections** visualize relationships between neighbors.
 - **EZ READ MODE** provides an accessible list/grid view of all buttons.
 - A “Grab My Button!” panel provides copy/paste HTML so others can link back.
 
 ## Files + assets referenced
 
 - **Local asset**: `neighborbutton.gif`
   - Used as the image for your own (“kick”) hub button.
   - Also shown in the copy/paste panel.
 - **External assets**: Most neighbor buttons use absolute `https://...` image URLs.
 - **No build system**: CSS and JS are inlined in `neighbors.html`.
 
 ## Page structure (DOM map)
 
 Core board:
 
 - `div#boardViewport.board-viewport`
   - Fullscreen viewport. Captures drag gestures.
 - `div#corkBoard.cork-board`
   - The corkboard surface.
   - Contains:
     - `svg#stringSvg.string-svg` (string overlay)
     - Generated `.button-card` nodes (one per neighbor template)
 
 Floating UI:
 
 - `button#ezReadBtn.ez-read-btn` (opens EZ Read modal)
 - `div#ezModal.ez-modal-backdrop` (modal backdrop)
   - Contains `div#ezGrid.ez-grid` populated by JS
 - `div.copy-section` fixed panel
   - `<details>` + `<summary>` disclosure
   - `<textarea>` containing the HTML snippet; clicking it selects the text
 
 Data templates:
 
 - A series of `<template class="neighbor-button" ...>` blocks define the neighbor list and connection graph.
 
 ## Visual layering (important z-index / pointer behavior)
 
 - `.string-svg` is `pointer-events: none` so strings never block clicking.
 - Buttons sit above strings:
   - Strings: `z-index: 1`
   - Button cards: `z-index: 2`
   - Pins: `z-index: 3`
 - Floating UI elements are above the board:
   - Copy section + EZ button: `z-index: 200`
   - Modal: `z-index: 300`
 
 ## Neighbor “data model”: templates drive everything
 
 Each neighbor is defined by a `<template class="neighbor-button">` with:
 
 - HTML `data-*` attributes (authoring metadata)
 - An embedded `<script type="application/json">` block (rendering metadata)
 
 ### Template attributes (authoring layer)
 
 - `data-id` (**required**)
   - Must be unique.
   - Used as the key in the JS map `buttonElements[id]`.
   - Used for string connections in `data-connects-to`.
 - `data-connects-to` (optional)
   - Comma-separated list of other button IDs.
   - Example: `data-connects-to="kick,onio"`
 - `data-x`, `data-y` (optional)
   - Percentage coordinates on the board.
   - If omitted, the JS assigns a position using the graph-based layout.
 - `data-rotation` (optional)
   - Rotation angle in degrees.
   - Used only when explicit `data-x` and `data-y` are provided.
 - `data-is-hub` (optional)
   - Present on the “kick” template.
   - Used by the layout system when `layout=graph` (default) to treat the node as a “hub” that other buttons can orbit around.
 
 ### JSON fields (rendering layer)
 
 The JSON contains:
 
 - `href`: link destination
 - `imgSrc`: image URL/path
 - `alt`: image alt text
 - `title`: tooltip text (used for both board buttons and EZ grid items)
 
 Example pattern:
 
 ```html
 <template class="neighbor-button" data-id="someId" data-connects-to="kick">
   <script type="application/json">
   {
     "href": "https://example.com",
     "imgSrc": "https://example.com/button.gif",
     "alt": "Example site button",
     "title": "Visit Example"
   }
   </script>
 </template>
 ```
 
 ## JavaScript behavior (feature-by-feature)
 
 All logic is contained in one immediately-invoked function expression (IIFE):
 
 - `(function() { ... })();`
 
 Key references:
 
 - `const corkBoard = document.getElementById('corkBoard');`
 - `const stringSvg = document.getElementById('stringSvg');`
 - `const templates = Array.from(document.querySelectorAll('template.neighbor-button'));`
 - `const buttonElements = {};` (map `id -> DOM element`)
 - `const knownTemplateIds = new Set();` (tracks unique `data-id` values)
 
 ### 1) Board sizing: always fills screen at any zoom level
 
 Function: `sizeBoardToViewport(currentZoom)`
 
 - Accepts the current zoom value so the unscaled board dimensions compensate for scale.
 - Formula (each axis):
   - `bw = Math.max((vw / z) * 1.5, 1200 / z, 1200)`
   - `bh = Math.max((vh / z) * 1.5, 900 / z, 900)`
 - This guarantees `bw * zoom >= vw * 1.5` — the scaled board always overflows the viewport by ~50%, giving pan room.
 - Called at init (with `z = 1`), on every `setZoom()` call, and on every `centerBoard()` (window resize).
 - Assigns computed dimensions to `corkBoard.style.width/height`.

### 1.5) Zoom controls

The page provides a fixed-position zoom slider in the top-left (`#zoomControls`):

- `#zoomSlider` — `<input type="range" min="0" max="200" value="100">` — controls corkboard zoom.
- `#zoomLabel` — displays the current zoom percentage (e.g. `100%`).

Slider mapping:

- Slider value `100` = `1.0` zoom (100%, default, natural size).
- Slider value `0` = `0.01` zoom (minimum, nearly invisible).
- Slider value `200` = `2.0` zoom (maximum, double size).
- Formula: `zoom = sliderValue / 100`.

Implementation notes:

- The board transform is applied as: `translate(-scrollLeft, -scrollTop) scale(zoom)`.
- Default zoom is `1.0` (slider at 100).
- Panning bounds are computed in *scaled* board units (`boardSize * zoom`).
- String geometry is computed in unscaled board coordinates by compensating for zoom during endpoint calculations.
- Zoom targets only the corkboard (`#corkBoard`), not the page or the viewport.
 
 ### 2) Automatic placement (graph layout) for templates without explicit coordinates
 
 If a template does *not* have both `data-x` and `data-y`, its position is computed by a **graph-based layout**.
 
 Layout selection is controlled by the `layout` query parameter:
 
 - Default: `neighbors.html?layout=graph`
   - `graph` is the default if no parameter is provided.
   - Uses a seeded PRNG (based on `data-id`) so placement is stable across reloads.
 - Optional: `neighbors.html?layout=random`
   - Places non-fixed nodes randomly (still within bounds), mainly for debugging/variety.
 - Also accepted (legacy “deterministic”) names:
   - `deterministic`, `stable`, `seeded`
   - These behave like `graph` with respect to random sources (seeded).
 
 Key invariants / overrides:
 
 - If **both** `data-x` and `data-y` are present, the node is treated as **fixed**.
 - If only one axis is provided (`data-x` or `data-y`), that axis is treated as a **lock** and the other axis is computed.
 - Fixed nodes also use `data-rotation` (or `0`). Non-fixed nodes get a seeded random rotation in `[-30, +30]`.
 
 Graph layout rules (high-level):
 
 - **Hubs** (`data-is-hub="true"`)
   - If there is exactly one hub and it is not fixed, it is placed at (50%, 50%).
   - If there are multiple hubs and they are not fixed, they are placed around a ring near center.
 - **Orbit around hubs**
   - Non-hub nodes connected to exactly one hub are placed in “orbit rings” around that hub.
 - **Between hubs**
   - Nodes connected to 2+ hubs are placed roughly between them (near the average of hub positions) with a small perpendicular jitter.
 - **Near non-hub neighbors**
   - Nodes that connect only to non-hubs (and no hubs) are placed near already-placed neighbors via a seeded BFS expansion.
 - **Unconnected nodes**
   - Nodes with no connections are placed via a simple packing scan within bounds.
 - **Relaxation pass**
   - After initial placement, multiple iterations of a lightweight relaxation step:
     - pushes overlapping nodes apart (using a spatial hash)
     - gently pulls connected nodes toward the average of their neighbors
 
 Constants used by layout:
 
 - `SPACING_MULT = 1.2` (global multiplier; increases spacing by ~20%)
 - `BUTTON_SPACING = 10 * SPACING_MULT` (minimum separation / collision radius in percentage space)
 - Bounds: `BOUNDS_MIN = 10`, `BOUNDS_MAX = 90`
 
 ### 3) Template parsing → generating `.button-card` DOM nodes
 
 For each template:
 
 1. Read:
    - `id = template.dataset.id`
    - `connectsTo = template.dataset.connectsTo.split(',')...`
 2. Determine x/y/rotation:
   - Explicit (both axes provided): parse `data-x`, `data-y`, and `data-rotation`
   - Partial explicit coordinates:
     - If only `data-x` is provided, X is locked and Y is graph-laid out
     - If only `data-y` is provided, Y is locked and X is graph-laid out
   - Else: graph-based placement (default) and seeded random rotation
 3. Validate template and parse JSON:
    - If `data-id` is missing: skip + `console.warn`
    - If `data-id` is duplicated: warn; only the first is treated as canonical
    - If JSON `<script type="application/json">` is missing: skip + warn
    - If JSON parse fails: skip + warn
    - If required JSON fields are missing (`href`, `imgSrc`): skip + warn
 4. Normalize connections:
   - Split by commas, `trim()`, remove empty strings
   - Deduplicate targets
   - Remove self-links (`id` connecting to `id`)
   - Filter out targets that do not exist as templates (unknown IDs are ignored)
 5. Create `div.button-card`:
    - Notes:
      - The cards are first built into a `DocumentFragment` and appended once, which scales better as the list grows.
 4. Create `div.button-card`:
    - absolutely positioned with `left/top` (percent)
    - centered via `translate(-50%, -50%)`
    - rotated via `rotate(...)`
    - contains:
      - `div.pin`
      - `<a href=... target="_blank" rel="noopener noreferrer">`
        - contains `div.button-frame` and the `<img>`
 5. Append to the corkboard and store in `buttonElements[id]`.
 
 ### 4) Drawing strings: SVG quadratic curves between pins
 
 Function: `drawStrings()`
 
 How it finds endpoints:
 
 - Uses `getBoundingClientRect()` for the board and for each `.pin`.
 - Converts pin coordinates to be **relative to the board**:
   - `x = pinRect.left - boardRect.left + pinRect.width / 2`
   - same for `y`
 
 How it draws:
 
 - For each connection, creates an SVG `<path>` with a quadratic curve:
   - `M x1 y1 Q midX midY x2 y2`
 - Adds a “sag” (droop) so strings feel like hanging thread:
   - `sag = Math.min(dist * 0.1, 30)`
   - midpoint y is increased by `sag`
 
 When it draws:
 
 - Shortly after load: `setTimeout(drawStrings, 100)`
   - This delay helps ensure layout is stable.
 - On resize: `window.addEventListener('resize', drawStrings)`
 - After panning ends: `endDrag()` calls `drawStrings()`
 - After `centerBoard()`: calls `drawStrings()`
 
 ### 4.5) Observed highlighting (hover → blue pins / strings)

The board supports an “observed” state:

- Hovering a **button card** highlights:
  - The hovered pin (blue)
  - Pins in **direct connection** (blue)
  - The **path back to `kick`** (if one exists): pins + strings along the shortest BFS path (blue)
- Hovering a **string** highlights:
  - The hovered string (blue)
  - The two endpoint pins (blue)

Implementation notes:

- CSS:
  - `:root --observed-blue`
  - `.pin.is-observed`
  - `.string-line.is-observed`
- String hover is enabled via:
  - `.string-svg { pointer-events: auto; }`
  - `.string-line { pointer-events: stroke; }`
- The JS builds:
  - `adjacency` (undirected) via `buildAdjacency(nodesById)` for BFS/pathfinding
  - `edgeElements` map (`edgeKey -> <path>`) created inside `drawStrings()`
  - `pinElements` map (`id -> .pin`) created when cards are generated

 ### 5) Drag-to-pan: mouse and touch (viewport pan)
 
 Panning is implemented by updating a virtual “scroll” offset and applying a transform:
 
 - `corkBoard.style.transform = translate(${-scrollLeft}px, ${-scrollTop}px)`
 
 Core functions:
 
 - `centerBoard()`
   - Resizes board, computes centered offsets, applies transform, redraws strings.
 - `startDrag(e)`
  - Does not start if the pointer is on interactive elements (`a`, `button`, `textarea`, `details`, `.copy-section`, `.button-card`, etc.).
   - Sets `.is-dragging` class for cursor feedback.
 - `doDrag(e)`
   - `preventDefault()` during touch move
   - Updates offsets with bounds so you can’t pan beyond the board.
 - `endDrag()`
  - Stops dragging and triggers `drawStrings()`.

Event wiring:
 
 - Mouse:
   - `mousedown` on viewport
   - `mousemove` and `mouseup` on `window`
 - Touch:
   - `touchstart` on viewport (`passive: true`)
   - `touchmove` on viewport (`passive: false`)
   - `touchend` on viewport
 
 ### 5.5) Button card dragging

Each `.button-card` on the corkboard can be dragged by the user to reposition it.

**Orbiter behavior:** An orbiter is a card whose *only* connection target is the card being dragged (i.e., `data-connects-to` has exactly one entry, pointing at the dragged card's ID). When a card is dragged, all its orbiters move along with it, preserving their relative positions.

How it works:

- `onCardMouseDown(e)` — fires on `mousedown` on a `.button-card`. Blocked on the inner `<a>` (link clicks are not drags). Calls `e.stopPropagation()` to prevent the viewport pan from starting.
- `getOrbiters(draggedId)` — scans `buttonElements` to find all cards that connect solely to `draggedId`.
- `cardPointerToBoard(clientX, clientY)` — converts a client-space coordinate to board-percentage space, accounting for the current `zoom`.
- `onCardMouseMove(e)` — moves the dragged card and all orbiters by the board-percentage delta from drag start. Calls `drawStrings()` continuously to keep strings live.
- `onCardMouseUp(e)` — ends the drag. If the card actually moved, registers a one-shot `click` capture listener to swallow the link click that fires after `mouseup`.

Because `startDrag` (viewport pan) also checks `.button-card` in its exclusion selector, there is no conflict between card drag and board pan.

Event wiring:

- Each card: `mousedown` → `onCardMouseDown`
- `window`: `mousemove` → `onCardMouseMove`
- `window`: `mouseup` → `onCardMouseUp`

State variables:

- `cardDrag` — null when idle; object `{ draggedId, orbiters, startBoard, initialPositions, moved }` during a drag.

### 6) EZ READ MODE modal (categorized grid view)
 
 Goal: provide a simple, non-pannable list of all neighbors.
 
 - The modal header text is: `WELCOME TO EZ MODE PLAYA`.
 - The JS populates `#ezGrid` by iterating all `templates` and generating **category sections**.
 - Each category section contains:
   - An `h3.ez-category-title` title
   - A `div.ez-category-grid` containing the grid items
 - Open:
   - Clicking `#ezReadBtn` adds class `.is-open` to `#ezModal`.
 - Close:
   - Clicking `#ezModalClose` removes `.is-open`.
   - Clicking the backdrop closes only if `e.target === ezModal`.
   - Pressing `Escape` closes the modal.

Category authoring rules:

- Templates can specify `data-category` (comma-separated) to appear in one or more categories.
  - Example: `data-category="MY PEEPS, QOOL"`
- If `data-category` is missing or empty, the button is treated as `Net Neighbors`.
- Preferred category display order:
  - `MY PEEPS`
  - `Net Neighbors`
  - `QOOL`
- Any additional categories you add later will still render (after the preferred ones).
 
 ## Feature inventory (checklist)
 
 - **Interactive corkboard**
   - fullscreen
   - pannable (mouse + touch)
 - **Observed highlighting**
   - hover a button: hovered pin + directly connected pins become blue
   - path back to `kick` (pins + strings) becomes blue if it exists
   - hover a string: that string + endpoint pins become blue
 - **Zoom slider**
   - range 0–200 (100 = 100% / default)
   - targets only the corkboard, not the page
   - live label shows current percentage
 - **Draggable button cards**
   - any `.button-card` can be dragged to reposition
   - orbiters (cards connected only to the dragged card) move with it
   - drag does not trigger a link navigation
 - **Neighbor buttons**
   - generated from `<template>` definitions
   - open links in new tabs (`noopener noreferrer`)
 - **Connection strings**
   - SVG paths between pins based on `data-connects-to`
   - redraw on drag end, card drag move, and resize
 - **EZ READ MODE**
   - modal overlay
   - grid of all neighbors
 - **"Grab My Button!" widget**
   - `<details>` drop-down panel
   - click-to-select copy snippet
 
 ## How to add a new neighbor safely
 
 1. Copy an existing `<template class="neighbor-button">` block.
 2. Choose a new unique `data-id`.
 3. Set JSON values:
    - `href`, `imgSrc`, `alt`, `title`
 4. Decide positioning:
    - **Option A (explicit)**: set `data-x`, `data-y`, optional `data-rotation`
    - **Option B (automatic)**: omit `data-x` and `data-y` and let the graph layout handle it
 5. Add `data-connects-to` if you want strings.
 
 ## Known gotchas / current issues to watch for
 
 ### Duplicate `data-id` values
 
 `buttonElements` is a plain object map keyed by `data-id`.
 
 Standardization change: templates are now pre-scanned into `knownTemplateIds`.
 
 - If a duplicate `data-id` is detected, the script logs a warning and only one will be treated as canonical.
 - Even though the page no longer hard-crashes, you should still treat **duplicate IDs as an error** and fix them.
 
 ### Hub layout depends on at least one hub
 
 The layout supports `data-is-hub="true"`, but if you have **zero** hubs then the algorithm falls back to packing / neighbor-expansion behavior for the whole graph.
 
 ### Strings depend on layout timing
 
 Because string endpoints are computed using `getBoundingClientRect()`, string drawing must happen only after elements have rendered. That’s why there is a `setTimeout(drawStrings, 100)`.
 
 ## Safe refactor points (where to change behavior)
 
 - **Board size**: adjust factor/minimums in `sizeBoardToViewport()`
 - **Layout rules**: tweak `BUTTON_SPACING`, `BOUNDS_MIN`, `BOUNDS_MAX`, orbit constants, and the relaxation loop in `layoutGraph()`
 - **String look**: edit `.string-line` CSS
 - **String geometry**: edit sag calculation in `drawStrings()`
 - **Observed highlighting**: edit `updateObserved()` (logic) and `.pin.is-observed` / `.string-line.is-observed` (styling)
 - **Modal**: edit `.ez-modal`, `.ez-grid`, and the grid population loop
 - **Zoom range/default**: change `min`/`max`/`value` on `#zoomSlider` and the `sliderToZoom()` formula
 - **Orbiter definition**: change `getOrbiters()` to adjust which cards count as orbiters
