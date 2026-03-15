# Superhuman Electron App Automation

## Launching with CDP

```bash
open -a Superhuman --args --remote-debugging-port=9222
```

Verify with:
```bash
curl -s http://localhost:9222/json
```

## Finding the Right CDP Page

Multiple pages are exposed. The main inbox page is the one matching:
- URL contains `mail.superhuman.com`
- URL does NOT contain `backend` or `serviceworker`
- Other pages include `superhuman-app://production/browserWindow.html`, `tabs.html`, and a `background_page.html` — skip these.

## Compose / Draft Workflow

1. **Press `C`** — opens New Message window, cursor lands in the "To" field
2. **Type the email address** — Superhuman auto-resolves contacts in the sidebar
3. **Press `Tab`** — moves cursor to the Subject field
4. **Type the subject**
5. **Press `Tab`** — moves cursor to the body/content field
6. **Type the body** (newlines via Enter key)
7. **Press `Escape` x3** — closes compose and saves as draft

## Key Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `C` | Compose new message |
| `E` | Mark Done |
| `H` | Set a reminder |
| `/` | Search |
| `Cmd+K` | Superhuman Command palette |
| `Tab` | Move between compose fields (To -> Subject -> Body) |
| `Escape` | Close/back out of compose (3x to fully save draft and return to inbox) |

## Automated Bulk Drafting

### Script Pattern (puppeteer-core)

```js
import puppeteer from 'puppeteer-core';

const browser = await puppeteer.connect({
  browserURL: 'http://localhost:9222'
});

// Find the main Superhuman page
const pages = await browser.pages();
const page = pages.find(p =>
  p.url().includes('mail.superhuman.com') &&
  !p.url().includes('backend') &&
  !p.url().includes('serviceworker')
);

// Compose a draft
await page.keyboard.press('c');           // Open compose
await page.waitForTimeout(600);
await page.keyboard.type(email);          // To field
await page.keyboard.press('Tab');         // -> Subject
await page.waitForTimeout(600);
await page.keyboard.type(subject);        // Subject field
await page.keyboard.press('Tab');         // -> Body
await page.waitForTimeout(600);
await page.keyboard.type(body, { delay: 8 });  // Body
await page.keyboard.press('Escape');      // Close compose (x3)
await page.waitForTimeout(600);
await page.keyboard.press('Escape');
await page.waitForTimeout(600);
await page.keyboard.press('Escape');
```

### Timing

- `STEP_DELAY = 600ms` between workflow steps
- `TYPE_DELAY = 8ms` between keystrokes
- Each draft takes ~15-20 seconds depending on body length

### CSV Input Format

Columns: `Name, Email Address, Confidence, Subject Line, Email Body`

The CSV parser must handle quoted fields with embedded newlines (email bodies span multiple lines).

## Notes

- Superhuman may show different account pages (e.g., multiple email accounts) — the page finder handles this by matching `mail.superhuman.com` broadly
- The `Escape` x3 pattern is important: first Escape closes the body editor, second closes the compose panel, third returns to inbox and saves the draft
