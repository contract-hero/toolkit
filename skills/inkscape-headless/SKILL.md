---
name: inkscape-headless
description: Use when working with SVG, PDF, or AI files through the inkscape MCP
  server headlessly (open_document / place_document / export_*), and especially when
  open_document rejects a file with "path rejected: outside workspace" or "input
  file exceeds the configured size limit", the source is a PDF or Illustrator (.ai)
  file rather than SVG, or the job is print-oriented (CMYK color, bleed, 100MB+
  embedded rasters). Covers getting oversized or non-SVG files in and out of the
  server's sandbox. Do NOT load for live-GUI editing of an open Inkscape window —
  that is inkscape-live's protocol.
---

# Inkscape MCP Headless: Sandbox, Big Files, Print PDFs

Empirically validated on Inkscape 1.4.4 / inkscape-mcp-server, macOS (2026-07),
with a 220MB Illustrator print PDF. The server's `how_do_i` covers routine tool
choice; this skill covers what discovery cannot tell you.

## Sandbox and limits (this machine)

- Workspace root: `~/workspace/inkscape-workspace` — the ONLY tree `open_document`,
  `save_document_as`, and `out_dir` accept. Not discoverable via `list_capabilities`;
  it is `INKSCAPE_MCP_WORKSPACE_ROOTS` on the registration in `~/.claude.json`.
- Limits (env-overridable, server restart required): `INKSCAPE_MCP_MAX_INPUT_BYTES`
  50MB on open/reload, `INKSCAPE_MCP_MAX_OUTPUT_BYTES` 100MB on every export,
  `INKSCAPE_MCP_MAX_EXPORT_PX` 8192 per raster side.
- `list_capabilities` returns ~58KB — query the saved JSON with jq, don't re-call.

## Getting files in

- `open_document` takes **SVG only**. Convert PDF/AI first with the CLI:
  `inkscape file.pdf --export-type=svg --export-filename=$ROOT/work.svg`
  (`--pdf-page` no longer exists in 1.4; single-page files need no page flag).
  Run it in the background — a 200MB PDF takes minutes.
- Copy source assets into the root; absolute paths outside it are rejected.

## Oversized SVG workaround (embedded raster)

A PDF import with a big image lands as one base64 data-URI — 33% larger than the
binary and usually over the 50MB open cap. Fix: extract the base64 payload to a
PNG file inside the workspace root and rewrite the `<image>` href to link it.
The SVG drops to KBs and opens fine.

- Use an **absolute** `file:///` href: working copies live in `.inkscape-mcp/`,
  so relative hrefs break at render time.
- Opened documents may reference local files (the XML safety layer only blocks
  DTD/XXE); `insert_svg_fragment`/`set_document_svg` scrub external refs — this
  workaround only works via `open_document`.
- Exports re-embed the linked image, so output PDFs are self-contained.

## Print/PDF gotchas

- Inkscape imports the PDF **crop box** as the canvas (bleed seemingly lost), but
  the `<page>` element keeps the media box — exports restore the full page with
  bleed. Don't hand-resize the canvas to compensate.
- Inkscape is RGB-internal: CMYK artwork exports as RGB (often 25% smaller —
  handy under the output cap). When the color space must survive (print shop
  asked for CMYK), compose/preview in the MCP but stamp the final overlay onto
  the original PDF with pypdf (`merge_transformed_page`); raise
  `pypdf.filters.MAX_DECLARED_STREAM_LENGTH` for >100MB streams. The stamped
  overlay is still device-RGB objects inside the CMYK page — fine for most
  large-format RIPs, but a strict prepress shop may want dark overlay elements
  re-authored as 100% K.
- Illustrator-compatible PDFs carry `/PieceInfo → /Illustrator` on the page: a
  private copy of the artwork that Illustrator prefers over the visible page.
  After stamping edits with pypdf, **delete the page's `/PieceInfo`** (and stale
  `/Thumb`) or Illustrator shows the pre-edit artwork. Inkscape exports don't
  have this problem. A PDF-compatible file renamed `.ai` is what print shops
  accept as "the .ai".
- `export_print_profile` pins PDF 1.4 and outlines text. `is_vector: false` on
  an image-dominant page is normal — verify with `pdfimages -list` (expect the
  original pixel dimensions, no unexpected extra rasters).

## Working on placed content

- Source elements without XML ids can't be targeted after `place_document`
  (deep-copies re-mint ids, id-less nodes stay unaddressable). To change a
  placed asset: `restore_snapshot` to the pre-place snapshot (in the
  `place_document` result), edit the source SVG, re-place at the same coords.
- `find_objects(accurate_bbox=true)` gives engine-true boxes for paths/groups —
  anchor placement math on those, not on raster measurements.

## Verify before shipping

Render a low-res proof (`pdftoppm -png -r 12`) and a full-res crop of the edited
region (`-r 72 -x … -W …`); check `pdfimages -list` for resolution parity. For
QR codes: OpenCV's detector fails on dot-style modules — threshold, dilate the
dots into solid squares, then decode (invert first for light-on-dark codes).
