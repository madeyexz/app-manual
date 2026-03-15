# Arc Browser Automation via CDP

## Setup

Arc is a Chromium-based browser that supports CDP natively.

### Launch with Debug Port

```bash
# Quit Arc first, then relaunch with CDP
/Applications/Arc.app/Contents/MacOS/Arc --remote-debugging-port=9222
```

### Verify Connection

```bash
curl -s http://127.0.0.1:9222/json/version
```

## Two Ways to Control Arc

### 1. agent-browser CLI (works immediately, no restart needed)

```bash
agent-browser --cdp 9222 open https://example.com
agent-browser --cdp 9222 snapshot -i          # get interactive elements
agent-browser --cdp 9222 click @e1            # click element
agent-browser --cdp 9222 screenshot /tmp/screenshot.png
```

### 2. chrome-devtools MCP (requires Claude Code restart)

Config in `~/.claude.json`:
```json
"chrome-devtools": {
  "type": "stdio",
  "command": "npx",
  "args": [
    "chrome-devtools-mcp@latest",
    "--browserUrl",
    "http://127.0.0.1:9222"
  ],
  "env": {}
}
```

After restart, `mcp__chrome-devtools__*` tools will connect to Arc.

## Key Notes

- Arc identifies as Chrome over CDP (it's Chromium-based)
- Arc preserves existing cookies/sessions, so logged-in sites work without re-auth
- The Arc window is visible and interactive — user can see all actions happening in real time
- Default chrome-devtools MCP auto-launches a separate headless Chrome — the `--browserUrl` flag overrides this to connect to Arc instead

## Gotcha: Auto-launched Chrome

Without `--browserUrl`, chrome-devtools MCP launches its own Chrome at:
`~/.cache/chrome-devtools-mcp/chrome-profile`

This is a separate browser the user can't see. Always use `--browserUrl` or `agent-browser --cdp` to control Arc.
