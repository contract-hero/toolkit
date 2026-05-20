# SVG Style Guide for Move Call Chain Diagrams

Diagrams are **hand-authored inline SVG** — a first-class visual language,
not a transcription of ASCII art or a Mermaid/Graphviz render.

**Do NOT:**
- Use Mermaid, Graphviz, PlantUML, or any DSL/renderer.
- Transcribe ASCII shapes (`+===+`, `/--\`, `< cond? >`) into SVG. Hard
  rectangles, straight `<line>` edges, and `<polygon>` "ASCII diamonds"
  are the stiff look we are leaving behind. Design natively: rounded
  nodes, curved edge paths, token-driven color, real type hierarchy.

This guide defines the *semantics* (which role each node plays, how edges
read, how branches and auth annotations work) and the *native SVG shape*
for each. The semantics are stable; the shapes are SVG primitives.

## Color, theme, and accessibility (read first)

- **Token-driven, dark/light required.** Every fill/stroke references a CSS
  custom property — never a raw hex. Define one token per tier in `:root`
  plus a `@media (prefers-color-scheme: dark)` override, following the
  "Categorical / tier scales" pattern in
  [html-artifact html-conventions](../../html-artifact/references/html-conventions.md).
  Edge strokes and node text use `currentColor` (which resolves to `var(--fg)`)
  so they track the theme automatically.
- **Never rely on color alone.** Color identifies a tier *at a glance*; the
  textual **tag** (`[PUB]`, `[priv]`, `[pkg]`, `{EXT}`, `{COND}`, `*EVT*`)
  is the accessible channel for colorblind readers and grayscale print.
  Every node keeps its tag.
- **Shape reinforces role** too: entry points read heavier (accent stroke +
  bold label), events are stadium-shaped, branches are diamonds, externals
  are dashed. Color + tag + shape = three redundant channels.

### Suggested tier tokens

Add these alongside the base palette (values are a starting point — tune to
the host palette, keeping AA contrast for the label text on each fill):

```css
:root {
  --node-pub:  #e8f0fe; --node-pub-stroke:  var(--accent);
  --node-priv: var(--code-bg); --node-priv-stroke: var(--border);
  --node-pkg:  #eef7ee; --node-pkg-stroke:  #5b8c5a;
  --node-ext:  var(--aside-bg); --node-ext-stroke: var(--fg-muted);
  --node-cond: #fff6e5; --node-cond-stroke: var(--warning);
  --node-evt:  #f3ecfb; --node-evt-stroke:  #8a63d2;
}
@media (prefers-color-scheme: dark) {
  :root {
    --node-pub:  #1c2b46; --node-pub-stroke:  var(--accent);
    --node-priv: var(--code-bg); --node-priv-stroke: var(--border);
    --node-pkg:  #1e2c1e; --node-pkg-stroke:  #6fa86e;
    --node-ext:  var(--aside-bg); --node-ext-stroke: var(--fg-muted);
    --node-cond: #3a2f17; --node-cond-stroke: var(--warning);
    --node-evt:  #2a2140; --node-evt-stroke:  #a98ae0;
  }
}
```

## Node tiers — semantics and native shape

| Move visibility / role     | Tag      | Native SVG shape |
|----------------------------|----------|------------------|
| `public`/`entry` (entry point) | `[PUB]`  | Rounded rect (`rx≈8`), **2px accent stroke**, bold label |
| `fun` (private helper)     | `[priv]` | Rounded rect (`rx≈8`), 1px muted stroke |
| `public(package)`          | `[pkg]`  | Rounded rect (`rx≈8`), 1px `--node-pkg-stroke` |
| External (framework/PAS/3rd-party) | `{EXT}` | Rounded rect, **dashed** stroke (`stroke-dasharray="4 3"`), qualifier in label |
| Branching condition        | `{COND}` | **Diamond** (`<polygon>` or a 45°-rotated square), two labeled out-edges |
| Event emission             | `*EVT*`  | **Stadium / pill** (`rx = height/2`) |

Drive all of this from **one inline `<style>` block** so every diagram in the
document shares the language. One class per tier:

```css
.node rect, .node polygon { stroke-width: 1; }
.node text  { fill: currentColor; font: 13px/1.2 ui-sans-serif, system-ui, sans-serif; }
.node .tag  { font-size: 10px; fill: var(--fg-muted); letter-spacing: .03em; }
.pub  rect { fill: var(--node-pub);  stroke: var(--node-pub-stroke);  stroke-width: 2; }
.pub  text.label { font-weight: 600; }
.priv rect { fill: var(--node-priv); stroke: var(--node-priv-stroke); }
.pkg  rect { fill: var(--node-pkg);  stroke: var(--node-pkg-stroke); }
.ext  rect { fill: var(--node-ext);  stroke: var(--node-ext-stroke); stroke-dasharray: 4 3; }
.cond polygon { fill: var(--node-cond); stroke: var(--node-cond-stroke); }
.evt  rect { fill: var(--node-evt);  stroke: var(--node-evt-stroke); }
.edge { fill: none; stroke: currentColor; stroke-width: 1.5; }
.edge.dashed { stroke-dasharray: 5 4; }      /* cross-module / external call */
.edge-label { font-size: 11px; fill: var(--fg-muted); }
.auth { font-size: 11px; fill: var(--node-pub-stroke); font-style: italic; }
.band { fill: var(--aside-bg); opacity: .5; }
.band-label { font-size: 11px; fill: var(--fg-muted); text-transform: uppercase; letter-spacing: .06em; }
```

A node is then a small group — rounded rect + centered label + corner tag:

```html
<g class="node pub" transform="translate(120,20)">
  <rect width="160" height="44" rx="8"/>
  <text class="label" x="80" y="26" text-anchor="middle">place_bid</text>
  <text class="tag"   x="152" y="38" text-anchor="end">[PUB]</text>
</g>
```

## Edges — curved, not straight

Use a cubic-Bézier `<path>` so flows read as smooth connectors, not pipes.
Define the arrowhead **once** in `<defs>` and reuse it:

```html
<defs>
  <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5"
          markerWidth="7" markerHeight="7" orient="auto-start-reverse">
    <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
  </marker>
</defs>

<!-- vertical-ish connector with a gentle curve -->
<path class="edge" d="M200,64 C200,84 200,84 200,104" marker-end="url(#arrow)"/>
```

- Straight-down flows can still use a short vertical `C` with equal control
  points (visually a line, but consistent with the curved family).
- **Cross-module / external** calls use `class="edge dashed"`.
- Label an edge with `<text class="edge-label">` near its midpoint; pad it a
  few px off the stroke so it never sits on the line.

## Auth annotations

Annotate the out-edge from every entry-point node with the authorization
object the call requires, as `<text class="auth">`. Escape generics:
`Auth&lt;Investor&gt;`, `Coin&lt;USDC&gt;`.

```html
<path class="edge" d="M200,64 C200,86 200,86 200,108" marker-end="url(#arrow)"/>
<text class="auth" x="208" y="90">Auth&lt;Investor&gt;</text>
```

## Branching

A `{COND}` diamond with two labeled out-edges. Reconverging branches point
their paths back to a single node below.

```html
<g class="node cond" transform="translate(150,120)">
  <polygon points="60,0 120,28 60,56 0,28"/>
  <text class="label" x="60" y="32" text-anchor="middle">oversubscribed?</text>
</g>
<path class="edge" d="M180,176 C150,196 130,196 110,212" marker-end="url(#arrow)"/>
<text class="edge-label" x="120" y="198">yes</text>
<path class="edge" d="M240,176 C270,196 290,196 310,212" marker-end="url(#arrow)"/>
<text class="edge-label" x="290" y="198">no</text>
```

## Grouping (phases) — labeled bands, not mega-boxes

Do not wrap groups in a giant bordered rect (brittle to size). Instead lay a
faint full-width **band** behind a phase and label it in the top-left:

```html
<rect class="band" x="0" y="100" width="400" height="120" rx="6"/>
<text class="band-label" x="10" y="118">Validation</text>
```

## One diagram per independent operation

Each independent public operation gets its **own `<svg>`**. Do NOT bundle
unrelated flows (e.g. `mint` and `burn`) in one `<svg>` — they compete for
space and stop being readable.

**Exception:** connected operations whose flows reference one another (a
dispatcher fanning out to variants, or branches that converge) MUST stay in
a single `<svg>` so the cross-references are visible.

Keep each `<svg>` under ~800px tall. If it grows past that, split into
sub-diagrams per logical phase (validate → core → finalize → emit), each its
own `<svg>` with a "continues below" `<figcaption>`. Make diagrams
responsive: `svg { max-width: 100%; height: auto; }`.

## Cross-module calls

Put the qualifier in the node label so the reader knows where the function
lives (`book_matching::match_from_bid`), and draw the incoming edge dashed
(`class="edge dashed"`). External packages additionally use the `.ext` node
class and `{EXT}` tag.

## Merge variants

When retail/institutional (or similar) entry points delegate to the same
`_impl`, draw both entry nodes converging onto one shared implementation node
rather than duplicating the downstream chain.

## Worked example

A minimal flow exercising an entry point, auth label, private helper, a
branch, and two package-visibility variants:

```html
<figure>
  <svg viewBox="0 0 420 300" role="img" aria-label="place_bid call chain">
    <defs>
      <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5"
              markerWidth="7" markerHeight="7" orient="auto-start-reverse">
        <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
      </marker>
    </defs>

    <g class="node pub" transform="translate(130,16)">
      <rect width="160" height="44" rx="8"/>
      <text class="label" x="80" y="26" text-anchor="middle">place_bid</text>
      <text class="tag"   x="152" y="38" text-anchor="end">[PUB]</text>
    </g>
    <path class="edge" d="M210,60 C210,74 210,74 210,88" marker-end="url(#arrow)"/>
    <text class="auth" x="218" y="78">Auth&lt;Investor&gt;</text>

    <g class="node priv" transform="translate(120,88)">
      <rect width="180" height="44" rx="8"/>
      <text class="label" x="90" y="26" text-anchor="middle">check_valid_bid</text>
      <text class="tag"   x="172" y="38" text-anchor="end">[priv]</text>
    </g>
    <path class="edge" d="M210,132 C210,146 210,146 210,156" marker-end="url(#arrow)"/>

    <g class="node cond" transform="translate(150,156)">
      <polygon points="60,0 120,26 60,52 0,26"/>
      <text class="label" x="60" y="30" text-anchor="middle">route?</text>
    </g>
    <path class="edge" d="M180,208 C150,228 120,228 100,244" marker-end="url(#arrow)"/>
    <text class="edge-label" x="118" y="230">retail</text>
    <path class="edge" d="M240,208 C290,228 320,228 320,244" marker-end="url(#arrow)"/>
    <text class="edge-label" x="300" y="230">inst</text>

    <g class="node pkg" transform="translate(30,244)">
      <rect width="140" height="40" rx="8"/>
      <text class="label" x="70" y="24" text-anchor="middle">fill_retail</text>
      <text class="tag"   x="132" y="35" text-anchor="end">[pkg]</text>
    </g>
    <g class="node pkg" transform="translate(250,244)">
      <rect width="140" height="40" rx="8"/>
      <text class="label" x="70" y="24" text-anchor="middle">fill_inst</text>
      <text class="tag"   x="132" y="35" text-anchor="end">[pkg]</text>
    </g>
  </svg>
  <figcaption>place_bid flow with retail/institutional split.</figcaption>
</figure>
```

## Legend block for the output document

Under the "How to Read These Diagrams" heading, render a small **SVG legend**
that reuses the same node classes (so the legend tracks the theme), plus a
one-line gloss per tag:

| Tag      | Meaning |
|----------|---------|
| `[PUB]`  | `public` / `entry` — transaction entry point (accent stroke, bold) |
| `[priv]` | `fun` — private module-local helper |
| `[pkg]`  | `public(package)` — intra-package helper |
| `{EXT}`  | external call — Sui framework, PAS, third-party (dashed node) |
| `{COND}` | branching predicate — diamond, labeled out-edges |
| `*EVT*`  | event emission — stadium-shaped node |

| Edge | Meaning |
|------|---------|
| solid arrow  | A calls B |
| dashed arrow | cross-module / external call |
| `Auth<Role>` near an edge | call requires the named `Auth<Role>` object |
