---
name: publish-html
description: Use when asked to share, publish, host, or distribute a self-contained HTML file — e.g. "share this with the team", "put this on GitHub Pages", "I need a URL for this", "publish this artifact", "where can I host this?". Triggers on phrases that imply turning a local `.html` file into something with a URL. Use ONLY when an HTML file already exists on disk — does not generate HTML.
---

# Publish HTML

You turn a local self-contained `.html` file into a shareable URL. You do this **only** when explicitly invoked. You **never** publish automatically, and you **always** ask the user whether the artifact is public-safe before pushing anything to the internet.

## The Iron Law

**Never publish without an explicit `AskUserQuestion` answer about sensitivity.** Filename heuristics, content greps, and "it looks public to me" are all insufficient. The user must affirmatively say "public-safe" before anything reaches a public surface. This rule has no exceptions.

## When to use this skill

- "Share this HTML file with a colleague" / "I need a URL for this"
- "Publish this audit / report / explainer / diagram"
- "Put this on GitHub Pages" / "Host this somewhere"
- An HTML file already exists on disk; the user wants someone else to be able to open it

## When NOT to use this skill

- The HTML file doesn't exist yet → use `html-artifact` (or a specialized HTML skill) first.
- The user wants to share *content* (not specifically an HTML file) → suggest the channel that matches the content (Slack message, gist, email).
- The user is on `main` of a project repo and the artifact belongs *in* that repo as committed content (e.g. a README in HTML form) → just commit it; don't route through this skill.

## Architecture

Two routes, picked per-artifact by the user:

| Sensitivity | Destination | Visibility |
|---|---|---|
| **Public-safe** | `alilloig/artifacts` GitHub repo, `docs/<slug>/index.html`, served via GitHub Pages | Public web, indexable |
| **Sensitive** | Secret gist via `gh gist create --secret` | URL-only access (unguessable token); no search-engine indexing |
| **Abort** | Nothing happens | — |

The artifacts repo is `https://github.com/alilloig/artifacts`. Its Pages URL pattern is `https://alilloig.github.io/artifacts/<slug>/`.

The `gh` token's `gist` scope is sufficient for the sensitive path. The `repo` scope is sufficient for the public path. Confirm with `gh auth status` if either route fails.

## Step 1: Identify the target file

Accept the file path from:

1. An explicit argument the user provided in the request, OR
2. The most recently-written `.html` file in the current working directory (use `ls -t *.html 2>/dev/null | head -1`), OR
3. Ask the user via `AskUserQuestion` if neither resolves.

Confirm the file exists and is non-empty (`test -s <path>`) before going further. If it doesn't exist, surface the error — do not improvise a substitute.

## Step 2: One-shot plan check (cache)

On first use only, run `gh api user --jq '.plan.name'` and cache the result to `~/.claude/.publish-html.json`:

```json
{
  "plan": "<value or null>",
  "scopes": "<gh auth status output line for scopes>",
  "checked_at": "<ISO timestamp>"
}
```

Skip the network call on subsequent runs if the cache exists and is less than 90 days old.

Notes:

- The `plan` field may be `null` if the `gh` token lacks the `user` scope. That's expected; the public/gist architecture works regardless of plan. If the user later wants private-repo Pages (paid plans), suggest `gh auth refresh -s user` once and re-cache.
- This step is informational. It does **not** gate publishing. The Iron Law still applies — the sensitivity question is what gates publishing.

## Step 3: The mandatory sensitivity question

Always call `AskUserQuestion` with this exact decision before any side effect:

> **"Is this artifact public-safe?"**
>
> - **Public-safe** — Publish to `alilloig.github.io/artifacts/`. World-readable and indexable.
> - **Sensitive** — Publish as a secret gist. URL-only access; not indexed; revocable.
> - **Abort** — Don't publish; print the local path only.

**Optional safety-net scan (advisory, never authoritative):** before asking, you may grep the file for obvious secrets (`grep -iE 'token|secret|password|api[_-]?key|bearer|private[_-]?key|client[_-]?secret' <path>`). If any match, surface the matched lines to the user **as context** in the question — but still ask. Do not auto-route based on the grep result.

**Counter-rationalizations** (all of these mean: ASK ANYWAY):

| Excuse | Reality |
|---|---|
| "The filename clearly looks safe" | Filenames lie. Ask. |
| "The user said 'share this' which implies public" | They might mean Slack, gist, or Pages. Ask. |
| "We just generated this content from a public source" | Sources change. The artifact may have private context now. Ask. |
| "The user is in a hurry" | A privacy leak is slower than 5 seconds of asking. Ask. |
| "I checked the content — no secrets" | Sensitivity ≠ secrets. Audits, internal analysis, client work can be sensitive without containing tokens. Ask. |

## Step 4a: Public path (only if user answered "public-safe")

1. Derive a kebab-case slug from the filename (or ask the user if ambiguous).
2. Clone or update a local copy of `alilloig/artifacts`:
   - If `~/.cache/publish-html/artifacts/` exists, `git -C ~/.cache/publish-html/artifacts pull --ff-only`.
   - Otherwise, `git clone --depth 1 git@github.com:alilloig/artifacts.git ~/.cache/publish-html/artifacts`.
3. Copy the file: `mkdir -p ~/.cache/publish-html/artifacts/docs/<slug>/ && cp <path> ~/.cache/publish-html/artifacts/docs/<slug>/index.html`.
4. Commit and push:
   - `cd ~/.cache/publish-html/artifacts && git add docs/<slug>/ && git commit -m "publish: <slug>" && git push origin main`.
5. Print the URL: `https://alilloig.github.io/artifacts/<slug>/`. Note the 1–10 minute publish delay.
6. Tell the user how to **unpublish**: `git -C ~/.cache/publish-html/artifacts rm -r docs/<slug>/ && git commit -m "unpublish: <slug>" && git push`.

## Step 4b: Sensitive path (only if user answered "sensitive")

1. Run `gh gist create <path> --secret --desc "<short description from user or filename>"`.
2. Capture the printed URL from the gist command.
3. To render the HTML (not display the raw source), construct the rendered URL via `https://htmlpreview.github.io/?<raw-gist-url>` or recommend the user open the gist's "Raw" link in a browser that previews HTML. Print both options.
4. Tell the user how to **revoke**: `gh gist delete <gist-id>`.

## Step 4c: Abort path (only if user answered "abort")

Print the local file path. Suggest `open <path>` for local viewing. Do nothing else.

## Step 5: Report

Print a concise summary:

- URL (or "not published" if aborted)
- Visibility (public / secret gist / local)
- How to revoke or unpublish
- Time-to-live note: "Pages may take up to 10 minutes to refresh; gists are immediate"

Do not editorialize. Do not auto-tweet, auto-DM, auto-Slack. The user takes the URL from here.

## Common mistakes

- **Inferring sensitivity from the filename or content.** Always ask. The Iron Law has no exceptions.
- **Auto-publishing as a hook on `Write` of `*.html`.** This skill is invocation-only. There must never be a PostToolUse hook that calls it.
- **Routing the public path without a slug**. A bare push to the artifacts repo without a `docs/<slug>/` subdirectory pollutes the root and breaks the URL pattern.
- **Skipping the cache for the plan check.** Hitting `gh api user` on every invocation is annoying; the cache is cheap.
- **Forgetting the revoke instructions.** The user must know how to take it down. Always print revoke commands.
- **Treating `htmlpreview.github.io` as official infra.** It's a third-party renderer for gist HTML. Mention it as an option, not a guarantee.

## Edge cases and recovery

- **The artifacts repo doesn't exist yet.** Tell the user. Do not auto-create it; that's a separate one-time setup the user does (`gh repo create alilloig/artifacts --public` + enable Pages on `main` `/docs`).
- **The push to artifacts is rejected** (someone else pushed in the meantime). `git pull --rebase origin main` then retry. If it fails twice, surface the error and stop.
- **The gist create fails for auth reasons.** Run `gh auth status`. If the `gist` scope is missing, prompt the user to run `gh auth refresh -s gist`.
- **The file is larger than 10 MB.** Gists may reject it. Consider compressing inline images or splitting the artifact. Do not silently strip content.

## What this skill does NOT cover

- **Authoring HTML** — use `html-artifact` or a specialized skill.
- **Private GitHub Pages** (paid plan only) — out of scope for the default flow. If the user explicitly wants it and has a Pro/Team plan, they can set up a private repo with Pages on their own and route artifacts there manually.
- **Custom domains, password protection, expiry policies** — not supported. Use a real hosting platform for those needs.
- **Mass-publishing or directory-walking** — only one file at a time, always with an explicit user-confirmed sensitivity answer.
