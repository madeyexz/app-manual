# Heptabase Automation via Electron CDP

## Connection Setup

1. **Quit Heptabase first** if it's already running — the CDP flag must be present at launch time
2. Relaunch with remote debugging:
   ```bash
   open -a "Heptabase" --args --remote-debugging-port=9222
   ```
3. Wait ~4 seconds for startup, then connect:
   ```bash
   agent-browser connect 9222
   ```

## Interacting with the UI

### Sidebar Navigation
- Sidebar items (Journal, Whiteboard, Card Library, etc.) appear as `button` and `link` refs in `agent-browser snapshot -i`
- Click the `link` ref to navigate (e.g., `agent-browser click @e4` for Journal)

### Journal
- The journal content area is a `[contenteditable]` div, **not** a standard textbox
- It does **not** appear in `agent-browser snapshot` as an interactive element
- To type in it:
  ```bash
  agent-browser eval 'document.querySelector("[contenteditable]")?.focus()'
  agent-browser press h
  agent-browser press i
  ```
- Pasting images works: copy PNG to macOS clipboard with `osascript`, then `agent-browser press Meta+v`

### Whiteboard Cards
- Cards are `div._whiteboardObject_6ni1o_9` elements
- Get positions via `getBoundingClientRect()` in eval
- Card content lives inside `div.ProseMirror` (contenteditable) within the card
- The whiteboard canvas is `div._whiteboardCanvas_1ozeb_1`

## Creating Edges (Connections) Between Cards

**The working method: Right-click -> "Draw connection" -> Click target card**

1. Right-click on the source card (Playwright `page.mouse.click(x, y, { button: 'right' })`)
2. Find the "Draw connection" menu item — it uses class `.hepta-menu-item`, **not** `[role="menuitem"]`
3. Click it via JS: `document.querySelectorAll('.hepta-menu-item')` -> find by innerText
4. Click on the destination card to complete the connection
5. Press `Escape` to exit connection mode

**What does NOT work for edges:**
- Dispatching JS MouseEvent/PointerEvent — Heptabase ignores `isTrusted: false` events
- `agent-browser hover --point` on card borders — times out with no actionable element
- Connection handles don't exist in DOM until hovered via real (trusted) mouse events

## Moving Cards (Drag & Drop) — DANGEROUS

Use Playwright's native mouse API (via a `.mjs` script connecting over CDP):

```js
await page.mouse.move(card.cx, card.cy);
await page.mouse.down();
// Move in ~20 steps for smooth drag
for (let i = 1; i <= 20; i++) {
  await page.mouse.move(card.cx + deltaX * (i/20), card.cy + deltaY * (i/20));
  await page.waitForTimeout(20);
}
await page.mouse.up();
```

### CRITICAL: Card Merging Danger

**Dragging a card OVER another card MERGES them** — the dragged card becomes an embedded block inside the target card. This is extremely easy to trigger accidentally and hard to undo.

- The merge happens when the drag path crosses over any other card, even briefly
- Multiple cards can get chain-merged into one card if the drag path crosses several
- Undo (`Cmd+Z`) does NOT reliably revert merges — it may only partially undo or trigger "Deleting card" confirmation dialogs

### Best Practice: Create Cards at Final Positions

**NEVER move cards by dragging if other cards are nearby.** Instead:
- Calculate the desired layout positions upfront
- Create each card directly at its target position via double-click at that coordinate
- This eliminates all merge risk

### Unmerging Embedded Cards

If cards accidentally merge, here's how to extract an embedded card:

1. **Click the parent card** (left-click to focus/enter it)
2. **Click on the embedded card** within it (left-click to focus on the embedded block)
3. **Hover over the embedded block** — a drag handle appears (class `_dragHandle_1xagl_1`)
4. A chip/label with the embedded card's title also appears on hover
5. **Drag the handle OUT** to empty whiteboard space (far from any other card)
6. The embedded card becomes a standalone card again

```js
// Find drag handle after focusing the embedded card
const dragHandle = await page.evaluate(() => {
  const el = document.querySelector('[class*="dragHandle"]');
  if (el) {
    const rect = el.getBoundingClientRect();
    return { x: Math.round(rect.x + rect.width/2), y: Math.round(rect.y + rect.height/2) };
  }
  return null;
});
// Drag it out to empty space
await page.mouse.move(dragHandle.x, dragHandle.y);
await page.mouse.down();
// Drag in steps AWAY from all cards
for (let i = 1; i <= 20; i++) {
  await page.mouse.move(dragHandle.x + (emptyX - dragHandle.x) * (i/20),
                         dragHandle.y + (emptyY - dragHandle.y) * (i/20));
  await page.waitForTimeout(30);
}
await page.mouse.up();
```

## Context Menu Structure

Right-click on a whiteboard card reveals these options:
- New chat, AI action, Default size, Fit to content
- Fold (`Cmd+Option+Enter`)
- **Draw connection** — this is the key one for edges
- Mindmap, Open in, Copy link, Add to Inbox, Manage tags, Move to, More, Copy, Remove, Delete

Menu items use class `hepta-menu-item` with text inside `p.truncate`.

## Running Playwright Scripts

Connect to Heptabase:
```js
import { chromium } from 'playwright';
const browser = await chromium.connectOverCDP('http://localhost:9222');
const page = browser.contexts()[0].pages()[0];
```

## Creating New Cards on Whiteboard

**Double-click on empty canvas** to create a new card. Workflow:

1. Double-click empty space -> card editor opens with "Heading 1" placeholder
2. Type the **title** (goes into the heading field)
3. Press **Enter** -> enters the card's **content editing space**
4. Use `page.keyboard.insertText(body)` to insert body text (more reliable than clipboard)
5. Press **Escape** to exit editing
6. Click empty space to deselect

**Important:** If you skip step 3 (Enter), the body text goes *outside* the card as a separate text element, not inside it.

## Selecting and Deleting Cards

- **Rubber-band select:** Click and drag on empty canvas to select multiple cards
- **Delete:** Press `Delete` after selecting
- `Cmd+A` enters text editing mode, NOT whiteboard selection — use drag-select instead

## Whiteboard Zoom

- `Cmd+scroll` (Meta + mouse.wheel) controls whiteboard canvas zoom
- `Cmd+-`/`Cmd+=` controls browser-level zoom, NOT the whiteboard zoom
- Zoom affects all screen coordinates — re-fetch positions after zooming

## Key Gotchas

- `agent-browser eval` is the command for running JS (not `evaluate`)
- Smart quotes in JS strings cause `SyntaxError: Invalid or unexpected token` — use straight quotes or write to a file and eval with `$(cat file.js)`
- `agent-browser snapshot -i -C` includes cursor-interactive elements (onclick divs)
- Whiteboard zoom level affects screen coordinates — always re-fetch `getBoundingClientRect()` before interacting
- After any navigation or state change, re-snapshot to get updated element refs
- Cards on a whiteboard can be very tall (e.g. 16000px) — a card that looks "expanded" may just be a large card at high zoom

### Smart Apostrophes Break Title Matching

Heptabase auto-converts straight apostrophes (`'`) to smart/curly apostrophes (`'`) when typing via `page.keyboard.type()`. This means:
- Code has `"Hemingway's"` but DOM has `"Hemingway\u2019s"`
- **Exact string comparison will fail** for any title containing apostrophes
- Fix: use fuzzy matching with `.includes(keyword)` where keyword is the part before the apostrophe

### Focus Before Focused Operations

**You MUST left-click a card first to focus/enter it before doing operations within it.** This applies to:
- Clicking embedded cards inside a parent card
- Accessing internal elements (drag handles, block menus)
- Right-clicking for context menus on specific content within a card
- Without focusing first, clicks may land on the whiteboard canvas layer, not the card layer

### Edge Creation Reliability

- **Single-click the source card first** to select it, THEN right-click for "Draw connection"
- If the right-click lands on an existing edge line instead of the card body, a different context menu appears (no "Draw connection" option)
- Click on the card's interior area, away from edges and card borders
- After creating each edge, **press Escape** to exit connection mode before creating the next edge
- Wait 800ms after clicking "Draw connection" before clicking the target card

### Text Input Methods

- `page.keyboard.type(text, { delay: 8 })` — for titles and short text; triggers Heptabase's text processing (smart quotes)
- `page.keyboard.insertText(text)` — for body content; more reliable than clipboard, inserts text as-is
- Clipboard-based approaches fail silently in Electron's sandbox

### Undo (`Cmd+Z`) Behavior

- Undo works for simple operations but is **unreliable for complex multi-step operations** like merges
- When undoing past card creation, Heptabase shows a **"Deleting card" confirmation dialog** — must click "Cancel" or "Continue"
- For clean rebuilds, **delete everything and recreate** rather than relying on undo

### Selecting Cards on the Whiteboard

- **Rubber-band select**: Click and drag on empty canvas space to select multiple cards
- **`Cmd+A` does NOT work** — it enters text editing mode, not whiteboard-level select-all
- For deleting many cards: rubber-band select -> `Delete` key -> confirm dialog if prompted
- May need multiple passes — some cards resist selection if overlapped or at canvas edges
- Zoom out first (`Meta + mouse.wheel(0, 200)`) to see and select all cards

### Canvas Zoom

- `Meta + mouse.wheel(0, 200)` = zoom OUT; `Meta + mouse.wheel(0, -200)` = zoom IN
- `Cmd+-`/`Cmd+=` controls BROWSER-level zoom, NOT the whiteboard canvas zoom — never use these
- Read current zoom from the zoom button: `document.querySelectorAll('button')` -> find one with `innerText.includes('%')`
- 35% zoom is good for seeing a full tree layout of ~12 cards on a 1280px screen
- After zooming, ALL screen coordinates change — must re-fetch `getBoundingClientRect()` for every card

## Complete Whiteboard Build Workflow (Proven Pattern)

For building a structured whiteboard with cards and edges from scratch:

1. **Design layout** — calculate (x, y) positions for each card in a tree/graph that avoids edge crossings
2. **Zoom to working level** (~35%) using `Meta + mouse.wheel`
3. **Create cards at final positions** — for each card:
   - Click empty space to deselect
   - Double-click at target (x, y)
   - Type title with `page.keyboard.type(title, { delay: 8 })`
   - Press Enter to enter content space
   - Insert body with `page.keyboard.insertText(body)`
   - Press Escape to exit editing
4. **Fetch all card positions** from DOM via `getBoundingClientRect()` on `[class*="whiteboardObject"]` elements
5. **Create edges** — for each edge:
   - Single-click source card center
   - Wait 200ms
   - Right-click source card center
   - Wait 800ms
   - Find and click "Draw connection" in `.hepta-menu-item` elements
   - Wait 800ms
   - Click destination card center
   - Wait 800ms
   - Press Escape
   - Wait 400ms
6. **Take screenshot** to verify
