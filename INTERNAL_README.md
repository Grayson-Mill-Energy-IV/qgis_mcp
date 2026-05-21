# Grayson Mill internal fork of `jjsantos01/qgis_mcp`

**Upstream:** https://github.com/jjsantos01/qgis_mcp
**Pilot version tag:** `pilot-v1.1` (patched build)
**Owner:** Corey [add email]
**Pilot end date:** [add date]

## Why this fork exists

We forked `jjsantos01/qgis_mcp` to (1) pin a known version for our QGIS+Claude
pilot, (2) keep a chokepoint for patches, and (3) maintain an audit trail of
any changes we make to the upstream code.

## Changes from upstream

### `execute_code` is disabled

The upstream plugin exposes an `execute_code` tool that runs arbitrary Python
inside the QGIS process via `exec()`. The plugin's socket server has no
authentication — any local process that can reach `localhost:9876` can send
commands to it. The combination creates a remote code execution surface that
is unacceptable for a corporate environment with access to N: drive, internal
data, and cached credentials.

For the Grayson Mill pilot we have disabled `execute_code` in three places:

1. **MCP server (`src/qgis_mcp/qgis_mcp_server.py`):** the `@mcp.tool()`
   registration is removed, so Claude Desktop never sees the tool exists.
2. **Plugin dispatch table (`qgis_mcp_plugin/qgis_mcp_plugin.py`):** the
   `"execute_code"` entry in the handler dictionary is commented out.
3. **Plugin handler method (`qgis_mcp_plugin/qgis_mcp_plugin.py`):** the
   method body is replaced with `raise Exception(...)` and the original
   implementation is preserved (but unwired) as `_execute_code_DISABLED`
   for traceability.

The other 14 tools are unchanged and cover the full range of operations the
pilot is designed to test: project management, layer loading, feature
inspection, processing algorithms, styling, rendering, and saving.

### Plugin metadata updated

`qgis_mcp_plugin/metadata.txt` has been updated to identify this as a
Grayson Mill build (`version=1.0-gmellc.1`) and to remove the upstream
placeholder author/email fields.

## What pilot users should do

- Install from the `pilot-v1.1` tag, not from `main`, and not from upstream.
- Run the plugin only while actively using it; stop it when done.
- Do **not** change the host setting from `localhost`.
- Do **not** open port 9876 in Windows Firewall.
- Review every file-write approval prompt in Claude Desktop carefully,
  especially `save_project` and `render_map` — paths are not validated.

## What pilot users should NOT do

- Do not install from upstream (`jjsantos01/qgis_mcp`).
- Do not modify files in their local clone.
- Do not re-enable `execute_code`. If you find yourself wanting it, talk to
  the pilot owner first.
- Do not enable this plugin on machines holding production data you cannot
  afford to lose. The pilot is for evaluating safety and utility, not for
  use on critical projects.

## Known limitations (carried over from upstream, not patched)

- **No socket framing.** The server uses a buffer that parses the entire
  contents as JSON on every receive. Rapid back-to-back commands or
  unbounded byte streams can cause hangs or memory growth. Restart QGIS
  if the plugin becomes unresponsive.
- **No path validation.** File paths supplied by Claude are passed directly
  to QGIS. Claude Desktop's approval prompts are the safety net here.
- **Liveness check bug.** A small bug in `qgis_mcp_server.py` causes the
  socket connection to be torn down and recreated on every tool call. This
  is a performance issue, not a security issue.

## Upstream sync

We are NOT pulling upstream changes during the pilot. If the pilot succeeds
and we move toward broader rollout, we will re-evaluate upstream commits
since `e37bf6f` and decide whether to merge them into a future pilot tag.

## Contact

For issues, questions, or to request a patch, contact the pilot owner.
