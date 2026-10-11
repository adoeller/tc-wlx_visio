# wlx_visio — Visio VSDX Viewer for Total Commander

A Total Commander Lister (WLX) plugin that renders `.vsdx` (Visio 2013+)
files directly — shapes, master/stencil inheritance, connectors, text,
groups, and layers — in both the F3 file viewer and the Ctrl+Q Quickview
panel. Read-only; no editing, no thumbnails.

Written in Object Pascal (Lazarus/FPC), built as a native Win32/Win64 DLL
with no LCL dependency (see [Architecture](#architecture) below).

## Status

All planned milestones for `.vsdx` support are implemented and verified
against a corpus of real `.vsdx` test files (see [Test data](#test-data)):

- ZIP/XML parsing, master (stencil) inheritance, formula evaluation for
  ShapeSheet cells (`PinX`, `Width`, `BeginX`, …)
- Geometry includes circular/elliptical arcs, ellipses, relative quadratic and
  cubic Beziers, polylines, rational NURBS, splines and clipped infinite lines.
  Groups, embedded PNG/JPEG/BMP/GIF/TIFF and rasterized EMF/WMF images,
  including images inherited from master shapes, are supported.
- Rich text supports character/paragraph runs, inherited formatting, Unicode,
  alignment, wrapping, indents, spacing, tab stops, justification, font scaling
  and superscript/subscript. GDI supplies font metrics and rasterization.
- Connectors with all 45 built-in arrowheads and seven arrow sizes; all built-in
  line patterns (0–23), transparency and round/flat/extended caps.
- Built-in fill patterns (0–40), multi-stop linear/radial/rectangular/path
  gradients, theme gradient stops, rotation and group gradient coordinates.
- Simple/oblique shadows with offset, pin-based scale, transparency, blur and
  group/sub-shape shadow behavior. Shadow geometry is included in fit/export.
- Foreign images are cropped to their border or referenced Geometry path;
  compound clipping paths retain holes under rotation and mirroring.
- Layers (visibility), background pages (rendered as the bottom layer of the
  pages that reference them via `BackPage`), multi-page navigation as a tab
  strip, zoom/pan, a self-drawn toolbar
- Files that are already open in another program (e.g. Visio itself) can
  still be opened read-only — the ZIP container is read with a
  deny-none share mode instead of the stricter default
- StyleSheet resolution follows shape, master and document style chains.
  DrawingML themes and Visio quick styles supply colors, fonts, line widths,
  transparency and variation matrices. ShapeSheet formulas include cross-shape,
  page/document and geometry-row references and rich text measurements.
- Geometry, images and text share one ordered scene in the viewer and PNG export.
- Context menu (right-click): Save as PNG, Copy Image (to clipboard), Copy
  Text; click-to-select shape text with Ctrl+click for multi-select, "Copy
  Text" respects the selection
- `ListPrint`/`ListPrintW`: shows the standard Windows print dialog and
  prints the current page, scaled to fit the printable area
- Background loading thread so large files don't freeze Total Commander's
  UI thread
- UI localization: English, German, Russian, Ukrainian (see
  [Configuration](#configuration))

**Not yet verified against a real Total Commander instance:** the F3-vs-
Quickview heuristic, whether TC forwards digit-key presses to the plugin as
expected, and CPU behavior in TC's idle loop. These were validated through
headless test drivers (see [Development / testing tools](#development--testing-tools))
that simulate window messages, not through an actual TC session.

**Not supported:** the legacy binary `.vsd` format (Visio 2010 and earlier).
This was scoped as a possible follow-up; see the note in
[Known limitations](#known-limitations).

## Known limitations

These are deliberate, documented simplifications — not oversights:

- **Curves:** curves are sampled into polylines; extreme zoom can reveal the
  finite sampling resolution. Regression tests cover analytic curve positions.
- **Custom stencil effects:** user-defined `USE()` marker/line/fill-pattern
  masters are not evaluated. Built-in marker and pattern enumerations are
  supported. Full 3-D effects, glow and the full ShapeSheet function catalog
  remain outside the five implemented P2 areas.
- **Raster precision:** marker contours, hatches, curves and path-gradient
  distance fields are rasterized/sampled; exact Visio pixels at arbitrary zoom
  are not guaranteed. These are tested against specification-driven geometry
  and pixel assertions; the user's original drawing was unavailable.
- **Text formatting:** advanced bullets, vertical writing and small caps are
  not reproduced. Cached field text is rendered; field expressions are not
  recalculated as a general Visio field engine.
- **Metafiles:** EMF/WMF images are rasterized through Windows GDI with a
  white background and a maximum of 1600 pixels on their longer side.
- **Filenames with non-Latin characters** (e.g. Cyrillic) are not verified
  to round-trip correctly end-to-end; the file's *content* (shape text, page
  names, search) is fully Unicode-safe, but the file *path* itself still
  passes through `AnsiString` in a few places.
- **F3 vs. Quickview** is detected via a size heuristic on `ParentWin`
  (no WLX API distinguishes the two contexts) — see the note above.
- **`ListPrint`'s GDI drawing sequence** (page scaling, `StretchDIBits`, …) was
  verified end-to-end against a real installed printer driver by bypassing
  the native print dialog (which can't be driven headlessly) — every API call
  in the sequence succeeded, but the actual printed/PDF output page itself
  could not be located afterward to inspect pixel-for-pixel in this
  environment.

## Requirements

- Total Commander (32- or 64-bit)
- Windows

No external DLLs to install — BGRABitmap is vendored directly into the
project (see [Third-party components](#third-party-components-and-licenses)).

## Installation

Copy `Visio.wlx` (32-bit) and/or `Visio.wlx64` (64-bit) into a plugin
folder and register them via Total Commander's plugin manager
(*Configuration → Options → Plugins → Lister (WLX) plugins → Configure
plugins… → New type…*), or install `pluginst.inf` directly by dragging it
onto Total Commander.

## Usage

- **F3** or **Ctrl+Q** on a `.vsdx` file opens the viewer.
- **Mouse wheel:** zoom in/out, centered on the cursor.
- **Left-click** on a shape's text: select it (highlighted). **Ctrl+click**
  adds/removes a shape from the selection. Click on empty space clears the
  selection. **Left-click drag** on empty space pans the view (unchanged).
- **Toolbar** (top strip): `−` / `+` zoom, `Fit` to window.
- **Page tabs** (second strip, shown only for multi-page files): click a tab
  to jump to that page. If there are more tabs than fit the window width,
  scroll the mouse wheel while hovering over the tab strip.
- **Page Up / Page Down:** switch pages (same as clicking a tab).
- **+ / -:** zoom in/out from the keyboard.
- **Right-click:** context menu with **Save as PNG…**, **Copy Image** (to the
  clipboard), and **Copy Text**. Save as PNG/Copy Image always act on the
  *entire* current page (not just the visible viewport, rendered fresh at the
  current zoom level, capped at 4000px on the longer side). Copy Text copies
  only the selected shapes' text if any are selected, otherwise the whole
  page's text (same as `Ctrl+C` in Total Commander's Lister, which also
  respects the selection).
- Total Commander's own **Select All** Lister command now selects every
  text shape on the page (previously a no-op, since there was no selection
  concept) — its own **Copy** then copies everything, same as before.
- Digit keys `0`–`9` are intentionally **not** intercepted by the plugin, so
  Total Commander can still use them for its own Lister-mode shortcuts if it
  does (see [Known limitations](#known-limitations)).

## Configuration

An optional `wlx_visio.ini` next to the plugin DLL(s) controls:

```ini
[Debug]
Log=1

[Language]
Code=de
```

- `[Debug] Log=1` enables a timestamped trace log (`wlx_visio_debug.log`,
  next to the DLL or in `%TEMP%` as a fallback) — off by default.
- `[Language] Code=` overrides UI language detection. Accepts `en`, `de`,
  `ru`, `uk`. Without this setting, the language is auto-detected from
  Windows' UI language, falling back to English.

(This only affects the plugin's own UI chrome — toolbar label, loading/error
messages. Diagram content is never translated.)

## Architecture

- Plain Win32 child window (`CreateWindowEx`), **no LCL** (`TForm`, `Forms`,
  `Controls`, …) anywhere in the plugin — a documented failure mode in this
  codebase's other Lazarus plugins is LCL idle-loop overhead spinning up
  inside the TC host process even without any visible form.
- Rendering via [BGRABitmap](https://github.com/bgrabitmap/bgrabitmap),
  built against its `BGRABitmapPack4NoLCL` variant (`-dBGRABITMAP_DONT_USE_LCL
  -dBGRABITMAP_DONT_USE_LAZFREETYPE`) for the same reason.
- Text is drawn via a raw GDI round-trip (`uvisio_gditext.pas`) — create a
  temporary 32bpp DIB, `DrawTextW` into it, copy back — rather than a
  BGRABitmap font renderer (which isn't available in the NoLCL/NoFreeType
  build anyway).
- Toolbar and page controls are **hand-drawn rectangles with click
  hit-testing**, not native `BUTTON` controls — this sidesteps the dark-mode
  subclassing work that native Win32 buttons would otherwise require.
- File loading and page switches run on a background `TThread`
  (`uVisioLoadThread.pas`); results are marshaled back to the UI thread via a
  posted window message, matching the threading model Total Commander
  expects from WLX plugins (all callbacks arrive on TC's own UI thread).

## Building from source

Requires Lazarus/FPC (developed against FPC 3.2.2). `lazbuild` is not
normally on `PATH`; the build script below expects to find it under
`%USERPROFILE%\lazarus\lazbuild.exe`.

```bash
build.ps1
```

This builds both the 32-bit (`Visio.wlx`) and 64-bit (`Visio.wlx64`)
targets. (`lazbuild`/`fpc` always append `.dll` to library targets on
Windows regardless of the configured output name — the script renames the
result afterward; this is a `lazbuild` quirk, not a project setting you
need to touch.)

## Test data

`testdata/` contains small `.vsdx` fixtures from
[dave-howard/vsdx](https://github.com/dave-howard/vsdx) (MIT-licensed) —
that project's own unit-test files, each targeting a specific feature
(master inheritance, connectors, groups, rotation, colors, …), which made
them a good match for milestone-by-milestone verification during
development. They are not related to this plugin beyond serving as test
input.

## Development / testing tools

Three command-line probes (not part of the shipped plugin) were used during
development to verify behavior without a running Total Commander instance:

- `vsdx_probe.exe <file.vsdx>` — dumps the parsed page/shape tree as text.
- `vsdx_render_probe.exe <file.vsdx> <out.png> [W] [H]` — renders a page
  through the exact same pipeline the plugin uses, to a PNG file.
- `wlx_ui_probe.exe <file.vsdx> <out-prefix>` — creates a real instance of
  the plugin's window in a plain host window and drives it with synthesized
  mouse/keyboard messages (page switching, zoom, fit), screenshotting each
  step. This is how the background-thread and toolbar logic were verified
  without touching Total Commander itself.

None of these are required for normal use of the plugin.

`tools/run_render_tests.ps1` generates `testdata/render_regression.vsdx`
and runs assertions for partial master geometry overrides/deletions,
instance-relative formulas, relative coordinates, mirrored arrows,
font IDs, document text styles, palette colors, master/page images,
EMF/WMF decoding and image/geometry drawing order. It also runs all
existing VSDX fixtures through the parser and render-data builder.
The PNG result is saved as `build/render-regression.png`.

Rendering fixes were checked against Microsoft's
[VSDX reference](https://learn.microsoft.com/en-us/office/client-developer/visio/visio-file-format-reference),
[Apache POI geometry inheritance](https://github.com/apache/poi/blob/trunk/poi-ooxml/src/main/java/org/apache/poi/xdgf/usermodel/section/GeometrySection.java)
and [libvisio transformation and foreign-data handling](https://github.com/LibreOffice/libvisio/blob/master/src/lib/VSDContentCollector.cpp).

## Third-party components and licenses

- **[BGRABitmap](https://github.com/bgrabitmap/bgrabitmap)** —
  vendored under `vendor/bgrabitmap/` (source only, no compiled `lib/`
  output). Licensed under the modified LGPL with a static-linking exception
  (see `vendor/bgrabitmap/COPYING.modifiedLGPL.txt`), which permits static
  linking into this closed-source-style plugin DLL.
- **fcl-xml** (`DOM`, `XMLRead`) and **paszlib** (`Zipper`) — part of the
  standard Free Pascal RTL, no additional licensing concerns.
- Test fixtures under `testdata/` — see [Test data](#test-data) above.

## P1 regression suite

Run `./tools/run_p1_tests.ps1`, optionally with `-Win32` or `-Heap`.
The fixture is generated locally and tests rich text, themes, geometry,
formulas and scene ordering. See [P1_TEST_REPORT.md](P1_TEST_REPORT.md)
for validation results and specification references.

## P2 regression suite

Run `./tools/run_p2_tests.ps1`, optionally with `-Win32` or `-Heap`.
The suite generates its own VSDX and preview galleries; no private drawing is
required. See [P2_TEST_REPORT.md](P2_TEST_REPORT.md) for scope, test results and
format references.
