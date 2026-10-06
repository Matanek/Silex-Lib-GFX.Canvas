# Changelog

## [Unreleased]

- Add `Painter.clip(rect, callback)` for rectangular group clipping, nested clips and retained revisions. Rasters and ordered content preserve the boundary after child effects; mesh-only extraction rejects it. Changing a clip invalidates text pixels while retaining glyph shaping.

- The default Canvas font uses the bundled monospace face for graphemes absent from its primary face and covered by that fallback. Measurement, rasterization, fixed grids and retained geometry share the same runs; explicit fonts are unchanged. Vector placements identify each glyph face.
