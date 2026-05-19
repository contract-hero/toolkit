---
name: html-artifact
description: Use when asked to produce a one-off self-contained HTML deliverable — visual report, explainer page, side-by-side comparison, box-and-arrow diagram, web page that visualizes a concept — and no more specialized skill applies. Triggers on phrases like "make an HTML for X", "visualize Y as a webpage", "create an explainer", "render this as a single .html file", "compare these as a webpage". Use when the request is for an ad-hoc visual deliverable that doesn't match for-dummies, game-design, move-call-chains, sui-marp-theme, marp-slide-content, pdf-visual-to-css-svg, or technical-docs-to-learning-materials.
---

# HTML Artifact

You produce a single self-contained `.html` file as a visual deliverable, using the shared conventions in `references/html-conventions.md`. Every output is double-click-openable with no network and no external assets.

**REQUIRED REFERENCE:** `references/html-conventions.md` — the self-containment rules, semantic structure, CSS aesthetic, and medium-fit decision criteria. Load it before writing.

## When to use this skill

- "Make an HTML for X" / "render this as a webpage" / "visualize this"
- Side-by-side comparisons that don't fit a markdown table
- Box-and-arrow diagrams, flow diagrams, dependency maps
- One-off explainers that benefit from collapsible sections, inline SVG, or visual emphasis prose can't carry
- Ad-hoc reports where the conventions in `html-conventions.md` are a sensible default

## When NOT to use this skill

A more specialized skill always wins. Defer to it.

| If the request is… | Use this instead |
|---|---|
| `*_FOR_DUMMIES.html` intro guide for a project | `for-dummies` |
| Game Design Document | `game-design` |
| Call-chain / function-flow diagram for a Move package | `move-call-chains` |
| Marp slide deck (slides, not a webpage) | `marp-slide-content` (+ `sui-marp-theme` for Sui branding) |
| CSS theme or SVG asset extracted from a PDF | `pdf-visual-to-css-svg` |
| Multi-file linked learning module | `technical-docs-to-learning-materials` |
| Sui SDK 2.0 migration audit report | `sui-2-migration-audit` |
| CLI documentation verification report | `cli-documentation-verification` |
| **Sharing or publishing** an existing HTML file | `publish-html` (separate concern — never bundle it here) |

When you're unsure whether a specialized skill applies, name the candidate and ask the user via `AskUserQuestion` before producing the artifact.

## Step 1: Decide if HTML is actually the right medium

Read the medium-fit decision in `references/html-conventions.md`. If none of the four criteria apply (spatial, comparison, interactive, multi-file collection), produce markdown instead and say so. The CLAUDE.md preference for HTML applies to *document deliverables*, not every output.

## Step 2: Decide the output path

Default: write next to the source file (the file or directory the artifact is *about*). If the request has no clear source anchor:

1. Check if the working directory has a natural artifacts location (e.g. `./artifacts/`, `./docs/`).
2. If not, use `AskUserQuestion` to ask the user where to write.

Filename: kebab-case, descriptive, ends in `.html` (e.g. `oauth-flow-explainer.html`, `pricing-comparison.html`). No `UPPER_SNAKE_CASE` — that's `for-dummies`'s convention specifically.

## Step 3: Apply the shared conventions

Load `references/html-conventions.md` and follow it. The conventions cover:

- Document shell (doctype, head, body skeleton)
- Semantic structure (which tags for what)
- Diagrams (inline SVG, not external `<img>`)
- CSS style (system fonts, max-width ~72ch, conservative palette, mobile-responsive)
- Collapsibles, callouts
- When (rarely) to add JavaScript

**Do not redefine conventions in the artifact's own `<style>` block beyond what the deliverable specifically needs.** The shared aesthetic exists so successive artifacts feel like they belong to the same family.

## Step 4: Verify before writing

Before the `Write` call, cross-check:

- [ ] No `<link href="https://…">`, no `<script src="https://…">`, no external font CDN
- [ ] All images either inline `<svg>` or base64 `data:` URIs
- [ ] Semantic tags used (not just `<div>` everywhere)
- [ ] Mobile-responsive (`@media (max-width: 720px)` or equivalent)
- [ ] Dark / light mode tokens present (`:root` palette + `@media (prefers-color-scheme: dark)` override) and component rules reference them via `var(--…)` — no hardcoded hex values in CSS
- [ ] Inline SVG uses `currentColor` for strokes/text and `var(--…)` for colored fills — no `fill="#000"` or `stroke="#000"` anywhere
- [ ] `<title>` is descriptive, not the filename
- [ ] Content actually benefits from HTML — re-read the medium-fit decision

## Step 5: Report the path

After writing, print the absolute path of the artifact and tell the user how to view it (`open <path>` on macOS, or just "double-click in Finder"). If the user is likely to want to share it, mention `publish-html` as the next step — never publish automatically.

## Common mistakes

- **Re-inventing conventions** — repeating the font stack, max-width, or color palette inline because the shared reference wasn't loaded. Always load `references/html-conventions.md` first.
- **Wrong skill chosen** — writing a "for dummies" guide via this skill instead of the dedicated `for-dummies` skill. Re-read the "When NOT to use" table.
- **Hidden external dependency** — `<link href="https://fonts.googleapis.com/…">` sneaks in via muscle memory. Use the system-font stack always.
- **Bundling the publish step** — never push, commit, or share the artifact from this skill. That's `publish-html`'s job, and the user must invoke it explicitly.
- **Over-styling** — gradients, glass-morphism, neon palettes, emoji-decorated headings. Conservative wins. The conventions list these as anti-patterns for a reason.
- **Misleading SVG nesting** — boxes inside boxes implying containment when the real relationship is peer-to-peer with an arrow. Re-check diagrams for accuracy.

## What this skill does NOT cover

- **Authoring the underlying content** beyond visual rendering choices. If the user asks for a technical explainer on a topic, do the research first as you would for any artifact — this skill only governs *how* to render the result as HTML.
- **Slide decks** — those are `marp-slide-content` + a theme skill, even when "make a deck" sounds adjacent.
- **Long-form documentation** with no spatial/comparison/interactive value — that's markdown.
- **Publishing or sharing** — `publish-html`, invoked separately.
