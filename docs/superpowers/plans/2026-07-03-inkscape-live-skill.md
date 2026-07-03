# `inkscape-live` Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `toolkit` plugin skill that loads the live-Inkscape-session operational protocol (arming, semantics, troubleshooting) whenever a session needs to read or edit the user's open macOS Inkscape document.

**Architecture:** Single `SKILL.md` in `skills/inkscape-live/` — pure knowledge skill, no assets or scripts. Content is fixed by the approved design spec (`docs/superpowers/specs/2026-07-03-inkscape-live-skill-design.md`): five sections (Preflight, Arming, Semantics, Working effectively, Troubleshooting).

**Tech Stack:** Claude Code plugin skill (markdown + YAML frontmatter). No code. Validation is structural (frontmatter parses, description within limits, sections match spec) since skills have no unit-test harness.

---

### Task 1: Write SKILL.md

**Files:**
- Create: `skills/inkscape-live/SKILL.md`

- [ ] **Step 1: Create the skill file**

Write `skills/inkscape-live/SKILL.md` with YAML frontmatter (`name: inkscape-live`, multi-line `description` that (a) triggers on live-document editing asks in English and Spanish, (b) explicitly excludes headless SVG file work) and the five body sections from the spec. Full content is authored in this step — the spec's Section 1–5 bullets are the source of truth; each becomes a `##` section. Must include verbatim:
- the never-call-`live_arm_socket` warning with the underscore-action-id reason
- the arming wait one-liner: `RV="$TMPDIR/inkscape-mcp-live.json"; for i in $(seq 1 600); do [ -f "$RV" ] && break; sleep 1; done; ls "$RV"`
- the four-row troubleshooting table (desync, missed window, missing menu item, false capability error)
- installer path `~/workspace/inkscape-mcp-server/scripts/install-live-helper.sh` with the user-must-run-it TCC note
- `INKSCAPE_MCP_PROCESS_TIMEOUT_S=300` requirement

- [ ] **Step 2: Structural validation**

Run:
```bash
python3 - <<'EOF'
import re, sys
raw = open('/Users/alilloig/workspace/toolkit/skills/inkscape-live/SKILL.md').read()
m = re.match(r'^---\n(.*?)\n---\n', raw, re.S)
assert m, 'frontmatter missing'
fm = m.group(1)
assert re.search(r'^name: inkscape-live$', fm, re.M), 'name field wrong'
desc = re.search(r'^description: (.+?)(?=^\w|\Z)', fm, re.M | re.S)
assert desc and 50 < len(desc.group(1)) < 4000, 'description length out of range'
body = raw[m.end():]
for section in ['Preflight', 'Arming', 'Session semantics', 'Working effectively', 'Troubleshooting']:
    assert re.search(rf'^## .*{section}', body, re.M | re.I), f'missing section: {section}'
assert 'live_arm_socket' in body and 'INKSCAPE_MCP_PROCESS_TIMEOUT_S=300' in body
assert 'inkscape-mcp-live.json' in body
print('SKILL.md structure OK')
EOF
```
Expected: `SKILL.md structure OK`

- [ ] **Step 3: Commit**

```bash
cd /Users/alilloig/workspace/toolkit
git add skills/inkscape-live/SKILL.md
git commit -m "feat: add inkscape-live skill (live macOS Inkscape session protocol)"
```

### Task 2: Register in plugin docs

**Files:**
- Modify: `README.md` (skills list, if one exists)
- Modify: `CHANGELOG.md` (unreleased entry)

- [ ] **Step 1: Check whether README lists skills**

Run: `grep -n "html-artifact\|for-dummies" /Users/alilloig/workspace/toolkit/README.md | head -5`
If it lists skills, add an `inkscape-live` row/bullet in the same format: "inkscape-live — live-edit the open macOS Inkscape document through the inkscape MCP server (arming protocol, session semantics, troubleshooting)." If README has no skills list, skip.

- [ ] **Step 2: Add CHANGELOG entry**

Prepend under an `## Unreleased` heading (create if absent, matching the file's existing style):
```markdown
- feat: `inkscape-live` skill — operational protocol for live-editing the open
  macOS Inkscape document via the inkscape MCP server (extension-socket arming,
  modal session semantics, desync troubleshooting).
```

- [ ] **Step 3: Commit**

```bash
cd /Users/alilloig/workspace/toolkit
git add README.md CHANGELOG.md
git commit -m "docs: register inkscape-live skill in README/CHANGELOG"
```

### Task 3: Live trigger validation

**Files:** none (verification only)

- [ ] **Step 1: Verify the plugin picks up the new skill**

The toolkit plugin is installed from the local checkout. Run:
`claude --version >/dev/null && ls ~/.claude/plugins 2>/dev/null | head` to locate how toolkit is loaded (marketplace cache vs local path). If loaded from the local repo path, the new skill is live immediately; if from a marketplace cache, note that the user needs a plugin refresh/version bump and report it instead of forcing a release.

- [ ] **Step 2: Dry-run the skill content**

Read `skills/inkscape-live/SKILL.md` top-to-bottom once as if executing it against the current machine state: every referenced path (`installer script`, rendezvous `$TMPDIR` file name, MCP tool names `check_live_support`, `live_connect`, `live_get_scene`, `live_render_view`, `live_insert_svg`, `live_wait_for_change`) must exist/be spelled exactly as the server exposes them (cross-check against `~/workspace/inkscape-mcp-server/src/inkscape_mcp/tools/live.py` tool names). Fix any drift and amend the Task 1 commit if needed.

- [ ] **Step 3: Report**

Tell the user the skill is in place and (per Step 1's finding) whether it is already loadable or needs a plugin refresh; offer the optional one-click live test (fresh session, "draw something in my open inkscape document").
