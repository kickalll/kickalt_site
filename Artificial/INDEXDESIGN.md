
# BONUSCONTEXT — `index.html` Developer Map (ASCII Aquarium)

This file is meant to let a future AI agent (or human) quickly understand how `index.html` works, what the major subsystems are, and where to edit things safely.

`index.html` is a single-file site with:

- **A full-screen ASCII "aquarium"** rendered as a grid of `.` characters.
- **Animated entities** (fish/vehicles/jellies/plants/rocks) defined by `<template class="entity">` blocks containing JSON.
- **Click interactions** (feed pellets, open draggable popups, open site map modal, chat popup).
- **A terminal easter-egg popup** that appears after a hidden action and accepts secret codes.

---

## 1) Mental model / "what runs what"

Everything interactive lives inside one IIFE (Immediately Invoked Function Expression) near the end of the file:

- The aquarium is a positioned container `.stage`.
- The background dot grid is a `<pre id="periods">` whose text is set to `rows` lines of `cols` dots.
- On window `load`:
  - The dot grid is measured and filled.
  - Entities are loaded from the `<template>` blocks.
  - Entities are rendered once.
  - A single world update loop starts via `startWorldLoop()`, plus `plants.forEach(startPlantAnimation)`.
- On stage click:
  - A pellet is spawned at the clicked location (art defined by `PELLET_ART` constant).
  - Pellets fall downward over time.
  - If a pellet overlaps a fish, fish points increase and the pellet disappears.

Separately:

- Clicking certain ASCII entities opens **draggable popups** (iframe windows).
- The **Site Map** modal (`#siteMapModal`) is a normal modal overlay.
- The **Chat** button opens a chat iframe popup.
- A **Terminal** popup is triggered by a hidden interaction and accepts secret codes via a data-driven lookup table.

---

## 2) Key DOM elements (IDs/classes)

### Speech tooltip + unlock toast

- **`#fishSpeechTooltip`**
  - `position: fixed`, `pointer-events: none`, hidden by default.
  - Shows a fish speech quote near the hovered fish (above, or below if fish is in top 10% of viewport).
- **`#unlockToast`**
  - `position: fixed`, bottom-centered, hidden by default.
  - Animates upward and fades out via `@keyframes unlockToastRise` when an unlock fires.
  - Triggered by `showUnlockToast(title, subtitle)`.



### Aquarium rendering

- **`.stage`**
  - Full-screen container that everything is rendered into.
- **`#periods`**
  - The background dot grid `<pre>`.
- **`.entity`**
  - Base class for entity `<pre>` layers.
  - `pointer-events: none` so entities don't block clicks unless wrapped.

### Clickable entities

- **`.entity-link`**
  - A wrapper `<a>` positioned in the stage.
- **`.entity-popup-window`**
  - Class on draggable popup windows.
  - Dragging is implemented by pointer events on `.modal-header`.

### UI controls (fixed overlay)

- **`#siteMapBtn`**
  - Button toggles site map modal.
- **`#chatBtn`**
  - Opens the chat popup.
- **`#fishPoints`**
  - Displays current fish points.
  - Clicking it toggles the Fish Game info modal.

### Modals / popups

- **`#siteMapModal`**
  - The backdrop for the site map modal.
- **`#siteMapClose`**
  - Close button in the site map modal.
- **`#siteMapChatBtn`**
  - Link inside site map that opens chat.
- **`#fishGameModal`**
  - Backdrop for the Fish Game info modal.
- **`#entityPopupsContainer`**
  - Container for draggable popups created in JS.

---

## 3) Entity templates system (how fish/plants/rocks are defined)

Entities are defined by blocks like:

```html
<template class="entity" data-type="fish" data-speed-ms="80">
  <script type="application/json">{ ... }</script>
</template>
```

### Supported `data-type` values

- **`fish`**
  - Moves randomly around the grid.
- **`vehicle`**
  - Same movement logic as fish (shared "swimmer" code).
- **`jelly`**
  - Same movement logic as fish/vehicle. Eats pellets like fish.
- **`plant`**
  - Stationary; toggles between `stateA` and `stateB` to "sway".
- **`rock`**
  - Stationary art.

### JSON fields (by type)

#### Fish / Vehicle / Jelly JSON

- **`artLeft`**: array of strings
- **`artRight`**: array of strings
- **Optional `linkUrl`**: if set, the entity becomes a clickable link opening in a new tab.
- **Optional `popupIframeUrl`**: if set, the entity becomes a button that opens a draggable iframe popup.
- **Optional `popupTitle`**: title shown in the popup header.
- **Optional `shadowColor` / `shadowHoverColor`**: CSS colors for `.entity-shadow`.

#### Plant JSON

- **`stateA` / `stateB`**: arrays of strings. Plant alternates between them.
- Plant positioning comes from template attributes:
  - **`data-x`**: column from the left
  - **`data-y-offset`**: rows up from bottom
- Plants can also be clickable if given `linkUrl` or `popupIframeUrl`.

#### Rock JSON

- **`art`**: array of strings
- Positioning:
  - `data-x`
  - `data-y-offset`

### Inline color markup inside ASCII art

The renderer supports a lightweight markup inside entity art lines:

- `[[PINK]]hello[[/]]` becomes `<span style="color: PINK">hello</span>`
- Supported tokens are CSS-ish color values:
  - Named colors like `PINK`
  - Hex like `#ff00aa`
  - `rgb(...)` / `rgba(...)`

Functions involved:

- `parseInlineColorMarkup(line)`
- `normalizeCssColorToken(token)`
- `artLinesToHtml(lines)`

Important: fish/vehicle use `innerHTML` to support this markup.

---

## 4) Tuning constants + core runtime state

### Named constants (top of IIFE)

All key tuning values are extracted into named constants for easy modification:

| Constant                | Default | Purpose                                          |
|-------------------------|---------|--------------------------------------------------|
| `SWIM_PROXIMITY_PAD`    | `2`     | Grid units of desired spacing between bboxes     |
| `INERTIA_WEIGHT`        | `3`     | How strongly swimmers prefer continuing direction|
| `TARGET_MIN_DISTANCE`   | `10`    | Minimum distance when picking a new target       |
| `PELLET_FALL_MS`        | `70`    | Milliseconds per pellet fall step                |
| `PELLET_WIDTH`          | `3`     | Grid columns a pellet occupies                   |
| `PELLET_ART`            | `[x]`  | Visual representation of a pellet                |
| `GRID_OVERSCAN`         | `4`     | Extra cols/rows rendered beyond viewport         |
| `SPEED_BOOST_MS`        | `1000`  | Duration of initial speed boost on page load     |
| `PLANT_SWAY_BASE_MS`    | `500`   | Minimum ms between plant sway toggles            |
| `PLANT_SWAY_RAND_MS`    | `700`   | Random extra ms added to sway interval           |
| `SWIMMER_STAGGER_MS`    | `500`   | Max random start delay for swimmer animations    |
| `TERMINAL_ZONE_X_FRAC`  | `0.1`   | Left fraction of screen for secret click zone    |
| `TERMINAL_ZONE_Y_FRAC`  | `0.9`   | Bottom fraction of screen for secret click zone  |
| `TERMINAL_CLICK_COUNT`  | `10`    | Clicks in zone needed to open terminal           |
| `POINTS_PER_PELLET`     | `1`     | Fish points awarded per pellet eaten              |

### Grid geometry

- `cols`, `rows`: how many character columns/rows fit the stage (rows includes `GRID_OVERSCAN`).
- `visibleRows`: rows without overscan — used for swimmer Y bounds so fish never swim off-screen vertically.
- `charWidth`, `charHeight`: measured character size (used to map grid coords to pixels).

### Entity collections

- `fishes`: moving fish entities
- `vehicles`: moving vehicle entities
- `jellies`: moving jelly entities (behave like fish, eat pellets)
- `plants`: swaying plants
- `rocks`: stationary rocks
- `pellets`: falling feed pellets

Each fish/vehicle/jelly swimmer also carries:
- `specialId` (string | null): from `data-special-id` template attribute
- `limitedSpeech` (boolean): from `data-limited-speech="true"` template attribute; restricts speech to quotes explicitly targeting this fish
- `entityType` (string): `'fish'`, `'vehicle'`, or `'jelly'`

### Points

- `fishPoints`: number
- `renderFishPoints()`: updates `#fishPoints` UI
- `addFishPoints(amount)`: increments `fishPoints`, calls `renderFishPoints()`, then `checkUnlocks(prev, next)`

### Unlocks

- `fishCanTalkUnlocked`: boolean — set `true` when 25 fishi points threshold is crossed
- `UNLOCK_THRESHOLDS`: array of `{ threshold, onUnlock }` — scalable registry; add an entry here to define new unlocks
- `checkUnlocks(prev, next)`: iterates thresholds and fires `onUnlock()` when a threshold is newly crossed
- All unlock state is **in-memory only** (resets on reload). See `persistenceplan.md` for the deferred persistence design.

### Fish speech quotes

- `fishQuotes`: in-memory array of `{ lines: string[], restrictedTo: string[] }` parsed from `fishquotes.csv`
- `loadFishQuotes()`: fetches `fishquotes.csv`, parses with `parseFishQuotesCsv()`, stores in `fishQuotes`; called during `window.load`
- `parseFishQuotesCsv(text)`: parses the CSV — each line is `pipe|separated|quote,restrictedId1;restrictedId2` (blank restrictedTo = any fish)
- `getEligibleQuotes(swimmer)`: returns quotes a given fish may say based on `limitedSpeech` and `specialId` rules
- `pickSpeechQuote(swimmer)`: picks a random eligible quote or `null`

### Speed boost

- `speedBoostUntilMs`: used by `speedFactor()` to temporarily speed up animations.
- `scaledDelay(ms)`: scales movement/animation delays based on speed boost.

---

## 5) Helper functions (DRY patterns)

### `createEntityOverlay(tpl, data)`

Shared helper that creates the correct overlay element for any entity type. Returns `{ el, shadowEl, mainEl }`.

Logic:
1. If `popupIframeUrl` exists → creates a popup overlay via `createPopupOverlay()`
2. Else if `linkUrl` exists → creates a linked overlay via `createLinkedOverlay()`
3. Else → creates a plain overlay via `createOverlay()`

This replaces 3× copy-pasted if/else blocks that were previously duplicated in fish, vehicle, and plant loading.

### `createDraggablePopup(extraClass, contentHtml, opts)`

Shared factory for all draggable popup windows. Returns `{ popup, state, bringToFront, closePopup }`.

What it handles:
- Creates positioned popup DOM with stacking offset
- Sets up drag-to-move via pointer events on `.modal-header`
- Manages `activePopups` array and z-ordering
- Close button wiring with optional `opts.beforeClose` and `opts.onClose` callbacks

Used by both `openEntityPopup()` and `openTerminalPopup()`, eliminating ~80 lines of duplicated drag/state boilerplate.

---

## 6) Rendering pipeline (ASCII grid → pixel positioning)

### Dot grid background

- `measureCharSize()`
  - Creates a hidden `span` with `.` to measure character width.
  - Uses computed `line-height` for height.
- `fillDots()`
  - Uses `.stage.getBoundingClientRect()` and measured char size.
  - Adds `GRID_OVERSCAN` extra rows/cols.
  - Computes `cols` and `rows`.
  - Sets `#periods.textContent` to a giant block of dots.

### Entity overlays

All entities render by setting:

- Their displayed text/HTML (ASCII art)
- Their CSS transform:
  - `translate(${x * charWidth}px, ${y * charHeight}px)`

Key functions:

- `renderSwimmer(swimmer)` — renders fish, vehicle, or jelly (uses `innerHTML` for color markup). Also manages ghost element for seamless horizontal wrapping.
- `renderPlant(plant)`
- `renderRock(rock)`
- `renderAllEntities()` — wraps swimmer X via `wrapX()`, clamps Y to `visibleRows`, and re-renders; uses `getSwimmers()` for unified fish+vehicle+jelly handling

Notes:

- For clickable entities (wrapped in `.entity-link`), width/height are explicitly set on the wrapper so clicks match the art block.

---

## 7) Movement + behavior

### "Swimmers" (fish + vehicles + jellies)

Fish, vehicles, and jellies share the same movement logic via a unified code path:

- `getSwimmers()` returns `fishes.concat(vehicles, jellies)`
- `moveSwimmer(swimmer)` moves one step using AABB candidate scoring
- `wrapX(x)` wraps X position via modular arithmetic for horizontal screen wrapping
- `loadEntities()` uses a single `if (type === 'fish' || type === 'vehicle' || type === 'jelly')` branch

How movement works:

- Each swimmer has:
  - Current position: `x`, `y`
  - Target position: `tx`, `ty`
  - Direction memory: `dirX`, `dirY` (for inertia)
  - Hover state: `isMouseOver` (freezes movement when true)
  - Timer: `nextMoveAtMs` (used by the world loop)
  - Ghost element: `ghostEl` (for seamless horizontal wrapping)
- When it reaches its target, it picks a new random target (minimum `TARGET_MIN_DISTANCE` away, wrap-aware).
- It evaluates 5 candidates: stay, left, right, up, down.
- Each candidate is scored by:
  - **Target distance** (wrap-aware on X)
  - **Inertia bonus** (`INERTIA_WEIGHT`) for continuing current direction
  - **Proximity penalty** for being within `SWIM_PROXIMITY_PAD` of another swimmer's bbox
  - Small random noise for natural variation
- Candidates that cause **AABB overlap** with another swimmer are **rejected** (hard block).
- If no valid candidate exists, swimmer stays put.

#### Escape mode (stuck swimmers)

If a swimmer is **currently overlapping** another (e.g. from spawn), it enters escape mode:
- Overlapping candidates are not hard-blocked; instead they receive a +200 penalty.
- This allows the swimmer to actively move away from the overlap instead of freezing.

#### Horizontal wrapping

- Swimmers wrap around the screen horizontally via `wrapX()`: `((x % cols) + cols) % cols`.
- X is never clamped — it wraps seamlessly.
- Collision detection uses **wrap-aware relative distance** (normalized to `[-cols/2, cols/2]`).
- Target distance also uses the shorter wrapping path.
- Each swimmer has a `ghostEl` — when `x + width > cols`, the ghost renders at `x - cols` to show the portion entering from the left. Hidden when fully on-screen.

#### Vertical bounds

- Y is clamped to `[0, visibleRows - height]` so swimmers never go off-screen vertically.
- `visibleRows` excludes `GRID_OVERSCAN`.

#### Mouse hover freeze

- `stage` has `mousemove` and `mouseleave` listeners.
- Mouse position is converted to pixel coords and checked against each swimmer's bbox (including the ghost position for wrapping fish).
- `swimmer.isMouseOver` is set accordingly; the world loop skips movement for hovered swimmers.

#### Fish speech tooltip

- After the `isMouseOver` loop in `mousemove`, `updateFishSpeechTooltip(hoveredSwimmer, bounds)` is called.
- `hoveredFishForSpeech` tracks which fish was last hovered; a new quote is picked only on hover-enter (not every frame).
- On `mouseleave`, the tooltip is hidden and speech state is reset.
- Quote positioning: above the fish by default; below if the fish's top edge is in the top 10% of the viewport.

Direction:

- `swimmer.goingRight = bestCand.dx > 0`
  - Controls whether `artRight` or `artLeft` is rendered.

#### Spawn-time non-overlap

- `loadEntities()` attempts up to 60 random placements per swimmer.
- Accepts only positions whose AABB doesn't overlap already-placed swimmers.
- Falls back to random position if all attempts fail (escape mode handles the rest).

### World update loop

- `startWorldLoop()` replaces per-swimmer timers with a single `requestAnimationFrame` loop.
- Each swimmer has `nextMoveAtMs`; the loop advances any swimmer whose timer is due and not hovered.
- Initial start is staggered by `Math.random() * SWIMMER_STAGGER_MS`.

### Plant "sway" animation

- `startPlantAnimation(plant)` toggles `plant.currentState` and re-renders.
- Timer: `PLANT_SWAY_BASE_MS + Math.random() * PLANT_SWAY_RAND_MS` between toggles.

---

## 8) Pellets (click-to-feed) + collision

### Spawning

- Stage click calls `spawnPelletAtClientPoint(e.clientX, e.clientY)` unless the click was on:
  - `.entity-link` (clickable fish/buttons)
  - `.entity-popup-window` (popup window)

`spawnPelletAtClientPoint()`:

- Converts client pixels to local stage coords.
- Converts pixels to grid coords via `charWidth/charHeight`.
- Creates a pellet object using constants:
  - `width: PELLET_WIDTH`, `height: 1`
  - `fallMs: PELLET_FALL_MS`
  - `el: createOverlay()` (a `<pre>`)
- Pushes into `pellets` and starts animation.

### Falling

- `startPelletAnimation(pellet)` increments `pellet.y` until it hits bottom.
- On each tick:
  - Removes if it hits bottom.
  - Removes (and increments points) if it overlaps a fish.
- Pellet visual: `PELLET_ART` constant (default `[x]`).

### Collision

- `pelletTouchesFish(pellet)` performs an axis-aligned overlap check against each fish and jelly.

Points:

- `removePelletEatenByFish(pellet)` increments `fishPoints` by `POINTS_PER_PELLET` and updates the UI.

---

## 9) Popups (draggable windows)

All draggable popups are built via `createDraggablePopup(extraClass, contentHtml, opts)`:

- Creates a positioned `.modal.entity-popup-window` DOM node
- Adds to `#entityPopupsContainer`
- Sets up drag-to-move via pointer events on `.modal-header`
- Manages `activePopups` array and z-ordering via `bringToFront()`
- Close button calls `opts.beforeClose()` then removes DOM then calls `opts.onClose()`

### A) Generic iframe popup windows (`openEntityPopup`)

- Opened via:
  - Clickable entities with `popupIframeUrl`
  - The chat button (`openChatPopup()`)
  - Terminal secret codes

Function:

- `openEntityPopup(iframeUrl, title, onClose)`
- Delegates to `createDraggablePopup('modal-iframe', ..., { beforeClose, onClose })`

Close behavior:

- `beforeClose`: clears the iframe to `about:blank`
- `onClose`: optional callback (e.g. chat uses it to reset `chatOpened`)

### B) Terminal popup window (`openTerminalPopup`)

- Delegates to `createDraggablePopup('', ...)` with terminal-specific HTML
- Not an iframe; contains:
  - Output area `.terminal-output`
  - Text input `.terminal-input`
  - "enter" submit button
- Wires up submit/enter event listeners after popup creation

---

## 10) Site map modal (non-draggable)

Elements:

- `#siteMapBtn`: toggles open/close
- `#siteMapModal`: backdrop
- `#siteMapClose`: close button in the site map modal.
- `#siteMapChatBtn`: link inside site map that opens chat.

Functions:

- `openSiteMap()` sets `.is-open` and aria attributes
- `closeSiteMap()` removes `.is-open` and sets aria-hidden

Extra close behavior:

- Clicking the backdrop closes the modal.
- Pressing `Escape` closes the site map (only).

---

## 11) Chat button behavior

State:

- `chatOpened` boolean prevents multiple chat popups.

Function:

- `openChatPopup()`
  - Opens the chat iframe popup via `openEntityPopup(chatUrl, 'chat', onClose)`
  - The `onClose` callback resets `chatOpened = false` so the chat can be reopened.

Auto-open:

- Auto-open timers were removed; chat only opens on user action.

---

## 12) Terminal easter egg + secret codes

### How it triggers

- Every stage click spawns a pellet.
- If the click occurs in a **bottom-left zone** (defined by constants):
  - `e.clientX <= window.innerWidth * TERMINAL_ZONE_X_FRAC`
  - `e.clientY >= window.innerHeight * TERMINAL_ZONE_Y_FRAC`
- A counter (`bottomLeftPelletCount`) increments.
- When it reaches `>= TERMINAL_CLICK_COUNT` and `terminalOpened` is false:
  - `openTerminalPopup()` is called.

### Codes (data-driven lookup table)

Terminal codes are defined in a `TERMINAL_CODES` object at the top of the terminal section. Each entry has an `action` and type-specific fields:

```js
const TERMINAL_CODES = {
    'onionclub':  { action: 'popup', url: onionClubUrl, title: 'onion cafe' },
    'oniondark':  { action: 'popup', url: onionDarkUrl, title: 'onion dark' },
    'fishi time': { action: 'points', amount: 1000000 },
    'breadlover': { action: 'popup', url: breadLoveURL, title: 'bread love' },
    'breadhater': { action: 'popup', url: breadHateURL, title: 'bread hate' },
};
```

To add a new terminal code, simply add an entry to this table. Supported actions:

- `'popup'`: opens an iframe popup with `url` and `title`
- `'points'`: adds `amount` to fish points

The `handleSubmit()` function looks up the code in the table and dispatches by action type. No if/else chain needed.

---

## 13) Resizing / viewport

- `queueFill()` recalculates dot grid and re-renders entities.
- Bound to:
  - `window.resize`
  - `visualViewport.resize` and `visualViewport.scroll` (mobile friendliness)

Important: entity `x/y` are in **grid coordinates**, so resizing changes `cols/rows` and clamps entities back into bounds.

---

## 14) Common edits (recipes)

### Add a new unlock threshold

Add an entry to `UNLOCK_THRESHOLDS` near the top of the IIFE:

```js
{
    threshold: 100,
    onUnlock() {
        // set a flag, show a toast, enable a feature...
        showUnlockToast('Something New?', 'UNLOCKED!!!');
    }
}
```

### Add fish quotes

Edit `fishquotes.csv`. Format: `line1|line2,restrictedId1;restrictedId2`

- Pipe `|` separates display lines (shown multi-line in the tooltip).
- Everything after the first comma is the restricted-to list (semicolon-separated special IDs, or blank for any fish).
- Lines beginning with `#` are comments.

### Mark a fish as special (restrict certain quotes to it)

1. Add `data-special-id="myfish"` to the fish `<template>` tag.
2. Add quotes to `fishquotes.csv` with `myfish` in the restrictedTo field.
3. Optionally add `data-limited-speech="true"` to prevent the fish from speaking unrestricted quotes.



### Add a new swimming fish or jelly

1. Copy an existing fish `<template class="entity" data-type="fish" ...>` (or use `data-type="jelly"` for a jelly).
2. Change `data-speed-ms` (lower = faster).
3. Replace `artLeft` / `artRight` arrays.
4. Optional:
   - Add `linkUrl` to open a URL in a new tab.
   - Add `popupIframeUrl` + `popupTitle` to open an iframe popup.
   - Add `shadowColor` / `shadowHoverColor` to style hover glow.

Jellies behave identically to fish (same movement, eat pellets) but are stored in the `jellies` array.

### Add a new popup "button" entity

Use `popupIframeUrl` in the JSON. The `createEntityOverlay()` helper automatically detects this and wires up the popup click handler.

### Add a new terminal secret code

Add an entry to the `TERMINAL_CODES` object:

```js
'mycode': { action: 'popup', url: 'https://example.com', title: 'my popup' },
```

Or for a points cheat:

```js
'mycode': { action: 'points', amount: 999 },
```

### Make fish avoid each other more (more spacing)

- Increase `SWIM_PROXIMITY_PAD` constant (default `2`) for wider spacing between bboxes.
- Increase `INERTIA_WEIGHT` constant (default `3`) for smoother, more directional movement.

### Change pellet behavior

- Pellet fallspeed: `PELLET_FALL_MS` constant (default `70`).
- Pellet art: `PELLET_ART` constant (default `[x]`).
- Pellet size for collision: `PELLET_WIDTH` constant (default `3`).

### Change point scoring

- Edit `POINTS_PER_PELLET` constant (default `1`).

### Change popup size

- `.modal-iframe-frame` CSS controls iframe height (currently `height: 450px`).
- `.modal` / `.modal-iframe` CSS controls width.

### Change plant sway speed

- Edit `PLANT_SWAY_BASE_MS` and `PLANT_SWAY_RAND_MS` constants.

### Change the terminal secret zone

- Edit `TERMINAL_ZONE_X_FRAC`, `TERMINAL_ZONE_Y_FRAC`, and `TERMINAL_CLICK_COUNT` constants.

---

## 15) Gotchas / invariants (don't break these)

- **Character measurement matters**: `charWidth/charHeight` must match the font sizing; avoid changing `pre` styles without checking alignment.
- **Entities use `innerHTML`** (not `textContent`) for fish/vehicle to support `[[COLOR]]...[[/]]` markup. If you switch to `textContent`, colored markup will stop working.
- **Stage click routing**: stage click handler intentionally ignores clicks inside `.entity-link` and `.entity-popup-window`.
- **Chat repeatability**: chat relies on the `openEntityPopup(..., onClose)` callback to reset `chatOpened`.
- **Popup close lifecycle**: `createDraggablePopup` calls `opts.beforeClose()` before removing the DOM, then `opts.onClose()` after. Entity popups use `beforeClose` to blank the iframe.
- **Function hoisting**: `createDraggablePopup` is declared after `openTerminalPopup` in the source, but since both are `function` declarations inside the IIFE, JS hoisting makes this work. Don't convert them to `const`/arrow functions without reordering.

---

## 16) Architecture summary (what was DRYed up)

The codebase was refactored for scalability and human modification:

1. **Named constants**: All magic numbers extracted to labeled constants at the top of the IIFE.
2. **`createEntityOverlay(tpl, data)`**: Eliminated 3× copy-pasted overlay creation (fish, vehicle, plant).
3. **Merged fish/vehicle/jelly loading**: `loadEntities()` uses a single branch for all swimmer types.
4. **`renderSwimmer()`**: Renamed from `renderFish()` since it renders fish, vehicles, and jellies. Also manages ghost elements for seamless wrapping.
5. **`startWorldLoop()`**: Replaced per-swimmer `startSwimmerAnimation()` with a single `requestAnimationFrame` world loop.
6. **AABB candidate scoring**: Replaced center-distance repulsion with strict bounding-box non-overlap and 5-candidate scoring (stay/left/right/up/down).
7. **Horizontal wrapping**: `wrapX()` helper + ghost elements for seamless edge-to-edge screen wrapping.
8. **Mouse hover freeze**: `mousemove`/`mouseleave` listeners set `swimmer.isMouseOver` to pause movement.
9. **`createDraggablePopup()`**: Extracted ~80 lines of shared popup+drag boilerplate used by both popup types.
10. **`TERMINAL_CODES` table**: Replaced if/else chain with data-driven lookup. Adding codes is now a one-liner.
11. **`renderAllEntities()`**: Uses `getSwimmers()` instead of separate fish/vehicle loops.

---

## 17) Where to look in `index.html` (navigation hints)

- **CSS**: at top inside `<style>`
- **UI buttons + modals**: near the top of `<body>` (IDs: `siteMapBtn`, `chatBtn`, `siteMapModal`, `entityPopupsContainer`)
- **Entity templates**: the big section of `<template class="entity">` blocks
- **Main JS IIFE**: near the end of the file starting with:
  - `const stage = document.querySelector('.stage');`
  - `const dotsPre = document.getElementById('periods');`
- **Tuning constants**: immediately after the unlock infrastructure block in the IIFE
- **Unlock infrastructure**: `UNLOCK_THRESHOLDS`, `checkUnlocks`, `addFishPoints` — immediately after `let fishPoints = 0;`
- **Fish speech system**: `parseFishQuotesCsv`, `loadFishQuotes`, `getEligibleQuotes`, `pickSpeechQuote`, `updateFishSpeechTooltip` — between `renderFishPoints` and `loadEntities`
- **Helper functions**: `createEntityOverlay` after `createPopupOverlay`; `createDraggablePopup` after popup infrastructure vars
- **Terminal codes table**: `TERMINAL_CODES` object, right before `openTerminalPopup()`
