# App Manual

App manuals for Claude Code agents to understand and manipulate Electron apps via Chrome DevTools Protocol (CDP).

Each manual documents how to launch, connect to, and automate a specific Electron application — including UI interaction patterns, keyboard shortcuts, gotchas, and proven workflows.

## Apps

| App | Manual | Description |
|-----|--------|-------------|
| Heptabase | [heptabase.md](./manuals/heptabase.md) | Knowledge base / whiteboard app — journal, cards, edges, drag-and-drop |
| Superhuman | [superhuman.md](./manuals/superhuman.md) | Email client — compose, draft, bulk drafting, keyboard shortcuts |
| Arc | [arc.md](./manuals/arc.md) | Chromium-based browser — CDP connection, agent-browser + MCP setup |

## General Pattern

All Electron apps follow the same connection pattern:

1. **Quit the app** if already running
2. **Relaunch with CDP flag:** `open -a "AppName" --args --remote-debugging-port=9222`
3. **Connect:** `agent-browser connect 9222` or use Playwright/puppeteer over CDP
4. **Interact:** snapshot, click, type, screenshot

## Tools

- [`agent-browser`](https://github.com/vercel-labs/agent-browser) — Rust CLI for browser/Electron automation
- [Playwright](https://playwright.dev/) — for complex interactions (drag-and-drop, trusted mouse events)
- [puppeteer-core](https://pptr.dev/) — alternative CDP client (used in Superhuman bulk drafter)
