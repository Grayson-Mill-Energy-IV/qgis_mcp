# QGIS + Claude Pilot — Setup Guide

**For pilot users:** follow these steps to install and verify the QGIS MCP integration. If anything breaks or doesn't match what's described, ping Corey before improvising — we want to keep the pilot environment consistent across users.

**Estimated time:** 30–45 minutes. If you hit 60 and aren't past Step 5, stop and ping Corey.

---

## What this is

Claude can now control QGIS directly through a secure local integration. Once set up, you can ask Claude things like *"load the well_pads shapefile and tell me how many features it has"* and Claude will actually do it in your live QGIS session.

## What this isn't

- This is a **pilot**, not a production tool. We're evaluating whether it earns its place in our workflow.
- Claude can only operate on QGIS while you have the plugin's server running. When you close QGIS, the connection drops.
- Don't enable this on machines holding production data you can't afford to lose. Use it on test data or copies.

---

## Step 0 — Confirm your Claude install

You should have the Microsoft Store version of Claude. Verify with:

```powershell
Get-ChildItem -Path $env:USERPROFILE -Filter "claude_desktop_config.json" -Recurse -ErrorAction SilentlyContinue | Select-Object FullName
```

You should see a path that looks like:

```
C:\Users\<your-username>\AppData\Local\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\claude_desktop_config.json
```

**Save this path** — you'll use it in Step 4. We'll call it `<config-path>` from here on.

If you see a different path (e.g. one that doesn't include `Packages\Claude_pzs8sxrjxfjjc`), stop and ping Corey — your install is different from what this doc assumes and we'll need to adjust.

If nothing is found at all, it just means you haven't opened the developer config yet. That's fine — the file will be created in Step 4 at the expected Store-version path.

---

## Step 1 — Confirm prerequisites

Open PowerShell and check:

```powershell
# QGIS
qgis --version    # If "not recognized," QGIS isn't on PATH — that's fine, just confirm you can launch it from Start menu
git --version     # Should return git version 2.x or higher
uv --version      # Should return 0.x or higher
```

If `git` or `uv` is missing:

```powershell
winget install --id=Git.Git
winget install --id=astral-sh.uv
```

Close and reopen PowerShell after installing, then re-check.

**Also need:** QGIS 3.28 or newer (3.40 LTR recommended). Check via QGIS → Help → About.

---

## Step 2 — Clone the Grayson Mill fork

You'll be installing from our internal fork, **not** the upstream public version. This pins you to a security-patched build.

```powershell
# Create or use a clones folder
mkdir C:\_Git Clones -ErrorAction SilentlyContinue
cd "C:\_Git Clones"

# Clone at the pilot tag (NOT the latest main)
git clone --branch pilot-v1.1 https://github.com/Grayson-Mill-Energy-IV/qgis_mcp.git
cd qgis_mcp
```

**If the clone fails with "Repository not found":** the fork is private and you need read access. Message Corey with your GitHub username and he'll grant it.

**Read the INTERNAL_README.md** in the cloned folder before proceeding. It explains what was patched and why, and what you should and shouldn't do.

---

## Step 3 — Install the QGIS plugin

1. **Close QGIS if it's open.** The plugin files are locked while QGIS is running.

2. **Copy the plugin folder** into your QGIS plugins directory:

   ```powershell
   $pluginDir = "$env:APPDATA\QGIS\QGIS3\profiles\default\python\plugins"

   # If there's a pre-existing qgis_mcp_plugin folder, back it up
   if (Test-Path "$pluginDir\qgis_mcp_plugin") {
       Rename-Item "$pluginDir\qgis_mcp_plugin" "$pluginDir\qgis_mcp_plugin.before-pilot-backup"
   }

   # Copy the patched plugin
   Copy-Item -Recurse "C:\_Git Clones\qgis_mcp\qgis_mcp_plugin" "$pluginDir\qgis_mcp_plugin"

   # Verify the right version is in place — should output Grayson Mill lines
   Get-Content "$pluginDir\qgis_mcp_plugin\metadata.txt" | Select-String "Grayson Mill"
   ```

   You should see at least two lines containing "Grayson Mill." If you don't, stop and ping Corey.

3. **Start QGIS.** Go to **Plugins → Manage and Install Plugins → All tab → search "QGIS MCP"**. You should see:

   > **QGIS MCP (Grayson Mill pilot build)**

   Check the box to enable it. The "(Grayson Mill pilot build)" suffix confirms you have the right version.

4. **Find the plugin's toolbar button** (a small icon in the QGIS toolbar) or open it via **Plugins → QGIS MCP**. A dock widget appears on the right side.

5. **Click "Start Server"** in the dock widget. Status should change to **"Server: Running on port 9876"**.

---

## Step 4 — Configure Claude

In Claude, open **Settings → Developer → Edit Config**. This opens (or creates) `claude_desktop_config.json` in your default text editor.

If the file is empty or just contains `{}`, paste this entire block:

```json
{
  "mcpServers": {
    "qgis": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "C:\\_Git Clones\\qgis_mcp",
        "src/qgis_mcp/qgis_mcp_server.py"
      ]
    }
  }
}
```

If the file already has other MCP servers in it, add only the `"qgis": { ... }` block inside the existing `mcpServers` object. **Don't forget the comma between entries** — JSON is unforgiving.

(The `--directory` flag tells `uv` exactly where to find the project, which is required because the Microsoft Store version of Claude doesn't honor a separate `"cwd"` config field.)

Save the file.

**Validate the JSON** using the `<config-path>` you noted in Step 0:

```powershell
Get-Content "C:\Users\<your-username>\AppData\Local\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\claude_desktop_config.json" | ConvertFrom-Json
```

(Replace `<your-username>` with your actual Windows username.)

No error output means the JSON parsed cleanly. An error means there's a syntax issue (usually a missing or extra comma).

---

## Step 5 — Restart Claude and verify

1. **Fully quit Claude.** Right-click the Claude icon in the Windows system tray (bottom-right, near the clock — you may need to click the up-arrow to see hidden icons) → **Quit**. Closing the window isn't enough.

2. **Reopen Claude.**

3. **Open a new conversation.** Click the **"+"** button at the bottom of the chat input → **Connectors**. You should see **`qgis`** in the list with a toggle that's **on**.

4. **Open the qgis connector's settings** (click on `qgis` in the list). You should see **"Other tools 14"** with a list including `ping`, `get_qgis_info`, `load_project`, etc. **There should be exactly 14 tools** — if you see 15 and `execute_code` is one of them, stop and ping Corey, you have the wrong plugin version.

5. **The smoke test.** In the new conversation, type:

   > Use the qgis ping tool, then call get_qgis_info to confirm what version of QGIS you're talking to.

   Claude will ask for approval before each tool call (the hand-icon "Needs approval" prompt). Approve both.

   You should see back something like:
   - `{"pong": true}` from ping
   - QGIS version info like `QGIS 3.40.x-Bratislava`

   **If you see those, you're done with setup.**

---

## Tool approval recommendation

By default, every QGIS tool prompts for approval each time. You can auto-approve safe (read-only) tools to reduce friction, but **leave write tools as "Needs approval"** for the pilot.

In the qgis connector settings, set these to ✅ (Always allow):

- `ping`
- `get_qgis_info`
- `get_layers`
- `get_project_info`
- `get_layer_features`
- `zoom_to_layer`

Leave these as ✋ (Needs approval) — they modify state:

- `load_project`, `create_new_project`, `save_project`
- `add_vector_layer`, `add_raster_layer`, `remove_layer`
- `execute_processing`
- `render_map`

---

## Day-to-day usage

Order matters:

1. **Open QGIS first.** Click the QGIS MCP toolbar button → **Start Server**.
2. **Then open Claude** (or start a new conversation if it's already open).
3. **Stop the server when you're done.** Click **Stop Server** in the dock widget before closing QGIS, especially if you'll be away from your machine.

---

## When things go wrong

| Symptom | Most likely cause | Fix |
|---|---|---|
| Claude says "I don't have QGIS tools" | You're in claude.ai (web) instead of the Claude app, or the connector isn't enabled | Check + → Connectors; toggle qgis on |
| Claude shows the qgis connector but tools fail | QGIS isn't running OR the plugin server isn't started | Start QGIS, click Start Server in the dock widget |
| ⚠️ warning on the qgis connector | The MCP server failed to spawn | Check the MCP log (see below) and ping Corey |
| Step 3 doesn't show "Grayson Mill pilot build" | You copied the wrong folder, or the clone wasn't from `pilot-v1.1` | Re-do Step 2 with `--branch pilot-v1.1` and Step 3 |

**To find Claude's MCP log:**

```powershell
Get-ChildItem -Path $env:USERPROFILE -Filter "mcp*qgis*.log" -Recurse -ErrorAction SilentlyContinue |
  Sort-Object LastWriteTime -Descending | Select-Object -First 1 |
  ForEach-Object { Get-Content $_.FullName -Tail 30 }
```

Paste the output when you ping Corey.

---

## Pilot rules — please read

- **Use it on real work as opportunities come up.** We want ~5 real sessions per user over the two weeks.
- **Log every session** in `pilot-log.md` (Corey will share the location). Even short sessions count.
- **Flag any "I almost let it do something bad" moments immediately.** Slack Corey. These are the most important signals from the pilot.
- **Don't run destructive operations on data you can't restore.** Make a backup first, even if Claude swears it's safe.
- **Don't enable any other MCP server alongside qgis without telling Corey.** Adding random connectors mid-pilot changes the threat model.
- **Don't modify the plugin code locally.** If you find a bug or want a feature, message Corey and we'll patch the fork.

---

## Contact

Issues, questions, or anything weird → Corey (Slack or email).

**Pilot end date:** TBD
