# Agent Town + Pi Integration — Implementation Log

This file records all completed work, technical decisions, validation, and known limitations for the Agent Town integration with Pi Coding Agent on Windows.

## 1. Preflight & Installation

- Verified environment: Node **v24.16.0**, npm **11.13.0**, Git **2.55.0.windows.3**, globally installed Pi **v0.99.2**.
- Inspected Pi package at `D:\Program\npm-global\node_modules\@earendil-works\pi-coding-agent` (especially `docs/extensions.md` and `dist/core/extensions/types.d.ts`) to confirm event APIs, session ID, model access, and extension semantics.
- Listed existing Pi packages (`pi list`) and preserved all existing extensions/configurations.
- Cloned Agent Town into `D:\Program\agent-town`; ran `npm install` and `npm install ws` (needed by bridge).
- Built the browser/library bundle with `npm run build:lib` producing `dist/agent-town.js` (ES module), `dist/agent-town.umd.cjs`, and declaration files.
- `npm audit` reported **17 dependency vulnerabilities**; `npm audit fix` was **not** run (no automatic dependency upgrades).

## 2. Pi Extension

**Path:** `C:\Users\muhamad.arwinwijaya\.pi\agent\extensions\agent-town.ts`

- Uses Pi's public `ExtensionAPI` and `ExtensionContext`. Session ID comes from `ctx.sessionManager.getSessionId()`, model display from `ctx.model?.name`/`model_select`.
- Stable Agent Town ID format: `pi-<session-id>` across model changes; model selection emits `PiModel` for display update (not remove/spawn).
- Supports events: `session_start`, `session_shutdown`, `before_agent_start`, `tool_execution_start`, `tool_execution_end`, `agent_end`, `model_select`.
- Strict tool allowlist: `read`, `edit`, `write`, `grep`, `find`, `ls`, `bash`, `powershell`.
- Privacy: file tools report sanitized basenames only; shell tools report `Running command`; no streaming output, raw prompts, raw shell commands, API keys, or auth headers are forwarded.
- Uses Node 24 native `fetch` + `AbortSignal.timeout(1200)`; POST is fire-and-forget and failures are caught. No network work runs in the extension factory itself.
- Enabled only when `AGENT_TOWN_ENABLED` equals exactly `"true"`. `AGENT_TOWN_URL` defaults to `http://127.0.0.1:3001`; only HTTP loopback hosts (`localhost`, `127.0.0.1`, `[::1]`) are accepted.

## 3. Bridge Server & Dashboard

**Bridge path:** `D:\Program\agent-town\examples\bridge-server.mjs`

- HTTP/WS bridge binds to `127.0.0.1:3001` only.
- POST `/hooks`: maps Pi hooks (`SessionStart`, `SessionEnd`, `PiModel`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Stop`) to WebSocket `spawn`, `update`, and `remove`; retains upstream mappings for Claude Code/Cursor/Codex.
- GET `/health`: reports `ok`, agent count, client count.
- GET `/agent-town.js`: serves the built ESM library; requires `npm run build:lib` first.
- GET `/live`: standalone pixel-agent page that directly imports the ESM library and connects to the bridge WebSocket. This is the actual browser-ready alternative to the example HTML with unresolved bare imports.
- Tracks current agent state in memory and replays active agents to new WS connections.
- Pi-specific validation accepts only `pi-...` ID format and safe tool/display fields, with generic messages for shell activity.
- Next.js app at `http://localhost:3000` is also available; the bridge-served `/live` page is the verified live telemetry view.

## 4. PowerShell Lifecycle Scripts

- **Start:** `D:\Program\agent-town\start-agent-town.ps1`. Checks process ownership before using ports 3001/3000. Can skip Next.js with `-NoDashboard` and browser auto-open with `-NoBrowser`. Uses separate `bridge.log`/`bridge-err.log` and `dev.log`/`dev-err.log`.
- **Stop:** `D:\Program\agent-town\stop-agent-town.ps1`. Stops only listeners that match Agent Town bridge / Next.js command lines; unrelated processes on those ports are not stopped.
- Full usage and troubleshooting: `D:\Program\agent-town\PI_AGENT_TOWN_INTEGRATION.md`.

## 5. Event Mapping

| Pi event | Bridge hook | Pixel status / action |
|---|---|---|
| `session_start` | `SessionStart` | `spawn`, idle |
| `before_agent_start` | `PreToolUse` (`Agent Thinking`) | thinking |
| `tool_execution_start` (read/find/grep/ls) | `PreToolUse` | reading |
| `tool_execution_start` (edit/write/bash/powershell) | `PreToolUse` | typing |
| `tool_execution_end` (success) | `PostToolUse` | thinking |
| `tool_execution_end` (error) | `PostToolUseFailure` | error |
| `agent_end` | `Stop` | success |
| `model_select` | `PiModel` | updates display name (same ID) |
| `session_shutdown` | `SessionEnd` | remove |

## 6. Tests & Results

- **Preflight:** Bridge health returned `{"ok":true,"agents":0,"clients":0}`.
- **Dashboard delivery:** `/live` returned HTTP **200** (`text/html`) and `/agent-town.js` returned HTTP **200** (`text/javascript`) after `npm run build:lib`.
- **WebSocket smoke test:** A WS client observed Pi `spawn → update (reading) → update (thinking) → update (success) → remove` from posted hooks, and a separate direct test saw spawn, tool update, stop, and end.
- **Extension hook flow:** With `AGENT_TOWN_ENABLED=true`, a `pi -p "bash node --version"` test emitted `SessionStart`, `PreToolUse`, `PostToolUse`, `Stop`, `SessionEnd` to the bridge (observed in bridge logs).
- **Pi offline fail-safe:** With the bridge stopped, `AGENT_TOWN_ENABLED=true pi -p "bash node --version"` completed normally and printed Node **v24.16.0**. A warning from the unrelated `@tintinweb/pi-tasks` package appeared but is not an Agent Town failure.
- **Process scripts:** PowerShell syntax parsed successfully; `start-agent-town.ps1 -NoDashboard -NoBrowser` started bridge PID, and stop script successfully shut it down without touching Pi.
- **Loopback binding:** Verified Windows listener was `127.0.0.1:3001`, not `0.0.0.0:3001`.

## 7. Limitations / Follow-up

- **Concurrent Pi sessions:** A real two-session simultaneous WebSocket observation was not conclusively captured. Stable per-session IDs are implemented, but distinct concurrent agent visualization should be explicitly re-tested interactively with two overlapping sessions. The prior claim of full concurrent-session validation was premature.
- **Browser visual verification:** HTTP routes and WS messages were verified; the page was not visually inspected in a browser automation run.
- **Subagents:** Dedicated subagent lifecycle visualization is not implemented because verified Pi subagent lifecycle events were not available. Tool activity remains on the main Pi pixel.
- **Next.js dashboard:** The bridge-served `/live` page is the working telemetry visualization. The Next.js app on `:3000` may still be the upstream playground rather than an integrated live-telemetry page.
- **Security scope:** Loopback binding and payload sanitization reduce exposure; no authentication is implemented on the local bridge. Other local processes can POST hooks. Avoid exposing this port via proxy/tunnel.
- **Upstream bridge behavior:** Legacy Claude Code/Cursor/Codex hook mapping is retained; those non-Pi payloads are not subjected to the same Pi-specific sanitization policy. Use the bridge only on a trusted local host.

## 8. Quick Start

```powershell
# One-time: ensure browser bundle exists
cd D:\Program\agent-town
npm run build:lib

# Start visualization
.\start-agent-town.ps1 -NoDashboard
# Open http://127.0.0.1:3001/live

# In another PowerShell window, enable telemetry for this Pi invocation
$env:AGENT_TOWN_ENABLED = "true"
$env:AGENT_TOWN_URL = "http://127.0.0.1:3001"  # optional; this is the default
pi

# Stop visualization
.\stop-agent-town.ps1
```

Pi/9router routing and model/provider configuration have **not** been modified by this integration.
