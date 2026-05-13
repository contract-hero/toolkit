---
name: move-call-chains
description: |
  Generates HTML call-chain diagrams for Sui Move packages, organized as user
  stories. This skill should be used when the user asks to "map the call chains",
  "diagram the function flows", "visualize Move package functions", "create call
  chain diagrams", "show me the function call graph", "map the public API", or
  wants to understand how public/entry functions chain through a Move codebase.
  Also trigger when the user says "call chains", "function flow diagram",
  "move flow map", or "/move-call-chains". Output is a single self-contained
  HTML file with inline SVG box-and-arrow diagrams (one per user story).
---

# Move Call Chain Diagrams

Generate a comprehensive call-chain reference document for Sui Move packages. The output is a single self-contained HTML file with inline SVG box-and-arrow diagrams showing every public/entry function's internal call chain, organized by user stories that follow the project's domain logic. The same visibility/role-tagging conventions from `references/ascii-style-guide.md` apply — they just map onto SVG primitives instead of ASCII characters.

## Process Overview

1. **Extract** — Run the extraction script to inventory all functions
2. **Explore** — Read key modules to trace call chains
3. **Group** — Organize functions into domain-driven user stories
4. **Diagram** — Generate one inline `<svg>` diagram per independent operation
5. **Verify** — Build a completeness checklist to ensure full coverage

## Step 1: Extract Function Inventory

Run the extraction script on all Move packages in the project:

```bash
python3 ~/.claude/skills/move-call-chains/scripts/extract-move-functions.py \
  packages/pkg1 packages/pkg2 [...]
```

This produces a TSV with columns: `PACKAGE | MODULE | VISIBILITY | FUNCTION | PARAMS`

Review the output to understand scope:
- Count public vs. public(package) vs. private functions
- Identify which modules are large (many functions) vs. small (accessors only)
- Note any `entry` functions (direct transaction entry points)

## Step 2: Trace Call Chains

For each public function that performs meaningful work (not a pure accessor), read the source file and trace:

1. What validation helpers does it call?
2. What other module functions does it delegate to?
3. Does it branch (e.g., oversubscribed vs. undersubscribed)?
4. What external package calls does it make (PAS, Sui framework)?
5. What events does it emit?

**Priority order:** Start with the most complex modules (typically trading/matching engines and allocation logic), then work through simpler lifecycle functions.

**Pure accessors** (functions that just return a field) do not need call chain tracing. List them in a table in the appendix instead.

## Step 3: Organize as User Stories

Group functions into user stories based on the project's domain lifecycle. Each story should answer: "As a [role], I want to [action], so that [outcome]."

**Pattern for grouping:**
- Functions that share the same shared object (e.g., `Subscription`) often belong to the same story
- Functions gated by the same `Auth<Role>` form natural admin vs. user groups
- The project's lifecycle stages (creation → primary market → secondary market → redemption) define the story order

**Typical story categories for a DeFi/bond project:**
- Platform bootstrap (init functions)
- Role/permission management
- Token/stablecoin management
- Asset creation and configuration
- Primary market operations (subscription, bidding, allocation)
- Secondary market operations (order placement, matching, cancellation)
- Distribution/payout operations
- Redemption/settlement

## Step 4: Generate SVG Diagrams

Read `references/ascii-style-guide.md` for the conceptual conventions (visibility tags, role labels, branching, cross-module edges, merging Retail/Institutional variants). The conventions are format-agnostic; render them in SVG instead of ASCII.

**Mapping ASCII → SVG:**
- ASCII box → `<g class="node pub">` containing a `<rect>` + `<text>` for the function name and a smaller `<text>` for the visibility tag
- Edge → `<line>` or `<path>` ending in an arrowhead `<marker>`
- Auth annotation on edge → `<text>` placed near the line midpoint
- Branch (`< cond? >`) → diamond `<polygon>` node with two outgoing edges
- Cross-module call → edge stroke-style differs (dashed) and the target node carries a `{EXT}` tag
- Event emission → small node with class `evt` and a `*EVT*` tag

**Key rules** (carry over verbatim):
- One `<svg>` per independent operation (do NOT combine unrelated flows)
- Connected operations with cross-references MUST stay in a single `<svg>`
- Every node carries a visibility/role tag as a small label: `[PUB]`, `[priv]`, `[pkg]`, `{EXT}`, `{COND}`, `*EVT*`
- Annotate entry-point edges with the required `Auth<Role>`
- Keep each diagram under ~800px tall; split into sub-diagrams per logical phase if it outgrows that
- Merge Retail/Institutional variants that delegate to the same `_impl` function

**Style the diagrams from one inline `<style>` block in `<head>`** so all stories share a visual language (one CSS class per visibility tier; one marker definition reused across diagrams).

**For each user story, write a `<section>`:**

```html
<section id="story-N-place-bid">
  <h2>Story N: [Title]</h2>
  <p><strong>As a</strong> [role], <strong>I want to</strong> [action], <strong>so that</strong> [outcome].</p>
  <figure>
    <svg viewBox="0 0 400 320" role="img" aria-label="place_bid call chain">
      <!-- node: place_bid [PUB] -->
      <g class="node pub" transform="translate(120,20)">
        <rect width="160" height="40"/>
        <text x="80" y="20" text-anchor="middle">place_bid</text>
        <text x="150" y="34" text-anchor="end" class="tag">[PUB]</text>
      </g>
      <!-- edge with auth -->
      <line x1="200" y1="60" x2="200" y2="100" marker-end="url(#arrow)"/>
      <text x="208" y="84" class="auth">Auth&lt;Investor&gt;</text>
      <!-- … more nodes / edges … -->
    </svg>
    <figcaption>Place-bid flow with retail/institutional split.</figcaption>
  </figure>
  <p>Additional notes or query-function tables as needed.</p>
</section>
```

Use `<table>` for any node-detail tables (cross-module call sites, event payload columns, etc.).

## Step 5: Build Completeness Checklist

After all diagrams are written, create three appendices:

**Appendix A: Pure Accessor Functions** — Table of all functions that return a field with no interesting call chain. Columns: Module, Function, Returns.

**Appendix B: Package-Internal Infrastructure** — Brief description of `public(package)` utility modules (constants, keys, version) that appear in diagrams but aren't directly user-callable. Include a table of key operations per module.

**Appendix C: Completeness Checklist** — Flat table listing EVERY public and public(package) function, with a column indicating which Story or Appendix covers it. Cross-reference against the Step 1 extraction output to ensure nothing was missed.

## Output File

Write the document to `CALL_CHAINS.html` at the project root (or the location the user specifies). Structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Function-Level Call Chain Reference</title>
  <style>/* … inline CSS: node classes per visibility, arrow marker, layout … */</style>
</head>
<body>
  <header><h1>Function-Level Call Chain Reference</h1></header>
  <nav aria-label="Stories">…anchor links to every story + appendix…</nav>
  <main>
    <section id="how-to-read"><h2>How to Read These Diagrams</h2><!-- legend with small SVG examples --></section>
    <section id="story-1-…"><h2>Story 1: …</h2><!-- svg + notes --></section>
    <section id="story-2-…"><h2>Story 2: …</h2><!-- … --></section>
    <!-- … -->
    <section id="appendix-a-accessors"><h2>Appendix A: Pure Accessor Functions</h2><table>…</table></section>
    <section id="appendix-b-infrastructure"><h2>Appendix B: Package-Internal Infrastructure</h2>…</section>
    <section id="appendix-c-checklist"><h2>Appendix C: Completeness Checklist</h2><table>…</table></section>
  </main>
</body>
</html>
```

If a legacy `CALL_CHAINS.md` already exists, leave it in place (git history may reference it) and write the new `.html` alongside it; the user can delete the `.md` afterwards.

## Additional Resources

### Scripts
- **`scripts/extract-move-functions.py`** — Extracts all function declarations from Move source files with visibility, module, and parameter info

### Reference Files
- **`references/ascii-style-guide.md`** — Conceptual conventions: visibility/role tag glossary, edge and auth-label syntax, branching rails, banner-style grouping, cross-module labelling. ASCII shapes documented there map onto SVG primitives per the Step 4 "Mapping ASCII → SVG" table.

---

## HTML Output Conventions

**REQUIRED REFERENCE:** Use [html-artifact:html-conventions](../html-artifact/references/html-conventions.md) for the self-contained HTML output rules — semantic HTML5, inline CSS/SVG, no external assets, system-font stack, max-width ~80ch, mobile-responsive, no JavaScript by default.

Skill-specific conventions:

- **Inline SVG diagrams** with `viewBox="…" role="img" aria-label="…"`. Define one arrowhead `<marker>` once in `<defs>` and reuse via `marker-end="url(#arrow)"`.
- **CSS classes for visibility tiers**: `.pub`, `.priv`, `.pkg`, `.ext`, `.cond`, `.evt` — drive the legend from the same stylesheet that styles diagram nodes, so visibility is style-driven, not text-driven.
- **One `<svg>` per independent operation.** Connected operations with cross-references stay in a single `<svg>`. Keep each diagram under ~800px tall; split into sub-diagrams per logical phase if it outgrows that.
- **Appendix tables** with `<thead>`/`<tbody>` for accessor / infrastructure / completeness coverage.
- **`<details><summary>`** for any "advanced" notes that would otherwise bloat a story's main flow.
