# Design: `inkscape-live` skill

**Date:** 2026-07-03
**Status:** Approved (brainstormed interactively; scope and location chosen by user)
**Target:** `skills/inkscape-live/SKILL.md` in the `toolkit` plugin

## Problem

The `inkscape` MCP server (jjjsood/inkscape-mcp-server, cloned at
`~/workspace/inkscape-mcp-server`, registered at user scope) can control the **live**
macOS Inkscape GUI document, but only through an operational protocol that is easy to
get wrong and was established through hours of empirical debugging on this machine
(2026-07-02/03). A fresh Claude session that doesn't know this protocol will:

- Try `live_arm_socket` (broken upstream — underscore action id, spawns a stray
  Inkscape instance instead of arming the user's).
- Lose the race against the helper's 120-second accept window.
- Misread "live session communication failed" (a desynced session after a swallowed
  render timeout) as a connection problem and retry uselessly.
- Not know edits are modal, snapshot-based, and merge as one undo step.

Headless (file-based) tools need no skill: the server's own `how_do_i` /
`list_capabilities` discovery layer covers them.

## Decision summary

| Question | Decision |
|---|---|
| Scope | Live-session workflow only; headless tools excluded |
| Location | `toolkit` plugin (`~/workspace/toolkit/skills/inkscape-live/`) |
| Arming-race handling | Inline foreground wait-loop one-liner in SKILL.md; no bundled assets |
| Structure | Single `SKILL.md`, no `references/`, no scripts |

## Skill structure

Frontmatter `name: inkscape-live`. Description triggers on: drawing/editing in the
user's open Inkscape document, "inkscape live", "add X to my open document",
"draw something in inkscape", Spanish variants ("dibuja en inkscape", "edita mi
documento de inkscape"). Explicitly scoped to the live GUI path — the description
tells sessions NOT to load it for headless SVG file work.

### Section 1 — Preflight

- Confirm the `inkscape` MCP server is connected; `check_live_support` must report
  `helper_installed: true` and the `extension-socket` transport.
- If the helper is missing: installer at
  `~/workspace/inkscape-mcp-server/scripts/install-live-helper.sh` (user must run it —
  `~/Library` is TCC-denied for Claude), then restart Inkscape.
- **Never call `live_arm_socket`** — upstream bug: its action id
  (`org.inkscape_mcp.live.noprefs`, underscore) never matches the registered
  hyphenated action, and it launches a NEW Inkscape instance rather than arming the
  user's running one.
- The server registration must carry `INKSCAPE_MCP_PROCESS_TIMEOUT_S=300` (see
  Troubleshooting for why).

### Section 2 — Arming protocol

- Only the user can arm: **Extensions ▸ inkscape-mcp ▸ inkscape-mcp Live Bridge** in
  the Inkscape GUI.
- The helper accepts exactly ONE connection and waits at most **120 s**; a disconnect
  consumes the arm (next session = next click). Never ask for the click before being
  ready to connect.
- Interactive pattern: tell the user to click, then block on the rendezvous file and
  connect the moment it appears:

  ```bash
  RV="$TMPDIR/inkscape-mcp-live.json"; for i in $(seq 1 600); do [ -f "$RV" ] && break; sleep 1; done; ls "$RV"
  ```

  then immediately `live_connect`.
- Background-job variant: a self-firing watcher (wait loop + driver in ONE background
  command) so no human-latency gap exists between arm and connect.

### Section 3 — Session semantics

- **Modal:** the GUI freezes while connected — normal, tell the user.
- **Snapshot:** the extension sees the document as of the arm click; user edits made
  after arming are invisible to the session.
- **One undo step:** all mutations merge into the document at disconnect as a single
  undoable operation.
- **Idle timeout:** 120 s of silence kills the session and unfreezes the GUI.
- Mutating tools (`live_insert_svg`, `live_apply_to_selection`,
  `live_set_selected_text`) require an explicit `approval_token` string.
- Work batched: connect → all reads/edits → disconnect promptly.

### Section 4 — Working effectively

- First call after connect: `live_get_scene` (no args needed) → canvas width/height
  in user units + visible objects with bboxes. Use canvas dims for placement math.
- `live_insert_svg`: compose fragments with explicit `id`s and a canvas-relative
  root `transform`; fragment is safe-parsed (no DTD/entities/network).
- Proof renders: `live_render_view` with `scale` (e.g. 0.1) — full-res renders of
  large documents are multi-MB and slow.
- Change detection between calls: `live_wait_for_change` (cheap revision/selection
  polling), not repeated full renders.

### Section 5 — Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `live session communication failed` on every call | Session desynced: hidden before-edit preview render exceeded the socket timeout (cold-start font-cache rebuild after Inkscape restart) and the wrapper swallowed it | Disconnect, re-arm, retry; ensure `INKSCAPE_MCP_PROCESS_TIMEOUT_S=300` on the registration |
| Rendezvous file never appears / vanishes | 120 s accept window missed, or arm consumed by an earlier connect | Ask user to click again; be ready to connect first |
| Extension absent from menu | Installed after Inkscape started (extensions load at startup) | Restart Inkscape |
| `the active live transport does not support this operation` after a failure | Same desync as above — capability table is fine | Same as row 1 |

## Testing

1. Structure/frontmatter validated against `plugin-dev:skill-development` conventions
   (description length, trigger phrasing, no dead references).
2. Live dry-run: fresh session with the skill installed, ask "draw something in my
   open inkscape document", verify the skill loads and the session follows the
   arming protocol correctly (one user click).

## Non-goals

- Headless/file-based tool guidance (server's `how_do_i` covers it).
- Drawing/illustration recipes (fragment aesthetics, gradients) — separate skill if
  ever needed.
- Fixing the upstream desync or `live_arm_socket` bugs (report upstream instead).
- Linux/Windows paths.
