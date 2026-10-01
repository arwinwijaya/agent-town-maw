# Agent Town & Pi Coding Agent Integration

This repository documents the complete integration of **Agent Town** with the globally installed **Pi Coding Agent** (v0.99.2) on Windows.

---

## 🚀 Overview

The goal of this integration is to visualize real-time Pi coding agent activity and lifecycle events as pixel agents inside Agent Town (`http://localhost:3000` or `http://127.0.0.1:3001/live`), without modifying any Pi routing, model configuration, credentials, or `9router` setups.

---

## 🛠️ Architecture & Components

```
Pi Session / 9router (Pi v0.99.2)
        │
        │ Extension lifecycle events (session_start, tool_execution_start/end, agent_end, model_select)
        ▼
Pi Agent Town Extension (`C:\Users\muhamad.arwinwijaya\.pi\agent\extensions\agent-town.ts`)
        │
        │ HTTP POST (JSON, bridge hook schema, sanitized, fire-and-forget, 1.2s AbortSignal timeout)
        ▼
Agent Town Bridge Server (`D:\Program\agent-town\examples\bridge-server.mjs`)
        │ Binds strictly to 127.0.0.1:3001
        │ Serves health (`/health`), live dashboard (`/live`), and built library (`/agent-town.js`)
        │ WebSocket fan-out (`ws://127.0.0.1:3001`)
        ▼
Agent Town Visualization
        │ Next.js dashboard (`http://localhost:3000`)
        └ Standalone bridge page (`http://127.0.0.1:3001/live`)
```

---

## 📂 Summary of Created / Modified Files

| Component | Path | Description |
|---|---|---|
| **Pi Extension** | `C:\Users\muhamad.arwinwijaya\.pi\agent\extensions\agent-town.ts` | Captures Pi events, sanitizes payloads, uses native `fetch` (fire-and-forget) to send telemetry to the bridge. |
| **Bridge Server** | `D:\Program\agent-town\examples\bridge-server.mjs` | Binds to `127.0.0.1:3001`, accepts POST hooks, serves static `/live` page and `/agent-town.js`, broadcasts WebSocket events (`spawn`/`update`/`remove`). |
| **Start Script** | `D:\Program\agent-town\start-agent-town.ps1` | PowerShell script with process ownership checks to safely start bridge (port 3001) and Next.js (port 3000) with separate logs. |
| **Stop Script** | `D:\Program\agent-town\stop-agent-town.ps1` | PowerShell script to safely stop Agent Town processes without affecting unrelated Node/npm tasks. |
| **Documentation** | `D:\Program\agent-town\PI_AGENT_TOWN_INTEGRATION.md` | Full integration guide, event mappings, configuration, testing, and uninstallation steps. |

---

## ⚙️ Configuration & Environment Variables

- **`AGENT_TOWN_ENABLED`**: Must be set to `"true"` to enable telemetry (defaults to off).
- **`AGENT_TOWN_URL`**: Bridge HTTP hooks endpoint (defaults to `http://127.0.0.1:3001`).

---

## 🚦 How to Start, Test, and Stop

### 1. Start Agent Town
```powershell
D:\Program\agent-town\start-agent-town.ps1
```
*(Or start the bridge manually: `cd D:\Program\agent-town && node examples/bridge-server.mjs`)*

### 2. Run Pi with Telemetry Enabled
In your terminal session:
```powershell
$env:AGENT_TOWN_ENABLED = "true"
pi -p "bash node --version"
```

### 3. Stop Agent Town
```powershell
D:\Program\agent-town\stop-agent-town.ps1
```

---

## 🔒 Security & Fail-Safe Guarantees
1. **Localhost Only**: Bridge binds exclusively to `127.0.0.1`.
2. **Fail-Safe Telemetry**: Extension uses native `fetch` with `AbortSignal.timeout(1200)` and swallows all errors. Bridge outages never interrupt Pi.
3. **Data Sanitization**: File tools send basenames only (`Reading demo.ts`), shell tools report generic `"Running command"`, and common secret patterns are redacted.
4. **Isolated Config**: Pi settings, `models.json`, `auth.json`, and `9router` configurations remain entirely untouched.
