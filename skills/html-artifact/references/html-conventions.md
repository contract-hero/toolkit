# HTML Conventions for Self-Contained Deliverables

Shared reference for skills that produce self-contained `.html` artifacts. Cross-link to this file instead of redefining these rules per skill.

## Self-containment is non-negotiable

A self-contained artifact must work when **double-clicked into a browser with no network**. That means:

- No external CSS, JS, fonts, or images. No CDNs. No `<script src="…">` to anything not on disk in the same file.
- One inline `<style>` block in `<head>`. One optional inline `<script>` block (most artifacts have none).
- Images go inline as `<svg>` or as base64-encoded `data:` URIs. No `<img src="…">` to external files.
- Fonts come from the OS via a system-font stack — never `<link href="https://fonts.…">`.

If you find yourself wanting an external asset, embed it or omit it.

## Document shell

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>…</title>
  <style>/* one inline block */</style>
</head>
<body>
  <header>…</header>
  <nav aria-label="Contents">…</nav>
  <main>
    <section id="…">…</section>
  </main>
  <footer>…</footer>
</body>
</html>
```

Adapt to content — don't over-prescribe. A flat one-page diagram doesn't need a `<nav>`.

## Semantic structure

Use semantic tags. Tables for tabular data, `<figure>`+`<figcaption>` for diagrams, `<aside>` for callouts, `<details><summary>` for collapsible drill-downs.

- `<header>`, `<nav>`, `<main>`, `<section id="…">`, `<article>`, `<aside>`, `<figure>`, `<figcaption>`, `<footer>`.
- Every section that the nav links to gets a stable `id` matching the link target.
- Tables: `<table>` with `<thead>`/`<tbody>`. Scannable columns.
- Code: `<pre><code>` for multi-line; `<code>` inline. No Prism/Highlight.js — a handful of CSS classes for keyword color is enough.

## Diagrams

Inline `<svg>` for anything bigger than a 1-line shape. `<img>` references to external files are forbidden (breaks self-containment).

- ASCII diagrams only for one-line shapes (`A → B → C`). Anything bigger goes in SVG.
- Use `<marker>` arrowheads for directional flow; label edges via `<text>`.
- Separate `<rect>`/`<g>` groups for separate entities. Don't nest boxes to imply containment unless one thing literally runs inside another — misleading nesting is the most common SVG-diagram bug.

## CSS style

Small inline stylesheet. Conservative aesthetic.

- **System-font stack**: `system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif` (or a monospace equivalent for code-heavy artifacts).
- **Max-width on prose**: ~70–80ch (`max-width: 72ch` or `max-width: 960px` for layouts with diagrams).
- **Mobile-responsive** via a single `@media (max-width: 720px)` block. Don't ship more than one breakpoint unless you need it.
- **Conservative aesthetic**: no gradients, no glass-morphism, no neon palettes, no emoji-decorated headings. Aim for "I trust this document" not "AI-styled landing page."
- **Color palette**: pick 2–4 semantic colors max. If the artifact uses state colors (success/warning/error/neutral), keep them muted and consistent.

## Collapsibles and callouts

- `<details><summary>` for long appendix lookups, FAQs, or "advanced" drill-downs that would bloat the main flow.
- `<aside class="note">`, `<aside class="warning">`, `<aside class="tip">` for in-context callouts. Place them where they matter, not in a separate "warnings" section.

## JavaScript

Default: none. Most static artifacts don't need it.

Use JS only when the artifact genuinely benefits from interactivity that can't be `<details>`-faked — clickable filtering, animated state transitions, form submission. When you do add JS, keep it inline, vanilla, and under ~50 lines.

## When HTML beats markdown (medium-fit decision)

HTML is the right medium when at least one of these is true. If none apply, write markdown.

1. **Spatial relationships matter.** Module maps, dependency graphs, layout diagrams — anything that needs boxes and arrows. Prose lists fail when the audience needs to point at one direction.
2. **Comparison is the point.** Side-by-side options, before/after, A-vs-B feature matrices. The audience compares better when items are adjacent, not sequential.
3. **Interactivity tightens the loop.** Collapsibles, filterable tables, copy buttons, anything that lets the reader explore at their own pace. If a reader needs to skim, expand, and re-skim, HTML beats prose.
4. **A linked collection beats a monolith.** When content spans multiple related topics that cross-link to each other (a 5-page learning module), produce a multi-file `index.html` + per-topic pages over one mega-document.

If the content is genuinely linear (a sequential tutorial, a single decision recommendation, a chat reply), markdown is fine. The CLAUDE.md preference for HTML applies to *document deliverables* — not every output.

## When in doubt

Lean on the working examples at <https://thariqs.github.io/html-effectiveness/>.
