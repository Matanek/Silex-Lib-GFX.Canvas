# Draw with GFX.Canvas

`GFX.Canvas` describes 2D drawing intentions without imposing a renderer. A
`Canvas` records paths, analytic shapes, fills, strokes, and text, then turns
them into vector geometry or an RGBA image.

[Lire cette documentation en français.](../FR/README.md)

## Install the package

```text
silex install GFX.Canvas
```

GFX.Canvas requires Silex 0.39.0 or newer.

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

The package embeds a proportional default font and a monospace font available
through `Canvas.Font.monospace()`. An application can also supply a path or
encoded bytes. SDL3_ttf stays behind the Font API; no SDL handle is public.

## Reuse drawing intentions

`Canvas.commands()` returns a copy of public commands, `Path.steps()` exposes
segments, and `Canvas.vector_geometry()` produces a GPU-independent mesh. An
extension can therefore interpret intentions, tessellate them itself, or reuse
the supplied tessellation.

```sx
for command in drawing.commands() {
    match command {
        fill_rect(value) => { print("rectangle") }
        text(value) => { print(value.value) }
        else => {}
    }
}
```

`Canvas.snapshot()` preserves geometry in its local coordinates without
inventing a drawing rectangle. Every text layer computes its own coverage,
including when its origin is negative. `Canvas.snapshot(width, height)` remains
available when a viewport, layout, or export must clip text to a precise frame.

For successive revisions, pass the previous snapshot as `reuse:previous`.
Identical text commands retain their revision and glyph coverage while vector
geometry can change independently.

A renderer that continuously animates the same drawing uses retained
preparation instead. `drawing.prepared()` provides the unbounded preparation
shared by every placement of that drawing;
`drawing.prepared(width, height)` retains one for each frame size. A later
revision of the same Canvas updates those objects in place, so a renderer can
share the compilation without rebuilding or replacing every component.

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

`Snapshot.text_layers()` gives every text command a stable identity and its own
lazy coverage image. A renderer can therefore upload only the changed label.
The coverage cache is bounded to 256 entries.

`TextStyle.fixed_advance` places Unicode graphemes on an explicit horizontal
grid while retaining one text command. This option suits terminal or editor
rows; ordinary text keeps the font's natural advances.

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
}
```

The fragment's functions belong to the producer. Boolean results invalidate
the current drawing. The session retains its state and receives platform input
but owns no window, Application, renderer, or ECS world.

## Understand the native boundary

GFX.Canvas owns SDL3_ttf and composes its provider with `GFX.SDL3` without
redistributing SDL3. Pinned artifacts exist for macOS ARM64, Linux x64, Windows
x64, and Windows ARM64; the latter remains a recognized experimental target in
the Silex matrix.
