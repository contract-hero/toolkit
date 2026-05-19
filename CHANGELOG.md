# Changelog

All notable changes to the `toolkit` plugin are documented in this file. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-05-19

### Added

- **Required dark / light mode support in the shared HTML conventions.** `skills/html-artifact/references/html-conventions.md` now mandates a CSS-custom-property token palette (`--bg`, `--fg`, `--fg-muted`, `--heading-fg`, `--accent`, `--border`, `--code-bg`, `--code-fg`, `--pre-bg`, `--aside-bg`, `--success`, `--warning`, `--error`) defined at `:root` with a `@media (prefers-color-scheme: dark)` override. Token values mirror Vlervcode's palette so artifacts opened inside the Vlervcode workspace browser feel visually coherent with the surrounding app chrome.
- **No-toggle, no-JS pattern.** OS preference is the single source of truth — `prefers-color-scheme` re-evaluates automatically when the user flips system appearance, so artifacts stay self-contained with zero JavaScript and zero theme-switcher UI.
- **`color-scheme: light dark` declaration** on `:root` so native browser UI (scrollbars, form controls, focus outline) renders in the matching scheme instead of flashing a light scrollbar on a dark page.
- **Inline-SVG convention update.** Strokes and text use `currentColor` (cascades to `var(--fg)`); colored fills reference tokens directly (`fill="var(--accent)"`). Hardcoded `#000` / `#fff` in SVG is now an explicit anti-pattern.
- **`skills/html-artifact/references/example-dark-light.html`** — a self-contained verification artifact exercising the full pattern (swatches, callouts, typography, code blocks, table, inline SVG with `currentColor`). Acts as the minimum compliant example for new artifact authors and a smoke test when the convention evolves.
- **New verify-step checklist item in `skills/html-artifact/SKILL.md`** asserting tokens are present, component rules use `var(--…)`, no hardcoded hex values, and no hardcoded SVG colors.

### Changed

- `skills/html-artifact/references/html-conventions.md` — the "pick 2–4 semantic colors max" bullet is replaced by "use the token palette, never hardcode hex." The CSS-style section keeps its existing rules (system fonts, max-width, single mobile breakpoint, conservative aesthetic) and adds the new "Dark / light mode (required)" section beneath it.

### Distribution

- Because the shared `html-conventions.md` is loaded by every artifact-producing skill (`html-artifact`, `for-dummies`, `move-call-chains`), the dark/light requirement automatically propagates to all of them — no per-skill edits needed.
- The `contract-hero` marketplace lists `toolkit` without a commit pin, so users who already have the marketplace registered receive this version on their next `/plugin update toolkit@contract-hero`.

## [0.1.0] - 2026-05-02

### Added

- Initial plugin scaffold bundling the four self-contained HTML deliverable skills: `html-artifact`, `publish-html`, `for-dummies`, `move-call-chains`.
- Shared `skills/html-artifact/references/html-conventions.md` reference loaded by all three artifact-producing skills.
- README and LICENSE.
