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

Generate a comprehensive call-chain reference document for Sui Move packages. The output is a single self-contained HTML file with inline SVG box-and-arrow diagrams showing every public/entry function's internal call chain, organized by user stories that follow the project's domain logic. The visibility/role-tagging conventions live in `references/svg-style-guide.md`, which defines the native-SVG visual language for these diagrams.

## Process Overview

1. **Extract** — Use the move-analyzer LSP to inventory all functions (regex script is a fallback)
2. **Explore** — Read key modules to trace call chains
3. **Group** — Organize functions into domain-driven user stories
4. **Diagram** — Generate one inline `<svg>` diagram per independent operation
5. **Verify** — Build a completeness checklist to ensure full coverage

## Step 1: Extract Function Inventory

Build the inventory from the **move-analyzer LSP**, not from regex. The
`mcp__plugin_sui-pilot_move-lsp__move_document_symbols` tool parses each file
with the real Move grammar, so it catches declarations a line-by-line regex
silently drops — `public entry fun`, `macro fun`, `public(package) macro fun`,
and signatures whose modifiers/params wrap across lines. (The tool's own
description says to prefer it over regex extraction.) The legacy
`scripts/extract-move-functions.py` is kept only as a **fallback for when
move-analyzer is unavailable** — see "Fallback" below; it is known to
under-count and must not be the primary source.

### Procedure

1. **Enumerate source files.** `Glob` each package's `sources/**/*.move`. Skip
   anything under `build/`. Skip `tests/**` unless the user asked to include
   test functions.

2. **Call the LSP per file.** For each file, call `move_document_symbols` with
   its absolute `filePath`. The result is:

   ```jsonc
   {
     "workspaceRoot": "…",
     "symbols": [
       { "name": "<module>", "kind": "module", "range": {…},
         "children": [
           { "name": "EFoo",  "kind": "constant", "range": {…} },
           { "name": "Bar",   "kind": "struct",   "range": {…}, "children": [/* fields */] },
           { "name": "place_bid", "kind": "function",
             "range": { "startLine": 241, "startCharacter": 20, … } },
           …
         ] }
     ]
   }
   ```

   - The **module name** is the top-level `kind: "module"` symbol's `name`.
   - Functions are the `kind: "function"` entries in that module's `children`.
   - `range.startLine` is **0-indexed** — add 1 to get the editor line number.
     Keep this line number; it makes Step 2 call-chain tracing far cheaper.
   - `range.startCharacter` is the **column where the function name begins**.

3. **Warm-up — empty `symbols` on the first call.** The very first
   `move_document_symbols` call against a freshly-opened package usually returns
   `"symbols": []` while move-analyzer indexes the workspace. This is **not** an
   empty module. If you get `[]`, call `move_diagnostics` once on any file in
   that package (it forces indexing), then re-call `move_document_symbols`.
   Subsequent files in the same workspace return immediately. Never record a
   module as "no functions" off a single empty response — always retry once.

4. **Classify visibility from the declaration line.** The LSP outline does not
   carry visibility, but it hands you the exact location, so read it straight
   from source. For each function symbol, `Read` the file at `startLine + 1` and
   inspect the text **before the name** — i.e. columns `0 .. startCharacter`,
   which is exactly the modifier prefix:

   | Prefix substring (cols `0..startCharacter`) | VISIBILITY  |
   |---------------------------------------------|-------------|
   | `fun `                                      | `private`   |
   | `public fun `                               | `public`    |
   | `public(package) fun `                      | `public_package` |
   | `entry fun `                                | `entry`     |
   | `public entry fun `                         | `public` + entry (tag as entry point) |
   | contains `macro fun `                       | append `macro` |

   Match on the keywords present (`public`, `public(package)`, `entry`,
   `macro`), not on exact column arithmetic — indentation and `macro`/`entry`
   combinations shift the column.

5. **Drop test functions.** A function is a test if the line(s) immediately
   above its declaration carry `#[test]` or `#[test_only]`, or the file lives
   under `tests/`. The LSP lists these like any other function, so filter them
   out unless tests were explicitly requested.

6. **Read multi-line signatures fully when needed.** For params, read from the
   `(` after the name to its matching `)`, which may span several lines (e.g.
   `mint_and_transfer` in the framework `coin` module). The param list is
   secondary to visibility + name + line for call-chain work, so capture it only
   where it clarifies a flow.

Assemble the same inventory the old TSV produced —
`PACKAGE | MODULE | VISIBILITY | FUNCTION | PARAMS` — now augmented with the
declaration **line number** from the LSP.

Review the inventory to understand scope:
- Count public vs. public(package) vs. private functions
- Identify which modules are large (many functions) vs. small (accessors only)
- Note any `entry` / `public entry` functions (direct transaction entry points)

### Fallback (move-analyzer unavailable)

If the sui-pilot move-lsp MCP server is not connected (e.g. `move-analyzer`
isn't installed), fall back to the regex script and **state in the output that
the inventory is best-effort and may under-count**:

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/move-call-chains/scripts/extract-move-functions.py \
  packages/pkg1 packages/pkg2 [...]
```

It emits the same `PACKAGE | MODULE | VISIBILITY | FUNCTION | PARAMS` columns
but misses `public entry fun`, `macro fun`, and wrapped declarations — so
cross-check anything that looks thin against the source before trusting it.

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

**REQUIRED REFERENCE:** Read `references/svg-style-guide.md` first. It defines the native-SVG visual language — tier tokens, node shapes, curved edges, branching, auth labels, phase bands, and the legend — plus a complete worked `<svg>`.

Diagrams are **hand-authored inline SVG designed natively**. Do NOT use Mermaid, Graphviz, or any DSL, and do NOT transcribe ASCII shapes — no hard-cornered boxes, straight `<line>` pipes, or `<polygon>` "ASCII diamonds" standing in for real shapes. Design for SVG: rounded nodes (`rx≈8`), cubic-Bézier edge `<path>`s, token-driven color per tier, a reused arrowhead `<marker>`, and real type hierarchy.

**Tier → shape (see the guide for the CSS and full example):**
- Entry point (`public`/`entry`) → rounded rect, 2px accent stroke, bold label, `[PUB]` tag
- Private helper → rounded rect, muted stroke, `[priv]`
- `public(package)` → rounded rect, pkg-token stroke, `[pkg]`
- External (framework/PAS/3rd-party) → rounded rect, **dashed** stroke, qualifier in label, `{EXT}`
- Branch → **diamond**, two labeled out-edges, `{COND}`
- Event → **stadium/pill** (`rx = height/2`), `*EVT*`

**Key rules:**
- One `<svg>` per independent operation (do NOT combine unrelated flows); connected operations with cross-references stay in a single `<svg>`.
- Every node keeps its textual tag — color is never the only channel (accessibility + grayscale print).
- Edges are curved `<path>`s with a single shared `<defs>` arrowhead; cross-module/external edges are dashed.
- Annotate entry-point edges with the required `Auth<Role>` (escape generics: `Auth&lt;Role&gt;`).
- Group phases with faint labeled **bands**, not enclosing mega-boxes.
- Keep each diagram under ~800px tall; split into per-phase sub-diagrams if it outgrows that. Merge variants that delegate to a shared `_impl` onto one node.

**Style every diagram from one inline `<style>` block in `<head>`** (one class per tier, one marker, tokens for dark/light) so all stories share the language. Make SVGs responsive: `svg { max-width: 100%; height: auto; }`.

**For each user story, write a `<section>`** wrapping a `<figure>` + `<svg>` + `<figcaption>`. Use the worked example in `references/svg-style-guide.md` as the structural template:

```html
<section id="story-N-place-bid">
  <h2>Story N: [Title]</h2>
  <p><strong>As a</strong> [role], <strong>I want to</strong> [action], <strong>so that</strong> [outcome].</p>
  <figure>
    <svg viewBox="0 0 420 300" role="img" aria-label="place_bid call chain">
      <!-- nodes (rounded rects / diamond / pill), curved <path> edges with
           marker-end="url(#arrow)", auth + edge labels — per the style guide -->
    </svg>
    <figcaption>place_bid flow with retail/institutional split.</figcaption>
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

### Primary tool
- **`mcp__plugin_sui-pilot_move-lsp__move_document_symbols`** — grammar-accurate
  function/struct/constant outline per file (see Step 1). This is the primary
  inventory source.

### Scripts
- **`scripts/extract-move-functions.py`** — **Fallback only.** Regex-based
  extraction of function declarations (visibility, module, params). Used when
  move-analyzer is unavailable; under-counts `public entry fun`, `macro fun`,
  and wrapped declarations.

### Reference Files
- **`references/svg-style-guide.md`** — The native-SVG visual language: tier tokens (dark/light), node shapes per visibility/role, curved edge `<path>`s + shared arrowhead marker, branching diamonds, auth labels, phase bands, cross-module/external styling, and a complete worked `<svg>` plus legend.

---

## HTML Output Conventions

**REQUIRED REFERENCE:** Use [html-artifact:html-conventions](../html-artifact/references/html-conventions.md) for the self-contained HTML output rules — semantic HTML5, inline CSS/SVG, no external assets, system-font stack, max-width ~80ch, mobile-responsive, no JavaScript by default.

Skill-specific conventions:

- **Inline SVG diagrams** with `viewBox="…" role="img" aria-label="…"`. Define one arrowhead `<marker>` once in `<defs>` and reuse via `marker-end="url(#arrow)"`. Strokes and text use `currentColor` so the diagram follows the dark/light theme; marker fills reference `var(--accent)` or tier tokens (see below) — never hardcoded hex.
- **CSS classes for visibility tiers**: `.pub`, `.priv`, `.pkg`, `.ext`, `.cond`, `.evt` — define one CSS-variable token per tier per branch (light + dark), following the "Categorical / tier scales" pattern in [html-conventions.md → Extending the palette](../html-artifact/references/html-conventions.md). Drive the legend from the same stylesheet that styles diagram nodes, so visibility is style-driven, not text-driven.
- **Dark / light verify.** Before writing the artifact, confirm: `:root` palette + `@media (prefers-color-scheme: dark)` override are present, every component rule references `var(--…)` (no raw hex), and every edge `<path>` / `<text>` / arrowhead either uses `currentColor` or a tier token.
- **One `<svg>` per independent operation.** Connected operations with cross-references stay in a single `<svg>`. Keep each diagram under ~800px tall; split into sub-diagrams per logical phase if it outgrows that.
- **Appendix tables** with `<thead>`/`<tbody>` for accessor / infrastructure / completeness coverage.
- **`<details><summary>`** for any "advanced" notes that would otherwise bloat a story's main flow.
