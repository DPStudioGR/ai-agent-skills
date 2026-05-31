---
name: chrome-cdp-connect
description: Use when the agent needs browser-cdp access — checks if Chrome CDP is already running on port 9222, launches it if not, and confirms the connection is ready. Use before any task requiring browser_cdp tool.
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [browser, cdp, chrome, automation, windows]
    related_skills: []
---

# Chrome CDP Auto-Connect

## Overview

The `browser_cdp` tool requires Chrome to be running with `--remote-debugging-port=9222`.
This skill lets the agent check, launch, and verify Chrome CDP automatically — no manual
intervention needed from the user.

Chrome is launched with an isolated profile (`C:\Temp\chrome-cdp`) so it never interferes
with the user's normal Chrome session. Both can run simultaneously.

## When to Use

- Before using the `browser_cdp` tool for any task
- When `browser_cdp` fails with a connection error
- When the user asks the agent to "open a browser", "control Chrome", or "use CDP"
- Do NOT use if the agent only needs normal browsing (`browser_navigate`, `browser_click`) — those use the built-in agent-browser and do not need CDP

## Step-by-Step Procedure

### Step 1 — Check if CDP is already running

Run via the `terminal` tool:

```powershell
netstat -ano | findstr ":9222 " | findstr "LISTENING"
```

- **If output is non-empty**: Chrome CDP is already running. Skip to Step 3.
- **If output is empty**: Chrome needs to be launched. Continue to Step 2.

### Step 2 — Launch Chrome with CDP enabled

Run via the `terminal` tool:

```powershell
if (-not (Test-Path "C:\Temp\chrome-cdp")) { New-Item -ItemType Directory -Path "C:\Temp\chrome-cdp" }

Start-Process -FilePath "C:\Program Files\Google\Chrome\Application\chrome.exe" -ArgumentList `
  "--remote-debugging-port=9222",
  "--user-data-dir=C:\Temp\chrome-cdp",
  "--no-first-run",
  "--no-default-browser-check",
  "--disable-background-networking"
```

Then wait for the port to become available:

```powershell
$deadline = (Get-Date).AddSeconds(15)
$ready = $false
while ((Get-Date) -lt $deadline) {
    $check = netstat -ano | Select-String ":9222 " | Select-String "LISTENING"
    if ($check) { $ready = $true; break }
    Start-Sleep -Milliseconds 500
}
if ($ready) { Write-Host "Chrome CDP ready on port 9222" }
else { Write-Host "Chrome CDP did not start in time — check Chrome installation" }
```

### Step 3 — Verify CDP endpoint is responsive

```powershell
try {
    $resp = Invoke-RestMethod -Uri "http://127.0.0.1:9222/json/version" -TimeoutSec 5
    Write-Host "CDP connected: $($resp.Browser)"
} catch {
    Write-Host "CDP endpoint not responding: $_"
}
```

A successful response looks like:
```json
{ "Browser": "Chrome/136.x.x.x", "webSocketDebuggerUrl": "ws://127.0.0.1:9222/..." }
```

### Step 4 — Proceed with browser_cdp tool

Once Step 3 confirms the connection, use the `browser_cdp` tool normally.
The `BROWSER_CDP_URL=http://127.0.0.1:9222` env var is already set in Hermes config.

## Navigating to a URL via CDP (optional)

To open a specific URL in the CDP Chrome instance after connecting:

```powershell
$tab = Invoke-RestMethod -Uri "http://127.0.0.1:9222/json/new?http://example.com" -Method PUT
Write-Host "Opened tab: $($tab.id)"
```

## Closing Chrome CDP (cleanup)

When done with CDP tasks, optionally close the Chrome instance to free resources:

```powershell
Get-Process chrome | Where-Object {
    $_.MainWindowTitle -eq "" -or
    (netstat -ano | Select-String ":9222" | Select-String $_.Id)
} | Stop-Process -Force
```

Or simply leave it running — Chrome's isolated profile does not affect normal browsing.

## Common Pitfalls

1. **Port 9222 conflict**: If something else is on 9222, check with `netstat -ano | findstr ":9222"` and kill the conflicting process, or change the port in both the launch command and `BROWSER_CDP_URL` in `.env`.

2. **Chrome path wrong**: The path `C:\Program Files\Google\Chrome\Application\chrome.exe` is correct for this machine. If Chrome updates move the binary, check `where.exe chrome` or the registry key `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\chrome.exe`.

3. **Profile locked**: If Chrome was previously closed uncleanly, the profile at `C:\Temp\chrome-cdp` may have a lock. Delete `C:\Temp\chrome-cdp\Default\lockfile` if Chrome refuses to start.

4. **CDP responds but pages don't load**: This is normal for a fresh profile — navigate using the CDP `Page.navigate` method or via the JSON endpoint (`/json/new?<url>`).

5. **Using browser_cdp without this skill**: The tool will fail with a WebSocket connection error if Chrome isn't running with `--remote-debugging-port=9222`. Always run this skill first.

## Verification Checklist

- [ ] `netstat -ano | findstr ":9222"` shows a LISTENING entry
- [ ] `http://127.0.0.1:9222/json/version` returns a valid JSON response with `Browser` field
- [ ] `browser_cdp` tool is available in current session (check with `hermes tools`)
- [ ] `BROWSER_CDP_URL=http://127.0.0.1:9222` is set in `~/.hermes/.env`
