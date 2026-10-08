# Changelog

## [Unreleased]

- Interactive `Canvas.Session` now requires `pointer_cursor() GFX.Window.Cursor`; return `standard` for the previous arrow behavior. Existing consumers and migration documentation are updated.

- Add `Painter.clip(rect, callback)` for rectangular group clipping, nested clips and retained revisions. Rasters and ordered content preserve the boundary after child effects; mesh-only extraction rejects it. Changing a clip invalidates text pixels while retaining glyph shaping.

- The default Canvas font uses the bundled monospace face for graphemes absent from its primary face and covered by that fallback. Measurement, rasterization, fixed grids and retained geometry share the same runs; explicit fonts are unchanged. Vector placements identify each glyph face.
