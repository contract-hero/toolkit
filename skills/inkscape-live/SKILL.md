---
name: inkscape-live
description: Live-edit the document currently OPEN in the macOS Inkscape GUI through
  the `inkscape` MCP server (jjjsood/inkscape-mcp-server, extension-socket transport).
  Use whenever the user asks to draw into, read, inspect, or modify their open/live
  Inkscape document — "draw something in inkscape", "add X to my open document",
  "edit my inkscape document", "what's in my inkscape canvas", "inkscape live",
  "dibuja en inkscape", "edita mi documento de inkscape". The live path has a strict
  operational protocol (user-click arming with a 120s window, modal sessions, one
  undo step per session) that WILL fail if improvised — load this skill BEFORE the
  first live_* tool call. Do NOT load for headless SVG file work (creating,
  converting, rendering, or exporting SVG files under the workspace root): the
  server's own how_do_i / list_capabilities discovery covers that without arming.
---

# Inkscape Live Session Protocol

Controls the document open in the running macOS Inkscape GUI via the `inkscape`
MCP server's extension-socket transport. Empirically validated on Inkscape 1.4.4 /
macOS (2026-07). Follow the phases in order.

## 1. Preflight

- Call `check_live_support`. Require `helper_installed: true` and an
  `extension-socket` entry in `transports`. `live_enabled` must be true.
- If the helper is NOT installed: the user must run
  `~/workspace/inkscape-mcp-server/scripts/install-live-helper.sh` themselves
  (`~/Library` is TCC-denied for Claude — suggest they type
  `! ~/workspace/inkscape-mcp-server/scripts/install-live-helper.sh`), then
  **restart Inkscape** (extensions load only at startup).
- **NEVER call `live_arm_socket`.** Upstream bug: it activates action id
  `org.inkscape_mcp.live.noprefs` (underscore), but GLib registers the action
  hyphenated (`org.inkscape-mcp.live.noprefs`), so the arm never fires — and its
  launcher spawns a NEW Inkscape instance instead of arming the user's window.
- The server registration must carry `INKSCAPE_MCP_PROCESS_TIMEOUT_S=300`
  (see Troubleshooting row 1 for why). It is set in this machine's user-scope
  registration; if a fresh setup lost it, re-add with
  `claude mcp add inkscape --scope user -e INKSCAPE_MCP_WORKSPACE_ROOTS=... -e INKSCAPE_MCP_PROCESS_TIMEOUT_S=300 -- uv run --directory ~/workspace/inkscape-mcp-server inkscape-mcp`.

## 2. Arming (one user click per session)

Only the user can arm the bridge, from the Inkscape GUI:
**Extensions ▸ inkscape-mcp ▸ inkscape-mcp Live Bridge**.

Hard constraints:

- The helper accepts exactly **one** connection and waits at most **120 s**
  after the click; then it gives up and the arm is lost.
- A disconnect **consumes** the arm — every new session needs a new click.
- Therefore: never ask for the click until you are ready to connect
  immediately afterward.

Interactive pattern (this is the whole trick):

1. Tell the user to click the menu item.
2. In the SAME turn, block on the rendezvous file (10-min patience for the
   human, 1 s poll so you win the 120 s race):

   ```bash
   RV="$TMPDIR/inkscape-mcp-live.json"; for i in $(seq 1 600); do [ -f "$RV" ] && break; sleep 1; done; ls "$RV"
   ```

3. The moment it exists, call `live_connect`. Do not run anything else in
   between.

Background-job variant: human notification latency will blow the 120 s window,
so fuse watcher + driver into ONE background command (poll loop that invokes
the connect-and-work driver the instant the file appears).

## 3. Session semantics (tell the user)

- **Modal:** the GUI is frozen while the session is connected. Expected — warn
  the user, batch your work, disconnect promptly.
- **Snapshot:** the session sees the document as of the arm click; GUI edits
  made after arming are invisible until the next session.
- **One undo step:** all mutations merge into the document at `live_disconnect`
  as a single undoable operation (one Cmd+Z reverts the whole session).
- **Idle timeout:** 120 s of socket silence kills the session and unfreezes the
  GUI.
- Mutating tools — `live_insert_svg`, `live_apply_to_selection`,
  `live_set_selected_text` — refuse without an explicit `approval_token`
  string (any non-empty value; it is an audit marker, not a secret).

## 4. Working effectively

- First call after connect: `live_get_scene` (no args) → canvas width/height in
  user units plus visible objects with bboxes. Derive all placement math from
  these dims; large-format documents (e.g. print backwalls) are thousands of
  user units across, so never assume web-page coordinates.
- `live_insert_svg`: compose the fragment with explicit `id`s on every element
  and a single canvas-relative `transform` on the root `<g>`. Fragments are
  safe-parsed (no DTD, no entities, no network); plain SVG only.
- Proof renders: `live_render_view` with `scale` well below 1 (e.g. 0.1 on a
  7500-unit canvas). Full-res renders of big documents are multi-MB and slow.
- Change detection between operations: `live_wait_for_change` (cheap
  revision/selection polling) — not repeated renders.
- Read `live_get_active_document` early and confirm with the user it is the
  document they mean (path + object count).

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `live session communication failed` on every mutation | Session desynced: mutating tools run a hidden full-canvas preview render first; after an Inkscape restart the cold-start font-cache rebuild pushes it past the socket timeout and the wrapper swallows the error, leaving stale bytes on the wire | `live_disconnect`, re-arm, retry (caches now warm). Prevent: `INKSCAPE_MCP_PROCESS_TIMEOUT_S=300` on the registration |
| Rendezvous file never appears, or vanished before connect | 120 s accept window missed, or the arm was consumed by an earlier connect | Ask the user to click again; have the wait-loop already running |
| Extension absent from the Extensions menu | Helper installed while Inkscape was running (extensions load at startup) | Restart Inkscape |
| `the active live transport does not support this operation` right after another failure | Same desync as row 1 — the capability table is fine, the session is dead | Same as row 1 |
