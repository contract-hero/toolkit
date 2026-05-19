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
- **Use the token palette below** — reference colors through CSS custom properties (`var(--…)`) instead of literal hex values. The one exception is `currentColor` inside inline SVG (it resolves to `var(--fg)` via the cascade). Pick the 2–4 semantic colors your artifact needs *from the tokens*; if you need a state color the palette doesn't cover, extend the tokens in both light and dark branches — don't drop a raw `#hex` inline.

## Dark / light mode (required)

Artifacts must follow the reader's OS appearance — light by default, dark when the system is in dark mode. The mechanism is pure CSS via `prefers-color-scheme` so the artifact stays self-contained and JS-free, and it works whether the artifact is double-clicked into a standalone browser or rendered inside Vlervcode's `srcdoc` iframe (which inherits the OS preference). The one caveat: opaque cross-origin sandboxed iframes may not propagate the preference automatically — when in doubt, test the rendering host.

Drop this token block at the top of the inline `<style>` and reference the variables everywhere instead of literal hex values:

```css
/* ★ TWEAKABLE — palette aligns with Vlervcode tokens so artifacts opened
   inside Vlervcode feel visually coherent with the app chrome. Adjust if
   the artifact needs a different identity, but keep both branches in sync
   and re-check contrast (target WCAG AA 4.5:1 for body text). */
:root {
  color-scheme: light dark;
  --bg: #ffffff;
  --fg: #1f1f1f;
  --fg-muted: #6e6e6e;
  --heading-fg: #1a1a1a;
  --accent: #1976d2;
  --border: #e0e0e0;
  --code-bg: #f0f0f0;
  --code-fg: #6e3700;
  --pre-bg: #fafafa;
  --aside-bg: #f5f5f5;
  --success: #2e7d32;
  --warning: #9a5b00;
  --error: #d32f2f;
}
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #1e1e1e;
    --fg: #cccccc;
    --fg-muted: #9e9e9e;
    --heading-fg: #e8e8e8;
    --accent: #4dabf7;
    --border: #2a2a2a;
    --code-bg: #2a2a2a;
    --code-fg: #f0e8d8;
    --pre-bg: #161616;
    --aside-bg: #242424;
    --success: #66bb6a;
    --warning: #f0c674;
    --error: #ff7676;
  }
}

body { background: var(--bg); color: var(--fg); }
h1, h2, h3, h4 { color: var(--heading-fg); }
a { color: var(--accent); }
small, figcaption, th { color: var(--fg-muted); }
code { background: var(--code-bg); color: var(--code-fg); }
pre { background: var(--pre-bg); border: 1px solid var(--border); }
aside { background: var(--aside-bg); border-left: 3px solid var(--accent); padding: .6rem .9rem; }
aside.success { border-left-color: var(--success); }
aside.warning { border-left-color: var(--warning); }
aside.error   { border-left-color: var(--error); }
```

Every token in `:root` is wired into at least one component rule above — the demo is the canonical mapping. When an artifact needs a token as text color rather than as a border accent (e.g. `color: var(--warning)` for an inline status word), the light values clear WCAG AA against `--bg`; the dark values clear AA against the dark `--bg` with margin.

Notes:

- **`color-scheme: light dark`** tells the browser to render native UI (scrollbars, form controls, default focus outline) in a matching scheme. Without it, scrollbars stay light on a dark page and look broken.
- **Don't add a manual toggle UI.** A one-off artifact isn't worth a JS theme-switcher; the OS preference is the source of truth.
- **Inline SVG must adapt too.** Use `currentColor` for strokes/text fills (`<text fill="currentColor">`, `<line stroke="currentColor">`) so SVG follows `var(--fg)`. For colored elements, set `fill="var(--accent)"` or similar — CSS variables cascade into inline SVG. Hardcoded `fill="#000"` on a dark-mode artifact is an instant tell.
- **Test both modes before reporting done.** Easiest path: open the artifact in a browser and toggle "Emulate CSS prefers-color-scheme" in devtools (Chrome: Rendering panel). Or on macOS, flip System Settings → Appearance and reload.

### Print mode (opt-in)

By default artifacts inherit their on-screen theme when printed — a dark-mode artifact prints as white-on-dark, which wastes ink and is hard to read. If print fidelity matters for the artifact (audit reports, reference cards, anything a reader might actually print), append a `@media print` block that forces every token back to its light value. The full override:

```css
@media print {
  :root {
    color-scheme: light;
    --bg: #ffffff;
    --fg: #1f1f1f;
    --fg-muted: #6e6e6e;
    --heading-fg: #1a1a1a;
    --accent: #1976d2;
    --border: #e0e0e0;
    --code-bg: #f0f0f0;
    --code-fg: #6e3700;
    --pre-bg: #fafafa;
    --aside-bg: #f5f5f5;
    --success: #2e7d32;
    --warning: #9a5b00;
    --error: #d32f2f;
  }
}
```

If the artifact is purely on-screen (an explainer for a chat reply, a one-off comparison), skip the print block — it's noise.

### Extending the palette

The base tokens cover prose, links, code, callouts, and the three state colors (success/warning/error). Artifacts with more semantic dimensions need additional tokens — keep them in the same shape:

- **Categorical / tier scales.** A move-call-chains diagram has six visibility tiers (`pub | priv | pkg | ext | cond | evt`); a log viewer has five severity levels. Define one token per category in both branches, named by *meaning*, not by hue: `--tier-public`, `--tier-private`, `--tier-package`, … — never `--blue`, `--red`. Drive the legend from the same stylesheet that styles the diagram nodes (style-driven, not text-driven).
- **State + emphasis pairs.** If you need both a border accent and a text color for the same state, add `--success-fg`, `--warning-fg`, `--error-fg` alongside the base tokens — body text needs AA contrast (4.5:1), borders don't. Don't reuse a border token as a text color without checking contrast first.
- **Diagram-only tokens.** Box fills, edge labels, marker arrowheads — name them by role (`--diagram-node-bg`, `--diagram-edge`, `--diagram-callout`) and ensure they stay distinguishable in both modes.

When in doubt, run the artifact through a contrast checker (Chrome devtools → Inspect → Accessibility, or browser-extension equivalents) before declaring the palette done.

## Collapsibles and callouts

- `<details><summary>` for long appendix lookups, FAQs, or "advanced" drill-downs that would bloat the main flow.
- `<aside class="note">`, `<aside class="success">`, `<aside class="warning">`, `<aside class="error">` for in-context callouts. The neutral `note` class uses `--accent` for its left rule; the three state classes swap in `--success`, `--warning`, `--error` respectively. Place callouts where they matter, not in a separate "warnings" section.

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
