---
name: teleprompter
description: |
  Turns any text — a Markdown file, a speech draft, deck speaker notes, pasted
  prose — into a self-contained HTML teleprompter for timing and delivering a
  live talk. Use when the user says "make a teleprompter", "turn this into a
  teleprompter", "I need to rehearse/time this speech", "prompt me through these
  speaker notes", or wants to practice reading a script at a target duration.
  Segments the text into spoken beats, estimates each beat's duration from word
  count at an adjustable speaking rate (WPM), and emits one double-click-openable
  `.html` file with a time-synced autoscroll (each line crosses the eye-line
  arrow in exactly its spoken slot), a beat-by-beat countdown mode, a "fit to a
  target time" control that back-solves the rate, mirror + fullscreen for real
  prompter rigs, and a running clock. Output is a single self-contained HTML
  file (inline CSS/JS, no network, no external assets).
allowed-tools: Read, Write, Glob, Grep, Bash
author: alilloig
version: 1.0.0
date: 2026-07-18
---

# Teleprompter

You turn a piece of writing into a **single self-contained `.html` teleprompter**
the user can open in a browser and read from while giving a talk. The hard part
is done by the bundled engine (`template.html`); your job is to segment the
source text into well-paced **beats** and inject them.

**REQUIRED FILE:** `template.html` (next to this SKILL.md). It is the tested
prompter engine — time-synced autoscroll, beat mode, WPM estimation, fit-to-time,
mirror/fullscreen. Do **not** rewrite it. You only replace two placeholders.

---

## The timing model (read this first)

There is no recorded audio, so a beat's duration is **estimated**:

```
spoken seconds  = words(text) / wpm * 60
beat duration   = spoken seconds + hold        (a normal beat)
beat duration   = hold                         (a silent `pause` beat)
```

- **WPM = the speaking rate**, adjustable live in the UI (default 130 — a
  deliberate public-speaking pace; conversational is ~150, slow/keynote ~120).
  A measured speaking rate already includes natural micro-pauses, so the engine
  adds **no** artificial per-line breath. Do not try to pad durations yourself.
- **`hold`** = extra seconds of deliberate silence *after* a beat (an emphasis
  pause, "let that land"). Optional, usually 0.
- **`pause` beats** = a planned silent gap (breath, dramatic beat, "wait for the
  slide"). Their `text` is a **stage direction**, not spoken, and shows dimmed.
- **Fit-to-time**: the user can type a target like `6:30` and the engine
  back-solves WPM. So you do **not** need to hit a target duration during
  segmentation — just pick a sensible default WPM and segment cleanly.

The engine recomputes the whole autoscroll schedule whenever WPM, holds, or font
size change, and keeps each beat's block crossing the eye-line for exactly its
estimated duration. Get the **segmentation and word content** right; the timing
follows.

---

## Step 1 — Get the source text

Resolve the input in this order:

1. **A path the user gave** (`.md`, `.txt`, `.markdown`, a script file) — `Read` it.
2. **Pasted text in the message** — use it directly.
3. **Deck speaker notes** — if pointed at a slide deck:
   - **Marp / Markdown decks**: speaker notes are HTML comments (`<!-- ... -->`)
     under each slide, and slides are split by `---`. Extract the comment bodies
     in order; if a deck has no notes, fall back to the visible slide prose the
     user wants spoken (ask which).
   - **`.pptx`**: notes live in `ppt/notesSlides/noteSlideN.xml` inside the zip.
     `unzip -o deck.pptx -d /tmp/deck` then read the `<a:t>` text runs from each
     `notesSlideN.xml` in slide order. If extraction is messy, show the user what
     you got and confirm before building.
   - **`.pdf` / other**: ask the user to paste the notes, or point you at a text
     source — don't guess spoken content from slide bullets alone.

If the input is ambiguous (which file? notes vs. body text?), ask **before**
building — a wrong source wastes the whole artifact.

---

## Step 2 — Segment into beats

A **beat** is one natural spoken unit that crosses the eye-line as a chunk.
Aim for a rhythm a speaker can follow: **one sentence, or a short tightly-linked
pair of sentences, per beat.** Guidelines:

- **Split on sentence boundaries.** Long sentences (>~30 words) with a natural
  clause break can split into two beats at the comma/semicolon/dash.
- **One bullet → one beat** for note-style source. Merge tiny fragments
  ("Thanks!", "Right.") into the neighbouring beat rather than making a 2-word beat.
- **Clean to *spoken* text.** Strip Markdown syntax (`#`, `*`, `` ` ``, link
  brackets → link text, list markers), expand abbreviations only if the speaker
  would say them out loud, drop anything not meant to be voiced (slide URLs,
  citations, `[click]` cues → turn those into `pause`/`hold`, not spoken words).
- **Headings / slide titles → `label`,** not spoken text. A `## Section` becomes
  the `label` on the first beat of that section (shown small + dimmed above the
  line, and used in the pacing HUD). Don't emit a spoken beat for a bare heading.
- **Explicit pause cues** in the source — `[pause]`, `(beat)`, `…` on its own,
  "— long pause —", "wait for laugh" — become a `pause` beat with a `hold`
  (default 2s; longer if the cue implies it) and a short stage-direction `text`
  like `( pause — let it land )`.
- **Keep the author's words.** Do not paraphrase, summarise, improve, or
  translate the speech. Fix only obvious Markdown artifacts. If the user asks you
  to also edit the writing, do that as a separate, explicit step.

Each beat is an object:

```js
{ label: "Opening",  text: "Good morning — thanks for having me." }
{ text: "Last year we shipped in twelve countries." }
{ pause: true, hold: 2.5, text: "( pause — let the number land )" }
```

Only include `label` on the first beat of a section. Only include `hold`/`pause`
when the source calls for it.

---

## Step 3 — Build the DECK and inject it

Assemble one object:

```js
const DECK = {
  title: "Keynote — Portable Memory",   // short talk title for the tab + header
  wpm: 130,                             // default rate; user tunes it live
  beats: [ /* the beats from Step 2, in order */ ],
};
```

Choose `wpm`:
- Default **130**. Use **120** if the user says "slow / deliberate / big room",
  **150** if "conversational / fast / casual".
- If the user gives a target duration *and* you can, set `wpm` so the estimate
  lands there: `wpm = totalSpokenWords * 60 / (targetSeconds − totalHold)`. The
  user can still refine with the "fit" box, so an approximate default is fine.

Then produce the output file from the template:

1. `Read` `template.html` from this skill directory.
2. Replace **`/*{{DECK}}*/`** with the pretty-printed `DECK` object literal
   (valid JS — the beats' `text` must be properly quoted/escaped). **Before
   substituting, replace every `</` in that serialized literal with `<\/`** — this
   neutralizes any stray `</script>` in beat text that would otherwise close the
   script tag at HTML-parse time (verified necessary; escaping alone can't fix it
   because it happens before the JS runs). The engine HTML-escapes `<`/`>`/`&` at
   render time, so you do not need to touch those.
3. Replace **both** `/*{{TITLE}}*/` occurrences (the `<title>` tag and the header
   `.title` span) with the plain-text title.
4. `Write` the result to **`<slug>-teleprompter.html`** in the user's working
   directory (slug = kebab-case of the title, e.g. `portable-memory-teleprompter.html`).
   If the source was a file, prefer `<source-basename>-teleprompter.html`.

**Escaping matters.** The beats become a JS array literal inside a `<script>`
tag — two layers to get right:
1. **Valid JS string**: escape backslashes and whichever quote you delimit with.
2. **The `</script>` hazard**: a literal `</script>` anywhere in a `text`/`label`
   ends the script tag early. Split it (e.g. `<\/script>`). Switching to JSON does
   NOT save you here — the browser scans for `</script>` before any JS parsing.

Angle brackets and ampersands in the *spoken* text are safe: the engine
HTML-escapes `text` and `label` (via its `esc()` helper) at every render site, so
`<`, `>`, `&` display literally and cannot inject markup. When in doubt, build the
DECK as JSON (valid JS here) — just still guard `</script>`.

---

## Step 4 — Report and hand off

After writing, tell the user:

- The **estimated runtime** at the default WPM and the **beat count**
  (e.g. "42 beats · ~6:10 at 130 wpm").
- The **controls they'll actually use**: `space` play/pause, `↑`/`↓` nudge WPM,
  `b` toggles beat-by-beat mode, `Mirror` for a beam-splitter rig, `Fullscreen`,
  and the **fit** box to lock the talk to a target time.
- That durations are **estimates** — one rehearsal read tells them the real rate;
  they adjust WPM (or type their measured time into "fit") and the scroll re-times.

Then surface the file. Per the user's global preference, end with a clickable
Vlervcode deep-link to the `.html` (form
`[<filename>](vlerv://open?path=<abs path>)`, path percent-encoded) so they can
open it immediately, and offer to tweak WPM, split/merge beats, or add pause cues.

Do not auto-publish it anywhere (that's `publish-html`, only on request).

---

## Notes & edge cases

- **Very long talks** (hundreds of beats) are fine — the engine handles arbitrary
  length; the schedule is O(beats). Keep beats sentence-sized regardless.
- **No headings in the source**: omit `label`s entirely; the prompter reads as one
  continuous flow, which is correct for a plain speech.
- **Bilingual / non-English source**: keep the source language verbatim; the
  word-count model works for any space-delimited language. For scripts without
  spaces (e.g. Chinese/Japanese) word count is a poor proxy: the "fit" box fixes
  the TOTAL runtime, but per-beat durations stay proportional to word count, so
  individual beats can cross the eye-line too fast or slow even when the total is
  right. For heavy CJK use, split beats into more even chunks (or add explicit
  `hold`s) and treat the scroll as a guide rather than exact per-line timing.
- **The user wants to re-time an existing prompter**: they just change WPM or the
  "fit" box in the browser — no rebuild needed. Only rebuild when the *text* or
  *segmentation* changes.
