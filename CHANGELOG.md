# Changelog

All notable changes to the `toolkit` plugin are documented in this file. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-05-19

### Added

- **Required dark / light mode support in the shared HTML conventions.** `skills/html-artifact/references/html-conventions.md` now mandates a CSS-custom-property token palette (`--bg`, `--fg`, `--fg-muted`, `--heading-fg`, `--accent`, `--border`, `--code-bg`, `--code-fg`, `--pre-bg`, `--aside-bg`, `--success`, `--warning`, `--error`) defined at `:root` with a `@media (prefers-color-scheme: dark)` override. Token values mirror Vlervcode's palette so artifacts opened inside the Vlervcode workspace browser feel visually coherent with the surrounding app chrome.
- **No-toggle, no-JS pattern.** OS preference is the single source of truth — `prefers-color-scheme` re-evaluates automatically when the user flips system appearance, so artifacts stay self-contained with zero JavaScript and zero theme-switcher UI.
- **`color-scheme: light dark` declaration** on `:root` so native browser UI (scrollbars, form controls, focus outline) renders in the matching scheme instead of flashing a light scrollbar on a dark page.
- **Inline-SVG convention update.** Strokes and text use `currentColor` (cascades to `var(--fg)`); colored fills reference tokens directly (`fill="var(--accent)"`). Hardcoded `#000` / `#fff` in SVG is now an explicit anti-pattern.
- **`skills/html-artifact/references/example-dark-light.html`** — a self-contained verification artifact exercising the full pattern (swatches, box-and-arrow diagram with `<line stroke="currentColor">` + arrowhead marker, callouts including the neutral `note` class, typography, code blocks, table). Acts as the minimum compliant example for new artifact authors and a smoke test when the convention evolves.
- **New verify-step checklist items in `skills/html-artifact/SKILL.md`** — one for token presence + `var(--…)` references, one for SVG `currentColor` discipline. Equivalent items added to `skills/for-dummies/SKILL.md` Step 5 verify checklist and `skills/move-call-chains/SKILL.md` HTML Output Conventions, so the requirement is enforced at every consumer's verify step (not only the shared reference).
- **"Print mode (opt-in)" subsection** in `html-conventions.md` with the full 13-token override block — eliminates the previous `...` ellipsis copy-paste hazard and gives print-friendly artifacts a single canonical override to drop in.
- **"Extending the palette" subsection** documenting how to add categorical/tier scales (move-call-chains visibility tiers, log levels), state-plus-emphasis pairs (`--warning-fg` when a state needs body-text contrast vs. a border accent), and diagram-only role tokens — without breaking the dark/light contract.

### Changed

- `skills/html-artifact/references/html-conventions.md` — the "pick 2–4 semantic colors max" CSS-style bullet now reads "use the token palette, never hardcode hex (one exception: `currentColor` inside inline SVG)." A new "Dark / light mode (required)" section follows the existing CSS-style rules, with the canonical CSS demo block now wiring **every** token (the original block only wired 8 of 13).
- **"Collapsibles and callouts" section** updated to name the four canonical classes — `note | success | warning | error` — matching the state tokens. Previously it named `note | warning | tip`, which contradicted the new palette.
- **Token value adjustments for WCAG AA when used as body text.** Light `--warning` deepened from `#b8862e` (3.06:1 on `--bg`) to `#9a5b00` (5.20:1). Dark `--fg-muted` lightened from `#858585` (4.32:1 on dark `--bg`, fails AA for body text) to `#9e9e9e` (6.13:1). Vlervcode keeps its original `#858585` for chrome labels where AA-for-body doesn't apply; the artifact palette now diverges where the use case differs.
- **README** — "all four skills share the conventions" corrected to "every artifact-producing skill," since `publish-html` only publishes existing HTML rather than rendering it.
- **Propagation model**: because the shared `html-conventions.md` is loaded by every artifact-producing skill (`html-artifact`, `for-dummies`, `move-call-chains`), the dark/light requirement propagates to all of them at the convention layer. The per-skill verify-step additions close the gap at the enforcement layer.
- **Distribution**: the `contract-hero` marketplace lists `toolkit` without a commit pin, so users who already have the marketplace registered receive this version on their next `/plugin update toolkit@contract-hero`. No marketplace edits required.

## [0.1.0] - 2026-05-02

### Added

- Initial plugin scaffold bundling the four self-contained HTML deliverable skills: `html-artifact`, `publish-html`, `for-dummies`, `move-call-chains`.
- Shared `skills/html-artifact/references/html-conventions.md` reference loaded by all three artifact-producing skills.
- README and LICENSE.

[Unreleased]: https://github.com/contract-hero/toolkit/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/contract-hero/toolkit/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/contract-hero/toolkit/releases/tag/v0.1.0
