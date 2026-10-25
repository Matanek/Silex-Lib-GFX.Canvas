# Draw with GFX.Canvas

`GFX.Canvas` describes 2D drawing intentions without imposing a renderer. A
`Canvas` records paths, analytic shapes, fills, strokes, text, and groups with
effects, then turns them into retained content or an RGBA image.

[Lire cette documentation en français.](../FR/README.md)

## Install the package

```text
silex install GFX.Canvas
```

GFX.Canvas requires Silex 0.43.0 or newer.

## Record and rasterize a drawing

```sx
use GFX.Canvas
use GFX.Color
use STD.Math

var drawing = Canvas()
drawing.paint(func(painter:&Canvas.Painter) {
    painter.fill(
        Canvas.Rect(Math.Vec2(), Math.Vec2(128.0, 48.0), 8.0),
        Canvas.Fill.solid(Color.indigo_600()),
    )
    painter.text(
        "Silex",
        Math.Vec2(12.0, 10.0),
        Canvas.TextStyle(size:18.0, color:Color.white()),
    )
})

let image = drawing.rasterize(128, 48)
```

`GFX.Font` owns the distributed proportional and monospace faces;
`Canvas.Font.default()` and `Canvas.Font.monospace()` remain compatibility
facades. An application can also supply a path, encoded bytes, or wrap an
already validated `GFX.Font.Face` with `Canvas.Font(face)`. Canvas consumes
GFX.Font runs and hinted coverage without exposing a native handle.

The default facade keeps Noto Sans for covered text. For a grapheme absent from
that face and fully covered by bundled Noto Sans Mono, Canvas uses the fallback
face. Graphemes stay whole and adjacent runs using the same face stay grouped.
Measurement, rasterization, fixed grids and retained layers share these positions
and the primary font baseline. `face()` identifies the primary face; explicitly
constructed fonts do not add a fallback. Characters absent from both faces remain
missing: this does not promise universal Unicode coverage or mixed bidi composition.

`Painter.text(..., clip:Canvas.Rect(...))` attaches an axis-aligned rectangular
clip to that text command. Raster and retained renderers can therefore keep an
overscan line alive until its last visible pixel while preventing it from
bleeding into adjacent controls. The clip is placement state, not font or
shaping-cache state.

## Fill a shape with an image

`ImagePaint` retains an independent image with a stable identity and revision.
Mapping remains separate from the contour: an analytic rectangle therefore
keeps its six vertices and distance coordinates while a renderer also receives
the image source and its logical coordinate system.

```sx
use GFX.Assets.Image
use GFX.Canvas
use STD.Math

var portrait = Canvas.ImagePaint(Image.rgba8(2, 1, [
    255, 96, 64, 255,
    64, 128, 255, 255
]))
let frame = Canvas.Rect(Math.Vec2(8.0), Math.Vec2(96.0, 64.0))

var drawing = Canvas()
drawing.paint(func(painter:&Canvas.Painter) {
    painter.fill(
        frame,
        Canvas.Fill.image(
            portrait,
            Canvas.ImageMapping.fit(
                frame:frame,
                mode:Canvas.ImageFit.cover,
                alignment:Math.Vec2(0.5),
                sampling:Canvas.ImageSampling.smooth
            )
        )
    )
})
```

`stretch` maps source and destination edges, `contain` preserves the whole
image with transparent bars, and `cover` fills the frame by cropping it.
`ImageMapping.tile(origin, texel_size, ...)` instead repeats the selected
source region on both axes, including before a negative origin. Coordinates
follow Canvas and images: top-left origin with Y down. An optional
`Image.Region` selects a half-open pixel rectangle and is validated against the
current image before preparation or rasterization.

`portrait.replace(next_image)` preserves identity, advances the revision, and
invalidates preparations that use it. A region that no longer fits fails
explicitly on the next use. `Canvas.rasterize`, `Surface.draw`, and Scene2D use
the same mapping. For an alternative renderer, `Prepared.content()` exposes an
`image` value containing the mask mesh, `ImagePaint`, and `ImageMapping`. Such
a fill makes `vector_geometry_compatible()` false because a mesh alone cannot
carry the image source without repurposing its analytic coordinates.

## Isolate a group and apply effects

`Painter.layer` groups several intentions before applying a shadow, blur, or
opacity. The shadow below is produced from the shape and text together: the
text remains shaped by `GFX.Font` and retains its `GlyphRun`, vector outlines,
and hinted coverage.

```sx
use GFX.Canvas
use GFX.Color
use STD.Math

var card = Canvas()
card.paint(func(painter:&Canvas.Painter) {
    painter.layer(
        [Canvas.Effect.shadow(
            offset:Math.Vec2(0.0, 6.0),
            radius:12.0,
            color:Color.black().with_alpha(0.25)
        )],
        func(layer:&Canvas.Painter) {
            layer.fill(
                Canvas.Rect(Math.Vec2(8.0), Math.Vec2(160.0, 56.0), 10.0),
                Canvas.Fill.solid(Color.slate_800())
            )
            layer.text(
                "Project",
                Math.Vec2(24.0, 18.0),
                Canvas.TextStyle(size:20.0, color:Color.white())
            )
        }
    )
})

let image = card.rasterize(184, 80)
```

Effects in one layer are evaluated from left to right. For example,
`[blur, shadow, opacity]` first blurs the group, computes the shadow from its
blurred alpha, then modulates both content and shadow. Nested layers are fully
evaluated from the inside out. Blur and composition use premultiplied alpha so
transparent edges do not acquire stray colors.

A layer's public bounds include its shapes, text ink, and effect expansion. A
framed rasterization clips only the final root result. Inner shadows, artistic
blend modes, and arbitrary filters are outside this surface.


## Clip a group

`Painter.clip(rect, callback)` retains only the part of the group inside that
logical rectangle, after fully evaluating its nested layers and effects. The
callback keeps Canvas coordinates; it does not introduce a translation.

This fragment assumes an existing Canvas named `drawing` and imports for
`GFX.Canvas`, `GFX.Color` and `STD.Math`.

```sx
drawing.paint(func(painter:&Canvas.Painter) {
    painter.clip(Canvas.Rect(Math.Vec2(8.0), Math.Vec2(120.0, 60.0)),
        func(clipped:&Canvas.Painter) {
            clipped.fill(Canvas.Circle(Math.Vec2(80.0), 64.0), Canvas.Fill.solid(Color.cyan_500()))
            clipped.text("Silex", Math.Vec2(8.0), Canvas.TextStyle(size:20.0))
        })
})
```

The rectangle must be finite, non-rounded and have non-negative dimensions.
Zero hides everything; nested clips intersect. The raster excludes pixels
before `ceil(minimum)` and from `ceil(maximum)` onward. Following callbacks
recover their original context. `Layer.clip()` and `LayerContent.clip()` expose
the limit; bounds include effects before intersecting with it.

A mesh alone cannot represent clipping: `vector_geometry_compatible()` returns
false and geometry-only access reports an error. Renderers consume
`Snapshot.content()` to preserve order, layers, clips and effects. Detached text
layers and glyphs describe their sources; they do not on their own represent
parent-group clips or effects. Framed/unframed rasters, `Surface`, and the
Scene2D compositor respect these groups. Changing a group clip invalidates
cached text pixels while preserving unchanged glyph shaping.

## Reuse drawing intentions

`Canvas.commands()` returns a faithful copy of public commands, including
nested layers, and `Path.steps()` exposes segments. An extension can therefore
interpret intentions without knowing about renderer caches.

```sx
for command in drawing.commands() {
    match command {
        fill_rect(value) => { print("rectangle") }
        text(value) => { print(value.value) }
        layer(value) => { print("layer $(value.id()), revision $(value.revision())") }
        else => {}
    }
}
```

`Canvas.vector_geometry_compatible()` reports whether the drawing can be
faithfully described by a mesh alone. `Canvas.vector_geometry()` produces that
mesh when the answer is true and explicitly refuses a Canvas whose effects
require raster composition; it never silently drops an effect.

`Canvas.snapshot()` preserves geometry in its local coordinates without
inventing a drawing rectangle. Every text layer computes its own coverage,
including when its origin is negative. `Canvas.snapshot(width, height)` remains
available when a viewport, layout, or export must clip text to a precise frame.

For successive revisions, pass the previous snapshot as `reuse:previous`.
Identical text commands retain their revision and glyph coverage while vector
geometry can change independently.
`Canvas.id()` remains stable for the drawing's lifetime; `Snapshot.id()` names
one retained compilation, while `Snapshot.canvas_id()` preserves the source
Canvas identity across revisions. A cache can therefore replace an obsolete
revision without confusing two distinct drawings.

A renderer that continuously animates the same drawing uses retained
preparation instead. `drawing.prepared()` provides the unbounded preparation
shared by every placement of that drawing;
`drawing.prepared(width, height)` retains one for each frame size. A later
revision of the same Canvas updates those objects in place, so a renderer can
share the compilation without rebuilding or replacing every component.

For content animation, `Path.reset()` preserves the path object, clears its
segments, and advances its revision before accepting new segments.
`TextBlock.replace(value)` likewise preserves the block identity, invalidates
its shaping, and advances only its revision when the value changes. A Canvas
that references these resources automatically updates its revision, bounds,
and preparation; it does not need to be cleared and repainted.

```sx
path.reset()..move_to(start)..line_to(animated_end)
label.replace("Frame B")
let current = drawing.prepared(320, 80)
```

`Snapshot.content()` and `Prepared.content()` preserve the general order as
`geometry`, `image`, `text`, and `layer` values. Layer content exposes its effects,
bounds, identity, and revision. An alternative renderer can therefore replay
shapes, text, and groups in author order without trying to reconstruct it from
an aggregate mesh and a separate text list.
Nested `LayerContent.content_view()` offers immediate traversal without taking
another list copy; its read-only view remains bound to the layer.

```sx
var prepared = drawing.prepared()

// After clear(), paint(...), or any other revision of the same Canvas:
prepared = drawing.prepared()
let geometry = prepared.geometry()
```

To explicitly own an independent preparation, create `Canvas.Prepared` and
pass it to `prepare(...)`. The Canvas then rewrites geometry into two
alternating retained storages: their capacity is recycled, the previous
topology remains available to detect unchanged geometry, and text layers
update independently.

```sx
var prepared = Canvas.Prepared()
drawing.prepare(prepared)

// After clear(), paint(...), or any other revision of the same Canvas:
drawing.prepare(prepared)
let geometry = prepared.geometry()
```

The `prepared(width, height)` and `prepare(prepared, width, height)` overloads
frame text layers. Circles,
rectangles, and straight lines compile to six-vertex analytic quads; the shader
evaluates radius, width, orientation, and cap instead of retessellating rounded
contours on the CPU.

`Snapshot.text_layers()` gives every text command a stable identity, its
canonical `GlyphRun`, and its own lazy coverage image. `glyph_run()` returns a
homogeneous line. `glyph_runs()` covers multiline text and lines using multiple
faces. `vector_outlines()` preserves the same glyph IDs, clusters, and positions for
a vector renderer. `coverage_runs(density)` exposes the same runs with their
rounded physical origin without rasterizing, so a renderer can consult its
atlas before requesting a missing glyph's coverage. A renderer can therefore
upload only the changed label. Canvas’s direct hinted-bitmap cache is bounded
to 2,048 slots and 16 MiB in addition to
GFX.Font’s native budgets.

`TextGlyphPlacement.face_identity` identifies the outline face and `embolden`
indicates synthetic bold. Vector caches distinguish glyph IDs from different
faces. `outline_identity` identifies normalized outlines with their face,
variations and synthesis independently of text size. It is opaque and can be
used as a cache key.

`TextLayer.shaping_revision()` changes only when text, face, size, variations,
or typographic placement changes. A retained renderer can therefore read
`color()` and update a color instance without decomposing or tessellating the
outlines again.

`TextStyle.fixed_advance` shapes the whole line, disables ligatures,
contextual alternates, and kerning that would cross a cell, then places each
Unicode cluster at an exact grid origin. Marks within one cluster stay in the
same run; no UTF-8 byte is handled in isolation. This option suits terminal or
editor rows; ordinary text keeps the font's natural advances.

## Paint a mutable surface

`Canvas.Surface` is a mutable RGBA pixel surface. It accepts the same commands,
composes images, and returns a `GFX.Assets.Image`.

Canvas depends on neither Viewer nor Scene2D. When a drawing needs display,
`GFX.Viewer` accepts the Canvas and its dimensions, while `GFX.Scene2D`
provides retained GPU rendering. The [2D gallery](https://github.com/Matanek/Silex-Examples/tree/main/Sources/ShapeGallery2D)
and [RetainedCanvasGeometry](https://github.com/Matanek/Silex-Benchmarks/blob/main/Sources/RetainedCanvasGeometry.sx)
benchmark belong to their central repositories.

## Produce an interactive drawing

`Canvas.Session` describes a stateful producer without choosing its presentation
architecture:

```sx
class Preview:Canvas.Session {
    func drawing(width:int, height:int) Canvas { return build(width, height) }
    func update(delta:float) bool { return advance(delta) }
    func handle(event:@GFX.Input.Event) bool { return interact(event) }
    func accepts_text_input() bool { return editing() }
    func text_input_target() Canvas.TextInputTarget? { return null }
    func pointer_cursor() GFX.Window.Cursor { return GFX.Window.Cursor.standard }
}
```

The fragment's functions belong to the producer. Boolean results invalidate
the current drawing. The session retains its state and receives platform input
but owns no window, Application, renderer, or ECS world.

## Understand the native boundary

GFX.Canvas no longer owns a font engine or font archive. It depends on GFX.Font
for faces, shaping, outlines, and hinted coverage. Its remaining SDL3 provider
only serves existing Canvas surfaces and composes `GFX.SDL3` without
redistributing SDL3. GFX.Font carries its own macOS ARM64, Linux x64, Windows
x64, and Windows ARM64 artifact matrix.

## Interactive session migration

A `Canvas.Session` implementation now provides
`pointer_cursor() GFX.Window.Cursor`. Return `GFX.Window.Cursor.standard` to
retain the arrow; a UI host forwards the method from its runtime or
`UI.Session`. This query is independent of drawing invalidation. Viewer applies
the intent after events even when the drawing is unchanged. Other Canvas
producers (fixed value, builder or animation) keep their API.

A session also provides `text_input_target() Canvas.TextInputTarget?`.
Return `null` when it does not supply an input area. A UI session forwards the
method from its runtime or `UI.Session`. For a custom editor, the target contains
an opaque `id` and a `Window.TextInputArea` in logical drawing coordinates;
keep the identity during composition and change it when switching editors or
cancelling from application code. Viewer then ends native composition and places
candidates near the caret.

## Display characters missing from the primary face

Bundled `Font.default()` and `Font.monospace()` select a fallback face for each
missing grapheme without splitting a combining accent or emoji sequence.
Noto Sans Mono supplies symbols, monochrome Noto Emoji supplies emojis, and
Noto Sans CJK SC supplies ideographs, kana and hangul. Covered Latin text keeps
its primary face. Fallback faces open only when needed and are reused; runtime
never downloads fonts or searches system fonts. The additional embedded bytes
amount to about 18.4 MB per executable using these fonts; they are not all
decoded at startup.

Measurement, retained drawing, raster, outlines and fixed grids use the same
runs. Joiners, variation selectors and tags guide shaping without requiring a
separate visible glyph. Explicitly loaded faces remain strict; an uncovered
character retains an inspectable missing glyph. Fallback does not implement
paragraph bidi composition or color emojis.
[Faces and regional forms](../../../GFX.Font/Docs/EN/README.md).

## Session accessibility semantics

`Canvas.Session.accessibility_source()` returns a `GFX.Accessibility.Source?`.
Its default body returns `null`: existing producers remain valid without an
additional method. Custom producers may provide their own source; `UI.Session`
supplies it automatically. Retain one reader during presentation because each
new source may track changes independently.

Viewer connects this capability to Window automatically on macOS. Native
actions use the producer’s state and wake the presenter; closing detaches peers
before destroying the window. Inspection and VoiceOver qualification is ongoing.
Linux and Windows bridges remain future work.
