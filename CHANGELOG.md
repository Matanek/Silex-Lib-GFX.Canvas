# Changelog

## [0.5.0] - 2026-11-09

- Clarify that TextInputTarget carries an area in logical drawing coordinates: the presenter converts it to window coordinates, as it does for the accessibility source. Viewer applies position and anchor automatically.

- Canvas.Session offers optional accessibility_source() with a null default, preserving existing conformances. Requires Silex 0.48.0 or newer, which supports protocol method bodies.

- Canvas.Session requires text_input_target(): an opaque input target and native input area, or null when the producer supplies no area. UI sessions forward it automatically; producers without text return null.

- Bundled fonts automatically resolve missing graphemes through Noto Sans Mono symbols, monochrome Noto Emoji and Noto Sans CJK SC. Fallback faces load lazily; accents, emoji sequences and metrics remain consistent across measurement, rendering and fixed grids. Explicit faces remain strict.

- Adopts one `Result` declaration: an ordinary call returns the value or
  stops the program on failure; `try call()` keeps the `Result`. Existing calls
  storing results and propagating failures are made explicit. This version
  requires Silex 0.48.0 or newer and the coordinated dependency versions.

- Interactive `Canvas.Session` now requires `pointer_cursor() GFX.Window.Cursor`; return `standard` for the previous arrow behavior. Existing consumers and migration documentation are updated.

- Add `Painter.clip(rect, callback)` for rectangular group clipping, nested clips and retained revisions. Rasters and ordered content preserve the boundary after child effects; mesh-only extraction rejects it. Changing a clip invalidates text pixels while retaining glyph shaping.

- The default Canvas font uses the bundled monospace face for graphemes absent from its primary face and covered by that fallback. Measurement, rasterization, fixed grids and retained geometry share the same runs; explicit fonts are unchanged. Vector placements identify each glyph face.
